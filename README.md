# Brother DCP-T300 for Void Linux

Packaging recipes to build the Brother DCP-T300 printer driver as native
XBPS packages for Void Linux using `xbps-src` from
[void-linux/void-packages](https://github.com/void-linux/void-packages).

## Status

- [x] Build validation passed for both packages (x86_64 and i686) in an
      isolated Void Linux container using `xbps-src`.
- [x] Physical printing — **TESTED — SUCCESSFUL** on Void Linux x86_64 with
      CUPS 2.4.x, using a Brother DCP-T300 over USB (VID:PID `04f9:0393`)
      and driver version 3.0.2. A one-page document printed successfully.
- [x] Scanner (CLI/SANE) — **TESTED — SUCCESSFUL** in an isolated Void Linux
      container using `brother-brscan4` (SANE backend `brother4`, device
      `brother4:bus2;dev4`, USB VID:PID `04f9:0393`). A scan produced a valid
      PNG. The scanner backend is a separate package, **not** part of this
      repository.
- [ ] Scanner (Simple Scan GUI) — installed and **detects** the DCP-T300, but
      a physical scan from the GUI was **not** validated: the container cannot
      claim the USB interface exclusively while the host owns the device
      (host-side USB claim). This is a container limitation, not a driver
      defect — scanning works via the CLI/SANE path.
- [ ] i686 physical printing — **not tested yet** (the i686 build succeeds,
      but has not been exercised on physical hardware)

During development and testing, the driver packages were installed and a
printer queue was exercised **only inside an isolated Void Linux container**.
No changes were made to the Ubuntu host.

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
- The runtime configuration file
  `/opt/brother/Printers/dcpt300/inf/brdcpt300rc` is written at setup time
  by `brprintconf_dcpt300` (driven by `brcupsconfpt1`) and reflects the
  user's printer settings, so it is declared in `mutable_files` and
  preserved across package upgrades.

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

> **Note:** physical printing has been verified successfully on Void Linux
> x86_64 (CUPS 2.4.x). The USB URI shown above is the one reported by the
> CUPS USB backend; adjust it (`lpoptions -p dcpt300 -o device-uri=...`) if
> your connection differs.
