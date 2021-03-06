---
layout: page
---



# Demo: Using R and Bioconductor for RNA-seq data analysis

*[back to course contents](..)*

In this demo we'll analyze some publicly available RNA-seq gene expression data using R and bioinformatics-focused R packages in [Bioconductor](http://bioconductor.org/). This demo assumes some familiarity with R (functions, functions, vectors, creating variables, getting help, subsetting, data frames, plotting, and reading/writing files). The starting point for this analysis is a *count matrix* - RNA-seq data has already been cleaned, aligned, and counted to the gene level. 

## Install and load packages

First, we'll need to install some add-on packages. Most generic R packages are hosted on the Comprehensive R Archive Network (CRAN, <http://cran.us.r-project.org/>). To install one of these packages, you would use `install.packages("packagename")`. You only need to install a package once, then load it each time using `library(packagename)`. Let's install the **gplots** and **calibrate** packages.


```r
install.packages("gplots")
install.packages("calibrate")
```

Bioconductor packages work a bit differently, and are not hosted on CRAN. Go to <http://bioconductor.org/> to learn more about the Bioconductor project. To use any Bioconductor package, you'll need a few "core" Bioconductor packages. Run the following commands to (1) download the installer script, and (2) install some core Bioconductor packages. You'll need internet connectivity to do this, and it'll take a few minutes, but it only needs to be done once.


```r
# Download the installer script
source("http://bioconductor.org/biocLite.R")

# biocLite() is the bioconductor installer function. Run it without any
# arguments to install the core packages or update any installed packages.
# This requires internet connectivity and will take some time!
biocLite()
```

To install specific packages, first download the installer script if you haven't done so, and use `biocLite("packagename")`. This only needs to be done once then you can load the package like any other package. Let's download the [DESeq2 package](http://www.bioconductor.org/packages/release/bioc/html/DESeq2.html):


```r
# Do only once
source("http://bioconductor.org/biocLite.R")
biocLite("DESeq2")
```

Now let's load the packages we'll use:


```r
library(DESeq2)
library(gplots)
library(calibrate)
```

Bioconductor packages usually have great documentation in the form of *vignettes*. For a great example, take a look at the [DESeq2 vignette for analyzing count data](http://www.bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.pdf).



## Publicly available RNA-seq data

Now, let's analyze some publicly available gene expression (RNA-seq) data. NCBI Gene Expression Omnibus (<http://www.ncbi.nlm.nih.gov/geo/>) is an international public repository that archives and freely distributes microarray, next-generation sequencing, and other forms of high-throughput functional genomics data submitted by the research community. Many publishers require gene expression data be submitted to GEO and made publicly available before publication. You can learn a lot more about GEO by reading their [overview](http://www.ncbi.nlm.nih.gov/geo/info/overview.html) and [FAQ](http://www.ncbi.nlm.nih.gov/geo/info/faq.html) pages. At the time of this writing, GEO hosts over 45,000 studies comprising over 1,000,000 samples on over 10,000 different technology platforms.

In this demonstration, we're going to be using data from GEO Series accession number GSE18508. You can enter this number in the search box on the GEO homepage, or use [this direct link](http://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE18508). In this study, the authors performed RNA-sequencing of mRNA from *Drosophila melanogaster* S2-DRSC cells that have been RNAi depleted of mRNAs encoding RNA binding proteins and splicing factors. This was done as part of the modENCODE project, published in: Brooks AN et al. Conservation of an RNA regulatory map between Drosophila and mammals. *Genome Res* 2011 Feb;21(2):193-202.

You can go to the [GEO accession page](http://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE18508) and download all the raw data at the bottom under supplementary data. However, this dataset is nearly 100GB. What I have in this repository is a spreadsheet containing a matrix of gene counts, where genes are in rows and samples are in columns. That is, this data has already been aligned and counted, and the number in the cell is the number of RNA-seq reads that mapped to that gene for that sample. The value in the *i*-th row and the *j*-th column of the matrix tells how many reads have been mapped to gene *i* in sample *j*. To do these steps yourself, you would need to align reads to the genome (e.g., using [STAR](https://code.google.com/p/rna-star/)) and count reads mapping to features (e.g., using a GTF file from Ensembl and a tool like [featureCounts](http://bioinf.wehi.edu.au/featureCounts/)).

Note: much of this was adapted from the [DESeq2 package vignette](http://www.bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.pdf).

## Load the data

First, set your working directory (folder) to the `lessons/intro-r-lifesci` subdirectory wherever you saved and extracted the [code repository for this lesson](https://github.com/bioconnector/workshops/archive/master.zip). This way, you can reference the data using a *relative path* (e.g. `data/pasilla_counts.csv`) instead of an *absolute path* (e.g. `C:/Users/name/downloads/workshops/lessons/data/pasilla_counts.csv`). You can do this either through the RStudio graphical menu (Session, Set Working Directory, Choose...), or through the `setwd()` function. You can check where you are with `getwd()`.


```r
getwd()
```

Next, load two different datasets: one with the count data, and one with sample information. Take a look at the first few rows of the count dataset, as well as the entire metadata data frame.


```r
# Load the count data
pasillacounts <- read.csv("data/pasilla_counts.csv", header = TRUE, row.names = 1)
head(pasillacounts)

# Load the sample metadata
pasillameta <- read.csv("data/pasilla_metadata.csv", header = TRUE, row.names = 1)
pasillameta
```

Notice something here. The values that are printed across the top are the `colnames` and the values printed in the gutter to the side are the `rownames`. Notice how the **colnames of the countdata**, which are the names of our samples, match the **rownames of the metadata*.

DESeq works on a particular type of object called a **DESeqDataSet**. The DESeqDataSet is a single object that contains input values, intermediate calculations like how things are normalized, and all results of a differential expression analysis. You can construct a DESeqDataSet from a count matrix, a metadata file, and a formula indicating the design of the experiment. The design formula expresses the variables which will be used in modeling. The formula should be a tilde (~) followed by the variables with plus signs between them. To do this, we run the `DESeqDataSetFromMatrix` function, giving it data frames containing the count matrix as well as the column (metadata) information, and the design formula.


```r
dds <- DESeqDataSetFromMatrix(countData = pasillacounts, colData = pasillameta, 
    design = ~condition)
dds
```

## Differential expression analysis

The standard differential expression analysis steps are wrapped into a single function, `DESeq`. This convenience function normalizes the library, estimates dispersion, and performs a statistical test under the negative binomial model.


```r
# Normalize, estimate dispersion, fit model
dds <- DESeq(dds)
```

The results are accessed using the function `results()`. The `mcols` function gives you more information about what the columns in the results tell you. 


```r
# Extract results
res <- results(dds)
head(res)
mcols(res)
```

You can write the results to a file if you wish, or even write out only the statistically significant ones using the `subset()` command conditioning on the `padj` variable (adjusted p-value). Finally, let's create an MA-plot, showing the log2 fold change over the normalized counts for each gene, colored red if statistically significant (padj<0.1). Get more help with `?plotMA`.


```r
# Write out all results
write.csv(res, file = "pasilla_results.csv")

# Write out significant results
write.csv(subset(res, padj < 0.05), file = "sig_pasilla_results.csv")

# Create MA Plot
plotMA(dds)
```

## Data transformation and visualization

The differential expression analysis above operates on the raw (normalized) count data. But for visualizing or clustering data as you would with a microarray experiment, you ned to work with transformed versions of the data. First, use a *regularlized log* transformation while re-estimating the dispersion ignoring any information you have about the samples (`blind=TRUE`). Perform a principal components analysis and hierarchical clustering.


```r
# Transform
rld <- rlogTransformation(dds)

# Principal components analysis
plotPCA(rld, intgroup = c("condition", "type"))

# Hierarchical clustering analysis
plot(hclust(dist(t(assay(rld)))))
```

## Multifactor design

See the [DESeq2 vignette](http://www.bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.pdf) for more information on multifactor designs. This is how you would take library type into account (single-end vs paired-end sequencing).

## Record package and version info with `sessionInfo()`

The `sessionInfo()` prints version information about R and any attached packages. It's a good practice to always run this command at the end of your R session and record it for the sake of reproducibility in the future.


```r
sessionInfo()
```

```
## R version 3.1.0 (2014-04-10)
## Platform: x86_64-apple-darwin13.1.0 (64-bit)
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
## [1] parallel  methods   stats     graphics  grDevices utils     datasets 
## [8] base     
## 
## other attached packages:
##  [1] calibrate_1.7.2         MASS_7.3-33            
##  [3] gplots_2.14.1           DESeq2_1.4.5           
##  [5] RcppArmadillo_0.4.320.0 Rcpp_0.11.2            
##  [7] GenomicRanges_1.16.3    GenomeInfoDb_1.0.2     
##  [9] IRanges_1.22.9          BiocGenerics_0.10.0    
## [11] knitr_1.6               BiocInstaller_1.14.2   
## 
## loaded via a namespace (and not attached):
##  [1] annotate_1.42.1      AnnotationDbi_1.26.0 Biobase_2.24.0      
##  [4] bitops_1.0-6         caTools_1.17         DBI_0.3.1           
##  [7] evaluate_0.5.5       formatR_0.10         gdata_2.13.3        
## [10] genefilter_1.46.1    geneplotter_1.42.0   grid_3.1.0          
## [13] gtools_3.4.1         KernSmooth_2.23-12   lattice_0.20-29     
## [16] locfit_1.5-9.1       RColorBrewer_1.0-5   RSQLite_0.11.4      
## [19] splines_3.1.0        stats4_3.1.0         stringr_0.6.2       
## [22] survival_2.37-7      tools_3.1.0          XML_3.98-1.1        
## [25] xtable_1.7-3         XVector_0.4.0
```
