# vphone-install-ipa

Install a **decrypted** `.ipa` onto a running [vphone](https://github.com/Lakr233/vphone-cli)
guest and leave the app running. One command.

```sh
vphone-install-ipa ~/Downloads/com.burbn.instagram_450.1.0.ipa
```

```
==> Finding a running guest
  machine pcc-research-01   udid 0000FE01-AF8B42DBEB26FF2B
==> Unpacking
  Instagram.app  com.burbn.instagram  450.1.0
==> Checking the binary is decrypted
  cryptid 0 — decrypted
==> Re-signing ad-hoc
  23 nested bundles
  flags=0x2(adhoc), deep verify ok
==> Repacking
  305M
==> Installing through installd
  Install: Complete
==> Launching

✓ com.burbn.instagram 450.1.0 is installed and running on pcc-research-01
```

## Install

```sh
brew install ideviceinstaller
git clone https://github.com/<you>/vphone-ipa-installer
ln -sf "$PWD/vphone-ipa-installer/bin/vphone-install-ipa" ~/.local/bin/
```

Needs `vphone-launchpad-cli` on `PATH` and a guest in `running` state.
With one running guest the machine is auto-detected; otherwise pass it:

```sh
vphone-install-ipa app.ipa my-machine
```

## The IPA has to be decrypted

The App Store and Apple ID **do not work** on vphone guests and are not planned
([vphone-cli#14](https://github.com/Lakr233/vphone-cli/issues/14)). With no
Apple ID there is no FairPlay licence, so an App Store download — including one
from `ipatool` — installs and then refuses to launch.

The script checks this up front and refuses an encrypted binary rather than
leaving you with an app that silently never starts.

Get a decrypted IPA by dumping it from a real jailbroken device with
`frida-ios-dump`, or from one of the decrypted-IPA sites (you are trusting a
stranger's binary — fine for a throwaway VM, not for a real phone).

## What it does

1. Picks the running guest and its UDID.
2. Refuses the IPA unless `cryptid` is 0.
3. Re-signs every nested bundle **ad-hoc, deepest first**, carrying each one's
   own entitlements across — then the app last.
4. Repacks and installs via `ideviceinstaller` (through `installd`, so app
   extensions actually launch).
5. Launches the app and confirms it reached the foreground, with a screenshot.

## Why it works without paying Apple

Ad-hoc signing (`codesign -s -`) produces a structurally valid signature that
chains to nothing. A stock iPhone rejects that. The vphone guest's patched
kernel relaxes *who* must have signed, so a well-formed ad-hoc signature is
enough — no $99 developer account.

**The same IPA will not install on a real iPhone.** That needs a developer
certificate, a free-Apple-ID sideload (7-day expiry), or a jailbreak.

## Guest shell

`guest-zshrc` and `guest-profile` give `ssh root@<guest>` a proper zsh: a
two-line git-aware prompt, autosuggestions, syntax highlighting, completion and
emacs keybindings.

```sh
ssh root@<guest-ip> 'mkdir -p ~/.zsh && cd ~/.zsh &&
  git clone -q --depth=1 https://github.com/zsh-users/zsh-autosuggestions &&
  git clone -q --depth=1 https://github.com/zsh-users/zsh-syntax-highlighting'
scp guest-zshrc   root@<guest-ip>:/var/root/.zshrc
scp guest-profile root@<guest-ip>:/var/root/.profile
```

Root's login shell can't be changed (`/etc/master.passwd` is on the sealed
read-only volume), so `.profile` execs zsh for interactive logins only.

See [CLAUDE.md](CLAUDE.md) for why each step exists and the failure each one
prevents.
