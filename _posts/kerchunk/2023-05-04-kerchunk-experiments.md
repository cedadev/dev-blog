---
layout: post
title:  "Experiments with Kerchunk"
author: Daniel Westwood
date:   2023-05-04 15:00:00
tags: [Kerchunk, fsspec]
---

Kerchunk is a library of python tools providing a single uniform method of representing chunked compressed data formats for more efficient cloud access without having to translate file formats. A significant amount of historic data exists in formats not ideal for cloud access (NetCDF/HDF5, GRIB etc.). Kerchunk was created as a way to avoid the lengthy process of converting files to zarr format for cloud optimisation, instead creating json files containing chunking information which can be used to build virtual datasets from multiple files using only certain chunks.

The experiments described here explored ways of reducing and optimising the file sizes and opening times of kerchunk files for a better user experience. Existing attempts to produce kerchunk files for large datasets have been met with the issue of an ever-growing kerchunk file size, thus ways to optimise storage are being explored. 

# Index

- [Kerchunk Background](#kerchunk-background)
- [Generators](#the-middle-bit)
  - [Limitations of Standard Generators](#standard-limitations)
  - [Custom Generator Syntax](#custom-generator-syntax)
- [Performance](#performance)
  - [file Sizes](#file-sizes)
  - [Packing and Unpacking](#packing-and-unpacking)
- [Next Steps](#next-steps)
- [Summary](#summary)

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

The custom generators replace all chunk references in the kerchunk file, which could be millions of entries into the `refs` dictionary. Extracting relevant chunk information is therefore done most efficiently by a single pass of all chunk references, resulting in an O(n) complexity in big O notation. This does require assembling multiple variables to cache the data, which is memory-intensive, but this is far better than having to complete multiple passes of millions of references. The chunk offsets and lengths are collected in this pass, as well as the index of the last chunk that occurs in each file. This means that for identifying which chunk belongs to which file, the kerchunk file only stores the name of each file once and the index of the last chunk.

### Unique Sized Chunks
First of the chunk structure disruption issues to be addressed is the varying sizes of chunks. The collected chunk offsets and lengths are converted to numpy arrays for easy handling, and the most common (modal) chunk size is calculated using scipy. This modal value is used to identify all chunks with an irregular size, to be stored in the `unique` attribute of the generator. This attribute contains the two sub attributes `lengths` and `ids` respectively storing the chunk sizes and indexes of unique chunks. Storing these as arrays rather than values of separate attributes like in normal kerchunk files means the values can be analysed immediately rather than having to collect each value from each chunk.

### Skipped Chunks
Secondly, certain chunks are skipped as they represent 0 bytes in storage. This is usually due to masks applied to the NetCDF file resulting in chunks containing all null values that are compressed to zero. These chunks are not predictable formulaically so must each be stored. This could be in the form of an array of chunk keys to ignore when unpacking the generator, but a more efficient method is to store the chunk coordinates of a missed chunk as a key in a `skipchunks` dictionary, where the value is a boolean array describing which chunks are skipped for each variable. In the case of a land sea mask to all variables, the boolean array would be true for all variables, but in some cases there may be different masks for different variables. In this format, the key is stored only once for a missed chunk (i.e `1.3.7`) and the boolean array is the smallest way to represent which variables the skipped chunk applies to.

### Gaps between Chunks
In some of the NetCDF files considered here and especially with larger files there are gaps between chunks where one chunk ends and the next one should start. This may be due to headers in the NetCDF file taking up some space or for other reasons. Regardless, when representing the chunks in a generator, these gaps must be taken into account in order to reflect the real positions of chunks in the files. They are dealt with similarly to the unique chunks, with a generator attribute `gaps` and sub attributes `lengths` and `ids` where this time the `id` is the id of a chunk with a preceeding gap of size `length`. The gaps are again determined by assessing the numpy arrays of offsets and lengths and finding discrepancies when adding lengths to the offsets that don't then line up with the next offset position.
One important feature to note that initially presented as a possible issue is where the gap size between chunks appears negative. This was initially thought to be an overflow issue, but is actually just where a chunk is either out of order in a file, or represents the end of a file and the start of the next. The latter of these could be corrected by adding in a reset to where the chunks per file are collected in the first pass of the references, but this may actually extend the generator construction time more than would be gained by removing these negative gap sizes. It is also useful in the case of out-of-order chunks to be represented in this way without having to add methods to check and remove negative gap sizes. Hence the negative gap sizes remain as an unexpected but useful feature.

### All other metadata
As well as the large scale arrays for storing various chunk structure disruption information, a few other key attributes are needed to reconstruct the set of all chunks on unpacking the generator. The file array mentioned before is an array of tuple pairs of filename and last chunk index, so during the unpacking, the chunks are assigned the correct file. There is also an array storing the names of variables represented by the generator, as some NetCDF files contain variables that are chunked in different ways. Currently the generator expects chunks to be stored __variable-wise__ (i.e every chunk at one coordinate, then every chunk at the next), rather than __chunk-wise__ (i.e every chunk for one variable, then every chunk for the next). The dimensions of the chunk structure are stored for ease of unpacking, as well as the initial offset (although this could be absorbed by the gap array for ease of use), and the dimensions of the iterator used in unpacking the generator, the format of which has been preserved from the old syntax. This could be simplified to a single number if necessary as the custom generator syntax doesn't expect iterators with step values or any extra complexities beyond the single iteration loop. Finally as a method of assessing the success of the custom generators, an attribute called `gfactor` has been included. This value ranges from 0 to 1 and is calculated by subtracting from 1, the total number of non-uniform (unique or skipped) chunks against the total number of chunks. The ideal case would be zero non-uniform chunks and a gfactor of 1, while the worst case is all chunks being unique, so the gfactor is zero. These cases plus the more typical case of a 0-1 gfactor are shown below. 

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

The custom packing adds some amount of processing time to create the generators and determine chunk structure disruptions. This process has been partially optimised by reducing the number of passes of the chunk references to a single pass and using numpy to assess the chunk sets for standard sizes and disruptions. Tests conducted using the aforementioned ESACCI LST data (gfactor: 0.4) indicated that the packing time scales with number of files considers and adds less than 10% to the construction time of the kerchunk files.

Unpacking the generator has proven considerably more difficult to optimise as each chunk is heavily dependent on the sizes and disruptions of chunks before it. The current method is a simple iteration specified in the generator `dimensions` attribute, which includes a memory offset counter and pseudo-pointers for the unique and gap arrays, incrementing for each gap as the iteration counter passes each chunk id. This method has proven much slower than anticipated, as the read time for a kerchunk file representing 1000 NetCDF files is doubled from 20 to 40 seconds. The read time increases further when attempting minimal unpacking to 1 minute 30 seconds, as the iteration is still required over all chunks - just that they are not all recorded. Clearly there is room for optimisation and improvement of current methods, possibly including numpy array analysis for efficiency or simply reorganising the checks within the iteration loop.

# Next Steps
While there has been considerable extension to the existing generator capabilities that now mean previously unrepresentable datasets can be represented, there are still some limitations to this process.
Firstly, there are some requirements of the NetCDF files being representing, specifically the formatting and inclusion of specific dimensions. The test case NetCDF files used in these experiments all had regular lat/lon grids with consistently shaped dimensions, i.e a square/cube shape with no data missed. This does not mean data which does not fall into this category cannot be represented, but it is unclear whether there would be issues and adjustments required to the scripts to handle more interesting data structures.

There is a method of the custom class for packing together the chunks into a generator that looks at a reference file, that being by default the first file in the list provided. From the reference file, the overall chunk structure and variables that follow that structure are retrieved. In some cases NetCDF files may have multiple chunking methods for different variables, which is not currently covered here. Also the method attempts to retrieve specific parameters from the NetCDF file like __time__ and __lat/lon__ variables to use in calculating dimensions. If these are not present, the custom generator aspect will be halted as some essential attributes will be unknown. Further development into handling different NetCDF internal formatting will be required to handle these sorts of teething issues.

Beyond the cases of using NetCDF files, the custom generators may be unable to handle other dataset types at present as there has been no testing with other file formats like GRIB, GeoTIFF or any other kind. The custom generator scripts are only known to work with a handful of test case data, so further experimentation and testing is required to see if any adjustments need to be made for different file structures.

# Summary
The custom generator experiments have proven very successful in extending the capabilities of the existing generator syntax to represent more complicated datasets in an efficient manner, and the kerchunk file compilation times and sizes are far improved over the standard original methods. What is disappointing however is that the virtual dataset construction, which would be the user experience in most cases, is far slower to process than even the original files. It is clear that at the moment the custom generator methods could not be incorporated for all kerchunk files as it would significantly impact users' experience with using kerchunk for cloud purposes, and further developments are needed to explore how to make generator unpacking more efficient.
