---
title: 'MODBrowser: A Multi-Omics Data Browser for Integrative Exploration and Discovery of Genomic and Protein Interaction Networks'
tags:
  - R
  - Shiny
  - genomics
  - proteomics
  - protein interaction
  - research database
authors:
  - name: Xiaopei Zhang
    orcid: 0009-0002-7980-8653
    affiliation: 1
  - name: Caleb Embree
    orcid: 0000-0002-7437-4510
    affiliation: 1
  - name: Katherine L.B. Borden
    corresponding: true
    affiliation: 1
affiliations:
  - name: Department of Pharmacology, Northwestern University Feinberg School of Medicine, United States
    index: 1
date: 27 April 2026
bibliography: paper.bib
---

# Summary

`MODBrowser` is an R package and Shiny application for building a searchable
browser of genomic, proteomic, and protein interaction datasets. A user can add
CSV, TXT, or XLSX files together with a simple metadata spreadsheet to build the
database index. This allows them to search by gene symbol or identifier, inspect
dataset-level metadata, and export result tables for downstream analysis. The
browser supports exact, family, and fuzzy gene-symbol search modes, gene
presence checks across datasets, and comparison of local findings with UniProt
and MINT protein interaction information [@uniprot2025; @mint2012].

The package is intended for research groups that need to compare laboratory
results with previous experiments or published resources but do not want every
query to require custom code. For example, a bench scientist can use the Shiny
interface to ask whether a gene or protein appears in any indexed dataset, view
the supporting rows, and export the matching evidence. A more technical user can
use the R functions directly to rebuild indexes, update datasets, and manage
local or packaged deployments.

# Statement of need

Modern molecular biology is data rich, and omics projects often generate many
spreadsheets and tabular files that are useful beyond the analysis in which they
were first created. These files may contain gene identifiers, gene symbols, fold
changes, proteomics measurements, interaction evidence, and other
analysis-specific outputs. However, depending on the analysis and the person
running it, these files often contain different column names, sheet structures,
and metadata conventions. Comparing a new laboratory result with this
collection can require repeated manual searching or the creation of per-user ad
hoc scripts. Those approaches are slow, difficult to reproduce, hard for
collaborators without coding experience to audit, and can result in false
negatives due to human error or missing datasets.

`MODBrowser` addresses this need by turning a folder of heterogeneous datasets
and a metadata workbook into a local, searchable research database. The metadata
workbook records which columns contain gene identifiers and symbols, while the R
package builds normalized index files and the Shiny application presents the
data in a point-and-click browser. This design lets researchers integrate newly
produced data with previously published or externally curated data and then
compare findings across sources without reimplementing the same import and
search logic.

The package is especially useful in collaborative laboratory settings where one
person can curate and update the database, allowing many others to search the
same datasets. It lowers the barrier for non-programming users, preserves
exportable search results, and keeps the database rebuild process reproducible
through explicit R functions. In addition, the local nature of the database
keeps data secure and allows for further customization on a lab-by-lab basis.

# State of the field

Several established tools support interactive biological data exploration or
network analysis. Cytoscape is a widely used environment for biological network
integration and visualization [@cytoscape2003]. The iSEE package provides a
general Shiny-based interface for exploring data stored in Bioconductor
`SummarizedExperiment` objects [@isee2018], and ShinyCell provides shareable
single-cell expression browsers [@shinycell2021]. Curated resources such as
UniProt and MINT provide high-quality protein annotation and protein-protein
interaction records [@uniprot2025; @mint2012].

These tools solve important parts of the broader problem, but they do not target
the specific workflow that motivated `MODBrowser`: maintaining a lab-scale
search browser over many simple files, including both local experiments and
published comparison datasets, agnostic of analysis type and data format, with
minimal setup for non-programming users. It provides users a tool to search
through multiple databases within their lab simultaneously and comprehensively.
`MODBrowser` is not intended to replace Cytoscape for network modeling or
Bioconductor tools for structured assay analysis. Instead, it fills an upstream
curation and triage role: researchers can quickly identify which datasets
contain a gene or protein of interest, recover the original rows, and compare
local hits with interaction evidence before deciding whether a more specialized
analysis environment is needed.

# Software design

`MODBrowser` is implemented as an R package built around a small set of core
indexing functions and a bundled Shiny application [@R; @shiny]. The main data
model separates raw datasets, dataset metadata, derived index files, and
exported search results. The design supports a single user running the browser
on a personal machine or a group installing the browser on a shared drive that
multiple users can access. Setup differs for these two use cases, but after
setup the browser workflow is the same.

A single user can start the package with `MODB.Run()`, which uses the data
bundled with the package by default. For multiple users running the browser from
a shared drive, such as a whole lab using the same browser setup, `modb.help()`
can be used to set up the folders for data and output in a location that all
users can access without needing to modify the package. Users can then run
`MODB.Run(use_current = TRUE)` to use these folders. For both use cases,
`modb.nowDataset()` can be used to see which datasets are in use. After making
updates to the input, `modb.sync.to.shinyapp()` synchronizes the datasets and
their descriptions into the background application.

Once users change the datasets and description, they must create the index. This
process can be done through the Shiny application via the "Rebuild Everything"
button in the left panel. The button calls the local R console to run
`modb.input.all()`, which allows non-programming users to maintain and update
the database after initial setup, including when the browser is used from a
shared drive.

During indexing, the package reads the dataset metadata, detects file type,
extracts the configured gene identifier and gene-symbol columns, normalizes
blank-like values, fills missing identifier-symbol pairs where possible, and
writes per-dataset index files. After indexing the database, `MODBrowser` is
ready to use.

The Shiny interface exposes common operations as tabs: search, check, dataset
information, output files, database updates, and UniProt/MINT comparison. The
Search tab is the primary feature, allowing users to input genes or proteins of
interest and retrieve detailed results about the query from all indexed
datasets. Search outputs, together with single-gene or single-protein summary
files, are automatically saved to disk and can be reviewed and exported through
the Output Files tab. When searching, the application runs `search.Symbol()` and
`print.Symbol()`, or `search.ID()` and `print.ID()`, automatically in the local
R console.

Search results preserve links back to the original dataset rows in the Check
tab, so exported evidence can be checked against the source table instead of
becoming a detached summary. The Check tab identifies which datasets contain a
specified gene or protein and reports the exact location of each match using the
format `fileName.sheet~row`, allowing users to locate the original records
efficiently. The Check workflow uses `search.Symbol()` or `search.ID()` without
the corresponding print functions, because it is intended to report dataset
presence and row locations rather than produce the full exported search output.

The UniProt/MINT Comparison tab retrieves protein and protein-protein
interaction information from UniProt and MINT about a queried protein or result
from the Search tab. Users can also compare interaction networks from their own
datasets with those from these curated databases using Venn diagrams. The
Dataset Information tab shows dataset metadata and structure, including fields
such as date, owner, research background, dataset type, and notes. This metadata
can be updated directly under the dataset folder, or with
`modb.sync.to.shinyapp()` if the browser is already running.

The central design trade-off is to prefer portable files and spreadsheet-based
configuration over a database server. This keeps installation and sharing simple
for small research groups, and because data can remain on institution-managed
local machines rather than third-party cloud services, it is also suitable for
unpublished or sensitive datasets that should not leave internal environments.
The package is therefore best suited to local or departmental deployments rather
than high-concurrency public web services. The requirement for local machines to
run the data browser is an R installation. The R functions remain available for
users who want scripted rebuilds or local customization.

# Research impact statement

`MODBrowser` was developed to support molecular pharmacology research workflows
where laboratory findings must be compared with previous experiments and
interaction databases. Its immediate impact is practical: it allows
collaborators without coding experience to perform cross-dataset searches,
inspect source evidence, and export comparison tables using a local browser. The
bundled example database and protocol materials provide reproducible reference
material for reviewers and new users.

The package also has credible near-term scholarly significance because it
encodes a reusable pattern common to many biomedical groups: converting
scattered analysis outputs into a searchable evidence browser that can combine
current laboratory datasets with published data. The package was developed to
support ongoing genomics, interactome, and proteomics research in the Borden Lab,
where it has enhanced collaboration and accelerated research by allowing easier
insights into data generated by other lab members. This supports more
transparent comparison of new findings against existing evidence and reduces
the amount of one-off code required for each query. Future releases can
strengthen this impact case by documenting external adoption, adding more public
example datasets, and archiving tagged releases with a software DOI.

# Acknowledgements

We thank members of the Borden Lab for feedback on laboratory data exploration
workflows and for motivating the need for an accessible comparison browser.
This work was supported by grants to KLBB from the NIH R01 98571 and 80728.

# References
