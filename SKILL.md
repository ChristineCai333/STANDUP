---
name: generate-algorithm
description: Generate ENGLISH_ALGORITHM_DESCRIPTION for SDTM variables by analyzing SAS code. Use when the user asks to "generate algorithm", "generate algorithm descriptions", "create variable algorithms from SAS code", provides a spec xlsx and SAS file for algorithm generation, or mentions ENGLISH_ALGORITHM_DESCRIPTION, SDTM algorithm, or variable derivation documentation.
arguments: [spec, code]
argument-hint: <spec.xlsx> <code.sas>
allowed-tools: Bash(python3 *) Read
---

## Purpose

Generate natural-language algorithm descriptions (ENGLISH_ALGORITHM_DESCRIPTION) for SDTM variables by reading SAS custom code and mapping each variable's derivation logic to a clear text description.

## Invocation

```
/generate-algorithm <spec.xlsx> <code.sas>
```

- `spec.xlsx`: Path to the specification file containing variable definitions
- `code.sas`: Path to the SAS program with custom derivation code

## Workflow

### Step 1: Extract Variables from Spec

Run the parse script to identify which variables need algorithm descriptions:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/parse_spec.py --mode extract --spec "$0" --sas-name "<STEM>"
```

Where `<STEM>` is the SAS filename without extension (e.g., `KIAC_SE` from `KIAC_SE.sas`).

This returns JSON with filtered variables where:
- ALGORITHM_STATUS is "CHGREQ" or "CHGOPT"
- TRANSFORMATION_TYPE is "CUSTOM", "ZD_DM", "LBSTRESC_CLRM", "VISITNUM_CLRM", "DIRECT_CONDITIONAL", or "DIRECT"

### Step 2: Read and Deeply Analyze the SAS Code

Use the Read tool to read the FULL SAS file. Then carefully identify:
- The macro boundary (`%macro ... %mend`)
- The custom code section (between framework macro calls)
- **ALL source datasets used** (raw.xxx, sdtm.xxx) — note which variables are kept from each
- **ALL data steps** that create intermediate or final datasets
- **ALL assignments to each target variable** — trace through every data step where it appears
- **The data flow**: which intermediate datasets feed into which later steps
- **Any internal macro definitions** and what they do (read the full macro body)

### Step 3: Load Domain Patterns

Determine program type from the filename suffix:
- Filename ends with `_SE` → Read `${CLAUDE_SKILL_DIR}/references/se-patterns.md`
- Filename ends with `_ZDDM` → Read `${CLAUDE_SKILL_DIR}/references/zddm-patterns.md`
- Filename ends with `_LBXL` → Read `${CLAUDE_SKILL_DIR}/references/lbxl-patterns.md`

Also read `${CLAUDE_SKILL_DIR}/references/examples.md` for output format reference.

### Step 4: Generate Algorithm Descriptions

For each filtered variable, perform a **deep trace** through the entire SAS program:

#### 4a. Exhaustive Variable Search

Scan the ENTIRE SAS program (not just the first occurrence) for every place the variable appears:
- Direct assignments: `VARIABLE = value;` or `VARIABLE = expression;`
- Conditional assignments: `if CONDITION then VARIABLE = value;`
- Assignments inside do-blocks: `if X then do; VARIABLE = ...; end;`
- Retain statements: `retain VARIABLE;`
- Keep/drop statements that include the variable
- Rename statements: `rename OLD = VARIABLE;`
- Input/format conversions applied to the variable
- Lag/overwrite logic: `VARIABLE = lag(VARIABLE);`
- Where the variable is set implicitly via `merge` or `set` from a dataset that contains it

#### 4b. Trace Source Data Lineage

For each assignment found, trace backward to identify:
- **Source dataset(s)**: The raw.xxx or intermediate dataset the value originates from (e.g., `raw.ds6001`, `raw.ec1001`, `sdtm.dm`)
- **Source variable(s)**: The specific variable(s) from that source used in the derivation (e.g., `DSSTDAT_IC`, `ECSTDAT`, `TRTCD`)
- **Filter conditions**: Any WHERE clauses, IF conditions, or subsetting that restrict which records contribute (e.g., `FORMEID = 'DS6001_LV1'`, `ROW_ILB ^= 'Y'`)
- **Join keys**: If a merge is involved, what variables are used in the BY statement

#### 4c. Capture All Specific Values

Document every concrete value the variable can take:
- All literal assignments: List every value (e.g., `'SCRN'`, `'LYQ2W'`, `'FU'`, `"SCREEN FAILURE"`)
- All conditions that lead to each value (e.g., `when TRTCD = 1, ARMCD = "E6Q2_S1"`)
- Blank/missing assignments: When the variable is set to `''` or `.`
- Default/else conditions: What happens when no condition matches
- Do NOT abbreviate with "..." — list ALL mapped values found in the code

#### 4d. Understand Intermediate Steps

If the variable's value passes through intermediate datasets or temporary variables:
- Trace the full chain: source dataset → intermediate dataset(s) → final output
- Note any transformations along the way (e.g., deduplication, sorting, first/last per group)
- Note any macro calls that process the variable (describe what the macro does to it)

#### 4e. Compose the Algorithm Description

Synthesize all findings into a natural-language description following examples.md style:
1. **Lead with source**: Always name the source dataset and source variable(s) first
2. **State the mapping/logic**: Describe conditional assignments, value mappings, or calculations
3. **Include all values**: Every specific value the variable can take must be listed
4. **Describe selection criteria**: first/last per group, deduplication, date selection
5. **Note override logic**: If multiple code blocks can set the variable, describe priority order
6. **Mention filters**: Any record-level restrictions that affect which data contributes

### Step 5: Write Output

Create a JSON file with variable-to-description mappings, then run:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/parse_spec.py --mode write --spec "$0" --sas-name "<STEM>" --output "<output_path>" --algorithms "<json_path>"
```

The output Excel file will be created in the same directory as the spec file, named `<STEM>_algorithms.xlsx`.

---

## SAS Code Analysis Guide

### What to Skip
- Framework macro calls: `%combine_sdtm`, `%seq`, `%sdtm_output`, `%zd_dm`, `%sdtm_raw_data_process`, `%lb_xl`
- These are standard pipeline macros — they don't contain variable derivation logic
- Note: `%lb_xl` performs conventional unit conversion (LB→XL) — it is standard processing, but custom code AFTER the macro call (appending, post-processing) IS relevant

### What to Analyze (read the FULL program, not just the first match)
- **Every data step** in the program — the variable may be set, modified, or overwritten in multiple steps
- **Every conditional assignment** (`if ... then VARIABLE = ...`) — collect ALL branches, not just the first
- **Merge statements**: identify source datasets, join keys (BY variables), and in= flags
- **SET statements**: identify which datasets feed into the step, and what variables they carry
- **Date calculations and format conversions**: input(), put(), concatenation with 'T'
- **Proc sort with nodupkey**: deduplication logic (which variables define uniqueness)
- **First./last. processing**: `if first.subjid;` or `if last.tpt;` — which record is kept
- **Retain + lag logic**: values carried forward or overwritten from adjacent records
- **Macro definitions within the code** (e.g., `%w24_date`) — read the macro body to understand what it does to the variable
- **Stacking via SET**: `data combined; set ds1 ds2 ds3;` — the variable may come from multiple sources
- **Rename/label statements** that affect the variable
- **Final dataset**: which data step produces the version of the variable that gets output

### Code-to-Description Mapping Rules

| SAS Pattern | Description Format |
|-------------|-------------------|
| `data X; set raw.ds1001(keep=SUBJID VAR1 VAR2);` | "Source: raw.ds1001 (VAR1, VAR2)" — always name the source dataset and fields |
| `if X=1 then var='A'; if X=2 then var='B';` | "Derived from [source dataset].[X]: when X = 1, VAR = 'A'; when X = 2, VAR = 'B'" |
| `merge ds1(in=a) ds2(in=b); by subjid;` | "Merge [ds1] with [ds2] by SUBJID" — name both datasets and join key |
| `var = sourcevar;` | "Set to [SOURCE_DATASET].[SOURCEVAR]" — always name both dataset and variable |
| `if first.subjid;` or `if last.subjid;` | "Take first/last record per subject" |
| `var = input(charvar, yymmdd10.);` | "Convert [source_dataset].[charvar] to date format" |
| `var = strip(DAT)\|\|'T'\|\|strip(TIM);` | "Set to [source].DAT concatenated with 'T' and [source].TIM as datetime" |
| `proc sort nodupkey; by subjid var;` | "Deduplicate by SUBJID and [var]" |
| `set ds1 ds2 ds3;` | "Combine records from [ds1], [ds2], [ds3]" |
| `retain; lag_var=lag(var);` | "Overwrite with adjacent record's value (lag logic)" |
| `if condition then var = 'SPECIFIC VALUE';` | "When [condition in plain English], VAR = 'SPECIFIC VALUE'" — always include the literal value |
| `where FORMEID = 'XX_LVn' and ROW_ILB ^= 'Y';` | Include these filters: "from [dataset] where FORMEID = 'XX_LVn' and ROW_ILB != 'Y'" |

---

## Algorithm Description Style

1. **Lead with source**: ALWAYS start by naming the source dataset(s) AND source variable(s). Example: "Derived from raw.trtasgn.TRTCD" or "Set to DS2001.DSSTDAT_IC"
2. **Include source variable explicitly**: Name the exact variable from the source dataset that provides the value (e.g., "ECSTDAT from raw.ec1001", not just "from raw.ec1001")
3. **Conditional mappings**: Use "when CONDITION, VARIABLE = VALUE" format in bulleted lists
4. **List ALL specific values**: Every literal value assigned in the code must appear in the description. Never abbreviate or use "..." 
5. **Date variables**: Describe source form/field, any concatenation (e.g., "ECSTDAT || 'T' || ECSTTIM"), and selection criteria (first/last)
6. **Multi-period derivations**: Organize by period (TPT = 0, 1, 2, etc.) with each as a bullet, and state the source for each period
7. **Override logic**: Describe as if/else chain in priority order, stating what condition triggers each value
8. **Simple assignments**: One sentence is sufficient — but still mention the source
9. **Be complete**: List ALL mapping values, ALL conditions, and ALL source datasets involved
10. **Filters and restrictions**: Include record-level filters (e.g., "where FORMEID = 'DS6001_LV1' and ROW_ILB != 'Y'")

---

## Edge Cases

- **Variable not found in code**: Write "Derived by standard macro processing (not in custom code section)"
- **TRANSFORMATION_TYPE = "ZD_DM"**: Write "The ZD_DM macro reads in the ZD domain. Any rescreen subjects will have multiple records in ZD (one per SUBJID, all share the same USUBJID). These rescreen subjects are condensed into 1 record per USUBJID in DM. The variable referred to in the INPUT_VAR parameter is the variable from ZD that will be populated in this output variable."
- **TRANSFORMATION_TYPE = "DIRECT"**: The variable is directly mapped from source without custom logic; describe only the post-macro custom change if one exists
- **TRANSFORMATION_TYPE = "DIRECT_CONDITIONAL"**: A conditional override applied after standard processing; describe the condition and result
- **LBXL programs with multiple PRIDs**: Note when the same logic applies to multiple PRIDs (e.g., "Applied to both LAB and LB9001 PRIDs")
- **XL origin inheritance variables**: Use concise "If missing, set to [LB source variable]" format
- **Pre-populated description exists**: Preserve it; do not overwrite unless user requests regeneration
- **Column name typos in spec**: The parse script handles fuzzy matching (POST_MACR vs POST_MACRO)
- **Sheet not found**: Try case-insensitive match; report available sheets if no match
