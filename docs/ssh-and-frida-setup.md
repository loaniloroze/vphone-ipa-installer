# Setting up ssh + frida-server on a vphone guest

How the guest this repo targets (`pcc-research-01`, iOS 27.0.1 / iPhone17,3,
roothide) was given key-based root ssh and a working `frida-server`. Replace the
machine name, IP and jbroot with your own.

```sh
M=pcc-research-01
IP=192.168.64.6                                       # guest NAT address
J=/private/var/containers/Bundle/Application/.jbroot-000114514191980C
```

Find the first two with `vphone-launchpad-cli guest list` / `guest info $M`, and
the jbroot by listing `/private/var/containers/Bundle/Application/` for the
`.jbroot-*` directory. The Mac can reach the guest's NAT address directly, so
**no `iproxy` is needed**.

## 0. Prerequisite: jailbreak the guest

ssh and frida both need the roothide bootstrap. Install it with the **OwnGoal
Bootstrap for vphone** app on the guest (search for it in the guest's app list /
UI and run the bootstrap). That is what provides `/usr/bin/ssh`, `dpkg`, `apt`
and `zsh` inside the jbroot.

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

### Optional: a usable root shell

`guest-zshrc` / `guest-profile` in this repo are the shell setup — copy them to
`/var/root/.zshrc` and `/var/root/.profile`. Root's login shell cannot be
changed (`/etc/master.passwd` is on the sealed system volume), so `.profile`
`exec`s zsh for interactive logins only. See CLAUDE.md for why `bindkey -e` is
mandatory there.

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

# d. ship it and build the deb on the guest (dpkg-deb lives there)
tar -czf payload.tgz -C pkg usr Library DEBIAN
scp -o StrictHostKeyChecking=no payload.tgz root@$IP:/tmp/
```

```sh
ssh root@$IP 'set -e; export PATH=/usr/bin:/usr/sbin:/bin:/sbin:$PATH
rm -rf /tmp/fb && mkdir -p /tmp/fb/pkg && tar -xzf /tmp/payload.tgz -C /tmp/fb/pkg
dpkg-deb -b /tmp/fb/pkg /tmp/fb/frida-roothide.deb
dpkg -i /tmp/fb/frida-roothide.deb
chown root:wheel /usr/sbin/frida-server
launchctl load -w /Library/LaunchDaemons/re.frida.server.plist
sleep 3; launchctl list | grep -i frida'
```

The LaunchDaemon plist is the one real trap: `launchd` is outside the jbroot, so
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
