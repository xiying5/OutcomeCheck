# OutcomeCheck

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24344065.svg)](https://doi.org/10.5281/zenodo.24344065)
**OutcomeCheck** is a pipeline developed to help researchers: (1) assess the completeness of trial outcome definitions, and (2) detect changes in outcome definitions between different registration versions on ClinicalTrials.gov. It is designed as a companion tool to the article, *"An automated approach to improve clinical trial registration and identify outcome changes on ClinicalTrials.gov"* (Ying X, et al. npj Digital Medicine 2026. doi:10.1038/s41746-026-03045-y).

---

## 1. What the pipeline does

Every trial registered on ClinicalTrials.gov has a version history. For each trial the pipeline retrieves two versions:

| Version | Definition |
| --- | --- |
| **Prospective** | The last version posted before the study start date |
| **Latest** | The most recent version |

It then asks the model to (a) define each outcome in each version and (b) judge whether each element of that definition changed between the versions. The elements are:

- specific measurement
- specific metric
- cutoff
- variable type (aggregation method)
- timepoint

Outcomes must first be matched across versions, since trials add, drop and reword outcomes. Outcomes with no counterpart are marked `[Missing outcome]` or `[Added outcome]` and have no change status.

---

## 2. Requirements

Python 3 with:

`openai` · `pandas` · `numpy` · `openpyxl` · `requests` · `tqdm` · `scikit-learn` · `scipy` · `python-dotenv`

```bash
pip install openai pandas numpy openpyxl requests tqdm scikit-learn scipy python-dotenv
```

The notebooks call the OpenAI API through an Azure OpenAI deployment (the `AzureOpenAI` client). They can be adapted to the OpenAI API directly by replacing the client initialisation.

---

## 3. Setup before running

### 3.1 Working directory

Every notebook begins with:

```python
%cd "/path/to/your/folder"
```

Replace the placeholder with the folder containing this README, the `nctid_new150.txt` file and the `Prompts` folder. All other paths in the code are relative to it. **This line appears in each of the twelve notebooks and must be edited in each one.**

### 3.2 API credentials

The prompting notebooks read four settings, which appear near the top as placeholders:

```python
os.environ['API_ENDPOINT']   = 'XX'   # Azure OpenAI resource endpoint
os.environ['API_DEPLOYMENT'] = 'XX'   # deployment name of the model
os.environ['API_VERSION']    = 'XX'   # Azure API version
os.environ['API_KEY']        = 'XX'   # secret key
```

Supply your own values. The notebooks call `load_dotenv()`, so the preferred approach is to place them in a `.env` file in the working directory and delete the four placeholder lines, which keeps the key out of the notebook. **Do not commit a key to a shared repository.**

### 3.3 Models used in the paper

`API_DEPLOYMENT` is deliberately left blank. The results reported in the paper were produced with three models:

- `gpt-4o-2024-08-06`
- `o1`
- `o3-mini-2025-01-31`

Sampling settings are set in the code: `max_tokens = 4096` and a fixed seed (`66`; one call in *Step 2_Define_Compare_Metric_Type_Time* uses `6666`). For GPT-4o, `temperature = 0` and `top_p = 1` are also set. The o1 and o3-mini models do not accept `temperature` or `top_p`, and the corresponding arguments are omitted.

> A fixed seed reduces but does not eliminate variation. Re-running the pipeline will not reproduce the published output cell for cell.

### 3.4 Directories

The notebooks create the folders they need (`os.makedirs`), so no manual setup is required. Seven are used:

| Folder | Contents |
| --- | --- |
| `out_pro/` | Outcomes, prospective version |
| `inf_pro/` | Trial-level information, prospective version |
| `out_last_cut/` | Outcomes, latest version, de-duplicated for cutoff |
| `out_last_meas/` | ...for specific measurement |
| `out_last_metric/` | ...for specific metric |
| `out_last_time/` | ...for timepoint |
| `out_last_agg/` | ...for aggregation method / variable type |

The latest version is de-duplicated separately for each element.

---

## 4. Input files

### 4.1 Trial list

`nctid_new150.txt` — the NCT IDs to process, one per line. Replace with your own list to run other trials.

### 4.2 Prompts

**`Prompts/Matching/`**

| Prompt | Purpose |
| --- | --- |
| `Task3_mat` | Match outcomes between the two registration versions |

**`Prompts/Define_Compare_Measurement/`**

| Prompt | Purpose |
| --- | --- |
| `Task_prompt1` | Classify the composition of an outcome |
| `Task_prompt2` | Define specific measurement for a single score or event |
| `Task_prompt3` | Define specific measurement for a composite outcome |
| `Task5_meas_simple` | Compare the defined specific measurement between versions |

**`Prompts/Define_Compare_Other elements/`**

| Prompt | Purpose |
| --- | --- |
| `Task1_cut` | Define cutoffs for outcomes in both versions |
| `Task1_metric` | Define specific metric, prospective version |
| `Task1_type` | Define variable type, prospective version |
| `Task2_metric` | Define specific metric, latest version |
| `Task2_type` | Define variable type, latest version |
| `Task4_time` | Compare timepoints between the two versions |
| `Task5_cut_simple` | Compare cutoffs between the two versions |

---

## 5. How the code works (step by step)

Run every notebook in **Step 1_Data preparation** first, then every notebook in **Step 2_Prompting**, in the order below. Later steps depend on files written by earlier ones.

### Step 1 — Data preparation

*No model calls; ClinicalTrials.gov API only.*

**1. `Step 1_Download info for pros version.ipynb`**
Identifies the last version posted before the study start date and retrieves its outcome information.
*Writes:* `out_pro/{nctid}.json`, `inf_pro/{nctid}.json`

**2. `Step 2_Download info for last version.ipynb`**
Retrieves outcome information for the latest version, once per element.
*Writes:* `out_last_cut/{nctid}.json`, `out_last_meas/{nctid}.json`, `out_last_metric/{nctid}.json`, `out_last_time/{nctid}.json`, `out_last_agg/{nctid}.json`

**3. `Step 3_Dup_removal_pros.ipynb`**
Removes duplicated outcome records in the prospective version.
*Writes:* `out_pro/{nctid}_merged.json`

**4. Duplicate removal for each element of the latest version.**
Run all five; each writes `{nctid}_merged.json` into its own folder.

| Notebook | Output folder |
| --- | --- |
| `Step 4_Dup_removal_last_cut.ipynb` | `out_last_cut/` |
| `Step 4_Dup_removal_last_meas.ipynb` | `out_last_meas/` |
| `Step 4_Dup_removal_last_metric.ipynb` | `out_last_metric/` |
| `Step 4_Dup_removal_last_time.ipynb` | `out_last_time/` |
| `Step 4_Dup_removal_last_type.ipynb` | `out_last_agg/` |

### Step 2 — Prompting

*Calls the model.*

**1. `Step 1_Match outcomes.ipynb`**
Matches outcomes between the two versions. **Must be run before the three notebooks below**, which all read its output.
*Reads:* `out_pro/{nctid}_merged.json`, `out_last_cut/{nctid}_merged.json`
*Writes:* `{nctid}_table3.csv` (one file per trial)

**2. `Step 2_Define_Compare_Cut.ipynb`**
Defines cutoffs in both versions and compares them.
*Writes:* `{nctid}_cut.csv`, `cutoff_definition.xlsx`, `cutoff_change_status.xlsx`, `processing_cost_cutoff.xlsx`

**3. `Step 2_Define_Compare_Meas.ipynb`**
Defines the specific measurement in both versions and compares it.
*Writes:* `{nctid}_meas.csv`, `{nctid}_table4.csv`, `measurement_definition.xlsx`, `measurement_change_status.xlsx`, `processing_cost_measurement.xlsx`

**4. `Step 2_Define_Compare_Metric_Type_Time.ipynb`**
Defines the specific metric and aggregation method in both versions, and compares metric, aggregation method and timepoint.
*Reads:* `out_pro/{nctid}_merged.json`, `out_last_metric/{nctid}_merged.json` (metric), `out_last_agg/{nctid}_merged.json` (aggregation method), `out_last_time/{nctid}_merged.json` (timepoint)
*Writes:* `{nctid}_comb.csv`, `metric_aggregation_method_definition_prospective.xlsx`, `metric_aggregation_method_definition_latest.xlsx`, `metric_aggregation_method_timepoint_change_status.xlsx`, `processing_cost_metric_aggregation_method_timepoint.xlsx`

The three Step 2 notebooks are independent of one another and may be run in any order once *Step 1_Match outcomes* has completed.

---

## 6. Output files

### 6.1 Per trial

One CSV per trial, written to the working directory:

| File | Contents |
| --- | --- |
| `{nctid}_table3.csv` | Matched pairs of outcomes across the two versions |
| `{nctid}_cut.csv` | Cutoff, both versions, with change status |
| `{nctid}_meas.csv` | Specific measurement, both versions, with change status |
| `{nctid}_table4.csv` | Full measurement parse, with the model's reasoning |
| `{nctid}_comb.csv` | Metric, aggregation method and timepoint, both versions, with change status and the model's reasoning |

### 6.2 Aggregated across trials

Excel workbooks appended to as each trial is processed:

- `metric_aggregation_method_definition_prospective.xlsx`
- `metric_aggregation_method_definition_latest.xlsx`
- `cutoff_definition.xlsx`
- `measurement_definition.xlsx`
- `metric_aggregation_method_timepoint_change_status.xlsx`
- `cutoff_change_status.xlsx`
- `measurement_change_status.xlsx`

These correspond to the seven files published with the paper for each model.

### 6.3 Cost and timing

- `processing_cost_cutoff.xlsx`
- `processing_cost_measurement.xlsx`
- `processing_cost_metric_aggregation_method_timepoint.xlsx`

Input tokens, output tokens and elapsed time per trial and task.

> **Note:** Because the workbooks are *appended to* rather than overwritten, re-running the pipeline over the same trials without first deleting or moving them will add a second copy of every row.

---

## 7. Notes

### Terminology

Three names are used for the same concept in different places:

| Where | Name used |
| --- | --- |
| Prompts and this pipeline | Variable type |
| Output columns and files | Aggregation method |
| The paper | Variable type |

"Aggregation method" and "variable type" are the same element. The published data files use "Aggregation method", following the column name produced by the code.

Two category labels also differ from the paper:

| In the output | In the paper |
| --- | --- |
| Change from baseline | Change between two time points |
| End value | Value at a time point |

### Trials with no matched outcomes

If no outcome in the latest version can be matched to any prospective outcome, the comparison notebooks produce no change status for that trial. Its rows carry the `[Missing outcome]` and `[Added outcome]` placeholders, and the change-status columns are blank.

### Cost

Every model call is logged. Expect the three Step 2 notebooks to dominate both runtime and cost; Step 1 makes no model calls.

### Reproducibility

Model output is not fully deterministic even with a fixed seed, and the model versions above may be retired by the provider. Re-running will produce similar but not identical results. 

---

## 8. Change log

**November 2026.** `Step 2_Define_Compare_Metric_Type_Time.ipynb` was updated so that each task reads the latest-version outcomes from its own folder: `out_last_metric/` for specific metric, `out_last_agg/` for aggregation method and `out_last_time/` for timepoint. The copy posted here earlier read `out_last_cut/` for all three tasks. That folder holds only the outcome title, description, time frame and classes, so the prompts for specific metric and aggregation method did not receive the results-section fields they refer to (measure type, measure of dispersion/precision, unit of measure, statistical analysis method, outcome analysis parameter type). No other notebook or prompt changed.

This applies to the copy of the notebook posted here. The main accuracy results reported in the paper, and the model output published with it, were generated with the results-section fields supplied to the metric and aggregation-method prompts.
