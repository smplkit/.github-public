# smplkit / .github-public

Shared code for smplkit SDKs:

- [`smplkit/java-sdk`](https://github.com/smplkit/java-sdk)
- [`smplkit/typescript-sdk`](https://github.com/smplkit/typescript-sdk)
- [`smplkit/csharp-sdk`](https://github.com/smplkit/csharp-sdk)
- [`smplkit/python-sdk`](https://github.com/smplkit/python-sdk)
- [`smplkit/go-sdk`](https://github.com/smplkit/go-sdk)

## Actions

### `smplkit/.github-public/actions/ci-gate@main`

A single-step gate that fails the workflow if any of its upstream jobs failed
or were cancelled. Composite actions cannot read the calling job's `needs`
context directly, so the caller passes it in as a JSON blob.

```yaml
ci-gate:
  if: always()
  needs: [lint, test, build]
  runs-on: ubuntu-latest
  steps:
    - uses: smplkit/.github-public/actions/ci-gate@main
      with:
        needs: ${{ toJson(needs) }}
```

### `smplkit/.github-public/actions/sdk-release-prepare@main`

Common release-scaffolding for the SDKs. On a normal push to `main`, runs
[`semantic-release`][sr] to compute the next version from conventional
commits and create the matching tag + GitHub release. On a manual
`workflow_dispatch` with a `manual_version` input, skips semantic-release and
creates the tag + release directly at `GITHUB_SHA` after validating that the
input is a valid semver and that no tag with that version already exists.

Outputs `released` (`"true"`/`"false"`) and `version` (semver, no `v` prefix)
so the calling workflow can run a language-specific publish step
conditionally.

```yaml
release:
  needs: [ci-gate]
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  permissions:
    contents: write
    issues: write
    pull-requests: write
    id-token: write
  steps:
    - uses: actions/checkout@v6
      with:
        fetch-depth: 0
        persist-credentials: false

    - id: prep
      uses: smplkit/.github-public/actions/sdk-release-prepare@main
      with:
        manual_version: ${{ github.event.inputs.manual_version }}
        github_token: ${{ secrets.GITHUB_TOKEN }}

    # Language-specific publish, only when the prep step actually released.
    - name: Publish
      if: steps.prep.outputs.released == 'true'
      run: ./publish.sh "${{ steps.prep.outputs.version }}"
```

## Contributing

External pull requests are welcome but rare — this repo's audience is
effectively the smplkit org's own SDK pipelines. If you have an idea, open an
issue first.

[gh-private]: https://github.com/smplkit/.github
[sr]: https://github.com/semantic-release/semantic-release
