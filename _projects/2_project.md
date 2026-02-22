---
layout: page
title: Automated Data Quality Checks for Annotation Pipeline (R + Batch)
description: Diff-based weekly QC with per-coder issue dumps for non-technical users
img: assets/img/3.jpg
importance: 1
category: work
---

## Overview

In our lab, we annotate infants' reaches to a toy with the goal of analyzing the temporal structure of the behavior. There are more than 500 videos to annotate, so different coders with varying levels of experience work on the task. It is crucial that the quality of annotation is maintained for future analysis. The quality control (QC) for annotation files is performed on a weekly basis.

Each annotation file has six columns:

- Tier (= Body part whose movements were annotated)
- ID of the video annotated
- (Movement) Onset
- (Movement) Offset
- Duration (= Offset - Onset)
- Movement Category Label

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/ELAN_output.png" title="annotated file" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    An example annotation of a video.
</div>

The last offset should be the same for all tiers. Labels should be one of the pre-defined labels, and the offset of previous event should be identical to the onset of the next event.

A naive approach re-checks **every file every week**, which is slow and redundant when most files were already verified the week before.

I built an **automated QC pipeline** that supports both:
- **Weekly incremental QC** (only checks *new or changed* assignments)
- **Monthly full QC** (runs a complete audit)

The system is designed to be usable by **non-technical users** via a **one-click Windows `.bat`** workflow.

---

## Problem

Weekly QC had three recurring pain points:

1. **Unclear action items**: QC outputs were not easily attributable to individual coders
2. **Operational friction**: QC required running scripts manually and interpreting logs.

---

## Solution

### 1) Cloud-synced assignment management (Microsoft OneDrive)

QC targets are defined in a shared assignment spreadsheet stored on **Microsoft OneDrive**.

The pipeline reads directly from a cloud-synced directory structure, allowing:

- Centralized file access across multiple lab machines
- Version-controlled assignment updates via OneDrive sync history
- Seamless collaboration without manual file transfers
- Consistent path resolution across users’ environments

To ensure reproducibility, the pipeline normalizes the spreadsheet into a standardized `reference.tsv`, which serves as the canonical input for downstream QC logic.

This design allows the QC workflow to operate reliably in a **cloud-backed research environment** while remaining executable via local scripts.

### 2) Incremental QC via snapshot diff

To avoid re-checking everything, the pipeline maintains a lightweight state:

- `reference_prev.tsv` - snapshot from the previous run
- `reference_new.tsv` - rows newly added since the previous run (diff)

Weekly QC consumes `reference_new.tsv` when available, falling back to full QC when needed.

### 3) Per-coder "issue dumps" (actionable outputs)

Instead of only providing summary based on annotation error categories, the pipeline produces **per-coder files** containing:

- `failed_log_coded`
- `offset_with_coder`
- `labels_with_coder`
- `continuous_with_coder`

Each QC run writes outputs into a **timestamped folder** (e.g., `qc_by_coder_YYYY-MM-DD_HH:MM`) so results are traceable and never overwritten.

### 4) One-click execution for non-technical users

QC is run via a Windows batch script:

- **Weekly**: incremental by default
- **Monthly**: full QC trigger (e.g., first Monday logic)

This design minimizes training burden: "double-click and wait for Done."

---

## Results / Impact

- **Reduced runtime** by avoiding full re-scans in weekly cycles
- **Improved usability**: coders receive exact rows they need to fix
- **More robust operations**: timestamped outputs + consistent reference state
- **Less human error**: standardized execution path via `.bat`

---

## Tech Stack

- **R** (tidyverse, fs, stringr) for QC logic and reporting
- **Microsoft OneDrive (cloud-synced storage)** for collaborative file management
- **Windows Batch** for one-click automation
- **Tabular reference ingestion** from a shared spreadsheet source
- **File-based state** to enable incremental processing

---

## What I’d Improve Next (Production-minded roadmap)

- Add unit tests for key QC rules and edge cases
- Add run metadata (QC version, input snapshot hash, summary JSON)
- Optional email/slack notification with links to the output folder
- Containerized execution for cross-machine reproducibility

---
