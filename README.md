# `.github`

This repository acts as a fallback for all repositories under my `crazy-max`
handle that don't have an actual `.github` directory with issue templates and
other community health files.

## Actions and reusable workflows

It also contains a collection of GitHub Actions and reusable workflows
that can be used in your own repositories.

### `releases-json`

[`releases-json` action](.github/workflows/releases-json.yml) generates a JSON
file with the list of releases for a given repository:

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

  releases-json-file:
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
