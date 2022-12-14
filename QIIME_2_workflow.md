# 1. Create directory in HOME

```
mkdir qiime-2-thesis-IDFG
cd qiime-2-thesis-IDFG
```

# 2. Create manifest file

### See Manifest.txt

Create a text file and populate two columns: sample-id and absolute-filepath.

# 3. Import into QIIME 2

### Used command for single-end reads and demultiplexed.
```
qiime tools import \
  --type 'SampleData[SequencesWithQuality]' \
  --input-path Manifest.txt \
  --output-path single-end-demux.qza \
  --input-format SingleEndFastqManifestPhred33V2
```

# 4. Visualize imported data

```
mkdir Visualizations

qiime demux summarize \
  --i-data single-end-demux.qza \
  --o-visualization Visualizations/single-end-demux.qzv

qiime tools view Visualizations/single-end-demux.qzv
```

#### or open https://view.qiime2.org and drag and drop QZV to visualize in web browser.

# 5. Quality control with DADA2

### After visualizing single-end-demux, chose a quality cutoff of 26, which allowed DADA2 to retain higher quality sequence data compared to lower values for quality cutoff. This cutoff corresponded to a p-trunc-len of 147.

```
qiime dada2 denoise-single \
  --i-demultiplexed-seqs single-end-demux.qza \
  --p-trunc-len 147 \
  --p-trim-left 0 \
  --o-representative-sequences rep-seqs-dada2.qza \
  --o-table table-dada2.qza \
  --o-denoising-stats stats-dada2.qza
```

# 6. Visualize feature table, sequences, and DADA2 statistics

```
qiime feature-table summarize \
  --i-table table-dada2.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization Visualizations/table-dada2.qzv

qiime feature-table tabulate-seqs \
  --i-data rep-seqs-dada2.qza \
  --o-visualization Visualizations/rep-seqs-dada2.qzv

qiime metadata tabulate \
  --m-input-file stats-dada2.qza \
  --o-visualization Visualizations/stats-dada2.qzv
```

# 7. Taxonomy - Train and run a feature classifier using SILVA database

### Convert SILVA 16S fasta to QZA
```
qiime tools import \
  --type 'FeatureData[Sequence]' \
  --input-path silva_138_99_16S.fna \
  --output-path silva-138-99-16S.qza
```
### Extract reference reads based on 515F/926R primers and previous p-trunc-len.
```
qiime feature-classifier extract-reads \
  --i-sequences silva-138-99-16S.qza \
  --p-f-primer GTGYCAGCMGCCGCGGTAA \
  --p-r-primer CCGYCAATTYMTTTRAGTTT \
  --p-trunc-len 147 \
  --p-min-length 100 \
  --p-max-length 400 \
  --o-reads silva-138-99-515f-926r-ref-seqs.qza
```

### Import taxonomy
```
qiime tools import \
  --type 'FeatureData[Taxonomy]' \
  --input-format HeaderlessTSVTaxonomyFormat \
  --input-path taxonomy_7_levels.txt \
  --output-path ref-taxonomy.qza
```

### Train the feature classifier
```
qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads silva-138-99-515f-926r-ref-seqs.qza \
  --i-reference-taxonomy silva-138-99-tax.qza \
  --o-classifier silva-138-99-515f-926r-classifier.qza
```

### Run the feature classifier
```
qiime feature-classifier classify-sklearn \
  --i-classifier silva-138-99-515f-926r-classifier.qza \
  --i-reads rep-seqs-dada2.qza \
  --o-classification taxonomy.qza
```

# 8. Filter out mitochondria, chloroplasts, and Unassigned bacteria

### Filter from table
```
qiime taxa filter-table \
  --i-table table-dada2.qza \
  --i-taxonomy taxonomy.qza \
  --p-exclude mitochondria,chloroplast,Unassigned \
  --o-filtered-table filtered-table.qza
```

### Visualize filtered table
```
qiime feature-table summarize \
  --i-table filtered-table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization Visualizations/filtered-table.qzv
```

### Filter from sequences
```
qiime taxa filter-seqs \
  --i-sequences rep-seqs-dada2.qza \
  --i-taxonomy taxonomy.qza \
  --p-exclude mitochondria,chloroplast,Unassigned \
  --o-filtered-table filtered-rep-seqs.qza
```

### Visualize filtered rep seqs
```
qiime feature-table tabulate-seqs \
  --i-data filtered-rep-seqs.qza \
  --o-visualization Visualizations/filtered-rep-seqs.qzv
```

# 9. Visualize taxonomy and make a relative abundance barplot

### Visualize taxonomy
```
qiime metadata tabulate \
  --m-input-file taxonomy.qza \
  --o-visualization Visualizations/taxonomy.qzv
```

### Make a taxa barplot displaying relative abundance (using filtered table)
```
qiime taxa barplot \
  --i-table filtered-table.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization Visualizations/taxa-bar-plots.qzv
```

# 10. Generate a tree for phylogenetic diversity analysis

```
qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences filtered-rep-seqs.qza \
  --o-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned-rep-seqs.qza \
  --o-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza
```

# 11. Make an alpha rarefaction plot

```
qiime diversity alpha-rarefaction \
  --i-table filtered-table.qza \
  --i-phylogeny rooted-tree.qza \
  --p-max-depth 25000 \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization Visualizations/alpha-rarefaction.qzv
```

### View the rarefaction plot and open filtered-table.qzv > Interactive Sample Detail. Choose a Sampling Depth with two considerations: 1) Captures most of the diversity in the rarefaction plot and 2) how many samples are retained in the analysis. 

# 12. Core metrics for Alpha and Beta diversity

```
qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree.qza \
  --i-table table-dada2.qza \
  --p-sampling-depth 12759 \
  --m-metadata-file sample-metadata.tsv \
  --output-dir core-metrics-results
```

# 13. Statistical significance for Alpha diversity

```
cd core-metrics-results

mkdir Alpha_diversity
```

### Shannon
```
qiime diversity alpha-group-significance \
  --i-alpha-diversity shannon_vector.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization Alpha_diversity/shannon_significance.qzv
```

### Observed OTUs
```
qiime diversity alpha-group-significance \
  --i-alpha-diversity observed_otus_vector.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization Alpha_diversity/observed_otus_significance.qzv
```

### Faith PD
```
qiime diversity alpha-group-significance \
  --i-alpha-diversity faith_pd_vector.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization Alpha_diversity/faith_pd_significance.qzv
```

### Evenness
```
qiime diversity alpha-group-significance \
  --i-alpha-diversity evenness_vector.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization Alpha_diversity/evenness_significance.qzv
```

# 14. Correlation analyses for Alpha diversity

### Shannon
```
qiime diversity alpha-correlation \
  --i-alpha-diversity shannon_vector.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization shannon_correlation_Spearman.qzv
```
 
### Observed OTUs
```
qiime diversity alpha-correlation \
  --i-alpha-diversity observed_otus_vector.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization observed_otus_correlation_Spearman.qzv
```

### Faith PD
```
qiime diversity alpha-correlation \
  --i-alpha-diversity faith_pd_vector.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization faith_pd_correlation_Spearman.qzv
```
 
### Evenness
```
qiime diversity alpha-correlation \
  --i-alpha-diversity evenness_vector.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization evenness_correlation_Spearman.qzv
```

# 15. Beta diversity PERMANOVA comparing infected and uninfected frogs (ChytridResult)

```
mkdir Beta_diversity
```

### Unweighted UniFrac 
```
qiime diversity beta-group-significance \
  --i-distance-matrix unweighted_unifrac_distance_matrix.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --m-metadata-column ChytridResult \
  --o-visualization Beta_diversity/unweighted_unifrac_significance_Chytrid.qzv \
  --p-pairwise
```

### Weighted UniFrac 
```
qiime diversity beta-group-significance \
  --i-distance-matrix weighted_unifrac_distance_matrix.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --m-metadata-column ChytridResult \
  --o-visualization Beta_diversity/weighted_unifrac_significance_Chytrid.qzv \
  --p-pairwise
```

# 16. Beta diversity Mantel test (ZoosporeEquivalents)

### Unweighted UniFrac
```
qiime diversity beta-correlation \
  --i-distance-matrix unweighted_unifrac_distance_matrix.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --m-metadata-column ZoosporeEquivalents \
  --p-intersect-ids \
  --o-metadata-distance-matrix Beta_diversity/unweighted_unifrac_correlation_zoospores.qza \
  --o-mantel-scatter-visualization Beta_diversity/unweighted_unifrac_correlation_zoospores.qzv
```

### Weighted UniFrac
```
qiime diversity beta-correlation \
  --i-distance-matrix unweighted_unifrac_distance_matrix.qza \
  --m-metadata-file ../sample-metadata.tsv \
  --m-metadata-column ZoosporeEquivalents \
  --p-intersect-ids \
  --o-metadata-distance-matrix Beta_diversity/unweighted_unifrac_correlation_zoospores.qza \
  --o-mantel-scatter-visualization Beta_diversity/unweighted_unifrac_correlation_zoospores.qzv
```

### Repeated above commands with latitude and longitude

# 17. Correlation analyses with Positive infection status samples only

### Run following steps and then repeat steps 13 and 15:
#### 1) Make a metadata file that includes only Positive samples
#### 2) Filter the feature table using that metadata file
#### 3) Rerun step 11 creating a separate core-metrics-results-positives

# 18. adonis analysis - PERMANOVA with multiple variables (infection, latitude, longitude)

### Unweighted UniFrac
```
qiime diversity adonis \
  --i-distance-matrix unweighted_unifrac_distance_matrix.qza \
  --p-formula "ChytridResult * Latitude * Longitude" \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization Beta_diversity/uwunifrac_adonis_chytrid_lat_long.qzv
```

### Weighted UniFrac
```
qiime diversity adonis \
  --i-distance-matrix weighted_unifrac_distance_matrix.qza \
  --p-formula "ChytridResult * Latitude * Longitude" \
  --m-metadata-file ../sample-metadata.tsv \
  --o-visualization Beta_diversity/wunifrac_adonis_chytrid_lat_long.qzv
```

# 19. Plot taxonomic relative abundance by infection status

```
cd ..
mkdir group_plot
```

### Makes a table grouping samples by infection status
#### Metadata file had two columns: SampleID (Positive and Negative) and SampleName (Infected and Uninfected)
```
qiime feature-table group \
  --i-table filtered-table.qza \
  --p-axis sample \
  --m-metadata-file sample-metadata.tsv \
  --m-metadata-column ChytridResult \
  --p-mode sum \
  --o-grouped-table group_plot/grouped-table.qza
```

### Makes taxa barplot from the grouped table
```
qiime taxa barplot \
  --i-table group_plot/grouped-table.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file group_plot/grouped-metadata.tsv \
  --o-visualization group_plot/grouped-taxa-bar-plots.qzv
```
