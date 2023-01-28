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
    secrets: inherit

  releases-json-file:
    runs-on: ubuntu-latest
    needs:
      - releases-json
    steps:
      -
        name: Create releases.json file
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            await fs.writeFileSync('releases.json', `${{ needs.releases-json.outputs.releases }}`);
      -
        name: Check file
        run: |
          jq . releases.json
```
