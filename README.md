# zvps-unikraft

Repack of [zvps](https://github.com/zv201413/zvps-super) (upstream:
`ghcr.io/zv201413/zvps`) as a **Unikraft Cloud** image, built 2026-10-25 and
verified running on Unikraft (entrypoint completes, supervisord brings up
sshd + ttyd, HTTPS answers 401 unauthenticated / 200 with credentials).
Published as `unikraft.io/sleeplesshermit/zvps:latest`.

## Why a repack?

The stock OCI image does not boot under Unikraft's `base-compat` runtime.
This project rebuilds its filesystem layer by layer and packages it as an
erofs rootfs driven by a Kraftfile (spec v0.7, target `kraftcloud/x86_64`,
cmd `/usr/local/bin/entrypoint.sh`).

## Build

```sh
export TMPDIR=$HOME/workspace/tmp-build   # build needs a roomy TMPDIR
sh assemble.sh                             # pull + overlay + hosts fix
unikraft build                             # uses ./Kraftfile and ./rootfs
```

`pull_extract.py` downloads the published image layers from GHCR and
extracts them into `./rootfs`, honouring OCI whiteouts.
`merge_rootfs.py` is an alternative merger that produces a single
`rootfs-merged.tar` instead of a directory tree.
`assemble.sh` then overlays the upstream `zvps-super` source files
(`entrypoint.sh`, `supervisord.conf`, `fragments/`, …) onto the rootfs.

## The /etc/hosts fix

The merged rootfs ships an **empty** `/etc/hosts` — Docker normally injects
one at runtime, Unikraft's base-compat runtime does not. Without a
`localhost` entry the name is sent to DNS and fails, and **cloudflared exits
at startup** because it cannot bind its metrics listener on localhost
(verified 2026-10-25; the rest of its connectivity checks pass).

`assemble.sh` therefore bakes the entry into the rootfs:

```text
127.0.0.1 localhost
```

(Equivalent runtime workaround: start cloudflared with
`--metrics 127.0.0.1:20241`.)

## Layout

| File | Purpose |
| --- | --- |
| `Kraftfile` | Unikraft packaging (erofs rootfs, base-compat runtime) |
| `assemble.sh` | Full rootfs assembly: pull → overlay → hosts fix |
| `pull_extract.py` | GHCR layer pull + extract with whiteout handling |
| `merge_rootfs.py` | Alternative: merge layers into one tar |
| `.gitignore` | Keeps the ~485 MB rootfs and layer blobs out of git |
