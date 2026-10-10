# What an offline Mac needs

Everything required to run `pcc-research-02` — iOS 26.5 with root ssh, zsh and
frida-server — on a Mac with **no internet connection**. Nothing here contacts
GitHub, `apt.owngoal.dev` or Apple.

The guest itself needs nothing: the roothide bootstrap, `openssh-server`, the
authorized key, the zsh setup and frida-server are all on `Disk.img`, and the
`kernel-exp-frida_*` patches are part of the boot chain inside the archive.
Everything below is **host** side.

## 1. The host itself

- A **physical** Apple Silicon Mac, macOS 15 or newer. PCC research guests do
  not run inside a nested macOS VM.
- **~75 GB free.** The archive is 9.7 GB but expands to a 64 GB `Disk.img`;
  both exist at once during the import.
- **Relaxed security settings, set in macOS Recovery.** This is the one step
  that cannot be carried on a USB stick, and it needs two reboots. Either path
  works:

  | | Recovery | After rebooting |
  | --- | --- | --- |
  | **A** — SIP off | `csrutil disable`<br>`csrutil allow-research-guests enable` | `sudo nvram boot-args="amfi_get_out_of_my_way=1 -v"`, reboot again |
  | **B** — SIP on, debugging relaxed | `csrutil enable --without debug`<br>`csrutil allow-research-guests enable` | allowlist `vphone-vm`; Launchpad's helper does it, or `sudo .../vphone-escalator allow .../vphone-vm` |

  `csrutil allow-research-guests enable` is required either way. This Mac runs
  path A (`csrutil status` → disabled, `boot-args` → `amfi_get_out_of_my_way=1`).
  Under path B the allowlist is per **bundle signature**, so it has to be
  redone whenever the Core Bundle changes.

## 2. Files to carry

Exactly two things are required:

| File | Size | Why |
| --- | --- | --- |
| `pcc-research-02.tzst` | 9.7 G | **the snapshot** — the whole machine |
| `VPhone.bundle` 2.9.2 | 86 M (16 M zipped) | the Core Bundle the machine is bound to, and the only way to get it without a network |

Useful but optional:

| File | Size | Why it is optional |
| --- | --- | --- |
| `vphone-launchpad.app` | 21 M | convenience. It supplies the GUI, the `vphone-launchpad-cli` wrapper and the privileged helper, but the bundle's own `VPhone.bundle/Contents/MacOS/vphone-cli` can `vm import` and `vm launch` on its own |
| `pcc-research-02.tzst.sha256` | 101 B | one line of text. `vm export` can leave a truncated archive that still looks plausible, so it is worth copying |
| `vphone-bundle-2.9.2.zip.sha256` | 101 B | same |

**Not needed:** `~/.vphone/gpu-drivers/`. An earlier revision of this document
listed it. Tested by moving the cache aside and restarting `pcc-research-02`:
the guest boots, SpringBoard renders normally and ssh and frida-server come up.
It is populated when a machine is *created* from a cloudOS IPSW, which an
imported machine never does.

Make the app and bundle copies on a Mac that already works:

```sh
ditto -c -k --sequesterRsrc --keepParent /Applications/vphone-launchpad.app ~/vphone-launchpad.zip
ditto -c -k --sequesterRsrc --keepParent \
  "/Library/Application Support/vphone-launchpad/Bundles/2.9.2/VPhone.bundle" \
  ~/vphone-bundle-2.9.2.zip
```

Carry the Launchpad app as the **notarized** release zip if you have it;
notarization is stapled, so Gatekeeper accepts it offline.

## 3. Host tools that are *not* required

- **No Homebrew, Python or Xcode.** A distributed `VPhone.bundle` has no
  runtime dependencies.
- `ssh` and `scp` ship with macOS, so the guest is reachable out of the box.
- `frida` / `frida-ps` on the host are only needed to *drive* frida from the
  Mac; the server inside the guest runs regardless. They install from PyPI, so
  carry a wheel if you want them offline.
- `ideviceinstaller` (Homebrew) is only needed for `vphone-install-ipa`.
  Carrying a bottle is the only offline route.

## 4. The procedure on the target

```sh
# a. Launchpad
ditto -xk vphone-launchpad.zip /Applications/
mkdir -p ~/.local/bin
ln -sf /Applications/vphone-launchpad.app/Contents/MacOS/vphone-launchpad-cli ~/.local/bin/
open /Applications/vphone-launchpad.app      # installs the privileged helper -- asks for an admin password

# b. the Core Bundle, copied into place (see the warning below)
sudo mkdir -p "/Library/Application Support/vphone-launchpad/Bundles/2.9.2"
sudo ditto -xk vphone-bundle-2.9.2.zip \
  "/Library/Application Support/vphone-launchpad/Bundles/2.9.2/"
vphone-launchpad-cli bundle verify 2.9.2
vphone-launchpad-cli bundle list             # expect policy/preflight: passed

# c. the machine
shasum -a 256 -c pcc-research-02.tzst.sha256
vphone-launchpad-cli exec vm import ~/pcc-research-02.tzst
vphone-launchpad-cli vm start pcc-research-02 --wait
vphone-launchpad-cli vm leases               # the guest's new IP
```

**Do not use `bundle install-local` for step b.** It deliberately renames what
it installs to `2.9.2-local.<hash>`, which no longer matches the `2.9.2` the
imported machine is bound to. If you already did, rebind rather than rename
anything:

```sh
vphone-launchpad-cli vm set-bundle pcc-research-02 2.9.2-local.<hash> --update-environment
```

Note that the Launchpad **app** version and the Core Bundle version are
independent — this Mac runs app 2.9.0 against bundle 2.9.2.

## 5. Checking it worked

```sh
IP=$(vphone-launchpad-cli vm leases | python3 -c \
  'import json,sys;print([l["address"] for l in json.load(sys.stdin) if l["machine"]=="pcc-research-02"][0])')

ssh root@$IP 'launchctl list | grep -iE "frida|sshd"'   # re.frida.server + com.openssh.sshd
printf 'echo $0; exit\n' | ssh -tt root@$IP             # /usr/bin/zsh, coloured prompt
frida-ps -U                                             # host-side, optional
```

Both daemons come back on their own after a reboot; no re-registration is
needed.

## 6. Two things that will bite

- **Do not run the copy and the original at the same time on one network.**
  `vm import` keeps the ECID, MAC address and UDID. For two live guests, clone
  with a new identity on the source host and export the clone:
  `vphone-launchpad-cli exec vm clone <name> <name>-b --new-identity`.
- **`ideviceinstaller` needs the new Mac trusted** by the guest's lockdownd
  before `vphone-install-ipa` works there. The UDID is preserved, the host
  trust relationship is not.

See [importing-a-ready-guest.md](importing-a-ready-guest.md) for how the
archive is made, and [ssh-and-frida-setup.md](ssh-and-frida-setup.md) for how
the guest was built in the first place.
