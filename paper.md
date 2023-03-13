---
title: "gesel: a Javascript package for client-side gene set enrichment"
tags:
  - Javascript
  - bioinformatics
authors:
  - name: Aaron T. L. Lun
    orcid: 0000-0002-3564-4813
    affiliation: 1
affiliations:
  - name: Genentech Inc., South San Francisco, USA
    index: 1
date: 12 March 2023
bibliography: ref.bib
---

# Summary

`gesel` is a Javascript package for performing gene set enrichment analyses within the browser. 
Developers can incorporate `gesel` into their web applications to provide gene set enrichment capabilities to their user experience.
Importantly, `gesel` operates fully within the user's device, i.e., the client; there is no need for a dedicated backend server to perfrom the computation.
This eliminates concerns around cost, scalability, latency, and data ownership that are associated with a backend architecture.
We demonstrate the use of `gesel` with a basic web application that perform enrichment analyses on user-supplied genes with sets derived from the Gene Ontology and MSigDB.

# Statement of need

Gene set enrichment analyses (GSEA) are commonly used to interpret the biological activity of a user-supplied list of interesting genes [@subramanian2005gene]. 
Briefly, this task involves quantifying the enrichment of each reference gene set's members inside the user-supplied list,
where the reference sets are derived from a variety of sources such as previous experimental studies or _de novo_ computational analyses.
GSEA allows scientists to summarize a large list of gene identifiers into a tangible biological concept such as "syntaxin binding" or "T cell receptor signaling pathway".
User-supplied lists are most typically derived from differential expression analyses of transcriptome-wide assays like RNA sequencing,
but any list of genes can be used here, e.g., cluster-specific marker lists from single-cell RNA sequencing studies.

Given the popularity of GSEA in transcriptomics, it is not surprising that many statistical methods and implementations are already available to perform this analysis.
Most existing GSEA tools operate inside frameworks like R/Bioconductor [@young2010gene; @wu2012camera; @korotkevich2021fast] and require both installation of software and associated programming knowledge to use.
Web applications like Enrichr and GeneTrail [@chen2013enrichr; @backes2007genetrail] provide more user-friendly interfaces that require less prior computational knowledge, targeted to the majority of bench scientists.
These applications use a conventional backend architecture where the browser sends a request containing the user-supplied list of genes to a backend server;
the backend then performs the analysis and returns the results to the user's device (i.e., the client) for inspection.

While common, this backend-based architecture is subject to a number of concerns around cost, scalability, latency, and data ownership.
The application maintainer is responsible for provisioning, deploying, monitoring and maintaining a backend server, which requires both money and time.
The maintainer is also responsible for scaling up the backend compute in response to increased usage, further increasing costs in an unpredictable manner. 
The user-supplied lists need to be transferred to the backend and the results need to be transferred back, introducing latency to the user experience.
Finally, the fact that the user's inputs are accessible to the backend introduces potential issues of data ownership, e.g., for confidential biomarker lists or signatures.

Here, we present `gesel` (https://npmjs.com/package/gesel), a Javascript library for gene set enrichment analyses that operates fully inside the client.
Web applications can easily incorporate `gesel` via the standard `npm` installation process, enabling developers to create user-friendly interfaces for GSEA in different contexts.
The browser will then handle all GSEA-related computation within these applications, eliminating the responsibility of maintaining a backend and avoiding any transfer of user data.
This obviates the problems associated with a backend architecture and allows the application to scale naturally to any number of user devices.
We demonstrate the use of `gesel` by creating a simple web application (https://ltla.github.io/gesel-app) for identifying interesting gene sets based on overlaps with user-supplied lists.

# Usage

`gesel`'s analysis involves testing for significant overlap between each reference gene set and the user-supplied list of genes.
While this is the simplest form of GSEA, it is fast, intuitive, mostly effective and avoids the need for users to specify a ranking across the supplied genes.
The algorithm can also be phrased as a search for the gene sets that contain at least one entry of the user-supplied list. 
To demonstrate, consider the following list of gene symbols mixed with Ensembl and Entrez identifiers.

```js
let user_supplied = [ "SNAP25", "NEUROD6", "ENSG00000123307", "1122" ];
```

Our first task is to map these user-supplied gene identifiers to `gesel`'s internal identifiers.
In this case, we are interested in human gene sets, hence the taxonomy ID in the `searchGenes()` call.

```js
let input_mapped = await gesel.searchGenes("9606", user_supplied);
console.log(input_mapped);
// [ [ 4639 ], [ 12767 ], [ 12577 ], [ 828 ] ]
```

To simplify matters, we will ony use the first matching `gesel` gene identifier for each user-supplied gene.
Other applications may prefer to handle multi-mapping genes by, e.g., throwing an error to require clarification from the user.

```js
let input_list = [];
for (const x of input_mapped) {
    if (x.length >= 1) {
        input_list.push(x[0]);
    }
}
console.log(input_list);
// [ 4639, 12767, 12577, 828 ]
```

We call `findOverlappingSets()` to search for all human gene sets that overlap the user-supplied list.
This returns an array of objects with the set identifier, the number of overlapping genes, the size of each set and the enrichment p-value based on the hypergeometric distribution. 
Applications can sort this array by the p-value to prioritize sets with significant overlap.

```js
let overlaps = await gesel.findOverlappingSets("9606", input_list);
console.log(overlaps.length);
// 935

console.log(overlaps[0]);
// { id: 379, count: 1, size: 10, pvalue: 0.0009525509051785397 }
```

Given a set identifier, we obtain that set's details with the `fetchSingleSet()` function.

```js
let set_details = await gesel.fetchSingleSet("9606", overlaps[0].id);
console.log(set_details);
// {
//   name: 'GO:0001504',
//   description: 'neurotransmitter uptake',
//   size: 10,
//   collection: 0,
//   number: 379
// }
```

The same approach can also be used to obtain the details of the collection containing that set.

```js
let parent_collection = await gesel.fetchSingleCollection("9606", set_details.collection);
console.log(parent_collection);
// {
//   title: 'Gene ontology',
//   description: 'Gene sets defined from the Gene Ontology (version 2022-07-01), sourced from the Bioconductor package org.Hs.eg.db 3.16.0.',
//   species: '9606',
//   maintainer: 'Aaron Lun',
//   source: 'https://github.com/LTLA/gesel-feedstock/blob/gene-ontology-v1.0.0/go/build.R',
//   start: 0,
//   size: 18933
// }
```

The membership of each set is obtained with the `fetchGenesForSet()` function.
This returns an array of `gesel`'s internal gene identifiers, which can be mapped to various standard identifiers or symbols using the `fetchAllGenes()` function.

```js
let set_members = await gesel.fetchGenesForSet("9606", overlaps[0].id);
console.log(set_members);
// Uint32Array(10) [
//     343, 1452, 2222,
//    4543, 4547, 4548,
//    4639, 6238, 6246,
//   14046
// ]

let all_symbols = (await gesel.fetchAllGenes("9606")).get("symbol");
console.log(Array.from(set_members).map(i => all_symbols[i]));
// [
//   [ 'ATP1A2' ],
//   [ 'SLC29A1' ],
//   [ 'SLC29A2' ],
//   [ 'SLC1A3' ],
//   [ 'SLC1A6' ],
//   [ 'SLC1A7' ],
//   [ 'SNAP25' ],
//   [ 'SYNGR3' ],
//   [ 'SLC6A5' ],
//   [ 'SLC38A1' ]
// ]
```

Each set also has some associated free text in its name and description.
`gesel` can query this text to find sets of interest, with some basic support for the `?` and `*` wildcards.

```js
let hits = await gesel.searchSetText("9606", "B immunity");
let first_hit = await gesel.fetchSingleSet("9606", hits[0]);
// {
//   name: 'GO:0019724',
//   description: 'B cell mediated immunity',
//   size: 4,
//   collection: 0,
//   number: 5715
// }

let hits2 = await gesel.searchSetText("9606", "B immun*");
let first_hit2 = await gesel.fetchSingleSet("9606", hits2[0]);
// {
//   name: 'GO:0002312',
//   description: 'B cell activation involved in immune response',
//   size: 2,
//   collection: 0,
//   number: 858
// }
```

The output of `searchSetText()` can then be combined with the output of `findOverlappingSets()` to implement advanced searches in downstream applications.

# Implementation details

`gesel` supports two modes of operation - a "full client-side" mode and a more lightweight "on-demand request" mode.

In full client-side mode, `gesel` will download the relevant database files from a static file server to the client.
All calls to `gesel` functions will then perform queries directly on the downloaded files;
this includes the identification of overlapping sets, querying the set-related text, and simple extraction of per-set/collection details.
In this mode, the user pays an up-front cost for the initial download such that all subsequent calculations are fully handled within the client.
This avoids any further network activity and the associated latency.
For many applications, the up-front cost is likely to be modest - for example, the total size of the default human gene set database 
(containing over 40,000 sets, including the Gene Ontology [@ashburner2000go] and most of MSigDB [@liberzon2011molecular]) is just over 9 MB -
so full client-side operation is simple and practical in most cases.

In the on-demand request mode, `gesel` will perform HTTP range requests to fetch relevant slices of each database file.
For example, `findOverlappingSets()` needs to obtain the mapping of each gene to the gene sets of which it is a member.
Rather than downloading the entire mapping file for all genes, `gesel` will ask the server to return the range of bytes containing only the mapping for the desired gene.
This is inspired by similar strategies for querying genomics data [@kancherla2020epiviz] and avoids transferring the entire file to the client, reducing the burden on the client device and network.
It is suited for applications that expect only sporadic usage of `gesel` such that an up-front download of the entire database cannot be justified.
It is also more scalable as the number of gene sets increases into the millions, where the total size of an up-front download may exceed 1 GB and become impractical.
Obviously, using this mode involves increased network activity and latency from multiple range requests if `gesel` functions are frequently called.
This is partially mitigated by `gesel`'s transparent caching of responses in memory.

In both cases, we stress that `gesel` only requires a static file server to host the database files and optionally to support range requests.
The client machine is still responsible for performing all of the calculations, which provides a number of advantages.
We do not have to provision and maintain a dedicated back-end server to handle the `gesel` queries, saving time and money;
rather, any generic static server can be used including free offerings, e.g., from GitHub.
The user receives the results as soon as the calculations are complete, enabling low-latency applications that minimize network traffic.
Similarly, there is no transfer of user-supplied gene lists to an external server, avoiding any questions over data ownership.
Most importantly, as each user brings their own compute to the application, it scales to any number of users at no cost to us (i.e., the `gesel` maintainers).

# Preparing database files

The `gesel` package works with any database files prepared according to the contract outlined in the feedstock repository [@geselfeedstock].
Briefly, this involves defining all appropriate synonyms for each gene, typically involving one or more of Ensembl identifiers, Entrez identifiers, or gene symbols;
details about each collection and set, including the name and description;
the gene membership of each set, and conversely, the identities of the sets that contain each gene;
and the identities of the sets associated with each token generated from the names/descriptions, for use in free-text searches.
We expect one set of files per species, meaning that only the relevant files are transferred to the client when one species (usually human or mouse) is of interest.

We apply some standard tricks to reduce the transfer size of the database files, particularly for the mappings between sets and their genes.
We convert all sets and genes into integer identifiers to avoid handling large arbitrarily named strings.
For each set, we sort the gene identifiers and store the differences between adjacent values, decreasing the number of digits (and bytes) that need to be stored and transferred.
`gesel` will then recover the gene identifiers by computing a cumulative sum for each set on the client machine.
The same approach is used to shrink the mappings from each gene to the identifiers of the sets in which it belongs.
Finally, we compress all files to be transferred, relying on the `pako` library [@pako] to perform decompression in the browser.

By default, `gesel` uses a simple database that incorporates gene sets from the Gene Ontology [@ashburner2000go] and (for human and mouse) the majority of MSigDB [@liberzon2011molecular].
(To avoid potential issues, only the MSigDB gene sets with permissive licensing are used here.)
This is currently hosted for free on GitHub without any need for a specialized backend server.
However, application developers can easily point `gesel` to a different database by simply changing the URL used in the various HTTP requests.
For example, we adapted the scripts in the feedstock repository to create a company-specific database of custom gene sets based on biomarker lists and other signatures. 
This is hosted inside our internal network for use by our in-house `gesel`-based applications.

# Demonstration 

To demonstrate the functionality of `gesel`, we developed a simple web application that tests for gene set enrichment among user-supplied genes [@geselapp].
This accepts several parameters - namely, the species of interest, the available collections for that species, a list of user-supplied genes, and a free-text query.
The application then returns a list of gene sets in the designated collections that overlap at least one user-supplied gene and satisfy the free-text query.
All matching gene sets are shown in a table, sorted by increasing p-value to focus on those with significant enrichment.
Clicking on a row corresponding to a particular gene set shows the identities of its genes, with highlights applied to those in the user-supplied list.
In addition, the parameters of each search are captured by query strings, allowing users to easily save and share searches by copying the URL from the browser's address bar.

One particularly useful feature of `gesel` is the ability to customize how it fetches assets from the static file server.
For our demonstration application, we override `gesel`'s default download function to instruct the browser to cache the responses on disk.
This means that the user can avoid re-downloading the database files upon subsequent visits to the application.
On a more technical note, we perform the download via a proxy to provide the correct cross-origin resource sharing (CORS) headers;
this is only necessary as GitHub does not provide public CORS access to all of its resources, and can be omitted for servers that are more appropriately configured. 

As `gesel` is a `npm` package, it can be easily incorporated into applications such as our demonstration site by following the standard `npm` installation process.
Each application is free to decide how it wants to conduct the gene set search.
The demonstration site uses almost all of `gesel`'s functionality, but this need not be the case - 
for example, some of our internal applications accept a user-supplied gene list but do not support free-text queries,
while others are only interested in finding the sets containing a single gene of interest.
This decision also determines whether an application should use the full client-side or on-demand request modes.

# Further comments

`gesel`'s design is greatly inspired by the "edge computing" paradigm.
The idea is to process data at or near the "edge" of the network (e.g., on client devices) rather than centralizing the compute in a data center.
By doing so, we can process user requests at greater speed and volume by avoiding a round-trip through the network.
From a developer perspective, performing the compute on the client is appealing as it requires very little maintenance;
there is no need to deploy and monitor a custom backend, and concerns over scaling and downtime are effectively irrelevant.
Indeed, if the analysis of single-cell data can be migrated to the client [@lun2022single], it is straightfoward to do the same for a relatively lightweight task like GSEA.

We note that there is some room for improvement in `gesel`'s range requests.
We could bundle multiple ranges into a single request, reducing the burden on the server and network by avoiding separate requests, e.g., when querying the overlapping sets for multiple genes.
While sensible, we have not implemented this approach because GitHub does not currently respect multipart ranges and we don't want to pay for our own static hosting.
We could also compress each requested byte range (e.g., with DEFLATE) during database preparation, thus reducing the number of bytes that need to be transferred per range request.
This is unlikely to have much of an effect at the current database size, given that each range request involves fewer than 300 bytes on average.
Nonetheless, both of these ideas may provide some opportunities for improving performance as the queries and databases increase in size.

# Acknowledgements

Thanks to Chris Bolen and colleagues, for the scientific questions that motivated the development of this library;
Hector Corrado Bravo, for his feedback on the uselessness of the early versions of the free-text search;
and Allison Vuong and Luke Hoberecht, for recovering ATLL's scarf when he forgot it while thinking about the library design during the team dinner.

# References
