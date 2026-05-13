# Client handoff — TLDR

This is a short “start here” for anyone picking up Kongsberg EM710-style data in MB-System (**format 58** = raw `MBF_EM710RAW`, **59** = processed `MBF_EM710MBA`). The longer notes under `consulting/` spell out parser locations, caveats, and one known limitation.

## What we extracted and why it’s useful

**Seabed image (“sample amplitude”) values** come from the **SS2 / seabed-image** path in MB-System (not from every datagram in the `.all` stream). For a first pass to your own detectors, we exported **per-beam raw sample bundles** with `mblist` using the Simrad-specific raw tokens (beam index `#`, samples per beam `.s`, first *N* amplitudes `.Np` in dB-ish form as MB-System lists them). That matches “open the seabed image datagram and pull the ΣNₛ sample block,” but **already unpacked** into MB-System’s internal ping layout.

For **“where on the seafloor”** style work (maps, masks, GIS), the more appropriate quick export is **`mblist -D5`**, which emits **longitude, latitude, sidescan/backscatter pixel values** on a processed file, i.e. MB-System has applied its normal pairing of records (ping counters, timestamps) and navigation/attitude interpolation so pixels are **georeferenced**, not raw datagram-only positions.

## Outcome (TN304 and downstream processing)

The **TN304 multibeam deliverable in MB58** does **not** include **complex per-beam time series** (in-phase and quadrature, or equivalent) that some proprietary **downstream processing** expects as its starting point. Within the Kongsberg-style record set that MB58 carries, the **closest analogue** is often described as the **central beams echogram** (vendor type **K** / id **0x4B**), but that path is still **magnitude-oriented** in the vendor sense; feeding a phase-sensitive pipeline would require **phase** (or full analytic signal), not amplitude alone—and in practice those echogram payloads are **not decoded into MB-System listing tools** such as `mblist` today (see below). So the client’s specific detector chain is **unlikely to run as-is on this TN304 export**; applying it would mean **changes upstream** (different export from the sonar / vendor chain, or new parsing and packaging).

That negative answer is still a **successful outcome for this task**: the goal was to determine **whether the files contain usable signal for the client’s purpose**, and to document what *is* extractable via MB-System for sanity checks and exploratory work (seabed-image amplitudes, georeferenced sidescan pixels, presence/absence checks for optional datagram types).

## Minimal operator recipe

Below is the **exact command sequence** used on an `*.mb58` Kongsberg-style line file (names shortened to `X_TGT.all.mb58`). Run from a shell in the directory that contains the data (or use absolute paths). Adjust the stem `X_TGT.all` to match your file.

```bash
# 1) Preprocess raw MB58 → MB59 plus sidecars (.inf, .fnv, .fbt, .baa/.bah/.bsa, etc.)
mbpreprocess --input="X_TGT.all.mb58" --verbose

# 2) Point further steps at the mbpreprocess output (same stem, .mb59)
IN="X_TGT.all.mb59"

# 3) Create/update the mbprocess parameter file (*.par next to $IN)
#    If mbset reports that no .par was created, force one with an explicit -P OUTFILE:… (see 05-…-qc.md).
mbset -f 59 -F 59 -v 1 "$IN"

# 4) Process the swath (merges nav, applies mbprocess pipeline — see MB-System docs)
#    https://www3.mbari.org/products/mbsystem/html/mbprocess.html
mbprocess -I"$IN" -V

# 5) Raw seabed-image sample amplitudes (per beam: index #, count .s, first 30 samples .30p), dB as listed by mblist
mblist -I"$IN" -MA -ON#.s.30p -G"," > "${IN}.amplitudes.csv"

# 6) Lon/lat + sidescan column (dump mode 5) for GIS / hotspot-style masks
#    If mbprocess wrote a different output name (often a “p” infix), pass -I that file instead of $IN.
mblist -I"$IN" -D5 -G"," > "${IN}.sidescan.csv"
```

**Notes**

- **`mbprocess`** expects **`$IN.par`** to exist; **`mbset`** step creates or updates it. On some builds, `mbset` only writes a new `.par` when at least one parameter changes from defaults—if needed, add e.g. `-P"OUTFILE:/abs/path/to/X_TGT.allp.mb59"` once, then re-run `mbset` / `mbprocess` as in **`05-mb58-mb59-extraction-hotspot-qc.md`**.
- **`mblist -D5`** is documented as longitude, latitude, sidescan; use the **processed** swath path if that is not the same as `$IN` after `mbprocess`.
- **Exploratory masks:** on `*.sidescan.csv`, global **P95 / P99** on the third column is a reasonable first-pass QC (see **`05-mb58-mb59-extraction-hotspot-qc.md`**).
- **`mbinfo`** (full read, not `--quick`): if **central beams echogram** datagrams (K / 0x4B) appear in the raw stream, a line is printed when the file is closed — see codebase change under `mb_io_struct.simrad_cbecho_datagram_count` / Simrad raw readers.

## One limitation to name explicitly

**Central beams echogram** in the Kongsberg manual (**type K / ID 0x4B**, `EM3_ID_CBECHO` in MB-System headers) is **not** decoded into the same structures that `mblist` uses for seabed-image samples. In the EM710 raw reader, payloads for that datagram type are **skipped**, not turned into listable fields—so **you cannot swap in a different `mblist` flag** and get K-echogram amplitudes the same way. That would require **vendor-aware parsing of the raw stream** or **new MB-System support**.

For questions that need code-level detail, see **`05-mb58-mb59-extraction-hotspot-qc.md`** and **`03-mb58-field-extraction-map.md`**.
