# Homebrew tap for Dekopon

[Dekopon](https://github.com/dekopon-agents/dekopon) is a capability-oriented control plane for self-hosted AI agents. This tap packages its released binaries for Homebrew on macOS and Linux.

```console
brew install dekopon-agents/tap/dekopon
brew pin dekopon
```

`dekopon-agents/tap` is the whole tap name: Homebrew expands it to this repository, `dekopon-agents/homebrew-tap`. Since Homebrew 6.0, Homebrew loads a non-official tap's code only after trust. Installing the fully qualified name taps this repository and trusts only the `dekopon` formula, for your user (`~/.homebrew/trust.json`). `brew pin` keeps a routine `brew upgrade` from replacing the service binaries; upgrade deliberately with `brew unpin dekopon && brew upgrade dekopon && brew pin dekopon`. Undo with `brew uninstall dekopon`, `brew untap dekopon-agents/tap` and `brew untrust dekopon-agents/tap`.

## What you get

`dekopon` installs two daemons, `dekopon-brokerd` (the authorization broker, which owns policy, provider credentials and audit) and `dekopon-gatewayd` (the unprivileged chat gateway), with their reference documents `README.md`, `BROKER.md` and `GATEWAY.md`. Both are Unix only. Installing them starts nothing, and neither runs without an owner-authored configuration file. No provider component, configuration or service is installed.

`dekopon-console`, the optional terminal console for a running broker, is a separate formula from the same release archives: `brew install dekopon-agents/tap/dekopon-console`.

Start from the [Dekopon install section](https://github.com/dekopon-agents/dekopon#install); on a Mac, follow [Run Dekopon on a Mac](https://github.com/dekopon-agents/dekopon/blob/main/docs/run-on-mac.md) for service accounts, configuration and launchd.

## Platforms

macOS on ARM64, and Linux on ARM64 and x86-64.

`Formula/dekopon.rb` is rendered from the archives a release actually attached, so a new target appears here as soon as a release ships it. The platform list is not maintained by hand in two repositories.

Intel Macs are not supported: releases publish no `x86_64-apple-darwin` archive, and the generator skips that retired target, so the formula offers no Intel macOS download.

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
  dekopon-<version>-aarch64-apple-darwin.tar.gz
```

## License

Dual-licensed under [Apache-2.0](LICENSE-APACHE) or [MIT](LICENSE-MIT), matching Dekopon itself.
