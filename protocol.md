# Part 1: RStudio Setup on AnVIL

## Step 1. Open Your Workspace and Configure Compute Settings

1. Navigate to your AnVIL Workspace
2. Click on the cloud icon on the far right to view Cloud Environment options
3. In the dialog box, click the **Settings** button under **RStudio**
4. Configure the following:

| Parameter | Selection |
|:-- | :-- |
| Application | RStudio |
| CPUs | 8 |
| Memory | 52 GB |
| GPU Configuration | Enable GPUs (Toggle ON)
| GPU Type | NVIDIA Tesla T4 |
| Number of GPUs | 1 |

5. Review the estimated hourly running cost displayed in the dialog, then scroll down and click **CREATE**

***Provisioning takes several minutes. AnVIL is requesting cloud instances and configuring GPU drivers.***

## Step 2. Launch RStudio

1. When your environment is ready, its status will change to **Green (Running)**
2. Click the **RStudio** icon, then click **Open**.
3. RStudio will open in a new browser tab. 

# Part 2: Install Packages and Load Data

## Step 1. Install Required Packages

Run the following in the R Console:

```
# AnVIL packages
BiocManager::install("AnVILGCP")

# Bioconductor packages
BiocManager::install("DESeq2")

# AI integration packages
install.packages(c("gander", "ellmer"))
```

## Step 2. Load Libraries

```
library(AnVILGCP)
library(tidyverse)
library(DESeq2)
library(gander)
library(ellmer)
```

## Step 3. Import data from Google Cloud Storage

Run the following two commands pin the R Console. These copy the airway dataset files from an AnVIL workspace bucket:

```
gcloud_storage( "cp gs://fc-493d543d-3286-48ad-aeec-0bcb84b06fe5/airwaycounts.csv . " )
gcloud_storage( "cp gs://fc-493d543d-3286-48ad-aeec-0bcb84b06fe5/sample_metadata.csv . " )
```

:white_check_mark: Checkpoint: In the Files pane (bottom-right in RStudio), confirm that airwaycounts,csv and sample_metadata.csv are now present in your working directory.

## Step 4. Load datasets into R

```
counts <- read.csv("airwaycounts.csv", row.names = 1, check.names = FALSE)
metadata <- read.csv("sample_metadata.csv", row.names = 1 )
```

:white_check_mark: Checkpoint: Run code `dim(counts)` - you should see genes x 8 samples. Run cod `metadata` to see the metadata file.

# Part 3: Local AI Setup (Ollama + Qwen3-Coder)

Instead of relying on paid cloud API, we'll run a free, open-source AI model locally on your AnVIL instance, taking advantage of the GPU you provisioned.

## Step 1. Opene RStudio Terminal

In RStudio, click the **Terminal** tab (next to the Console tab)

> [!IMPORTANT]
> All commands in Part 3, Step 2 must be run in the Terminal (not inside R Console)

## Step 2. Install Ollama server and Download the Model

```
# Create a directory and downolad the local Ollama 
mkdir ollama
curl -fsSL https://github.com/ollama/ollama/releases/download/v0.24.0/ollama-linux-amd64.tar.zst | tar x --zstd -C ollama

# Start the Ollama server (runs the background)
ollama/bin/ollama

# Download the AI Model (Qwen3-Coder, will take a few minutes)
ollama/bin/ollama pull qwen3-coder
```

What's happening:
- Ollama is a tool for running large language models locally
- Qwen3-Coder is an open-source model optimized for code generation. It understands R, Bioconductor, and bioinformatics workflows.

## Step 3. Connect R to your Local AI server

*Switch back to the R Console tab*

> [!IMPORTANT]
> Execute following commands inside R Console (not in Terminal)

```
# Connect R to your local AI
chat <- chat_ollama(
  base_url = Sys.getenv("ollama/bin/ollama", "http://localhost:11434"),
  model = "qwen3-coder",
)

# Test the Connection - ask AI a question
chat$chat("Tell me one fact about bacterial genomes")
```

## Step 4. Configure gander  

```
#Set gander's default chat model to your local AI
options(gander.chat = chat)
```

**Set a keyboard shortcut** for gander - this is how you'll invoke gander throughout the workshop:

***In RStudio: Navigate to Tools → Modify Keyboard Shortcuts… → search for “gander” → assign Shift+Cmd+g***


## Step 5. You are ready to begin gandering in RStudio...

Your RStudio session now has an AI research partner that lives on your AnVIL instance. Let's put it to work.

-----

# Part 4: Differential Expression with DESeq2 + gander

> [!NOTE]
> **A Critical tip for Data Analysis with Gander: Minimize Context Noise**
> 
> For this section, you MUST create a new .R script (open a **fresh `.R` script**) containing only the necessary inputs to keep Gander focused and avoid cluttering its context window**:
> 1. `read.csv()' for counts
> 2. `read.csv()`for metadata
> 3. The target R object for Gander to analyze (e.g. [imported] counts)
>
> *Keeping the script lean prevents background code clutter from interfering with Gander's responses.*

## How to use gander shortcut: the works is always the same

> [!IMPORTANT]
> 1. **Highlight** an object: e.g. 'counts' \
> 2. **Evoke gander** with your pre-set shortcut *Shift+Cmd+g* \
> 3. **Enter a prompt** - a short, concrete instruction of what you want to do in plain language \
> 4. **Review the output** before running the generated code

# Explore counts and metadata

> [!IMPORTANT]
> HIGHLIGHT: 'counts' \
> PROMPT: Describe my counts dataset \
> PROMPT: Summarize my counts dataset

```
counts
```

To see what context gander used to generate its response:

```
gander_peek()
```

> [!IMPORTANT]
> HIGHLIGHT: 'metadata' \
> PROMPT: Describe my metadata file 

```
metadata
```

# Filter Low-Expression Genes

> [!IMPORTANT]
> HIGHLIGHT: 'counts' \
> PROMPT: Filter genes with ≥ 1 counts in all samples, create a new counts object, and summarize

```
counts
```

:eyes: **sample gander output**

```
counts_filtered <- counts[rowSums(counts >= 1) == ncol(counts), ]
summary(counts_filtered)
```

# See what gander saw:

```
gander_peek()
```

:white_check_mark: Checkpoint: The number of rows in filtered counts should be smaller than counts after removing low-expression genes

# Run DESeq2 on filtered counts 

> [!IMPORTANT]
> HIGHLIGHT: 'filtered_counts' AND 'metadata' (select both lines for gander to see both the count matrix and the metadata to understand the full analysis context)
>     # the name of the filtered output file can vary (e.g. counts_filtered or filtered_counts or other) 
> PROMPT: Perform differential expression with filtered counts using DESeq2 and `dex` as the design condition and save the results in a new object

```
filtered counts
metadata
```
Check what gander saw:

```
gander_peek()
```

:eyes: **sample gander output**

```
library(DESeq2)

# Create DESeq dataset
dds <- DESeqDataSetFromMatrix(countData = counts_filtered,
                              colData = metadata,
                              design = ~ dex)

# Run DESeq
dds <- DESeq(dds)

# Get results
results <- results(dds, contrast = c("dex", "trt", "untrt"))
```
#Convert the DESeqResults object into a standard data frame, preserve gene names from row names into a dedicated column, and sort rows by raw p-value.

results_df <- as.data.frame(results) %>%
  rownames_to_column(var = "gene") %>%
  arrange(pvalue)


Check what gander saw:

```
gander_peek()
```

# Part 5: Data Analysis and Visualization

## Examine significant genes

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: View top 10 significant genes based on padj

:eyes: **sample gander output**

```
top_10_genes <- results[order(results$padj), ][1:10, ]
top_10_genes
```

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: View top 10 significant genes based on fold change

:eyes: **sample gander output**

```
top_10_genes <- results[order(abs(results$log2FoldChange), decreasing = TRUE), ][1:10, ]
top_10_genes
```

## Convert Ensembl ids to Gene Symbols

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: Convert ENSEMBL IDs to gene symbols and view first 10 gene symbols

:eyes: **sample gander output**

```
library(org.Hs.eg.db)
results_genes <- as.data.frame(results)
results_genes$external_gene_name <- mapIds(org.Hs.eg.db, keys=rownames(results_genes), column="SYMBOL", keytype="ENSEMBL", multiVals="first")
head(results_genes$external_gene_name, 10)
```

> [!CAUTION]
> May get an ERROR due to missing the required library 'org.Hs.eg.db'. 
> Can choose to manually install with `install.packages("BiocManager")`, or, use gander to troubleshoot!

Install the required package

```
BiocManager::install("DESeq2")
```

then, re-run gander-suggested code

```
library(org.Hs.eg.db)
results_genes <- as.data.frame(results)
results_genes$external_gene_name <- mapIds(org.Hs.eg.db, keys=rownames(results_genes), column="SYMBOL", keytype="ENSEMBL", multiVals="first")
head(results_genes$external_gene_name, 10)
```

## Create an MA plot

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: Create an MA plot using DESeq2

:eyes: **sample gander output**

```
library(ggplot2)
plotMA(results, main="MA Plot", ylim=c(-5,5))
```


# Part 6: Gene Enrichment Analysis

After differential expression you have hundreds of genes — too many to read one by one. Functional enrichment analysis translates differentially expressed genes into biological themes, pathways and functions.

## Perform gene set enrichment analysis

Step 1: Extract significant results for gene set analysis

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: Create a new significant results object with upregulated genes based on padj and log2FC

:eyes: **sample gander output**

```
res_up <- results[which(results$padj < 0.05 & results$log2FoldChange > 0), ]
```

Step 2: Map ENSEMBL IDs to ENTREZ IDs

> [!IMPORTANT]
> HIGHLIGHT: 'res_up' \
> PROMPT: Map Ensembl IDs to Entreez IDs and Symbols 

:eyes: **sample gander output**

```
res_up_mapped <- merge(
  as.data.frame(res_up),
  bitr(rownames(res_up), fromType = "ENSEMBL", toType = c("ENTREZID", "SYMBOL"), OrgDb = org.Hs.eg.db),
  by.x = 0, by.y = "ENSEMBL"
)

```

Step 3: Use clusterProfiler to perform gene enrichment analysis

> [!CAUTION]
> May get an ERROR due to missing the required library 'clusterProfiler'. 
> Can choose to manually install with `BiocManager::install("clusterProfiler")`, or, use gander to troubleshoot!

Install the missing package

```
BiocManager::install("clusterProfiler")
```

:eyes: **sample gander output**

```
library(clusterProfiler)
library(org.Hs.eg.db)

go_enrichment <- enrichGO(
  gene = res_up_mapped$ENTREZID,
  OrgDb = org.Hs.eg.db,
  keyType = "ENTREZID",
  ont = "BP",
  pvalueCutoff = 0.05,
  qvalueCutoff = 0.05
)

kegg_enrichment <- enrichKEGG(
  gene = res_up_mapped$ENTREZID,
  organism = "hsa",
  pvalueCutoff = 0.05,
  qvalueCutoff = 0.05
)

go_enrichment
kegg_enrichment
```
Step 4: Visualize GO enrichment results using clusterProfiler
