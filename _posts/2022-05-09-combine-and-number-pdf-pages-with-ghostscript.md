---
layout: post
title: Combine and Number PDF Pages with Ghostscript
redirect_from:
  - /2022/05/09/ghostscript/
---

Ghostscript is an excellent interpreter for PostScript and PDF files and is installed by default on almost all modern Linux distributions.

In this article, I will present how to unify multiple PDFs into one file, create a bookmarks list within it, and add page numbers to each page of the generated document.

<!--more-->

<p class="message">
As of this writing, the current version of Ghostscript is 9.56.1. The full code described in the article can be found <a href="https://github.com/zpieslak/scripts/tree/main/gs">here</a>.
</p>

## Combine files

First, let's start with combining PDFs into one file. This task is relatively easy and can be accomplished using the following command:

    gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=output.pdf first_input.pdf second_input.pdf

The options `-dNOPAUSE` and `-dBATCH` disable interactive prompts, `-sDEVICE` selects an output device (a list of all Ghostscript devices can be found [here](https://ghostscript.com/docs/9.56.1/Devices.htm)) and `-sOutputFile` directs the output to file. For additional information, refer to the [full reference](https://ghostscript.com/docs/9.56.1/Readme.htm).

## Pdf bookmarks

Now that we know how to combine the files, let's proceed with generating the PDF bookmarks. To generate bookmarks in the combined PDF, we need to determine the position (page) of each bookmark in the combined file. This can be accomplished by simply calculating the number of pages in each input PDF.

The following command takes an input file `first_input.pdf` and outputs the number of pages to the standard output (stdout).

    gs -dQUIET -dNODISPLAY --permit-file-read=first_input.pdf -c "(first_input.pdf) (r) file runpdfbegin pdfpagecount = quit"

Here, `-dQUIET` suppresses standard stdout comments produced by Ghostscript; `-dNODISPLAY` runs Ghostscript with a null device; `--permit-file-read=first_input.pdf` grants Ghostscript read permission for the input file; and `-c` allows the execution of PostScript code from the command line, negating the need to provide a file. Both `runpdfbegin` and `pdfpagecount` are PostScript procedures provided by Ghostscript. For more details, refer to the [reference](https://github.com/ArtifexSoftware/ghostpdl/blob/1149c5ab914c7695caa8951bb8213f4241c51104/Resource/Init/pdf_main.ps)).

Another property we may need from the input file is its title (for example, to use it as the bookmark title). This can be achieved with the following command:

    gs -dBATCH -dQUIET -dPDFINFO -dNODISPLAY first_input.pdf 2>&1 | grep "Title: " | awk -F ': ' '{ print $NF }'

Once we determine the position of the input file, we can generate PDF bookmarks. Additionally, it would be beneficial to add new metadata to the generated file. An example of PostScript code would look like the following:

    % Main file metadata
    [ /Title (My Title for output pdf)
      /Subject (My Subject for output pdf)
      /Author (John Doe)
      /DOCINFO pdfmark

    % Bookmark list
    [
      /Page 2
      /Title (My Title for first input pdf)
      /OUT pdfmark
    [
      /Page 3 % Value here is calculated from first_input.pdf pages count + 1
      /Title (My Title for second input pdf)
      /OUT pdfmark

We can supply this code using the `-c` option (as described earlier) or by providing it within a separate file. In this example, it’s saved as `bookmark.ps`.

    gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=output.pdf bookmark.ps first_input.pdf second_input.pdf

## Page numbers

The final, and most complex, step is to add page numbers to each page of the output PDF. Ideally, the page number should be placed at the bottom right of the page. Example PostScript code would look as follows:

    % Set PageCount
    globaldict /PageCount 1 put
    <<
      % A procedure to be executed at the end of each page
      /EndPage {
        % Run on showpage or copypage phase
        exch pop 0 eq dup {
          % Prepare PageCount
          /Helvetica 12 selectfont PageCount =string cvs

          % Read string size and put width to the stack
          dup stringwidth pop

          % Read device size
          currentpagedevice /PageSize get 0 get

          % Place in bottom right
          exch sub 60 sub 30

          % Draw
          moveto show

          % Increment PageCount
          globaldict /PageCount PageCount 1 add put
        } if
      } bind
    >>
    setpagedevice

If we save the above code as `pages.ps`, the full command would look like this:

    gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=output.pdf bookmark.ps pages.ps first_input.pdf second_input.pdf

Full PostScript reference can be found [here](https://www.adobe.com/jp/print/postscript/pdfs/PLRM.pdf).

## Wrapping up

By integrating all the above solutions, we can create a simple Bash script that will accept PDF files as input and produce a merged PDF as output.

    #!/bin/bash

    # Usage
    # gs/merge.sh *.pdf

    # Setup
    OUTPUT_FILE=output.pdf
    CURRENT_DIR=$(dirname "$0")
    PAGE_NUMBER=1
    BOOKMARKS="[
      /Title (Output)
      /Subject (Subject)
      /Author (John Doe)
      /DOCINFO pdfmark
    "

    # Loop through all files
    for FILE in "$@"; do
      # Count pages
      PAGES_COUNT=$(gs -dQUIET -dNODISPLAY --permit-file-read=$FILE -c "($FILE) (r) file runpdfbegin pdfpagecount = quit")
      TITLE=$(gs -dBATCH -dQUIET -dPDFINFO -dNODISPLAY $FILE 2>&1 | grep "Title: " | awk -F ': ' '{ print $NF }')

      # Assign file path if Title is empty
      TITLE=${TITLE:-$FILE}

      # Create bookmarks
      BOOKMARKS="$BOOKMARKS [
        /Page $PAGE_NUMBER
        /Title ($TITLE)
        /OUT pdfmark
      "

      # Increment page number
      PAGE_NUMBER=$(($PAGE_NUMBER + $PAGES_COUNT))
    done

    # Run main command
    gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=$OUTPUT_FILE "$CURRENT_DIR/pages.ps" $@ -c "$BOOKMARKS"
