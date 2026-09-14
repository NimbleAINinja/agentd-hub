# agentd-hub

`agentd-hub` is a local, read-only multi-machine aggregator for
[Agentd](https://github.com/clickety-clacks/agentd). It discovers machines from
Tailscale or a hosts file, runs the documented Agentd CLI through existing SSH
access, and serves complete current snapshots on one loopback HTTP listener.

Agentd Hub is an optional add-on to
[Agentd](https://github.com/clickety-clacks/agentd), the local daemon and CLI.
Install Agentd on each source machine, then use hub for a central network view.
Agentd works independently of hub.
Download published builds from [GitHub Releases](https://github.com/clickety-clacks/agentd-hub/releases).

## Installation

### Prerequisites

- An x86-64 Linux machine with glibc for the published binary.
- `curl`, `tar`, and GNU coreutils for the install commands below.
- An SSH client and non-interactive SSH access to each source machine.
- [Agentd](https://github.com/clickety-clacks/agentd) installed and running on each
  source machine, with its CLI on the remote `PATH` or in `~/.local/bin`.
- Tailscale for automatic discovery, or a hosts file as a fallback.

### Install a release

The example below installs v0.1.1 from its published archive. Run the steps in
one shell session. Use a tagged release for deployments.

1. Choose a version from [GitHub Releases](https://github.com/clickety-clacks/agentd-hub/releases)
   and download its archive and `SHA256SUMS` into a temporary directory:

   ```sh
   cd "$(mktemp -d)"
   ver=0.1.1
   base="https://github.com/clickety-clacks/agentd-hub/releases/download/v${ver}"
   curl -fsSLO "${base}/agentd-hub-${ver}-x86_64-unknown-linux-gnu.tar.gz"
   curl -fsSLO "${base}/SHA256SUMS"
   ```

2. Verify the archive against the published checksum, then extract it:

   ```sh
   sha256sum -c SHA256SUMS &&
     tar xzf "agentd-hub-${ver}-x86_64-unknown-linux-gnu.tar.gz"
   ```

3. Place the binary on your `PATH`:

   ```sh
   install -Dm755 "agentd-hub-${ver}-x86_64-unknown-linux-gnu/agentd-hub" \
     "$HOME/.local/bin/agentd-hub"
   export PATH="$HOME/.local/bin:$PATH"
   agentd-hub --help
   ```

Add `~/.local/bin` to your shell startup file's `PATH` to keep the command
available in future sessions.

## Usage

By default, the hub listens only on `127.0.0.1:8787`.

```sh
agentd-hub
```

Open [the roster page](http://127.0.0.1:8787/) in a browser on the same machine.
The command runs in the foreground. Press Ctrl+C to stop it.

To supply a fallback hosts file, create `hosts.txt` with your SSH targets:

```text
# One SSH target per line
user@machine-a
user@machine-b
```

Start with the hosts file:

```sh
agentd-hub --hosts-file ./hosts.txt
```

To use only the targets in a file and skip Tailscale discovery, use
`--sources-file`. This is the right choice for a fixed deployment topology:

```sh
agentd-hub --sources-file ./sources.txt
```

To use IPv6 loopback instead:

```sh
agentd-hub --listen '[::1]:8787' --hosts-file ./hosts.txt
```

Tailscale discovery runs first. The hosts file is used only when Tailscale
fails, returns invalid JSON, or returns no machine. Each non-empty,
non-comment line is one SSH target.

`--sources-file` and `--hosts-file` are mutually exclusive. Source names belong
in the deployment file, not in the Agentd Hub binary or repository.

### HTTP endpoints

The server exposes:

- `GET /snapshot` for one complete JSON snapshot.
- `GET /events` for complete-snapshot SSE events.
- `GET /` for the self-contained text-only roster page.

The hub has no authentication and refuses non-loopback listeners. A deployment
that exposes the listener through a reverse proxy or `tailscale serve` must
provide any required access control. The hub stores no roster,
event, or revision history and sends no commands to Agentd or agents.

## Development

Rust 1.97 or later is required.

```sh
cargo build --release --locked
cargo test --locked --all-targets
```

## Releasing

Releases are cut by CI, never by hand-uploading assets.

1. Bump the version in [Cargo.toml](Cargo.toml), [Cargo.lock](Cargo.lock), and
   `AGENTD_HUB_RELEASE_VERSION` in [the packaging script](scripts/package-release.sh)
   in one commit and push it to `main`. Wait for [CI](.github/workflows/ci.yml)
   to pass its formatting, Clippy, and locked-test checks.
2. Tag that commit `vX.Y.Z` (matching the three versions above) and push the
   tag. The [release workflow](.github/workflows/release.yml) verifies that the
   tag, crate, and packaging versions match,
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

## Help and contributions

Use [GitHub Issues](https://github.com/clickety-clacks/agentd-hub/issues) to report
bugs or request features. Include the version, the command you ran, and relevant
error output when reporting a bug.

To contribute, open a pull request with a description of the change and run the
checks in the [CI workflow](.github/workflows/ci.yml).
