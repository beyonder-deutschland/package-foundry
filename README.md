[![PyPI release](https://img.shields.io/pypi/v/package-foundry.svg)](https://pypi.org/project/package-foundry/)
[![Downloads](https://static.pepy.tech/badge/package-foundry)](https://pepy.tech/project/package-foundry)
[![Linting](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json)](https://github.com/astral-sh/ruff)
[![Coding Style](https://img.shields.io/badge/code%20style-Ruff-000000.svg)](https://github.com/astral-sh/ruff)

# Package Foundry

This repository will help keep all Python packages following a certain basic structure tidy and up-to-date. It's being
maintained by [Beyonder GmbH](https://beyonder.de).

This package will render all required configuration and installation files for your target package.

Typical use-cases:

- A new Python or Django version was released
- A Python or Django version was deprecated
- You want to update the Sphinx documentation builder
- You want to update the linter versions
- You want to add the third-party dependencies

## Versioning

This project follows the CalVer versioning pattern: `YY.MM.[RELEASE]`

## How to update a package

These steps will tell you how to update a package which was created by using this updater.

- Navigate to the root directory of **your** package and activate its virtual environment
- Run `package-foundry render-templates`
- Review the changes in the rendered files
- Increment `__version__` in `<module_name>/__init__.py` (e.g. `__version__ = "1.2.3"`). The build reads the package
  version from there
- Add an entry for the new version to `CHANGES.md`
- Release the new version as described in the "Releasing a new version" section of the rendered `README.md`

All commands and scripts in this README and in the rendered files (`package-foundry`, `pytest`, `scripts/unix/*`,
`scripts/windows/*`) expect to be run from the root directory of your package.

## Migrating from ambient-package-update

`package-foundry` is the successor of `ambient-package-update`. To move a package over:

- Replace `ambient-package-update` with `package-foundry` in your development dependencies and install it
- Run `package-foundry migrate` in the main directory of your package. It renames `.ambient-package-update/` to
  `.package-foundry/` and switches the imports in `metadata.py` and in ejected templates to `package_foundry`
- Run `package-foundry render-templates`

Until a package is migrated, `render-templates` stops with a hint to run `migrate`. Running `migrate` again is safe,
it finishes an interrupted migration.

## How to create a new package

Just follow these steps if you want to create a new package and maintain it using this updater.

1. Create a new repository at GitHub and clone it.
2. Create a virtual environment in the repository root, activate it and install `package-foundry`:
   ```bash
   pip install package-foundry
   ```
3. Create a directory `.package-foundry` and a file `metadata.py` inside. The
   [Metadata reference](#metadata-reference) below explains all fields.

```python
from package_foundry.metadata.author import PackageAuthor
from package_foundry.metadata.constants import (
    DEV_DEPENDENCIES,
    DEPLOYMENT_STATUS_STABLE,
    LICENSE_MIT,
    SUPPORTED_DJANGO_VERSIONS,
    SUPPORTED_PYTHON_VERSIONS,
)
from package_foundry.metadata.maintainer import PackageMaintainer
from package_foundry.metadata.package import PackageMetadata
from package_foundry.metadata.readme import ReadmeContent
from package_foundry.metadata.ruff_ignored_inspection import (
    RuffIgnoredInspection,
    RuffFilePatternIgnoredInspection,
)

METADATA = PackageMetadata(
    package_name="my_package_name",
    github_package_group="beyonder-deutschland",
    licenser="Beyonder GmbH",
    license=LICENSE_MIT,
    development_status=DEPLOYMENT_STATUS_STABLE,
    claim="A short one-line description of what your package does.",
    authors=[
        PackageAuthor(
            name="Beyonder GmbH",
            email="hello@beyonder.eu",
        ),
    ],
    maintainer=PackageMaintainer(
        name="Beyonder GmbH",
        url="https://beyonder.de/",
        email="hello@beyonder.eu",
    ),
    readme_content=ReadmeContent(
        tagline="A fancy tagline for your new package",
        content="""A multiline string containing specific things you want to have in your package readme.
""",
    ),
    dependencies=[
        "my_dependency>=1.0",
    ],
    supported_python_versions=SUPPORTED_PYTHON_VERSIONS,
    supported_django_versions=SUPPORTED_DJANGO_VERSIONS,
    has_migrations=False,
    optional_dependencies={
        "dev": [
            *DEV_DEPENDENCIES,
        ],
        # you might add further extras here
    },
    # Example of a global ruff ignore
    ruff_ignore_list=[
        RuffIgnoredInspection(key="XYZ", comment="Reason why we need this exception"),
    ],
    # Example of a file-based ruff ignore
    ruff_file_based_ignore_list=[
        RuffFilePatternIgnoredInspection(
            pattern="**/tests/missing_init/*.py",
            rules=[
                RuffIgnoredInspection(key="INP001", comment="Missing by design for a test case"),
            ],
        ),
    ],
)
```

4. Render the templates from the repository root:
   ```bash
   package-foundry render-templates
   ```
   If the module directory (e.g. `pony_express/`) or its `__init__.py` does not exist yet, the command creates it
   with the `claim` as docstring and `__version__ = "1.0.0"`. Every run replaces that docstring with the current
   `claim`, so change the `claim` in `metadata.py` instead of the docstring.
5. Mark the rendered shell scripts as executable in git. Otherwise they cannot be run directly after a checkout on
   Linux or macOS. This is needed once per package:
   ```bash
   git add scripts/unix
   git update-index --chmod=+x scripts/unix/install_requirements.sh scripts/unix/publish_to_pypi.sh
   ```
6. Install your package with its development dependencies by running `scripts/unix/install_requirements.sh` or
   `scripts/windows/install_requirements.ps1`. Both call `uv sync`, which installs the package in editable mode.
   Without uv, run `pip install -e ".[dev]"`. To try your local changes in another project, install the checkout
   there in editable mode as well, e.g. `pip install -e ../pony-express`
   (see [pip's editable installs](https://pip.pypa.io/en/stable/topics/local-project-installs/#editable-installs)).
7. Put your tests into a `tests/` directory in the repository root, next to the module directory and not inside it.
   The rendered pytest, tox and coverage configuration runs `tests/` and measures the coverage of the module only,
   and the tests stay out of the published wheel. A Django app that only exists for the tests goes into `testapp/`
   next to it. If `tests_require_django` is `True` (the default), add a Django `settings.py` to the repository root as
   well, because the tests run with `--ds settings`.
8. Write the documentation. `render-templates` creates `docs/conf.py`, `docs/Makefile` and `docs/make.bat`, the
   content is up to you. Add at least these two files:
   - `docs/index.rst`, the start page:
     ```rst
     pony-express
     ============

     .. mdinclude:: ../README.md

     .. toctree::
        :maxdepth: 1

        features/changelog
     ```
   - `docs/features/changelog.rst`, which the rendered `pyproject.toml` links to as "Changelog":
     ```rst
     .. mdinclude:: ../../CHANGES.md
     ```

   Build the documentation locally with `sphinx-build docs/ docs/_build/html/`.
9. Publish the documentation on [Read the Docs](https://readthedocs.org/): import the GitHub repository there. The
   project slug has to be the package name with hyphens (e.g. `pony-express`), because the rendered `pyproject.toml`
   links to `https://<slug>.readthedocs.io`. The rendered `.readthedocs.yaml` configures the build. Check under
   *Admin → Integrations* that the GitHub webhook exists, so the documentation is rebuilt on every push.
10. Set up the release pipeline as described in "First-time setup" in the rendered `README.md`, add a first entry to
    `CHANGES.md` and release the first version.

### Metadata reference

`PackageMetadata` (in `package_foundry.metadata.package`) is the single configuration object that drives
all rendered files. The following fields are **required** (no default value):

| Field | Type | Description |
|-------|------|-------------|
| `package_name` | `str` | The distribution/module name, e.g. `django_pony_express`. Underscores are converted to hyphens for the GitHub/PyPI name. |
| `github_package_group` | `str` | The GitHub owner/organization the repo lives under, e.g. `beyonder-deutschland`. Used for repo, issue and security-advisory URLs. |
| `licenser` | `str` | The copyright holder written into the `LICENSE.md` file. |
| `authors` | `list[PackageAuthor]` | One or more `PackageAuthor(name, email)` entries; rendered into `pyproject.toml`. |
| `maintainer` | `PackageMaintainer` | A single `PackageMaintainer(name, url, email)`. The email is used as the Code-of-Conduct contact. |
| `development_status` | `str` | A trove classifier such as `DEPLOYMENT_STATUS_STABLE`. See `metadata.constants` for the presets. |
| `readme_content` | `ReadmeContent` | Controls the generated `README.md` (see below). |
| `claim` | `str` | One-line package description. Used as the `pyproject.toml` description (shown on PyPI) and as the `__init__.py` docstring. Not part of the README, which uses `readme_content.tagline` instead. |
| `has_migrations` | `bool` | Whether the package ships Django migrations. Enables the migration-integrity CI job when `True`. |
| `dependencies` | `list[str]` | Runtime dependencies (PEP 508 specifiers). |
| `supported_python_versions` | `list[str]` | Python versions for the CI test matrix, e.g. `SUPPORTED_PYTHON_VERSIONS`. Every version has to appear in `DJANGO_PYTHON_COMPATIBILITY` and be supported by at least one of the `supported_django_versions`. At least one version has to be in `READTHEDOCS_PYTHON_VERSIONS`; the newest of those is used for Read the Docs builds. |
| `supported_django_versions` | `list[str]` | Django versions for the CI test matrix, e.g. `SUPPORTED_DJANGO_VERSIONS`. Every version has to be a key in `DJANGO_PYTHON_COMPATIBILITY`, which drives the matrix exclusions. |

The following fields are **optional** (defaults shown):

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `min_coverage` | `float` | `100.0` | Coverage threshold the CI coverage job enforces. |
| `license` | `str` | `LICENSE_MIT` | `LICENSE_MIT` or `LICENSE_GPL`; selects the rendered license file. |
| `license_year` | `int` | current year | Copyright year in the license file. |
| `main_branch` | `str` | `"master"` | Default branch name used in CI triggers and docs links. |
| `tests_require_django` | `bool` | `True` | Whether the test setup needs a Django settings module. |
| `github_package_name` | `str` | derived | Overrides the GitHub repo name (defaults to `package_name` with hyphens). |
| `module_name` | `str` | derived | Overrides the importable module name (defaults to `package_name` with underscores). |
| `optional_dependencies` | `dict[str, list[str]]` | `{}` | Extras, e.g. `{"dev": [*DEV_DEPENDENCIES]}`. The `dev` extra is what CI, tox and Read the Docs install, so `render-templates` stops if it is missing. |
| `ruff_ignore_list` | `list[RuffIgnoredInspection]` | `[]` | Global ruff rule ignores. |
| `ruff_file_based_ignore_list` | `list[RuffFilePatternIgnoredInspection]` | `[]` | Per-file-pattern ruff ignores. |
| `script_executables` | `list[ScriptExecutable]` | `[]` | Console entry points, each `ScriptExecutable(name, import_path)`. |
| `gitignore_list` | `list[str]` | `[]` | Extra `.gitignore` entries appended to the defaults. |

`ReadmeContent(tagline, content, uses_internationalisation=True)` controls the generated README: `tagline` is the
short headline, `content` is the free-form body, and `uses_internationalisation` toggles the translation-workflow
section in `CONTRIBUTING.md`.

Useful presets live in `package_foundry.metadata.constants`: `DEV_DEPENDENCIES`, `SUPPORTED_PYTHON_VERSIONS`,
`SUPPORTED_DJANGO_VERSIONS`, `READTHEDOCS_PYTHON_VERSIONS`, the `LICENSE_*` values, and the `DEPLOYMENT_STATUS_*` classifiers.

### Customizing the templates

To customize the templates, you can use the `eject-template` command.
Simply run

```bash
package-foundry eject-template
```

from the root of your project and select the template you want to eject.
The chosen template will be copied to `.package-foundry/templates`, ready to be customized.

If you want to overwrite template manually, you can find the default templates in the `package_foundry/templates` directory.
You can overwrite them by creating a `.package-foundry/templates` directory in your project
and create a new file with the same name as the template you want to overwrite.

## Releasing a new version

Releases are fully automated. Push a version tag and the pipeline will build, sign with
[Sigstore](https://www.sigstore.dev/), publish to PyPI via
[Trusted Publishing](https://docs.pypi.org/trusted-publishers/), and create a GitHub Release —
no API tokens needed.

```bash
git tag v<version>          # e.g. git tag v26.3.1
git push origin v<version>
```

Tags **must** start with `v`. Tags without the prefix won't trigger the pipeline.

### First-time setup

Before the pipeline can run for the first time, an admin must:

1. **Create GitHub Environment `pypi`**
   - Go to *Settings → Environments → New environment*, name it exactly `pypi`
   - Under *Deployment branches and tags*, add a tag rule with pattern `v*`
   - Optionally add required reviewers for a manual approval gate

2. **Configure PyPI Trusted Publisher**
   - Go to *PyPI → Project settings → Publishing → Add a new publisher*
   - Fill in: Owner `beyonder-deutschland`, Repository `package-foundry`,
     Workflow `release.yml`, Environment `pypi`

## Changelog

Can be found at [GitHub](https://github.com/beyonder-deutschland/package-foundry/blob/main/CHANGES.md).
