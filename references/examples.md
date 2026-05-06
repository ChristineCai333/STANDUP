# Algorithm Generation Examples

This file contains worked examples showing how SAS code maps to ENGLISH_ALGORITHM_DESCRIPTION text. Use these as templates for generating descriptions.

---

## Example 1: KIAC_SE — ETCD (Element Code)

### SAS Code Pattern:
```sas
if visitnum='1' then do;  
    etcd='SCRN'; tpt=0;
end;
if 2<=input(visitnum, best.)<3 then do;
    if trtcd0=1 then do; etcd = 'LYQ2W';tpt=1;end;
end; 
if 3<=input(visitnum, best.)<7 then do;
    if trtcd0=1 then do; etcd = 'LYQ4WP2';tpt=2;end;
end; 
if 7<=input(visitnum, best.)<=10 and not f then do;
    if trtcd0=1 then do; etcd = 'LYQ4WP3';tpt=3;end;
end; 
if (visdat>=socstdt>.) then do;
    if trtcd0=1 then do; etcd = 'SOC';tpt=4;end;
end; 
if visitnum in ('801','802') then do;  
    etcd = 'FU';tpt=5;
end;
```

### Generated Algorithm:
```
Assign based on VISITNUM and TRTCD0.
- VISITNUM = 1, ETCD = 'SCRN' ; 
- 2 ≤ VISITNUM < 3, TRTCD0 = 1, ETCD = 'LYQ2W'; 
- 3 ≤ VISITNUM < 7, TRTCD0 = 1, ETCD = 'LYQ4WP2' ; 
- 7 ≤ VISITNUM ≤ 10, TRTCD0  = 1, ETCD = 'LYQ4WP3'; 
- If VISDAT >= SOCSTDT), ETCD = 'SOC'; 
- VISITNUM in ('801', '802'), ETCD = 'FU'. Additionally, patients with a DS6001_LV7 disposition form are appended with ETCD = 'FU'.
```

### Pattern Notes:
- Conditional assignments become bulleted mappings
- Visit ranges preserved as inequalities
- Additional data steps that append records mentioned at the end

---

## Example 2: KIAC_SE — SESTDTC (Element Start Date)

### SAS Code Pattern:
```sas
/* Screening start date from DS2001 */
data SESTDTC_scrn;
    set raw.ds2001(...);
    if ROW_ILB ^= 'Y' and upcase(strip(DSSCAT_IC))="STUDY" and formeid = 'DS2001_LV1';
    SESTDTC = strip(DSSTDAT_IC);
    tpt=0;

/* Treatment start from EC1001 */
data st_ex_;
    set raw.ec1001(...);
    if ROW_ILB ^= 'Y' and FORMEID = 'EC1001_LV6' and ecoccur ne 'N';
    if EcSTTIM ne '' and EcSTDAT ne '' 
    then SESTDTC = strip(EcSTDAT)||'T'||substr(strip(EcSTTIM),1,5);
    if 2<=input(visitnum, best.)<3 then tpt=1;
    if 3<=input(visitnum, best.)<7 then tpt=2;
    if 7<=input(visitnum, best.)<=10 then tpt=3;
/* First record per subject per tpt */
data sestdtc_tx;
    set st_ex_;
    by subjid tpt sestdtc;
    if first.tpt;

/* SOC start from discontinuation */
data ex_soc;
    merge sv1001(in=a) disc_soc(in=b);
    by subjid;
    if a and b;
    sestdtc=put(socstdt,yymmdd10.);
    tpt=4;

/* Follow-up start: last date across multiple datasets at visit 10 */
%w24_date(indata=sv1001, cond=..., outdata=sv, datevar=visdat);
%w24_date(indata=lab, cond=..., outdata=lab, datevar=LBDTM);
...
```

### Generated Algorithm:
```
For Screening(TPT = 0), set to DS2001.DSSTDAT_IC where FORMEID = "DS2001_LV1' and DSSCAT_IC = 'STUDY';
- For Treatment period (TPT = 1, 2, 3), set to ECSTDAT || 'T' || ECSTTIM from EC1001_LV6. Take the earliest record per subject per tpt;
- For SOC (TPT = 4), set to discontinuation date from DS6001_LV5, where DSDECOD != 'COMPLETED';
- For follow up start date (TPT = 5), set to last date across (1. The last visit 10 assessment date from SV, LAB, VS, eCOA; 2. For early-discontinued patients, set to the last ED date from DS6001_LV6, LAB, eCOA, before VISIT 801. 
```

### Pattern Notes:
- Multi-period date derivations organized by TPT value
- Source dataset and specific form/field referenced
- Selection criteria (first/last per group) mentioned
- Date concatenation logic described

---

## Example 3: KIAC_SE — SEENDTC (Element End Date)

### SAS Code Pattern:
```sas
/* Screen failure end from DS6001_LV4 */
data seendtc_2;
    set raw.ds6001(...);
    if formeid = 'DS6001_LV4' and ROW_ILB ne 'Y';
    tpt=0;
    SEENDTC = DSSTDAT;

/* Treatment end from DS6001_LV5 */  
data seendtc_txed;
    set raw.ds6001(...);
    if formeid in ('DS6001_LV5') and row_ILB ne 'Y';
    SEENDTC = DSSTDAT;

/* Last activity across sources for treatment end */
data tx_end;
    set seendtc_txed lb_end qs_end vs_end sr_end ec_end sv_end trtdspn_end;
    proc sort; by subjid endtc;

/* Follow-up end from DS6001_LV7 + last FU assessments */
data seendtc_ptfu;
    set raw.ds6001(...);
    if formeid = 'DS6001_LV7' and row_ILB ne 'Y';
    SEENDTC = DSSTDAT;

/* Lag logic: overwrite SEENDTC with next element's SESTDTC */
data se3;
    retain temp;
    lag_sest=lag(sestdtc);
    if not first.subjid then do;
        if seendtc^= lag_sest then seendtc=lag_sest;
    end;
```

### Generated Algorithm:
```
For TPT = 0, SEENDTC set to DS6001_LV4 screen failure date; 
- For Treatment disposition end, SEENDTC set to DSSTDAT from DS6001_LV5; 
- For Treatment activity end , set to last date per subject across (LAB, LB9001, eCOA, VS1001, SR1001, EC1001, SV1001, TRTDSPN) source data with each data filtered by VISITNUM < 801;
- For Follow-up end date, set to last date per subject across (DS6001_LV7, LAB, eCOA, VS1001) follow up visits;
Finally, data is sorted and each element's SEENDTC is overwritten with the next element's SESTDTC with lag.
```

### Pattern Notes:
- Period-organized description
- Multiple source datasets listed together when combined
- Lag/overwrite logic described as final step
- Filter conditions (VISITNUM < 801) mentioned

---

## Example 4: DSAG_ZDDM — ARMCD (Arm Code)

### SAS Code Pattern:
```sas
if trtcd=1 then do;
    armcd="E6Q2_S1";
    arm="Eltrekibart 600 mg IV Q2W Stage 1";
end;
if trtcd=2 then do;
    armcd="M3Q4_S1";
    arm="Mirikizumab 300 mg IV Q4W Stage 1";
end;
...
if screenf=1 then do;
    armcd='';
    arm='';
end;
if nott=. and screenf=. and arm='' then do;
    armcd='';
    armnrs="NOT ASSIGNED";
end;
```

### Generated Algorithm:
```
Derive from raw.trtasgn with:
- when TRTCD = 1, ARMCD = "E6Q2_S1",
- when TRTCD = 2, ARMCD = "M3Q4_S1",
- when TRTCD = 3, ARMCD = "E6Q2M3Q4_S1",
- when TRTCD = 4, ARMCD = "PBOQ2_S1",
...
- when TRTCD = 14, ARMCD = "PBOQ2_M3Q4_M2Q4_N_S1".
- If DS6001.DSDECOD_13 = "SCREEN FAILURE", then ARMCD is set to blanks.
- If the record has TRTCD missing, then ARMCD is also set to blanks.
```

### Pattern Notes:
- Source dataset identified first (raw.trtasgn)
- Code-value mappings as numbered list
- Override conditions listed after mappings
- All TRTCD values and their corresponding codes listed

---

## Example 5: DSAG_ZDDM — ARMNRS (Arm Non-Assignment Reason)

### SAS Code Pattern:
```sas
if nott=1 and screenf=. and ecoccur='' then do;
    armnrs="ASSIGNED,NOT TREATED";
    actarm='';
    actarmcd='';
end;

if nott=. and screenf=. and arm='' then do;
    armcd='';
    arm='';
    actarmcd='';
    actarm='';
    armnrs="NOT ASSIGNED";
end;

if screenf=1 then do;
    armcd='';
    arm='';
    armnrs="SCREEN FAILURE";
    actarmcd=''; 
    actarm='';
end;
```

### Generated Algorithm:
```
If DS6001.DSDECOD_13 = "SCREEN FAILURE", then ARMNRS = "SCREEN FAILURE";
else if the subject was assigned a treatment (TRTCD is not missing), but with EOCCUR != "Y" and is not a SCREEN FAILURE, then ARMNRS = "ASSIGNED, NOT TREATED";
else if the subject has TRTCD missing, and is not a SCREEN FAILURE, then ARMNRS = "NOT ASSIGNED";
else ARMNRS is set to blanks.
```

### Pattern Notes:
- Override logic described as if/else chain
- Conditions translated from code flags to business meaning (screenf=1 → "SCREEN FAILURE")
- SAS flags (nott, screenf) interpreted in context of their derivation
- Priority order matters: screen failure checked first

---

## Example 6: DSAG_ZDDM — ACTARMUD (Actual Arm Unplanned Description)

### SAS Code Pattern:
```sas
if armcd^='' and armcd=actarmcd then ACTARMUD='';
```

### Generated Algorithm:
```
Set to blanks when ARMCD is not missing and ARMCD = ACTARMCD.
```

### Pattern Notes:
- Simple conditional → concise one-line description
- No bullet list needed for single-condition logic

---

## Style Guidelines Summary

1. **Start with source**: Name the source dataset(s) at the beginning
2. **Conditional mappings**: Use "when X = value, Y = result" format in bulleted lists
3. **Date derivations**: Describe source form/field, concatenation logic, and selection criteria
4. **Multi-period**: Organize by TPT/period value
5. **Override logic**: Describe as if/else chain with priority order
6. **Deduplication**: Mention "first/last per subject" or "per group" when relevant
7. **Simple conditions**: One sentence is sufficient
8. **Completeness**: List ALL mapping values, don't abbreviate with "..."
