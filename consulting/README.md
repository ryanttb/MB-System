# Consulting Onboarding Bundle

This directory contains practical working docs for MB-System consulting tasks:

- `06-client-handoff-tldr.md`: **short client-facing summary** (what was extracted, minimal recipe, one key limitation).
- `01-build-baseline.md`: repeatable build/install baseline (CMake-first).
- `02-minimal-toolchain-tutorial.md`: first-day operator workflow.
- `03-mb58-field-extraction-map.md`: MB58 parser-to-output mapping guide.
- `04-tn304-fit-assessment-criteria.md`: structured TN304 fit/gap evaluation.
- `05-mb58-mb59-extraction-hotspot-qc.md`: seabed-image parsing concepts, `mbpreprocess`/`mbprocess` parameter files, and exploratory lon/lat sidescan extracts with quantile QC masks.

Suggested order:

0. Skim `06-client-handoff-tldr.md` if you only need the executive summary.
1. Build and validate environment.
2. Run tutorial workflow on a known sample file.
3. Execute MB58 field mapping for required outputs.
4. Run TN304 fit assessment with agreed acceptance criteria.
5. Read the MB58/MB59 extraction + QC note when handing off CSV extracts to client-side algorithms.
