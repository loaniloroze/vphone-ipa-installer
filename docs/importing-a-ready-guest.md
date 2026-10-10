# Importing a ready-made guest

A guest that already has root ssh, the roothide bootstrap, frida-server and the
installer's prerequisites can be moved to another Mac as a single archive, so
the target host never runs `vm create`.

## The file

```
~/pcc-research-01.tzst          # 11G, zstd
~/pcc-research-01.tzst.sha256   # checksum
```

Contains the whole machine directory: `Disk.img`, `SEPStorage`, `nvram.bin`,
`config.plist`, the boot ROMs and the patch records. The IPSW
(`FirmwareOriginals/`) is **excluded** — the target does not need it.

## Making one

The guest must be **stopped**: SEP state and the disk advance together, and a
mismatched pair panics on boot.

```sh
vphone-launchpad-cli vm stop <name>
vphone-launchpad-cli exec vm snapshot create <name> pre-export   # local rollback point
rm -f ~/.vphone/machines/<name>/vphone.sock                      # see below
vphone-launchpad-cli exec vm export <name> -o ~/<name>.tzst
shasum -a 256 ~/<name>.tzst > ~/<name>.tzst.sha256
```

**`vphone.sock` must be deleted first.** A stopped guest leaves its control
socket behind and `vm export` uses gnutar, which cannot archive a socket:

```
Error: Unable to write to <name>/vphone.sock
       (gnutar format cannot archive sockets)
```

The export aborts partway and leaves a **truncated archive that still looks
plausible** — ~7-11G of a 64G stream. Always check the exit status and verify
with `zstd -t` before trusting the file. The socket is recreated by `vm start`.

Snapshots are deliberately not included, so `pre-export` stays on the source
host only.

## Using one

```sh
vphone-launchpad-cli bundle install-release 2.9.0     # must match the source's bundle
shasum -a 256 -c pcc-research-01.tzst.sha256
vphone-launchpad-cli exec vm import ~/pcc-research-01.tzst
vphone-launchpad-cli vm start pcc-research-01 --wait
vphone-launchpad-cli vm leases                        # the guest's new IP
```

Then `ssh root@<new-ip>` — the authorized key is on the guest's disk, so it
works immediately. frida-server comes up from its LaunchDaemon.

## What changes on the new host

- **The IP.** A fresh DHCP lease; it is not necessarily the source's. Read it
  from `vm leases`.
- **Host trust.** `ideviceinstaller` (and so `vphone-install-ipa`) needs the new
  Mac to be a trusted host for the guest's lockdownd; the UDID itself is kept.
- **Bundle version.** The machine is bound to the Core Bundle it was created
  with. Install that version on the target first.
- **Kernel patches** are part of the boot chain in the archive, so the frida
  `kernel-exp-frida_*` patches survive. No `fw set-patches` / `update-kernel`
  needed.

## Do not run both copies at once

`vm import` keeps the ECID, MAC address and UDID, so the copy cannot coexist
with its source on the same network. For two live guests, clone with a new
identity on the source host and export the clone instead:

```sh
vphone-launchpad-cli exec vm clone <name> <name>-b --new-identity
```

That copy gets a new UDID on first boot, which means re-trusting the host and a
new `ideviceinstaller -u` target.
