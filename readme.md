# OPD Patient Journey — Turnaround Time & Bottleneck Analysis

A Python + Power BI project analyzing a 62,422-row outpatient event log to answer three operational questions for a hospital OPD (Outpatient Department): where time is lost in the patient journey, which doctors show statistically real (not just noisy) turnaround differences, and whether turnaround time is associated with patient volume.

**Dataset:** [HospitalOPD](https://data.mendeley.com/datasets/j2hd2239dr/1) (Mendeley, DOI: 10.17632/j2hd2239dr.1) — a public, anonymized OPD event log, April 2024–April 2026, covering Appointment, Consultation, Laboratory, Imaging, and Pharmacy activities.

---

## Problem Statement

> Hospital leadership wants to understand where time is lost in the OPD patient journey — from appointment to consultation to ancillary services (lab, imaging) — and which factors (doctor, service type, day-of-week, hour-of-day) are most associated with longer turnaround times, in order to set realistic turnaround-time SLAs and flag operational outliers for review.

*This is a framing scenario for a personal portfolio project, not a request from a real hospital.*

**Scope decisions, stated explicitly:**
- Pharmacy is excluded as a standalone stage (0.22% of appointments — too sparse for reliable per-stage analysis) and reduced to a binary flag.
- Journey time is defined as `MAX(all stage end timestamps) − appointment_start`, since no sequencing field exists to distinguish parallel from sequential stages.
- Language is **associative**, not causal — there is no staffing/capacity denominator in this dataset, so findings describe factors *associated with* elevated journey time, not proven causes.
- "Turnaround time" here means the appointment-to-prescription gap, which includes consultation time. It is not a pure waiting time.

---

## Repository Structure

```
data/
  raw/HospitalOPD_noname.csv                 ← never modified, ground truth
  processed/
    opd_appointment_clean.csv                ← one row per appointment (fact table)
    opd_lab_events_clean.csv                 ← one row per genuine lab test event
    opd_imaging_events_clean.csv             ← one row per genuine imaging event
    doctor_significance_results.csv          ← Mann-Whitney U + effect size, per doctor
    volume_correlation_results.csv           ← Spearman correlation, hourly & daily
    data_quality_log.csv                     ← full audit decision table
notebooks/
  data_understanding.ipynb                   ← full audit, hypothesis testing, cleaning, CSV export
  analysis.ipynb                             ← bottleneck ranking, doctor stats, time patterns
powerbi/
  OPD_Dashboard.pbix
```

**Run order:** `data_understanding.ipynb` must be run first (produces the three clean CSVs from raw) → `analysis.ipynb` (produces the two statistical-results CSVs) → Power BI file (imports all clean/results CSVs; no re-cleaning happens in Power Query, only presentation and modeling).

---

## Data Quality — Why This Section Matters

This dataset required substantial forensic work before any analysis was trustworthy. Every decision below was tested against evidence, not assumed. Full detail in `data_understanding.ipynb`; summarized in `data_quality_log.csv` and on the dashboard’s Data Quality page.

| Issue | Scope | Root Cause | Handling |
|---|---|---|---|
| `Unnamed: 2` fully-null column | 62,422 rows | Export artifact | Dropped at load |
| `lab_end_datetime` malformed (`MM:SS.0`, no date) | 4,702 of 18,029 lab rows (26.1%) | Persistent, ongoing export defect since 2025-03, present in the source file as downloaded from Mendeley — confirmed not test-specific, not ID-range-specific, not a one-time event. Two reconstruction hypotheses tested and **falsified**: (1) pair with `lab_start`’s date as a clock time → 100% negative durations; (2) treat as elapsed duration → distribution structurally capped under 60 min, incompatible with the true ~125 min median | Treated as unrecoverable; excluded from lab turnaround; reported as an open defect, not silently patched |
| Extreme appointment→prescription gaps | 65 rows | Partially explained: 2 of the 5 largest trace to a confirmed single-digit year-entry typo (~365-day offset); remainder unexplained after testing and ruling out record-linkage | Excluded from turnaround regardless of cause |
| True full-row duplicates | 8 rows / 4 pairs | Double data entry | Dropped |
| `lab_id` / `imaging_request_id` not globally unique | 606 / 60 IDs | ID space reused across unrelated appointments weeks-to-months apart; no periodic reuse pattern found | Composite key `(appointment_id, id)` required for all joins/counts — never trust the raw ID alone |
| Row multiplication via prescription × lab cross-join | ~3,000+ rows | Table is exported as a join across prescription orders × lab orders per appointment (confirmed: row count = presc_orders × lab_ids for 93.6% of multi-prescription appointments) | Deduplicated to one row per genuine `(appointment_id, lab_id)` event |
| Imaging incomplete (start logged, no end) | 10 rows | Genuinely unresolved studies | Kept, flagged — a real clinical signal, not an error |
| Duration outliers (prescription/imaging/lab) | Varies | Prescription/imaging: implausible values (negative, or exceeding same-visit norms). Lab: **not** capped — 68.6% of >1-day lab durations are culture/sensitivity tests, which genuinely take 1–3 days by microbiology convention | Domain-informed exclusion rules; lab duration split by `lab_slow_test` flag rather than a blanket cap |
| Non-clinical entries in doctor field | 6 appointments | `Admin` account, an IT-staff account with a distinct doctor_id from the real physician, 2 untitled entries | Excluded from all doctor-level analysis |

---

## Key Findings

### 1. Which stage of the OPD journey contributes the most time?

| Stage | Median | Population |
|---|---|---|
| Appointment → Prescription | 27 min | ~all visits |
| Prescription duration | 0 min | ~all visits |
| Lab (routine) | 121 min | ~2.4% of visits |
| **Lab (culture tests)** | **~3.2 days** | ~1% of visits |
| Imaging | 3 min | ~1% of visits |

For the ~80% of visits that are consult-only, total journey time is trivial. Among visits requiring lab work, **routine lab is already the largest normal-flow contributor — over 4x the appointment gap itself.** Culture-based tests dwarf everything else in the dataset (~38x routine lab), but this is inherent to microbiology turnaround time, not a compressible operational delay.

### 2. Doctor-level outliers — statistically real or just noise?

A naive ranking of 60 doctors by median gap would be misleading: **55 of 60 came back “statistically significant” via Mann-Whitney U** even after Benjamini-Hochberg correction (which limits the expected share of false positives among the results called significant). With sample sizes into the thousands, even trivial differences become significant. Adding **rank-biserial effect size** cut this to a much shorter list:

- **12 doctors (20%) show a large effect (r > 0.5, a conventional rule-of-thumb cutoff)**. For intuition: r = 0.90 corresponds to roughly a 95% chance that a random appointment from that doctor is slower than a random appointment from another doctor. The group splits into a slower group (Doctor A, r = 0.90, 7.0x the population median; Doctor B, r = 0.84, 8.6x) and a **faster** group (multiple doctors running 3–8x quicker than peers with normal same-day rates).
- **Doctor X** is a separate, structural case: 33.0% same-day prescription rate vs. 92.6–98.8% for every peer — a workflow-level anomaly, excluded from the ranking and flagged for operational review rather than treated as a performance score.
- The 8-hour same-day/delayed threshold used throughout is **data-driven**, found via a genuine valley in the population histogram (only 8 of 49,817 appointments fall between 8–16 hours), not an arbitrary cutoff.

**Limits of this comparison:** each doctor is compared with all other doctors, with no adjustment for time of day, specialty, or case mix. Faster doctors may differ in documentation timing rather than speed, and slower doctors may see more complex patients. Treat the list of 12 as a set of hypotheses for review, not a performance ranking. Doctors are labeled by letter here because an unadjusted comparison is not a fair basis for naming individuals.

### 3. Is turnaround time associated with patient volume?

- **Hourly:** Spearman ρ = 0.771, p < 0.0001 — a strong association across hour-of-day buckets: busier hours show longer median turnaround. Caveats: the unit of analysis is the hour bucket, not the patient, and every bucket is weighted equally however few appointments it contains. Low-volume hours have unstable medians (see the hour-3 diagnostic below), so a check excluding hours with fewer than 100 appointments is listed under Future Work. This is an association, not evidence that volume causes delay.
- **Daily:** ρ = 0.667, p = 0.102 — with only 7 data points (one per weekday), a Spearman correlation would need to be about 0.79 or higher to reach p < 0.05, so ρ = 0.667 cannot be distinguished from chance. Reported as inconclusive, not as evidence that no pattern exists.
- An apparent spike at hour 3 was investigated and traced to Doctor A (18% of hour-3 appointments) rather than a genuine time-of-day effect — confirmed by recomputing the median with that doctor excluded (50 min → 29 min, back in line with neighboring hours).

---

## Dashboard

Three pages in `OPD_Dashboard.pbix`:
1. **Overview** — KPIs, stage bottleneck comparison, full reference table
2. **Doctor Performance** — leaderboard (filtered to clinical prescribers, ≥30 appointments), effect-size chart with statistical backing
3. **Time Patterns** — hourly/daily volume vs. turnaround time, with the hour-3 diagnostic explained inline

**Model:** star schema — `opd_appointment_clean` (fact) related to `opd_lab_events_clean` and `opd_imaging_events_clean` (many-to-one on `appointment_id`), and to `DateDim` (one-to-many). Statistical results (`doctor_significance_results`, `volume_correlation_results`) are imported as standalone reference tables, computed in Python and not recreated in DAX — deliberately, since neither Mann-Whitney U nor Spearman correlation belongs in a BI tool’s calculation layer.

---

## Limitations

- All findings are associations from one hospital’s public event log; none is a causal estimate.
- Doctor comparisons are unadjusted for time of day, specialty, and case mix.
- The hourly correlation weights all 24 hour buckets equally, including low-volume ones.
- The 4,702 malformed lab end-timestamps are unrecoverable, so lab turnaround is computed on the remaining rows only.

## Future Work

- Recompute the hourly Spearman correlation excluding hours with fewer than 100 appointments, and as a volume-weighted version, to test how stable ρ = 0.771 is.
- Adjust the doctor comparison for hour of day and specialty (for example with a stratified comparison or a regression model).
- Check whether the “faster” doctors differ in when they enter prescription timestamps.

---

## Tech Stack

- **Python** (pandas, scipy) — data cleaning, hypothesis testing, statistical analysis
- **Power BI** — data modeling, DAX measures, dashboard/report

## Methodology Notes for Reviewers

- Every exclusion rule in this project is backed by a tested hypothesis, not a percentile cutoff applied by default.
- Two reconstruction attempts for the `lab_end_datetime` defect were tried and explicitly rejected when the evidence didn’t support them — documented as findings, not hidden.
- Statistical significance was checked against effect size before being reported as a finding, avoiding the standard large-sample-size trap.
- All sample-size floors (30 appointments/doctor, exclusion of <1% pharmacy volume) are stated and justified, not applied silently.
