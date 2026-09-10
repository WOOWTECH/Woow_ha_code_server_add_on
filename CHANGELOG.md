# Changelog

## 0.1.2

- **Stops silent Unicode-space path corruption.** pi folds U+00A0, U+2000-200A, U+202F, U+205F and U+3000 to an ASCII space on every read, write and edit, and builds its read fallback chain from the already-folded path — so the exact path the caller asked for is never tried. With `Q1　報告.txt` (U+3000) and `Q1 報告.txt` (ASCII) both present, reading the first returned the **second** file's contents with `isError: false`, and writing to the first overwrote the second. U+3000 IDEOGRAPHIC SPACE is ordinary in Traditional Chinese and Japanese filenames, so on a zh-TW install this is data loss, not an edge case. `patches/fix-unicode-space-paths.mjs` (byte-identical with the podman package) turns the folding into a read-only *fallback*: the exact path is tried first and writes are never rewritten, while a path pasted with a non-breaking space still resolves. The build fails rather than skipping if upstream refactors the file.

- **Ships `patches/f1-verify.mjs` inside the image, and the smoke tests run it.** This is the more important half. The original incident was not a broken patch script — it was a patch script that existed in a sibling repo and was never invoked, which no build-time assertion can catch. The verifier asserts the marker is present in *every* `path-utils.js` copy **and** that the behaviour is correct against a real filesystem. CI additionally asserts the Dockerfile still calls both scripts.

- **`pi-code` carries git identity across the HOME switch.** `pi-code` re-points `HOME` at the pi state volume, which means the ACP panel's pi resolved a different `~/.gitconfig` from the terminal's pi — the same repo could get commits from two different authors. It now captures the login HOME's `.gitconfig` into `GIT_CONFIG_GLOBAL` before overwriting `HOME`. Not reproducible on HA (both are root), but `pi-code` is byte-identical across all three deployments, so the fix and its hash land here too.

- **Corrects two false comments.** `/etc/profile.d/pi.sh` and the podman package's Containerfile both claimed the `~/.pi` symlink was a `$HOME/.pi` fallback that "resolves to the same store". It is not: only `agent/skills` lives under it, and a pi started without `PI_CODING_AGENT_DIR` finds its skills and then fails with "No API key found" — verified. It is now described as the skills bridge it is, with a note explaining why symlinking `auth.json` in would make things worse (pi rewrites it write-temp-then-rename on OAuth refresh, splitting the credential store).

- **Comments in the shared files are now platform-neutral, and CI enforces it.** The 0.1.1 copies of `pi-code` and `pi.sh` were verbatim podman-package files whose comments named `/home/coder` and "the .197 podman host" — this image runs as root and has no `coder` user, and its `~/.pi` symlink is created at runtime by `init-woow`, not by the image build. A reader debugging "will the symlink come back after a rebuild?" was actively misled. CI now fails on any `/home/coder` in `Dockerfile`, `rootfs/` or `patches/`.

- **`tests/smoke-toolchain.sh` (new).** pip/venv, `npm install -g` (including the result being on `PATH` in a *login* shell), `git commit`, panel-vs-terminal git identity, and `pi` resolving identically in login and non-login shells. Every check exists because the 2026-09 field test found it broken on a deployment that passed all the other suites. HA passes all of them unchanged — it inherits pip and runs as root — which is exactly why they are worth asserting rather than assuming.

- **`/opt/pi-agent-skel/npm-global/bin`** added so `/data/pi-agent` has the same shape on all three deployments, and `npm install -g --prefix /data/pi-agent/npm-global <pkg>` is a working recipe for a global that survives a restart.

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
