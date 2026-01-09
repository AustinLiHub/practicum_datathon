# Copilot Instructions - UofT RSM Practicum Datathon

## Project Overview
A Jupyter-based causal inference analysis for SOBSS (Scotiabank Small Business Services) advertising campaign evaluation. Uses Difference-in-Differences (DiD) econometric modeling to measure the effectiveness of branded advertising across US and Canada regions/channels (SEO, PPC).

## Architecture & Data Flow

### Data Structure (Multi-Sheet Excel → Multi-DataFrame Analysis)
- **Source**: `SOBSS Branded PS Data 2023.xlsx` (4 sheets: US SEO, US PPC, Canada PPC, Canada SEO)
- **Loading Pattern**: Read all sheets, concatenate with `source_sheet` tracking column, then split into region/channel-specific DataFrames
- **Key Columns**: `year`, `month`, `day`, `Hour`, `BRAND_ads_ON_Flag` (treatment indicator), `Country`, `Users`, `NewUsers`, `Trials` (response variables)
- **Cleaning**: Column name normalization (e.g., `'BRAND_ads_ON_Flag (valid during test period only)'` → `'BRAND_ads_ON_Flag'`), drop unused columns (`Hour_Designation`, `cnt_obsrv`)

### Analysis Pipeline
1. **Data Loading & Normalization** → 2. **Channel/Region Splitting** → 3. **Aggregation** (by country totals) → 4. **EDA & Diagnostics** → 5. **DiD Model Estimation** → 6. **Robustness Checks**

### DataFrame Naming Convention
- `df_raw` = concatenated raw data from all sheets
- `df_us_seo`, `df_us_ppc`, `df_ca_ppc`, `df_ca_seo` = channel-specific subsets
- `df_total_US`, `df_total_Canada` = country-level aggregates (used for DiD models)

## Key Patterns & Conventions

### Data Validation Pattern
```python
# Always validate required columns exist before use
for col in ['Users', 'NewUsers', 'Trials']:
    if col not in df_raw.columns:
        raise KeyError(f"Required column '{col}' not found")
    df_raw[col] = pd.to_numeric(df_raw[col], errors='coerce')

# Verify aggregation keys before groupby
agg_keys = ['year', 'month', 'day', 'Hour', 'BRAND_ads_ON_Flag']
for k in agg_keys:
    if k not in df_raw.columns:
        raise KeyError(f"Missing aggregation key: {k}")
```

### Aggregation Pattern (Country Totals)
- Filter by Country string match (`'Country'.str.lower().str.contains('united')` for US, `'canada'` for Canada)
- Group by temporal + treatment columns: `['year', 'month', 'day', 'Hour', 'BRAND_ads_ON_Flag']`
- Sum response variables: `['Users', 'NewUsers', 'Trials']`
- Rename columns with clear region prefix (e.g., `Users` → `total_users_US`)

### DiD Model Structure (Expected)
- **Specification**: `response ~ time + treatment + (time × treatment) + controls`
- **Approach**: Estimate separate models for each response variable (`Users`, `NewUsers`, `Trials`)
- **Implementation**: `statsmodels.formula.api.ols()` with proper fixed effects
- **Interpretation**: Interaction coefficient = treatment effect (effect of BRAND ads on outcome)

### Notebook Cell Organization
- **Cell 1**: Imports + dependencies
- **Cell 2**: Data loading and sheet inspection
- **Cell 3**: Column normalization and cleanup
- **Cell 4**: Type conversion and channel splitting
- **Cell 5**: Aggregation to country totals (critical for DiD setup)
- **Cells 6+**: EDA, visualizations, model estimation, diagnostics

## Dependencies & Environment
```python
# Core data science
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# Econometrics & ML
import statsmodels.api as sm
from statsmodels.formula.api import ols
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split
```

**Setup**: Uncomment cell 1 pip install line if running in fresh environment.

## Workflow Notes

### Before Writing New Code
- Check that required columns exist (use validation pattern above)
- Verify aggregation keys are present in all rows before groupby
- Document any deviations from single-model DiD approach

### When Extending EDA
- Create separate plots for each response variable (Users, NewUsers, Trials)
- Always show parallel trends test before DiD estimation (critical assumption check)
- Include balance checks: compare treatment vs. control group characteristics in pre-period

### When Adding Robustness Checks
- Vary time windows (exclude specific months/periods if needed)
- Check sensitivity to clustering level (by country vs. broader)
- Run models with/without controls to assess covariate balance

## File Organization
- Keep Excel data file in root of `practicum_datathon/`
- Save processed/aggregated DataFrames as intermediate variables in notebook (not separate files unless needed)
- Document business context: SOBSS case study in `../SOBS MMA 2025 Case v1217.pdf`
