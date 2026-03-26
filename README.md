# `.github`

This repository acts as a fallback for all repositories under my `crazy-max`
handle that don't have an actual `.github` directory with issue templates and
other community health files.

It also contains a collection of GitHub Actions and reusable workflows
that can be used in your own repositories.

___

* [Actions](#actions)
  * [`container-logs-check`](#container-logs-check)
  * [`docker-scout`](#docker-scout)
  * [`gotest-annotations`](#gotest-annotations)
  * [`install-k3s`](#install-k3s)
* [Reusable workflows](#reusable-workflows)
  * [`list-commits`](#list-commits)
  * [`pr-assign-author`](#pr-assign-author)
  * [`releases-json`](#releases-json)
  * [`zizmor`](#zizmor)

## Actions

### `container-logs-check`

[`container-logs-check` composite action](.github/actions/container-logs-check/action.yml)
checks for a string in container logs. This can be used as a _poor man's_ e2e
testing for containers.

```yaml
name: test

permissions:
  contents: read

on:
  push:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      -
        name: Run container
        run: |
          docker run -d --name test crazymax/samba:4.18.2
      -
        name: Check container logs
        uses: crazy-max/.github/.github/actions/container-logs-check@v1
        with:
          container_name: test
          log_check: " started."
          timeout: 20
```

Action logs:

```
Setting timezone to UTC
Initializing files and folders
Setting global configuration
Creating user foo/foo (1000:1000)
Added user foo.
Creating user yyy/xxx (1100:1200)
Added user yyy.
Add global option: force user = foo
Add global option: force group = foo
Creating share public
Creating share share
Creating share foo
# Global parameters
[global]
  disable netbios = Yes
  ...
  strict locking = No
  vfs objects = fruit streams_xattr
  wide links = Yes
smbd version 4.18.2 started.
🎉 Found " started." in container logs
```

### `docker-scout`

[`docker-scout` composite action](.github/actions/docker-scout/action.yml) scans
Docker images for vulnerabilities using [Docker Scout](https://github.com/docker/scout-cli).

```yaml
name: ci

permissions:
  contents: read

on:
  push:

jobs:
  scout:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      -
        name: Scout
        id: scout
        uses: crazy-max/.github/.github/actions/docker-scout@v1
        with:
          format: sarif
          image: alpine:latest
      -
        name: Upload SARIF report
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: ${{ steps.scout.outputs.result-file }}
```

You can find the list of available inputs directly in [the action configuration](.github/actions/docker-scout/action.yml).

### `gotest-annotations`

[`gotest-annotations` composite action](.github/actions/gotest-annotations/action.yml)
generates GitHub annotations for generated Go JSON test reports.

```yaml
name: ci

permissions:
  contents: read

on:
  push:

jobs:
  go:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v4
      -
        name: Test
        run: |
          mkdir -p ./testreports
          go test -coverprofile ./testreports/cover.out -json ./... > ./testreports/test-report.json
      -
        name: Generate annotations
        uses: crazy-max/.github/.github/actions/gotest-annotations@v1
        with:
          directory: ./testreports
```

### `install-k3s`

[`install-k3s` composite action](.github/actions/install-k3s/action.yml)
installs [k3s](https://k3s.io/) on a runner.

```yaml
name: ci

permissions:
  contents: read

on:
  push:

jobs:
  install-k3s:
    runs-on: ubuntu-latest
    steps:
      -
        name: Install k3s
        uses: crazy-max/.github/.github/actions/install-k3s@v1
        with:
          version: v1.32.2+k3s1
```

## Reusable workflows

### `list-commits`

[`list-commits` reusable workflow](.github/workflows/releases-json.yml)
generates a JSON matrix with the list of commits for a pull request.

```yaml
name: ci

permissions:
  contents: read

on:
  push:
  pull_request:

jobs:
  list-commits:
    uses: crazy-max/.github/.github/workflows/list-commits.yml@v1
    with:
      limit: 10

  validate:
    runs-on: ubuntu-latest
    needs:
      - list-commits
    strategy:
      fail-fast: false
      matrix:
        commit: ${{ fromJson(needs.list-commits.outputs.matrix) }}
    steps:
      -
        name: Checkout
        uses: actions/checkout@v4
        with:
          ref: ${{ matrix.commit }}
```

> [!NOTE]
> `limit` input is optional and defaults to `0` (unlimited).

### `pr-assign-author`

[`pr-assign-author` reusable workflow](.github/workflows/pr-assign-author.yml)
assigns the author of a pull request as an assignee.

```yaml
name: assign-author

permissions:
  contents: read

on:
  pull_request_target:
    types:
      - opened
      - reopened

jobs:
  run:
    uses: crazy-max/.github/.github/workflows/pr-assign-author.yml@v1
    permissions:
      contents: read
      pull-requests: write
```

### `releases-json`

[`releases-json` reusable workflow](.github/workflows/releases-json.yml)
generates a JSON file with the list of releases for a given repository. Releases
tags should be [semver](https://semver.org/) compliant and should not contain
`latest` or `edge` tags that are handled internally by this action.

```yaml
name: ci

on:
  push:

jobs:
  releases-json:
    uses: crazy-max/.github/.github/workflows/releases-json.yml@v1
    with:
      repository: docker/buildx
      artifact_name: buildx-releases-json
      filename: buildx-releases.json
    secrets: inherit

  releases-json-check:
    runs-on: ubuntu-latest
    needs:
      - releases-json
    steps:
      -
        name: Download
        uses: actions/download-artifact@v6
        with:
          name: buildx-releases-json
          path: .
      -
        name: Check file
        run: |
          jq . buildx-releases.json
```

Typical usage is to generate and put this file on your repository if you don't
want to use the GitHub API to get the list of releases and therefore avoid
hitting the rate limit (see [docker/buildx#1563](https://github.com/docker/buildx/pull/1563)
for more info).

For example on [crazy-max/ghaction-hugo](https://github.com/crazy-max/ghaction-hugo)
repo, I use this workflow to generate a JSON file with the list of Hugo releases
and then use it in the action to check for latest and tagged releases:

```js
async function getRelease(version) {
  const url = `https://raw.githubusercontent.com/crazy-max/ghaction-hugo/master/.github/hugo-releases.json`;
  const response = await fetch(url);
  const releases = await response.json();
  console.log(JSON.stringify(releases[version], null, 2));
}

getRelease('latest');
```

```
{
  "id": 89231061,
  "tag_name": "v0.110.0",
  "html_url": "https://github.com/gohugoio/hugo/releases/tag/v0.110.0",
  "assets": [
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_checksums.txt",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_darwin-universal.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_dragonfly-amd64.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_freebsd-amd64.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_Linux-64bit.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_linux-amd64.deb",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_linux-amd64.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_linux-arm.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_linux-arm64.deb",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_linux-arm64.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_netbsd-amd64.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_openbsd-amd64.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_windows-amd64.zip",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_0.110.0_windows-arm64.zip",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_extended_0.110.0_darwin-universal.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_extended_0.110.0_Linux-64bit.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_extended_0.110.0_linux-amd64.deb",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_extended_0.110.0_linux-amd64.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_extended_0.110.0_linux-arm64.deb",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_extended_0.110.0_linux-arm64.tar.gz",
    "https://github.com/gohugoio/hugo/releases/download/v0.110.0/hugo_extended_0.110.0_windows-amd64.zip"
  ]
}
```

[This workflow runs](https://github.com/crazy-max/ghaction-hugo/blob/101b97bf572a55e490c065774f1aed175182dc6e/.github/workflows/hugo-releases-json.yml)
on push and schedule event, generates the JSON file and opens a pull request
if it contains new releases, so it's kept in sync with [https://github.com/gohugoio/hugo](https://github.com/gohugoio/hugo).

### `zizmor`

[`zizmor` reusable workflow](.github/workflows/zizmor.yml) scans GitHub Actions
workflows under `.github` by default with [Zizmor](https://github.com/zizmorcore/zizmor)
and uploads the SARIF report to GitHub code scanning.

```yaml
name: ci

permissions:
  contents: read

on:
  push:
  pull_request:

jobs:
  zizmor:
    uses: crazy-max/.github/.github/workflows/zizmor.yml@v1
    permissions:
      contents: read
      security-events: write
    with:
      min-severity: medium
      min-confidence: medium
      persona: pedantic
      no-online-audits: true
```

Here are the main inputs for this reusable workflow:

| Name                | Type   | Default   | Description                                                            |
|---------------------|--------|-----------|------------------------------------------------------------------------|
| `path`              | String | `.github` | Path passed to `zizmor` as the scan target.                            |
| `version`           | String |           | Install a specific zizmor version.                                     |
| `collect`           | List   |           | Extra artifact collection modes passed as repeated `--collect=` flags. |
| `min-severity`      | String |           | Minimum severity to report.                                            |
| `min-confidence`    | String |           | Minimum confidence to report.                                          |
| `persona`           | String |           | Zizmor persona to use for findings and output tuning.                  |
| `offline`           | Bool   | `false`   | Disable network access for audits.                                     |
| `no-online-audits`  | Bool   | `false`   | Skip online audits while keeping the rest of the scan enabled.         |
| `strict-collection` | Bool   | `false`   | Fail when artifact collection cannot be completed.                     |

> [!NOTE]
> This workflow scans `.github` by default, not the whole codebase. Override
> `path` if you need to target a different workflow or action directory.

You can find the list of available inputs directly in [the reusable workflow](.github/workflows/zizmor.yml).
