# corto
_corto_ (Correlation Tool): an R package to generate correlation-based DPI networks.

To install the _corto_ stable version from CRAN:
```{r install, eval=FALSE}
install.packages("corto")
```

Alternatively, you can install the _corto_ developmental version directly from Github:
```{r}
library(devtools)
install_github("chunxuan-hs/corto")
```

![corto logo correlation tool](https://giorgilaborg.files.wordpress.com/2019/10/cortoicon.png)


# Donation
If you help us buying a cup of coffee, we will convert the caffeine into code improvements!

[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=LQ9T3X5EAD4YY&currency_code=EUR&source=url)

# Introduction
The _corto_ ("correlation tool") package provides a pipeline to infer networks between "centroid" and "target" variables in a dataset, using a combination of Pearson correlation and Data Processing Inequality (DPI), first proposed in [1]. The main application of _corto_ is in the field of Bioinformatics and Transcriptomics, where co-occurrence between variables can be used as a mean to infer regulatory mechanisms [2] or gene functions [3]. In this field, usually the tested features are genes (or rather, their expression profile across samples), whereas the centroids are Transcription Factors (TFs) and their targets are Target Genes (TGs). The TF-TG co-expression can hint at a causal regulatory relationship, as proven in many studies [4,5,6]. The _corto_ tool replicates the well-established pipeline of the ARACNe family of tools [7,8,9]

_corto_ focuses on tho aspects:

1. Gene Network Inference, with DPI steps, Bootstrapping and optional CNV correction
2. Gene Network Interrogation, via Master Regulator Analysis (MRA), with functions to visualize user-provided top deregulated networks on a specific signature.


## The _corto_ Gene Network Inference Algorithm
In brief, _corto_ operates using the following steps:

1. Load a gene expression matrix and a list of user-provided centroids (e.g. TFs). Data should be normalized via Variance Stabilizing Transformation (VST) for RNA-Seq, and Robust Multiarray Average (RMA) for microarrays, as described in [10].
2. Removal of genes with zero variance across the dataset and rank-transformation of each gene expression profile.
3. Calculate all pairwise Pearson correlation coefficients between centroid and target features. The rank transformation operated by the Pearson correlation coefficient is a common procedure used in co-expression based studies, due to its benefits in reducing the effects of outliers [11].
4. Filter out all centroid-target features whose correlation coefficient _absolute value_ is below the provided p-value threshold _p_.
5. Apply DPI to all centroid-centroid-target triplets, in order to identify which centroid-target correlation is stronger and identify the most likely association (e.g. which TF is the most likely regulator of the TG in the dataset).
6. All edges are tested for Data Processing Inequality in a number of bootstrapped versions of the same input matrix (specified by the _nbootstraps_ parameter, 100 by default, as in ARACNE-AP [8]). This step can be run in parallel by specifiying the number of threads using the _nthreads_ parameter. The number of occurrences of each edge (if they survived DPI) is calculated. This number can range from 0 (the edge is not significant in any bootstrap) to _nbootstraps_+1 (the edge is significant in all bootstraps, and also in the original matrix).
7. A network is generated in the form of a list, where each element is a centroid-based list of targets. In order to follow the structure of the _regulon_ class implemented by downstream analysis packages (e.g. VIPER [12]), each target is characterized by two parameters:
    + The _tfmode_, the mode of action, i.e. the Pearson correlation coefficient between the target and the centroid in the original input matrix
    + The _likelihood_ of the interaction, calculated as the number of bootstraps in which the edge survives DPI, divided by the total number of bootstraps (in this impelmentation, the presence of an edge in the original, non-bootstrapped matrix is considered as an additional evidence)


# Running _corto_
Here is how to run _corto_. First, install the package:
```{r install, eval=FALSE}
install.packages("corto")
```

Then, load it:
```{r load}
library(corto)
```

Then, you can see how the input matrix looks like. For example, this dataset comes from the TCGA mesothelioma project [13] and measures the expression of 10021 genes across 87 samples:
```{r load1}
load(system.file("extdata","inmat.rda",package="corto"))
inmat[1:5,1:5]
```
```{r load2}
dim(inmat)
```
Another input needed by _corto_ is a list of centroid features. In our case, we can specify a list of TFs generated from Gene Ontology with the term "Regulation of Transcription" [14].

```{r load3}
load(system.file("extdata","centroids.rda",package="corto"))
centroids[15]
```
```{r load4}
length(centroids)
```

Finally, we can run _corto_. In this example, we will run it with p-value threshold of 1e-30, 10 bootstraps and 2 threads
```{r runcorto,message=FALSE,results="hide"}
regulon<-corto(inmat,centroids=centroids,nbootstraps=10,p=1e-30,nthreads=2)
# Input Matrix has 87 samples and 10021 features
# Correlation Coefficient Threshold is: 0.889962633618839
# Removed 112 features with zero variance
# Calculating pairwise correlations
# Initial testing of triplets for DPI
# 246 edges passed the initial threshold
# Building DPI network from 37 centroids and 136 targets
# Running 100 bootstraps with 2 thread(s)
# Calculating edge likelihood
# Generating regulon object
```

The regulon object is a list:
```{r prinregulon}
regulon[1:2]
```

The regulon in this dataset is composed of 34 final centroids with at least one target:
```{r prinregulon2}
length(regulon)
```
```{r prinregulon3}
names(regulon)
```

# Correcting with CNV data
As an additional, optional feature, __corto__ gives the user the possibility to provide Copy Number Variation (CNV) data, if available, which can generate spurious correlations between TFs and targets [15]. The gene expression profiles for the target genes are corrected via linear regression and the residuals of the expression~cnv model substitute the original gene expression profile.

In this example, a CNV matrix is provided. The analysis will be run only for the features (rows) and samples (columns) present in both matrices
```{r runcnv}
load(system.file("extdata","cnvmat.rda",package="corto",mustWork=TRUE))
regulon <- corto(inmat,centroids=centroids,nthreads=2,nbootstraps=10,verbose=TRUE,cnvmat=cnvmat,p=0.01)
```

# Applying the network to another dataset
__corto__ provides a function, mra(), to apply a previously calculated network to another dataset, in order to predict the relative levels of the centroids from their targets, measured in a different context. This can be done in two ways. Centroids can be any numerical variable, such as Transcription Factors, or Metabolites.

## Sample-by-sample centroid prediction
The network (regulon in the example below) is directly applied to a multi-sample dataset in the form of a matrix (expmat in the example below), where columns are samples and rows are targets (e.g. transcripts). In this case, the output is a matrix with the same number of samples, including as rows the predicted centroids. The scores are intended as Normalized Enrichment Scores (NESs), based on a normalized T-test for one sample vs. the rest dataset. The NES is positive if the centroid network is higher in the sample vs the mean of the dataset, negative if lower.
```{r mra1}
predicted<-mra(expmat,regulon=regulon)
```
## Contrast centroid prediction
As in the previous procedure, but the centroid score is provided as a differential NES between two sample groups (expmat1 and expmat2 in the example below). The resulting NES is positive if the centroid network is upregulated in expmat1 vs expmat2 (or in expmat1 vs the mean of the dataset), negative if downregulated.
```{r mra2}
predicted<-mra(expmat1,expmat2,regulon)
```
Only for the contrast version of the mra() function, a mraplot() function can be applied to the output, in order to graphically show the most differentially significant centroids in the queried contrast (figure from [16])
```{r mra3}
mraplot(predicted)
```
![Figure1](https://user-images.githubusercontent.com/1401900/159940897-4d7372d0-0a32-44b4-9269-d2fa4aa6d9af.png)



# References
[1] Reverter, Antonio, and Eva KF Chan. "Combining partial correlation and an information theory approach to the reversed engineering of gene co-expression networks." Bioinformatics 24.21 (2008): 2491-2497.

[2] D’Haeseleer, Patrik, Shoudan Liang, and Roland Somogyi. "Genetic network inference: from co-expression clustering to reverse engineering." Bioinformatics 16.8 (2000): 707-726.

[3] Hansen, Bjoern O., et al. "Elucidating gene function and function evolution through comparison of co-expression networks of plants." Frontiers in plant science 5 (2014): 394.

[4] Basso, Katia, et al. "Reverse engineering of regulatory networks in human B cells." Nature genetics 37.4 (2005): 382.
[5] Amar, David, Hershel Safer, and Ron Shamir. "Dissection of regulatory networks that are altered in disease via differential co-expression." PLoS computational biology 9.3 (2013): e1002955.

[6] Vandepoele, Klaas, et al. "Unraveling transcriptional control in Arabidopsis using cis-regulatory elements and coexpression networks." Plant physiology 150.2 (2009): 535-546.

[7] Margolin, Adam A., et al. "ARACNE: an algorithm for the reconstruction of gene regulatory networks in a mammalian cellular context." BMC bioinformatics. Vol. 7. No. 1. BioMed Central, 2006.

[8] Lachmann, Alexander, et al. "ARACNe-AP: gene network reverse engineering through adaptive partitioning inference of mutual information." Bioinformatics 32.14 (2016): 2233-2235.

[9] Khatamian, Alireza, et al. "SJARACNe: a scalable software tool for gene network reverse engineering from big data." Bioinformatics 35.12 (2018): 2165-2166.

[10] Giorgi, Federico M., et al. "Comparative study of RNA-seq-and microarray-derived coexpression networks in Arabidopsis thaliana." Bioinformatics 29.6 (2013): 717-724.

[11] Usadel, Björn, et al. "Co‐expression tools for plant biology: opportunities for hypothesis generation and caveats." Plant, cell & environment 32.12 (2009): 1633-1651.

[12] Alvarez, Mariano J., Federico Giorgi, and Andrea Califano. "Using viper, a package for Virtual Inference of Protein-activity by Enriched Regulon analysis." Bioconductor (2014): 1-14.

[13] Ladanyi, Marc, et al. "The TCGA malignant pleural mesothelioma (MPM) project: VISTA expression and delineation of a novel clinical-molecular subtype of MPM." (2018): 8516-8516.

[14] Gene Ontology Consortium. "The Gene Ontology (GO) database and informatics resource." Nucleic acids research 32.suppl_1 (2004): D258-D261.

[15] Schubert, Michael, et al. "Gene networks in cancer are biased by aneuploidies and sample impurities." Biochimica et Biophisica Acta - Gene Regulatory Models (2019): 194444. DOI: https://doi.org/10.1016/j.bbagrm.2019.194444

[16] Mercatelli, Daniele, et al. "corto: a lightweight R package for gene network inference and master regulator analysis." Bioinformatics, Vol 36, Issue 12, June 2020. DOI: https://doi.org/10.1093/bioinformatics/btaa223
