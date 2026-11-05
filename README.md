# python-mashumaro

**This repository publishes nothing today.** It holds a complete, verified
backport of `python-mashumaro` to **noble** (24.04) that cannot be used,
and the record of why.

`python-mashumaro` entered Ubuntu in **resolute** (26.04). resolute and
stonking build it from their own archives and need nothing from here. noble
has none, and
[`python-kasa`](https://forgejo.bobbalisa.org/pillarsdotnet/python-kasa)
build-depends on `python3-mashumaro (>= 3.15)` on every series, so noble
would be the one target — and noble is the one series that cannot have it.

## Why noble cannot have it

The archive now carries 3.17, which imports `evaluate_forward_ref` and
`get_annotations` from `typing_extensions`. Both are new in
`typing_extensions` 4.13, so upstream's `>= 4.14.0` floor is a real one, and
`${python3:Depends}` comes out as:

```
Depends: python3-msgpack, python3-tomli-w, python3-yaml,
         python3-typing-extensions (>= 4.14.0), python3:any
```

noble ships `python3-typing-extensions` **4.10.0**, and no noble pocket —
release, updates, security or backports — has anything newer:

```
python-typing-extensions | 4.10.0-1 | noble
python-typing-extensions | 4.15.0-2 | resolute
python-typing-extensions | 4.16.0-5 | stonking
```

So the deb installs nowhere on noble. The *build* is fine: it was run
end to end in an `ubuntu:24.04` container on 2026-11-05 and produced
`python3-mashumaro_3.17-1~ppa1~ubuntu24.04.1_all.deb`. It is the result that
has no use, which is why nothing is scheduled rather than something being
patched.

The registry still carries `python3-mashumaro 3.15-1~ppa1~ubuntu24.04.1` for
noble, built by hand in October 2026. 3.15 asked only for
`typing_extensions >= 4.1` and still used `setup.py`. Nothing here rebuilds
it: 3.15 is in Debian stable and in no Ubuntu suite at all, so a repository
that tracks the Ubuntu archive cannot reach it.

## What is in here

Everything the build needs, and no workflow to run it.

```
patches/noble/       # appended to the distribution's own series
    ppa-0001-declare-the-licence-in-a-table-for-setuptools-68.patch
    series
drops/noble          # control lines to strip for noble: python3-orjson
.forgejo/
    workflows/mirror.yml     # rebuild the github branch, push it to GitHub
    registry-upload          # the whole build, with a guard
```

There is **no `.forgejo/workflows/registry.yml`**, which is what "no
targets" means in practice: no push, no cron and no dispatch builds
anything. `registry-upload` refuses to run without `BUILD_ANYWAY=1` and
prints the reason, so a hand-run cannot quietly produce the uninstallable
deb either.

The two pieces of delta are kept because they are correct and were paid for:

- **`patches/noble/ppa-0001`** puts the licence declaration back into a
  table. 3.17 moved to PEP 639, so `pyproject.toml` declares a bare SPDX
  string, and noble's `setuptools` 68.1.2 validates against the older
  schema, where `license` must be a table with either a `file` or a `text`
  key. A bare string matches *both* definitions, so it is valid by neither:

  ```
  ValueError: invalid pyproject.toml config: `project.license`.
  configuration error: `project.license` must be valid exactly by one
  definition (2 matches found)
  ```

- **`drops/noble`** strips `python3-orjson` from the binary package's
  `Depends`. noble has no `python3-orjson` in any component. `mashumaro`
  reaches `orjson` only through `mashumaro.codecs.orjson`, which nothing on
  noble could import in any case. Verified: the built deb's `Depends` loses
  the line and the control data mentions `orjson` nowhere.

The test suite cannot run on noble either — `Build-Depends-Indep` wants
`python3-pendulum` 3, `python3-typing-extensions (>= 4.14.0)` and
`python3-orjson` — so the script's `nocheck` retry takes over. Upstream's
tests do run on resolute and stonking, where the archive builds this source
package itself.

## Turning it on

Adding `.forgejo/workflows/registry.yml` is the whole job. Copy
[`python-ajsonrpc`](https://forgejo.bobbalisa.org/pillarsdotnet/python-ajsonrpc)'s
copy and give it one matrix entry:

```yaml
          - series: noble
            release: "24.04"
```

Do that when **either** of these becomes true:

- noble gains `python3-typing-extensions (>= 4.14.0)` from somewhere. It
  will not: noble is frozen and backports has nothing.
- the archive's `python-mashumaro` drops back to a floor noble can meet, in
  which case check whether `patches/noble/` is still needed at all.

Alternatively, backport `python-typing-extensions` 4.16.0-5 from stonking to
noble in its own repository and make this one depend on that landing first.
That was considered and set aside on 2026-11-05: it upgrades a library many
packages import, fleet-wide, to unblock one.

## Related

- [`python-kasa`](https://forgejo.bobbalisa.org/pillarsdotnet/python-kasa) —
  what wanted this; its noble build keeps its `nocheck` fallback.
- [`python-ajsonrpc`](https://forgejo.bobbalisa.org/pillarsdotnet/python-ajsonrpc)
  — the same arrangement for a gap that *could* be closed, and the script
  this one is derived from.
