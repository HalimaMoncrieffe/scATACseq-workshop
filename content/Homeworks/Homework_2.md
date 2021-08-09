---
title: "Homework 2"
---

## Goals

1. Import counts from mouse E18 brain scATACseq (5K) (provided by 10X Genomics) in R and process them  

> Processed data for mouse E18 brain scATACseq can be obtained directly from [10X Genomics](https://support.10xgenomics.com/single-cell-atac/datasets/1.2.0/atac_v1_E18_brain_fresh_5k)

2. Compare `Signac` processing and `SingleCellExperiment` processing

3. Transfer annotations from mouse E18 brain scRNAseq (5K) to mouse E18 brain scATACseq

> Processed data for mouse E18 brain scRNAseq can be obtained directly from [10X Genomics](https://support.10xgenomics.com/single-cell-gene-expression/datasets/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K)

4. Find genes associated to DA peaks and run GO over-enrichment / gene set enrichment analysis  

## 1. Process mouse E18 brain 5K scATACseq data

### Download data 

scATACseq for mouse E18 5K brain can be downloaded from 
[10X Genomics](https://support.10xgenomics.com/single-cell-atac/datasets/1.2.0/atac_v1_E18_brain_fresh_5k). 

```sh
mkdir -p data/Homework_2/scATAC
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_analysis.tar.gz -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_analysis.tar.gz
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_filtered_peak_bc_matrix.h5 -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_filtered_peak_bc_matrix.h5
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_filtered_tf_bc_matrix.h5 -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_filtered_tf_bc_matrix.h5
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_fragments.tsv.gz -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_fragments.tsv.gz
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_fragments.tsv.gz.tbi -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_fragments.tsv.gz.tbi
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_peak_annotation.tsv -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_peak_annotation.tsv
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_peaks.bed -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_peaks.bed
curl https://cg.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_possorted_bam.bam -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_possorted_bam.bam
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_possorted_bam.bam.bai -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_possorted_bam.bam.bai
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_singlecell.csv -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_singlecell.csv
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_web_summary.html -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_web_summary.html
curl https://cf.10xgenomics.com/samples/cell-atac/1.2.0/atac_v1_E18_brain_fresh_5k/atac_v1_E18_brain_fresh_5k_cloupe.cloupe -o data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_cloupe.cloupe
```

### Import data in R 

```r
library(Signac)
library(Seurat)
library(tidyverse)
library(BiocParallel)
library(plyranges)

## -- Read brain ATAC-seq counts
brain_assay <- Signac::CreateChromatinAssay(
    counts = Seurat::Read10X_h5("data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_filtered_peak_bc_matrix.h5"),
    fragments = 'data/Homework_2/scATAC/atac_v1_E18_brain_fresh_5k_fragments.tsv.gz',
    sep = c(":", "-"),
    genome = "mm10",
    min.cells = 1
)

## -- Read metadata
metadata <- vroom::vroom(
    file = "data/Homework_2/atac_v1_adult_brain_fresh_5k_singlecell.csv",
    col_names = TRUE
) %>% 
    filter(barcode != 'NO_BARCODE') %>% 
    mutate(cell = barcode) %>%
    column_to_rownames('barcode') %>% 
    as.data.frame()

## -- Create Seurat object
brain <- Seurat::CreateSeuratObject(
    counts = brain_assay,
    assay = 'peaks',
    project = 'ATAC',
    meta.data = metadata
)

## -- Add genome annotations
Annotation(brain) <- AnnotationHub::AnnotationHub()[['AH49547']] %>% 
    filter(gene_type == 'protein_coding', type == 'gene') %>% 
    mutate(gene_biotype = gene_type)
```

### QCing data 

```r
## -- Nucleosome signal 
brain <- NucleosomeSignal(object = brain)
brain$nucleosome_group <- ifelse(brain$nucleosome_signal > 4, 'NS > 4', 'NS < 4')
range(brain$nucleosome_group)
table(brain$nucleosome_group)

## -- Frag. size distribution
frags <- vroom::vroom('/data/20210804_scATACseq-workshop/data/MouseBrain/atac_v1_adult_brain_fresh_5k_fragments.tsv.gz', col_names = FALSE) %>% 
    setNames(c('chr', 'start', 'stop', 'cell', 'cluster'))
frags_chr12 <- filter(frags, chr == 'chr12', start < 50000000, cluster %in% c(1:10))
x <- left_join(frags_chr12, dplyr::select(brain@meta.data, cell, nucleosome_group)) %>% 
    mutate(width = stop - start) 
p <- ggplot(x, aes(x = width)) + 
    geom_bar() + 
    facet_wrap(~nucleosome_group+cluster, scales = 'free', ncol = 10) + 
    xlim(c(0, 800))

## -- TSS enrichment
brain <- TSSEnrichment(brain, fast = TRUE)
brain$high.tss <- ifelse(brain$TSS.enrichment > 2, 'High', 'Low')
df <- frags_chr12 %>% 
    left_join(dplyr::select(brain@meta.data, cell, high.tss)) %>% # For each fragment, recover cell TSS enrich. status
    dplyr::rename(c('seqnames' = 'chr', 'end' = 'stop')) %>% 
    as_granges() %>% 
    join_overlap_left(plyranges::select(mm_promoters, prom_id)) %>% # Link each fragment to overlapping promoter
    as_tibble() %>% 
    drop_na(prom_id) %>% # Remove fragments which are not overlapping any promoter
    left_join(
        as_tibble(mm_promoters), by = c('prom_id')
    ) %>% 
    mutate(
        midprom = start.y + (end.y - start.y)/2, 
        distance = midprom - start.x
    ) %>%
    filter(high.tss == 'High') %>%
    count(distance)
p<- ggplot(df, aes(x = distance, y = n)) + 
    geom_col() + 
    geom_smooth(method = 'loess', span = 0.05) + 
    theme_minimal() + 
    theme(legend.position = 'none') + 
    labs(title = 'Fragment "cut" sites', x = 'Distance from TSS', y = '# of cut sites') + 
    xlim(c(-1000, 1000))
```

### Subsetting data

```r
brain$FRiP <- brain$peak_region_fragments / brain$passed_filters * 100
brain$FRiBl <- brain$blacklist_region_fragments / brain$peak_region_fragments
brain <- subset(
    brain,
    peak_region_fragments > 3000 & # Keep cells with high number of fragments in peaks
        peak_region_fragments < 100000 & # Remove cells with number of fragments in peaks too high
        FRiP > 50 & # Keep cells with high FRiP 
        FRiBl < 0.05 & # Keep cells with low FRiBl
        nucleosome_signal < 4 & # Remove cells with high nucleosome signal
        TSS.enrichment > 2 # Keep cells with good TSSES
)
brain <- brain[
    rowSums(GetAssayData(brain, slot = "counts")) > 10 & 
    rowSums(GetAssayData(brain, slot = "counts") > 0) > 10, 
]
```

### Normalizing and reduce dimensionality data 

```r
brain <- Signac::RunTFIDF(brain) # Normalize data
brain <- Signac::FindTopFeatures(brain, min.cutoff = 'q0') # Find variable features
brain <- Signac::RunSVD(object = brain) # Perform dimensionality reduction
```

### Clustering 

Now that cells are embedded in a lower dimensional space, they can be graph-clustered just like "regular" cells from scRNAseq. 

```r
brain <- FindNeighbors(object = brain, reduction = 'lsi', dims = 2:30)
brain <- FindClusters(object = brain, algorithm = 3, resolution = 1.2, verbose = FALSE)
brain <- RunUMAP(brain, reduction = 'lsi', dims = 2:30)
Seurat::DimPlot(brain, group.by = 'ident', label = 'ident')
```

## 2. Compare Signac processing and SingleCellExperiment processing



## 3. Integrate scATACseq and scRNAseq 

### Download data 

scRNAseq for mouse E18 5K brain can be downloaded from 
[10X Genomics](https://support.10xgenomics.com/single-cell-gene-expression/datasets/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K). 

```sh
mkdir data/Homework_2/scRNA
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_web_summary.html -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_web_summary.html
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_metrics_summary.csv -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_metrics_summary.csv
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_count_cloupe.cloupe -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_count_cloupe.cloupe
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_count_sample_alignments.bam -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_count_sample_alignments.bam
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_count_sample_alignments.bam.bai -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_count_sample_alignments.bam.bai
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_count_sample_barcodes.csv -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_count_sample_barcodes.csv
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_count_sample_feature_bc_matrix.h5 -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_count_sample_feature_bc_matrix.h5
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_count_sample_feature_bc_matrix.tar.gz -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_count_sample_feature_bc_matrix.tar.gz
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_count_sample_molecule_info.h5 -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_count_sample_molecule_info.h5
curl https://cf.10xgenomics.com/samples/cell-exp/6.0.0/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K/SC3_v3_NextGem_DI_Neurons_5K_SC3_v3_NextGem_DI_Neurons_5K_count_analysis.tar.gz -o data/Homework_2/scRNA/SC3_v3_Neurons_5K_count_analysis.tar.gz
```