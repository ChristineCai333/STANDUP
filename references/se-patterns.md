# SE Domain (Subject Elements) — Analysis Patterns

## Domain Structure

SE records one row per subject per study element (screening, treatment periods, follow-up). Key variables:
- **ETCD**: Element code identifying which study phase
- **SESTDTC**: Element start datetime
- **SEENDTC**: Element end datetime
- **SEUPDES**: Unscheduled element description

## Treatment Period (TPT) Concept

SE programs organize derivation by period:
- TPT = 0: Screening
- TPT = 1: Induction / initial treatment period
- TPT = 2-3: Maintenance / extended treatment periods
- TPT = 4: Standard of Care (SOC) or rescue
- TPT = 5: Follow-up

## ETCD Derivation Patterns

Typically derived from VISITNUM ranges crossed with treatment code (TRTCD/TRTCD0):

```
Pattern: if VISIT_RANGE then if TRTCD = X then ETCD = 'CODE'
```

Look for:
- Visit number ranges (e.g., `2<=input(visitnum, best.)<3`)
- Treatment code conditions (e.g., `trtcd0=1`, `trtcd in (1,2,3)`)
- SOC flag conditions (e.g., `scan(armcd, 3,'_')='SOC'`)
- Follow-up visits (e.g., `visitnum in ('801','802')`)
- Appended disposition data (DS6001 forms for early termination)

## SESTDTC Derivation Patterns

Different sources per period:

**Screening (TPT=0):**
- Source: raw.ds2001 (disposition/informed consent)
- Field: DSSTDAT_IC
- Filter: FORMEID = 'DS2001_LV1' and DSSCAT_IC = 'STUDY'

**Treatment periods (TPT=1,2,3):**
- Source: raw.ec1001 (exposure/dosing)
- Field: ECSTDAT || 'T' || ECSTTIM (date-time concatenation)
- Filter: FORMEID = 'EC1001_LV6', ECOCCUR ne 'N'
- Selection: First record per subject per TPT (earliest dose in period)

**SOC period (TPT=4):**
- Source: raw.ds6001 (disposition) merged with raw.sv1001 (visits)
- Logic: Discontinuation date from DS6001_LV5 where DSDECOD != 'COMPLETED'
- Or: First visit date on/after SOC start

**Follow-up (TPT=5):**
- Multiple source strategy:
  1. Last visit-N assessment date across SV, LAB, VS, eCOA datasets
  2. For early-discontinued: last ED date from DS6001_LV6, LAB, eCOA
- Macro pattern: `%w24_date()` or `%w52_date()` iterates across datasets
- Selection: Last date per subject across all sources

## SEENDTC Derivation Patterns

**Screen failure (TPT=0):**
- Source: raw.ds6001, FORMEID = 'DS6001_LV4'
- Field: DSSTDAT

**Treatment end (various TPTs):**
- Disposition end: raw.ds6001, FORMEID = 'DS6001_LV5' (treatment discontinuation)
- Activity end: Last date per subject across multiple sources (LAB, eCOA, VS, SR, EC, SV, TRTDSPN) filtered to VISITNUM < 801
- Take whichever is later between disposition and activity

**Follow-up end (TPT=5):**
- Source: raw.ds6001, FORMEID = 'DS6001_LV7' (study completion/withdrawal)
- Combined with: Last follow-up assessment across LAB, eCOA, VS for visits >= 801

**Lag/overwrite logic (critical):**
- After all periods calculated, sort by subject descending TPT
- Each element's SEENDTC is overwritten with the next element's SESTDTC
- This ensures continuity: end of one period = start of next

## Common Source Datasets

| Dataset | Contains | Common Fields |
|---------|----------|--------------|
| raw.ds2001 | Informed consent/screening | DSSTDAT_IC, FORMEID, DSSCAT_IC |
| raw.ds6001 | Disposition events | DSSTDAT, FORMEID, DSDECOD, ROW_ILB |
| raw.ec1001 | Exposure/dosing | ECSTDAT, ECSTTIM, ECENDAT, ECENTIM, ECOCCUR, FORMEID |
| raw.sv1001 | Subject visits | VISDAT, VISITNUM, VISITOCCUR |
| raw.trtasgn | Treatment assignment | TRTCD, SUBJID |
| raw.lab | Lab results | LBDTM, VISIT |
| raw.vs1001 | Vital signs | VSDAT, VSTIM, VSPERF |
| raw.sr1001 | Skin response | SRDAT, SRTIM |
| raw.trtdspn | Treatment dispensing | TRTCD, PKGDTTMC, VISID |
| eCOA datasets | Electronic outcomes | ECOASTDT, ECOAENDT, VISIT |

## Key SAS Patterns to Recognize

1. **Date concatenation**: `strip(DAT)||'T'||substr(strip(TIM),1,5)` → datetime from date+time fields
2. **First/last per group**: `proc sort; by subjid var; data; set; by subjid var; if first.subjid;`
3. **Period assignment**: `if VISIT_RANGE then tpt=N` → assigns records to periods
4. **Nodupkey dedup**: `proc sort nodupkey; by subjid tpt;` → one record per subject/period
5. **Retain + lag**: `retain temp; lag_var=lag(var);` → carry forward / overwrite with adjacent values
6. **Macro iteration**: `%macro_name(indata=..., cond=..., datevar=...)` → repeated logic across datasets
7. **Set stacking**: `data combined; set ds1 ds2 ds3;` → combine multiple sources
