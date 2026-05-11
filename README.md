# Zippey: A Git filter for friendly handling of ZIP-based files

## Motivation

There are many types of ZIP-based files that contain plain text content,
such as:

* Microsoft Office
  (`.docx`, `.xlsx`, `.pptx`)
* OpenOffice
  (`.odt`)
* Java Archive
  (`.jar`)
* FreeCAD
  (`.fcstd`)

They can not efficiently be tracked by git,
since the compression smears what parts have been modified
and what parts remain the same across commits.
This saves the archived text files as a new binary blob,
every time the file is modified,
which prevents Git from versioning these files in a meaningful way.

## Method

Zippey is a Git filter that unzips zip-based files
into a simple text format during `git add`/`git commit` ("clean" process)
and recover the original zip-based file after `git checkout` ("smudge" process).

## Benefits

1. Since the diff is taken on the "clean" file,
it is likely that the real changes to the file can be reflected in a meaningful way
by the built-in `git diff` command.
This solves the problem of diffs for these files
normally being useless and unreadable for humans.
2. The second benefit, is that the repository might end up much smaller in disc-size.
Imagine you have one huge zip archive (~100MB), and over a series of 10 commits,
you change only a small portion of one text file contained in this archive.
Traditionally, the archive data might likely change completely in each commit,
and thus your repo size would grow by 10 * 100MB.
With this filter though, the repo will only grow by the size of the actual changes
in the text file, which might sum up to only 1KB.

## File Format

Zippey uses a standard MIME multipart format (RFC 2046) to store the unzipped content.
The file starts with global MIME headers, followed by individual parts for each file in
the archive.

Each part contains:

1.  **Headers**:
    *   `Content-Type`: The detected MIME type of the file.
    *   `Content-Disposition`: Includes the original filename
         (including the path, if the zipped file contains directories)
    *   `Content-MD5`: MD5 sum of the raw content.
    *   `Content-Length-Raw`: Size of the original file in bytes.
    *   `Content-Length-Encoded`: Size of the stored content (Base64-encoded or raw).
    *   `Content-Transfer-Encoding`: `8bit` for text files, `base64` for binary files.
2.  **Body**:
    *   Text files are stored as-is (`8bit`).
    *   Binary files are stored using MIME-compliant Base64 encoding.

Example of a part header:

    --======ZIPPEY_FILE_BOUNDARY======
    Content-Type: application/octet-stream
    Content-Disposition: attachment; filename="image.png"
    Content-MD5: d41d8cd98f00b204e9800998ecf8427e
    Content-Length-Raw: 1234
    Content-Length-Encoded: 1672
    Content-Transfer-Encoding: base64


## How to use

### Setup

Before your first use of this filter,
you need to set it up.

Make sure that you have Python installed in your system,
and that the `python` command is available through your `PATH` env variable under Windows.

Then you need to install this filter with Git.

For this, clone the repository and change into it:

    git clone https://github.com/rockstorm101/zippey.git
    cd zippey

Then, we add the filters,

on Unix/Linux/BSD/OSX:

    # smudge filter
    git config --global --replace-all filter.zippey.smudge "$PWD/zippey.py d"
    # clean filter
    git config --global --replace-all filter.zippey.clean  "$PWD/zippey.py e"

on Windows:

    # smudge filter
    git config --global --replace-all filter.zippey.smudge "python %cd%/zippey.py d"
    # clean filter
    git config --global --replace-all filter.zippey.clean  "python %cd%/zippey.py e"

### Use

Now we still need to enable the filter,
which is best done by adding a `.gitattributes` file to each repository
in which you want to use this filter.

This is sample content of a `.gitattributes` file
that enforces Microsoft Word files to use this filter:

    *.docx filter=zippey

## License

This project is licensed under the BSD 3-clause "New"
or "Revised" license (bsd-3-clause).

## Side-effects

In the repository, the file is _not_ saved as an archive,
but as an uncompressed,
text-based representation of the content and meta information.
There are two circumstances under which this might be a problem:

1. If you download a file handled by this filter
  directly from an online repository website -
  for example through the GitHub web interface -
  it will still be in the text format.
  This also applies if you download the whole repo content
  as an archive.
2. If you have a local repository,
  but you do not have this filter setup,
  you also end up with the file in text format.

__Workaround:__ If it is just a single file, you can recover it by:

    python zippey.py d < downloaded-text-file > recovered-zip-based-file

## Unit-Tests

These check basic validy of the code as is.
Run like this:

    python test_zippey.py

