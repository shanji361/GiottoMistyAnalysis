---
title: "Xenium Human Lung Cancer Analysis with MISTy"
output: 
  html_document:
    number_sections: true
    toc: true
pkgdown:
  as_is: true
vignette: >
  %\VignetteIndexEntry{Xenium Human Lung Cancer Analysis with MISTy}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}
---
# 1. Summary
This tutorial analyzes a Xenium-derived human lung cancer dataset using Giotto and MISTy to identify cell types, estimating pathway activities, and modeling spatial cell–cell interactions to gain insights into the tumor microenvironment. 

# 2. Environment Setup and Library Initialization
**Setting up the working environment and loading necessary libraries for spatial transcriptomics analysis.**

```{r, eval=FALSE}
# Configure Python environment for Giotto
library(reticulate)
use_python("C:/Users/your_username/anaconda3/envs/giotto_env/python.exe", required = TRUE)

# Load core libraries
library(Giotto)
library(mistyR)

# Parallel processing
library(future)
plan(multisession)

# Data manipulation libraries
library(data.table)
library(janitor)
library(Matrix)
library(dplyr)
library(tidyverse)

# Pathway analysis libraries
library(msigdbr)
library(decoupleR)
library(SummarizedExperiment)
```
# 3. Data Download and Setup
## 3.1 Configure Paths and Download Data
Setting up data paths and downloading the Xenium lung cancer dataset.
```{r, eval=FALSE}
# Set up paths
data_path <- "/Users/your_username/data/xenium_lung"
save_dir <- '/Users/your_username/results/xenium_lung'
dir.create(save_dir, recursive = TRUE)

# Download the mini dataset
options("timeout" = Inf)
download.file(
  url = "https://zenodo.org/records/13207308/files/workshop_xenium.zip?download=1",
  destfile = file.path(save_dir, "workshop_xenium.zip"),
  mode = "wb",           # Force binary mode
  method = "libcurl",    # Use libcurl method
  timeout = 300          # Set longer timeout
)

# Extract the downloaded data
unzip(file.path(save_dir, "workshop_xenium.zip"), 
      exdir = data_path)
```
**Note:** This is a subset of the full dataset. 

```{r, eval=FALSE}
- Full: -16.039, 12342.984, -3511.515, -294.455 (xmin, xmax, ymin, ymax)
- Mini: 6000,    7000,      -2200,     -1400    (xmin, xmax, ymin, ymax)
```

![H&E Image](he.png)


# 4. Giotto Object Creation 

## Prerequisite Steps (Omitted)

> **Note:** The following preprocessing steps are **omitted here** for brevity, as they are already covered in [the Giotto 2024 workshop tutorial (sections 10.1 to 10.7)](https://drieslab.github.io/giotto_workshop_2024/xenium-1.html#aggregate-analyses-workflow):

<details>
<summary>Click to expand the omitted preprocessing steps</summary>

- **Creating the Giotto object:** Initialized using `createGiottoXeniumObject()` from the raw Xenium directory.
- **Setting analysis instructions:** Configured save directory and plot-saving behavior via the `instructions()` function.
- **Adding spatial information:** Added centroid positions for `"cell"` and `"nucleus"` polygon types with `addSpatialCentroidLocations()`.
- **Spatial visualization:** Generated basic spatial plots using `spatInSituPlotPoints()` with and without image overlays.
- **Manual data import:** Loaded raw transcript and expression data using `importXenium()`, and adjusted the expression file type to `"mtx"`.
- **Metadata and expression integration:** Loaded and assigned expression matrices and cell metadata to the Giotto object using `setGiotto()`.
- **Image processing:** Loaded multiple morphology image channels using `createGiottoLargeImageList()`, adjusted contrast, and attached them to the object.
- **Quality control and normalization:** Computed overlaps, filtered low-quality data, normalized the expression matrix, and added per-cell statistics.
- **Dimensionality reduction and clustering:** Ran PCA, UMAP, and Leiden clustering, followed by visualization of the clusters in 2D and 3D plots.

</details>

We now continue from the step of identifying **cluster-specific marker genes** using the **Scran** method.


# 5. Marker Gene Analysis
## 5.1 Find Cluster-Specific Markers
```{r, eval=FALSE}
# Find marker genes using scran method
res_scran <- findMarkers_one_vs_all(g, 
                                    cluster_column = "leiden_clus", 
                                    method = "scran",
                                    expression_values = "normalized"
)

# Get top 2 genes per cluster
topgenes_scran <- res_scran[, head(.SD, 2), by = 'cluster']

# Create violin plots for top marker genes
violinPlot(g, 
           feats = topgenes_scran$feats, 
           cluster_column = "leiden_clus"
)
``` 

![Violin Plot](10-violinPlot.png)


# 6. Cell Type Annotation
## 6.1 Manual Cell Type Assignment
Based on marker gene analysis, manually assign cell types to clusters.

```{r, eval=FALSE}
# Define cell types based on marker gene analysis
cell_types <- c(
  "CD8+ T cell",        # CD8A, CD3D, CD3E, CD2, GZMK, CTLA4
  "Alveolar epithelial cell",  # EPCAM, SFTA2, KRT7, GPRC5A, EGFR
  "Fibroblast",         # PDGFRA, FBN1, LTBP2, COL5A2, VCAN
  "Macrophage",         # CD68, CD14, MS4A4A, CD163, MRC1
  "B cell",             # MS4A1, CD19, CD79A, BANK1, IRF8
  "Ciliated cell",      # FOXJ1, DNAAF1, CFAP53, CCDC39, SOX2
  "Endothelial cell",   # VWF, PECAM1, CD34, CLEC14A, EGFL7
  "Basal epithelial cell",  # SOX2, CYP4B1, TSPAN19, MET, EHF
  "Club cell",          # GPRC5A, CYP2B6, CFTR, ADAM28, AGR3
  "Smooth muscle cell", # ACTA2, MYH11, CNN1, MYLK, DES
  "Plasma cell",        # MZB1, SLAMF7, PRDM1, CD27, TNFRSF17
  "Mast cell"           # KIT, CPA3, MS4A2, HPGDS, IL1RL1
)

# Assign names to cell types vector
names(cell_types) <- 1:length(cell_types)

# Annotate Giotto object with cell types
g <- annotateGiotto(gobject = g, 
                    spat_unit = "cell", 
                    annotation_vector = cell_types,
                    cluster_column = 'leiden_clus', 
                    name = 'subannot_clus')

# Store annotated object
xenium_lungcancer_test <- g
```
# 7. Pathway Activity Analysis
## 7.1 Prepare Data for Pathway Analysis
```{r, eval=FALSE}
# Extract raw expression matrix
raw_matrix <- xenium_lungcancer_test@expression$cell$rna$raw[]
symbols <- rownames(raw_matrix)

# Check for duplicate gene symbols
d.index <- which(duplicated(symbols))
print(d.index) # no duplicates found

# Extract spatial coordinates
geometry <- getSpatialLocations(
  gobject = xenium_lungcancer_test, 
  spat_unit = "cell",
  output = "data.table"
)

#MISTy expects the spatial coordinate columns to be named row and col, not sdimx and sdimy
geometry <- geometry[, c("sdimx", "sdimy")]
setnames(geometry, c("sdimx", "sdimy"), c("row", "col"))

# Prepare expression matrix for pathway analysis
expression <- as.matrix(xenium_lungcancer_test@expression$cell$rna$raw[])
```
## 7.2 Run PROGENy Pathway Analysis

```{r, eval=FALSE}
# Get PROGENy model for human
model <- get_progeny(organism = "human", top = 500)

# Estimate pathway activity using multivariate linear model
est_path_act <- run_mlm(expression, model, .mor = NULL) 

# Convert to wide format for downstream analysis
est_path_act_wide <- est_path_act %>% 
  pivot_wider(id_cols = condition, names_from = source, values_from = score) %>%
  column_to_rownames("condition") 

# Clean column names
colnames(est_path_act_wide) <- est_path_act_wide %>% 
  clean_names(parsing_option = 0) %>% 
  colnames(.)

# Add pathway results to Giotto object
xenium_lungcancer_test@expression[["cell"]][["rna"]][["progeny"]] <- t(est_path_act_wide)
```

## 7.3 Prepare Cell Type Composition Matrix

```{r, eval=FALSE}
# Extract cell type metadata
metadata <- getCellMetadata(g, spat_unit = "cell")
cell_types <- metadata$subannot_clus

# Create one-hot encoded matrix for cell types
cell_type_factor <- factor(cell_types)
cell_type_onehot <- model.matrix(~ cell_type_factor - 1)
colnames(cell_type_onehot) <- gsub("cell_type_factor", "", colnames(cell_type_onehot))

# Set proper row names using cell IDs
actual_cell_ids <- colnames(xenium_lungcancer_test@expression$cell$rna$raw)
rownames(cell_type_onehot) <- actual_cell_ids

# Clean column names
colnames(cell_type_onehot) <- cell_type_onehot %>% 
  as_tibble() %>%
  clean_names(parsing_option = 0) %>% 
  colnames(.)

# Convert to tibble for MISTy
composition_xenium <- as_tibble(cell_type_onehot)
```
# 8. MISTy Analysis
## 8.1 Create MISTy Views
Setting up different spatial views for MISTy analysis.

```{r, eval=FALSE}
# Create intraview from cell-type composition
comp_views <- create_initial_view(composition_xenium) 

# Create juxtaview and paraview from pathway activity
path_act_views <- create_initial_view(est_path_act_wide) %>%
  add_juxtaview(geometry, neighbor.thr = 20) %>% 
  add_paraview(geometry, l = 50, family = "gaussian")

# Combine composition and pathway activity views
com_path_act_views <- comp_views %>%
  add_views(create_view("juxtaview.path.20", 
                        path_act_views[["juxtaview.20"]]$data, 
                        "juxta.path.20")) %>% 
  add_views(create_view("paraview.path.50", 
                        path_act_views[["paraview.50"]]$data, 
                        "para.path.50"))
```
## 8.2 Run MISTy Analysis
```{r, eval=FALSE}
# Run MISTy analysis
run_misty(com_path_act_views, "result/xenium_lung/comp_path_act")

# Collect results
misty_results_com_path_act <- collect_results("result/xenium_lung/comp_path_act/")

# Plot improvement statistics
misty_results_com_path_act %>%
  plot_improvement_stats("intra.R2") %>%
  plot_improvement_stats("gain.R2") 
```
![IntraGain](com_path_act_IntraGain.png)

```{r, eval=FALSE}

# Plot view contributions
misty_results_com_path_act %>% 
  plot_view_contributions()
```
![Contributions](com_path_act_Contributions.png)

```{r, eval=FALSE}
# Plot interaction heatmap
misty_results_com_path_act %>%
  plot_interaction_heatmap("juxta.path.20", clean = TRUE)
```
![Juxta](com_path_act_Juxta.png)

## 8.3 Alternative MISTy Analysis (Bypass Intra)
Running MISTy analysis without intraview self-prediction.

```{r, eval=FALSE}
# Run MISTy bypassing intraview self-prediction
run_misty(com_path_act_views, "result/xenium_lung", 
          model.function = linear_model, bypass.intra = TRUE)

# Collect and visualize results
misty_results_com_path_act_linear <- collect_results("result/xenium_lung")

misty_results_com_path_act_linear %>%
  plot_improvement_stats("gain.R2") %>%
  plot_view_contributions()
```
![Linear Gain](com_path_act_linear_Gain.png)
```{r, eval=FALSE}
misty_results_com_path_act_linear %>%
  plot_interaction_heatmap("juxta.path.20", clean = TRUE) 
```
![Linear Juxta](com_path_act_linear_Juxta.png)
# 9. Visualization and Results
## 9.1 Add Normalized Pathway Data to Giotto Object
```{r, eval=FALSE}
# Add raw pathway data
xenium_lungcancer_test@expression[["cell"]][["progeny"]][["raw"]] <- t(est_path_act_wide)

# Create and add normalized pathway data (z-score normalization)
est_path_act_normalized <- scale(est_path_act_wide)
xenium_lungcancer_test@expression[["cell"]][["progeny"]][["normalized"]] <- t(est_path_act_normalized)
```
## 9.2 Spatial Visualization
### Cell Type Distribution
```{r, eval=FALSE}
# Plot spatial distribution of cell types
spatPlot2D(xenium_lungcancer_test,
           spat_unit = "cell", 
           cell_color = "subannot_clus",
           show_image = TRUE,
           point_size = 1.1,
           point_alpha = 0.8, 
           cell_color_gradient = c("blue", "orange"))
```
![2D Spatial Plot 11](11-spatPlot2D.png)
### Pathway Activity Visualization
```{r, eval=FALSE}
# Plot TRAIL pathway activity
spatFeatPlot2D(xenium_lungcancer_test,
               spat_unit = "cell", 
               feat_type = "progeny",
               feats = c("trail"),
               show_image = TRUE,
               point_size = 1.3,
               cow_n_col = 1, 
               cell_color_gradient = c("blue", "red"))
```
![2D Spatial Feature Plot 12](12-spatFeatPlot2D.png)
```{r, eval=FALSE}
# Highlight specific cell types (e.g., B cells)
spatPlot2D(xenium_lungcancer_test,
           spat_unit = "cell",
           cell_color = "subannot_clus",
           select_cell_groups = "B cell", 
           show_image = TRUE,
           point_size = 1.3)
```
![2D Spatial Plot 13](13-spatPlot2D.png)
```{r, eval=FALSE}
# Plot VEGF and WNT pathways
spatFeatPlot2D(xenium_lungcancer_test,
               spat_unit = "cell", 
               feat_type = "progeny",
               feats = c("vegf", "wnt"),
               show_image = TRUE,
               point_size = 0.8,
               cow_n_col = 2,  
               cell_color_gradient = c("blue", "green"))
```
![2D Spatial Feature Plot 14](14-spatFeatPlot2D.png)
```{r, eval=FALSE}
# Plot TNF-alpha and TRAIL pathways
spatFeatPlot2D(xenium_lungcancer_test,
               spat_unit = "cell", 
               feat_type = "progeny",
               feats = c("tnfa", "trail"),
               show_image = TRUE,
               point_size = 0.8,
               cow_n_col = 2,  
               cell_color_gradient = c("blue", "green"))
```
![2D Spatial Feature Plot 15](15-spatFeatPlot2D.png)
```{r, eval=FALSE}
# Plot TGF-beta and Androgen pathways
spatFeatPlot2D(xenium_lungcancer_test,
               spat_unit = "cell", 
               feat_type = "progeny",
               feats = c("tgfb", "androgen"),
               show_image = TRUE,
               point_size = 0.8,
               cow_n_col = 2,  
               cell_color_gradient = c("blue", "green"))
```
![2D Spatial Feature Plot 16](16-spatFeatPlot2D.png)
```{r, eval=FALSE}
# Plot NF-kB and p53 pathways
spatFeatPlot2D(xenium_lungcancer_test,
               spat_unit = "cell", 
               feat_type = "progeny",
               feats = c("nfkb", "p53"),
               show_image = TRUE,
               point_size = 0.8,
               cow_n_col = 2,  
               cell_color_gradient = c("blue", "green"))
```
![2D Spatial Feature Plot 17](17-spatFeatPlot2D.png)
```{r, eval=FALSE}
# Plot EGFR and Estrogen pathways
spatFeatPlot2D(xenium_lungcancer_test,
               spat_unit = "cell", 
               feat_type = "progeny",
               feats = c("egfr", "estrogen"),
               show_image = TRUE,
               point_size = 0.8,
               cow_n_col = 2,  
               cell_color_gradient = c("blue", "green"))
```
![2D Spatial Feature Plot 18](18-spatFeatPlot2D.png)
```{r, eval=FALSE}
devtools::session_info()
```

```
Session info 
 setting value
 version  R version 4.4.1 (2024-06-14 ucrt)
 os       Windows 11 x64 (build 26100)
 system   x86_64, mingw32
 ui       RStudio
 language (EN)
 collate  English_United States.utf8
 ctype    English_United States.utf8
 tz       America/New_York
 date     2025-07-10
 rstudio  2024.09.0+375 Cranberry Hibiscus (desktop)
 pandoc   3.2 @ C:/Program Files/RStudio/resources/app/bin/quarto/bin/tools/ (via rmarkdown)

Packages 
 package             * version date (UTC) lib source
 abind                  1.4-8    2024-09-12 [1] CRAN (R 4.4.1)
 arrow                  18.0.0   2024-10-28 [1] https://apache.r-universe.dev (R 4.4.1)
 assertthat             0.2.1    2019-03-21 [1] CRAN (R 4.4.2)
 babelgene              22.9     2022-09-29 [1] CRAN (R 4.4.2)
 backports              1.5.0    2024-05-23 [1] CRAN (R 4.4.0)
 beachmat               2.20.0   2024-05-01 [1] Bioconduc~
 Biobase              * 2.66.0   2024-10-29 [1] Bioconduc~
 BiocGenerics         * 0.52.0   2024-10-29 [1] Bioconduc~
 BiocNeighbors          2.0.0    2024-10-29 [1] Bioconduc~
 BiocParallel           1.38.0   2024-05-01 [1] Bioconduc~
 BiocSingular           1.20.0   2024-05-01 [1] Bioconduc~
 bit                    4.5.0    2024-09-20 [1] CRAN (R 4.4.2)
 bit64                  4.5.2    2024-09-22 [1] CRAN (R 4.4.2)
 bluster                1.16.0   2024-10-29 [1] Bioconduc~
 cachem                 1.1.0    2024-05-16 [1] CRAN (R 4.4.1)
 cellranger             1.1.0    2016-07-27 [1] CRAN (R 4.4.2)
 checkmate              2.3.2    2024-07-29 [1] CRAN (R 4.4.1)
 cli                    3.6.3    2024-06-21 [1] CRAN (R 4.4.1)
 cluster                2.1.6    2023-12-01 [2] CRAN (R 4.4.1)
 codetools              0.2-20   2024-03-31 [2] CRAN (R 4.4.1)
 colorRamp2             0.1.0    2022-12-21 [1] CRAN (R 4.4.1)
 colorspace             2.1-1    2024-07-26 [1] CRAN (R 4.4.1)
 cowplot                1.1.3    2024-01-22 [1] CRAN (R 4.4.1)
 crayon                 1.5.3    2024-06-20 [1] CRAN (R 4.4.1)
 crosstalk              1.2.1    2023-11-23 [1] CRAN (R 4.4.1)
 curl                   5.2.3    2024-09-20 [1] CRAN (R 4.4.1)
 data.table           * 1.16.0   2024-08-27 [1] CRAN (R 4.4.1)
 dbscan                 1.2-0    2024-06-28 [1] CRAN (R 4.4.1)
 decoupleR            * 2.10.0   2024-06-16 [1] Bioconductor 3.19 (R 4.4.0)
 DelayedArray           0.32.0   2024-10-29 [1] Bioconduc~
 deldir                 2.0-4    2024-02-28 [1] CRAN (R 4.4.0)
 devtools               2.4.5    2022-10-11 [1] CRAN (R 4.4.2)
 digest                 0.6.37   2024-08-19 [1] CRAN (R 4.4.1)
 distances              0.1.11   2024-07-31 [1] CRAN (R 4.4.2)
 dplyr                * 1.1.4    2023-11-17 [1] CRAN (R 4.4.1)
 dqrng                  0.4.1    2024-05-28 [1] CRAN (R 4.4.2)
 edgeR                  4.4.0    2024-10-29 [1] Bioconduc~
 ellipsis               0.3.2    2021-04-29 [1] CRAN (R 4.4.1)
 evaluate               1.0.0    2024-09-17 [1] CRAN (R 4.4.1)
 fansi                  1.0.6    2023-12-08 [1] CRAN (R 4.4.1)
 farver                 2.1.2    2024-05-13 [1] CRAN (R 4.4.1)
 fastmap                1.2.0    2024-05-15 [1] CRAN (R 4.4.1)
 filelock               1.0.3    2023-12-11 [1] CRAN (R 4.4.2)
 forcats              * 1.0.0    2023-01-29 [1] CRAN (R 4.4.2)
 fs                     1.6.4    2024-04-25 [1] CRAN (R 4.4.1)
 furrr                  0.3.1    2022-08-15 [1] CRAN (R 4.4.2)
 future               * 1.34.0   2024-07-29 [1] CRAN (R 4.4.2)
 future.apply           1.11.3   2024-10-27 [1] CRAN (R 4.4.2)
 generics               0.1.3    2022-07-05 [1] CRAN (R 4.4.1)
 GenomeInfoDb         * 1.42.0   2024-10-29 [1] Bioconduc~
 GenomeInfoDbData       1.2.13   2024-11-08 [1] Bioconductor
 GenomicRanges        * 1.58.0   2024-10-29 [1] Bioconduc~
 ggplot2              * 3.5.1    2024-04-23 [1] CRAN (R 4.4.1)
 ggrepel                0.9.6    2024-09-07 [1] CRAN (R 4.4.1)
 Giotto               * 4.2.1    2025-02-18 [1] Github (drieslab/Giotto@7a5ac04)
 GiottoClass          * 0.4.7    2025-02-18 [1] Github (drieslab/GiottoClass@2ad48fa)
 GiottoUtils            0.2.4    2025-02-18 [1] Github (drieslab/GiottoUtils@f2e0aab)
 GiottoVisuals          0.2.12   2025-02-18 [1] Github (drieslab/GiottoVisuals@6d3d44a)
 globals                0.16.3   2024-03-08 [1] CRAN (R 4.4.0)
 glue                   1.8.0    2024-09-30 [1] CRAN (R 4.4.1)
 gridExtra              2.3      2017-09-09 [1] CRAN (R 4.4.2)
 gtable                 0.3.5    2024-04-22 [1] CRAN (R 4.4.1)
 gtools                 3.9.5    2023-11-20 [1] CRAN (R 4.4.1)
 hms                    1.1.3    2023-03-21 [1] CRAN (R 4.4.2)
 htmltools              0.5.8.1  2024-04-04 [1] CRAN (R 4.4.1)
 htmlwidgets            1.6.4    2023-12-06 [1] CRAN (R 4.4.1)
 httpuv                 1.6.15   2024-03-26 [1] CRAN (R 4.4.1)
 httr                   1.4.7    2023-08-15 [1] CRAN (R 4.4.1)
 igraph                 2.0.3    2024-03-13 [1] CRAN (R 4.4.1)
 IRanges              * 2.38.1   2024-07-03 [1] Bioconduc~
 irlba                  2.3.5.1  2022-10-03 [1] CRAN (R 4.4.1)
 janitor              * 2.2.1    2024-12-22 [1] CRAN (R 4.4.2)
 jsonlite               1.8.9    2024-09-20 [1] CRAN (R 4.4.1)
 knitr                  1.48     2024-07-07 [1] CRAN (R 4.4.1)
 labeling               0.4.3    2023-08-29 [1] CRAN (R 4.4.0)
 later                  1.3.2    2023-12-06 [1] CRAN (R 4.4.1)
 lattice                0.22-6   2024-03-20 [2] CRAN (R 4.4.1)
 lazyeval               0.2.2    2019-03-15 [1] CRAN (R 4.4.1)
 lifecycle              1.0.4    2023-11-07 [1] CRAN (R 4.4.1)
 limma                  3.62.1   2024-11-03 [1] Bioconduc~
 listenv                0.9.1    2024-01-29 [1] CRAN (R 4.4.2)
 locfit                 1.5-9.10 2024-06-24 [1] CRAN (R 4.4.2)
 logger                 0.4.0    2024-10-22 [1] CRAN (R 4.4.2)
 lubridate            * 1.9.3    2023-09-27 [1] CRAN (R 4.4.2)
 magick                 2.8.5    2024-09-20 [1] CRAN (R 4.4.2)
 magrittr               2.0.3    2022-03-30 [1] CRAN (R 4.4.1)
 MASS                   7.3-60.2 2024-04-26 [2] CRAN (R 4.4.1)
 Matrix               * 1.7-0    2024-04-26 [2] CRAN (R 4.4.1)
 MatrixGenerics       * 1.16.0   2024-05-01 [1] Bioconduc~
 matrixStats          * 1.4.1    2024-09-08 [1] CRAN (R 4.4.1)
 memoise                2.0.1    2021-11-26 [1] CRAN (R 4.4.1)
 metapod                1.14.0   2024-10-29 [1] Bioconduc~
 mime                   0.12     2021-09-28 [1] CRAN (R 4.4.0)
 miniUI                 0.1.1.1  2018-05-18 [1] CRAN (R 4.4.1)
 mistyR               * 1.12.0   2024-05-01 [1] Bioconductor 3.19 (R 4.4.0)
 msigdbr              * 7.5.1    2022-03-30 [1] CRAN (R 4.4.2)
 munsell                0.5.1    2024-04-01 [1] CRAN (R 4.4.1)
 OmnipathR              3.12.4   2024-10-02 [1] Bioconductor 3.19 (R 4.4.1)
 parallelly             1.38.0   2024-07-27 [1] CRAN (R 4.4.1)
 pillar                 1.9.0    2023-03-22 [1] CRAN (R 4.4.1)
 pkgbuild               1.4.4    2024-03-17 [1] CRAN (R 4.4.1)
 pkgconfig              2.0.3    2019-09-22 [1] CRAN (R 4.4.1)
 pkgload                1.4.0    2024-06-28 [1] CRAN (R 4.4.1)
 plotly                 4.10.4   2024-01-13 [1] CRAN (R 4.4.1)
 plyr                   1.8.9    2023-10-02 [1] CRAN (R 4.4.1)
 png                    0.1-8    2022-11-29 [1] CRAN (R 4.4.0)
 prettyunits            1.2.0    2023-09-24 [1] CRAN (R 4.4.1)
 profvis                0.4.0    2024-09-20 [1] CRAN (R 4.4.1)
 progeny                1.26.0   2024-05-01 [1] Bioconductor 3.19 (R 4.4.0)
 progress               1.2.3    2023-12-06 [1] CRAN (R 4.4.2)
 progressr              0.14.0   2023-08-10 [1] CRAN (R 4.4.1)
 promises               1.3.0    2024-04-05 [1] CRAN (R 4.4.1)
 purrr                * 1.0.2    2023-08-10 [1] CRAN (R 4.4.1)
 R.methodsS3            1.8.2    2022-06-13 [1] CRAN (R 4.4.0)
 R.oo                   1.26.0   2024-01-24 [1] CRAN (R 4.4.0)
 R.utils                2.12.3   2023-11-18 [1] CRAN (R 4.4.2)
 R6                     2.5.1    2021-08-19 [1] CRAN (R 4.4.1)
 ragg                   1.3.3    2024-09-11 [1] CRAN (R 4.4.1)
 rappdirs               0.3.3    2021-01-31 [1] CRAN (R 4.4.1)
 RColorBrewer           1.1-3    2022-04-03 [1] CRAN (R 4.4.0)
 Rcpp                   1.0.13-1 2024-11-02 [1] CRAN (R 4.4.1)
 RcppAnnoy              0.0.22   2024-01-23 [1] CRAN (R 4.4.1)
 readr                * 2.1.5    2024-01-10 [1] CRAN (R 4.4.2)
 readxl                 1.4.3    2023-07-06 [1] CRAN (R 4.4.2)
 remotes                2.5.0    2024-03-17 [1] CRAN (R 4.4.1)
 reshape2               1.4.4    2020-04-09 [1] CRAN (R 4.4.1)
 reticulate           * 1.39.0   2024-09-05 [1] CRAN (R 4.4.1)
 rex                    1.2.1    2021-11-26 [1] CRAN (R 4.4.3)
 rjson                  0.2.23   2024-09-16 [1] CRAN (R 4.4.1)
 rlang                  1.1.4    2024-06-04 [1] CRAN (R 4.4.1)
 rlist                  0.4.6.2  2021-09-03 [1] CRAN (R 4.4.2)
 rmarkdown              2.28     2024-08-17 [1] CRAN (R 4.4.1)
 rstudioapi             0.16.0   2024-03-24 [1] CRAN (R 4.4.1)
 rsvd                   1.0.5    2021-04-16 [1] CRAN (R 4.4.1)
 rvest                  1.0.4    2024-02-12 [1] CRAN (R 4.4.2)
 S4Arrays               1.6.0    2024-10-29 [1] Bioconduc~
 S4Vectors            * 0.44.0   2024-10-29 [1] Bioconduc~
 ScaledMatrix           1.12.0   2024-05-01 [1] Bioconduc~
 scales                 1.3.0    2023-11-28 [1] CRAN (R 4.4.1)
 scattermore            1.2      2023-06-12 [1] CRAN (R 4.4.1)
 scran                  1.34.0   2024-10-29 [1] Bioconduc~
 sctransform            0.4.1    2023-10-19 [1] CRAN (R 4.4.2)
 scuttle                1.16.0   2024-10-29 [1] Bioconduc~
 selectr                0.4-2    2019-11-20 [1] CRAN (R 4.4.2)
 sessioninfo            1.2.2    2021-12-06 [1] CRAN (R 4.4.1)
 shiny                  1.9.1    2024-08-01 [1] CRAN (R 4.4.1)
 SingleCellExperiment   1.28.0   2024-10-29 [1] Bioconduc~
 snakecase              0.11.1   2023-08-27 [1] CRAN (R 4.4.2)
 SparseArray            1.6.0    2024-10-29 [1] Bioconduc~
 SpatialExperiment      1.16.0   2024-10-29 [1] Bioconduc~
 statmod                1.5.0    2023-01-06 [1] CRAN (R 4.4.1)
 stringi                1.8.4    2024-05-06 [1] CRAN (R 4.4.0)
 stringr              * 1.5.1    2023-11-14 [1] CRAN (R 4.4.1)
 SummarizedExperiment * 1.36.0   2024-10-29 [1] Bioconduc~
 systemfonts            1.1.0    2024-05-15 [1] CRAN (R 4.4.1)
 terra                  1.8-0    2024-11-08 [1] Github (rspatial/terra@b0f2224)
 textshaping            0.4.0    2024-05-24 [1] CRAN (R 4.4.1)
 tibble               * 3.2.1    2023-03-20 [1] CRAN (R 4.4.1)
 tidyr                * 1.3.1    2024-01-24 [1] CRAN (R 4.4.1)
 tidyselect             1.2.1    2024-03-11 [1] CRAN (R 4.4.1)
 tidyverse            * 2.0.0    2023-02-22 [1] CRAN (R 4.4.2)
 timechange             0.3.0    2024-01-18 [1] CRAN (R 4.4.2)
 tzdb                   0.4.0    2023-05-12 [1] CRAN (R 4.4.2)
 UCSC.utils             1.2.0    2024-10-29 [1] Bioconduc~
 urlchecker             1.0.1    2021-11-30 [1] CRAN (R 4.4.1)
 usethis                3.0.0    2024-07-29 [1] CRAN (R 4.4.1)
 utf8                   1.2.4    2023-10-22 [1] CRAN (R 4.4.1)
 uwot                   0.2.2    2024-04-21 [1] CRAN (R 4.4.1)
 vctrs                  0.6.5    2023-12-01 [1] CRAN (R 4.4.1)
 viridisLite            0.4.2    2023-05-02 [1] CRAN (R 4.4.1)
 vroom                  1.6.5    2023-12-05 [1] CRAN (R 4.4.2)
 withr                  3.0.1    2024-07-31 [1] CRAN (R 4.4.1)
 xfun                   0.48     2024-10-03 [1] CRAN (R 4.4.1)
 xml2                   1.3.6    2023-12-04 [1] CRAN (R 4.4.1)
 xmlparsedata           1.0.5    2021-03-06 [1] CRAN (R 4.4.3)
 xtable                 1.8-4    2019-04-21 [1] CRAN (R 4.4.1)
 XVector                0.44.0   2024-05-01 [1] Bioconduc~
 yaml                   2.3.10   2024-07-26 [1] CRAN (R 4.4.1)
 zlibbioc               1.50.0   2024-05-01 [1] Bioconduc~

 [1] C:/Users/your_username/AppData/Local/R/win-library/4.4
 [2] C:/Program Files/R/R-4.4.1/library

Python configuration 
 python:         C:/Users/your_username/anaconda3/envs/giotto_env/python.exe
 libpython:      C:/Users/your_username/anaconda3/envs/giotto_env/python310.dll
 pythonhome:     C:/Users/your_username/anaconda3/envs/giotto_env
 version:        3.10.2 | packaged by conda-forge | (main, Mar  8 2022, 15:47:33) [MSC v.1929 64 bit (AMD64)]
 Architecture:   64bit
 numpy:          C:/Users/your_username/anaconda3/envs/giotto_env/Lib/site-packages/numpy
 numpy_version:  2.1.3
 
 NOTE: Python version was forced by use_python() function


```
