# Plan: vendor v3 wM-Bus dependencies locally

## Status

Implemented and validated with ESPHome 2026.10.1; see the validation record
below.

## Goal

Remove the two Git-based library dependencies from the v3 `wmbus` component so
the ESPHome Arduino toolchain no longer needs to convert those repositories as
PlatformIO libraries. Preserve the existing component behavior and leave the
PlatformIO build working.

## What the repository branches show

- The v3 component currently declares `SPI`, SmartRC CC1101 Driver, and
  `wM-Bus-drivers` in `components/wmbus/__init__.py`. The latter two are fetched
  as Git libraries. The driver dependency is pinned to `1.3.13`; SmartRC is now
  pinned to `v3.0.2`.
- v3 includes `drivers.h` from `wM-Bus-drivers` in `components/wmbus/wmbus.h`.
  That header then includes the individual meter-driver headers.
- v3 includes SmartRC's `ELECHOUSE_CC1101_SRC_DRV.h` from
  `components/wmbus/rf_cc1101.h` and
  `components/wmbus/cc1101_rf_settings.h`. `rf_cc1101.cpp` calls the SmartRC
  global driver and its SPI/register methods.
- `main` organizes its meter implementations as source files within
  `components/wmbus_common` and implements its CC1101 radio directly within
  `components/wmbus_radio`. Its component setup selects source files locally.
- `main` is not a direct compatibility template for v3: it documents a
  migration to ESP-IDF and dropping Arduino, and its radio implementation is
  not the SmartRC library. The useful precedent is that implementation code is
  owned and compiled as part of the ESPHome component, not downloaded as a
  PIO library.

## Dependency provenance to preserve

This instruction file records the upstream versions copied into this branch
and their resolved commit SHAs. Keep these records and the upstream
license/notice files with the vendored code so a future update can identify
both its source and terms.

| Library | Upstream repository | Pinned source version on v3 | Resolved commit | Local folder |
|---|---|---|---|---|
| wM-Bus drivers | `https://github.com/SzczepanLeon/wmbus-drivers` | tag `1.3.13` | `c856f7d1eadb337ae3e6f59f3eb3ca652e371735` | `components/wmbus_drivers/` |
| SmartRC CC1101 Driver | `https://github.com/LSatan/SmartRC-CC1101-Driver-Lib` | tag `v3.0.2` | `33a9af9f2c5936d625db65ae2be128eede753397` | `components/smarttc_cc1101/` |

For any future re-vendoring, update the version and resolved commit SHA in
this record, copy from the named upstream tag rather than a moving branch,
preserve the associated license/notice files, and review source/include/build
changes before updating the component.

The SmartRC tag contains an MIT `LICENSE`, which must be copied with the
library. The wM-Bus drivers tag has no repository-level `LICENSE` or `NOTICE`
file; preserve all per-file copyright/license notices. Several meter headers
identify their upstream wmbusmeters source as GPL-3.0-or-later. Do not invent a
replacement license file; keep this provenance visible and verify applicable
license obligations before redistribution.

## Proposed layout

Use the three requested top-level folders under `components/`. Keep all
existing v3 component and sensor files in `wmbus`; use the other two folders
only for the vendored library contents and the minimum ESPHome integration
metadata needed to build them:

```text
components/
  wmbus/                          # Existing v3 component and sensor platforms
    sensor/
    text_sensor/
  wmbus_drivers/                  # Contents of wmbus-drivers v1.3.13
    drivers.h
    driver_*.h
    <any other required upstream files>
  smarttc_cc1101/                 # Contents of SmartRC v3.0.2
    <upstream source/header layout>
```

The `wmbus_drivers` folder should contain the pinned v3 dependency, not the
newer `main` branch's different wmbusmeters implementation. Keep the current
v3 sensor and text-sensor files where they are in `wmbus`; do not change the
existing `platform: wmbus` YAML.

Preserve each library's upstream file layout where practical, especially
relative includes. Include required license and attribution files alongside
the copied code. If ESPHome does not expose or compile one of the top-level
vendored folders as part of the `wmbus` build, add only the minimal component
metadata and dependency/autoload wiring needed to make it available. Do not
add additional top-level component folders. The build test must establish the
actual include paths and source discovery; do not assume nested files are
compiled merely because they are present on disk.

This layout is proposed rather than assumed to be the only working layout. If
ESPHome's source discovery for the supported toolchains does not compile a
nested SmartRC `.cpp` file, adjust the build integration or source placement
minimally and verify it with both toolchains.

## Implementation actions after approval

1. **Confirm exact upstream inputs**
   - Use SmartRC tag `v3.0.2`, matching the current v3 dependency pin.
   - Use `wmbus-drivers` tag `1.3.13`, matching the current v3 dependency pin.
   - Resolve each tag to its commit SHA and record it in the provenance table
     above before copying. Keep that record current if a future version is
     intentionally adopted.
   - Copy only files needed to build and use the component, plus required
     license/notice files. Do not silently modify upstream library behavior.

2. **Add the vendored files**
   - Add the wmbus-drivers v1.3.13 files to the top-level
     `components/wmbus_drivers/` folder.
   - Add the SmartRC v3.0.2 files to the top-level
     `components/smarttc_cc1101/` folder (folder name as requested).
   - Keep each library's source and headers in a layout compatible with its
     existing includes. Keep `drivers.h` and its included headers together.
   - Keep the existing `components/wmbus/sensor/` and
     `components/wmbus/text_sensor/` files in place.
   - Add minimal component metadata and dependency/autoload wiring only if
     required for ESPHome to include the vendored files in generated builds;
     do not create further top-level folders.
   - Do not set an `external_components.components` filter in the standard
     usage example: ESPHome discovers all external components by default, then
     `wmbus` loads its local dependencies through `AUTO_LOAD`. If users choose
     to add a filter, document that it must include the vendored components.

3. **Update component references**
   - Update the SmartRC includes in `rf_cc1101.h` and
     `cc1101_rf_settings.h` to reference the top-level vendored folder.
   - Update the `drivers.h` include in `wmbus.h` to reference the top-level
     `wmbus_drivers` folder.
   - Update any further include paths only if required by the copied upstream
     layout.
   - Remove the two Git dependency declarations from `to_code`.
   - Keep `cg.add_library("SPI", None)`: that is a framework library
     declaration, not either external Git dependency.

4. **Ensure vendored implementation sources are built**
   - Check the actual SmartRC `v3.0.2` file layout and identify its compiled
     source files, if any.
   - Verify ESPHome makes the files in `wmbus_drivers` and
     `smarttc_cc1101` available to the `wmbus` component with both the
     PlatformIO and Arduino toolchains. If it does not, wire required files
     into the build using the existing ESPHome mechanism rather than relying
     on header visibility alone.
   - Avoid adding unrelated dependency declarations or changing the CC1101
     receive implementation as part of this dependency packaging change.

5. **Update documentation**
   - Document that these two libraries are vendored and identify their
     upstream versions/revisions.
   - Explain the intended Arduino-toolchain support only after it has passed
     the build checks below.

## Validation and acceptance criteria

1. Run ESPHome configuration validation for an ESP8266 configuration using
   this external component and its existing `platform: wmbus` sensor entries.
2. Build that configuration using the Arduino toolchain. Confirm the original
   `Invalid PIO library ... missing library.json and library.properties`
   failure is gone and the build proceeds through compilation and linking.
3. Build the same or equivalent configuration with the existing PlatformIO
   toolchain to check for regressions.
4. Verify generated build inputs include SmartRC implementation source files
   (if the selected upstream version has them) and all wM-Bus driver headers.
5. Confirm the existing sensor and text-sensor platform names and options still
   validate and compile without changing their files or YAML interface, and
   confirm the standard README example works without a component filter.
6. Review the final diff to ensure only vendoring, required include/build
   wiring, and directly related documentation changed.

A successful compile verifies build compatibility, not radio operation. If
hardware is available, follow up with a CC1101 receive test and a decoded meter
telegram check.

## Validation record

- ESPHome configuration validation passed for an ESP8266 NodeMCU build with
  both `sensor: platform: wmbus` and `text_sensor: platform: wmbus`.
- The native Arduino toolchain compiled and linked successfully. The generated
  build included `SmartRC_CC1101.cpp` and the vendored headers; the original
  PIO library metadata error did not occur.
- The PlatformIO toolchain compiled and linked the same configuration
  successfully.
- A separate native Arduino configuration without any `components:` filter
  also validated and compiled successfully using `AUTO_LOAD`.
- Python syntax checks, local-header closure checks, dependency metadata/API
  checks, and `git diff --check` passed.
- No physical CC1101 hardware test was performed.

## Out of scope

- Porting v3 to the newer `main` architecture or its ESP-IDF/ESP32-only
  components.
- Replacing SmartRC with the CC1101 implementation from `main`.
- Updating either upstream library version while vendoring.
- Moving/refactoring the existing sensor platforms or changing the
  component's public YAML schema.
