## green iguana microbiome - Diet x LPS
## Lab study - Diet x Immune challenge
## Karen M. Kapheim
## February 4, 2022

This is processing of 16S amplicon sequences in the Qiime2 pipeline.
Samples were collected with cloacal swabs from green iguanas in the Denardo lab 
at ASU.
DNA was extracted at USU, PCR and sequencing was performed at Shedd with
515f and 806rB primers.

PCR includes both a Zymobiomics Microbial Mock Community and a no template control.


NOTES: Working through tutorials on https://docs.qiime2.org/2018.8/interfaces/q2cli/
Working in interactive mode.

## Gather data

Sample data: `GreenIguanaManifest_subset.csv` 
# modified from what Erin sent `GreenIguanaManifest.xls`

Sequence data: `/uufs/chpc.utah.edu/common/home/kapheim-group2/GreenIguanas/2021-raw-data/FASTQ_Generation_2021-11-13_11_11_13Z-487151666/`

Experimental Information: 

1. What is the effect of diet?

> Compare baseline vs pre_LPS1

2. How does diet interact with immune challenge over short/long term?

> Compare glucose X LPS in a 2x2     
> Separate analyses for timepoints 24 h, 72 h, pre_biopsy    

Use all extraction blanks

Need to pull out the relevant samples.

#### Get sequences

```
cd /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguanas_LPSdiet
# Make a list of directories from which to copy seq files
awk -F ',' '{print $19}' GreenIguanaManifest_subset.csv > seq_paths.txt
# Remove the header line that was also printed
sed '1d' seq_paths.txt > seq_paths_noheader.txt
# check
wc -l GreenIguanaManifest_subset.csv
wc -l seq_paths.txt
wc -l seq_paths_noheader.txt
# 209 samples
# copy files
```

There was some corruption/weird characters at the end of each line due to importing 
from excel. Ended up just copying and pasting the column into a new text file 
`seq_paths_copied.txt` and manually deleting the weird characters in emacs. 
Use this to copy files.

```
mkdir rawseqs
cd rawseqs
for FILE in $(cat ../seq_paths_copied.txt) ; do  cp -R ${FILE} ./; done
# get the sequences out of the directories
find . -name '*.fastq.gz' -exec mv {} ./ \;
# rename directory
cd ..
mv rawseqs/ rawseq_dirs
mkdir rawseqs
mv ./rawseq_dirs/*.fastq.gz ./rawseqs/
# check that we have the correct number of seqs
ls ./rawseqs/ | wc -l
# delete the directory of directories
rm -R rawseq_dirs/
```

Now we should have just the fastq files we want in the `rawseqs` directory.

#### Make manifest

```
ls ./rawseqs/ > dietLPS_manifest.csv
```

Open in LibreOffice and add header and other columns

headers:
`sample-id,absolute-filepath,direction`

#### Set-up


```
salloc --time=24:00:00 --account=kapheim-np --partition=kapheim-np --nodes=1 -c 48 
module load anaconda3/2019.03 qiime2/2019.4
cd /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguanas_LPSdiet/
```

## Import sequence data

Working from https://docs.qiime2.org/2018.8/tutorials/importing/.


```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguanas_LPSdiet/import
qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path dietLPS_manifest.csv \
  --output-path ./import/LPSdiet-pe-demux.qza \
  --input-format PairedEndFastqManifestPhred33
qiime demux summarize \
  --i-data ./import/LPSdiet-pe-demux.qza \
  --o-visualization ./import/LPSdiet-pe-demux.qzv
```


# Trim adapters

Following
https://forum.qiime2.org/t/demultiplexing-and-trimming-adapters-from-reads-with-q2-cutadapt/2313    
https://docs.qiime2.org/2018.8/plugins/available/cutadapt/trim-paired/    
https://rachaellappan.github.io/VL-QIIME2-analysis/pre-processing-of-sequence-reads.html   

Used primers for 515f and 806r.



```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguanas_LPSdiet/trimmed
qiime cutadapt trim-paired \
  --i-demultiplexed-sequences ./import/LPSdiet-pe-demux.qza \
  --p-cores 48  \
  --p-front-f GTGYCAGCMGCCGCGGTAA \
  --p-front-r GGACTACNVGGGTWTCTAAT \
  --o-trimmed-sequences ./trimmed/LPSdiet-pe-trimmed.qza \
  --verbose
&> ./trimmed/primer_trimming.log
qiime demux summarize \
  --i-data ./trimmed/LPSdiet-pe-trimmed.qza \
  --o-visualization ./trimmed/LPSdiet-pe-trimmed.qzv
```

#### Check quality and sequence length

```
qiime tools view ./import/LPSdiet-pe-demux.qzv
qiime tools view ./trimmed/LPSdiet-pe-trimmed.qzv
```


## Denoising

Used visualization to choose length to truncate sequences to based on where median quality score
drops below 30.     

The reverse reads look a lot more variable in terms of quality. 

Optimal truncating:   

f-248, r-219   


```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguanas_LPSdiet/dada2
qiime dada2 denoise-paired \
  --p-n-threads 48 \
  --i-demultiplexed-seqs ./trimmed/LPSdiet-pe-trimmed.qza \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f 248 \
  --p-trunc-len-r 219 \
  --o-table ./dada2/LPSdiet_table.qza \
  --o-representative-sequences ./dada2/LPSdiet_repseqs.qza \
  --o-denoising-stats ./dada2/LPSdiet_denoising-stats.qza
```

Taking a long time.




#### Summarize the denoising   

No metadata file    

```
qiime feature-table summarize \
  --i-table ./dada2/LPSdiet_table.qza \
  --o-visualization ./dada2/LPSdiet_table_viz.qzv
qiime feature-table tabulate-seqs \
  --i-data ./dada2/LPSdiet_repseqs.qza \
  --o-visualization ./dada2/LPSdiet_repseqs_viz.qzv
```



#### Visualize the denoising stats   





```
qiime metadata tabulate \
  --m-input-file ./dada2/LPSdiet_denoising-stats.qza \
  --o-visualization ./dada2/LPSdiet_denoising-stats_viz.qzv
```



## Taxonomic Classification

#### Import classifiers for training datasets

From: https://docs.qiime2.org/2021.11/data-resources/#taxonomy-classifiers-for-use-with-q2-feature-classifier 

>QIIME-compatible SILVA releases (up to release 132), as well as the licensing \
information for commercial and non-commercial use, are available at \
https://www.arb-silva.de/download/archive/qiime.


```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguanas_LPSdiet/training-feature-classifiers
cd training-feature-classifiers
wget https://www.arb-silva.de/fileadmin/silva_databases/qiime/Silva_132_release.zip
unzip Silva_132_release.zip
rm -R __MACOSX/
```

#### Training on Silva 132 database

*Import sequences and taxonomy*

Sequences

```
cd /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguanas_LPSdiet
qiime tools import \
  --type 'FeatureData[Sequence]' \
  --input-path ./training-feature-classifiers/SILVA_132_QIIME_release/rep_set/rep_set_16S_only/99/silva_132_99_16S.fna \
  --output-path ./training-feature-classifiers/SILVA_132_QIIME_release/silva_132_99_16S.qza
```

Now taxonomy

Using 7 level taxonomy

```
qiime tools import \
  --type 'FeatureData[Taxonomy]' \
  --input-format HeaderlessTSVTaxonomyFormat \
  --input-path ./training-feature-classifiers/SILVA_132_QIIME_release/taxonomy/16S_only/99/taxonomy_7_levels.txt\
  --output-path ./training-feature-classifiers/SILVA_132_QIIME_release/silva_132_99_16S_7level_tax.qza
```


*Extract reference reads*

>It has been shown that taxonomic classification accuracy of 16S rRNA gene sequences\
 improves when a Naive Bayes classifier is trained on only the region of the target \
 sequences that was sequenced (Werner et al., 2012). This may not necessarily \
 generalize to other marker genes (see note on fungal ITS classification below). \
 We know from the Moving Pictures tutorial that the sequence reads that we’re trying \
 to classify are 120-base single-end reads that were amplified with the 515F/806R \
 primer pair for 16S rRNA gene sequences. We optimize for that here by extracting \
 reads from the reference database based on matches to this primer pair, and then \
 slicing the result to 120 bases.

```
qiime feature-classifier extract-reads \
  --i-sequences ./training-feature-classifiers/SILVA_132_QIIME_release/silva_132_99_16S.qza \
  --p-f-primer GTGYCAGCMGCCGCGGTAA \
  --p-r-primer GGACTACNVGGGTWTCTAAT \
  --p-min-length 100 \
  --p-max-length 400 \
  --o-reads ./training-feature-classifiers/SILVA_132_QIIME_release/silva_132_99_16S_refseqs.qza
```

This step took a long time (~20 min) to complete.


 
*Formatting* - Skipped this 2/4/22

From F. Oliaro's workflow:     

> for god knows why when you import the SILVA taxonomy there are spaces in \
certain names which messes up QIIME so we need to remove those before moving \
forward

      # 4 step process that we can combine into 1 script
 
```
qiime tools export \
  --input-path ./training-feature-classifiers/SILVA_132_QIIME_release/silva_132_99_16S_7level_tax.qza \
  --output-path ./training-feature-classifiers/taxonomy-with-spaces/
qiime metadata tabulate \
  --m-input-file ./training-feature-classifiers/taxonomy-with-spaces/taxonomy.tsv \
  --o-visualization ./training-feature-classifiers/taxonomy-with-spaces/taxonomy-as-metadata.qzv
qiime tools export \
  --input-path ./training-feature-classifiers/taxonomy-with-spaces/taxonomy-as-metadata.qzv \
  --output-path ./training-feature-classifiers/taxonomy-without-spaces/
qiime tools import \
  --type 'FeatureData[Taxonomy]' \
  --input-path ./training-feature-classifiers/taxonomy-without-spaces/metadata.tsv \
  --output-path ./training-feature-classifiers/taxonomy-without-spaces/silva_132_99_16S_7level_tax_nospaces.qza
``` 
    
 
*Train the classifier*





7 level taxonomy - with spaces

Got an error with the sklearn classifier using the classifier without spaces 
so trying the original.

```
qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads ./training-feature-classifiers/SILVA_132_QIIME_release/silva_132_99_16S_refseqs.qza \
  --i-reference-taxonomy ./training-feature-classifiers/SILVA_132_QIIME_release/silva_132_99_16S_7level_tax.qza \
  --o-classifier ./training-feature-classifiers/SILVA_132_QIIME_release/silva_132_99_16S_7tax_withspaces_classifier.qza
``` 


#### Classify rep sequences

*Test the classifier*

> Verify that the classifier works by classifying the representative sequences \
> and visualizing the resulting taxonomic assignments.

7 level taxonomy - with spaces

The classify-sklearn step took a long time .

```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguanas_LPSdiet/classified
qiime feature-classifier classify-sklearn \
  --i-classifier ./training-feature-classifiers/SILVA_132_QIIME_release/silva_132_99_16S_7tax_withspaces_classifier.qza \
  --i-reads ./dada2/LPSdiet_repseqs.qza \
  --o-classification ./classified/LPSdiet_SILVA_132_99_16S_taxonomy_withspaces.qza
qiime metadata tabulate \
  --m-input-file ./classified/LPSdiet_SILVA_132_99_16S_taxonomy_withspaces.qza \
  --m-input-file ./dada2/LPSdiet_repseqs.qza \
  --o-visualization ./classified/LPSdiet_SILVA_132_99_16S_taxonomy_withspaces_viz.qzv
```




## Generate a phylogenetic tree de novo


Use existing pipelines to generate a de novo tree that includes all sequences.

https://docs.qiime2.org/2020.2/tutorials/phylogeny/

[Scroll down to 'Pipelines' near the bottom]

> This pipeline will start by creating a sequence alignment using MAFFT, after \
which any alignment columns that are phylogenetically uninformative or ambiguously \
aligned will be removed (masked). The resulting masked alignment will be used to \
infer a phylogenetic tree and then subsequently rooted at its midpoint. Output files\
 from each step of the pipeline will be saved. This includes both the unmasked and \
masked MAFFT alignment from q2-alignment methods, and both the rooted and unrooted \
phylogenies from q2-phylogeny methods.

```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguanas_LPSdiet/phylogeny
qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences ./dada2/LPSdiet_repseqs.qza \
  --output-dir ./phylogeny/merged_mafft-fasttree-output_7tax
```



## Statistical analysis


Export to R and analyze in RStudio on laptop.  

File `green_iguana_LPSdiet_Jan2022.Rmd`

Exported:   

| element | file |
| --- | --- | 
| table | ./dada2/LPSdiet_table.qza | 
| tree | ./phylogeny/merged_mafft-fasttree-output_7tax/rooted_tree.qza |
| taxonomy | ./classified/LPSdiet_SILVA_132_99_16S_taxonomy_withspaces.qza |
| metadata | GreenIguanaManifest_subset.csv |


 
