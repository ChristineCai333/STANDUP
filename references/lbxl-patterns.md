# LBXL Domain (Laboratory / Conventional Units) — Analysis Patterns

## Domain Structure

LBXL programs produce two output domains from a single program:
- **LB**: Laboratory results in original/standard units
- **XL**: Laboratory results converted to conventional units (via `%lb_xl` macro)

The program typically processes multiple PRIDs (process IDs) for different lab data sources (central lab, local lab, derived), then applies unit conversion for the XL domain.

## Program Flow

```
%macro lbxl;
  1. %sdtm_raw_data_process(domain=lb, prid=LAB)    → central lab data
     - Custom post-processing (LBSTRESC, LBNRIND fixes)
  2. %combine_sdtm(domain=lb, prid=LAB, ...)
     - Post-processing (VISIT/VISITNUM missing logic, range cleaning)

  3. %sdtm_raw_data_process(domain=lb, prid=LB9001)  → local lab data
  4. %combine_sdtm(domain=lb, prid=LB9001, ...)
     - Post-processing (range cleaning)

  5. %sdtm_raw_data_process(domain=lb, prid=LBDRVD)  → derived lab data
  6. %combine_sdtm(domain=lb, prid=LBDRVD, ...)

  7. %seq(domain=lb, input_datasets=LB9001 LBDRVD LAB)
     - Post-processing (extreme value bounds, LBGLSUCD cleaning)

  8. %sdtm_output(domain=lb, ...)  → final LB output

  9. Split data by source (central lab vs. local lab)
  10. %lb_xl(domain=xl, input_dataset=lb_from_local)   → XL for local labs
  11. %lb_xl(domain=xl, input_dataset=lb_from_lab)     → XL for central labs
  12. Append XL datasets together + post-processing
  13. %sdtm_output(domain=xl, ...)  → final XL output
%mend;
```

## LB Domain Variable Patterns

### LBSTRESC (Character Result/Finding in Standard Format)

**Pattern**: Resolve missing LBSTRESC for character/non-numeric results.

```sas
if (anyalpha(lborres)>0 or indexc(lborres,'>','<','-','_','+')>0
    or (lbtestcd='HCG' and LBGLSUCD='0')) and lbstresc='' then do;
    lbstresc=upcase(lborres);
end;
```

Description style:
```
If LBORRES contains alphabetic characters or special characters (>, <, -, _, +),
or if LBTESTCD='HCG' and LBGLSUCD='0', and LBSTRESC is missing,
then set LBSTRESC = upcase(LBORRES).
EXCEPT: for LBTESTCD='DTIL8': set LBSTRESC = LBORRES;
if LBORRES contains alphabetic characters, set LBSTRESC = upcase(LBORRES).
```

### LBNRIND (Reference Range Indicator)

**Pattern**: Populate missing normal range indicator from alert flag.

```sas
if lbtestcd>'' and lbnrind='' and LBALRTFL>'' then lbnrind=LBALRTFL;
```

Description style:
```
If LBTESTCD is not missing and LBNRIND is blank and LBALRTFL is not missing,
set LBNRIND = LBALRTFL.
```

### VISIT / VISITNUM (Visit Name / Visit Number)

**Pattern**: Clear visit info when date is missing.

```sas
if lbdtc='' then do; visitnum=.; visit=''; end;
```

Description style:
```
If LBDTC is missing, set to missing.
```

### LBORNRLO / LBORNRHI (Original Reference Range Low/High)

**Pattern**: Clean character dot values. Applied across multiple PRIDs.

```sas
if strip(lbornrlo)='.' then lbornrlo='';
if strip(lbornrhi)='.' then lbornrhi='';
```

Description style:
```
If the value is '.' (character dot), set to blank. Applied to both LAB and LB9001 PRIDs.
```

### LBSTNRLO / LBSTNRHI (Standard Normal Range Low/High)

**Pattern**: Set extreme values to missing (data quality bounds).

```sas
if LBSTNRLO<-900000 then LBSTNRLO=.;
if LBSTNRHI>900000 then LBSTNRHI=.;
```

Description style:
```
If LBSTNRLO < -900000, set to missing.
If LBSTNRHI > 900000, set to missing.
```

## XL Domain Variable Patterns

### Central Lab vs. Local Lab Split

A key feature of LBXL programs is splitting lab data before XL processing:
- **Central labs** (input_ds_no_use_in_adam = 'LAB'): May use different conversion specs
- **Local labs** (all other PRIDs): Standard XL processing

Both are processed through `%lb_xl` separately then appended back together.

### XLSTRESC (XL Character Result)

**Pattern**: Populate missing XLSTRESC from XLORRES, with special test handling.

```sas
if xlstresc='' and xlorres^='' then xlstresc=strip(upcase(xlorres));
if xltestcd='DTIL8' then do;
    xlstresc=xlorres;
    if anyalpha(xlorres)>0 then xlstresc=upcase(xlorres);
end;
```

Description style:
```
If XLSTRESC is missing and XLORRES is not missing, set XLSTRESC = upcase(XLORRES).
EXCEPT: for XLTESTCD='DTIL8': set XLSTRESC = XLORRES;
if XLORRES contains alphabetic characters, set XLSTRESC = upcase(XLORRES).
```

### XLSTRESN (XL Numeric Result)

**Pattern**: Convert character result to numeric when appropriate.

```sas
if anyalpha(xlstresc)=0 and xlstresn=. then xlstresn=input(xlstresc, best.);
```

Description style:
```
If XLSTRESC contains no alphabetic characters and XLSTRESN is missing,
set XLSTRESN = input(XLSTRESC, best.).
```

### XLSTRESU (XL Result Unit)

**Pattern**: Clear units when result is missing.

```sas
if xlstresc='' then xlstresu='';
```

Description style:
```
If XLSTRESC is missing, set XLSTRESU to missing.
```

### XLSTNRLO / XLSTNRHI (XL Standard Normal Range)

**Pattern**: Same extreme value bounds as LB domain.

```sas
if XLSTNRLO<-900000 then XLSTNRLO=.;
if XLSTNRHI>900000 then XLSTNRHI=.;
```

Description style:
```
If XLSTNRLO < -900000, set to missing.
If XLSTNRHI > 900000, set to missing.
```

### XL Origin Variables (Inheritance Pattern)

**Pattern**: XL origin/metadata variables inherit from LB when missing. This is a common pattern where XL records populated by `%lb_xl` may lack origin tracking fields.

```sas
if XLALRTFL_ORIGIN='' then XLALRTFL_ORIGIN = LBALRTFL_ORIGIN;
if XLGLSLCD_ORIGIN='' then XLGLSLCD_ORIGIN = LBGLSLCD_ORIGIN;
if XLGLSTCD_ORIGIN='' then XLGLSTCD_ORIGIN = LBGLSTCD_ORIGIN;
if XLGLSUCD_ORIGIN='' then XLGLSUCD_ORIGIN = LBGLSUCD_ORIGIN;
if XLLNCVER_ORIGIN='' then XLLNCVER_ORIGIN = LBLNCVER_ORIGIN;
if XLRESSRC_EVAL=''   then XLRESSRC_EVAL   = LBRESSRC_EVAL;
if XLRESSRC_ORIGIN='' then XLRESSRC_ORIGIN = LBRESSRC_ORIGIN;
if XLUSCHFL_ORIGIN='' then XLUSCHFL_ORIGIN = LBUSCHFL_ORIGIN;
```

Description style:
```
If missing, set to [corresponding LB variable].
```

Examples:
- XLALRTFL_ORIGIN: "If missing, set to LBALRTFL_ORIGIN"
- XLGLSLCD_ORIGIN: "If missing, set to LBGLSLCD_ORIGIN"

### XL Cleanup Rules

**Pattern**: Clear dependent variables when primary result is missing.

```sas
if xlorres='' then xlorresu='';
if xlstresc='' then xlstresu='';
if xltestcd='' then do; xlstresc=''; xlstresn=.; end;
```

## Key SAS Patterns to Recognize in LBXL

1. **Character detection**: `anyalpha(var)>0` → variable contains letters
2. **Special character detection**: `indexc(var,'>','<','-','_','+')>0` → contains operators/symbols
3. **Extreme value bounds**: `< -900000` or `> 900000` → data quality threshold
4. **Character dot cleaning**: `strip(var)='.'` → artifact from numeric-to-character conversion
5. **Central/local lab split**: `if input_ds_no_use_in_adam = 'LAB' then output lb_from_lab`
6. **Conventional unit conversion**: Handled by `%lb_xl` macro (standard framework)
7. **Origin variable inheritance**: `if XL_ORIGIN_VAR='' then XL_ORIGIN_VAR = LB_ORIGIN_VAR`
8. **Test-specific handling**: `if lbtestcd='DTIL8'` → special logic for specific lab tests
9. **Result cascade clearing**: When result is blank, clear associated unit/numeric fields
10. **CNVNRLO/CNVNRHI conversion**: Raw conventional normal range values converted to numeric for `%lb_xl` input

## Common Source Variables

| Variable | Domain | Description |
|----------|--------|-------------|
| LBORRES | LB | Original result |
| LBSTRESC | LB | Character result in standard format |
| LBSTRESN | LB | Numeric result in standard units |
| LBNRIND | LB | Normal range indicator |
| LBALRTFL | LB | Alert flag (source for LBNRIND) |
| LBORNRLO/HI | LB | Original normal range low/high |
| LBSTNRLO/HI | LB | Standard normal range low/high |
| LBGLSUCD | LB | GLS unit code |
| LBTESTCD | LB | Test short name |
| LBDTC | LB | Date/time of collection |
| XLORRES | XL | Original result (conventional units) |
| XLSTRESC | XL | Character result (conventional units) |
| XLSTRESN | XL | Numeric result (conventional units) |
| XLORRESU | XL | Original result unit |
| XLSTRESU | XL | Standard result unit |
| XLSTNRLO/HI | XL | Standard normal range (conventional) |
| CNVNRLO/HI | Raw | Conventional normal range (from raw data) |

## Framework Macros (Skip in Analysis)

- `%sdtm_raw_data_process`: Standard raw data reading/mapping
- `%combine_sdtm`: Merges macro1 output with metadata
- `%seq`: Assigns LBSEQ sequence numbers
- `%sdtm_output`: Writes final domain dataset
- `%lb_xl`: Standard conventional unit conversion (unless custom XL specs override)

## Variations Across Studies

- **DSAF**: Central lab data split with conventional unit variables kept from raw data; DTIL8 special handling
- **Other studies**: May have different PRID sets, different special test codes, or simpler single-source lab processing
- **XL specs modification**: Some studies modify xl_specs between central and local lab processing to handle different conversion logic
