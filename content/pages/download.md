---
title: "Download ukue"
url: "/download/"
type: page
draft: false
---

The `ukue` command is one file with nothing else to install. Each archive below holds it with its license files. The links always point to the latest release.

<div style="overflow-x:auto">

| System | Download |
|---|---|
| Linux, x86-64 | [ukue_linux_amd64.tar.gz](https://github.com/ukue-queue/ukue/releases/latest/download/ukue_linux_amd64.tar.gz) |
| Linux, ARM64 | [ukue_linux_arm64.tar.gz](https://github.com/ukue-queue/ukue/releases/latest/download/ukue_linux_arm64.tar.gz) |
| macOS, Apple silicon | [ukue_darwin_arm64.tar.gz](https://github.com/ukue-queue/ukue/releases/latest/download/ukue_darwin_arm64.tar.gz) |
| macOS, Intel | [ukue_darwin_amd64.tar.gz](https://github.com/ukue-queue/ukue/releases/latest/download/ukue_darwin_amd64.tar.gz) |
| Windows, x86-64 | [ukue_windows_amd64.zip](https://github.com/ukue-queue/ukue/releases/latest/download/ukue_windows_amd64.zip) |

</div>

*Checksums are in [SHA256SUMS](https://github.com/ukue-queue/ukue/releases/latest/download/SHA256SUMS). Every release is listed on [GitHub](https://github.com/ukue-queue/ukue/releases).*

## Install

On Linux or macOS:

```sh
tar -xzf ukue_linux_amd64.tar.gz
sudo mv ukue /usr/local/bin/
ukue version
```

The Linux binaries are static, so they run on any distribution. The macOS binaries aren't signed by Apple, so macOS may block the first run; allow it in System Settings under Privacy & Security, or run `xattr -d com.apple.quarantine ukue`. On Windows, unzip the archive and put `ukue.exe` in a folder on your PATH.

Each binary is built by [the release workflow](https://github.com/ukue-queue/ukue/blob/main/.github/workflows/release.yml) on GitHub. Before a release goes out, the workflow runs the Linux, Windows and Apple silicon builds through a short test.

## Build It Yourself

You need Go 1.25 or later and a C compiler, because ukue compiles SQLite into itself.

```sh
go install github.com/ukue-queue/ukue/cmd/ukue@latest
```

## Use It as a Go Library

```sh
go get github.com/ukue-queue/ukue
```

The [quick start](https://ukue.com/quick-start/) shows the code.

## License

ukue is open source under the Apache License 2.0. The binaries also contain go-sqlite3 (MIT license), SQLite (public domain) and the Go runtime (BSD license). Their notices come in each archive.
