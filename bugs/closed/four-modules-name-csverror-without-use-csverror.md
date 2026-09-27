---
package: csv-nv
title: Four modules name CsvError without use csverror, so a single-file check of any of them fails
slug: four-modules-name-csverror-without-use-csverror
status: wontfix
discovered: 2026-09-27
fixed:
version: 0.2.1
toolchain: 0.12.1-16-g668d23ff0
platform: linux/x86_64
fixed_in:
class: loud
refs: []
upstream:
related: []
area:
discovered_during:
duplicate_of:
tracker:
---

# Four modules name CsvError without use csverror, so a single-file check of any of them fails

## Symptoms

`novo build --check` of a single module under `src/` fails for four of
the five modules: `dialect`, `record`, `reader` and `writer`.  Only
`csverror` passes.  `dialect` and `record` fail with E2003 on a
`CsvError` variant; `reader` and `writer` fail through `dialect`, which
they `use`, with E2033 on the type:

```text
src/dialect.nv:273:13: type error: undefined function: 'BadDialect' [E2003]
src/record.nv:129:23: type error: undefined function: 'RaggedRow' [E2003]
src/dialect.nv:271:29: type error: undefined type: `CsvError`. ... [E2033]
```

`novo pkg build` and every test file pass.  Seen on toolchain 0.12.1 and
on main at 7619c3201.

## Repro

```sh
cd csv-nv
novo build --check src/dialect.nv    # exit 1, E2003
novo build --check src/csverror.nv   # exit 0
novo pkg build                       # exit 0
```

Expected: every module checks on its own, as the package build accepts
them.

## Cause

The four modules name `CsvError` or its variants without a `use
csverror` line.  The package build checks every module under `src/`
together, and SPEC § 9.2 and § 9.4 allow a module to name a sibling's
type and variants that way, so the package is correct.  The single-file
check loaded only the modules the file names with `use`, so it refused
what the package build accepts.  The defect is the toolchain's, filed
there as `novo/build-check-of-a-package-module-refuses-a-siblings-type-the-package-build-accepts`.

The same shape is `calendar-nv/six-modules-name-calerror-without-use-calerror`.

## Workaround

Check the package with `novo pkg build`, which checks every module.

## Fix

The package was correct: SPEC § 9.2 and § 9.4 allow a module to name a
sibling's type and variants without a `use` line, and `novo pkg build`
accepts it.  The single-file check was the toolchain's defect, filed as
`novo/build-check-of-a-package-module-refuses-a-siblings-type-the-package-build-accepts`
and fixed for novo 0.13.0: a module under `src/` is now checked with the
package's whole module set.  Every module of this package checks on its
own with that toolchain.  Nothing in the package changes.

## History

- 2026-09-27 — filed while deciding where the calendar-nv filing of the
  same shape belongs.
- 2026-09-27 — the defect was the toolchain's single-file check, fixed
  in novo 0.13.0; no change to the package.
- 2026-09-27 — closed as wontfix
