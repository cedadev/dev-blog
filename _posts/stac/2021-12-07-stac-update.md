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

This article assumes you have some background understanding of what we 
are doing. If you have not already, the [background](),[requirements]() and [progress]()
sections of the previous blog will give the background.

We hope to create a full stack solution for other organisations with similar desires, 
including an indexing framework, API server and clients.

# Indexing Framework

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

Item ID generation is what brings related assets together. The item generator pulls
out named facets. Aggregation facets, defined in the item description, tell the system
what constitutes a meaningful blob. All assets from which I can extract these facets, and their
values are the same, are related. 

This approach creates a different problem, discussed later.

## Collection Generator

The [collection generator]() works slightly differently to the other two. Whereas the asset
and item generators are designed to work on a stream of assets, the collection generator
works to summarise the related items.

It is conceivable that this could be run on a schedule, summarising daily or at whatever interval
most fits your use case.

All 3 of the generators work on a similar pattern: 

{% include figure.html 
    image_url="/assets/img/posts/2021-12-07-stac-update/generator_workflow.png" 
    image_style="width: 100%"
    description="Processing workflow for generating content for the STAC API using the CEDA python packages" 
%} 

# STAC API

The second update comes to our API server. We are using [STAC FastAPI]() a community
project building a STAC server on the FastAPI framework. It comes with sample implementations
for Postgres and using SQLAlchemy. We have developed an [Elasticsearch backend]().

Aside from porting the base API to work with Elasticsearch, we have created Elasticsearch
backends for [pygeofilter]() to enable us to provide the [filter extension](). This gives
rich search capability. It also provides `queryables` which describe the facets available for
each collection. Through this mechanism, there is no finer reduction of facets.

One of our desires was free-text search capability. This is not defined in the specification.
Technically, you could probably construct queries using the filter extension which would
satisfy this need but users are familiar with simple querystring syntax. Elasticsearch
provides powerful free-text capabilities, so we have added an extension providing simple
search using the `q` parameter. 

{% include figure.html 
    image_url="/assets/img/posts/2021-12-07-stac-update/free-text-search.png" 
    image_style="width: 100%"
    description="Example queries using the q parameter" 
%} 

This extension has been listed on the [official stac-api-specification] page
and is available for all to use.

Faceted search is high on our priority list and we have been trying out solutions.
The `/collections/<id>/queryables` endpoint gives collection level facets. The
global `/queryables` endpoint give the intersect of all available facets. There is 
difficulty in providing these values for a heterogeneous archive as intersecting >2

