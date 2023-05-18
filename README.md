# `.github`

This repository acts as a fallback for all repositories under my `crazy-max`
handle that don't have an actual `.github` directory with issue templates and
other community health files.

It also contains a collection of GitHub Actions and reusable workflows
that can be used in your own repositories.

___

* [Actions](#actions)
  * [`container-logs-check`](#container-logs-check)
  * [`gotest-annotations`](#gotest-annotations)
  * [`install-k3s`](#install-k3s)
* [Reusable workflows](#reusable-workflows)
  * [`list-commits`](#list-commits)
  * [`releases-json`](#releases-json)

## Actions

### `container-logs-check`

[`container-logs-check` composite action](.github/actions/container-logs-check/action.yml)
checks for a string in container logs. This can be used as a _poor man's_ e2e
testing for containers.

```yaml
name: test

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
        uses: crazy-max/.github/.github/actions/container-logs-check@main
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

### `gotest-annotations`

[`gotest-annotations` composite action](.github/actions/gotest-annotations/action.yml)
generates GitHub annotations for generated Go JSON test reports.

```yaml
name: ci

on:
  push:

jobs:
  go:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v3
      -
        name: Test
        run: |
          mkdir -p ./testreports
          go test -coverprofile ./testreports/cover.out -json ./... > ./testreports/test-report.json
      -
        name: Generate annotations
        uses: crazy-max/.github/.github/actions/gotest-annotations@main
        with:
          directory: ./testreports
```

### `install-k3s`

[`install-k3s` composite action](.github/actions/install-k3s/action.yml)
installs [k3s](https://k3s.io/) on a runner.

```yaml
name: ci

on:
  push:

jobs:
  install-k3s:
    # does not work with ubuntu-22.04 atm:
    # E0310 17:16:57.210215    2047 memcache.go:238] couldn't get current server API group list: Get "https://127.0.0.1:6443/api?timeout=32s": dial tcp 127.0.0.1:6443: connect: connection refused
    runs-on: ubuntu-20.04
    steps:
      -
        name: Checkout
        uses: actions/checkout@v3
      -
        name: Install k3s
        uses: crazy-max/.github/.github/actions/install-k3s@main
        with:
          version: v1.21.2-k3s1
```

## Reusable workflows

### `list-commits`

[`list-commits` reusable workflow](.github/workflows/releases-json.yml)
generates a JSON matrix with the list of commits for a pull request.

```yaml
name: ci

on:
  push:
  pull_request:

jobs:
  list-commits:
    uses: crazy-max/.github/.github/workflows/list-commits.yml@main
    with:
      limit: 10
    secrets: inherit

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
        uses: actions/checkout@v3
        with:
          ref: ${{ matrix.commit }}
```

> **Note**
>
> `limit` input is optional and defaults to `0` (unlimited).

### `releases-json`

[`releases-json` reusable workflow](.github/workflows/releases-json.yml)
generates a JSON file with the list of releases for a given repository.

```yaml
name: ci

on:
  push:

jobs:
  releases-json:
    uses: crazy-max/.github/.github/workflows/releases-json.yml@main
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
        uses: actions/download-artifact@v3
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
