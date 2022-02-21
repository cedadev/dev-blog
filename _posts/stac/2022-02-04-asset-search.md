---
layout: post
title:  "Asset Specification and Asset Search"
author: Richard Smith
date:   2022-02-04 17:00:00
tags: [stac, search]
---

The STAC specification nests `Assets` (downloadable objects) within an `Item` (group of related Assets).
In the CEDA archive, an Item could have 1000s of assets. How do we represent and retrieve Items with large assets counts?
Enabling paginated Asset lists, Item sub-setting and cross-Item Asset search.

## Index

* [Background](#background)
* [Proposal](#proposal)
* [Outcome](#outcome)

## Background

Assets are nested objects as part of the [item-spec](https://github.com/radiantearth/stac-spec/blob/master/item-spec/item-spec.md). 
This provides easy access to the related assets in the context of their item. We interpret an item to be a collection of assets with 
the same set of properties which are intended to be consumed together. For example:
- A satellite scene
- An aerial image
- A research flight
- A single variable generated a single realisation of a model run
- A time-series product (could be processed satellite data, climate model run, etc.)

Assets for an Item consist of metadata (e.g. thumbnails, ISO19115 records) and data files.

The current approach, nesting these Assets in the Item response works well when:
- The number of Assets is low
- The time period specified is small (when many files span the time period)

Conversely, when the Asset count is high (time-series) and users might want to select a subset, 
the **cost of returning the Item object is high**. Extreme examples include:

ECMWF-ERA Datasets 2.5 Degree gridded product is comprised of 133,000 files:
- decadal `(23,000)` / yearly `(2,920)` / monthly `(248)` / daily `(8)`

CCI Sea Level L4:
- Full Dataset `(553)` / decadal `(120)` / yearly `(12)`

An argument could be made to split this into multiple Items using perhaps, decadal/yearly/monthly/daily components, 
whichever results in a reasonable number of Assets, but this optimises to fit the standard and make it 
easier for the catalog builder, not the user. In fact, there are _two problems_ with this approach:
 1. It can lead to duplication across the catalog, where the same data is described in Items that aggregate at different
    time periods.
 2. The user is _very likely_ to want to search the full time-series as a _single entity_. Splitting it into subsets
    is an artefact of the cataloguing process that makes extra work for end-users - in which they have to aggregate 
    multiple Items themselves.

As a user searching the catalog, it makes sense to return a homogeneous chunk (Item) and have the Assets attached to this object.

Assuming we decide to capture all Assets in a single Item, we need a clean solution for:
- Handling Items with many Assets (e.g. >> 100)
- Sub-setting along time - i.e. filtering Assets by time selection

The jump from Collections to Items is done via a _link_ item, using relation `item`.
The `/collections/<collectionId>/items` path provides a paginated list of Items. 
It follows that Assets could be represented in the same way. i.e. `/collections/<collectionId>/items/<itemId>/assets`

### An aside: aggregation summaries of Assets

In a perfect future, we won't need to worry about Assets because data services will exist to handle Assets
transparently so that the user query/request gets converted into a workflow description and returns only the 
data payload requested.

In the world of STAC Items, we can expect that service endpoints will be included in the Item record, as
_aggregated assets_. There are some example aggregation formats that might be relevant in this area:
- [kerchunk](https://pypi.org/project/kerchunk/) - descriptive files representing an access point for reading multiple data files (Assets)
- [Zarr](https://zarr.readthedocs.io/) - a cloud-compatible format that separates data access and serialisation for multi-dimensional arrays

But that's the future - let's get back to the our proposal...

## Proposal

In order to support Assets as _first-class objects_ in STAC, we propose the following extensions:
1. `SPEC EXTENSION` Add a link relation "asset" to allow linkage to a separate Asset object.
2. `SPEC EXTENSION` Add an [asset-spec](https://github.com/cedadev/stac-asset-spec) which defines the Asset representation. This follows closely the [item-spec](https://github.com/radiantearth/stac-spec/blob/master/item-spec/item-spec.md), although properties are preferred in the Item. Properties are allowed as per the current asset-spec that can be used for subsetting (e.g. datetime)
3. `API EXTENSION` Add [asset-list](https://github.com/cedadev/stac-asset-list/blob/main/README.md) to provide `/collections/<collectionId>/items/<itemId>/assets` which is the list of Assets for a given itemId. This will provide a paginated list of all Assets and is linked to using a link object as part of an Item with relation "asset" .
4. `API EXTENSION` Add [asset-search](https://github.com/cedadev/stac-asset-search) to allow cross-item searching of individual data files
   - `API EXTENSION` Asset search enables API based sub-setting, using the `/collections/<collectionId>/items/<itemId>/assets` URL. e.g.`/collections/<collectionId>/items/<itemId>/assets?datatime=start/end`

## Schemas

Our proposed schemas are available here:
- [Asset Spec](https://github.com/cedadev/stac-asset-spec)
- [Asset Search](https://github.com/cedadev/stac-asset-search)

## Outcome

Moving the potentially long list of Assets to a paginated endpoint allows clients to selectively 
retrieve Assets for Items which have been specifically requested, rather than for all Assets irrespective 
of their need. This can be represented as a paginated table in user interfaces. Presenting the metadata 
Assets within the Item record is still useful as it allows the user to dig deeper and understand whether 
this is the Item they require. 

_Asset search_ allows for filtering, either within an item (e.g date-time or role to retrieve only data Assets) 
or cross-Item selection, where a subset of related Assets from different Items might be desired. In the 
first case, all Assets point back to a single Item record. In the latter case, Assets may point back to a 
number of different Items (potentially in different Collections).

The power of Asset-level search is that the user can search dimensions of the "data hypercube" that 
yields a true intersect of relevant content, rather than being constrained by the limitations of how items 
are defined in a particular STAC catalog. A single search gets the results that are otherwise gained from 
repeated search-and-filter operations, thereby streamlining the discovery process.

Whilst many catalog managers, and scientific domains, may not need Asset-level search, it is a powerful
solution for managing large archive or heterogeneous environmental data. We intend to implement it in our
emerging STAC solution for CEDA. We also expect it to meet the needs of the next generation of Earth 
System Grid Federation (ESGF) Search system in the near future.

