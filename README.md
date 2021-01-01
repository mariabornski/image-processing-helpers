# image-processing-helpers
Collection of scripts for working with image metadata across multiple directories

## What is this?
At this time, this document describes plans, not reality (since I'm writing this
before writing the actual code :-D )

The idea here is to allow updating of metadata for large numbers of images
via the command line.

## How will this work?
### Setup
Config files should be placed in the same directory as the images to be updated.
Two types of config files are supported -- `all_images.json`, which will be
applied to all images in the directory, and `<name>.json`, which will be applied to
the corresponding `<name>.jpeg` file.  `all_images.json` is mandatory but can be empty.
`<name>.json` is optional.  Settings in `<name>.json` will override settings in
`all_images.json`.  `all_images.json` applies only to files at the same directory
level -- subdirectories must have their own `all_images.json` to be acted upon.

For example, given a directory structure like:
```
scanned_slides/
    trip1/
        all_images.json
        image1.json
        image1.jpeg
        image2.jpeg
    trip2/
        all_images.json
        image4.jpeg
        image5.jpeg
        sub_trip2/
            image7.jpeg
```

Running on `scanned_slides` would change the jpeg files as follows:
* `scanned_slides/trip1/image1.jpeg`
  * All settings from `scanned_slides/trip1/image1.json`
  * Any settings in `scanned_slides/trip1/all_images.json` not overridden in `image1.json`
* `scanned_slides/trip1/image2.jpeg`
  * All settings in `scanned_slides/trip1/all_images.json`
* `scanned_slides/trip2/image4.jpeg`
  * All settings in `scanned_slides/trip2/all_images.json`
* `scanned_slides/trip2/image5.jpeg`
  * All settings in `scanned_slides/trip2/all_images.json`
* `scanned_slides/trip2/sub_trip2/image7.jpeg`
  * No change

### Input
We'll have the following logical modes for specifying which files to process:
* Specify a directory, let the script find and process all images in all directories accessible from that directory containing an `all_images.json` file
  * Directory list would basically be `find <directory> *name all_images.json *printf %h`
* Specify a directory, let the script process all images in ONLY that directory.
  * Here, do not traverse sub-directories.  `all_images.json` must exist in the specified directory that will be operated on.
* Specify a list of one or more image files to be operated on
  * Each image file must have a corresponding `all_images.json`, and may have a corresponding `<file>.json`
  * Only the specified files will be operated on, other files in the same directory will be ignored
  * The specified files may be from different directories

Additionally, we'll allow the user to specify how we'll output the modified files.
In general, to avoid damaging original image files, we'll try very hard to avoid
overwriting the original files, so:
* Allow the user to specify an output directory
* Allow the user to specify a prefix (or suffix? TBD) for the modified files
* If the input and output directories are the same, require the user to specify a prefix
* Maybe: allow the user to specify that files should be output "flat" to one directory?
  * If so, could prefix with file path but with `/` (or `\`, based on the OS) replaced with `_` (eg `trip1_image1.jpeg`, `trip2_image4.jpeg`, etc)

Within the output directory, output with the same directory structure we read from.  So, we'll need to keep
track of:
* Common root/prefix directory for all files
  * Where the user specified a directory, either for us to traverse or for us to read only that directory, the user specified directory is the common root in question
  * Where the user specified specific files, we'll have to calculate the common root
  * This will mostly, I think, be used for figuring out whether our input and output directories are the same
* Additional sub-directories/prefix directories for each file
* Filename without directories or extension
* File extension

Taking a couple of examples and building on our previous example to do some psuedo code,
if our input directory was `scanned_slides/`, then our file info for a couple of our files
would be something like:
```
{ name:"image7" ; extension:".jpeg"; subpath:"trip2/sub_trip2" }
{ name:"image1" ; extension:".jpeg"; subpath:"trip1" }
```

Our output info, then, would something like:
```
output_file = filesystem.glob(output_directory, (this_image.sub_path, prefix + this_image.name + extension))
```

If we wanted to allow a "flat" output, then our output file would be more like:
```
output_file = filesystem.glob(output_directory, (replace(this_image.sub_path, "/", "_") + prefix + this_image.name + extextension))
```

That said, we'd need to do the above in a way that handles different OSes correctly -- the directory
separators are, of course, different on different operating systems.

### Code flow
So then, our flow will be something like:
* Calculate what work to do:
  * Calculate what directories to operate on
    * Depending on how we were invoked, this will be calculated via one of the following:
      * `find <directory> -name all_images.json -printf %h`
      * Specified directory on the command line
      * Union of the directories that the images passed on the command line live in
    * Skip a directory we would otherwise attempt to process if any of the following are true:
      * Directory contains no image files
      * `all_images.json` is empty AND (no `<file>.json` instances present OR all `<file>.json` instances also empty)
  * Calculate what image files to operate on
    * Again, depending on how we were invoked, this will be one of the following:
      * All supported image files in each directory with an `all_images.json`
      * All supported image files in the directory specified on the command line
      * Specific image files specified on the command line
* Double check our planned operations:
  * Print messages for the user about:
    * Any directories we skipped that had `all_images.json` but no supported image files
    * Any directories we skipped that had supported image files but empty `all_images.json` and no `<file>.json`
    * Any specified but unsupported image files that we're skipping
  * Exit if:
    * Directories to operate on is empty
    * Image files to operate on is empty
* For each directory we are operating on:
  * Process and ingest the settings by:
    * Ingesting `all_images.json` into a default `settings` structure of some kind
    * For each `<file>.json`:
      * Copying the default `settings` to a `<file> -> settings` mapping
      * Updating said `settings` for this `<file>` with any settings from `<file>.json`
        * Where there was a value in the defauts, this will overwrite it
        * Where there was no value in the defaults, this will add a value
  * For each supported image file from this directory that we are modifying:
    * If there is a corresponding `<file> -> settings` mapping, transform the file accordingly
    * If there is no corresponding `<file> -> settings` mapping, transform the file according to
  the defaults
    * Write the transformed image to the appropriate output location (as described above)

### More Info
#### What actually goes in these config files?

Metadata updates & potentially watermarking.

This explains how to use exiftool to update metadata: https://www.linux.com/news/how-add-metadata-digital-pictures-command-line/
This explains how to use ImageMagick to add watermarks: https://stackoverflow.com/questions/5533458/how-to-add-watermarks-to-images-via-command-line-hopefully-using-irfanview

ImageMagick docs: https://imagemagick.org/index.php
ExifTool docs: https://exiftool.org/


#### What image types will be supported?

JPEG/JPG for sure, potentially others.

I'm writing this to help myself process some old family photos that I'm scanning from
negatives and slides.  My slide and negatives scanner outputs .jpeg, so that will be my priority.
