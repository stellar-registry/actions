# stellar-registry/actions

Reusable GitHub workflows for releasing Soroban contracts: version detection,
release PRs, attested builds, and on-chain publishes to a
[Stellar Registry](https://github.com/stellar-registry/contracts).

Together they form a full pipeline that needs **no GitHub App, no crates.io,
and no manual steps** after merging a PR:

```
merge a feature PR
      │
      ▼
release-pr ──────────► opens/updates one "chore: release" PR
      │                (git-cliff semver bumps + changelogs)
      ▼  merge it
detect-releases ─────► tags <contract>-v<version> (pure git, no cargo package)
      │
      ▼  same run, via `needs:`
contract-release ────► scaffold build + optimize + GitHub release + attestation
      │
      ▼
registry-publish / registry-publish-assets
                 ────► verify provenance, publish on-chain, attach receipt
```

The build/publish halves hang off `detect-releases` outputs with `needs:` in
the **same run** — tags pushed with the default `GITHUB_TOKEN` don't retrigger
workflows, so a tag-triggered pipeline would need a GitHub App token; this
shape needs nothing.

Live consumers: [perch](https://github.com/stellar-registry/perch)
(smart-account authors, per-contract crate versions) and
[oz-combined-wasms](https://github.com/stellar-registry/oz-combined-wasms)
(classic-key author, single workspace version, 31 wasms per release).

## Workflows

### `detect-releases.yml`

Tag-on-manifest-bump release detection. A contract is "released" when its
manifest `version` has no matching `<tag_prefix><version>` tag yet; the job
tags it and emits `releases` (`[{package_name, tag, version}]`) +
`releases_created` for downstream jobs. Pure git — built for repos whose
releases are on-chain wasm publishes, where `cargo package`-based detection
(release-plz) can't resolve git-only / intra-workspace deps.

```yaml
detect-releases:
  if: github.event_name == 'push'
  permissions:
    contents: write
  uses: stellar-registry/actions/.github/workflows/detect-releases.yml@<sha>
  with:
    # Per-crate versions (defaults: manifest crates/<name>/Cargo.toml, tag <name>-v<ver>):
    contracts: '[{"name": "my-contract"}, {"name": "my-other-contract"}]'
    # …or a single workspace version tagged v<ver>:
    # contracts: '[{"name": "my-repo", "manifest": "Cargo.toml", "tag_prefix": "v", "version_from": "workspace"}]'
```

### `release-pr.yml`

git-cliff-driven version bumps + changelogs, maintained as a single release
PR — release-plz's UX rebuilt on git history. Scope each contract with
`include_paths` covering the intra-workspace libs it embeds, so a lib fix
re-releases the contract that ships it. The caller repo must have a
**complete** git-cliff config (partial configs silently merge with defaults).

```yaml
release-pr:
  if: github.event_name == 'push'
  permissions:
    contents: write
    pull-requests: write
  uses: stellar-registry/actions/.github/workflows/release-pr.yml@<sha>
  with:
    contracts: >-
      [{"name": "my-contract",
        "include_paths": ["crates/my-contract/**", "crates/my-lib/**"]}]
```

### `contract-release.yml`

Attested build: `stellar scaffold build` with `source_repo` metadata,
`stellar contract optimize`, a GitHub release holding
`<package>_v<version>.wasm`, the hash submitted to stellar.expert
contract-validation, and a GitHub build-provenance attestation for the asset.

```yaml
build:
  permissions:
    id-token: write
    contents: write
    attestations: write
  uses: stellar-registry/actions/.github/workflows/contract-release.yml@<sha>
  with:
    package: my-contract
    release_name: my-contract-v1.2.3
  secrets:
    release_token: ${{ secrets.GITHUB_TOKEN }}
```

### `registry-publish.yml`

On-chain publish of a **single** wasm to an **explicit registry contract id**,
authored by a **smart account** (C…): downloads the release asset, verifies
its provenance (`gh attestation verify --signer-workflow`), cross-checks the
wasm's `binver` metadata, then `contract upload` + smart-account-signed
`publish_hash` (+ optional content-addressed `deploy_stateless`) via the
CAP-71-signing [theahaco/stellar-cli](https://github.com/theahaco/stellar-cli)
fork. Attaches a publish receipt to the release.

```yaml
publish:
  needs: build
  permissions:
    contents: write
    id-token: write
  uses: stellar-registry/actions/.github/workflows/registry-publish.yml@<sha>
  with:
    registry: ${{ vars.REGISTRY_CONTRACT_ID }}
    author: ${{ vars.AUTHOR_ADDRESS }}          # smart account (C…)
    wasm_name: my-contract
    version: "1.2.3"
    release_tag: my-contract-v1.2.3
    wasm_asset: my-contract_v1.2.3.wasm
    network: testnet
  secrets:
    signing_key: ${{ secrets.CI_PUBLISH_SECRET_KEY }}  # CAP-71 delegate key
    release_token: ${{ secrets.GITHUB_TOKEN }}
```

### `registry-publish-assets.yml`

Batch publish of **every wasm on a GitHub release** via the
[stellar-registry CLI](https://github.com/stellar-registry/cli) — the
classic-key path: the author is a G… account (default: the signing key
itself), and names resolve through the network's verified registry, including
`<channel>/<name>` sub-registry names via `name_prefix`. Verifies each
asset's provenance first. Idempotent: already-published (same-hash) versions
are skipped so reruns after partial failures are safe; a same-version
different-hash asset refuses. One failing asset doesn't abort the batch.

```yaml
publish:
  needs: build
  permissions:
    contents: write
  uses: stellar-registry/actions/.github/workflows/registry-publish-assets.yml@<sha>
  with:
    release_tag: v0.7.3
    name_prefix: "oz/"        # publishes oz/<package> per <package>_v<ver>.wasm asset
    network: mainnet
  secrets:
    signing_key: ${{ secrets.REGISTRY_PUBLISH_SECRET_KEY }}
    release_token: ${{ secrets.GITHUB_TOKEN }}
```

## Pinning

Pin `uses:` references to a commit sha, not a branch — these workflows sign
attestations and submit transactions; a branch ref would let a later change to
this repo alter what your release run executes.
