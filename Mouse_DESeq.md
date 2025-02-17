rm(list=ls())

options(java.parameters = "- Xmx1024m")
setwd('/chg3/analysts/abadea/Badea_6520_201014A7/analysis')
library(DESeq2)
library(tximport)
library(readr)
library(readxl)
library(dplyr)
library(gplots)
library(ggplot2)
library(xlsx)
library(ggpubr)
library(GenomicFeatures)
library(ggsci)

# Load in metadata
meta <- as.data.frame(read_excel("MouseMetadata.xlsx", sheet = "PCA", col_names = TRUE))
head(meta)
dim(meta)

# Remove bad FastQ files and samples with missing metadata
meta <- meta[which(meta$SampleID!="210201-19"),]
meta <- meta[which(meta$SampleID!="210201-10"),]
meta <- meta[which(meta$SampleID!="210201-12"),]
meta <- meta[which(meta$SampleID!="210201-13"),]
meta <- meta[which(meta$SampleID!="210201-9"),]
meta <- meta[,c("SampleID","Rep","Filename","Genotype","Sex","AgeRounded","Diet","RIN","Qubit","Batch")]

# Convert and scale variables
meta$Batch <- as.factor(meta$Batch)
meta$Rep <- as.factor(meta$Rep)
meta$SampleID <- as.factor(meta$SampleID)
meta$Sex <- as.factor(meta$Sex)
meta$Genotype <- as.factor(meta$Genotype)
meta$Diet <- as.factor(meta$Diet)
meta$Qubit <- scale(meta$Qubit)
meta$AgeRounded <- scale(as.numeric(meta$AgeRounded))
head(meta)
dim(meta)

# Make tx2gene df
txdb <- makeTxDbFromGFF("/chg3/analysts/abadea/Badea_6520_201014A7/salmon/mouse_index/Mus_musculus.GRCm39.105.gtf.gz", organism = "Mus musculus")
k <- keys(txdb, keytype = "TXNAME")
tx2gene <- select(txdb, k, "GENEID", "TXNAME")
head(tx2gene)

# Load in estimated counts
files <- file.path("/chg3/analysts/abadea/Badea_6520_201014A7/salmon/quants",meta$Filename,"quant.sf")
names(files) <- meta$Filename
all(file.exists(files))
txi <- tximport(files, type="salmon", tx2gene=tx2gene, ignoreTxVersion = TRUE)

# Create DESeq object
dds <- DESeqDataSetFromTximport(txi, colData = meta, design = ~ 1)

# Filter out low read count genes
keep <- rowSums(counts(dds)) >= 10
dds <- dds[keep,]

# Combine replicates
ddsColl <- collapseReplicates(dds, dds$SampleID, dds$Rep)
matchFirstLevel <- dds$SampleID == levels(dds$SampleID)[1]
stopifnot(all(rowSums(counts(dds[,matchFirstLevel])) == counts(ddsColl[,1])))

# Save unregressed normalized counts
ddsColl <- estimateSizeFactors(ddsColl)
write.table(counts(ddsColl, normalized = TRUE), "MouseNormalizedCounts.txt", sep = "\t", quote = FALSE)

# Create DESeq2 object
design(ddsColl) <- formula(~ Batch + Qubit + AgeRounded + Sex + Diet + Genotype)
ddsDE <- DESeq(ddsColl)

# APOE33 vs APOE22
resultsNames(ddsDE)
resNorm <- lfcShrink(ddsDE, coef=7, type="apeglm")
write.xlsx(resNorm, file="results.xlsx", sheetName="APOE33_vs_APOE22", row.names=TRUE, append = FALSE)

rownames(resNorm[order(-abs(resNorm$log2FoldChange))[1:20],])
df <- as.data.frame(rownames(resNorm))
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000006154"] <- "EPS8L1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000022217"] <- "EMC9"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000022790"] <- "IGSF11"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000026923"] <- "NOTCH1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000031493"] <- "GGN"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "EENSMUSG00000035270"] <- "IMPG2"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000037747"] <- "PHYHIPL"

png(file="APOE33_vs_APOE22.png")
par(mar=c(3,3,3,1))
ggmaplot(as.data.frame(resNorm, col.names = TRUE), main = expression("APOE22" %->% "APOE33"),
   fdr = 0.05, fc = 0, size = 0.4,
   palette = pal_simpsons("springfield")(16)[c(8,12,3)],
   genenames = as.vector(df$`rownames(resNorm)`),
   legend = "top", top = 0, label.select = c("EPS8L1","EMC9","IGSF11","NOTCH1","GGN","IMPG2","PHYHIPL"),
   font.label = c("bold", 11), label.rectangle = TRUE,
   font.legend = "bold",
   font.main = "bold",
   ggtheme = ggplot2::theme_minimal())

dev.off()



# APOE44 vs APOE22
resultsNames(ddsDE)

resNorm <- lfcShrink(ddsDE, coef=9, type="apeglm")
write.xlsx(resNorm, file="results.xlsx", sheetName="APOE44_vs_APOE22", row.names=TRUE, append = TRUE)

rownames(resNorm[order(-abs(resNorm$log2FoldChange))[1:20],])
df <- as.data.frame(rownames(resNorm))
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000006154"] <- "EPS8L1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000018865"] <- "SULT4A1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000022292"] <- "RRM2B"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000026181"] <- "PPM1F"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000026321"] <- "TNFRSF11A"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000031292"] <- "CDKL5"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000035293"] <- "G2E3"

png(file="APOE44_vs_APOE22.png")
par(mar=c(3,3,3,1))
ggmaplot(as.data.frame(resNorm, col.names = TRUE), main = expression("APOE22" %->% "APOE44"),
   fdr = 0.05, fc = 0, size = 0.4,
   palette = pal_simpsons("springfield")(16)[c(8,12,3)],
   genenames = as.vector(df$`rownames(resNorm)`),
   legend = "top", top = 0, label.select = c("EPS8L1","SULT4A1","RRM2B","PPM1F","TNFRSF11A","CDKL5","G2E3"),
   font.label = c("bold", 11), label.rectangle = TRUE,
   font.legend = "bold",
   font.main = "bold",
   ggtheme = ggplot2::theme_minimal())

dev.off()



# HN vs APOE22
resultsNames(ddsDE)

resNorm <- lfcShrink(ddsDE, coef=11, type="apeglm")
write.xlsx(resNorm, file="results.xlsx", sheetName="HN_vs_APOE22", row.names=TRUE, append = TRUE)

rownames(resNorm[order(-abs(resNorm$log2FoldChange))[1:20],])
df <- as.data.frame(rownames(resNorm))
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000027959"] <- "SASS6"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000028524"] <- "SGIP1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000031012"] <- "CASK"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000036104"] <- "RAB3GAP1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000040181"] <- "FMO1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000054942"] <- "MIGA1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000094075"] <- "IGHV1"

png(file="HN_vs_APOE22.png")
par(mar=c(3,3,3,1))
ggmaplot(as.data.frame(resNorm, col.names = TRUE), main = expression("APOE22" %->% "HN"),
   fdr = 0.05, fc = 0, size = 0.4,
   palette = pal_simpsons("springfield")(16)[c(8,12,3)],
   genenames = as.vector(df$`rownames(resNorm)`),
   legend = "top", top = 0, label.select = c("SASS6","SGIP1","CASK","RAB3GAP1","FMO1","MIGA1","IGHV1"),
   font.label = c("bold", 11), label.rectangle = TRUE,
   font.legend = "bold",
   font.main = "bold",
   ggtheme = ggplot2::theme_minimal())

dev.off()


# APOE44 vs APOE33
ddsColl$Genotype <- relevel(ddsColl$Genotype, ref = "APOE33")
ddsDE <- DESeq(ddsColl)
resultsNames(ddsDE)

resNorm <- lfcShrink(ddsDE, coef=9, type="apeglm")
write.xlsx(resNorm, file="results.xlsx", sheetName="APOE44_vs_APOE33", row.names=TRUE, append = TRUE)

rownames(resNorm[order(-abs(resNorm$log2FoldChange))[1:20],])
df <- as.data.frame(rownames(resNorm))
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000006154"] <- "EPS8L1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000018865"] <- "SULT4A1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000015087"] <- "RABL6"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000020190"] <- "MKNK2"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000020253"] <- "PPM1M"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000024399"] <- "LTB"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000026946"] <- "NMI"

png(file="APOE44_vs_APOE33.png")
par(mar=c(3,3,3,1))
ggmaplot(as.data.frame(resNorm, col.names = TRUE), main = expression("APOE33" %->% "APOE44"),
   fdr = 0.05, fc = 0, size = 0.4,
   palette = pal_simpsons("springfield")(16)[c(8,12,3)],
   genenames = as.vector(df$`rownames(resNorm)`),
   legend = "top", top = 0, label.select = c("EPS8L1","SULT4A1","RABL6","PPM1M","MKNK2","LTB","NMI"),
   font.label = c("bold", 11), label.rectangle = TRUE,
   font.legend = "bold",
   font.main = "bold",
   ggtheme = ggplot2::theme_minimal())

dev.off()


# APOE33HN vs APOE33
resultsNames(ddsDE)

resNorm <- lfcShrink(ddsDE, coef=8, type="apeglm")
write.xlsx(resNorm, file="results.xlsx", sheetName="APOE33HN_vs_APOE33", row.names=TRUE, append = TRUE)

rownames(resNorm[order(-abs(resNorm$log2FoldChange))[1:20],])
df <- as.data.frame(rownames(resNorm))
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000005667"] <- "MTHFD2"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000018585"] <- "ATOX1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000020649"] <- "RRM2"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000020669"] <- "SH3YL1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000031543"] <- "ANK1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000032498"] <- "MLH1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000038539"] <- "ATF5"

png(file="APOE33HN_vs_APOE33.png")
ggmaplot(as.data.frame(resNorm, col.names = TRUE), main = expression("APOE33" %->% "APOE33HN"),
   fdr = 0.05, fc = 0, size = 0.4,
   palette = pal_simpsons("springfield")(16)[c(8,12,3)],
   genenames = as.vector(df$`rownames(resNorm)`),
   legend = "top", top = 0, label.select = c("MTHFD2","ATOX1","RRM2","SH3YL1","ANK1","MLH1","ATF5"),
   font.label = c("bold", 11), label.rectangle = TRUE,
   font.legend = "bold",
   font.main = "bold",
   ggtheme = ggplot2::theme_minimal())

dev.off()


# HN vs APOE33
resultsNames(ddsDE)

resNorm <- lfcShrink(ddsDE, coef=11, type="apeglm")
write.xlsx(resNorm, file="results.xlsx", sheetName="HN_vs_APOE33", row.names=TRUE, append = TRUE)

rownames(resNorm[order(-abs(resNorm$log2FoldChange))[1:20],])
df <- as.data.frame(rownames(resNorm))
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000028524"] <- "SGIP1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000033581"] <- "IGF2BP2"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000036192"] <- "RORB"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000036790"] <- "SLITRK2"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000042742"] <- "BMT2"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000054942"] <- "MIGA1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000064259"] <- "VN1R5"

png(file="HN_vs_APOE33.png")
ggmaplot(as.data.frame(resNorm, col.names = TRUE), main = expression("APOE33" %->% "HN"),
   fdr = 0.05, fc = 0, size = 0.4,
   palette = pal_simpsons("springfield")(16)[c(8,12,3)],
   genenames = as.vector(df$`rownames(resNorm)`),
   legend = "top", top = 0, label.select = c("SGIP1","IGF2BP2","RORB","SLITRK2","BMT2","MIGA1","VN1R5"),
   font.label = c("bold", 11), label.rectangle = TRUE,
   font.legend = "bold",
   font.main = "bold",
   ggtheme = ggplot2::theme_minimal())

dev.off()




# APOE44HN vs APOE44
ddsColl$Genotype <- relevel(ddsColl$Genotype, ref = "APOE44")
ddsDE <- DESeq(ddsColl)
resultsNames(ddsDE)

resNorm <- lfcShrink(ddsDE, coef=10, type="apeglm")
write.xlsx(resNorm, file="results.xlsx", sheetName="APOE44HN_vs_APOE44", row.names=TRUE, append = TRUE)

rownames(resNorm[order(-abs(resNorm$log2FoldChange))[1:20],])
df <- as.data.frame(rownames(resNorm))
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000020253"] <- "PPM1M"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000020893"] <- "PER1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000022836"] <- "MYLK"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000027605"] <- "ACSS2"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000028159"] <- "DAPP1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000031425"] <- "PLP1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000038949"] <- "CNST"

png(file="APOE44HN_vs_APOE44.png")
ggmaplot(as.data.frame(resNorm, col.names = TRUE), main = expression("APOE44" %->% "APOE44HN"),
   fdr = 0.05, fc = 0, size = 0.4,
   palette = pal_simpsons("springfield")(16)[c(8,12,3)],
   genenames = as.vector(df$`rownames(resNorm)`),
   legend = "top", top = 0, label.select = c("PPM1M","PER1","MYLK","ACSS2","DAPP1","PLP1","CNST"),
   font.label = c("bold", 11), label.rectangle = TRUE,
   font.legend = "bold",
   font.main = "bold",
   ggtheme = ggplot2::theme_minimal())

dev.off()



# HN vs APOE44
resultsNames(ddsDE)

resNorm <- lfcShrink(ddsDE, coef=11, type="apeglm")
write.xlsx(resNorm, file="results.xlsx", sheetName="HN_vs_APOE44", row.names=TRUE, append = TRUE)

rownames(resNorm[order(-abs(resNorm$log2FoldChange))[1:20],])
df <- as.data.frame(rownames(resNorm))
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000028524"] <- "SGIP1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000036790"] <- "SLITRK2"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000054942"] <- "MIGA1"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000064259"] <- "VN1R5"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000067780"] <- "PI15"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000072294"] <- "KLF12"
df["rownames(resNorm)"][df["rownames(resNorm)"] == "ENSMUSG00000109764"] <- "KLKB1"

png(file="HN_vs_APOE44.png")
ggmaplot(as.data.frame(resNorm, col.names = TRUE), main = expression("APOE44" %->% "HN"),
   fdr = 0.05, fc = 0, size = 0.4,
   palette = pal_simpsons("springfield")(16)[c(8,12,3)],
   genenames = as.vector(df$`rownames(resNorm)`),
   legend = "top", top = 0, label.select = c("SGIP1","SLITRK2","MIGA1","VN1R5","PI15","KLF12","KLKB1"),
   font.label = c("bold", 11), label.rectangle = TRUE,
   font.legend = "bold",
   font.main = "bold",
   ggtheme = ggplot2::theme_minimal())

dev.off()
