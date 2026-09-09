# Changelog

## 0.1.1

- **Sync `pi-seed` with the fix found deploying the k3s sibling.** `pi-seed`'s `chmod 700` on the pi state directory requires owning it, not just having write access — on a fresh k3s PVC (root-owned even after `fsGroup` grants group-write) this crash-looped the initContainer forever under `set -eu`. This add-on's own `init-woow` runs as root, so the bug never manifested here, but `pi-seed` is byte-identical across all three deployments and hash-pinned in `rootfs/opt/SHA256SUMS` — re-vendored from `Woow_podman_code_server_package` to keep it that way. No behavior change on HA.

## 0.1.0

- **Initial release.** Layers pi 0.83.0, pi-acp 0.0.33, and the ACP Client 0.2.0 chat sidebar onto `ghcr.io/hassio-addons/vscode:7.0.0`.

- **Replaces unversioned `init_commands` wiring.** The community `vscode` mirror add-on gets its pi install from ad-hoc `packages`/`init_commands` options typed into the add-on's Info tab — no version pin, no repo, no CI. The live install on that path had drifted to pi 0.74.2 with Node 20, silently, because nothing asserted a version. This add-on pins pi 0.83.0 and Node 22 in the image itself, verified by a build-time assertion (`test "$(pi --version)" = "0.83.0"`).

- **Adds pi-acp + the ACP Client extension.** Neither exists on the `init_commands` path — only a bare `pi-ha` terminal wrapper does. This add-on adds the full ACP chat sidebar, matching the podman package's feature set.

- **Settings applied via an idempotent `jq` merge, not a baked default.** Upstream's `init-code-server` only re-copies its own default `settings.json` when the live file's sha512 matches one of 14 known-default hashes — a new default baked into the image would never reach an existing install. `init-woow` merges the required ACP + workspace-trust keys into whatever settings.json already exists, unconditionally, every boot, preserving any unrelated user edits.

- **PI_\* environment reaches the extension host and every integrated terminal**, not just a separate wrapper binary — written to `/run/s6/container_environment/` so upstream's own `with-contenv` service inherits it. Fixes the class of bug where `pi-ha` works but plain `pi` in a terminal re-prompts for login.

- **pi state is private to this add-on.** `/data/pi-agent` is this add-on's own Supervisor-managed `/data`, seeded by `pi-seed` (byte-identical to the podman package and the k3s chart — see `rootfs/opt/SHA256SUMS`). Nothing is shared with the separate `Woow HA Pi Agent` add-on or with the podman/k3s deployments; each authenticates its own pi via one `pi login`.

- **F1/F2 fallback ladder for the ACP chat webview**, in case Home Assistant ingress ever behaves differently on some installs than the one this was verified against: a seeded `pi` terminal task (F1, always present) and an optional non-ingress direct port with HTTP Basic auth (F2, opt-in via `direct_password`).
