---
layout: post
title: Ghostscript
---

<p class="message">
At the time of writing, the current ghostscript version is 9.56.1. Full code described in the article, can be found <a href="https://github.com/zpieslak/scripts/tree/main/gs">here</a>.
</p>

Ghostscript is an excellent interpreter for Postscript and PDf files. It is installed on almost all modern Linux distributions per default.


In this article, I would like to present how to unify multiple Pdfs into one file, create a bookmarks list inside it and add page numbers on each page of the generated document.

<!--more-->

## Combine files

Firstly let's start with combining Pdfs into one file. It is relatively easy and can be done with the following command:

    gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=output.pdf first_input.pdf second_input.pdf

Where `-dNOPAUSE` and `-dBATCH` disables interactive prompts, `-sDEVICE` chooses a so-called output device (all ghostscript devices can be found [here](https://www.ghostscript.com/doc/current/Use.htm#Output_device)) and `-sOutputFile` sends the output to file. Full [reference](https://www.ghostscript.com/doc/current/Use.htm).

## Pdf bookmarks

Once we know how to combine the files, let's continue with generating the Pdf bookmarks. In order to generate bookmark in combined Pdf, we need to know what will be the position (page) of the bookmark in the combined file. This can be done, by simple calculating the number of the pages of each Pdf input.

The following command takes an input file `first_input.pdf` and outputs the number of pages to stdout.

    gs -dQUIET -dNODISPLAY --permit-file-read=first_input.pdf -c "(first_input.pdf) (r) file runpdfbegin pdfpagecount = quit"

Where `-dQUIET` removes any standard stdout comments, produced by ghostscript, `-dNODISPLAY` runs with null device, `--permit-file-read=first_input.pdf` gives ghostscript permission to read the input file, `-c` allows running PostScript code in commandline, instead of providing a file. `runpdfbegin` and `pdfpagecount` are PostScript procedures, provided by ghostscript ([reference](https://github.com/ArtifexSoftware/ghostpdl/blob/1149c5ab914c7695caa8951bb8213f4241c51104/Resource/Init/pdf_main.ps)).

Another property we may need from input file is its title (for example to provide it as boookmark title). This can be done with following command:

    gs -dBATCH -dQUIET -dPDFINFO -dNODISPLAY first_input.pdf 2>&1 | grep "Title: " | awk -F ': ' '{ print $NF }'

Once we know what will be the position of the input file, we can generate Pdf bookmarks. Plus, it would be nice to add new metadata to the generated file. An example PostScript code would look as below:

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

We can provide this code either by `-c` option (as described earlier) or by providing a separate file. In this example we saved it as `bookmark.ps`.

    gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=output.pdf bookmark.ps first_input.pdf second_input.pdf

## Page numbers

The last thing (and the most complex) is to add page numbers to each page of the output edf. Preferably, page number should be added to the right bottom of the page. The example PostScript code would look like:

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

If we save the above code as `pages.ps`. The full command would look like this:

    gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=output.pdf bookmark.ps pages.ps first_input.pdf second_input.pdf

Full PostScript reference can be found [here](https://www.adobe.com/jp/print/postscript/pdfs/PLRM.pdf).

## Wrapping up

Taking all of the solutions above, we can create a simple bash script that will take Pdf files as input and output the merged Pdf.

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

      # Incrment page number
      PAGE_NUMBER=$(($PAGE_NUMBER + $PAGES_COUNT))
    done

    # Run main command
    gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=$OUTPUT_FILE "$CURRENT_DIR/pages.ps" $@ -c "$BOOKMARKS"
