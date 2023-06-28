---
layout: post
title:  "Experiments with Kerchunk"
author: Daniel Westwood
date:   2023-05-04 15:00:00
tags: [Kerchunk, fsspec]
---

TL;DR Kerchunk stores netcdf chunk information to be used to create virtual xarray datasets. If there are many chunks, there will be many references in the kerchunk file. The number of references can be reduced by formulating a description of the chunks rather than just listing each one, the existing formula syntax is limited so a new custom syntax is explored here

Kerchunk is a library of python tools providing a single uniform method of representing chunked compressed data formats for more efficient cloud access without having to translate file formats. A significant amount of historic data exists in formats not ideal for cloud access (NetCDF/HDF5, GRIB etc.). Kerchunk was created as a way to avoid the lengthy process of converting files to zarr format for cloud optimisation, instead creating json files containing chunking information which can be used to build virtual datasets from multiple files using only certain chunks.

The experiments described here explored ways of reducing and optimising the file sizes and opening times of kerchunk files for a better user experience. Existing attempts to produce kerchunk files for large datasets have been met with the issue of an ever-growing kerchunk file size, thus ways to optimise storage are being explored. 

# Index

- [Index](#index)
- [Kerchunk Background](#kerchunk-background)
- [Generators](#generators)
  - [Limitations of Standard Generators](#limitations-of-standard-generators)
  - [Custom Generator Syntax](#custom-generator-syntax)
- [Performance](#performance)
  - [file Sizes](#file-sizes)
  - [Packing and Unpacking](#packing-and-unpacking)
- [Next Steps](#next-steps)
- [Summary](#summary)
- [Getting in touch](#getting-in-touch)

# Kerchunk Background

The purpose of kerchunk is to provide a method of reading data from traditional file systems like Quobyte, using cloud-based services. Kerchunk provides a unified way to represent chunked data formats of various types and file formats, allowing more efficient access from the cloud. Cloud-optimised file formats like Zarr do exist, but it would be time consuming and computationally intensive to convert all historic data of different formats to Zarr. With kerchunk, the existing files are not altered, instead a representation of the chunk structure of the files is stored in the cloud as a kerchunk file and can be used in conjuction with other python libraries to build a virtual dataset for analysis.

{% include figure.html
    image_url="assets/img/posts/2023-05-04-kerchunk-experiments/workflow.png"
    description="Figure 1: Typical file access and where kerchunk fits into the workflow."
%}

Figure 1 shows the kerchunk workflow which replaces the Zarr/Object Store requirement for cloud optimisation. For files stored in traditional formats, kerchunk uses an fsspec ReferencefileSystem object to build a picture of the chunk structure within the files, and a Kerchunk (json) file  stores minimal chunk information and metadata. It is the kerchunk files which are then used to construct virtual datasets at the application layer, typically using Xarray and Dask for lazy evaluation.

This article details the experiments undertaken to explore ways of further compressing kerchunk files (minimised/generator-based) using __generators__. This workflow addition consists of extra steps for packing and unpacking generators in the form of overriding methods of the Kerchunk construction classes, resulting in smaller kerchunk files stored in the cloud but may result in longer processing times for kerchunk files. It may also be possible to use a technique called __minimal unpacking__ which involves unpacking only the required chunks, which would ideally decrease analysis times for the virtual datasets, but may need further optimisation and considerations of various approaches.

# Generators

The main consideration for these experiments was the use of generators to describe chunk layouts within the files, rather than listing each and every chunk in order in the file, leading to very large kerchunk files and long read times.

Generators act as a description of how to build the chunk arrays that are present in a usual kerchunk file, without having to specify every single chunk in the file. For example, if all chunks in a NetCDF file are the same size and there are no gaps, the generator can specify a start point (offset) for each chunk, the length of each chunk and the number of chunks in a formulaic manner that can be expanded to give the list of chunks.
```
"gen": {
    ...
    "offset": {{1000 + 1000*i}},
    "length": 1000,
    "dimensions":{
        "i":{
            "stop":100
        }
    }
}
```
This generator would build a set of 100 chunks of 1000 bytes each, starting at an offset 1000 bytes and incrementing by one chunk for each chunk. The above represents an ideal case where every chunk is uniform in size and spaced evenly so there are no gaps between chunks. In compressed NetCDF files this is rarely the case, as was discovered during these experiments.

## Limitations of Standard Generators
The current kerchunk library uses jinja2 templates to construct file names based on the iterators provided across the sets of chunks. This works fine for simple repetitive filenames with predictable patterns of years and months but when looking at daily files, things become more complicated. The years and months of monthly files can be expressed as a formula in a jinja2 template as there are always 12 months per year. But there are a differing number of daily files per month, which jinja2 cannot account for in a single template. The standard way of dealing with many file names without having to repeat sections would be to put the filename into the kerchunk file as a template and then use that template in place of the file name.
```
"templates":{
    "template_name": {
        "a": "file1.nc",
        "b": "file2.nc",
        ...
    }
}
```
This works well for small chunking sets, but when dealing with thousands of files, each with thousands of individual chunks, even the relatively simple `{{a}}` style syntax adds to the resulting kerchunk file size significantly, and would complicate things when trying to create appropriate template names for so many files. Using a single character would quickly become insufficient.

Aside from file generation, there is a much larger issue that puts a limit on whether generators can be used at all. The generator is structured as a simple iterator over n chunks in sequence, designed for describing uniform chunks separated by a uniform gap in memory or where chunks can be described formulaically. For most circumstances however, especially when dealing with extensive file sets of large data products, the NetCDF files containing the data are compressed. This alters the chunking structure of each file so the chunks are no longer uniformly sized or spaced. There may be chunks of varying byte-sizes in storage, gaps between the end of one chunk and the start of the next, and chunks that are entirely absent of data as they contain null values that are ignored in compression. 

{% include figure.html
    image_url="assets/img/posts/2023-05-04-kerchunk-experiments/chunk_structure.png"
    description="Figure 2: A realistic representation of chunks within a single NetCDF file laid out in order in storage"
    image_style="default: 'width: 50%; margin-left: auto; margin-right: auto; display: block'"
%}

As shown in figure 2, there are multiple shapes and sizes to the chunks that disrupt any standard regular size that would be present without compression. Since the point of kerchunk is to represent the NetCDF files without having to convert or change existing files, this is the chunk format that must be represented, hence custom syntax is required.

## Custom Generator Syntax
As has been alluded to, the existing syntax for generators is simply insufficient to describe the majority of real-world data that kerchunk would likely be used for. There was therefore a need to develop a new syntax and layout for custom generators to handle the issues in real data presented above. The approaches described here are a first attempt at optimising the generator syntax and are by no means a final solution. There is considerable room for improvement and optimisation, which will be discussed.

The custom generators replace all chunk references in the kerchunk file, which could be millions of entries into the `refs` dictionary. Extracting relevant chunk information is therefore done most efficiently by a single pass of all chunk references, resulting in an O(n) complexity in big O notation. This does require assembling multiple variables to cache the data, which is memory-intensive, but this is far better than having to complete multiple passes of millions of references. The custom generator script takes the existing set of refs and instead represents them in a condensed format using the attributes below:

- __File list__: The chunk offsets and lengths are collected in this pass, as well as the index of the last chunk that occurs in each file. This means that for identifying which chunk belongs to which file, the kerchunk file only stores the name of each file once and the index of the last chunk.
- __Unique chunks__: Collect all chunk sizes, determine the modal chunk size and the array of chunks that differ from that size. The custom generator syntax is to store the `lengths` and `ids` arrays as two sub-items under `unique` key in the resulting dictionary.
- __Skipped chunks__: The generator stores the total number of chunks there should be when unpacking, but some chunks are placed as keys in the `skipchunks` dict. The value associated with the chunk key is an array of 1s and 0s representing for which variables that chunk is missing (land/sea flags mean all 0s if applied to all variables).
- __Chunk gaps__: Determined using the subtraction of two numpy arrays which determines all chunks for where a gap exists before the next chunk in the series. Stored in the same format as __Unique chunks__ with `lengths` of gaps and `ids` of chunks with the preceeding gap. `lengths` of gaps can be negative here, which is a feature to allow for out-of-order chunking and end-of-file changes.
- __Variables__: List of names of all the variables.
- __Varwise__: Boolean flag for chunk layout (varwise for each variable chunk or chunkwise for all chunks of one variable in order)
- __Dims__: The sizes of dimensions that are chunked, the first dimension is always time or per-file chunking, followed by dimensions of any internal netcdf chunking.
- __Dimensions__: Specifies iterators for unpacking the generators (this syntax has been preserved from the original generator syntax.)
- __Gfactor__: This value ranges from 0 to 1 and is calculated by subtracting from 1, the total number of non-uniform (unique or skipped) chunks against the total number of chunks. The ideal case would be zero non-uniform chunks and a gfactor of 1, while the worst case is all chunks being unique, so the gfactor is zero. These cases plus the more typical case of a 0-1 gfactor are shown in figure 3. 

These attributes are used to rebuild the complete chunk list upon usage of the kerchunk file.

# Performance
The kerchunk experiments aim to explore and optimise two main issues with the standard kerchunk files; the size of the kerchunk files and the time taken to write to and read from them. Some kerchunk files can balloon to MB or GB in size depending on the NetCDF internal chunking, which also impacts the read and write times for these files. Ideally both the size of kerchunk files and the time taken to construct and use those files can be reduced significantly using generators. So far with these experiments, the file sizes have been successfully reduced at the cost of a small increase in read times, as there are now added generator packing and unpacking processes that only add to the read/write times. Future improvements would be to consider the library dependencies of kerchunk, how the virtual datasets are assembled from the chunk information and whether those elements can be optimised as well.

## file Sizes

{% include figure.html
    image_url="assets/img/posts/2023-05-04-kerchunk-experiments/chunk_distributions.png"
    description="Figure 3: Pie Charts describing chunk distributions in various test examples, with the best case (left) of all uniform chunks and worst case (right) of all unique chunks"
    image_style="default: 'width: 80%; margin-left: auto; margin-right: auto; display: block'"
%}

Figure 3 shows a pie chart of the distributions of chunks within some test case NetCDF files. Ignoring the worst case scenario where custom generators would not be used, the ESACCI LST file gives a typical case of a non-zero amount of unique and skipped chunks. In this particular instance the gfactor was around 0.4, meaning only 40% of the chunks conformed to a standard size. This factor holds approximately constant the more files are considered in the kerchunk file, and leads to a compression of between 8 and 9 times the original file size, falling below 9 at around 100 files. With only a few files considered however, the compression is less effective as the metadata of the kerchunk file (.zarray and .zattrs) and the base64 encoded major dimensions (lat and lon) are the same size regardless of number of files. This is especially true for NetCDF files that contain no internal chunking, as the list of chunks is much smaller.

{% include figure.html
    image_url="assets/img/posts/2023-05-04-kerchunk-experiments/file_sizes.png"
    description="Figure 4: Linear relation of Kerchunk file increase with number of Netcdf (NetCDF files) for the standard and custom generated cases."
%}

Figure 4 shows the linear increase in file sizes for the kerchunk files with an increasing number of netcdf files (NetCDF files) are considered. The non-compressed metadata contributes around 30 KB file size to each file, with the original files increasing by 1 MB per added NetCDF file, and the generated files increasing by 0.1 MB.


{% include figure.html
    image_url="assets/img/posts/2023-05-04-kerchunk-experiments/reductions.png"
    description="Figure 5: Proportional reduction in size between standard (original) Kerchunk files and custom generated variants."
%}

The observed size reduction across the 50 files considered here remains just below 1 order of magnitude, meaning that for the largest Kerchunk files for the ESACCI LST data at around 6000 MB, the generated Kerchunk file should be around 9 times smaller. This reduction depends on the gfactor of the data, but also the increase per NetCDF file. The BICEP NetCDF files have no internal spatial chunking so the increase per NetCDF file is very small and the size reduction of the Kerchunk files is only around 0.8 times, despite the BICEP files having a very high gfactor.

## Packing and Unpacking
Part of the issue with current kerchunk files is that for large file sets (6000+ in the case of ESACCI LST), the kerchunk files take a long time to construct in the first instance but also reading the kerchunk files to construct virtual datasets takes a long time as well. The latter of these is more impactful as the virtual dataset construction occurs every time a new analysis is executed whereas the kerchunk file is only constructed once.

The custom packing adds some amount of processing time to create the generators and determine chunk structure disruptions. This process has been partially optimised by reducing the number of passes of the chunk references to a single pass and using numpy to assess the chunk sets for standard sizes and disruptions. Tests conducted using the aforementioned ESACCI LST data (gfactor: 0.4) indicated that the packing time scales with number of files considered and adds less than 10% to the construction time of the kerchunk files.

Unpacking the generator has proven considerably more difficult to optimise as each chunk is heavily dependent on the sizes and disruptions of chunks before it. The current method is a simple iteration specified in the generator `dimensions` attribute, which includes a memory offset counter and pseudo-pointers for the unique and gap arrays, incrementing for each gap as the iteration counter passes each chunk id. This method has proven much slower than anticipated, as the read time for a kerchunk file representing 1000 NetCDF files is doubled from 20 to 40 seconds. The read time increases further when attempting minimal unpacking to 1 minute 30 seconds, as the iteration is still required over all chunks - just that they are not all recorded. Clearly there is room for optimisation and improvement of current methods, possibly including numpy array analysis for efficiency or simply reorganising the checks within the iteration loop.

# Next Steps
While there has been considerable extension to the existing generator capabilities that now mean previously unrepresentable datasets can be represented, there are still some limitations to this process.
- __Requirements__ of the NetCDF files being represented; formatting and inclusion of specific dimensions (i.e __time__ and __lat/lon__), possible dependency on regular lat/lon grids.
- __Reference files__; The custom generator code selects the first file to act as a reference for chunk structure but this only works if all files being assessed follow that structure.
- __Other data formats__; The custom generator code has not been tested with other file formats like GRIB, GeoTIFF etc. Adjustments may need to be added for different file structures, especially with manipulating and removing repeated dimensions.

# Summary
The custom generator experiments have proven very successful in extending the capabilities of the existing generator syntax to represent more complicated datasets in an efficient manner, and the kerchunk file compilation times and sizes are far improved over the standard original methods. What is disappointing however is that the virtual dataset construction, which would be the user experience in most cases, is far slower to process than even the original files. It is clear that at the moment the custom generator methods could not be incorporated for all kerchunk files as it would significantly impact users' experience with using kerchunk for cloud purposes, and further developments are needed to explore how to make generator unpacking more efficient.

# Getting in touch
If you would like to discuss this topic or other kerchunk questions, please contact me by [email](daniel.westwood@stfc.ac.uk).