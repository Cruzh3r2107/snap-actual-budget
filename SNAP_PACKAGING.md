# Packaging Actual Budget's sync server as a snap

This repo packages `@actual-app/sync-server` — the self-hosted sync
backend for [Actual Budget](https://actualbudget.org/) — plus its bundled
`@actual-app/web` static frontend, as a single strictly-confined snap named
`vsingh-actual`.

Source of truth for every decision below is `snap/snapcraft.yaml`. This
document explains build/install/test steps and a few choices that aren't
obvious from the YAML alone.

## Prerequisites

- `snapcraft` (this was built and tested against 9.0.0, targeting `base:
  core26`)
- LXD, for snapcraft's default isolated build environment:
  ```
  sudo snap install lxd
  sudo lxd init --auto
  ```
- Network access from the build environment: the build clones
  `actualbudget/actual` at tag `v26.8.0` from GitHub, downloads the official
  Node.js v22.18.0 linux-x64 tarball from nodejs.org, and runs a full `yarn
  install`. All three need outbound network access during the build (the
  default snapcraft LXD builder has it).

## Build

From the project root:

```bash
export SNAPCRAFT_BUILD_INFO=1
snapcraft pack
```

This produces `vsingh-actual_26.8.0_amd64.snap` (or similar, depending on
architecture) in the project root.

Do **not** pass `--destructive-mode` — that builds directly on the host
instead of in the isolated LXD container and can pollute the host with
build-only dependencies (build-essential, python3, the Node.js tarball,
etc.).

## Why the source is a local path, not a remote git URL

`.gitignore` in this repo is deliberately "ignore everything, then
re-include only the packaging files" (`snap/`, `README.md`,
`SNAP_PACKAGING.md`, `SNAP-PROGRESS.md`). The full upstream Actual checkout
that lives alongside these files on disk is **not** committed here — it
only exists locally so the analyzer/packager tooling had something to
inspect. Anyone who clones just this packaging repo would not have that
tree, which is why the original intent (per `.gitignore`'s own comment) was
for `snap/snapcraft.yaml`'s `sync-server` part to pull source directly from
`https://github.com/actualbudget/actual.git` at `source-tag: v26.8.0`, so
the packaging repo alone would be self-sufficient.

**That was tried first and reverted.** The snapcraft LXD build instance in
this environment can reach `nodejs.org` and `archive.ubuntu.com` but times
out connecting to `github.com` specifically (`git clone` failed with
`Failed to connect to github.com port 443 ... Couldn't connect to server`
after the DNS resolved fine — confirmed independently with `curl` from
inside the build instance). Since that block is specific to this build
environment's egress policy, not a universal constraint, `source:` is
currently set to `.` (the project root, which snapcraft already mounts into
the build instance, so no extra network access is needed to obtain it). If
your build environment has unrestricted access to github.com, switch it
back:

```yaml
source: https://github.com/actualbudget/actual.git
source-type: git
source-tag: v26.8.0
```

That restores the "packaging repo alone is self-sufficient" property this
was originally designed for.

## Why `plugin: nil` with a hand-rolled `override-build`

`@actual-app/sync-server` is one workspace in a Yarn Berry (v4) monorepo
built with `lage`, and its production `node_modules` needs to be pruned and
partially reconstructed (workspace symlinks dereferenced) after the build —
none of snapcraft's built-in plugins (`nodejs`, `npm`) model that.
`override-build` reproduces `sync-server.Dockerfile`'s `builder` stage
end-to-end, step by step (see the comments inline in `snapcraft.yaml`):

1. Download and unpack the **official Node.js v22.18.0 linux-x64 tarball**
   (not core26's archive `nodejs`, which is 22.22.1) so the build uses the
   exact version pinned by `.nvmrc` and `engines.node` in `package.json`.
   `NODE_OPTIONS=--max_old_space_size=8192` is set in `build-environment`
   — without it, the build OOMs.
2. `yarn install` (full, non-production), via the vendored
   `.yarn/releases/yarn-4.17.1.cjs` release, run with the freshly unpacked
   node — no reliance on a system yarn/corepack.
3. Seed a throwaway git repo (`git init && add -A && commit`) purely so
   `lage`'s task hasher — which shells out to `git ls-tree HEAD` during
   initialization even when caching is disabled for a target — has
   something to hash against. This inner repo is unrelated to, and does
   not interact with, this outer packaging repo's own git history.
4. `./bin/package-browser --skip-translations` + `yarn workspace
   @actual-app/sync-server build` (equivalent to `yarn build:server`, i.e.
   `build:browser && workspace build`). **`--skip-translations` is
   required**: without it, `bin/package-browser` git-clones
   `actualbudget/translations` into `packages/desktop-client/locale`,
   which would make this build depend on network access to a *second*,
   unrelated upstream repo.
5. `yarn workspaces focus @actual-app/sync-server --production` to prune
   dev dependencies out of `node_modules`.
6. Dereference the `@actual-app/web` and `@actual-app/sync-server`
   workspace symlinks in `node_modules` and manually recreate
   `node_modules/@actual-app/web/{package.json,build}` from the built
   `packages/desktop-client` artifacts. `load-config.js` resolves the web
   UI's static assets via `require.resolve('@actual-app/web/package.json')`
   at runtime — if the symlink is left in place, the snap builds clean but
   fails at runtime with a module-not-found error once the symlink target
   (the source tree, not part of the runtime payload) is gone.
7. Stage **exactly** `node_modules/`, `packages/sync-server/package.json`,
   and `packages/sync-server/build/` into `$CRAFT_PART_INSTALL`, as
   siblings (`node_modules` next to `package.json` next to `build/`) —
   this mirrors `sync-server.Dockerfile`'s prod stage layout and is what
   the built bundle's relative resolution (migrations glob-inlined at
   Vite build time, `require.resolve('@actual-app/web/...')`) expects.
8. Copy the Node binary built in step 1 into
   `$CRAFT_PART_INSTALL/bin/node` — just the executable, not the full
   tarball (npm/corepack/headers aren't needed at runtime; the native
   addons are already compiled by this point).
9. Write `$CRAFT_PART_INSTALL/bin/run-daemon`, a wrapper script (see next
   section) — a bare `command:` string can't `cd` first or run migrations,
   and snapd does not set the app's working directory to `$SNAP`.

## The `bin/run-daemon` wrapper and app command

The app's `command:` is `bin/run-daemon`, not a bare `bin/node build/app.js`
string, because:

- snapd does not set the process's CWD to `$SNAP`, but the runtime payload's
  sibling layout (`node_modules` next to `package.json` next to `build/`)
  needs `$SNAP` as the working directory for Node's module resolution to
  find everything the same way it would under `sync-server.Dockerfile`'s
  `WORKDIR /app`.
- Database migrations need to run before `app.js` starts (see next
  section).

## Migrations: run idempotently on every daemon start, not (only) from the install hook

`app.js` never creates its own data directories or runs migrations. Only
`packages/sync-server/migrations/1694360000000-create-folders.js` creates
`serverFiles`/`userFiles`, and it only runs via `node
build/scripts/run-migrations.js up`. Nothing in `app.ts`/`app.js` invokes
that automatically.

Two options were available: run migrations once from the `install` hook, or
run them (safely) every time the daemon starts. **This packaging chose the
latter** — `bin/run-daemon` runs `node build/scripts/run-migrations.js up`
before `exec`-ing `build/app.js`, on every service start/restart:

- The `migrate` package tracks applied state in `<dataDir>/.migrate`, so
  re-running `up` when there's nothing new to apply is a fast no-op — it is
  safe to do unconditionally.
- snapd's `install` hook only fires on install, not on every future
  `refresh`. If a future update to this snap ships new migrations, an
  install-hook-only strategy would silently skip them on upgrade. Running
  at every daemon start (which happens after every `refresh` when the
  service is restarted) means new migrations are applied automatically.

The `install` hook (`snap/hooks/install`) is still required and does one
thing: `mkdir -p $SNAP_COMMON/data{,/server-files,/user-files}`. This
guarantees `$SNAP_COMMON/data` exists (and thus `account-db.js`'s
`getAccountDb()` has somewhere to open `account.sqlite`) even before the
very first successful migration run, and pre-creates `server-files`/
`user-files` defensively — the actual migration itself only does a
non-recursive `fs.mkdir`, so its parent (`dataDir`) must already exist.

## Environment variables

| Variable | Value | Purpose |
|---|---|---|
| `NODE_ENV` | `production` | disables the dev-only rate-limit bypass and other dev branches |
| `ACTUAL_DATA_DIR` | `$SNAP_COMMON/data` | relocates all persistent state (account.sqlite, server-files/, user-files/, the `.migrate` state file) out of the read-only `$SNAP` |

No other filesystem paths outside `dataDir` are referenced anywhere in
`packages/sync-server/src` (checked `app.ts`, `account-db.js`, `db.js`,
`migrations.ts`) — no hardcoded `/etc`, `/var`, or `$HOME` use, so no
`layout:` remapping is needed.

To change the listen port/hostname or point elsewhere, override via
`snap set` is not wired up in this initial packaging — set
`ACTUAL_PORT`/`ACTUAL_HOSTNAME`/etc. by editing the `environment:` block in
`snap/snapcraft.yaml` and rebuilding, or extend `bin/run-daemon` to read
`snapctl get` config keys if that's wanted later.

## Interfaces

| Interface | Auto-connected | Reason |
|---|---|---|
| `network` | yes | Outbound HTTPS to bank-aggregation providers (GoCardless, SimpleFin, Akahu, Pluggy, EnableBanking) and OpenID provider discovery/token/userinfo endpoints when OpenID login is configured. |
| `network-bind` | yes | `app.listen(port, hostname)` (default port 5006, hostname `::`) serves both the sync API and the bundled web UI. |

Both are auto-connected on install — no manual `snap connect` is needed.

## Install and test

First test in devmode (bypasses strict-confinement AppArmor/seccomp
enforcement, useful for isolating packaging bugs from confinement bugs):

```bash
sudo snap install --devmode ./vsingh-actual_26.8.0_amd64.snap
sudo snap start vsingh-actual
journalctl -u snap.vsingh-actual.vsingh-actual -f
curl http://localhost:5006/info
```

Once that works, test with real strict confinement:

```bash
sudo snap remove vsingh-actual
sudo snap install --dangerous ./vsingh-actual_26.8.0_amd64.snap
sudo snap start vsingh-actual
```

(`--dangerous` is required instead of a plain `snap install` because the
snap isn't signed/from the store.)

## Troubleshooting

- **AppArmor/seccomp denials**: `sudo snap run --shell vsingh-actual` to get
  a shell inside the confined environment, or inspect
  `journalctl -xe | grep -i denied` for `audit: ... apparmor="DENIED"`
  lines. `snappy-debug` can help decode which interface a denial maps to.
- **Service won't start / crashes immediately**:
  `journalctl -u snap.vsingh-actual.vsingh-actual -n 100 --no-pager`.
- **Module-not-found for `@actual-app/web`**: means the
  `node_modules/@actual-app/web` symlink-dereference step in
  `override-build` didn't run or was skipped — check the build log for
  step 6.
- **Migrations not applying / `auth` table missing**: confirm
  `bin/run-daemon` actually ran (check for its `Migrations: DONE` log line
  in `journalctl`) before `app.js` started.

## Notes carried over from `snap-analysis.json`

- **Node version**: this snap builds and ships **Node.js v22.18.0**, taken
  from the official linux-x64 tarball, matching this checkout's `.nvmrc`.
  The `24.18.1` string that appears in the analysis file only documents a
  discrepancy in the originally-supplied context — 24.18.1 is not a real
  version for this project and does not appear anywhere in
  `snap/snapcraft.yaml`.
- **Native modules compiled at build time**: `better-sqlite3`, `argon2`,
  and `bcrypt` all need `node-gyp` (hence `build-essential` + `python3` in
  `build-packages`) and are explicitly allow-listed in
  `dependenciesMeta` with `"built": true"` in the root `package.json`,
  which is required because `.yarnrc.yml` sets `enableScripts: false`
  globally.
- **Confinement**: kept `strict` — nothing in the sync-server source execs
  arbitrary host binaries or otherwise needs classic confinement.
- **Grade**: `devel`, matching `snap-analysis.json`'s `project.grade`. Bump
  to `stable` deliberately once this has been validated end-to-end (this
  initial packaging pass has not run `snap-validator`).
