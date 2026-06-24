# 🔷 Azure Data Factory — End-to-End Pipeline Project

> A production-style data engineering project built on **Azure Data Factory** and **Azure Blob Storage** — demonstrating real-world pipeline orchestration, metadata-driven file processing, and multi-branch CSV transformation at scale.

<br>

## 📌 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture](#-architecture)
- [Storage Structure](#-storage-structure)
- [Pipeline Details](#-pipeline-details)
- [Data Flow: transformCSV](#-data-flow-transformcsv)
- [Key Concepts Demonstrated](#-key-concepts-demonstrated)
- [Tech Stack](#-tech-stack)
- [Repository Structure](#-repository-structure)
- [Getting Started](#-getting-started)

<br>

---

## 📖 Project Overview

This project ingests raw retail sales transaction data (CSV format), processes it through a series of orchestrated ADF pipelines, filters files based on naming conventions, and applies multi-branch data transformations — landing final output in a dedicated reporting container.

| Detail | Info |
|---|---|
| **Dataset** | Retail Sales Transactions |
| **Format** | CSV |
| **Fields** | `transaction_id`, `transactional_date`, `product_id`, `customer_id`, `payment`, `credit_card`, `loyalty_card`, `cost`, `quantity`, `price` |
| **Storage Account** | `storagedatafactoryzaman` |
| **Trigger** | Scheduled (ProdPipeline) |

<br>

---

## 🏗️ Architecture

```
                    ┌──────────────────────────────────────┐
                    │           ProdPipeline               │
                    │       (Master Orchestrator)          │
                    │         [Schedule Trigger]           │
                    └────────┬─────────────────┬───────────┘
                             │                 │
               ┌─────────────▼──────┐   ┌──────▼──────────────────┐
               │   pipelinemanager  │   │    OnlySelectedFiles     │
               │                    │   │                          │
               │  1. Copy CSV       │   │  1. GetMetadata          │
               │     source →       │   │     (list files)         │
               │     destination    │   │  2. ForEach + If         │
               │  2. Delete source  │   │     (filter "Fact"       │
               │     (move pattern) │   │      files only)         │
               │  3. Execute:       │   │  3. Copy → reporting     │
               │     pipelineGIT    │   │  4. Execute:             │
               └────────┬───────────┘   │     transformCSV         │
                        │               └──────────────────────────┘
               ┌────────▼───────────┐
               │     pipelineGIT    │
               │                    │
               │  HTTP Linked Svc   │
               │  GitHub raw URL    │
               │  → ADLS dest.      │
               └────────────────────┘
```

Both `pipelinemanager` and `OnlySelectedFiles` are executed sequentially by `ProdPipeline` using **Execute Pipeline** activities with *Wait on Completion* enabled.

<br>

---

## 🗄️ Storage Structure

**Storage Account:** `storagedatafactoryzaman`

| Container | Purpose |
|---|---|
| `source` | Raw input CSV files (`Fact_Sales_1.csv`, etc.) |
| `destination` | Intermediate working folder (`csvfiles/`) |
| `reporting` | Final output — filtered Fact files + dataflow output |
| `$logs` | System-generated Azure Storage logs |

<br>

---

## 🔧 Pipeline Details

### `ProdPipeline` — Master Orchestrator
The entry point, triggered on a **schedule**. Executes two child pipelines sequentially using the **Execute Pipeline** activity with *Wait on Completion* enabled.

![ProdPipeline](https://github.com/user-attachments/assets/3b71f1d8-799f-4c4f-893e-9574cbe7be9f)

---

### `pipelinemanager` — Ingestion & File Management
Handles raw data ingestion and source file lifecycle:

| Step | Activity | Description |
|---|---|---|
| 1 | **Copy CSV** | Copies `Fact_Sales_1.csv` from `source` → `destination/csvfiles` |
| 2 | **Delete** | Removes the file from `source` after successful copy *(move pattern)* |
| 3 | **Execute Pipeline** | Triggers `pipelineGIT` to pull data from GitHub |

![pipelinemanager](https://github.com/user-attachments/assets/8258da11-85cb-4cc8-8e7a-dbe010524c3a)

---

### `pipelineGIT` — GitHub HTTP Ingestion
Pulls a CSV file from a **GitHub raw URL** via an **HTTP Linked Service** and lands it directly into the `destination` container in ADLS.

---

### `OnlySelectedFiles` — Metadata-Driven File Filtering
Dynamically selects and processes files based on naming conventions:

| Step | Activity | Description |
|---|---|---|
| 1 | **GetMetadata** | Retrieves all child items from the `destination` container |
| 2 | **ForEach + If** | Iterates files; processes only those whose name starts with `Fact` |
| 3 | **Copy Activity** | Copies matched files to the `reporting` container |
| 4 | **Execute Pipeline** | Invokes the `transformCSV` data flow for transformation |

![OnlySelectedFiles](https://github.com/user-attachments/assets/8aed9cd4-356e-49c1-a920-ee633a54e176)

<br>

---

## 🔀 Data Flow: `transformCSV`

Multi-branch transformation applied to filtered `Fact` files:

```
sourceCSV
    │
    ▼
selectcols          ← column pruning & renaming
    │
    ▼
filter1             ← row-level condition filtering
    │
    ▼
[Conditional Split — by credit_card type]
    ├── visa        ──────────────────────────────────┐
    ├── mastercard  ────────────────────────────────► aggregate
    └── amex  ──► derivedColumn (enriched fields) ───►    │
                                                          ▼
                                                      alterRows     ← insert / upsert / delete policies
                                                          │
                                                          ▼
                                                        sink         → reporting container
```

### Transformations Applied

| Transformation | Purpose |
|---|---|
| **Select** | Column pruning and renaming for downstream consistency |
| **Filter** | Removes rows not meeting business conditions |
| **Conditional Split** | Branches data by `credit_card` values: Visa, Mastercard, Amex |
| **Derived Column** | Computes enriched/calculated fields on the Amex branch |
| **Aggregate** | Applies group-level aggregations across all branches |
| **Alter Rows** | Defines row-level insert, upsert, and delete policies |
| **Sink** | Writes transformed output to the `reporting` container |

![transformCSV Data Flow](https://github.com/user-attachments/assets/45f302df-5ac1-4c73-b890-894fbce1ed25)

<br>

---

## 💡 Key Concepts Demonstrated

| Concept | Implementation |
|---|---|
| Pipeline Orchestration | Master/child pattern using Execute Pipeline activity |
| Metadata-Driven Processing | GetMetadata + ForEach + If condition for dynamic file filtering |
| Move Pattern | Copy Activity + Delete Activity simulating atomic file move |
| HTTP Ingestion | GitHub raw URL as a source via HTTP Linked Service |
| Visual ETL | ADF Mapping Data Flows with multi-branch transformations |
| Automation | Schedule trigger on `ProdPipeline` |
| Sequential Execution | Wait on Completion between child pipelines |

<br>

---

## 🧰 Tech Stack

| Tool | Role |
|---|---|
| **Azure Data Factory** | Pipeline orchestration, data flows, triggers |
| **Azure Blob Storage (ADLS)** | Source, destination, and reporting containers |
| **HTTP Linked Service** | GitHub CSV ingestion |
| **ADF Mapping Data Flows** | Visual multi-branch ETL transformation |

<br>

---

## 📁 Repository Structure

```
adf-pipeline/
├── pipeline/
│   ├── ProdPipeline.json          # Master orchestrator pipeline
│   ├── pipelinemanager.json       # Ingestion + file move + GIT trigger
│   ├── OnlySelectedFiles.json     # Metadata-driven file filter + copy
│   ├── pipelineGIT.json           # HTTP (GitHub) → ADLS ingestion
├── dataflow/
│   └── transformCSV.json          # Multi-branch Mapping Data Flow
├── dataset/
│   └── (ADF dataset definitions — CSV source, HTTP, ADLS sink)
├── linkedService/
│   ├── AzureBlobStorage.json      # Azure Storage linked service config
│   └── HttpServer.json            # GitHub HTTP linked service config
├── trigger/
│   └── (Schedule trigger for ProdPipeline)
├── factory/
│   └── (ADF factory-level settings)
├── publish_config.json            # ADF publish branch configuration
└── README.md
```

<br>

---

## 🚀 Getting Started

### Prerequisites
- An active **Azure subscription**
- An **Azure Data Factory** instance
- An **Azure Storage Account**
- Access to this GitHub repository

### Setup Steps

**1. Clone this repository**
```bash
git clone https://github.com/Shahzaman-Jalil/adf-pipeline.git
```

**2. Connect ADF to this repo**

In ADF Studio: `Manage → Git configuration → Connect` and point to this repository.

**3. Update Linked Services**

After importing, re-enter credentials for:
- `AzureBlobStorage.json` — your Storage Account connection string
- `HttpServer.json` — your GitHub raw file base URL

> ⚠️ **Note:** Linked Service credentials are **not stored** in this repository for security reasons. You must re-enter them after import.

**4. Upload source data**
```
Upload Fact_Sales_1.csv → source container
```

**5. Publish & Run**

Click **Publish All** in ADF Studio, then trigger `ProdPipeline` manually or wait for the schedule trigger.

<br>

---

## 👤 Author

**Shah Zaman Jalil**

[![GitHub](https://img.shields.io/badge/GitHub-Shahzaman--Jalil-181717?style=flat&logo=github)](https://github.com/Shahzaman-Jalil)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-shahzaman-jalil-0A66C2?style=flat&logo=linkedin)](https://www.linkedin.com/in/shah-zaman-jalil)

---

*Built as part of a structured Azure Data Engineering learning curriculum — **Phase 2: Cloud Pipelines & Batch Processing***
