conda activate r_env

R

# Load libraries
install.packages("vegan")
library(vegan)   # For diversity metrics
library(tidyverse)

# Load your data
data <- read.delim("path/to/your_file.tsv", header = TRUE)

# Prepare the data for diversity calculations
rownames(data) <- data[, 1] # Use the first column as row names (ASV IDs or taxonomy)
data <- data[, -1] # Remove the first column from the abundance table


## Friday:
# Creating a metadata file first:

    awk '{print $1}' 'raw_data/Alpha Diversity Metrics .tsv' >> working_data/metadata.tsv

## Added 2 more columns (genotype and id) manually !!
    conda activate qiime2-amplicon
    
    qiime metadata tabulate --m-input-file working_data/metadata.tsv --o-visualization working_data/metadata.qzv

## Importing files:
    for file in raw_data/illumina_16S_reads/*R1_001.fastq.gz;do
        r2=$(echo "$file" | sed 's/R1/R2/')
        echo '$PWD'/$file >> temp
        echo '$PWD'/$r2 >>temp2
    done
    
    cat working_data/manifest.tsv

    qiime tools import \
        --type "SampleData[PairedEndSequencesWithQuality]" \
        --input-format PairedEndFastqManifestPhred33V2 \
        --input-path working_data/manifest.tsv \
        --output-path working_data/demux_seqs.qza

## Checking sequences and sequence depth of input files:
    
    cd working_data && ls

    qiime demux summarize \
        --i-data ./demux_seqs.qza \
        --o-visualization ./demux_seqs.qzv

## Denoising with DADA2 (not sure if I need to do again, but just trying)

    qiime dada2 denoise-paired --verbose --i-demultiplexed-seqs ./demux_seqs.qza --p-trim-left-f 0 --p-trim-left-r 0 \
        --p-trunc-len-f 0 --p-trunc-len-r 0 --p-max-ee-f 5 --p-max-ee-r 5 --p-min-overlap 9 --o-table ./dada2_table.qza \
        --o-representative-sequences ./dada2_rep_set.qza --o-denoising-stats ./dada2_stats.qza

## Genarating the feature table file from denoised statistics

    qiime metadata tabulate \
        --m-input-file ./dada2_stats.qza  \
        --o-visualization ./dada2_stats.qzv


## Feature table summary 

    qiime feature-table summarize \
        --i-table ./dada2_table.qza \
        --m-sample-metadata-file ./metadata.tsv \
        --o-visualization ./dada2_table.qzv

## Visulaize Representative Sequences

    qiime feature-table tabulate-seqs \
        --i-data ./dada2_rep_set.qza \
        --o-visualization ./dada2_rep_set.qzv

## Phylogenetic tree

    qiime phylogeny align-to-tree-mafft-fasttree \
        --i-sequences ./dada2_rep_set.qza \
        --o-alignment ./aligned-rep-seqs.qza \
        --o-masked-alignment ./masked-aligned-rep-seqs.qza \
        --o-tree ./unrooted-tree.qza \
        --o-rooted-tree ./rooted-tree.qza

## Exporting tree for later use:

    qiime tools export \
        --input-path ./rooted-tree.qza \
        --output-path ./exported-tree

## Taxonomic assignment:

    qiime feature-classifier classify-consensus-vsearch \
        --p-threads 8 \
        --i-query ./dada2_rep_set.qza \
        --i-reference-reads ./silva-138-99-seqs.qza \
        --i-reference-taxonomy ./silva-138-99-tax.qza \
        --p-perc-identity 0.99 \
        --o-classification ./vsearch_taxonomy.qza \
        --o-search-results ./top_hits.qza \
        --verbose

## visualization of taxonomy

    qiime metadata tabulate \
        --m-input-file ./vsearch_taxonomy.qza \
        --o-visualization ./vsearch_taxonomy.qzv \
        --verbose

## 

## Taxa barplot and Taxa Collapse

    qiime taxa barplot \
        --i-table ./dada2_table.qza \
        --i-taxonomy ./vsearch_taxonomy.qza \
        --m-metadata-file metadata.tsv \
        --o-visualization taxa-bar-plots.qzv \
        --verbose

    qiime taxa collapse \
        --i-table ./dada2_table.qza \
        --i-taxonomy ./vsearch_taxonomy.qza \
        --p-level 6 \
        --o-collapsed-table ./collapsed-table-l6.qza \
        --verbose

## Normalize counts to Relative Frequencies

    qiime feature-table relative-frequency \
        --i-table ./collapsed-table-l6.qza \
        --verbose \
        --o-relative-frequency-table ./relative--frequency-table.qza


## Diversity core-metrics-Phylogenetic

    qiime diversity core-metrics-phylogenetic \
        --p-sampling-depth 1000 \
        --i-phylogeny ./rooted-tree.qza \
        --i-table ./dada2_table.qza \
        --m-metadata-file ./metadata.tsv \
        --output-dir ./core-metrics-results \
        --p-n-jobs-or-threads 8 

## Feature table transpose

    qiime feature-table transpose \
        --i-table ./relative--frequency-table.qza \
        --o-transposed-feature-table ./transposed_relative_freq.qza \
        --verbose

## Metadata Tablulate

    qiime metadata tabulate \
        --m-input-file ./metadata.tsv \
        --o-visualization ./metadata_viz.qzv

