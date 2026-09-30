# Docker Sandboxes rv kit

A kit that installs [rv](https://rv.dev), a very fast Ruby and gem
manager, and wires up shell activation so `ruby` / `gem` commands and
`.ruby-version` switching work inside the sandbox out of the box.

## Usage

`rv` is agent-agnostic — pair it with whichever agent you're using. Add
it to the `kits` list of your `sbxenv.yaml`:

```yaml
kits:
  - source: ./sbx-rv-kit
```

Once attached, `rv` and `rvx` are on PATH, and any interactive bash
shell has rv's activation and completions loaded:

```console
agent@sandbox:~$ rv --version
rv 0.7.1
agent@sandbox:~$ cd ~/my-project   # picks up .ruby-version
agent@sandbox:~$ rv ruby install   # installs the project's pinned Ruby
agent@sandbox:~$ rv clean-install  # installs the gems from Gemfile.lock
```

## How the install works

The install runs in three steps:

1. **`xz-utils`** is installed via apt (as root). It's needed both to
   extract rv's own `.tar.xz` release and for the Ruby tarballs rv
   downloads at runtime.
2. **The prebuilt rv v0.7.1 binary** for the sandbox's architecture
   (`aarch64` or `x86_64`, Linux/glibc) is downloaded from the
   [spinel-coop/rv](https://github.com/spinel-coop/rv) GitHub release,
   verified against a SHA256 digest pinned in `spec.yaml`, and `rv` plus
   `rvx` are installed to `/usr/local/bin/`. A digest mismatch or any
   other architecture fails the install with an explicit error.
3. **Shell activation** is appended to the agent user's `~/.bashrc`
   (see below).

Version and per-arch digest live in git, so reviewers can see exactly
which binary lands on PATH. To bump rv, change `RV_VERSION` and both
`SHA256` values in `spec.yaml` — the digests come from the release's
`rv-<target>.tar.xz.sha256` assets.

## Shell activation

The install step appends this block to the agent user's `~/.bashrc`:

```bash
# rv (added by sbx-rv-kit)
eval "$(rv shell init bash)"
eval "$(rv shell completions bash)"
```

`rv shell init` sets up PATH / `GEM_HOME` for the active Ruby and
switches versions automatically when you `cd` into a directory with a
`.ruby-version`. The append is guarded with `grep -qF`, so re-running
the install doesn't duplicate the lines.

This only covers **interactive bash shells**. Non-interactive shells
(e.g. commands an agent runs through its tool) usually don't source
`~/.bashrc`, so `rv` is on PATH but `ruby` / `gem` may not be. In that
case either use `rv run <command>` (which also auto-installs the pinned
Ruby), or run through a login/interactive shell. For zsh or fish, wire
activation up yourself with the corresponding `rv shell init zsh|fish`.

## Network policy

The kit allows exactly the hosts that the install and rv's common
runtime paths need:

- `github.com` — the rv release download at install time, and the
  prebuilt Ruby tarballs from
  [spinel-coop/rv-ruby](https://github.com/spinel-coop/rv-ruby) that
  `rv ruby install` fetches at runtime
- `api.github.com` — Ruby version resolution
  (`repos/spinel-coop/rv-ruby/releases/latest`). Without it,
  `rv ruby install` / `rv run` fail even for an exact version
- `objects.githubusercontent.com`,
  `release-assets.githubusercontent.com` — redirect targets of GitHub
  release-asset downloads. For rv and rv-ruby every download was
  confirmed to 302 to `release-assets.…`; `objects.…` is allowed as
  well, since GitHub doesn't guarantee the same target host for every
  repo or over time
- `rubygems.org` — rv's default gem source, used by `rv clean-install`,
  `rvx` and `rv tool install`. Gem downloads, gemspecs and the
  dependency API are all served from this host directly, with no
  separate CDN domain

If your project pulls gems from other sources (a private gem server,
`git:` sources in the `Gemfile` on hosts other than github.com, …),
add those hosts to `permissions.network.allow` in `spec.yaml`.
