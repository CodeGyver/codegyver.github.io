---
layout: post
title: Exiftool AWS Lambda
---

Exiftool is excellent file metadata parser and analyzer. Internally, it is written in perl and provides a support for variety of file formats.

In this article I will present how to use it in disk and RAM limited environment like AWS Lambda.

<!--more-->

<p class="message">
The article describes code, which can be found <a href="https://github.com/zpieslak/exiftool-aws-lamdba">here</a>.
</p>

## AWS Lambda

AWS Lambda is Amazon serverless solution, that allows to run arbitrary code in few predefined languages. Internally it is a virtual private server that allows us to define some of its computation resources (like RAM and - indirectly - CPU). Unlike "standard" VPS its purpose is to run per defined event and then tear down. Although managing the Lambda instances is Amazon internal implementation, we can distinguish "cold" and "warm" phases. "Cold" is simple a phase that instance needs to started from scratch. "Warm" on the other hand, re-uses the already running instance (as Lambda are not torn down momentary). As a result, two consecutive runs of AWS Lambda will not last the same time (the second run will be faster - since it uses the "warm" phase).

Because of its purpose (event driven and ephemeral runs) the AWS Lambda are itself very limited environments. They use Amazon Linux 2 (a fork of Red Hat Enterprise Linux) with addition of the language tool that is used for particular image (python, ruby, node, etc.). If we want to add additional binaries, we have a few options: create a custom docker image, use custom boot script or use Lambda `Layer`. The benefit over using `Layer`, is that we are not charged for initial duration of the "cold" phase.

## Layer

`Layer` is simply a custom zip file that we create. When Lambda is prepared to run, its contents are unzipped in `/opt` directory of the instance. Per default `opt/bin` is added to `PATH` environment variable and `/opt/lib` to `LD_LIBRARY_PATH` - [reference](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html). So for example, if we want to add new executable we could simply zip it as `bin/executable` and it should be available to run from Lambda code.

In our case, we need to zip [exiftool](https://exiftool.org) and all its dependencies (`perl` in this case). In order to mimic the Lambda environment as close as possible, I decided to use a base Lambda docker image and perform any needed changes there. The code that builds a base Lambda image with all necessary build tools is put under `Dockerfile.layer`:

    FROM public.ecr.aws/lambda/python:3.9

    # Install build tools
    RUN yum install -y vim make gcc gzip tar zip

    COPY entrypoint.layer.sh ./

    # Fix permissions
    RUN chmod +x entrypoint.layer.sh

    ENTRYPOINT ["./entrypoint.layer.sh"]

Since fetching and zipping files needs access to shared layer (in order to copy the result to the mother machine) and it does not need to leverage the docker build cache, I thought it would be more natural to put it in "run" part of the docker.


The code that fetches the executables is put under `entrypoint.layer.sh`.

    #!/bin/bash

    # Setup
    BIN_PATH=/opt
    EXIFTOOL_URL=https://exiftool.org/Image-ExifTool-12.44.tar.gz
    LAYER_PATH=/var/task/layer
    PERL_URL=https://www.cpan.org/src/5.0/perl-5.34.1.tar.gz

    # Install perl
    curl $PERL_URL | tar xz
    cd perl-*
    ./Configure -de -Dman1dir=none -Dman3dir=none -Dprefix=$BIN_PATH
    make
    make install

    # Install exiftool
    curl $EXIFTOOL_URL | tar xz
    mkdir -p $BIN_PATH/bin/lib
    cp ./Image-ExifTool*/exiftool $BIN_PATH/bin/
    cp -r ./Image-ExifTool*/lib/* $BIN_PATH/bin/lib/
    sed -i "1 s|^.*$|#!$BIN_PATH/bin/perl -w|" $BIN_PATH/bin/exiftool

    # Copy files to shared path
    cd $BIN_PATH
    zip -FSr $LAYER_PATH/exiftool.zip bin/perl lib/perl*/* bin/exiftool bin/lib/*

The above code can be wrap with `docker build` and `docker run` commands, or alternatively it can be wrap with `docker compose` and put under `docker-compose.yml`:

    ---
    version: '3.8'

    services:
      layer:
        build:
          context: .
          dockerfile: Dockerfile.layer
        container_name: exiftool_layer_container
        image: exiftool_layer
        ports:
          - 8080:8080
        volumes:
          - ./layer:/var/task/layer

In order to generate the layer, the following command can be run (as a result we should have the `exiftool.zip` file in the `./layer` directory):

    docker compose -f docker-compose.yml up --build --force-recreate layer

## Application

Our application will react on hook, provided on successful file upload to predefined S3 bucket. It is presented on AWS diagram below

    +---------------+                            +-------------------------+
    |               |  s3:ObjectCreated:* event  |                         |
    |   S3 bucket   +--------------------------->|   Exiftool AWS Lambda   |
    |               |                            |                         |
    +---------------+                            +-------------------------+

The constraints that we will put is 512 MB of temp disk size (default) and 256 MB of RAM. In order to be able to fetch large files and not exhaust Lambda, we will need to use a couple of features and combine them together.

### Exiftool usage

Exiftool provides `-fast` flag, which forces executable to exit, as soon as it fetches the metadata. So as a result, only a part of the file is read (this is especially useful for large files).

The other feature we will use, is allow using `stdin` as input instead of file path. This will be useful for better manage the system cache (to not download the whole file at once from remote destination).

Given we have a test file called [test.pdf](https://github.com/zpieslak/exiftool-aws-lamdba/blob/main/tests/fixtures/files/test.pdf), we can run the following code:

    cat tests/fixtures/files/test.pdf | exiftool -fast -json -

And the results would be:

    [{
      "SourceFile": "-",
      "ExifToolVersion": 12.42,
      "FileSize": "0 bytes",
      "FileModifyDate": "2022:08:22 14:29:40+02:00",
      "FileAccessDate": "2022:08:22 14:29:40+02:00",
      "FileInodeChangeDate": "2022:08:22 14:29:40+02:00",
      "FilePermissions": "prw-------",
      "FileType": "PDF",
      "FileTypeExtension": "pdf",
      "MIMEType": "application/pdf",
      "PDFVersion": 1.4,
      "Linearized": "No",
      "PageCount": 1,
      "XMPToolkit": "XMP toolkit 2.9.1-13, framework 1.6",
      "Producer": "GPL Ghostscript 9.56.1",
      "ModifyDate": "2022:05:31 15:10:30+02:00",
      "CreateDate": "2022:05:31 15:10:30+02:00",
      "CreatorTool": "GNU Enscript 1.6.6",
      "DocumentID": "uuid:9881c87a-18ff-11f8-0000-8aa3554e5edc",
      "Format": "application/pdf",
      "Title": "Enscript Output",
      "Creator": "",
      "Author": ""
    }]

The exiftool will break the bash pipe and exit as soon as it gets all information.

One thing to note, as presented above, we are missing file size (all other metadata headers are extracted). However, since this property is provided by AWS S3 (via Content-Length header), I think this is acceptable.

### S3 GETObject

To retrieve the file itself, we will use [S3 GETObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html) API call. Fortunately S3 provides ability to fetch only certain bytes of the file, via [Range header](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html#API_GetObject_RequestSyntax). Having the above we can construct the python loop that fetches consecutive bytes of the file and fills exiftool stdin in each iteration. Once exiftool gathers all information it throws `BrokenPipeError` and loop terminates (preventing from downloading any further chunks). It is presented in the code below:


    import json
    import operator
    import os
    import subprocess
    from .s3.object import Object as S3Object
    from .s3.object_iterator import ObjectIterator as S3ObjectIterator
    from typing import Any, Dict


    def handler(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
        bucket, object = operator.itemgetter('bucket', 'object')(
            event['Records'][0]['s3']
        )

        s3_object_iterator = S3ObjectIterator(
            S3Object(bucket['name'], object['key'])
        )

        output = subprocess.Popen(
            [str(os.getenv('EXIFTOOL_BIN')), '-json', '-fast', '-'],
            stdout=subprocess.PIPE,
            stdin=subprocess.PIPE,
            stderr=subprocess.PIPE
        )

        # BrokenPipeError is expected on larger files as exiftool
        # will broke pipe with `fast` flag
        if output.stdin is not None:
            try:
                for chunk in iter(s3_object_iterator):
                    output.stdin.write(chunk)
            except BrokenPipeError as exception:
                print(exception)

        # Wait for process termination
        stdout, stderr = output.communicate()

        return json.loads(
            stdout.decode()
        )[0]

Full working code, can be found [here](https://github.com/zpieslak/exiftool-aws-lamdba).

## Conclusion

Although AWS Lambda allows to manually adjust temp disk size and / or add RAM, it also goes with higher pricing. I believe that using the bare minimum provided by AWS, forces us to write more optimized code. Adding any additional packages should be well thought. Also for the majority of mime types we do not need to download the whole file to read its metadata and - in most cases - 4 MB or 8 MB is enough (since the extracted information is stored on the beginning of the file).
