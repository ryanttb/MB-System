# MB58 / MB59 extraction, geolocation, and exploratory “hotspot” QC

This note summarizes concepts useful when handing Kongsberg EM raw data (MB-System format **58** = `MBF_EM710RAW`, **59** = `MBF_EM710MBA`) to a client who will run their own signal processing. It is **not** a replacement for the vendor datagram manual; it ties **where fields are parsed in MB-System** to **what you can export for sanity checks**.

## 1) Seabed image datagram: the two “repeat cycle” blocks

In the Kongsberg “Seabed image / sidescan” style records, the manual’s patterns map to two concrete parser steps:

- **Repeat cycle — N entries (per-beam / per-column metadata):** one small fixed-size record per “beam group” in the sidescan image (sorting, number of samples for that group, center sample, etc., depending on generation).
- **Repeat cycle — ΣNₛ entries (sample amplitudes):** one contiguous block of samples whose **total length** is the **sum of per-beam sample counts** (with MB-System tracking each beam’s start index in that buffer).

**EM710 / format 59 path (Simrad3):** `mbr_em710raw_rd_ss2()` in `src/mbio/mbr_em710raw.c`  
- Loops `png_nbeams_ss` times to fill `png_beam_samples[]`, `png_start_sample[]`, etc.  
- Then reads `2 * png_npixels` bytes into `png_ssraw[]` (short samples) where `png_npixels` is the running sum of per-beam sample counts.

**Older EM300-style path (Simrad2):** `mbr_em300raw_rd_ss()` in `src/mbio/mbr_em300raw.c`  
- Same idea: per-beam loop, then `fread` of `png_npixels` **bytes** into `png_ssraw[]` (char samples) for that generation.

**Field memory / comments:** `mbsys_simrad3.h` / `mbsys_simrad2.h` (e.g. `png_ssraw` holds the packed sample stream; “0.5 dB” resolution is handled when values are **interpreted** as dB, not always as raw storage type).

**Tools note:** `mblist` can expose raw per-beam sample counts and the first *n* amplitudes per beam using the `"-O ... .s .30p ..."` style options (see `mblist` man page / `src/utilities/mblist.cc`), with amplitudes scaled to dB in the listing helpers.

## 2) “Location” of a seabed-image record: it is not self-contained

A seabed-image datagram is **not** a full geodetic fix by itself. In MB-System, **context** (which ping, which time, which ship attitude, and the bathymetry/beam geometry) is recovered by **pairing** with other records and **time-based interpolation**.

**Practical join keys (conceptual):**

- **Ping counter** (`png_count` vs `png_ss_count` in the Simrad3 store) to ensure the image belongs to the same ping as the bathy record.
- **Timestamps** (date + msec fields) to match or repair consistency.
- **Serial / head identifiers** matter mainly for **multi-head** or **split streams**, not for final lat/lon by themselves.

**EM710 reader behavior (illustrative):** in `mbr_em710raw`’s data path, survey records can **reconcile** bathy vs sidescan timestamps and **zero** sidescan if the ping counters do not line up. After that, **navigation and attitude** are applied when building the generic MB-System view of a ping (the details live in the `mbsys_simrad3` path and `mb_*int_*` interpolation helpers).

**For “where on the seafloor” (client goal):** you generally want **processed, georeferenced pixel or beam outputs** (not the raw `Y` datagram in isolation). Pixels in `mblist -D5` are **derived** using the same motion/geometry stack MB-System uses for sidescan mapping, subject to the quality of navigation and attitude.

## 3) `mbpreprocess` vs `mbset` / `mbprocess` (why `.par` showed up)

These utilities play different roles:

- **`mbpreprocess`:** modern, often **long-option** style (`--input`, `--format`, …). It can emit a **new swath file** (commonly `*.mb59` for this workflow) plus sidecars such as `.inf`, `.fnv`, `.fbt`, backscatter tables (`.baa`, `.bah`, `.bsa`), etc.  
  **It does not automatically create** an `mbprocess` parameter file (`*.par`).

- **`mbprocess`:** reads processing controls from **`inputfile.par`** next to the swath file (`"<path>/<basename>.par"`). If that file does not exist, current `mbprocess` behavior is to **skip** the file (it `stat()`s the `.par`).

- **`mbset`:** creates or updates **`.par`** files. Important behavior: **`mbset` only writes a new `.par` if the processing parameter structure actually differs** from what it read as defaults (or from an existing `.par`). If you run `mbset -V -L` and nothing changes, you may see **“no parameter file exists - not created.”**  
  **Force-create pattern:** set at least one explicit parameter, commonly the output file:

  ```bash
  mbset -I "/abs/path/to/file.mb59" \
    -P"OUTFILE:/abs/path/to/filep.mb59" \
    -V
  ```

  Then run `mbprocess -I "/abs/path/to/file.mb59" -V` to materialize the processed output (often `…p.mb59` depending on naming).

Use **absolute paths** when experimenting; relative-path confusion is a common reason files appear “missing.”

## 4) Exploratory georeferenced extract for client handoff (`mblist -D5`)

For a **quick lon/lat + sidescan/backscatter pixel stream** suitable for GIS or downstream masking:

```bash
mblist -I "/abs/path/to/processed.mb59" -F59 -D5 -G"," > ss_lonlat.csv
```

`-D5` is documented as **longitude latitude sidescan** (see `mblist` manual page). This is appropriate as an **exploratory** client input when their team owns detection science.

## 5) Simple global thresholds (P95 / P99) as QC masks

Sort the intensity column and pick quantiles:

```bash
awk -F, 'NF>=3 && $3!="NaN"{print $3+0}' ss_lonlat.csv | sort -n > ss_sorted.txt

N=$(wc -l < ss_sorted.txt)
L95=$(awk -v n="$N" 'BEGIN{printf "%.0f", 0.95*n}')
L99=$(awk -v n="$N" 'BEGIN{printf "%.0f", 0.99*n}')
sed -n "${L95}p" ss_sorted.txt   # empirical P95 value
sed -n "${L99}p" ss_sorted.txt   # empirical P99 value
```

Then threshold export files for the client, e.g.:

```bash
TH=-12.8   # example: use the computed P95 value
awk -F, -v TH="$TH" 'BEGIN{OFS=","}
NF>=3 && $1!="NaN" && $2!="NaN" && $3!="NaN" {
  ss=$3+0
  if (ss >= TH) print $1+0,$2+0,ss
}' ss_lonlat.csv > ss_hotspots_P95.csv
```

**Sanity expectation:** on `N` valid samples, a **true global P95** mask keeps **about 5%** of rows; **P99** keeps **about 1%**. Row counts will differ slightly if invalid samples were removed before sorting, or if rounding differs.

**Caveat:** bright pixels are **not automatically geology**. Specular returns, nadir effects, noise spikes, and processing artifacts can dominate extremes. For operational geology, clients often move from **global quantiles** to **local contrast** (per ping / rolling window / segmentation). Your role here is **clean extraction + documented first-pass masks**.

## 6) What to document alongside deliverables

When passing files to a client algorithm team, include:

- **Source file lineage:** original `*.mb58`, `mbpreprocess` outputs (`*.mb59` + sidecars), and whether `mbprocess` was run (exact `…p.mb59` name).
- **MB-System version** (`mbversion` / build identification).
- **Export definition:** `mblist -D5` → lon, lat, pixel intensity; units as interpreted by MB-System for that format.
- **Threshold methodology:** `N`, P95/P99 values, row counts, and any NaN filtering rules.

This aligns **parser reality** (repeat cycles, pairing rules) with **operator artifacts** (CSV extracts) without over-claiming geological interpretation.

## 7) Limitation: Central beams echogram (vendor type **K**, ID **0x4B**) vs seabed image (`mblist -O … .s … .p`)

Client documentation often refers to **“Central beams echograms”** with **datagram type K** and hex **4Bh**. In MB-System headers this is **`EM3_ID_CBECHO` / `EM3_CBECHO`** (same low byte **`0x4B`**):

```314:317:src/mbio/mbsys_simrad3.h
#define EM3_ID_TILT 0x4A
#define EM3_ID_CBECHO 0x4B
#define EM3_ID_RAWBEAM4 0x4E
```

**Do not expect the seabed-image `mblist` recipe to apply unchanged.**

- **Seabed image sample amplitudes** (the workflow using `mblist … -MA -ON#.s.Np` with the raw `.p` token) come from MB-System’s **SS2 / seabed-image** path (`mbr_em710raw_rd_ss2()`, type **`EM3_SS2` / `0x59`**, etc.), which unpacks into `png_beam_samples[]` / `png_ssraw[]` for listing.

- **Central beams echogram (`0x4B`)** is a **different datagram layout**. In `mbr_em710raw_rd_data()` there is **no dedicated reader branch** for `EM3_CBECHO` alongside bathy, rawbeam4, SS2, water column, etc. Datagram types that are not handled explicitly fall through to the **generic skip path** (payload bytes are consumed but **not decoded into `mbsys_simrad3` fields** that `mblist` can export):

```4868:4875:src/mbio/mbr_em710raw.c
		else {
#ifdef MBR_EM710RAW_DEBUG
			fprintf(stderr, "skip over %d bytes of unsupported datagram type %x\n", *record_size_save, type);
#endif
			for (int i = 0; i < *record_size_save - 4; i++) {
				read_len = 1;
				status = mb_fileio_get(verbose, mbio_ptr, (char *)&junk, &read_len, error);
			}
```

**Practical takeaway for deliverables:** tell clients that **per-sample amplitude CSVs from `mblist` raw sidescan tokens are seabed-image / SS2-class data**, not central-beams echogram (**K**). Extracting **K** amplitudes requires **vendor-aware parsing** of the raw stream (or future MB-System support), not a different `mblist` flag on the same export path.
