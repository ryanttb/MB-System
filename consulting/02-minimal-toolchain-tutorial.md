# MB-System Minimal Toolchain Tutorial

This tutorial gives a repeatable, first-day workflow:

`inspect -> edit/configure -> process -> generate product`.

It is intentionally minimal and uses command-line tools first.

## 0) Inputs and assumptions

- MB-System is installed and commands are in `PATH`.
- You have one swath file, referenced below as `survey.mbXX`.
- You know the format ID, or can detect it with `mbformat`.

## 1) Inspect and identify data

Check recognized formats and identify your file:

```bash
mbformat -L
mbinfo -I survey.mbXX
```

List selected fields quickly (time, lon, lat, depth):

```bash
mblist -I survey.mbXX -OXYz -G"," | head -n 20
```

Create or validate a datalist:

```bash
echo "survey.mbXX <format_id> 1.0" > datalist.mb-1
mbdatalist -I datalist.mb-1
```

## 2) Create initial ancillary products

Generate quick-read support files (`.inf`, `.fbt`, `.fnv`):

```bash
mbdatalist -I datalist.mb-1 -O
```

## 3) Edit and configure processing controls

Automated first-pass bathymetry cleaning:

```bash
mbclean -I survey.mbXX
```

This creates/updates:

- `survey.mbXX.esf` (edit save file)
- `survey.mbXX.par` (processing parameters)

Optional interactive steps for analysts:

- `mbedit -I survey.mbXX` for manual beam editing
- `mbnavedit -I survey.mbXX` for navigation cleanup
- `mbvelocitytool -I survey.mbXX` for SVP updates

## 4) Process to a finalized swath file

Run processing controlled by `.par` and sidecars:

```bash
mbprocess -I survey.mbXX
```

Typical output is `surveyp.mbXX` plus ancillary files.

## 5) Generate a first data product

Create a quick grid from the processed file:

```bash
mbgrid -I surveyp.mbXX -O survey_grid -A2 -C2 -E1/1
```

Then inspect result metadata:

```bash
gmt grdinfo survey_grid.grd
```

## 6) Quick QA checklist

- `mbinfo` on raw and processed files gives sensible bounds/counts.
- `.par` exists and includes intended processing switches.
- `.esf`/`.nve`/`.svp` sidecars exist when corresponding edits were performed.
- `mbgrid` completes and generated grid has expected region/resolution.

## 7) Suggested team exercise (45-60 minutes)

- Engineer A: inspect + initial clean (`mbinfo`, `mblist`, `mbclean`)
- Engineer B: navigation/SVP edits (`mbnavedit`, `mbvelocitytool`)
- Both: run `mbprocess`, compare `mbinfo` before/after, and generate one `mbgrid` product.
