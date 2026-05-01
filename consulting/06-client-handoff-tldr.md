# Client handoff — TLDR

This is a short “start here” for anyone picking up Kongsberg EM710-style data in MB-System (**format 58** = raw `MBF_EM710RAW`, **59** = processed `MBF_EM710MBA`). The longer notes under `consulting/` spell out parser locations, caveats, and one known limitation.

## What we extracted and why it’s useful

**Seabed image (“sample amplitude”) values** come from the **SS2 / seabed-image** path in MB-System (not from every datagram in the `.all` stream). For a first pass to your own detectors, we exported **per-beam raw sample bundles** with `mblist` using the Simrad-specific raw tokens (beam index `#`, samples per beam `.s`, first *N* amplitudes `.Np` in dB-ish form as MB-System lists them). That matches “open the seabed image datagram and pull the ΣNₛ sample block,” but **already unpacked** into MB-System’s internal ping layout.

For **“where on the seafloor”** style work (maps, masks, GIS), the more appropriate quick export is **`mblist -D5`**, which emits **longitude, latitude, sidescan/backscatter pixel values** on a processed file, i.e. MB-System has applied its normal pairing of records (ping counters, timestamps) and navigation/attitude interpolation so pixels are **georeferenced**, not raw datagram-only positions.

## Minimal operator recipe (high level)

1. **Optional but common:** `mbpreprocess` on the raw `*.mb58` (long options in current MB-System) → often yields a companion **`*.mb59`** plus sidecars (`.inf`, `.fnv`, `.fbt`, backscatter tables, etc.).
2. **`mbprocess` needs a parameter file:** `inputfile.par` beside the swath. If it is missing, `mbprocess` skips the file. Use **`mbset`** to create/update `.par`; if `mbset -V -L` prints that nothing was written, force a write with an explicit parameter (e.g. **`OUTFILE`** to your intended `…p.mb59`**).
3. **Exploratory hotspot CSV:** run **`mblist -D5`** on the processed output, comma-delimit, then apply **global P95/P99** thresholds on the intensity column as a sanity mask (expect ~5% / ~1% row counts at true quantiles, modulo NaN filtering).

## One limitation to name explicitly

**Central beams echogram** in the Kongsberg manual (**type K / ID 0x4B**, `EM3_ID_CBECHO` in MB-System headers) is **not** decoded into the same structures that `mblist` uses for seabed-image samples. In the EM710 raw reader, payloads for that datagram type are **skipped**, not turned into listable fields—so **you cannot swap in a different `mblist` flag** and get K-echogram amplitudes the same way. That would require **vendor-aware parsing of the raw stream** or **new MB-System support**.

For questions that need code-level detail, see **`05-mb58-mb59-extraction-hotspot-qc.md`** and **`03-mb58-field-extraction-map.md`**.
