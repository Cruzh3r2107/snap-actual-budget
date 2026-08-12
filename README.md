# actual-budget-snap

Actual Budget is a super fast and privacy-focused app for managing your finances. At its heart is the well proven and much loved Envelope Budgeting methodology. You own your data and can do whatever you want with it. Featuring multi-device sync, optional end-to-end encryption and so much more.

This repository packages the Actual sync server as the `vsingh-actual` snap. It
builds from upstream `actualbudget/actual` at tag `v26.8.0` and ships both the
sync/API backend and the web UI as a single strictly-confined daemon.

## Install

```bash
sudo snap install --dangerous ./vsingh-actual_26.8.0_amd64.snap
```

## Use it

Browse to <http://localhost:5006>. The web UI and the sync API are served on
the same port, so there is nothing else to run and no reverse proxy required
for local use.

The daemon starts automatically after install:

```bash
snap services vsingh-actual
snap logs vsingh-actual -f
```

## Your data

Everything lives in `/var/snap/vsingh-actual/common/data`:

```
account.sqlite     accounts and sync metadata
server-files/      per-budget sync databases
user-files/        uploads
```

This is `$SNAP_COMMON`, so it survives refreshes **and** reverts. Back up that
directory and you have backed up everything.

Database migrations run automatically on every start and are idempotent, so a
refresh that ships new migrations applies them with no action from you.

## Interfaces

`network` and `network-bind` are the only interfaces used, and both connect
automatically. Nothing to connect by hand.

## Troubleshooting

The server binds all interfaces on port 5006, so it is reachable from your LAN.
Put it behind a reverse proxy or a VPN if that is not what you want.

If the service will not start, `snap logs vsingh-actual -n 50` shows the
migration output and the listening line.

---

Build instructions and packaging internals are in
[SNAP_PACKAGING.md](SNAP_PACKAGING.md).
