---
layout: post
title:  "Search Futures - Update"
author: Richard Smith
date:   2021-12-07 17:00:00 +0100
categories: search stac indexing
tags: [search, indexing, stac]
---

In July, we [posted our progress so far]() and gave some
intentions for the future. This post looks at the progress since
then and showcases a minimum viable product.

The first change is a bit of re-branding.
The indexing framework is lead by the [asset-scanner]().
This provides the base classes and includes the re-useable
processors, e.g. [regex processor](). The asset scanner also
provides the entry point to the indexing chain
via the command-line command `asset_scanner`.

Three more packages comprise the individual workflows to
convert a stream of assets into content for the STAC catalog.
They have been given the naming convention `*-generator`

## Asset Generator

The [asset generator]() is responsible for extracting
basic file-level information needed for serving files. 
- Location
- Size
- Last Modified

We have tested this system both against traditional disk
filesystems and Object Store using Google Cloud and Amazon S3.

We are also in the process of adding support for property
extraction at the asset level. For example, you might want to 
extract the datetime for a particular file. Although the STAC specification
does not support searching these properties, it might
be benficial for downstream clients to have this information available.

## Item Generator

The [item generator]() forms the glue which brings
together assets and collections. Item IDs are pulled 
from the [item description]() files and given to an item. 
Item IDs are generated, based on the content, and assigned to the relevant assets.

