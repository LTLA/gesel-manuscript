---
title: "gesel: a Javascript package for client-side gene search searches"
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

# Statement of need

# Usage

`gesel` aims to find the gene sets that overlap a user-supplied list of genes.
To demonstrate, we'll consider the following list of gene symbols mixed with Ensembl and Entrez identifiers.

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

To simplify matters, we'll take the first matching `gesel` gene identifier for each user-supplied gene.
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

Each set also has some associated free text in its name and description.
`gesel` provides functionality to query this text to find sets of interest, with simple support for wildcard searches.

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
this includes the identification of overlapping sets, searching on the set-related text, and simple extraction of per-set/collection details.
In this mode, the user pays an up-front cost for the initial download such that all subsequent calculations are fully handled within the client.
This avoids any further network activity and the associated latency.
For many applications, the up-front cost is likely to be modest - for example, the total size of the default human gene set database 
(containing over 40,000 sets, including the Gene Ontology [@ashburner2000go] and most of MSigDB [@liberzon2011molecular]) is just over 9 MB -
so full client-side operation is simple and practical in most cases.

In the on-demand request mode, `gesel` will perform HTTP range requests to fetch relevant slices of each database file.
For example, `findOverlappingSets()` needs to obtain the mapping of each gene to the gene sets of which it is a member.
Rather than downloading the entire mapping file for all genes, `gesel` will ask the server to return the range of bytes containing only the mapping for the desired gene.
This avoids transferring the entire file to the client, reducing the burden on the client device and network.
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
We convert all sets and genes into integer identifiers to avoid handling large, arbitrarily named strings.
For each set, we sort the gene identifiers and store the differences between adjacent values, decreasing the number of digits (and bytes) that need to be stored and transferred.
`gesel` will then recover the gene identifiers by computing a cumulative sum for each set on the client machine.
The same approach is used to shrink the mappings from each gene to the identifiers of the sets in which it belongs.
Finally, we compress all files to be transferred, relying on the `pako` library [@pako] to perform decompression in the browser.

By default, `gesel` uses a simple database that incorporates gene sets from the Gene Ontology [@ashburner2000go] and (for human and mouse) the majority of MSigDB [@liberzon2011molecular].
(To avoid potential issues, only the gene sets with permissive licensing are used here.)
This is currently hosted for free on GitHub using the Releases of the feedstock repository, without any need for a specialized backend server.
However, application developers can easily point `gesel` to a different database by simply changing the URL used in the various function calls.
For example, we created a database of company-specific gene sets based on biomarker lists, custom signatures, etc., with some simple adjustments of the scripts in the feedstock repository. 
This is hosted inside our internal network for querying by our in-house applications.

# Demonstration 

# Further comments

# Acknowledgements

# References

