# Brother DCP-T300 for Void Linux

Packaging recipes to build the Brother DCP-T300 printer driver as native
XBPS packages for Void Linux using `xbps-src` from
[void-linux/void-packages](https://github.com/void-linux/void-packages).

## Status

- [x] Build validation (x86_64) passed for both packages in an isolated Void
      Linux container using `xbps-src`; the resulting `.xbps` packages were
      inspected but **not installed**.
- [ ] Physical printing — **not tested yet**
- [ ] Scanner — **not tested yet**
- [ ] i686 build — **not built yet** (template supports `i686` and `x86_64`)

No printer queue was created and no CUPS configuration was touched while
developing these recipes.

## Packages

| Package | Contents |
|---|---|
| `brother-dcpt300-lpr` | LPR driver: `/opt/brother/Printers/dcpt300/...` filter chain and `/usr/bin/brprintconf_dcpt300` |
| `brother-dcpt300-cupswrapper` | CUPS filter `/usr/lib/cups/filter/brother_lpdwrapper_dcpt300` and PPD `/usr/share/ppd/Brother/brother_dcpt300_printer_en.ppd` |

### Architecture

- `i686`
- `x86_64` — requires `glibc-32bit` because the Brother binaries are 32-bit
  ELF executables (interpreter `/lib/ld-linux.so.2`).

### Repository

Both packages belong to the **`nonfree`** repository: they ship proprietary
Brother binaries.

## Licensing and provenance

The Brother binaries are proprietary software owned by Brother Industries,
Ltd. This repository **does not distribute** those binaries and does **not**
grant redistribution rights for them.

The binaries are fetched at build time from the official Brother download URL
using the `distfiles` + `checksum` declared in each template, the same way
`xbps-src` handles any upstream source.

The packaging templates and the static CUPS wrapper script
(`srcpkgs/brother-dcpt300-cupswrapper/files/brother_lpdwrapper_dcpt300`) are
the part of this repository that we publish; see `LICENSE` for the exact
licensing boundaries.

## Design notes

- The path `/opt/brother/Printers/dcpt300` is preserved verbatim because the
  Brother binaries have the path hard-coded at build time
  (e.g. `brdcpt300filter` reads `/opt/brother/Printers/%s/inf/ImagingArea`).
- The upstream installer `cupswrapperdcpt300` is **not** used. It relies on
  init.d scripts, `lpadmin`, and CUPS queue manipulation, which do not fit the
  Void Linux model. It is removed from the package.
- Instead, a static CUPS filter wrapper
  (`brother_lpdwrapper_dcpt300`) is installed directly into
  `/usr/lib/cups/filter/`, and the PPD is installed into
  `/usr/share/ppd/Brother/`. The wrapper converts a PostScript job into
  Brother raster data and never touches queue/configuration; the CUPS queue
  is created by the administrator or user with `lpadmin`.
- Dependency notes: `a2ps` and `ghostscript` are needed by the LPR filter
  chain (`filterdcpt300`, `psconvertij2`), `psutils` is used by the CUPS
  wrapper for N-up features, and `cups` provides the filter directory.

## Building

Requirements: a checkout of `void-packages`, a working `xbps-src` bootstrap,
and the `nonfree` repository (enabled by default).

```sh
git clone https://github.com/void-linux/void-packages.git
cd void-packages

# Copy the srcpkgs/ tree from this repository into void-packages/srcpkgs/
cp -r /path/to/brother-dcpt300-void/srcpkgs/brother-dcpt300-lpr srcpkgs/
cp -r /path/to/brother-dcpt300-void/srcpkgs/brother-dcpt300-cupswrapper srcpkgs/

./xbps-src pkg brother-dcpt300-lpr
./xbps-src pkg brother-dcpt300-cupswrapper
```

The resulting packages are written to `hostdir/binpkgs/nonfree/`.

## Using the packages (after build)

Install the packages, then create the printer queue yourself, for example:

```sh
sudo xbps-install --repository=./binpkgs/nonfree brother-dcpt300-lpr \
    brother-dcpt300-cupswrapper
lpadmin -p dcpt300 -E -v usb://Brother/DCP-T300 \
    -P /usr/share/ppd/Brother/brother_dcpt300_printer_en.ppd
```

> **Note:** printing has not been physically tested yet. Adjust the device URI
> (`lpoptions -p dcpt300 -o ...`) to match your connection.
