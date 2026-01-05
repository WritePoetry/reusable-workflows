# Reusable GitHub Actions Workflows

This repository contains **multiple reusable GitHub Actions workflows** with distinct purposes. Each workflow can be called independently or chained together, depending on the project needs.

## Table of Contents

* [1. Build Workflow](#1-build-workflow)
* [2. Create Release Workflow](#2-create-release-workflow)
* [3. PHP Unit Tests Workflow](#3-php-unit-tests-workflow)
* [4. Remote Sync Workflow](#4-remote-sync-workflow)

---

## Features

* Centralized build and release process.
* Multi-version PHP testing.
* Safe folder synchronization with configurable mappings.
* Reusable across multiple projects.

---

## 1. Build Workflow
Compiles assets (NPM/Composer) and prepares artifacts for deployment or release. Specifically designed for WordPress projects using `@wordpress/scripts`.

### Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `artifact_name` | Name of the artifact for sync. If empty, the folder upload is skipped. | No | `""` |
| `create_zip` | If `true`, generates a ZIP package renamed with the version. | No | `false` |
| `paths` | Files or folders to exclude from the build artifact. | No | `.git, .github, node_modules` |

### Requirements in the Consuming Project
* **npm Scripts**: Must include `"plugin-zip": "wp-scripts plugin-zip"`.
* **Versioning**: The final zip file name is formed as: `<package-name>-<version>.zip` (from `package.json` and Git tag).

### Usage Example
```yaml
build:
  uses: WritePoetry/reusable-workflows/.github/workflows/build.yml@main
  with:
    create_zip: true
  permissions:
    contents: write
    packages: read
    actions: read
  secrets: inherit
```

---

## 2. Create Release Workflow
Downloads a ZIP artifact and creates an official GitHub Release with automated changelogs.

### Inputs
| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `artifact_zip_name` | The name of the artifact containing the ZIP file. | No | `release-zip` |

### Usage Example

```yaml
jobs:
  create_release:
    needs: build
    uses: WritePoetry/reusable-workflows/.github/workflows/create-release.yml@main
    permissions:
      contents: write
      packages: read
      actions: read
    secrets: inherit
```
---

## 3. PHP Unit Tests Workflow

Runs PHPUnit tests across multiple PHP versions, with optional WordPress integration testing and coding-standards checks.

### Inputs

| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `php_versions` | JSON array of PHP versions used for matrix testing | No | `['7.4', '8.0', '8.1', '8.2']` |
| `phpunit9_config` | PHPUnit 9-specific configuration file, used only when detected | No | `phpunit-9.xml` |

### Requirements in the Consuming Project

* A valid `composer.json` including PHPUnit (and optionally WPCS).
* The script `bin/install-wp-tests.sh` for WordPress integration tests.

### Behavior

* **Matrix Testing**: Each PHP version runs independently.
* **Database**: Spins up MySQL 5.7 and prepares `wordpress_test` automatically if needed.
* **Multisite**: Executes tests in both standard and Multisite (`WP_MULTISITE=1`) environments.

### Usage Example

```yaml
jobs:
  tests:
    uses: WritePoetry/reusable-workflows/.github/workflows/php-tests.yml@v1
    with:
      php_versions: "['8.1','8.2']"
    secrets: inherit

```

---

## 4. Remote Sync Workflow

Synchronizes selected local directories to a remote server using SSH and rsync. Supports multiple folder mappings through matrix execution.

### Inputs

| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `host` | Remote server host | Yes | — |
| `username` | SSH username for the remote server | Yes | — |
| `folder` | Base folder on the remote server where synchronization occurs | Yes | — |
| `paths` | JSON array of `{from, to, exclude}` mappings | No | Default WordPress structure |

### Secrets

| Name | Description | Required |
| --- | --- | --- |
| `ssh-key` | Private SSH key for rsync authentication | Yes |

### Usage Example

```yaml
jobs:
  deploy_files:
    uses: WritePoetry/reusable-workflows/.github/workflows/remote-sync.yml@main
    with:
      host: example.com
      username: deploy
      folder: /var/www/html/wp-content
      paths: |
        [
          {"from": "themes", "to": "/themes/"},
          {"from": "plugins", "to": "/plugins/", "exclude": ".git* node_modules"}
        ]
    secrets:
      ssh-key: ${{ secrets.SERVER_SSH_DEPLOY_KEY }}

```

---

## General Usage Note

You can chain these workflows to create a complete CI/CD pipeline:

```yaml
jobs:
  run_tests_first:
    uses: WritePoetry/reusable-workflows/.github/workflows/php-tests.yml@v1
    secrets: inherit

  build:
      needs: run_tests_first
      uses: WritePoetry/reusable-workflows/.github/workflows/build.yml@main
      with:
        create_zip: true
      permissions:
        contents: write
        packages: read
        actions: read
      secrets: inherit

  call_shared_release:
    needs: build
    uses: WritePoetry/reusable-workflows/.github/workflows/create-release.yml@main
    permissions:
      contents: write
      packages: read
      actions: read
    secrets: inherit

```

> **Note on Persistence**: Artifacts are the only way to transfer physical files between jobs. We use upload-artifact to save build results and download-artifact in subsequent jobs (like Release or Sync) to retrieve them. Output variables are used only to pass metadata (like filenames), not the files themselves.