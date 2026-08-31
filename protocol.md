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
curl -fsSL https://ollama.com/download/ollama-linux-amd64.tar.zst | tar x --zstd -C ollama

# Launch the server in the background using nohup to redirect logs and keep the API running quietly.
nohup ollama/bin/ollama serve > ollama.log 2>&1 &

# Download the model
ollama/bin/ollama pull qwen3-coder

# Do a test prompt
ollama/bin/ollama run llama3.1 "Say hi"

# Check GPU usage
ollama/bin/ollama ps
```

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

Step 1: Set gander's default chat model to your local AI

```
options(gander.chat = chat)
```

Step 2: **Set a keyboard shortcut** for gander - this is how you'll invoke gander throughout the workshop:

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

## Explore counts and metadata

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

## Filter Low-Expression Genes

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

## See what gander saw:

```
gander_peek()
```

## Run DESeq2 on filtered counts 

> [!IMPORTANT]
> HIGHLIGHT: 'filtered_counts' AND 'metadata' (select both lines for gander to see both the count matrix and the metadata to understand the full analysis context) \
>     # the name of the filtered output file can vary (e.g. counts_filtered or filtered_counts or other) \
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

# Part 6: Data Analysis and Visualization

## Summarize results
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

## Ask a specific question
> [!IMPORTANT]
> HIGHLIGHT: results \
> PROMPT: How many adjusted p-values were less than 0.1?

```
results
```

:eyes: **sample gander output**

```
sum(results$padj < 0.1, na.rm = TRUE)
```

## Examine significant genes

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: View top 10 significant genes based on padj

```
results
```

:eyes: **sample gander output**

```
results %>% 
  as.data.frame() %>% 
  rownames_to_column(var = "gene") %>% 
  arrange(padj) %>% 
  head(10) %>% 
  select(gene, padj)
```

## Create an MA plot

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: Use DESeq2 to create an MA plot

:eyes: **sample gander output**

```
library(ggplot2)
data <- as.data.frame(results)
data$significant <- ifelse(data$padj < 0.05, "Significant", "Not Significant")
ggplot(data, aes(x = baseMean, y = log2FoldChange)) +
  geom_point(aes(color = significant), alpha = 0.6) +
  scale_x_log10() +
  labs(title = "MA Plot", x = "Average Expression (baseMean)", y = "Log2 Fold Change") +
  theme_minimal() +
  scale_color_manual(values = c("Not Significant" = "black", "Significant" = "red"))
```


# Part 6: Gene Enrichment Analysis

After differential expression you have hundreds of genes — too many to read one by one. Functional enrichment analysis translates differentially expressed genes into biological themes, pathways and functions.

## Perform gene set enrichment analysis

Step 1: Extract significant results for gene set analysis

> [!IMPORTANT]
> HIGHLIGHT: 'results' \
> PROMPT: Create a new object with significantly upregulated genes based on padj and log2FC and exclude NA values

:eyes: **sample gander output**

```
significant_genes <- results %>%
  as.data.frame() %>%
  filter(!is.na(padj) & !is.na(log2FoldChange)) %>%
  filter(padj < 0.05 & abs(log2FoldChange) > 1) %>%
  rownames_to_column(var = "gene") %>%
  column_to_rownames(var = "gene")
```




`

