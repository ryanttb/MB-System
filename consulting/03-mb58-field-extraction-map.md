# MB58 Field Extraction Map (Kongsberg EM710 RAW)

This guide maps MB58 (`MBF_EM710RAW`, format id 58) from parser internals to user-facing extraction paths.

## 1) Routing and format identity

- Format constants:
  - `MBF_EM710RAW` (58)
  - `MBF_EM710MBA` (59)
  - Source: `src/mbio/mb_format.h`
- Runtime registration and dispatch:
  - `mbr_register_em710raw`
  - `mbr_register_em710mba`
  - Source: `src/mbio/mb_format.c`

## 2) Primary parser modules to inspect

- Vendor/raw parser: `src/mbio/mbr_em710raw.c`
- MB-System processing variant: `src/mbio/mbr_em710mba.c`
- System translation layer: `src/mbio/mbsys_simrad3.c` + `src/mbio/mbsys_simrad3.h`

## 3) MB58 datagram handlers (where fields enter MB-System)

Representative read handlers in `mbr_em710raw.c`:

- Runtime and metadata: `mbr_em710raw_rd_start`, `mbr_em710raw_rd_run_parameter`
- Timing/position/attitude: `mbr_em710raw_rd_clock`, `mbr_em710raw_rd_pos`, `mbr_em710raw_rd_attitude`, `mbr_em710raw_rd_netattitude`
- Orientation/support: `mbr_em710raw_rd_heading`, `mbr_em710raw_rd_tilt`, `mbr_em710raw_rd_ssv`, `mbr_em710raw_rd_tide`, `mbr_em710raw_rd_height`
- Water column model: `mbr_em710raw_rd_svp`, `mbr_em710raw_rd_svp2`
- Seafloor sounding/quality:
  - `mbr_em710raw_rd_bath2`
  - `mbr_em710raw_rd_rawbeam4`
  - `mbr_em710raw_rd_quality`
  - plus sidescan-related handlers in the same module

## 4) Translation to normalized outputs

After parsing into `mbsys_simrad3` storage, extraction functions expose normalized fields:

- Core data records: `mbsys_simrad3_extract`
- Navigation-only export: `mbsys_simrad3_extract_nav`
- N-sample nav export: `mbsys_simrad3_extract_nnav`
- Travel-time/angle export: `mbsys_simrad3_ttimes`
- Altitude/depth support: `mbsys_simrad3_extract_altitude`
- Sound velocity profile export: `mbsys_simrad3_extract_svp`

These are the bridge from Kongsberg-specific datagrams to generic MB-System tools.

## 5) User-facing commands for field verification/extraction

Use these commands while mapping required fields:

```bash
# Confirm format and basic metadata
mbinfo -I line.all

# List selected per-sounding values (edit -O selectors as needed)
mblist -I line.all -OXYz -G","

# Navigation-focused extraction
mbnavlist -I line.all

# SVP and CTD related listing
mbsvplist -I line.all
mbctdlist -I line.all

# Beam/trace style diagnostics
mbdumpesf -I line.all.esf
```

## 6) Practical extraction checklist for Prometheus MB58 tasks

For each required field, record:

1. Field name and units required by downstream analysis.
2. Source datagram/handler in `mbr_em710raw.c`.
3. Normalized extractor function in `mbsys_simrad3.c`.
4. Verification command and expected output shape.
5. Acceptance rule (range checks, null handling, timestamp continuity).

Use this table format:

| Required field | Datagram handler | MB-System extractor | CLI verification | Pass criteria |
|---|---|---|---|---|
| Example: vessel heading | `mbr_em710raw_rd_heading` | `mbsys_simrad3_extract_nav` | `mbnavlist -I line.all` | heading non-null and time-ordered |

## 7) Notes on MB59 and KMALL

- MB59 (`MBF_EM710MBA`) uses a parallel path with `mbr_em710mba.c`; when working from processed outputs, check this module as well.
- If files are `.kmall`, follow the MB261 path (`mbr_kemkmall.c` + `mbsys_kmbes.c`) instead of MB58.
