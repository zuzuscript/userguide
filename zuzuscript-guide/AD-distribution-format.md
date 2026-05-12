# Appendix D: Zuzu Distribution Packaging Format (ZDF-1)

A Zuzu distribution is a tar archive containing metadata,
Zuzu modules, and optional executable Zuzu scripts.

## Archive layout

A valid distribution includes:

- `zuzu-distribution.json` at the distribution root.
- `Build.zzs`, optional.
- `modules` directory containing modules, optional.
- `scripts` directory containing scripts, optional.
- `tests` directory containing tests, optional.
- `inc` directory containing helper modules which will not be installed, optional.
- `README.md`, optional.
- `CONTRIBUTING.md`, optional.
- `SECURITY.md`, optional.
- `LICENSE`, optional.

These must be inside one single top-level directory named
`${name}-${version}`.

Example:

- geo-utils-1.0.0/zuzu-distribution.json
- geo-utils-1.0.0/README.md
- geo-utils-1.0.0/modules/geo/utils.zzm
- geo-utils-1.0.0/scripts/lat-lon-converter.zzs
- geo-utils-1.0.0/tests/unit-tests.zzs
- geo-utils-1.0.0/tests/cli-tests.zzs

## Build script (`Build.zzs`)

Soon after unpacking the tar archive, Zuzuzoo will run `Build.zzs`
(if it exists) with `inc` (if it exists) in the library search path.
`Build.zzs` can perform tasks like altering `zuzu-distribution.json`
to add or remove dependencies based on the local environment.

The Build.zzs script may interact with the user, so STDOUT/STDERR
should not be hidden.

## Metadata schema (`zuzu-distribution.json`)

Required top-level fields:

- `name` (string)
- `version` (string)
- `author` (string)
- `license` (string)

Optional top-level field:

- `abstract` (string)
- `repo` (string)
- `status` (one of "stable", "trial")
- `dependencies` (object of module minimum versions)

The `abstract` field is a short plain text description of the
distribution's purpose. Repositories may display or store only the
first 140 characters.

The `repo` field is an `http` or `https` URL for the source repository
for the distribution, such as a GitHub, GitLab, or self-hosted Git
repository page.

For `dependencies`, each key must be a non-empty module name and
each value must be a non-empty minimum required version string, which
can be "0" if any version will do.

Example:

```json
"dependencies": {
	"std/math": "1.0.0",
	"acme/time": "0.4.2"
}
```

Dependencies are module names, not tarball names or distribution
package names.

Zuzuzoo should determine which version (if any) of the required
modules are currently installed. If dependencies are not met, it should
request:

	http://zuzulang.org/module/MODULENAME

So for example, for module foo/bar, it should request:

	http://zuzulang.org/module/foo/bar

This URL (perhaps via redirects) will provide a ZDF-1 tar archive
containing the required module. Each dependency should be installed
by Zuzuzoo, though cycle detection is necessary.

Ideally the JSON metadata should be parsed using PairLists,
preserving order, so that dependencies can be installed in the same
order they were specified.

It should also be possible to check the latest stable version of
any module using:

	http://zuzulang.org/module/foo/bar.json

This should return just the JSON metadata for the distribution
instead of the whole tar archive.

## Tests

Before installation, `zuzuzoo` runs all discovered tests and emits
TAP output for pass/fail reporting. Tests are run with the `modules`
and `inc` directories added to the Zuzu library search path.

Installation aborts if any test fails, unless `--force` is used.
Use `--no-test` to skip tests.

## Installer destinations

`bin/zuzuzoo` installs to defaults:

- modules: `$HOME/.zuzu/modules`
- scripts: `$HOME/.zuzu/bin`
- metadata: `$HOME/.zuzu/meta`

On Windows, the default per-user module directory is
`%LOCALAPPDATA%\Zuzu\modules`.

With `--global`:

- modules: `/var/lib/zuzu/modules`
- scripts: `/usr/local/bin`
- metadata: `/var/lib/zuzu/meta`

On Windows, the global module directory is `%ProgramData%\Zuzu\modules`.

Overrides:

- `--lib-dir=DIR`
- `--bin-dir=DIR`
- `--meta-dir=DIR`

Test control:

- `--force` (continue install even if tests fail)
- `--no-test` (skip running tests before install)

## Installed metadata

On install, metadata is copied to the metadata destination as:

- `${name}-${version}.json`

The JSON data needs to be modified to add `modules` and `scripts`
top-level keys listing which modules and scripts were installed
as part of the distribution, along with a SHA-256 digest of each
installed module or script.

Before installing a new version, `zuzuzoo` removes prior installed
versions of the same distribution by reading previous metadata files
from the same metadata directory.

## Uninstall

Use:

- `zuzuzoo --remove=NAME`
- `zuzuzoo --remove=NAME --version=VERSION`

Uninstall removes files recorded in installed metadata, then removes
matching metadata file(s).
