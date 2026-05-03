# smplkit / .github-public

Public sibling of the private [`smplkit/.github`][gh-private] repo. Hosts the
GitHub Actions composite actions that need to be reachable from
**public-visibility** repositories — primarily the five smplkit SDKs:

- [`smplkit/java-sdk`](https://github.com/smplkit/java-sdk)
- [`smplkit/typescript-sdk`](https://github.com/smplkit/typescript-sdk)
- [`smplkit/csharp-sdk`](https://github.com/smplkit/csharp-sdk)
- [`smplkit/python-sdk`](https://github.com/smplkit/python-sdk)
- [`smplkit/go-sdk`](https://github.com/smplkit/go-sdk)

GitHub Actions does not allow public repositories to consume composite actions
from private repositories, regardless of the org-level access policy. So any
shared action that runs in a public repo's workflow lives here; anything else
stays in the private `.github` repo with the rest of the org's internal
infrastructure tooling.

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

## Security and pinning

Workflows in the SDK repos currently reference these actions at `@main`. That
means every push to this repo's `main` branch is consumed immediately by the
next CI run in each downstream SDK — including the release jobs that hold
publishing credentials for Maven Central, npm, NuGet, and PyPI. Treat changes
here with the same care you would changes to a CI/CD pipeline that ships
production code.

For stricter supply-chain hygiene, downstream consumers can pin to a
full-length commit SHA instead of `@main`:

```yaml
- uses: smplkit/.github-public/actions/ci-gate@<40-char-sha>
```

## Branch protection

`main` is protected against force-pushes, deletion, and direct pushes from
non-administrators. Changes go through pull requests with at least one
approving review and a linear history; conventional-commit message format is
enforced at the org level.

## Contributing

External pull requests are welcome but rare — this repo's audience is
effectively the smplkit org's own SDK pipelines. If you have an idea, open an
issue first.

[gh-private]: https://github.com/smplkit/.github
[sr]: https://github.com/semantic-release/semantic-release
