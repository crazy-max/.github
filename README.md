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
  * [`build-distribute-mp`](#build-distribute-mp)
  * [`bake-distribute-mp`](#bake-distribute-mp)
  * [`list-commits`](#list-commits)
  * [`pr-assign-author`](#pr-assign-author)
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

### `docker-scout`

[`docker-scout` composite action](.github/actions/docker-scout/action.yml) scans
Docker images for vulnerabilities using [Docker Scout](https://docs.docker.com/scout/).

```yaml
name: ci

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
        uses: crazy-max/.github/.github/actions/docker-scout@main
        with:
          version: "1.11.0"
          format: sarif
          image: alpine:latest
      -
        name: Upload SARIF report
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: ${{ steps.scout.outputs.result-file }}
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
        uses: actions/checkout@v4
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
    runs-on: ubuntu-latest
    steps:
      -
        name: Install k3s
        uses: crazy-max/.github/.github/actions/install-k3s@main
        with:
          version: v1.32.2+k3s1
```

## Reusable workflows

### `build-distribute-mp`

[`build-distribute-mp` reusable workflow](.github/workflows/build-distribute-mp.yml)
distributes multi-platform builds across runners efficiently.

```yaml
name: ci

on:
  push:
  pull_request:

jobs:
  build:
    uses: crazy-max/.github/.github/workflows/build-distribute-mp.yml@main
    with:
      push: ${{ github.event_name != 'pull_request' }}
      cache: true
      meta-image: user/app
      build-platforms: linux/amd64,linux/arm64
      login-username: ${{ vars.DOCKERHUB_USERNAME }}
    secrets:
      login-password: ${{ secrets.DOCKERHUB_TOKEN }}
```

Here are the main inputs for this reusable workflow:

| Name              | Type     | Default | Description                                                                                                                                                                                                                                                |
|-------------------|----------|---------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `runner`          | String   | `auto`¹ | Runner instance (e.g., `ubuntu-latest`).                                                                                                                                                                                                                   |
| `push`            | Bool     | `false` | Push image to registry.                                                                                                                                                                                                                                    |
| `cache`           | Bool     | `false` | Enable GitHub Actions cache backend.                                                                                                                                                                                                                       |
| `cache-scope`     | String   |         | Which scope GitHub Actions cache object belongs to if `cache` enabled.                                                                                                                                                                                     |
| `cache-mode`      | String   | `min`   | Cache layers to export if `cache` enabled (one of `min` or `max`).                                                                                                                                                                                         |
| `summary`         | Bool     | `true`  | Enable [build summary](https://docs.docker.com/build/ci/github-actions/build-summary/) generation.                                                                                                                                                         |
| `meta-image`      | String   |         | Image to use as base name for tags. This input is similar to [`images` input in `docker/metadata-action`](https://github.com/docker/metadata-action?tab=readme-ov-file#images-input) used in this reusable workflow but accepts a single image name.       |
| `build-platforms` | List/CSV |         | List of target platforms for build. This input is similar to [`platforms` input in `docker/build-push-action`](https://github.com/docker/build-push-action?tab=readme-ov-file#inputs) used in this reusable workflow. At least two platforms are required. |
| `login-registry`  | String   |         | Server address of Docker registry. If not set then will default to Docker Hub. This input is similar to [`registry` input in `docker/login-action`](https://github.com/docker/login-action?tab=readme-ov-file#inputs) used in this reusable workflow.      |
| `login-username`² | String   |         | Username used to log against the Docker registry. This input is similar to [`username` input in `docker/login-action`](https://github.com/docker/login-action?tab=readme-ov-file#inputs) used in this reusable workflow.                                   |
| `login-password`  | String   |         | Specifies whether the given registry is ECR (auto, true or false). This input is similar to [`password` input in `docker/login-action`](https://github.com/docker/login-action?tab=readme-ov-file#inputs) used in this reusable workflow.                  |

> [!NOTE]
> ¹ `auto` will choose the best matching runner depending on the target
> platform being built (either `ubuntu-latest` or `ubuntu-24.04-arm`).
> 
> ² `login-username` can be used as either an input or secret.

You can find the list of available inputs directly in [the reusable workflow](.github/workflows/build-distribute-mp.yml).

### `bake-distribute-mp`

[`bake-distribute-mp` reusable workflow](.github/workflows/bake-distribute-mp.yml)
distributes multi-platform builds across runners efficiently.

```hcl
variable "DEFAULT_TAG" {
  default = "app:local"
}

// Special target: https://github.com/docker/metadata-action#bake-definition
target "docker-metadata-action" {
  tags = ["${DEFAULT_TAG}"]
}

// Default target if none specified
group "default" {
  targets = ["image-local"]
}

target "image" {
  inherits = ["docker-metadata-action"]
}

target "image-local" {
  inherits = ["image"]
  output = ["type=docker"]
}

target "image-all" {
  inherits = ["image"]
  platforms = [
    "linux/amd64",
    "linux/arm/v6",
    "linux/arm/v7",
    "linux/arm64"
  ]
}
```

```yaml
name: ci

on:
  push:
  pull_request:

jobs:
  build:
    uses: crazy-max/.github/.github/workflows/bake-distribute-mp.yml@main
    with:
      target: image-all
      push: ${{ github.event_name != 'pull_request' }}
      cache: true
      meta-image: user/app
      login-username: ${{ vars.DOCKERHUB_USERNAME }}
    secrets:
      login-password: ${{ secrets.DOCKERHUB_TOKEN }}
```

Here are the main inputs for this reusable workflow:

| Name              | Type   | Default | Description                                                                                                                                                                                                                                           |
|-------------------|--------|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `runner`          | String | `auto`¹ | Runner instance (e.g., `ubuntu-latest`).                                                                                                                                                                                                              |
| `target`          | String |         | Multi-platform target to build. This input is similar to [`targets` input in `docker/bake-action`](https://github.com/docker/build-push-action?tab=readme-ov-file#inputs) used in this reusable workflow but accepts a single target.                 |
| `push`            | Bool   | `false` | Push image to registry.                                                                                                                                                                                                                               |
| `cache`           | Bool   | `false` | Enable GitHub Actions cache backend.                                                                                                                                                                                                                  |
| `cache-scope`     | String |         | Which scope GitHub Actions cache object belongs to if `cache` enabled.                                                                                                                                                                                |
| `cache-mode`      | String | `min`   | Cache layers to export if `cache` enabled (one of `min` or `max`).                                                                                                                                                                                    |
| `summary`         | Bool   | `true`  | Enable [build summary](https://docs.docker.com/build/ci/github-actions/build-summary/) generation.                                                                                                                                                    |
| `meta-image`      | String |         | Image to use as base name for tags. This input is similar to [`images` input in `docker/metadata-action`](https://github.com/docker/metadata-action?tab=readme-ov-file#images-input) used in this reusable workflow but accepts a single image name.  |
| `login-registry`  | String |         | Server address of Docker registry. If not set then will default to Docker Hub. This input is similar to [`registry` input in `docker/login-action`](https://github.com/docker/login-action?tab=readme-ov-file#inputs) used in this reusable workflow. |
| `login-username`² | String |         | Username used to log against the Docker registry. This input is similar to [`username` input in `docker/login-action`](https://github.com/docker/login-action?tab=readme-ov-file#inputs) used in this reusable workflow.                              |
| `login-password`  | String |         | Specifies whether the given registry is ECR (auto, true or false). This input is similar to [`password` input in `docker/login-action`](https://github.com/docker/login-action?tab=readme-ov-file#inputs) used in this reusable workflow.             |

> [!NOTE]
> ¹ `auto` will choose the best matching runner depending on the target
> platform being built (either `ubuntu-latest` or `ubuntu-24.04-arm`).
> 
> ² `login-username` can be used as either an input or secret.

You can find the list of available inputs directly in [the reusable workflow](.github/workflows/bake-distribute-mp.yml).

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

on:
  pull_request_target:
    types:
      - opened
      - reopened

permissions:
  contents: read
  pull-requests: write

jobs:
  run:
    uses: crazy-max/.github/.github/workflows/pr-assign-author.yml@main
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
        uses: actions/download-artifact@v4
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
