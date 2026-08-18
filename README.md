# Homebrew tap for Dekopon

[Dekopon](https://github.com/dekopon-agents/dekopon) is a capability-oriented control plane for self-hosted AI agents. This tap packages its released binaries for Homebrew on macOS and Linux.

```console
brew tap dekopon-agents/tap
brew trust dekopon-agents/tap
brew install dekopon
```

`brew tap dekopon-agents/tap` is the whole name: Homebrew expands it to this repository, `dekopon-agents/homebrew-tap`.

The `brew trust` line is not optional on Homebrew 6. Homebrew refuses to load a formula from any non-official tap until you trust it, and `brew install` without it fails with `Refusing to load formula dekopon-agents/tap/dekopon from untrusted tap`. Trusting records this tap in `~/.homebrew/trust.json` and nothing else; `brew untap` plus removing that entry undoes it.

## What you get

One formula, four executables — because Dekopon publishes one archive per platform carrying all of them, and four formulae would each download the same tarball:

| Executable | What it is |
|---|---|
| `dekopon` | Local agent catalog and model-account CLI. Reads the catalog and nothing else. |
| `dekopon-run` | One-shot runner: import-free components, a sandboxed shell, a model prompt loop, and an unprivileged broker client. |
| `dekopon-brokerd` | The authorization broker daemon. Owns policy, provider credentials, and audit. Unix only. |
| `dekopond` | The unprivileged chat gateway daemon. Holds chat and model credentials, no broker authority. Unix only. |

`dekopon` and `dekopon-run` work immediately. `dekopon-brokerd` and `dekopond` are daemons: installing them starts nothing, and neither runs without an owner-authored configuration file. `brew install` prints the paths to `BROKER.md` and `GATEWAY.md`, which are installed alongside them.

The checked-in JSONPlaceholder example component is installed at `$(brew --prefix)/share/dekopon/providers/jsonplaceholder-provider.wasm`. It imports broker-owned HTTP, so it is a broker provider — name that path in `broker.yaml`. `dekopon-run inspect` is deliberately import-free and refuses to instantiate it.

The end-to-end walkthrough is [`examples/rubber-stamper`](https://github.com/dekopon-agents/dekopon/tree/main/examples/rubber-stamper).

## Platforms

macOS on ARM64, and Linux on ARM64 and x86-64.

`Formula/dekopon.rb` is rendered from the archives a release actually attached, so a new target appears here as soon as a release ships it. The platform list is not maintained by hand in two repositories.

Intel Macs are the one deliberate exception. `0.3.0` did ship an `x86_64-apple-darwin` archive, but [dekopon-agents/dekopon#74](https://github.com/dekopon-agents/dekopon/pull/74) dropped that target from the release matrix, so `0.4.0` onward has none. Offering `0.3.0`'s copy would install cleanly on an Intel Mac and then dead-end, because the next `brew upgrade` would find nothing for that platform — worse than being plainly unsupported. The generator recognizes the target and skips it. Download the [v0.3.0 archive](https://github.com/dekopon-agents/dekopon/releases/tag/v0.3.0) directly or build from a checkout instead.

## How the formula is updated

`Formula/dekopon.rb` is generated, not hand-written. Editing it directly is fine for an emergency, but the next release overwrites it. The generator lives in the Dekopon repository at [`.github/scripts/render-homebrew-formula.py`](https://github.com/dekopon-agents/dekopon/blob/main/.github/scripts/render-homebrew-formula.py), and [`.github/workflows/homebrew-tap.yml`](https://github.com/dekopon-agents/dekopon/blob/main/.github/workflows/homebrew-tap.yml) runs it when a release is published. It enumerates the release's assets, takes each `sha256` from the `.sha256` sidecar the release published rather than recomputing it, and pushes here only when the rendered bytes differ. Re-running a release commits nothing; re-running an *older* release is refused rather than rolling the tap backwards. A target the generator does not recognize fails the job instead of vanishing from the formula; a target it has retired (see [Platforms](#platforms)) is skipped, so an immutable older release cannot reintroduce a platform the project no longer builds.

### Maintainer setup: the cross-repository credential

`GITHUB_TOKEN` is scoped to the repository whose workflow is running, so it cannot push here. The workflow mints a short-lived installation token from a GitHub App instead. **This is a one-time manual setup, and nothing automatic can do it for you.**

In the `dekopon-agents` organization, at **Settings → Developer settings → GitHub Apps → New GitHub App**:

1. Name it something like `dekopon-tap-updater`. Homepage URL can be this repository.
2. **Uncheck Webhook → Active.** The default is on, and a webhook with no listener is just noise.
3. Under **Permissions → Repository permissions**, set **Contents: Read and write**. Nothing else.
4. Create the app, then note the numeric **App ID**.
5. **Generate a private key.** GitHub shows you the `.pem` download exactly once. Save it before leaving the page; if you lose it, generate a new one and delete the old.
6. **Install the app** — this is the step that actually grants it anything, and the one people miss. On the app's page choose **Install App → dekopon-agents → Only select repositories → `homebrew-tap`**. Without it the workflow authenticates fine and then 404s on push, which reads like a permissions bug and is not one.

Then, on `dekopon-agents/dekopon`, under **Settings → Secrets and variables → Actions → Repository secrets**, add:

| Secret | Value |
|---|---|
| `TAP_APP_ID` | The numeric App ID from step 4. |
| `TAP_APP_PRIVATE_KEY` | The full contents of the `.pem` file, `-----BEGIN` line through `-----END` line. |

Until `TAP_APP_ID` is set, the workflow logs a warning and skips; it does not fail a release.

An App rather than a personal access token because the minted token expires in an hour and carries one permission on one repository, so a leaked log leaks something already expiring; because the credential belongs to the organization rather than to one maintainer, so it survives that person rotating their own; and because it will not quietly expire a year from now and break releases.

## Verifying a release

Dekopon's archives are provenance-attested. Independently of Homebrew:

```console
gh attestation verify --repo dekopon-agents/dekopon \
  dekopon-0.3.0-aarch64-apple-darwin.tar.gz
```

## License

Dual-licensed under [Apache-2.0](LICENSE-APACHE) or [MIT](LICENSE-MIT), matching Dekopon itself.
