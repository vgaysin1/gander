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

---

# Part 2: Local AI Setup (Ollama + Qwen3-Coder)

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

---

# Part 3: Install Packages 

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

## Step 3. Configure gander  

```
#Set gander's default chat model to your local AI
options(gander.chat = chat)
```

**Set a keyboard shortcut** for gander - this is how you'll invoke gander throughout the workshop:

***In RStudio: Navigate to Tools → Modify Keyboard Shortcuts… → search for “gander” → assign Shift+Cmd+g***

## Step 4. You are ready to begin gandering in RStudio...

Your RStudio session now has an AI research partner that lives on your AnVIL instance. Let's put it to work.

---

# Part 4: Import data

## Step 1. Import data from Google Cloud Storage

Run the following two commands pin the R Console. These copy the airway dataset files from an AnVIL workspace bucket:

```
gcloud_storage( "cp gs://fc-493d543d-3286-48ad-aeec-0bcb84b06fe5/airwaycounts.csv . " )
gcloud_storage( "cp gs://fc-493d543d-3286-48ad-aeec-0bcb84b06fe5/sample_metadata.csv . " )
```

:white_check_mark: Checkpoint: In the Files pane (bottom-right in RStudio), confirm that airwaycounts,csv and sample_metadata.csv are now present in your working directory.

## Step 2. Load datasets into R

```
counts <- read.csv("airwaycounts.csv", row.names = 1, check.names = FALSE)
metadata <- read.csv("sample_metadata.csv", row.names = 1 )
```

:white_check_mark: Checkpoint: Run code `dim(counts)` - you should see genes x 8 samples. Run cod `metadata` to see the metadata file.


-----

# Part 5: Differential Expression with DESeq2 + gander

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

# a. Explore counts and metadata

> [!IMPORTANT]
> HIGHLIGHT: 'counts' \
> PROMPT: Describe my counts dataset

```
counts
```

:eyes: **sample gander output**

`counts is a data frame containing read counts for multiple samples across several rows. The dataset appears to have 5 rows and 8 columns, with each column representing a different sample (likely from RNA-seq or similar sequencing experiments) and each row representing a gene or feature. The values are integer counts, with most entries being zero, suggesting this might be a sparse count matrix where many genes have no detectable expression in certain samples. The first few rows show counts ranging from small values around 30-60 up to approximately 1,100 for the highest values in each sample.`


To see what context gander used to generate its response:

```
gander_peek()
```

> [!IMPORTANT]
> HIGHLIGHT: 'counts' \
> PROMPT: Summarize key stat metrics for counts dataset 

```
counts
```

:eyes: **sample gander output**

```
library(dplyr)

counts %>%
  summarise_all(list(mean = mean, sd = sd, min = min, max = max)) %>%
  mutate_if(is.numeric, round, 2)
```

> [!IMPORTANT]
> HIGHLIGHT: 'metadata' \
> PROMPT: Describe my metadata file 

```
metadata
```
:eyes: **sample gander output**

`metadata is a data frame containing experimental metadata for RNA-seq samples, with columns for sample identifiers (SampleName, Run, Experiment, Sample, BioSample), experimental conditions (cell line, dex treatment, albuterol treatment), and sequencing details (avgLength). It appears to contain 49 rows of sample information from a genomic study.`


> [!IMPORTANT]
> HIGHLIGHT: 'metadata' \
> PROMPT: View my metadata file 

```
metadata
```
:eyes: **sample gander output**

```
metadata %>%
  head() %>%
  as.data.frame()
```

# b. Filter Low-Expression Genes

> [!IMPORTANT]
> HIGHLIGHT: 'counts' \
> PROMPT: Filter genes with ≥ 1 counts in all samples, create a new counts object, then summarize

```
counts
```

:eyes: **sample gander output**

```
counts_filtered <- counts[rowSums(counts >= 1) == ncol(counts), ]
summary(counts_filtered)
```

:white_check_mark: Checkpoint: The number of rows in filtered counts should be smaller than counts after removing low-expression genes

# See what gander saw:

```
gander_peek()
```

# c. Run DESeq2 on filtered counts 

> [!IMPORTANT]
> HIGHLIGHT: 'filtered_counts' AND 'metadata' (select both lines for gander to see both the count matrix and the metadata to understand the full analysis context) \
>     # the name of the filtered output file can vary (e.g. counts_filtered or filtered_counts or other) 
> PROMPT: Perform differential expression with filtered counts using DESeq2 and `dex` as the design condition then save the results as a new object

```
filtered_counts
metadata
```

:eyes: **sample gander output**

```
library(DESeq2)
dds <- DESeqDataSetFromMatrix(countData = counts_filtered, colData = colData, design = ~ dex)
dds <- DESeq(dds)
results <- results(dds)
```

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: View results

```
results
```

:eyes: **sample gander output**

```
as.data.frame(results) %>% 
  rownames_to_column('gene') %>%
  arrange(desc(abs(log2FoldChange))) %>%
  head(10)
```

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: Summarize results

```
results
```
:eyes: **sample gander output**

```
library(dplyr)

results %>%
  as.data.frame() %>%
  summarise(
    total_genes = n(),
    significant_up = sum(log2FoldChange > 0 & padj < 0.1, na.rm = TRUE),
    significant_down = sum(log2FoldChange < 0 & padj < 0.1, na.rm = TRUE),
    mean_log2_fold_change = mean(log2FoldChange, na.rm = TRUE),
    median_padj = median(padj, na.rm = TRUE)
  ) %>%
  print()
```


# Part 6: Data Analysis and Visualization

## Examine significant genes

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: View top 10 significant genes based on padj

:eyes: **sample gander output**

```
results_df[order(results_df$padj), ][1:10, ]
```

## Create an MA plot

> [!IMPORTANT]
> HIGHLIGHT: 'results_df' \
> PROMPT: Create an MA plot using DESeq2

:eyes: **sample gander output**

```
library(ggplot2)
plotMA(results_df, main="MA Plot", ylim=c(-5,5))
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
