# amnezia-box builds

Cross-platform builds of [sing-box](https://sing-box.sagernet.org/) with **AmneziaWG**
support — the obfuscated WireGuard variant that survives DPI-based blocking of plain
WireGuard handshakes.

Upstream sing-box does not ship AmneziaWG. The protocol comes from the
[amnezia-box](https://github.com/hoaxisr/amnezia-box) fork, which adds an `awg`
endpoint behind the `with_awg` build tag. This repository is the Android client (a fork
of [sing-box-for-android](https://github.com/SagerNet/sing-box-for-android)) plus the CI
that produces AWG-enabled binaries for every other platform from the same core.

| Target | Workflow | Output |
|---|---|---|
| Android (arm64, armv7, x86, x86_64) | [`build.yaml`](.github/workflows/build.yaml) | APK with an AWG-enabled `libbox.aar` |
| Linux (amd64, arm64, arm, 386, riscv64, …) | [`build_linux.yaml`](.github/workflows/build_linux.yaml) | static `sing-box` + sha256 |
| Windows (amd64, arm64) | [`build_windows.yaml`](.github/workflows/build_windows.yaml) | `sing-box.exe` + sha256 |
| OPNsense / FreeBSD 15 (amd64) | [`build_opnsense.yaml`](.github/workflows/build_opnsense.yaml) | installable `.pkg` with rc.d service, `sing-boxctl` and configd actions |

Every build is pure Go with `CGO_ENABLED=0`, so the binaries are static and depend on
nothing at runtime. `naive_outbound`/cronet is intentionally left out — it needs
libcronet through CGO.

## Getting a build

The workflows are `workflow_dispatch`: open **Actions**, pick the platform, press **Run
workflow**, and download the artifact when it finishes. Inputs let you point at another
core repo, another core ref, and (for the binaries) the architectures to build.

Two core lines are maintained upstream:

| `core_ref` | sing-box line | Notes |
|---|---|---|
| `1.12.12-awg` | 1.12.12 | stable; the default for the Linux and Windows builds |
| `awg-1.14` | 1.14.0-beta.14 | current beta; the default for Android and OPNsense |

The lines are **not** config-compatible in both directions — 1.14 deprecates
`store_rdrc` and the legacy `download_detour`, and the DNS server format changed in
1.12. Pick one line and stay on it.

For the Android build both sides must be on the same line: the app's Kotlin talks to
libbox APIs that move between releases, so `app_ref` and `core_ref` are checked against
each other and the run fails early on a mismatch instead of dying in a Kotlin compile
error ten minutes in.

## Configuring AmneziaWG

In sing-box 1.12 and 1.14 AWG is an **endpoint**, not an outbound (1.13-era examples
with a `wireguard` outbound and an `awg` block do not apply):

```json
"endpoints": [
  {
    "type": "awg",
    "tag": "wg-ep",
    "private_key": "<client private key>",
    "address": "10.10.0.150/32",
    "mtu": 1420,
    "jc": 5, "jmin": 50, "jmax": 1000,
    "s1": 28, "s2": 137,
    "h1": "1631980850", "h2": "1967581631", "h3": "1485046168", "h4": "1803539852",
    "peers": [
      {
        "address": "203.0.113.10",
        "port": 55566,
        "public_key": "<server public key>",
        "preshared_key": "<psk>",
        "allowed_ips": "0.0.0.0/0",
        "persistent_keepalive_interval": 25
      }
    ]
  }
]
```

The obfuscation parameters map straight from an AmneziaWG client config: `Jc → jc`,
`Jmin → jmin`, `Jmax → jmax`, `S1 → s1`, `S2 → s2`, `H1..H4 → h1..h4`. Note that
**`h1`–`h4` are strings** here, and that `allowed_ips` is a filter, not a route: packets
outside those prefixes never enter the tunnel.

Generate a key pair with the binary itself:

```sh
sing-box generate wg-keypair
```

A complete, ready-to-edit config — AWG endpoint, split DNS, socks/http inbounds,
selector over several outbounds, routing rules, cache and Clash API — lives in
[`opnsense/etc/config.example.json`](opnsense/etc/config.example.json) and passes
`sing-box check` untouched.

## OPNsense

The FreeBSD package installs the binary, an rc.d service, a `sing-boxctl` wrapper and a
configd action set, so the service is manageable from the shell and through the OPNsense
API:

```sh
pkg add -f ./sing-box-<version>-freebsd-amd64.pkg
sing-boxctl check && sing-boxctl enable && sing-boxctl start
configctl singbox status
```

Full manual: [`opnsense/README.md`](opnsense/README.md).

## Platform notes

Things that bit us and are handled in CI, worth knowing if you build by hand:

- **`with_awg` is not in the core's default tags** on the 1.14 branch
  (`release/DEFAULT_BUILD_TAGS_OTHERS`). Building without it produces a binary that
  parses an `awg` endpoint and then refuses it with *"Awg is not included in this
  build"*. Every workflow appends the tag and then greps the artifact to prove the stub
  is absent.
- **FreeBSD panics on a `direct` outbound** in 1.14: sing-tun has no interface monitor
  outside linux/windows/darwin, and `direct.Outbound.Start()` dereferences it anyway, so
  any config with a `direct` outbound dies at startup in `fetchMyAddresses`. The
  OPNsense build patches the guard in and then actually starts the daemon to prove it.
- **`pkg add` segfaults on OPNsense** if the package was created by pkg ≥ 2.4: OPNsense
  ships pkg 2.3.1, which does not understand the newer per-file manifest layout, drops
  the checksum and crashes. The packaging job rewrites the manifest into the flat layout
  both versions read.
- **Android needs `with_awg` injected** into `cmd/internal/build_libbox`, which carries
  its own tag list; the workflow patches it and then counts `amneziawg` symbols in the
  built `libbox.so`.

## Credits and license

- sing-box — [SagerNet](https://github.com/SagerNet/sing-box), nekohasekai
- AmneziaWG support — [amnezia-box](https://github.com/hoaxisr/amnezia-box)
- Android client — fork of [sing-box-for-android](https://github.com/SagerNet/sing-box-for-android)

```
Copyright (C) 2022 by nekohasekai <contact-sagernet@sekai.icu>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <http://www.gnu.org/licenses/>.

In addition, no derivative work may use the name or imply association
with this application without prior consent.
```

Under the license, forks of the app are not allowed to be listed on F-Droid or other app
stores under the original name.
