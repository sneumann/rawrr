# Agent guide for rawrr

## Project layout

rawrr is an R package that wraps Thermo Fisher Scientific's RawFileReader
through a C# command-line assembly.

- `R/rawrr.R`: readers, S3 objects, plotting, and the assembly invocation helper.
- `R/dotNetAssembly.R`: assembly installation, building, and runtime checks.
- `R/accessorFunctions.R`: accessors and generics.
- `inst/rawrrassembly/rawrr.cs`: command-line dispatch and RawFileReader calls.
- `inst/rawrrassembly/rawrr.csproj`: .NET target and vendor dependencies.
- `tests/testthat/`: R unit and integration tests.
- `inst/extdata/sample.raw`: bundled RAW file used by `sampleFilePath()`.
- `man/` and `NAMESPACE`: generated R documentation and exports.
- `vignettes/`: longer usage examples.

## Making changes

Keep changes focused and preserve existing public APIs unless the task explicitly
requires changing them. Follow the surrounding R and C# conventions.

For behavior changes, first extend the relevant existing tests and confirm that
they fail for the intended reason before implementing the functionality.
Prefer the bundled sample file for focused integration tests.

R readers invoke the assembly with `system2()`. Several commands write R code
that `.rawrrSystem2Source()` loads into an environment named `e`. Keep both sides
of this contract consistent, quote command-line paths, and surface extraction
errors rather than returning success-shaped empty data.

Chromatograms and spectra use the existing `rawrrChromatogram`,
`rawrrChromatogramSet`, `rawrrSpectrum`, and `rawrrSpectrumSet` S3 classes.
Preserve their fields and attributes when extending reader functionality.

Verify vendor API capabilities against the pinned dependencies and real data.
Do not assume that a trace name produced by another tool is a name stored in
the RAW file. Distinguish calculated traces from independently stored data.

Document exported R functions with roxygen comments beside their definitions.
Regenerate the corresponding `man/` files and `NAMESPACE` using
`roxygen2::roxygenise()` when changing documentation or exports; inspect the
result for unrelated generated changes.

## Build prerequisites

Use the R version required by `DESCRIPTION` and a .NET SDK compatible with the
target in `inst/rawrrassembly/rawrr.csproj`. CI configuration is in
`.github/workflows/r.yml` and `.github/workflows/cd_rawrrassembly.yml`.

The Thermo Fisher NuGet packages must be available as a local NuGet source;
they are not supplied by the normal nuget.org feed. Obtain the versions pinned
in the project file from the vendor's RawFileReader distribution and respect
its license in `inst/rawrrassembly/RawFileReaderLicense.txt`. Vendor download
locations can change: do not silently upgrade dependencies to make a build work.

From the repository root, with the vendor feed configured:

```sh
dotnet build inst/rawrrassembly/rawrr.csproj
dotnet publish inst/rawrrassembly/rawrr.csproj -c Release -r linux-x64
```

Use the runtime identifier matching the target platform. R locates the executable
through `.rawrrAssembly()`, not the project's build directory. Use
`buildRawrrExe()` and its documented setup to build into the package's cache
location, or explicitly arrange for tests to use the locally built executable.
Do not accidentally validate changed R code against an older downloaded assembly.

## Validation

Install the package and configure the assembly before running reader tests.
With the current package installed, a focused test can be run from the root:

```sh
Rscript -e 'library(rawrr); testthat::test_file("tests/testthat/test-chromatogram.R")'
```

Choose the test file covering the change. The full installed-package test entry
point is `tests/testthat.R`. Some tests require Bioconductor dependencies and
network access, notably ExperimentHub data downloads. Report missing prerequisites
separately from code failures.

For broader package validation, follow the `rcmdcheck::rcmdcheck()` invocation in
`.github/workflows/r.yml`. Documentation-only changes do not require reader tests.
Never commit generated assembly binaries, `bin/`, `obj/`, downloaded vendor
packages, or temporary RAW files and test output.
