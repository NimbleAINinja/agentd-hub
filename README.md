# agentd-hub

`agentd-hub` is a local, read-only multi-machine aggregator for
[Agentd](https://github.com/clickety-clacks/agentd). It discovers machines from
Tailscale or a hosts file, runs the documented Agentd CLI through existing SSH
access, and serves complete current snapshots on one loopback HTTP listener.

The current release is **v0.1.0**.

## Install

Install a tagged, published release. Do not build from a branch for a real
deployment.

1. Pick the latest release from
   <https://github.com/clickety-clacks/agentd-hub/releases> and download its
   archive and `SHA256SUMS`:

   ```sh
   ver=0.1.0
   base="https://github.com/clickety-clacks/agentd-hub/releases/download/v${ver}"
   curl -fsSLO "${base}/agentd-hub-${ver}-x86_64-unknown-linux-gnu.tar.gz"
   curl -fsSLO "${base}/SHA256SUMS"
   ```

2. Verify the archive against the published checksum, then extract it:

   ```sh
   sha256sum -c SHA256SUMS
   tar xzf "agentd-hub-${ver}-x86_64-unknown-linux-gnu.tar.gz"
   ```

3. Place the binary on your `PATH`:

   ```sh
   install -Dm755 "agentd-hub-${ver}-x86_64-unknown-linux-gnu/agentd-hub" \
     "$HOME/.local/bin/agentd-hub"
   agentd-hub --help
   ```

Every source machine must already run Agentd and be reachable over your
existing non-interactive SSH. The hub adds no listener or credential to Agentd;
it invokes the documented Agentd CLI over SSH.

## Run

By default, the hub listens only on `127.0.0.1:8787`.

```sh
agentd-hub
agentd-hub --hosts-file ./hosts.txt
agentd-hub --listen '[::1]:8787' --hosts-file ./hosts.txt
```

Tailscale discovery runs first. The hosts file is used only when Tailscale
fails, returns invalid JSON, or returns no machine. Each non-empty,
non-comment line is one SSH target.

The server exposes only:

- `GET /snapshot` for one complete JSON snapshot.
- `GET /events` for complete-snapshot SSE events.
- `GET /` for the self-contained text-only roster page.

The hub has no authentication and refuses non-loopback listeners. A deployment
that exposes the listener owns that choice (for example a reverse proxy or
`tailscale serve` in front of the loopback bind). The hub stores no roster,
event, or revision history and sends no commands to Agentd or agents.

## Build and test (development)

Rust 1.97 or later is required.

```sh
cargo build --release --locked
cargo test --locked --all-targets
```

## Releasing

Releases are cut by CI, never by hand-uploading assets.

1. Bump the version in `Cargo.toml`, `Cargo.lock`, and
   `scripts/package-release.sh` (`AGENTD_HUB_RELEASE_VERSION`) in one commit and
   push it to `main`. CI (`ci.yml`) runs format, clippy, and locked tests.
2. Tag that commit `vX.Y.Z` (matching the three versions above) and push the
   tag. The release workflow (`release.yml`) verifies tag/crate/package parity,
   runs the gates, builds `--release --locked`, packages twice and byte-compares
   the archives, then publishes the GitHub release with the archive and
   `SHA256SUMS`.

To inspect the archive locally without publishing:

```sh
cargo build --release --locked
scripts/package-release.sh --dry-run
scripts/package-release.sh
```

This writes `target/release-assets/agentd-hub-<version>-<rust-host>.tar.gz` and
`target/release-assets/SHA256SUMS`.
