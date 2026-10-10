# Setting up ssh + frida-server on a vphone guest

How the guests this repo targets (`pcc-research-01`, iOS 27.0.1, and
`pcc-research-02`, iOS 26.5 — both iPhone17,3 class, roothide) were given
key-based root ssh, a usable zsh and a working `frida-server`. Replace the
machine name, IP and jbroot with your own.

Every step below runs **from the Mac**. Section 0 includes the one part the
vphone documentation presents as a manual on-screen action; it can be driven
over RPC too.

```sh
M=pcc-research-02
IP=192.168.64.11                                      # guest NAT address
J=/private/var/containers/Bundle/Application/.jbroot-000114514191980C
```

Find the first two with `vphone-launchpad-cli vm list` / `vm leases`, and
the jbroot by listing `/private/var/containers/Bundle/Application/` for the
`.jbroot-*` directory. The Mac can reach the guest's NAT address directly, so
**no `iproxy` is needed**.

## 0. Prerequisite: jailbreak the guest

ssh, zsh and frida all need the roothide bootstrap. It arrives in two halves,
and **only the first has an RPC**.

### a. The jbroot and Irisin

```sh
vphone-launchpad-cli guest rpc $M bootstrap.install '{"layout":"roothide"}'
vphone-launchpad-cli guest rpc $M bootstrap.status '{}'      # phase: completed
```

This creates `$J` and installs the Irisin package manager — and nothing else.
`$J/usr/bin` is still empty afterwards; do not mistake `completed` for a usable
environment.

### b. `owngoal-bootstrap-vphone`, from the host

The tools (`apt`, `dpkg`, `bash`/`zsh`/`dash`, `sudo`, coreutils,
**`openssh-server`**, `curl`, `git`, `vim`) come from the single meta-package
`owngoal-bootstrap-vphone`, installed **inside Irisin**. There is no RPC for it
— `packages.compare` is the only package method vphoned exposes — so the app has
to be driven. Do not install those packages one at a time: `openssh-server`
declares some dependencies circularly and separate installs fail partway.

Two facts make this scriptable rather than a trip to the screen:

```sh
# taps take SCREEN PIXELS (1290x2796), not points -- the coordinate_scale: 3
# that screen.screenshot reports is NOT applied to tap coordinates
vphone-launchpad-cli guest send $M '{"t":"tap","x":644,"y":2538}'

# {"t":"type","text":...} is accepted but never lands text in a focused field.
# Set the clipboard, tap the field, then tap the Paste callout.
vphone-launchpad-cli guest rpc $M clipboard.set '{"text":"owngoal-bootstrap"}'
```

A tap's reply already contains a fresh screenshot; `screen.screenshot` gives the
same thing with its dimensions, which is what you measure the next tap against.

The path through the UI, from a freshly bootstrapped guest:

1. **Welcome → Next**
2. **Add Repositories →** `Add` on **OwnGoal Packages** (`https://apt.owngoal.dev`)
   → **Next**
3. **Proceed with Caution → Get Started**
4. search tab → paste `owngoal-bootstrap` → tap the result
5. **INSTALL → Confirm** (queues 89 packages, downloads start on their own)
6. the first download attempt failed on both guests — tap **Retry**, everything
   then reports `Downloaded`
7. **Queue Install →** the alert *"dpkg Is Not Installed"* → **Bootstrap
   Install** (not *Install Anyway*: it places every package's files before any
   maintainer script runs, which is the whole point on a guest with no dpkg)
   → **Execute**

About a minute later the list reads `Installed` and port 22 is open.

## 1. Root ssh key

Chicken-and-egg: you cannot ssh in to install the ssh key. Push it with vphoned
RPCs instead. **RPC paths are real device paths, so they must be prefixed with
the jbroot** — inside an ssh session the same files appear at `/var/root/.ssh`.

```sh
PUB=$(cat ~/.ssh/id_ed25519.pub)

vphone-launchpad-cli guest rpc $M files.mkdir "{\"path\":\"$J/var/root/.ssh\"}"

python3 - "$J" "$PUB" <<'EOF' > /tmp/ak.json
import json, sys
print(json.dumps({"path": sys.argv[1] + "/var/root/.ssh/authorized_keys",
                  "content": sys.argv[2] + "\n"}))
EOF
vphone-launchpad-cli guest rpc $M files.write "$(cat /tmp/ak.json)"

vphone-launchpad-cli guest rpc $M files.chmod "{\"path\":\"$J/var/root/.ssh\",\"mode\":\"700\"}"
vphone-launchpad-cli guest rpc $M files.chmod "{\"path\":\"$J/var/root/.ssh/authorized_keys\",\"mode\":\"600\"}"

ssh -o StrictHostKeyChecking=no root@$IP id        # expect uid=0
```

The `python3` heredoc is there only to JSON-escape the key; building that
argument by hand in the shell is how you get a silently truncated key.

Note that inside an ssh session `/` is the **jbroot**; the real device
filesystem is mounted at **`/rootfs`**. When an RPC and ssh disagree about
whether a file exists, that prefix is the reason.

### A usable root shell

`guest-zshrc` / `guest-profile` in this repo are the shell setup. zsh itself
came with the bootstrap in step 0b; these add the prompt, completion, history
and keybindings, plus two plugins:

```sh
scp guest-zshrc   root@$IP:/var/root/.zshrc
scp guest-profile root@$IP:/var/root/.profile
ssh root@$IP 'export PATH=/usr/bin:/usr/sbin:/bin:/sbin:$PATH; mkdir -p ~/.zsh
git clone --depth 1 https://github.com/zsh-users/zsh-autosuggestions      ~/.zsh/zsh-autosuggestions
git clone --depth 1 https://github.com/zsh-users/zsh-syntax-highlighting  ~/.zsh/zsh-syntax-highlighting'
```

`JBROOT` is hardcoded in `guest-zshrc`; check it still matches `$J` on your
guest (vphone has used the same `.jbroot-000114514191980C` on every machine so
far). Root's login shell cannot be changed (`/etc/master.passwd` is on the
sealed system volume), so `.profile` `exec`s zsh for interactive logins only —
`scp` and `ssh host 'cmd'` keep working on plain sh. See CLAUDE.md for why
`bindkey -e` is mandatory there.

Confirm it took, which a plain `ssh host 'cmd'` cannot show you:

```sh
printf 'echo $0; exit\n' | ssh -tt root@$IP      # /usr/bin/zsh, coloured prompt
```

## 2. frida-server

frida publishes no roothide build: the iOS `.deb` is `iphoneos-arm64` and
installs into `/var/jb`. The guest is **arm64e** with a jbroot at `/`, so the
package has to be repacked.

```sh
S=$PWD/frida && mkdir -p $S && cd $S

# a. grab and unpack frida's iOS deb
curl -sL -o frida.deb \
  https://github.com/frida/frida/releases/download/17.17.0/frida_17.17.0_iphoneos-arm64.deb
mkdir -p x && cd x && ar x ../frida.deb
mkdir -p d c && tar -xf data.tar.* -C d && tar -xf control.tar.* -C c && cd ..

# b. rebuild the tree without the /var/jb prefix
rm -rf pkg && mkdir -p pkg/DEBIAN pkg/usr/sbin pkg/usr/lib pkg/Library/LaunchDaemons
cp -R x/d/var/jb/usr/sbin/frida-server pkg/usr/sbin/
cp -R x/d/var/jb/usr/lib/frida-1.0     pkg/usr/lib/
cp    x/c/control                      pkg/DEBIAN/
cp    x/d/var/jb/Library/LaunchDaemons/re.frida.server.plist \
      pkg/Library/LaunchDaemons/

# c. the daemon's Program must be a REAL path -> jbroot-prefixed
/usr/libexec/PlistBuddy -c "Set :Program $J/usr/sbin/frida-server" \
  pkg/Library/LaunchDaemons/re.frida.server.plist
/usr/libexec/PlistBuddy -c "Set :ProgramArguments:0 $J/usr/sbin/frida-server" \
  pkg/Library/LaunchDaemons/re.frida.server.plist

# d. ship it and build the deb on the guest (dpkg-deb lives there)
tar -czf payload.tgz -C pkg usr Library DEBIAN
scp -o StrictHostKeyChecking=no payload.tgz root@$IP:/tmp/
```

```sh
ssh root@$IP 'set -e; export PATH=/usr/bin:/usr/sbin:/bin:/sbin:$PATH
rm -rf /tmp/fb && mkdir -p /tmp/fb/pkg && tar -xzf /tmp/payload.tgz -C /tmp/fb/pkg
sed -i "s/^Architecture: .*/Architecture: iphoneos-arm64e/" /tmp/fb/pkg/DEBIAN/control
dpkg-deb -b /tmp/fb/pkg /tmp/fb/frida-roothide.deb
dpkg -i /tmp/fb/frida-roothide.deb
chown root:wheel /usr/sbin/frida-server
launchctl load -w /Library/LaunchDaemons/re.frida.server.plist
sleep 3; launchctl list | grep -i frida'
```

The `sed` is not cosmetic. frida's control file says `iphoneos-arm64`, the
guest is `iphoneos-arm64e`, and `dpkg -i` refuses the package outright:
*package architecture (iphoneos-arm64) does not match system
(iphoneos-arm64e)*. The payload is the same either way; only the declaration is
wrong.

The LaunchDaemon plist is the other real trap: `launchd` is outside the jbroot, so
a `Program` of `/usr/sbin/frida-server` resolves against the real root and the
daemon never starts. It must carry the `$J` prefix, while the binary itself is
installed at the jbroot-relative `/usr/sbin/frida-server` that ssh sees.

Check from the Mac:

```sh
frida-ls-devices
frida-ps -U
```

## 3. Kernel patches frida actually needs

A running server is not enough — attaching needs extra CFW patches. The
`standard` preset does not include them:

```sh
vphone-launchpad-cli exec fw set-patches $M --preset standard \
  --allow kernel-exp-frida_thread_set_state_entitlement_flag \
  --allow kernel-exp-frida_vm_map_delete_immutable_code
vphone-launchpad-cli exec fw update-kernel $M     # kernelcache patch, keeps data
```

Without `frida_thread_set_state_entitlement_flag`, attach fails even though
Stalker-only workloads look fine.

## 4. Attach, don't spawn

```sh
vphone-launchpad-cli guest unlock $M
vphone-launchpad-cli guest rpc $M apps.launch '{"bundle_id":"com.apple.Preferences"}'
frida -U -n Preferences -q -e 'console.log("ATTACH OK:", Process.arch, ObjC.available)'
```

`frida -f <bundle-id>` (spawn) still fails on these guests with `0x6000000`.
Launch the app via the `apps.launch` RPC first, then attach by name.

## Summary of the gotchas

| Symptom | Cause |
| --- | --- |
| RPC says a file is missing that ssh can see | RPC needs real paths (`$J/...`), ssh sees the jbroot as `/` |
| frida LaunchDaemon never starts | `Program` not jbroot-prefixed; launchd is outside the jbroot |
| `frida-ps -U` works, attach fails | missing `kernel-exp-frida_thread_set_state_entitlement_flag` |
| `frida -f` fails with `0x6000000` | spawn unsupported — use `apps.launch` + attach |
| truncated `authorized_keys` | hand-escaped JSON; generate it with `python3` |
| `dpkg -i` rejects the repacked frida deb | `DEBIAN/control` still says `iphoneos-arm64`; the guest is `arm64e` |
| taps land in the wrong place | tap coordinates are pixels, not points — ignore `coordinate_scale` |
| `{"t":"type"}` types nothing | use `clipboard.set` + the **Paste** callout |
| `bootstrap.status` says `completed` but `$J/usr/bin` is empty | that RPC only installs Irisin; step 0b is still to come |
