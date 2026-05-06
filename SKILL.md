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
- TRANSFORMATION_TYPE is "CUSTOM" or "ZD_DM"

### Step 2: Read the SAS Code

Use the Read tool to read the full SAS file. Identify:
- The macro boundary (`%macro ... %mend`)
- The custom code section (between framework macro calls)
- Source datasets used (raw.xxx, sdtm.xxx)

### Step 3: Load Domain Patterns

Determine program type from the filename suffix:
- Filename ends with `_SE` → Read `${CLAUDE_SKILL_DIR}/references/se-patterns.md`
- Filename ends with `_ZDDM` → Read `${CLAUDE_SKILL_DIR}/references/zddm-patterns.md`

Also read `${CLAUDE_SKILL_DIR}/references/examples.md` for output format reference.

### Step 4: Generate Algorithm Descriptions

For each filtered variable, analyze the SAS code and produce an algorithm description:

1. **Locate relevant code**: Find where the variable is assigned/derived in the SAS program
2. **Trace data flow**: Identify source datasets → intermediate steps → final assignment
3. **Describe in natural language**: Following the style in examples.md

### Step 5: Write Output

Create a JSON file with variable-to-description mappings, then run:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/parse_spec.py --mode write --spec "$0" --sas-name "<STEM>" --output "<output_path>" --algorithms "<json_path>"
```

The output Excel file will be created in the same directory as the spec file, named `<STEM>_algorithms.xlsx`.

---

## SAS Code Analysis Guide

### What to Skip
- Framework macro calls: `%combine_sdtm`, `%seq`, `%sdtm_output`, `%zd_dm`, `%sdtm_raw_data_process`
- These are standard pipeline macros — they don't contain variable derivation logic

### What to Analyze
- Data steps between the macro header and framework calls
- Conditional assignments (`if ... then VARIABLE = ...`)
- Merge statements (identify source datasets and join keys)
- Date calculations and format conversions
- Proc sort with nodupkey (deduplication logic)
- Macro definitions within the code (e.g., `%w24_date`)

### Code-to-Description Mapping Rules

| SAS Pattern | Description Format |
|-------------|-------------------|
| `if X=1 then var='A'; if X=2 then var='B';` | "when X = 1, VAR = 'A'; when X = 2, VAR = 'B'" |
| `merge ds1 ds2; by subjid;` | "Merge [ds1 purpose] with [ds2 purpose] by subject" |
| `if first.subjid;` or `if last.subjid;` | "Take first/last record per subject" |
| `var = input(charvar, yymmdd10.);` | "Convert [charvar] to date format" |
| `var = strip(DAT)\|\|'T'\|\|strip(TIM);` | "Set to DAT concatenated with TIM as datetime" |
| `proc sort nodupkey; by subjid var;` | "Deduplicate by subject and [var]" |
| `set ds1 ds2 ds3;` | "Combine records from [ds1], [ds2], [ds3]" |
| `retain; lag_var=lag(var);` | "Overwrite with adjacent record's value (lag logic)" |

---

## Algorithm Description Style

1. **Lead with source**: Start by naming the source dataset(s) or form
2. **Conditional mappings**: Use "when CONDITION, VARIABLE = VALUE" as bulleted list
3. **Date variables**: Describe source form/field, any concatenation, and selection criteria (first/last)
4. **Multi-period derivations**: Organize by period (TPT = 0, 1, 2, etc.) with each as a bullet
5. **Override logic**: Describe as if/else chain in priority order
6. **Simple assignments**: One sentence is sufficient — no bullets needed
7. **Be complete**: List ALL mapping values. Do not use "..." or abbreviate

---

## Edge Cases

- **Variable not found in code**: Write "Derived by standard macro processing (not in custom code section)"
- **TRANSFORMATION_TYPE = "ZD_DM"**: Write "The ZD_DM macro reads in the ZD domain. Any rescreen subjects will have multiple records in ZD (one per SUBJID, all share the same USUBJID). These rescreen subjects are condensed into 1 record per USUBJID in DM. The variable referred to in the INPUT_VAR parameter is the variable from ZD that will be populated in this output variable."
- **Pre-populated description exists**: Preserve it; do not overwrite unless user requests regeneration
- **Column name typos in spec**: The parse script handles fuzzy matching (POST_MACR vs POST_MACRO)
- **Sheet not found**: Try case-insensitive match; report available sheets if no match
