# ZDDM Domain (Demographics) — Analysis Patterns

## Domain Structure

ZDDM programs derive demographics variables in a ZD (intermediate) domain, then transform to DM via the `%zd_dm` macro. Key variables:
- **ARMCD / ARM**: Planned treatment arm code and description
- **ACTARMCD / ACTARM**: Actual treatment arm code and description
- **ARMNRS**: Reason for non-assignment to arm
- **ACTARMUD**: Description of actual unplanned treatment

## Program Flow

```
%macro zddm;
  1. %sdtm_raw_data_process(domain=zd, prid=DM1001)  → creates zd_DM1001_mac1output
  2. Custom code section → derives ARM/ACTARM/ARMNRS variables
  3. %combine_sdtm(domain=zd, prid=DM1001, input_dataset=...)
  4. %seq(domain=zd, ...)
  5. %sdtm_output(domain=zd, ...)
  6. %zd_dm(domain=dm, ...)  → transforms ZD to DM
  7. %sdtm_output(domain=dm, ...)
%mend;
```

## ARM / ARMCD Derivation Patterns

**Source**: raw.trtasgn (treatment assignment — planned/randomized arm)

```
Pattern:
  1. Read raw.trtasgn, keep SUBJID and TRTCD
  2. Take last record per subject (proc sort + last.subjid)
  3. Map TRTCD to ARMCD/ARM with if-then blocks
```

Description style:
```
Derive from raw.trtasgn with:
- when TRTCD = 1, ARMCD = "CODE1",
- when TRTCD = 2, ARMCD = "CODE2",
...
- If [screen failure condition], then ARMCD is set to blanks.
- If the record has TRTCD missing, then ARMCD is also set to blanks.
```

## ACTARM / ACTARMCD Derivation Patterns

**Source**: raw.trtdspn (treatment dispensing — what was actually given)

```
Pattern:
  1. Read raw.trtdspn, keep SUBJID and TRTCD
  2. Take last dispensation record per subject
  3. Map TRTCD to ACTARMCD/ACTARM with if-then blocks (same structure as ARM)
```

The key difference from ARM: ACTARM uses the dispensing dataset (what was actually administered) vs. the assignment dataset (what was planned).

## ARMNRS (Non-Assignment Reason) Patterns

**Override priority order** (from highest to lowest):

1. **Screen Failure**: `if screenf=1 then armnrs="SCREEN FAILURE"` — from raw.ds6001 where DSDECOD = "SCREEN FAILURE"
2. **Not Assigned**: `if trtcd=. and screenf=. then armnrs="NOT ASSIGNED"` — no treatment code assigned
3. **Assigned Not Treated**: `if trtcd^=. and ecoccur ne 'Y' and screenf=. then armnrs="ASSIGNED,NOT TREATED"` — assigned but never dosed

Description style:
```
If DS6001.DSDECOD_13 = "SCREEN FAILURE", then ARMNRS = "SCREEN FAILURE";
else if the subject was assigned treatment but ECOCCUR != "Y", then ARMNRS = "ASSIGNED, NOT TREATED";
else if TRTCD is missing, then ARMNRS = "NOT ASSIGNED";
else ARMNRS is set to blanks.
```

## ACTARMUD Patterns

Typically simple:
```
Set to blanks when ARMCD is not missing and ARMCD = ACTARMCD.
```

In more complex studies, may describe unplanned treatment if ACTARM differs from ARM.

## Common Source Datasets

| Dataset | Contains | Key Fields |
|---------|----------|-----------|
| raw.trtasgn | Treatment assignment (planned) | SUBJID, TRTCD |
| raw.trtdspn | Treatment dispensing (actual) | SUBJID, TRTCD, VISID, PKGDTTMC |
| raw.ec1001 | Exposure (dosing confirmation) | SUBJID, ECOCCUR, VISITNUM |
| raw.ds6001 | Disposition | SUBJID, DSDECOD, DSDECOD_13, FORMEID |

## Key SAS Patterns to Recognize

1. **Code-value mapping blocks**: Series of `if trtcd=N then do; armcd="X"; arm="Y"; end;`
2. **Last per subject**: `proc sort; by subjid trtcd; data; set; by subjid; if last.subjid;`
3. **Dosing flag derivation**: `if ecoccur='Y' then dosed=1` → used later in ARMNRS
4. **Screen failure flag**: `data screenf; set raw.ds6001; where DSDECOD_13="SCREEN FAILURE";`
5. **Override cascade**: Multiple if-then blocks that set ARM/ARMNRS, with screen failure last (overrides all)
6. **Visit-specific dosing flags**: Some studies use `where VISID>=X` for period-specific dispensing (e.g., `dspwk24fl`)
7. **Treatment code exclusion**: `if trtcd=15 then delete` — codes that represent non-treatment records

## Variations Across Studies

- **KIAC**: Uses separate trtasgn (ARM) and trtdspn (ACTARM) with 14 treatment codes including responder/non-responder rerandomization
- **DSAG**: Similar structure but may have different treatment code ranges and arm names
- **DSAF**: May include additional visit-level dosing flags (v3dosefl, v7dosefl) for more complex arm assignment based on which visits had dosing
