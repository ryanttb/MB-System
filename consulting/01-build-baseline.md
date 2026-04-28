# MB-System Build Baseline (Prometheus)

This baseline is designed for repeatable onboarding on modern Linux workstations.

## Build policy

- Primary path: **CMake**.
- Fallback path: **Autotools** only for legacy OS constraints.
- Target install prefix: `/usr/local`.
- Shell assumptions: `bash`, `sudo` access.

## Why CMake first

- `BuildAndInstall.md` marks CMake as the recommended path for current OS versions.
- CMake avoids common runtime linker problems that often require `LD_LIBRARY_PATH` in Autotools installs.
- CMake flow is simpler to teach and repeat across multiple engineers.

## Baseline prerequisites (Ubuntu 20.04/22.04, Debian 11/12)

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
  build-essential cmake gfortran \
  netcdf-bin libnetcdf-dev libgdal-dev \
  gmt libgmt6 libgmt-dev libproj-dev \
  libfftw3-3 libfftw3-dev libmotif-dev \
  xfonts-100dpi libglu1-mesa-dev \
  libopencv-dev
```

Python 3 and Pillow are expected by MB-System workflows; install if absent:

```bash
python3 --version
python3 -m pip install --user Pillow
```

## Build and install (CMake baseline)

From repository root:

```bash
mkdir -p build
cd build
cmake ..
make -j"$(nproc)"
sudo make install
```

## Post-install checks

Ensure executables are visible:

```bash
which mbinfo
mbinfo -h
mbformat -L | head
```

Enable GMT plugin modules (`mbcontour`, `mbswath`, `mbgrdtiff`) by setting:

```bash
gmt gmtset GMT_CUSTOM_LIBS /usr/local/lib/mbsystem.so
```

Confirm path includes `/usr/local/bin`:

```bash
echo "$PATH"
```

If missing, add this to `~/.profile` or `~/.bashrc`:

```bash
export PATH=/usr/local/bin:$PATH
```

## Standard troubleshooting

- If `cmake ..` fails: verify GMT/PROJ/GDAL/netCDF dev packages are installed.
- If `mbcontour` or `mbswath` fail under `gmt`: recheck `GMT_CUSTOM_LIBS`.
- If GUI programs fail (`mbedit`, `mbnavedit`): verify X11/Motif/OpenGL packages.

## Newer compiler compatibility note (Ubuntu 25+)

On newer Linux toolchains, some legacy GCTP sources may fail due to old K&R-style
function declarations being treated as zero-argument function pointers.

Typical error pattern during `make`:

- `too many arguments to function 'inv_trans[*insys]'`
- `too many arguments to function 'for_trans[*outsys]'`

If encountered, update GCTP function pointer declarations and related prototypes to
explicit typed signatures `(double, double, double *, double *)` in:

- `src/mbtrnav/gctp/source/gctp.c`
- `src/mbtrnav/gctp/source/proj.h`
- `src/mbtrnav/gctp/source/inv_init.c`
- `src/mbtrnav/gctp/source/for_init.c`

Also check SOM helper prototypes for the same issue in:

- `src/mbtrnav/gctp/source/somfor.c`

## Post-build smoke test (2-3 minutes)

Run these checks after `sudo make install`:

```bash
which mbinfo
which mbformat
mbinfo -h
mbformat -L | head -n 20
```

If you have a sample swath file available (`line.mbXX`):

```bash
mbinfo -I line.mbXX
mblist -I line.mbXX -OXYz -G"," | head -n 10
```

Optional GMT module check:

```bash
gmt mbcontour -?
```

Expected outcome:

- Commands are found in `/usr/local/bin` (or your configured install prefix)
- `mbformat -L` returns supported format list
- `mbinfo -I ...` returns valid metadata for a sample file
- `gmt mbcontour -?` shows module help (if `GMT_CUSTOM_LIBS` is configured)

## Legacy fallback (Autotools)

Use only when CMake is not viable on the target OS:

```bash
./configure --enable-mbtrn --enable-mbtnav --enable-opencv \
  --with-opencv-include=/usr/include/opencv4 \
  --with-opencv-lib=/lib/x86_64-linux-gnu
make -j"$(nproc)"
sudo make install
```
