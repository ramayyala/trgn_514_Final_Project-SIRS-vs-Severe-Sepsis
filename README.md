---
title: "SIRS VS SEPSIS"
author: "Ram Ayyala"
date: "11/29/2021"
output: powerpoint_presentation
always_allow_html: true
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = FALSE)

```


```{r include=FALSE}
library(data.table)
library(tidyverse)
library(ggfortify)
library(factoextra)
library(ggdendro)
library(dendextend)
library(gplots)
library(flextable)
library(ggstatsplot)
library(splitstackshape)
```

## Introduction: Terms

**Sepsis** is a not a disease, like many think, but rather is the body's response to an infection, just in a very extreme manner 1.
- There is no singular cause for sepsis, as it is brought on by many heterogeneous pathophysiologies, but it is generally brought on due to an infection 1. 
- Occurs when the human body's immune system stops fighting the infection and starts to turn on itself 4. 
- Levels of Sepsis:
  - **Uncomplicated Sepsis** is the early stages of sepsis in which patients typically suffer from chills, elevated heart rate, and elevated breathing rate 8.
  - **Severe Sepsis** occurs when there is organ damage due to the inflammatory response 7. 
  - **Septic Shock** occurs when there is a drastic drop in blood pressure (< 65mm Hg) causing high levels of lactic acid in the blood 6. 

**Systemic Inflammatory Response Syndrome (SIRS)** is an exaggerated defense mechanism similar to sepsis.  
- Symptoms include low blood pressure, abnormal body temperature, or abnormal white blood cell count 2. 
- Like Sepsis, there is no singular cause for SIRS. 
- Unlike sepsis, SIRS is caused by a variety of stressors like infections, surgeries, inflammation, etc 2.
. 

## Introduction: A Priori hypothesis

- Hypothesis: **The Severe Sepsis samples will be more highly associated with the upregulated platelet aggregation genes than the SIRS samples.**  

## Introduction: Sequencing and Data 

- Data was acquired via sequencing of **peripheral blood RNA** from whole blood cells of 129 subjects using an Illumina Genome Analyzer 9. 
  - SIRS: **n=23** patients 9. 
  - Sepsis: **n=106** patients, of which 78 survived 9. 
- Used 16 samples of the 129 subjects, 4 from each of the following groups:
  - SIRS
  - Severe Sepsis
  - Septic Shock
  - Uncomplicated Sepsis 
- Reads were single ended and not run in multiple lanes nor provided as multiple SRR files 

## Pre-Alignment Sequencing Metrics:

- Subjects were chosen such that they had similar number of reads and belonged to one of the 4 groups. 
```{r echo=F}
reads <- read.csv('linecount.csv')
names(reads)[1] <- "Samples"
reads <- subset(reads, select = -c(line.count))
reads <- as.data.frame(reads)
groups <- data.frame(c(rep("Septic Shock",4),rep("Severe Sepsis",4),rep("SIRS",4),rep("Uncomplicated Sepsis",4)))
colnames(groups) <- c('Groups')
reads['Groups'] <- groups$Groups
names <- data.frame(c("Sept_Shock_1","Sept_Shock_2","Sept_Shock_3","Sept_Shock_4","Sev_Sepsis_1","Sev_Sepsis_2","Sev_Sepsis_3","Sev_Sepsis_4","SIRS_1","SIRS_2","SIRS_3","SIRS_4","US_1","US_2","US_3","US_4"))
colnames(names) <- c('Names')
reads['Names'] <- names$Names
reads[,c("Names","Reads")] %>% flextable() %>% set_caption(caption = "Preliminary Reads of each sample", style= "Table Caption") %>%
  theme_vanilla() %>% autofit()
```

```{r echo=FALSE}
summaryfun <- function(x)list(N=length(x),Mean=mean(x),Median=median(x),SD=sd(x),Min=min(x),Max=max(x))
summaryfun(reads$Reads) %>% as.data.frame() %>% flextable() %>%
  set_caption(caption = "Preliminary Reads Descriptive Stats") %>%
  theme_vanilla() %>% autofit()
```

```{r}
reads %>% 
  mutate(Names=fct_reorder(Names, Reads, na.rm=TRUE)) %>%
  ggplot(aes(x=Names, y=Reads, fill=Groups)) +
  geom_bar(stat="identity")+
  labs(title = "Reads per Sample by Groups", x= "Samples", y="Reads") +
  scale_fill_brewer(palette = "Spectral") +
  theme(axis.text.x = element_text(face = "bold", 
                           size = 8, angle = 45,hjust = 1))
```

## Tophat2 - alignment / counting / stats process description:

![TopHat2 Workflow](Figures/TopHat2_Workflow.png)


## Tophat2 - alignment metrics and statistics:

```{r echo=F}
summaryfun(reads$Aligned_Reads) %>% as.data.frame() %>% flextable() %>% 
  set_caption(caption = "Aligned Reads Descriptive Stats") %>%
  theme_vanilla() %>% autofit() 
```

```{r echo=F}
summaryfun(reads$Percent_Alignment) %>% as.data.frame() %>% flextable() %>% 
  set_caption(caption = "Percent Aligned Descriptive Stats") %>%
  theme_vanilla() %>% autofit() 
```

```{r echo=F}
reads %>% 
  mutate(Names=fct_reorder(Names, Percent_Alignment, na.rm=TRUE)) %>%
  ggplot(aes(x=Names, y=Percent_Alignment, fill=Groups)) +
  geom_bar(stat="identity")+
  labs(title = "Percent Aligned Reads per Sample by Groups", x= "Samples", y="% Aligned") +
  scale_fill_brewer(palette = "Spectral") +
  theme(axis.text.x = element_text(face = "bold", 
                           size = 8, angle = 45,hjust = 1))
```

## Results:FPKMs Across all Samples
```{r echo=F}
#CUFFDIFF NO STATS
#Load Data
data <- read.table('CuffDiff_no_stats/genes.fpkm_tracking', header=TRUE, sep="\t")
#Select only FPKM columns
df<-data[ , grepl( "FPKM" , names( data ) ) ]
# Add gene id to the sorted dataframe
gene_id <- data$gene_id
df <- cbind(df,gene_id)
# remove 0s from df if all of row is 0 
df <-df[rowSums(df[1:16])>0,]
#set gene_id as index
rownames(df) <- df$gene_id
gene_id <- df$gene_id
df <- subset (df, select = -gene_id)
#pseudo count and log10 scale
df <- df+0.01
df <- log10(df) 
#rename colnames to simpler names
colnames(df) <- names$Names
#Stack df
stacked_df <- stack(df)
# Add groups category to the stacked df 
groups <- data.frame(c(rep("Septic Shock",147580),rep("Severe Sepsis",147580),rep("SIRS",147580),rep("Uncomplicated Sepsis",147580)))
colnames(groups) <- c('Groups')
stacked_df['Groups'] <- groups$Groups
# Boxplot across all Samples
ggplot(stacked_df, aes(x = ind, y = values,fill=Groups)) +
  geom_boxplot() +
  labs(title = "Normalized FPKM Across all Samples", x= "Samples", y="Normalized FPKM") +
  scale_fill_brewer(palette="Spectral") +
  theme(axis.text.x = element_text(angle = 45,hjust = 1)) +
  ylim(-5,8)
```
## Results:PCA of FPKMs Across all Samples
```{r echo=F}
#PCA
t_df <- t(df)
groups <- data.frame(c("Septic Shock","Septic Shock","Septic Shock","Septic Shock","Severe Sepsis","Severe Sepsis","Severe Sepsis","Severe Sepsis","SIRS","SIRS","SIRS","SIRS","Uncomplicated Sepsis","Uncomplicated Sepsis","Uncomplicated Sepsis","Uncomplicated Sepsis"))
colnames(groups) <- c('Groups')
autoplot(prcomp(t_df), data=groups, colour='Groups',label=TRUE,label.size=3.5) + labs(title = "PCA Analysis of FPKM values of All Samples")
```

## Results: PCA Analysis of Filtered FPKM values of All Samples
```{r echo=F}
#Load Data
different <- read.table('CuffDiff_stats/different.csv', header=TRUE, sep=",")
data <- read.table('CuffDiff_no_stats/genes.fpkm_tracking', header=TRUE, sep="\t")
#Select only SIRS and SEV_SEP FPKM columns
df<-data[ , grepl( "FPKM" , names( data ) ) ]
#df<-as.data.frame(df[ , grepl( paste(c("sirs","sev_sep"),collapse="|") , names( df ) ) ])
# Add gene id to the sorted dataframe
gene_id <- data$gene_id
df <- cbind(df,gene_id)

#Limit FPKM list to genes selected in different dataframe
df <- as.data.frame(setDT(df)[gene_id %chin% different$gene_id])

# remove 0s from df if all of row is 0 
df <- df[rowSums(df[,1:8])>0,]
#set gene_id as index
rownames(df) <- df$gene_id
gene_id <- df$gene_id
df <- subset (df, select = -gene_id)
#pseudo count and log10 scale
df <- df+0.01
df <- log10(df) 
#rename colnames to simpler names
colnames(df) <- names$Names
#PCA
t_df <- t(df)
groups <- data.frame(c("Septic Shock","Septic Shock","Septic Shock","Septic Shock","Severe Sepsis","Severe Sepsis","Severe Sepsis","Severe Sepsis","SIRS","SIRS","SIRS","SIRS","Uncomplicated Sepsis","Uncomplicated Sepsis","Uncomplicated Sepsis","Uncomplicated Sepsis"))
colnames(groups) <- c('Groups')
autoplot(prcomp(t_df), data=groups, colour='Groups',label=TRUE,label.size=3.5) + labs(title = "PCA Analysis of Filtered FPKM values of All Samples")
```
## Results: Dendrogram of Filtered FPKM values of All Samples
```{r echo=F}
dist.eucl <- dist(t_df, method="euclidean")
dist.cor <- get_dist(t_df,method="pearson")
hclust.complete.eucl <- hclust(d=dist.eucl, method = "complete")
dend <- hclust.complete.eucl %>% as.dendrogram %>%
  set("branches_k_color",value = c("red", "blue"), k = 2) %>% set("branches_lwd", 1) %>%
  set("labels_col", value = c("red", "blue"), k=2) %>% set("labels_cex", 0.5) %>% 
  set("leaves_pch", 20) %>% set("leaves_cex", 1) %>% set("leaves_col", value = c("red", "blue"), k=2)
# plot the dend in usual "base" plotting engine:
#plot(dend)
ggd1 <- as.ggdend(dend)
ggplot(ggd1,horiz = TRUE,theme = theme_minimal()) +
  labs(title = "Dendrogram of All Samples", x= "Samples", y="Height")
```

## Results: Heatmap of Filtered FPKM values of All Samples VS Filtered Genes
```{r}
#HEATMAP
#ggheatmap(df,dist_method="euclidean",hclust_method="ward.D2",text_show_rows=waiver())
heatmap.2(as.matrix(df),labRow = FALSE,srtCol=40,ylab="Genes",
          main = "Samples vs Filtered Genes") 

```

## Results: PCA Analysis of Filtered FPKM values of SIRS vs Sepsis
```{r echo=F}
#Load Data
different <- read.table('CuffDiff_stats/different.csv', header=TRUE, sep=",")
data <- read.table('CuffDiff_no_stats/genes.fpkm_tracking', header=TRUE, sep="\t")
#Select only SIRS and SEV_SEP FPKM columns
df<-data[ , grepl( "FPKM" , names( data ) ) ]
df<-as.data.frame(df[ , grepl( paste(c("sirs","sev_sep"),collapse="|") , names( df ) ) ])
# Add gene id to the sorted dataframe
gene_id <- data$gene_id
df <- cbind(df,gene_id)

#Limit FPKM list to genes selected in different dataframe
df <- as.data.frame(setDT(df)[gene_id %chin% different$gene_id])

# remove 0s from df if all of row is 0 
df <- df[rowSums(df[,1:8])>0,]
#set gene_id as index
rownames(df) <- df$gene_id
gene_id <- df$gene_id
df <- subset (df, select = -gene_id)
#pseudo count and log10 scale
df <- df+0.01
df <- log10(df) 
#rename colnames to simpler names
names<-names$Names[5:12]
colnames(df) <- names
#PCA
t_df <- t(df)
groups <- data.frame(c("Severe Sepsis","Severe Sepsis","Severe Sepsis","Severe Sepsis","SIRS","SIRS","SIRS","SIRS"))
colnames(groups) <- c('Groups')
autoplot(prcomp(t_df), data=groups, colour='Groups',label=TRUE,label.size=3.5) + labs(title = "PCA Analysis of Filtered FPKM values of SIRS VS SEPSIS")
```
## Results: Dendrogram of Filtered FPKM values of Severe Sepsis vs SIRS
```{r echo=F}
dist.eucl <- dist(t_df, method="euclidean")
dist.cor <- get_dist(t_df,method="pearson")
hclust.complete.eucl <- hclust(d=dist.eucl, method = "complete")
dend <- hclust.complete.eucl %>% as.dendrogram %>%
  set("branches_k_color",value = c("red", "blue"), k = 2) %>% set("branches_lwd", 1) %>%
  set("labels_col", value = c("red", "blue"), k=2) %>% set("labels_cex", 0.5) %>% 
  set("leaves_pch", 20) %>% set("leaves_cex", 1) %>% set("leaves_col", value = c("red", "blue"), k=2)
# plot the dend in usual "base" plotting engine:
#plot(dend)
ggd1 <- as.ggdend(dend)
ggplot(ggd1,horiz = TRUE,theme = theme_minimal()) +
  labs(title = "Dendrogram of Severe Sepsis vs SIRS ", x= "Samples", y="Height")
```
## Results: Heatmap of Filtered FPKM values of SIRS & Severe Sepsis Samples VS Filtered Genes
```{r}
#HEATMAP
#ggheatmap(df,dist_method="euclidean",hclust_method="ward.D2",text_show_rows=waiver())
heatmap.2(as.matrix(df),labRow = FALSE,srtCol=30,ylab="Genes",
          main = "SIRS/Severe Sepsis Samples VS Filtered Genes")

```

## Pathway Analysis:Upregulated Genes
```{r echo=FALSE}
IPA <- read.csv('IPA_upregulated_platelets.csv')
names(IPA)[1] <- "Category"
IPA %>% flextable() %>% set_caption(caption = "IPA Upregulated Pathway", style= "Table Caption") %>%
  theme_vanilla() %>% autofit()
```
## Pathway Analysis:Upregulated Platelet Aggregation Genes PCA
```{r echo=F}
#Load Data
different <- read.table('CuffDiff_stats/different.csv', header=TRUE, sep=",")
data <- read.table('CuffDiff_no_stats/genes.fpkm_tracking', header=TRUE, sep="\t")
IPA<-cSplit(IPA,"Molecules",",")
Molecules<-t(IPA)[5:12,]
#Select only SIRS and SEV_SEP FPKM columns
df<-data[ , grepl( "FPKM" , names( data ) ) ]
df<-as.data.frame(df[ , grepl( paste(c("sirs","sev_sep"),collapse="|") , names( df ) ) ])
# Add gene id to the sorted dataframe
gene_id <- data$gene_short_name
df <- cbind(df,gene_id)

#Limit FPKM list to genes selected in different dataframe
df <- as.data.frame(setDT(df)[gene_id %chin% Molecules])

# remove 0s from df if all of row is 0 
df <- df[rowSums(df[,1:8])>0,]
#set gene_id as index
rownames(df) <- df$gene_id
gene_id <- df$gene_id
df <- subset (df, select = -gene_id)
#pseudo count and log10 scale
df <- df+0.01
df <- log10(df) 
#rename colnames to simpler names
colnames(df) <- names
#PCA
t_df <- t(df)
groups <- data.frame(c("Severe Sepsis","Severe Sepsis","Severe Sepsis","Severe Sepsis","SIRS","SIRS","SIRS","SIRS"))
colnames(groups) <- c('Groups')
autoplot(prcomp(t_df), data=groups, colour='Groups',label=TRUE,label.size=3.5) + labs(title = "PCA Analysis of Upregulated Platelet Aggregation Genes in SIRS VS SEPSIS")
```
## Pathway Analysis:Upregulated Platelet Aggregation Genes Heatmap
```{r echo=F}
#HEATMAP
#ggheatmap(df,dist_method="euclidean",hclust_method="ward.D2",text_show_rows=waiver())
heatmap.2(as.matrix(df),srtCol=30)
```
## Pathway Analysis: Findings
From the IPA Pathway analysis, we see that there is an upregulation in genes associated with platelet aggregation in patients with severe sepsis.
- When a patient is infected, the body will activate its defense mechanisms which can be boiled down to two main mechanisms:
  - **inflammatory response**10
  - **coagulation activation**10
- In short, the constant activation of the immune system leads to in an increase in platelet aggregation as that is the natural response
- However, it must be noted, that late stage sepsis, such as severe sepsis or septic shock, but mainly septic shock are generally associated with **thrombocytopenia** or **low-platelet-count**, but there is previous evidence supporting that there is an increase in thrombocytosis in the earlier stages of sepsis 12. 


## Pathway Analysis:Downregulated Genes
```{r echo=F}
IPA <- read.csv('IPA_downregulated_B_lymphocytes.csv')
names(IPA)[1] <- "Category"
names(IPA)[4] <- "# Molecules"
IPA %>% flextable() %>% set_caption(caption = "IPA Downregulated Pathway", style= "Table Caption") %>%
  theme_vanilla() %>% autofit()
```

## Pathway Analysis: Findings
From the IPA Pathway analysis, we see that there is an downregulation in genes associated with cell viability and activation of B lymphocytes, which can cause **lymphopenia**. 
- When a patient has sepsis, it is believed that there is a type of immune suppression that occurs behind the scenes leading to increased morbidity and mortality rates 12. 
  - One key immunosuppression, is the apoptosis of certain immune cells like **B-lymphocytes** 12.
  - Septic patients have been found to have a higher rate of lymphopenia which is associated with a higher prevalance for septic shocks and higher mortality rates 12. 
  

## Conclusion & Future Studies
- Overall, it is plausible to say that there is a difference in the genes expressed between Severe Sepsis and SIRS, despite Sepsis essentially being a derivative of SIRS. Moreover, we see that there is a significant upregulation of genes associated with platelet aggregation in the Severe Sepsis samples in comparison to the SIRS samples. Furthermore, it is not unreasonable to state that their is probably a significant downregulation of genes associated with cell vaibility and activation of B lymphocytes given the evidence presented in previous literature. 

- In the future:
  - It is necessary to make sure that all of the samples have a high read alignment percentage as in this study, 2 of the uncomplicated sepsis samples have abysmal levels of alignment percentage, making their contribution to the study practically useless. 
  - Investigate the effects of sepsis on thrombocytopenia and thrombocytosis as it seems to have a significant effect on both diseases. 
  - Investigate more into sepsis's effects on lymphopenia as it wasn't feasible to investigate the significance of the downregulation of the FCLR3 gene given that there were barely any downregulated genes to begin with in the IPA analysis. 


## References: 
1. https://pubmed.ncbi.nlm.nih.gov/25538794/ 
2.https://www.cancer.gov/publications/dictionaries/cancer-terms/def/sirs
3. https://www.ncbi.nlm.nih.gov/books/NBK547669/
4. https://www.cdc.gov/sepsis/what-is-sepsis.html
5. https://www.sepsis.org/sepsis-basics/what-is-sepsis/
6. https://www.mayoclinic.org/diseases-conditions/sepsis/symptoms-causes/syc-20351214
7. https://www.sepsis.org/sepsis-basics/what-is-sepsis/severe-sepsis/#:~:text=Severe%20sepsis%20occurs%20when%20one,or%20organs%20that%20are%20affected.
8. https://www.nhsinform.scot/illnesses-and-conditions/blood-and-lymph/septic-shock
9. https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE63042
10. https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6679237/
11. https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6377227/
12. https://journals.lww.com/ccmjournal/Abstract/2013/12001/1047___Thrombocytosis_versus_Thrombocytopenia_as.1001.aspx
