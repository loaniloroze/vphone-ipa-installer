# CLAUDE.md

Guidance for Claude Code (and humans) working in this repo.

## What this is

`bin/vphone-install-ipa` installs a **decrypted** `.ipa` onto a running
[vphone](https://github.com/Lakr233/vphone-cli) guest — a virtual iPhone on an
Apple Silicon Mac — and leaves the app running. One command, no interaction.

```sh
vphone-install-ipa ~/Downloads/com.burbn.instagram_450.1.0.ipa
```

`guest-zshrc` / `guest-profile` are the shell setup for `ssh root@<guest>`.

## Why each step exists

Nothing here is incidental; every stage works around a specific wall. Read this
before changing the script.

### 1. The App Store is not an option

Apple ID sign-in and the App Store do not work on vphone guests and are not
planned — see [vphone-cli#14](https://github.com/Lakr233/vphone-cli/issues/14).
Apple rejects the VM's device identity (guests report a synthetic serial such
as `vphone-1337`). The Store shows "Cannot Connect"; sign-in fails with a
verification error.

Consequence: there is **no FairPlay licence** on the guest, so an App Store IPA
(e.g. from `ipatool`) installs fine and then refuses to launch. The input must
already be decrypted.

### 2. `cryptid` must be 0

The script refuses to continue otherwise. `LC_ENCRYPTION_INFO_64.cryptid != 0`
means the main binary is still FairPlay-encrypted. Failing loudly here is much
better than a successful install that silently never launches.

### 3. Re-signing ad-hoc is mandatory

A code signature hashes the binary's pages. **Decryption rewrites those pages**,
so the original App Store signature no longer matches its own content. Leaving
it in place fails at `VerifyingApplication (40%)` with `0xE8008019`
("does not have a valid signature") — documented in
[vphone-cli#547](https://github.com/Lakr233/vphone-cli/issues/547).

Stripping the signature is not an option either: `installd` will not install an
unsigned bundle.

The guest's CFW kernel patches (`kernel-boot-amfi_*`, `txm-boot-trustcache_bypass`)
relax *who* must have signed — a **well-formed ad-hoc signature is accepted**,
`flags=0x2(adhoc)`, `TeamIdentifier=not set`. That is the one state that works,
and why no Apple developer account is needed. **A stock iPhone still rejects it.**

### 4. Deepest-first, and enumerate — never hardcode directories

Each nested bundle carries its own signature, and a parent's signature includes
hashes of its children. Sign a parent first and re-signing a child underneath
invalidates the parent. So: sort candidate paths by depth descending, sign
children, then the `.app` last.

**The trap that cost real time:** Instagram keeps extensions in **both**
`PlugIns/` *and* `Extensions/`. Signing only `PlugIns/` produced
`0xE8008019` on `InstagramExtensionLockScreenCamera.appex`. The script now
`find`s every `.appex`, `.framework`, `.bundle` and `.dylib` anywhere in the
app. Do not reintroduce a fixed directory list.

### 5. Entitlements must be carried across

Ad-hoc signing **drops entitlements** unless they are passed back explicitly.
Without them the app loses keychain access, app groups and extension
permissions. Each bundle's own set is read with
`codesign -d --entitlements :- --xml` and fed to `codesign --entitlements`.

Resource-only bundles (no executable) often cannot be signed; those failures are
counted and reported, not fatal.

### 6. `ideviceinstaller`, not `apps.install`, not `devicectl`

| Route | Verdict |
| --- | --- |
| `ideviceinstaller -u <UDID> install` | **Use this.** Goes through `installd`, the real path. App extensions get registered and launch. |
| vphoned `apps.install` RPC | Simpler and re-signs for you, but bypasses `installd`; its app extensions never launch ([#546](https://github.com/Lakr233/vphone-cli/issues/546)). |
| `xcrun devicectl` | **Never.** Times out against these guests. |

## Shell quirks the script works around

- `set -euo pipefail` + a helper whose last command may fail will abort the run
  *inside the function*, before any `return 0`. Every `codesign` call therefore
  ends in `|| SKIPPED+=(...)`.
- `ideviceinstaller` redraws progress with `\r` when stdout is not a tty, so the
  whole run lands on one line. The log is passed through `tr '\r' '\n'` before
  grepping.
- Build the array with a `while read` fed by process substitution, not a
  pipeline — a piped `while` runs in a subshell and loses the array.

## Guest shell files

`guest-zshrc` → `/var/root/.zshrc`, `guest-profile` → `/var/root/.profile`.

Root's login shell **cannot** be changed: `/etc/master.passwd` lives on the
sealed read-only system volume, so it stays `/bin/sh` (which is why the default
prompt shows a literal `\h:\w \u$` — dash does not expand bash escapes).
`.profile` therefore `exec`s zsh for *interactive* logins only, leaving `scp`
and `ssh host 'cmd'` on plain sh so they keep working.

`bindkey -e` is required and must be explicit: zsh picks its keymap from
`$EDITOR`, and `EDITOR=vim` silently selects **vi** mode, where Ctrl-A/Ctrl-E
are unbound and leak through as control characters.

On roothide, an ssh session's `/` is the jbroot; the real device filesystem is
at **`/rootfs`**. vphoned RPCs use real paths. When ssh and RPC disagree about
whether a file exists, that prefix is why.

## Requirements

- macOS on Apple Silicon, a vphone guest in `running` state
- `vphone-launchpad-cli` on `PATH`
- `ideviceinstaller` (`brew install ideviceinstaller`)

## Conventions

- zsh, `set -euo pipefail`, no external deps beyond the two tools above.
- Fail early and explain *why* in the error, not just what.
- Keep the step banners; they are how a user locates a failure.
- When adding a workaround, record the symptom it fixes here too.
