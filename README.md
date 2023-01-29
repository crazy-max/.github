# `.github`

This repository acts as a fallback for all repositories under my `crazy-max`
handle that don't have an actual `.github` directory with issue templates and
other community health files.

It also contains a collection of GitHub Actions and reusable workflows
that can be used in your own repositories.

___

* [Actions](#actions)
* [Reusable workflows](#reusable-workflows)
  * [`releases-json`](#releases-json)

## Actions

_N/A_

## Reusable workflows

### `releases-json`

[`releases-json` action](.github/workflows/releases-json.yml) generates a JSON
file with the list of releases for a given repository.

```yaml
name: test

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
