---
title: Obtaining Genomes from the GEM Db
author: Adrià Auladell
date: '2020-11-24'
slug: obtaining-genomes-from-the-gem-db
categories: []
tags:
  - databases
  - bioinformatics
subtitle: ''
summary: ''
authors: []
lastmod: '2020-11-24T11:42:14+01:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

Last month a new shiny database was born for the joy and desperation of all the microbiologists interested in MAGs. The Genomic catalog of Earth Microbiomes (GEM db) is one of the most comprehensive datasets right now regarding the diversity of bacteria and archaea. The GEMdb expanded the known phylogenetic diversity of bacteria and archaea by 44% and it is available for all kind of analyses. Since I was interested in solely my subject of study, here I will show how I downloaded all the MAGs i was interested.

Here I will do an extended explanation, if you are an experienced user you can jump directly to the [github repository](https://github.com/adriaaula/download_GEMdb) with the scripts :^) 

First of all, by going to the [paper data availability](https://www.nature.com/articles/s41587-020-0718-6#data-availability) we see that there are several options for downloading.

By going to the [NERSC portal](https://portal.nersc.gov/GEM) and clicking to genomes, we observe several files:

![nserc\_dir](./nserc_dir.png)

We will retrieve the `genome_metadata.tsv`. If you click with the right in the mouse you can copy the link directly.

    # we make a directory to save all the data
    mkdir gem_db_subset
    cd gem_db_subset/
    wget  https://portal.nersc.gov/GEM/genomes/genome_metadata.tsv

To check the file I used Wei Shen [csvtk](https://github.com/shenwei356/csvtk). It is a useful command-line tool to work both with csv and tsv. Yup, it is yet *another* tool to do things, maybe unnecessary. But I think it makes way easier some processes. Sorry not sorry ;^)

    # We discover the headers for each column
    csvtk headers -t genome_metadata.tsv

    # We check the first values for some of the headers we are interested
    csvtk cut -f completeness,ecosystem_type,taxonomy  -t genome_metadata.tsv  | csvtk head

We are specifically interested in **high quality genomes** coming from a **marine ecosystem.** To check how it is specified "marine" in this `tsv` we will check it with a grep

    csvtk cut -f completeness,ecosystem_type,taxonomy  -t genome_metadata.tsv  | grep 'arine'

Having a clear picture, we just have to filter all the results to keep only the genomes of interest. Get out, soil people.

    csvtk filter -f "completeness>=70" -t genome_metadata.tsv | csvtk grep -t  -p Marine -f ecosystem_type | csvtk uniq > marine_mags_gemdb.tsv

And now we want to retrieve each of the genome files in the db. Each genome is stored with:

-   `faa`: the aminoacidic sequences.

-   `fna`: the nucleotide contigs, without annotating, just the raw sequences.

-   `ffn`: the annotation of the faa sequences in nucleotides.

We could download all the genomes and then filter out the ones we are interested, but a more elegant way is to search specifically each genome and afterwards tidy all the data. The NSERC portal makes it easy with the directories, `faa/`, `fna/` and `ffn/`.

We therefore iterate through a loop and get each of the sequences separately.


    mkdir faa
    mkdir fna
    mkdir ffn
    
    for genome_name in $(cut -f1,1 marine_mags_gemdb.tsv );
    do
        wget -P faa/ https://portal.nersc.gov/GEM/genomes/faa/${genome_name}.faa.gz;
        wget -P fna/ https://portal.nersc.gov/GEM/genomes/fna/${genome_name}.fna.gz;
        wget -P ffn/ https://portal.nersc.gov/GEM/genomes/ffn/${genome_name}.ffn.gz;

    done;



And voilà!

A new dataset to play with.

I hope you find it useful :)

## Trouble Trouble Trouble! 

1. By working in this I found that there are duplicates in the `genome_metadata.tsv`! Which is a nice thing to discover since maybe the creators will solve it! To solve the problem in our side I included a line that erases these duplicates using sort. The amount of rows went from ~10.000 to ~6.000. Great :) 

2. But surprise! Additionally to the duplicates we found initially there other pervasive duplicates. Some of the duplicates present some column values changed, and therefore the `sort` doesn't delete them. Let's see an example: 

```
3300025281_5	3300025281	2837964	108	41511	0	1	1	36	94.88	1.32	88.28	MQ	OTU-4815	d__Bacteria;p__Proteobacteria;c__Gammaproteobacteria;o__Pseudomonadales;f__Pseudohongiellaceae;g__UBA5109;s__GCA_002414385.1	Environmental	Aquatic	Marine	Deep ocean	163.4956	9.2013
3300025281_5	3300025281	2837964	108	41511	0	1	1	36	94.88	1.32	88.28	MQ	OTU-4815	d__Bacteria;p__Proteobacteria;c__Gammaproteobacteria;o__Pseudomonadales;f__Pseudohongiellaceae;g__UBA5109;s__GCA_002414385.1	Environmental	Aquatic	Marine	Deep ocean	-163.4956	9.2013
```

These values only changed by the `-` sign in the LAT. Small things that create duplicates :) 

To filter out these problems we can perform again a sort before performing the loop. See here. 

3. Finally, by downloading many many files sometimes there is a problem in the connection, or a random error in the process, making that some files are lost. To check that everything is correct and that we have all the values we are interested, we can do a last check to redownload any genome not present in our directories. See [redownloading script](https://github.com/adriaaula/download_GEMdb/blob/main/redownloading_faulty_genomes.sh).  


