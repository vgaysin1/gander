
---
title: "AI-in-RStudio-on-AnVIL"
output: html_document
date: "2026-09-28"
---

# Table of Contents

00. About this Course
  - Learning goals
  - Abstract
  - Introduction
  - Ethical use of AI for data analysis
  - Methods
  - Troubleshooting Tips
01. Setup
02. Install packages
0.3

---

# 00 About this Course

## Learning Goals

By the end you will be able to:
1. Provision a GPU-enabled RStudio environment on AnVIL and read its cost and hardware limits before starting work.
2. Serve a coding model locally with Ollama on that instance, confirm that the GPU is doing the work, and connect R to it through ellmer.
3. Point gander at the local model and invoke it from a keyboard shortcut on the code you have selected.
4. Move data from a workspace bucket into the analysis environment and check that the sample table and the count matrix actually line up.
5. Run the analysis with the model's code — filtering lowly expressed genes, fitting a DESeq2 model, and reading the results object.
6. Make figures and one gene list — an MA plot, a summary table, a top-by-significance table, and a functional enrichment result.
7. Stop the runtime so the workshop does not keep billing.

## Abstract

This workshop puts a large language model inside RStudio on AnVIL, using an open-source model that runs on the same cloud instance as the analysis. No paid API, no key to manage, and no data leaving the environment.

gander is an RStudio assistant that sees your session - the objects in your environment, the columns in your data frames, the code around your cursor — and hands that context to an ellmer chat. 
ellmer is the client that speaks to a model, and it speaks to Ollama. 

The analysis is the airway dataset: airway smooth muscle cells, four cell lines, treated with dexamethasone or left untreated. You filter the counts, test for differential expression with DESeq2, read the results table, plot it, and take the gene list into a functional enrichment step. The glucocorticoid response is well characterized, which matters more than it sounds: at several points there is an independently known answer, and checking the model's output against it is the difference between a demonstration and a lesson.

The pattern is the transferable part. State an intent. Prompt the assistant. Read the code it returns before running it. Run it. Inspect the output. Hand back the error verbatim when it fails. 
That loop is the same whether the objects are counts, variants, or a spreadsheet.

Keywords: AI-assisted data analysis, gander, ellmer, Ollama, AnVIL, large language models, reproducible research, RNA-seq, DESeq2, genomic data science education

---

## Introduction

Welcome to gander-assisted data analysis in RStudio on AnVIL. This course is designed to show you how to utilize LLMs in the context of your RStudio environment on AnVIL.

**Gander and Ellmer tools**

Gander and ellmer perform different jobs, and neither one is the AI model itself.
- `ellmer` is the underlying client package that connects R to various LLM providers. It knows how to speak to a model and handles the mechanics of communication, including authentication and streaming responses token by token It supports major backend providers such as Anthropic, OpenAI, Google, Ollama, and others.
- `gander` is an R package that integrates an AI assistant in your RStudio session and is aware of the objects in your active session. It gathers context from your R environment - such as the objects in your environment, the column names and types in your data frames, the code around your cursor - and hands that context to an ellmer chat.

**Pick your path: Hosted vs. Local**

You have two primary paths for running models in RStudio in this course. Choose the path that best fits your security, budget, and hardware constraints. 

| Feature | Hosted API| Local (Ollama)|
| :--| :--|:--|
| Cost |Per-token| Free|
| Data| code and data are sent to external cloud | Data never leaves your machine (private) |
| Hardware | None or Minimal | Substantial ~19 GB disk; GPU strongly preferred |
| Model quality | Highest available frontier models| Generally smaller, somewhat weaker models |
| Best for | General public datasets and prototyping | Proprietary, sensitive, or restricted data|

**Picking a model/provider:**
Selecting the right model depends heavily on how it is hosted and what task it needs to perform.
- Hosted APIs: If you select the hosted path, you will configure an API key for a kajor provider through 'ellmer', tapping into state-of-the-art cloud models.
- Local models: For local execution via Ollama, we demonstrate using code-optimized models such as qwen3-coder trained specifically on code; General-purpose models of the same parameter size will often produce code that looks plausible at glance, but fails during execution. 

> [!IMPORTANT]
> In this course we will focus exclusively on the **Local (Ollama)** path, which requires access to AnVIL.

**Bulk RNA-seq as working example:**
We use bulk RNA-seq analysis for several reasons. 
- Canonical pipeline: RNA-seq follows a well-established, multi-step pipeline including quality control, normalization, differential expression testing, and visualization. Readers work through the airway dataset end to end: inspecting and filtering counts, testing for differential expression with DESeq2, interpreting PCA, MA, and volcano plots, and debugging the errors that arise.
- Airway Dataset: We utilize the 'airway' dataset, an RNA-seq transcriptome profiling study of airway smooth muscle cells responding to dexamethasone treatment (PubMed 24926665). 
- Transferable Patterns: The core interaction loop - stating an intent, prompting your assistant, reading and reviewing the returned code, executing it, and verifying the output — is the same loop whether the analysis is RNA-seq, another omics assay, or a spreadsheet.

## Ethical use of AI for data analysis

**Where the data actually goes**. A hosted API sends code and data context to an external provider. A local model does not — "local" means the model runs on a cloud virtual machine that belongs to your workspace, inside Google Cloud. 
Data is restricted by an institutional agreement, a data use agreement, or an IRB protocol still requires that agreement to cover this environment. 
"Local model" removes one exposure (an outside vendor); it does not remove the need to know where your workspace lives, who can attach to it, and what the bucket's access controls are.

**No key to leak**. The local path has no API key, which removes an entire class of mistake. The shortcut is not free of consequences: an assistant with no key still writes code that runs against restricted data.

**Human oversight**. The model is not responsible for your results. You are. Every step in this workshop ends with you reading the output rather than accepting it.

**Literacy over convenience.** When the assistant suggests a function you do not recognize, ask it to explain the line rather than running it. The prompt "Annotate and explain this code" is the single highest-value prompt in this document.

## Questions

1. **What is the researcher's role when integrating generative AI into scientific analysis?**
   
  > A. You can safely accept generated code blindly if it runs without syntax errors.
  > 
  > B. The AI tool assumes full scientific responsibility for the validity of the results.
  >   
  > C. You, the researcher, remain entirely responsible for the final analytical output, statistical validity, and biological interpretations.
  >   
  > D. Domain expertise is no longer required once a model is successfully integrated into RStudio.
  >

Correct Answer: C (AI tools do not replace domain expertise; researchers must maintain oversight and think critically about every analytical step).

---

## Methods 

**Runtime**. AnVIL cloud environment, RStudio application, 8 CPUs, 52 GB memory, 1 × NVIDIA Tesla T4 GPU (16 GB of VRAM). RStudio is served in a browser tab, not run as a desktop application, which affects keyboard shortcuts (§3.5).

**Packages**. Four families: bucket access (AnVILGCP), data wrangling (tidyverse), differential expression (DESeq2, airway via BiocManager), and the AI layer (gander, ellmer). The model is served by Ollama, installed from the Terminal.

**Workflow**. Seven parts, one stage each: provisioning, package installation, local model setup, data import, differential expression, interpretation, and functional enrichment. §08 closes the environment down. Each stage is taught as a loop rather than a recipe — state an intent, prompt the model, read the code, run it, inspect, and when it fails, pass the error back verbatim before accepting a fix.

**Dataset**. The airway dataset: dexamethasone-treated versus untreated human airway smooth muscle cells, four cell lines, eight samples, 63,677 genes (Himes et al. 2014). It is used here because it fits on a laptop, is freely available, and has a well-characterized glucocorticoid response that lets you check the model's biological interpretation against what is independently known.

---

## Troubleshooting tips

**Prompt rule of thumb**. Keep prompts short and concrete. Not "help me do RNA-seq" — ask "Filter genes with at least 1 count in all samples, create a new object, then summarize".

**Minimize context noise** — and on this path, treat it as a hard requirement. 
Open a fresh  script containing only what the task needs: the two `read.csv()` lines and `the object` you want analyzed. 
A long script sends a long pile of unrelated code, and ellmer's Ollama client caps input at 2,048 tokens, so unrelated code is not just noise here, it is competition for a small budget.

**Troubleshooting errors**
- Ask again, verbatim. A second sample of the same answer is informative.
- Ask again, more specifically. Name the object, the column, and the output format you want.
- Highlight the error and the code together. The pairing is the request.
- Remove context. Use the plain chat object (chat$chat("...")) instead of the gander add-in. With no surrounding code injected, the question has room to be answered.
- Add context. The opposite move for the opposite failure: put the data frame and the failing line in the script and re-ask with both highlighted.

**AnVIL-specific failures**

---

## How to use gander shortcut:

> [!IMPORTANT]
> 1. **Highlight** an object: e.g. 'counts' \
> 2. **Evoke gander** with your pre-set shortcut *Shift+Cmd+g* \
> 3. **Enter a prompt** - a short, concrete instruction of what you want to do in plain language \
> 4. **Review the output** before running the generated code

## Questions

1. **What is the sequence and best practice when interacting with the AI assistant gander in RStudio?**

  > A. Open a blank script, type your prompt in plain language, and automatically execute the generated code without inspection.
  >   
  > B. Highlight a specific object for context, invoke gander with the shortcut, enter a short and concrete prompt, and carefully review the output before running the generated code.
  > 
  > C. Trigger the shortcut first, type a broad multi-task instruction like "do all my RNA-seq analysis," and let the AI overwrite your workspace files.
  > 
  > D. Skip highlighting any objects, use the shortcut to launch a local Ollama server, and let the model automatically push results to GitHub.
  > 

Correct Answer: B (You highlight an object for context, trigger gander, supply a clear prompt, and always review the AI's code output before execution).

---

# 01: RStudio Setup with GPU on AnVIL

## 1.1 Open your workspace and configure compute settings

1. In AnVIL, navigate to your workspace
2. Click on the cloud icon on the far right to view **Cloud Environment** options
3. In the dialog box, under **RStudio** click **Settings**
4. Configure the:

| Parameter | Selection |
|:-- | :-- |
| Application | RStudio |
| CPUs | 8 |
| Memory | 52 GB |
| GPU Configuration | Enable GPUs (Toggle ON)
| GPU Type | NVIDIA Tesla T4 |
| Number of GPUs | 1 |

5. Review the estimated hourly running cost displayed in the dialog
6. Scroll down and click **CREATE**

***Provisioning takes several minutes. AnVIL is requesting cloud instances and configuring GPU drivers.***

### FYI What you just provisioned, and what it will and will not hold

The GPU is 16 GB. The model is about 19 GB. 
A Tesla T4 has 16 GB of VRAM, and qwen3-coder:30b model we plan on using needs roughly 19 GB at 4-bit quantization. 
The model therefore does not fit entirely on the card, and Ollama will place some layers on the CPU. That is not a misconfiguration — with 8 CPUs and 52 GB of memory, it works, and it is slower than a fully resident model would be.

Three ways to respond, in order of how much the workshop depends on it:
- Accept the split. Everything in this document works. Expect the first token to take seconds rather than a fraction of a second.
- Use a smaller tag. qwen3-coder:7b fits in VRAM comfortably and will answer faster; however, you will likely trade quality.
- Provision a larger GPU if the workspace offers one (currently it does not), and accept the higher hourly cost.

## 1.2 Launch RStudio

1. When your environment is ready, its status will change to **green (Running)**
2. Click the **RStudio** icon, then **Open**.
3. RStudio will open in a new browser tab. 

> [!NOTE]
> You are running RStudio Server in a browser tab, not a desktop application. Two consequences: files are written to the instance's disk, not your laptop; and keyboard shortcuts are mediated by the browser.

## Questions

1. **Why does enabling a GPU matter for this workshop on AnVIL?**

  > A. DESeq2 requires a GPU.
  >   
  > B. The language model is served locally and runs faster on one.
  > 
  > C. It speeds up the bucket copy
  > 
  > D. it is required by AnVIL for RStudio
  > 

Correct Answer: B (The language model is served locally and runs faster on one).

---

# 02 Install Packages 

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

---

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

## Step 4. Configure gander  

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

