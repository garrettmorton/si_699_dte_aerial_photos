# SI 699 - DTE Aerial Photo Collection Digital Curation Project
# Batch Processing Workflow Proof of Concept


## Table of Contents

* 1.0 [Purpose of Scripts](#purposeOfScripts)
* 2.0 [Script Descriptions](#scriptDescriptions)
  * 2.1 [process_batch.py](#processBatch)
  * 2.2 [extract_using_pypdf.py](#extractUsingPyPDF)
  * 2.3 [extract_using_poppler.py](#extractUsingPoppler)
  * 2.4 [georeference_links.py](#georeferenceLinks)
* 3.0 [Script Use and Access](#scriptUseAndAccess)


## <a name='purposeOfScripts'></a>Purpose of Scripts

These scripts were written as part of a digital curation project for a mastery course at the University of Michigan School of Information. Our client, Wayne State University Libraries Digital Collections unit, asked us to investigate the PDF files that comprise the [DTE Aerial Photo Collection](https://digital.library.wayne.edu/dte_aerial/). To assist with migrating the collection into the library's collection platform, we were asked look for ways to extract the images from the aerial photograph PDFs as image files, as well as for ways to recreate the location information contained with the index map PDFs in records for each individual image.

The Python scripts contained with this repository are designed to solve both of these tasks. `extract_using_pypdf.py` and `extract_using_poppler.py` are two scripts capable of extracting JPEG bytestreams and image metadata from the photo PDFs and writing the bytestreams to new files, as well as extracting document and link metadata from the index PDFs, in a target directory. `georeference_links.py` uses the link metadata from an index PDF generated by `extract_using_pypdf.py` to convert internal PDF coordinates into geocoordinates (longitude-latitude pairs). `process_batch.py` integrates the workflows laid out in the `extract_using_pypdf.py` and `georeference_links.py` scripts and combines the results of both into comprehensive records for each new JPEG file created. Each script -- its dependencies, its inputs and outputs, and how to run it -- is described in detail below.


## <a name='scriptDescriptions'></a>Script Descriptions


### <a name='processBatch'></a>process_batch.py

This script is designed to process the contents of a target directory in the DTE Aerial Photo Collection, where there will be one index PDF and some number of image PDFs, linked to in the index PDF. The script imports the `extract_using_pypdf.py` and `georeference_links.py` scripts described below and executes their workflow functions, ultimately creating new JPEG files for all image PDFs and various metadata files in JSON. One of the JSON files contains comprehensive records for the images -- integrating descriptive, locational, and technical metadata. The core of the script is a matching algorithm that seeks to pair image metadata records with link records from the index PDF that have been georeferenced.

#### Use

To run the script, enter the following command at your command prompt of choice. The command line options are described below.

`python process_batch.py [mode] [input path] [output path]`

There are two possible options for `[mode]`: `process` or `load`. `process` will run start fresh executions of the extraction and georeferencing workflows. `load` will instead open the metadata files produced by the last `process` run.

The value entered for `[input path]` should be a valid relative path from the current working directory to the target directory to process. If no value is entered for `[input path]` or `[output path]` (see below), the path used for the proof of concept ( 'input/pdf_files/part1/macomb/1961/' ) will be set.

The value entered for `[output path]` should be a valid relative path from the current working directory to the target directory to process. If no value is entered, the path used for the proof of concept ( 'output/' ) will be set.

#### Inputs

As `process_batch.py` also executes the workflows in `extract_using_pypdf.py` and `georeference_links.py`, it shares their inputs. See the descriptions below for details. While the workflow functions in those scripts write the data they collect to JSON files, they also return the data collected during them directly, making it unnecessary to load their inputs through file operations when using the `process` mode. With the `load` mode, the outputs from the two other workflow scripts are loaded: `batch_metadata.json` from the pypdf2 output subdirectory and `georeferenced_links.json` and from the output directory; both file names prefixed with `[county]_[year]_`, where `[county]` and `[year]` are the names of the county and year referenced in the path to the directory.

In addition, the script loads data from two csv files: `manual_pairs.csv` and `files_without_links.csv`. In testing the proof of concept, we discovered that the link metadata in the collection's index PDFs can be incorrect or incomplete. We encountered two main issues: 1) sometimes links would point to an incorrect file, leading to duplicate instances of file identifiers referenced in links and image files without any links referencing them; and 2) sometimes an image file would have no corresponding link because it was never embedded in the file, even though the identifier is displayed on the index map.

To help resolve these issues, we built tests into the script that check for anomalies, including whether no or multiple matches are made, or whether there are links that have not been paired with an image. If multiple are found, the PDF Object ID Numbers for each link, the identifiers used by the internal PDF file structure, will be reported as well. Once the script user has explained the flags, they can be manually addressed by adding data to `manual_pairs.csv` and `files_without_links.csv`. The format and source of the data for each of these files is explained below.

`manual_pairs.csv`

To fix the incorrect linking reported by the script, the user will need to visually inspect the index PDF and determine the PDF coordinates of apparent link locations, which we recommend doing using the open source [GNU Image Manipulation Program, or GIMP](https://www.gimp.org/) (a process described below in the `georeference_links.py` section). Using the coordinates and comparing them with those of the link records listed in the JSON document `[county]_[year]_georeferenced_links.json`, the correct pairs of images and links (identified by the PDF Object ID Numbers) can be determined. These matches can then be added in new rows to the `manual_pairs.csv` document, which must be encoded in UTF-8. The CSV file should have the following headers and values:

Index File Name | Image Identifier | PDF Object ID Number
---|---|---
The name of the targeted directory's index file, including the file ending | The string used in the image file name, a combination of letters, dashes, and numbers | The internal numeric identifier for the PDF link object, found in the output of `extract_using_pypdf.py` and `georeference_links.py`

`files_without_links.csv`

In cases where an image has no corresponding link record but its identifier appears in the index PDF, the locational details can be added to the image record using `files_without_link.csv` by specifying the PDF coordinates (found using GIMP or another means) for the identifier. Once the script loads the PDF coordinates, it converts them using functions from `georeference_links.py` to geocoordinates. The CSV file should have the following headers and values:

Index File Name	| File Identifier | GIMP X Coordinate | GIMP Y Coordinate
---|---|---|---
The name of the targeted directory's index file, including the file ending | The string used in the image file name, a combination of letters, dashes, and numbers | The X value to be converted to a longitude | The Y value to be converted to a latitude

#### Outputs

In addition to the outputs produced by the `extract_using_pypdf.py` and `georeference_links.py` workflows, the `process_batch.py` script produces a comprehensive metadata file containing image records called `dte_aerial_[county]_[year]_image_records.json`, where `[county]` and `[year]` are the names of the county and year referenced in the path to the directory.

Each image record in the JSON file contains the file name of the new JPEG file name, as well as descriptive, technical, and preservation metadata gathered by the scripts. An example of the output is provided below.

```
{
    "Descriptive": {
        "Year": "1961",
        "Index County": "Macomb",
        "File Identifier": "fm-11-100",
        "ArcGIS Current County": "Macomb County",
        "ArcGIS Geocoordinates": {
            "Longitude": -82.74493365978033,
            "Latitude": 42.77413107704377
        }
    },
    "Technical": {
        "Width": 5354,
        "Height": 5100,
        "ColorSpace": "DeviceGray",
        "BitsPerComponent": 8,
        "Filter": "DCTDecode"
    },
    "Preservation": {
        "Related Index File Name": "macomb61Index.pdf",
        "Match Details": {
            "Matching Method": "String matching on image file identifiers and file identifiers from links",
            "Link PDF Object ID Number": 631
        },
        "PDF Source Relative Path": "input\\pdf_files\\part1\\macomb\\1961\\fm-11-100.pdf",
        "Date and Time Created": "2019-6-3-19:36"
    },
    "File Name": "dte_aerial_fm-11-100.jpg"
}
```

#### Dependencies

Besides the dependencies passed on to it by `extract_using_pypdf.py` and `georeference_links.py` (see below), the script uses no third-party libraries. Local libraries referenced include the aforementioned scripts and an additional function file, `misc_functions.py`, which contains helper functions invoked by multiple scripts. The `sys`, `json`, `csv`, and `copy` standard Python libraries are also used.


### <a name='extractUsingPyPDF'></a>extract_using_pypdf.py

This script presents one of two programmatic solutions to the task of extracting JPEGs and document and link metadata from the collection's PDFs. Because it runs faster, is easier to setup, and gathers more technical metadata, we elected to integrate this script with `process_batch.py` over the other extraction solution (`extract_using_poppler.py`, described below). The script makes use of the third-party Python library PyPDF2 to process a target directory in the collection, handling PDFs with aerial photographs and the PDFs with index maps (there is likely only one of these) differently. Embedded JPEG bytestreams are isolated and written to new files, and metadata from both image and index PDFs are gathered and written to a JSON file. The general workflow of this script (and the `extract_using_poppler.py` script) is depicted in the diagram below.

<img src="static/extraction_workflow.jpeg" alt="Extraction Workflow Diagram" width="400"/>

#### Use

To run the script, enter the following command at your command prompt of choice. If the script itself is run and not imported from `process_batch.py`, the program targets the directory specified in the Main Program for processing.

`python extract_using_pypdf.py`

#### Inputs

The files that serve as input for this script are the aerial photograph or image PDFs and index map PDFs (usually one) contained within a directory specified by a relative path. A helper function collects the file paths for each file and then opens them individually as it executes the workflow. The program expects each image PDF to be named with an identifier string that ties it to a location on the index map (both visually within the map and through a file name used in an embedded link). The program also expects the index map PDF will be named with the name of the county depicted, the last two digits of the year it corresponds to, and then the string "Index".

#### Outputs

For each image PDF in the directory targeted for processing, the script will output a JPEG image with the same file identifier string, prefixed by `dte_aerial_`, to the output directory specified in the script's Main Program or through input to a function invocation. If the workflow is run through `process_batch.py`, a batch metadata file will be created called `[county]_[year]_batch_metadata.json`, where `[county]` and `[year]` are the names of the county and year referenced in the path to the directory. If the script is run directly, a batch metadata file called `sample_poppler_batch_metadata.json` will be created. Either batch metadata files will be saved to the same output directory as the JPEG images.

#### Dependencies

This script uses [PyPDF2](https://pythonhosted.org/PyPDF2/), an open-source library for reading and writing PDF files. The entire codebase is available in a [GitHub repository](https://github.com/mstamy2/PyPDF2). The use of PyPDF2 and some script features (particularly the bytestream extraction using an object attribute) were inspired by [an answer to a Stack Overflow question by sylvain](https://stackoverflow.com/questions/2693820/extract-images-from-pdf-without-resampling-in-python/34116472#34116472).


### <a name='extractUsingPoppler'></a>extract_using_poppler.py

This script presents one of two programmatic solutions to the task of extracting JPEGs and document and link metadata from the collection's PDFs. The script employs the third-party PDF rendering library Poppler to process a target directory in the collection, handling PDFs of aerial photographs and the PDFs of index maps (there is typically only one of these) differently. Embedded JPEG bytestreams are written to new files (using a command-line utility), and metadata from both image and index PDFs are gathered and written to a JSON file. The general workflow of this script is depicted in the diagram in the `extract_using_pypdf.py` section above.

#### Use

To run the script, enter the following command at your command prompt of choice. The script will target the directory provided in the Main Program for processing.

`python extract_using_poppler.py`

#### Inputs

The files that serve as input for this script are the aerial photograph or image PDFs and index map PDFs (usually one) contained within a directory specified by a relative path. A helper function collects the file paths for each file and then opens them individually as it executes the workflow. The program expects each image PDF to be named with an identifier string that ties it to a location on the index map (both through a string displayed visually in the map and through a file name used in an embedded link). The program also expects the index map PDF will be named with the name of the county depicted, the last two digits of the year it corresponds to, and then the string "Index".

#### Outputs

For each image PDF in the directory targeted for processing, the script will output a JPEG image with the same file identifier string, prefixed by `dte_aerial_`, to the output directory specified in the script's Main Program (`output/poppler/`). A batch metadata file called `sample_poppler_batch_metadata.json` will be created and saved in the same output directory.

#### Dependencies

`extract_using_poppler.py` makes use of an open source PDF rendering library and set of command-line utilities called [Poppler](https://poppler.freedesktop.org/). We wrote this script to run on a Linux operating system, as that way Poppler is easier to access. Working with the codebase through Python required the use of an intermediary API, [PyGObject](https://pygobject.readthedocs.io/en/latest/index.html). The [Poppler-specific PyGObject documentation](https://lazka.github.io/pgi-docs/#Poppler-0.18) proved useful in writing this script. In addition, a local library is referenced, the shared function file `misc_functions.py`. The `time`, `json`, and `subprocess` standard Python libraries are also used. The `subprocess` module is used to run one of the Poppler command-line utilities, `pdfimages`.


### <a name='georeferenceLinks'></a>georeference_links.py

The algorithm in this script uses the PDF rendering coordinates for the links in the index map PDF to determine real-world geographic coordinates for the images represented by those links. Using ArcGIS Desktop, we visually determined that the maps in the index PDFs use the Michigan State Plane coordinate system and are correctly oriented. Due to the Cartesian nature of the State Plane system and its local accuracy, we are able to use a linear transformation on the PDF rendering coordinates to calculate approximate geographic coordinates for the images.

In order to determine and apply the appropriate linear transformation, the algorithm uses the non-argument input of a CSV file called `address_pairs.csv`. This file needs to contain information about two different points on the index map (any two different street intersections are acceptable). The CSV contains one row for each different index PDF, with the following columns (explanation is provided below each):

Index File Name | Address 1 | Address 1 GIMP X Coordinate | Address 1 GIMP Y Coordinate | Address 2 | Address 2 GIMP X Coordinate | Address 2 GIMP Y Coordinate
---|---|---|---|---|---|---
The name of the index PDF described | A single string describing intersection #1 (e.g. "Bordman Road and Fisher Road, Bruce Township, MI 48065") | The PDF x coordinate for intersection #1 | The PDF y coordinate for intersection #1 | A single string describing intersection #2 | The PDF x coordinate for intersection #2 | The PDF for coordinate of intersection #2

The PDF rendering coordinates can be found using [GIMP](https://www.gimp.org/) or other image editing software such as Photoshop. The following directions apply to GIMP. After importing the index PDF into the application, the PDF coordinates for the intersection can be determined by hovering the cursor over the intersection and noting the coordinates listed at the bottom of the window. However, in a PDF, (0,0) is located at the bottom left-hand corner, increasing in the up and right directions, and in GIMP, (0,0) is located at the top left-hand corner, increasing in the down and right directions. To correct this, the image must be flipped vertically before reading the coordinates. Make sure these coordinates are displayed as points (pt) and not as pixels; PDF rendering is based on points and not pixels in order to preserve print output across systems.

<img src="static/finding_pdf_coordinates_in_gimp.png" alt="Finding PDF rendering coordinates using GIMP software" width="500"/>

The script takes these intersections and queries the ArcGIS API to find their geographic coordinates. It then uses the known equivalence of the geographic coordinates and PDF coordinates from the two intersections to calculate the linear transformation used to determine geographic coordinates of the index PDF links.

#### Use

To run the script, enter the following command at your command prompt of choice. The script will target the batch metadata file targeted in the Main Program for processing.

`python georeference_links.py [batch metadata path]`

The value entered for `[batch metadata path]` should be a valid relative path from the current working directory to the target batch metadata file with link records to georeference. The script's Main Program runs the georeferencing workflow on a user-defined batch metadata file to generate a sample JSON file. The functions in this script are called by the primary workflow script, `process_batch.py`, to georeference the images extracted by `extract_using_pypdf.py`.

#### Inputs

This script's primary function, `run_georeferencing_workflow()` takes as input the path to the metadata JSON file created by the `run_pypdf2_workflow()` function (also called by `process_batch.py`), the desired name of the output metadata file, and the path to the directory in which to create the output metadata file. In order to run the workflow also requires a CSV called `address_pairs.csv` (described above) in the input folder.

#### Outputs

This script's primary function, `run_georeferencing_workflow()`, returns a data dictionary containing a) information used in the georeferencing process (two address pairs and the calculated conversion formula constants) and b) a dictionary for each image link in the index PDF containing its PDF Object ID Number, the image identifier it links to, the link's PDF coordinates, the image's calculated longitude and latitude, and the county associated with the image. The workflow function also writes this data to an output JSON file with name and location specified by the function's arguments.

#### Dependencies

This script uses the [ArcGIS API for Python](https://developers.arcgis.com/python/), which comes with a number of another dependencies (see `arcgis_requirements.txt`). An [installation guide](https://developers.arcgis.com/python/guide/install-and-set-up/) is available. In addition, a local library is referenced, the shared function file `misc_functions.py`. The `time`, `csv`, and `sys` standard Python libraries are also used.


## <a name='scriptUseAndAccess'></a>Script Use and Access

This project is licensed under the terms of the MIT license. Attribution and linking back to the repository would be appreciated.
