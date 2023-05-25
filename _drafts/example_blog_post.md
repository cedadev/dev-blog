---
layout: post
title:  "Migrating CEDA Documents Repository to the Zenodo"
author: Adrian Dębski, Graham Parton
date:   2023-05-25 13:00:00
tags: [migration, CEDA Docs, Zenodo]
---

The first paragraph of the blog post is really important. It is shown on the home page
alongside the title and should give browsing users enough information about the article
to know whether they want to read it or not. It should be short and give an over view 
of what you are going to talk about.

# Index

* [Background](#background)
* [Process of migration](#process-of-migration)
* [The CEDA Document Repository](#the-ceda-document-repository)
* [Overview of the migration process](#overview-of-the-migration-process) 
* [Getting content from CEDA Docs using Python](#getting-content-from-ceda-docs-using-python)
* [Mapping metadata](#mapping-metadata)
* [Uploading to Zenodo](#uploading-to-zenodo)
* [Handling versioning](#handling-versioning)
* [Major problems](#major-problems)
* [Measuring success](#measuring-success)
* [Post migration work](#post-migration-work)
* [References](#references)

# Background

At the Centre for Environmental Data Analysis (CEDA) a wide range of services are provided to aid the research community to find, access, process and analyse environmental data. Ensuring these remain secure, reliable and robust to support CEDA’s growing user community entails dedicated staff effort and resources, alongside periodic reviews to identify vulnerabilities and continued relevance of services. These last two aspects of service maintenance may lead to the decision to retire services, a process that requires careful consideration to ensure minimal impact to users and overall delivery of services. 

One such service CEDA has operated over many years is a document repository, an instance of an ‘EPrints’ repository as a store of ‘grey literature’ items relevant to the work and services of CEDA and its user community. Over the past few months this is one service that has completed its lifecycle, a process which highlights some salient points around the transition to the end of service delivery. More specifically, it may aid other operators of similar ‘EPrints’ based services also wishing to follow similar transitions. 

# Process of migration

## The CEDA Document Repository
CEDA originally set up the CEDA Document Repository to act as a ‘grey literature’ store to support CEDA’s community, to aid access to items that relate to, and aid the use of, data in the CEDA archives. Such content would be wide ranging from research aircraft flight logs and instrument calibration details through to minutes from meetings, poster presentations and annual reports.  Launched on 30th October 2008 as one of the outputs of the “Overlay Journal Infrastructure for Meteorological Sciences (OJIMS) project” (Callaghan et al. 2009), it was initially funded by the project for its first year with “the principal expenditure devoted to moderating the deposit of new items into the repository.”  and as “the costs of running the repository within the BADC [now CEDA] in the long term were not found to be prohibitive. Hence the repository will be maintained for the foreseeable future now that the OJIMS Project has ended.” 

Whilst the cost of operating the system remains sustainable, and there is a need for such a service to curate such grey literature items, the intervening 14 years has seen an evolution of the surrounding repository landscape. Institutional repositories are now commonplace alongside generic services such as FigShare and Zenodo. As such, CEDA was able to conduct a review of the existing EPrints service run on a dedicated server within the CEDA infrastructure. This review highlighted the ongoing overheads needed to keep this service secure and the diminished need for such a service to remain in-house indicated that alternative options should be explored.  

One option examined was the Zenodo repository service which is operated by CERN which has a broad remit that would be meet the grey literature requirements offered by the CEDA Document Repository. This service also has ‘communities’ to curate collections of related items within Zenodo. Additionally, it offers the advantages of improved item version control and provision of DOIs for items as well as scope for inclusion of code from services such as Github. This provided CEDA with a viable alternative solution to a grey literature store, though Zenodo would not be deemed suitable as an archive for data produced from efforts funded by the Natural Environment Research Council (NERC) [mainly due to the lack of ISO19115 metadata fields supporting discovery of geospatial data and reduced ability to have the item access required to manage content for the long-term]. 

In terms of overheads Zenodo would present very similar levels of effort with regards to content moderation for items submitted to be included within a CEDA ‘community’ within Zenodo compared to editorial responsibilities for the CEDA Eprints service. However, CEDA would not have to maintain the underlying service itself, including user management. 

Consequentially, in Summer 2022 it was decided to begin the process of porting content from the CEDA operated EPrints document repository to a new CEDA Document Repository Zenodo Community. This main effort to enact this transfer centred around mapping metadata fields from the EPrints to Zenodo and addressing various content issues that were identified along the way. These aspects are explored in the following sections. 

## Overview of the migration process 
To ensure a smooth migration of content to allow the closure of the CEDA EPrints service three significant parts needed to be concluded: 

* Finding a way to download access content from the EPrints service via Python 

* Mapping CEDA Docs metadata into Zenodo format  

* Uploading files and metadata to Zenodo 

As will be detailed below, whilst appearing to be straightforward a range of issues arose from each of stage. 

## Getting content from CEDA Docs using Python 
Before embarking on the migration process differences in item metadata were known between the CEDA EPrints repository and the Zenodo schemas. However, in order to establish mappings between the two metadata schemas, the nature of the content also of the metadata fields also needed to be examined to further understand the metadata to be handled. Additionally, gaining programmatic access to the content of the CEDA Eprints service would allow progress to be tracked to verify the success of the whole process. 

Following an initial server-side exploration of the structure of the EPrints holdings to determine how deposited content was stored avenues to harvest item metadata were then explored, preferably with a Python based solution. Initially the EPrints OAI-PMH's interface was explored, allowing basic record metadata to be harvested, but not all the desired content from the records. However, examining individual records showed that there existed various endpoints for each record to obtain the metadata in various representations, including a JSON representation. Whilst it was known that there were 1250 deposited items within the EPrints collection the actual record IDs were not known initially (many IDs have been used for items that had been withdrawn from submission over time, so didn’t resolve to any curated content), though the maximum record ID (1492) was known. A brute-force approach could have been used to test each possible URL for to all IDs up to the maximum number, but this solution was neither elegant nor accurate with some IDs found to be missed. Instead, the service’s OAI 2.0 endpoint was used which returned XML representation of the whole repository (divided into parts, 100 records per page). This provided a mechanism to obtain a list of all valid record IDs. 

## Mapping metadata 
With a full list of valid record IDs enabling URLs for their JSON endpoints to be easily constructed the next stage was to review the structure and contact of the metadata themselves to enable mapping to the Zenodo schema to be established. To aid this exploration phase a Python notebook was prepared allowing rapid content harvesting and filtering. Whilst the notebook was used quite dynamically t 
 
There were about 70 unique attributes which needed to be mapped to the corresponding Zenodo fields, which readily fell into a few major categories. The first were where direct mappings between the EPrints and Zenodo schemas were readily established thanks to their identical or near-identically named fields and scope (e.g. notes, title, abstract, creators). The second category was those fields where direct mapping wasn’t obvious, but a similar field within Zenodo was identified that had sufficient scope. In this second case the mapping was more complicated, often requiring cross-referencing between various fields to determine the mapping to be used (e.g. type: “monograph” and monograph_type = “book” lead to an upload_type of “publication” and publication_type = “book”; type: “conference_item” and pres_type = “lecture” resulting in an upload_type: “publication” and publication_type: “conferencepaper”).  

Finally, as the EPrints schema is more extensive then the presently available Zenodo schema there were cases where no obvious target field existed, yet there was a need to preserve the source metadata. In this latter case the content from these fields was mapped to the generic ‘Additional Notes’ attribute within the Zenodo schema (e.g. output_media, which defines the type of media the original data was provided via; funders, as there's no ‘funder’ subtype for contributors in Zenodo). 

 
Additionally, whilst exploring the content a range of source metadata content issues were identified that needed to be resolved either at source (by removing or amending erroneous or irrelevant content) or as part of the metadata pipeline (see ‘Major Issues’ below).  

To complete the mapping exercise new metadata requirements were also established, also stored within the ‘additional notes’ section within Zenodo: 

* The EPrints item URL would be added, partly as providence information but also to enable a redirection service to be set up for the old EPrints. I.e. when a user requested an old CEDA Document Repository URL (e.g. http://cedadocs.ceda.ac.uk/1492/) it would then return a URL to a record within Zenodo (https://zenodo.org/record/7357335 in this case). 

* A note regarding the transfer to Zenodo (including date of transfer) was added as part of the Zenodo item’s providence 

The full mapping used can be found in the code based used for the migration process available at https://doi.org/10.5281/zenodo.7611795 (Debski and Parton, 2023). 

## Uploading to Zenodo 
Having established the pipelines needed to harvest item metadata and content from our EPrints system the next stage was to prepare the upload pipeline to Zenodo. This centred around the Zenodo Rest API which has well detailed, clear documentation (link: https://developers.zenodo.org/), alongside a sandbox service to enable such development. 
 
During the process development and testing a wide range of issues were encountered. One of the most common was a hitting API request limits (I.e. a 429 status code). Unfortunately, Zenodo does not provide the maximal allowed number of requests in the response message, therefore it took several attempts to establish a rate threshold that would avoid being blocked by Zenodo, but still achieve tasks to enable testing and verification in a reasonable timeframe. (Additionally, to aid tracking and follow-up actions after migration the final implementation script needed to permit sufficient time for Digital Object Identifier (DOI) to be minted for the uploaded content). 
 
With the exclusion of one large deposit (which contained almost 6,000 files!), and based on an initial sample of 100 records, the time for a full migration was an estimated 3 1/2 hours. The initial sample break down of times for each step was: 

* 1s to process the requests to create an upload folder 

* 0.83s for uploading a single file 

* 1.48s files per metadata record 

* 1s for DOI minting 

* Plus, all the necessary sleep times for each stage/between each record 

A full migration took 4 hours 20 minutes (excluding the outlier case highlighted above). The difference was due to the relatively large size of some of records, underestimation of the time needed to publish a record and connection issues and server-side delays. 

## Handling versioning 
One choice that needed to be made whilst setting up the migrations was how to handle items in the CEDA Document Repository that had been chained together as different versions. Within the Eprints instance that was our source, each version was its own well-defined deposit with its own metadata record and deposited content, but with links provided between them. At the same time within the Zenodo system there were two possible approaches to managing such versioning: either to have one Zenodo item under which different versions of the content were available, but essentially the same metadata record; or, to have individual deposited items connected via the ‘related identifiers’ attribute with the ‘newer version’ and/or ‘previous version’ relationship used. The latter of these two options was chosen given that the source items had originally been individual, standalone items with their own URLs and metadata entries to preserve their distinction, only requiring post-migration linking to be added between the different records.

## Major problems 
Within the EPrints document repository service there had been a number of fields where free text entries had been permitted leading to some pre-processing before migration to Zenodo. These included: 

* Keywords: Zenodo stored these as a list of string items – one for each keyword/phrase, but the Eprints source only had these as a single string with no consistent separator. Whilst Python allows you to split the string by multiple separators many were simple white spaces separated, but this would lead to individual words and not whole key phases. Consequentially, a dictionary was established to provide the key phrase mappings needed to translate the EPrints keywords over to individual Zenodo keyword entries. 

* Subject to keywords: The CEDA Document Repositor EPrints service made use of subject classification scheme utilised also by the NERC NORA document repository (https://nora.nerc.ac.uk/), a NERC specific subject classification scheme. However, some of these subjects were also found to map to concepts within the Library of Congress subject classifications. Making use of these permitted entries to be added in the ‘subjects’ attribute for the Zenodo items, where a URL to a code list item was required in addition to the value. The advantage of doing this would be to enable items with such tagging to also be found in subject based searches within Zenodo, thus providing enhanced discoverability. To support this a mapping was established for 17 of the 23 EPrints subjects and the Library of Congress end points. The remaining 6 were added as normal keywords within Zenodo. 

* type of deposition: within Zenodo there is one main ‘type’ attribute which may result in an additional one of two extra fields needing populating to describe the sub-type of object. However, the EPrintts service provided 4 fields to define the sub-type of the record with no clear mapping to the Zenodo options. To address this the source EPrints options were reviewed and a mapping established based on record IDs.  

* Official URLs: before migrating these were tested to verify that they were correct and found that most were redirects or broken links, though not all redirects were to relevant resources (e.g. resolving to generic site pages for broken links). After reviewing the various types of link resolutions and, where available, the suggested redirect end point a manual review was conducted for all the URLs to establish a reviewed (and amended) mapping to alternative URLs for the migration process based on the authors’ knowledge of the domain. 

* In a few exceptional cases there were a small handful of records that had one or two extra metadata fields containing content, but they were deemed to be of little actual value and so were not mapped to Zenodo (e.g. rev_number – an internal revision number in EPrints, inpublished, num_pieces). 

* One record, a scraping of an old website for the FAAM aircradt, highlighted above, contained over 6000 files within the deposit. This itself caused issues with the migration and had to be handled separately. It consisted of a full scraping of a website but on examination around half the content needed to be removed before migration. Additionally, a separate migration attempt for this one item hit Zenodo limits on the number of individual files that could be added to a Zenodo item, leading to various issues with the service. Subsequently, a solution to tar the remaining items together was found to work, which also lead to improved file listing in the preview view for the item (see https://doi.org/10.5281/zenodo.7415563). 

The work required to review the actual content and either manually adjusting the source content (to correct for identified issues/adjust incorrect selections) and preparing the mappings used within the code was the most time-consuming stage in the migration process. 

## Measuring success 
The goal of this exercise was for as full a migration as possible to enable the closure of the CEDA Document Repository EPrints service. Success of this operation was measured by tracking the process a at all stages through the migration, capturing any errors that were encountered and following up with any adjustments to the processes to address these issues. 

With the exception of the FAAM website record highlighted above needing its own migration run, all remaining 1249 items were successfully migrated over to Zenodo with all metadata content. The additional run to also migrate the FAAM website record was also successfully completed.  

## Post migration work 

Following successful migration of content from the CEDA Document Repository service to the CEDA community on Zenodo a number of follow-up actions were undertaken to further aid usability and maintainability of items. 

Firstly, a re-direction service was set up within the CEDA system as explained above. This was to assist any previous citations of items in the CEDA Document Repository that may be used by members of the user community to ensure that such users would find the new Zenodo location. 
 

Secondly, any known CEDA links to CEDA document repository items (e.g. within the CEDA data catalogue pages) were updated to the DOI reference of the items to aid long-term linking. 
 

Thirdly, contact was attempted with the original depositors of items to offer to transfer the items now in Zenodo associated with the CEDA account to them for onward maintenance. Only a handful of original depositors came forward to request these transfers, which Zenodo were able to do with little issue. 

New items can now be uploaded to Zenodo and submitted for acceptance within the CEDA Document Repository Zenodo Community. 

Finally, the EPrints service and server were retired fully, marking the end of the successful migration of this service.  

## References 

Sarah Callaghan, Sam Pepler, Fiona Hewer, Paul Hardaker, Alan Gadian (2009). How to publish data using overlay journals: the OJIMS project. Ariadne, Issue 61. http://www.ariadne.ac.uk/issue61/callaghan-et-al/ 
Adrian Dębski, & Graham Parton. (2023). cedadev/cedadocs_migrate: v1.0.0 (Initial). Zenodo. https://doi.org/10.5281/zenodo.7611795 
