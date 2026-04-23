================================================================================
                         COMPLETE WORKFLOW VISUALIZATION
================================================================================

1. CONFIGURATION (config.R)
   ┌─────────────────────────────────────────────────────────────┐
   │ models: baseline, ar, ar_exog, species_specific, trait     │
   │ run_mvgam = TRUE, run_fable = TRUE                         │
   │ use_ordinal = TRUE  ← Controls if RPS is calculated        │
   │ ordinal_breaks = [0.33, 0.50, 0.67] ← Category boundaries  │
   │ train_years = 20, test_years = 2                           │
   └─────────────────────────────────────────────────────────────┘
                              ↓

2. DATA LOADING (data_functions.R)
   ┌─────────────────────────────────────────────────────────────┐
   │ Bird Counts (1991-2024, 6 species)                         │
   │  + Water Covariates (breed_season_depth, dry_days, etc.)   │
   └─────────────────────────────────────────────────────────────┘
                              ↓
   [year, species, count, breed_season_depth, dry_days, recession, ...]
                              ↓

3. SLIDING WINDOW SPLIT (evaluation.R - fit_sliding_window)
   ┌───────────────────────────────────────────────────────────────────┐
   │ Window 1: Train [1991-2010] → Test [2011-2012]                  │
   │ Window 2: Train [1992-2011] → Test [2012-2013]                  │
   │ Window 3: Train [1993-2012] → Test [2013-2014]                  │
   │ ...                                                               │
   │ Window 7: Train [1997-2016] → Test [2017-2018]                  │
   └───────────────────────────────────────────────────────────────────┘
                              ↓

4. MODEL FITTING (for each window)
   ┌─────────────────────────────────────────────────────────────┐
   │                 MVGAM Models                                │
   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
   │  │  baseline    │  │     AR       │  │   trait      │     │
   │  │ count~series │  │ + covariates │  │ + VAR        │     │
   │  │   nb()       │  │ + mvgam::AR()│  │ + trait map  │     │
   │  └──────────────┘  └──────────────┘  └──────────────┘     │
   │         ↓                ↓                  ↓               │
   │    [MCMC sampling 4 chains × 3000 iterations]              │
   │         ↓                ↓                  ↓               │
   │   Fitted models with posterior distributions               │
   └─────────────────────────────────────────────────────────────┘
                              ↓

5. PREDICTION (for test data)
   ┌─────────────────────────────────────────────────────────────┐
   │ predict(model, newdata = test_data)                         │
   │                                                              │
   │ Returns for each species × year:                            │
   │  - Estimate (mean prediction)                               │
   │  - Q2.5, Q25, Q50, Q75, Q97.5 (quantiles)                  │
   │  - Est.Error (standard error)                               │
   └─────────────────────────────────────────────────────────────┘
                              ↓
   [Species: gbhe, Year: 2011, Estimate: 45.3, Q2.5: 30, Q97.5: 65, ...]
                              ↓

6. EVALUATION - TWO PARALLEL PATHS
   
   ┌──────────────────────────┐         ┌─────────────────────────────┐
   │   NUMERIC EVALUATION     │         │   ORDINAL EVALUATION        │
   │       (CRPS)             │         │   (RPS, if use_ordinal=TRUE)│
   └──────────────────────────┘         └─────────────────────────────┘
              ↓                                      ↓
   ┌──────────────────────────┐         ┌─────────────────────────────┐
   │ Score predicted          │         │ 1. Calculate quantiles from │
   │ distribution vs          │         │    TRAINING data:           │
   │ actual count             │         │    low = quantile(0.33)     │
   │                          │         │    medium = quantile(0.50)  │
   │ CRPS = ∫|F(x)-1{x≥y}|dx │         │    high = quantile(0.67)    │
   │                          │         │                              │
   │ Lower = better           │         │    [Per species!]           │
   └──────────────────────────┘         └─────────────────────────────┘
              ↓                                      ↓
   ┌──────────────────────────┐         ┌─────────────────────────────┐
   │ CRPS for each:           │         │ 2. Convert TEST data to     │
   │  - model                 │         │    categories:              │
   │  - species               │         │    if count < 16.5: "Low"   │
   │  - window                │         │    elif count < 25: "Medium"│
   │                          │         │    elif count < 36.5: "High"│
   │ Example:                 │         │    else: "Very High"        │
   │  baseline, gbhe: 34.4    │         └─────────────────────────────┘
   │  ar, gbhe: 50.3          │                      ↓
   └──────────────────────────┘         ┌─────────────────────────────┐
              ↓                          │ 3. Convert PREDICTIONS to   │
   ┌──────────────────────────┐         │    probabilities:           │
   │ Calculate skill scores:  │         │                              │
   │                          │         │    pred_sd = (Q97.5-Q2.5)/  │
   │ crps_skill =             │         │              (2×1.96)       │
   │   1 - (crps/crps_baseline)│        │                              │
   │                          │         │    prob_low = P(count<16.5) │
   │ Skill > 0 = better than  │         │    prob_medium = P(16.5-25) │
   │             baseline      │         │    prob_high = P(25-36.5)   │
   │ Skill < 0 = worse than   │         │    prob_very_high = P(>36.5)│
   │             baseline      │         │                              │
   └──────────────────────────┘         │    [Using normal approx]    │
                                         └─────────────────────────────┘
                                                     ↓
                                         ┌─────────────────────────────┐
                                         │ 4. Calculate RPS:           │
                                         │                              │
                                         │    Observed: [0,0,1,0]      │
                                         │    (category 3 = "High")    │
                                         │                              │
                                         │    Predicted: [0.1,0.3,0.4, │
                                         │                0.2]          │
                                         │                              │
                                         │    RPS = sum of squared     │
                                         │          cumulative errors  │
                                         │                              │
                                         │    Lower = better           │
                                         └─────────────────────────────┘
                                                     ↓
                                         ┌─────────────────────────────┐
                                         │ rps_skill =                 │
                                         │   1 - (rps/rps_baseline)    │
                                         └─────────────────────────────┘
              ↓                                      ↓
   ┌──────────────────────────────────────────────────────────────────┐
   │                   COMBINE METRICS                                │
   │  [model, species, test_start, crps, crps_skill, rps, rps_skill] │
   └──────────────────────────────────────────────────────────────────┘
                              ↓

7. AGGREGATE ACROSS WINDOWS
   ┌─────────────────────────────────────────────────────────────┐
   │ Combine results from all 7 sliding windows                  │
   │                                                              │
   │ Final metrics table:                                         │
   │  model | species | test_start | crps_skill | rps_skill     │
   │  ──────┼─────────┼────────────┼────────────┼──────────      │
   │  ar    | gbhe    | 2011       | -0.46      | 0.00          │
   │  ar    | gbhe    | 2012       | 0.23       | 0.15          │
   │  ...                                                         │
   └─────────────────────────────────────────────────────────────┘
                              ↓

8. VISUALIZATION (plotting.R)
   ┌─────────────────────────────────────────────────────────────┐
   │ Plot 1: CRPS skill over time (by species)                   │
   │ Plot 2: RPS skill over time (by species)                    │
   │ Plot 3: Combined metrics (faceted grid)                     │
   │ Plot 4: Best model counts (which model wins most often)     │
   └─────────────────────────────────────────────────────────────┘
                              ↓
   results/mvgam_crps_skill_over_time.png
   results/mvgam_rps_skill_over_time.png
   results/mvgam_all_skills_over_time.png
   results/mvgam_best_model_counts.png

================================================================================
KEY CONCEPTS:

CONFIG ordinal_breaks [0.33, 0.50, 0.67]
   ↓
   "Where to draw the lines between Low/Medium/High/Very High"
   
EVALUATION pnorm(low, mean=Estimate, sd=pred_sd)
   ↓
   "What's the probability this prediction falls in each category"

CRPS: "How accurate are my numeric predictions?"
RPS:  "How accurate are my categorical predictions?"

Skill Score = 1 - (model_score / baseline_score)
   > 0: Better than baseline
   = 0: Same as baseline
   < 0: Worse than baseline
================================================================================
