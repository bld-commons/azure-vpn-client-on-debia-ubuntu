# Azure VPN Client 3.0.0 on Debian / Ubuntu

Installation of the official Microsoft **Azure VPN Client 3.0.0** (Linux) on
Debian and Ubuntu, with the fixes needed to make **Connect** work.

Microsoft **retired the Linux client on 2026-08-31** (it was officially
supported only on Ubuntu 20.04/22.04), but the package still works and is
currently the only way to use point-to-site VPN with **OpenVPN + Microsoft
Entra ID** authentication (plain OpenVPN cannot do the Entra ID login).

## Tested / supported versions

| OS | Status |
|---|---|
| **Ubuntu 24.04.5 LTS** | **Source machine** (`francesco-home`): client 3.0.0 installed and working out of the box — Ubuntu ships `systemd-resolved` active by default |
| **Debian 13 (trixie)** | **Configured and tested** — this guide was written and verified here, GNOME/Wayland; needs the `systemd-resolved` fix (steps 5–6) |
| Ubuntu 22.04 LTS | Officially supported by Microsoft documentation (pre-retirement); same generation as this 3.0.0 package, expected to work as-is |
| Ubuntu 20.04 LTS | Officially supported by older client versions; this 3.0.0 `.deb` needs `libssl3` and will **not** install |
| Debian 12 (bookworm) | Expected to work (`libssl3` present); not tested |
| Older than Ubuntu 22.04 / Debian 12 | Not compatible (missing `libssl3`, `libstdc++6 >= 12`) |

Package details: depends on `libssl3`, `libstdc++6 (>= 12)`, `libcurl4`,
`libgtk-3-0` → Ubuntu 22.04+ generation; on Ubuntu 24.04 / Debian 13 the
`t64` variants (`libssl3t64`, `libcurl4t64`, ...) `Provides:` the old names,
so `dpkg -i` resolves dependencies cleanly (verified with `dpkg --audit` on
both systems).

## Contents of this folder

| File | Purpose |
|---|---|
| `microsoft-azurevpnclient_3.0.0_amd64.deb` | The client package |
| `49-azurevpnclient-dns.rules` | polkit rule for passwordless DNS |
| `README.it.md` | This document in Italian |

---

## Steps

### 1. Package

The installer is `microsoft-azurevpnclient_3.0.0_amd64.deb`, included in
this folder.

### 2. Installation

```bash
sudo dpkg -i microsoft-azurevpnclient_3.0.0_amd64.deb
```

The package `postinst` automatically runs
`setcap cap_net_admin+eip /opt/microsoft/microsoft-azurevpnclient/microsoft-azurevpnclient`.
This capability is what lets the client create the tun interface and routes
**without root**. Verify with:

```bash
getcap /opt/microsoft/microsoft-azurevpnclient/microsoft-azurevpnclient
# expected: ... cap_net_admin=eip
```

On Debian 13 / Ubuntu 24.04 the dependency names were renamed to the `t64`
variants (`libssl3t64`, `libcurl4t64`, `libgtk-3-0t64`, ...), but those
packages `Provides:` the old names, so the dependencies resolve automatically
(`dpkg --audit` is clean). If `dpkg -i` still reports something missing:

```bash
sudo apt -f install
```

### 3. VPN profile

Copy the profile XML from the machine that has it:

```bash
mkdir -p ~/vpn
scp <source-host>:<path>/azurevpnconfig.xml ~/vpn/
```

Then in the client: **Import** → select the XML file → profile created.

### 4. Initial problem: Connect fails on DNS

Without the changes below, Connect fails with:

```
Connection failed: Couldn't set DNS server/domains.
```

**Cause**: the client sets the VPN DNS by calling `SetLinkDNS` on
**systemd-resolved** over D-Bus. On Debian 13 (and any install without
`systemd-resolved`) only NetworkManager handles DNS, so the call fails.
Ubuntu ships `systemd-resolved` active by default (verified on Ubuntu 24.04,
the source machine), which is why Connect works there without changes.
**Running the client as root does not fix this** — the D-Bus service simply
does not exist.

Note: previously `/etc/resolv.conf` had been made immutable (`chattr +i`) to
stop it being overwritten; with `systemd-resolved` in place this is no longer
needed (see Notes).

### 5. Install and configure systemd-resolved

```bash
# 1. remove the immutable attribute from /etc/resolv.conf if present
sudo chattr -i /etc/resolv.conf

# 2. install and enable systemd-resolved
sudo apt install systemd-resolved
sudo systemctl enable --now systemd-resolved

# 3. NetworkManager delegates DNS to resolved
sudo mkdir -p /etc/NetworkManager/conf.d
printf '[main]\ndns=systemd-resolved\n' | sudo tee /etc/NetworkManager/conf.d/10-systemd-resolved.conf
sudo systemctl restart NetworkManager

# 4. /etc/resolv.conf becomes a fixed link to the stub resolver (127.0.0.53)
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

**Reboot the machine at this point** (`sudo reboot`). Restarting
NetworkManager with the new DNS configuration can drop the connection (we
lost internet until reboot), and after a reboot the whole DNS stack comes up
clean.

### 6. polkit rule for passwordless DNS

Without it, Connect can hang on a polkit authentication prompt (the `.pkla`
file shipped in the package is no longer read by modern polkit, and the
bundled `.rules` file matches the wrong path):

```bash
sudo cp 49-azurevpnclient-dns.rules /etc/polkit-1/rules.d/
```

Rule content (members of group `sudo` may set DNS servers/domains via
resolved without a password):

```js
polkit.addRule(function(action, subject) {
    if ((action.id == "org.freedesktop.resolve1.set-dns-servers" ||
         action.id == "org.freedesktop.resolve1.set-domains") &&
        subject.isInGroup("sudo")) {
        return polkit.Result.YES;
    }
});
```

### 7. Start and verify

After the reboot, launch the client:

```bash
# launch the client (or from the GNOME launcher: "Azure VPN Client")
/opt/microsoft/microsoft-azurevpnclient/microsoft-azurevpnclient
```

System checks:

```bash
ls -la /etc/resolv.conf     # -> /run/systemd/resolve/stub-resolv.conf
resolvectl status           # resolved active, uplink DNS visible
resolvectl query <internal-vpn-host>   # end-to-end DNS test via the VPN
```

Client logs:

```
~/.config/microsoft-azurevpnclient/logs/AzureVPNClientUI.log
~/.config/microsoft-azurevpnclient/profiles/<profile-name>
```

Expected in the log: `= Connected` and no
`Couldn't set DNS server/domains`.

---

## Notes

- **Why root is not needed**: the `postinst` applies `cap_net_admin` to the
  binary; the only privileged operation was the DNS call, handled by the
  polkit rule in step 6.
- **Why `chattr +i` is no longer needed**: with resolved, `/etc/resolv.conf`
  is a stable symlink to `127.0.0.53` that nothing rewrites. VPN DNS is
  assigned only to the VPN interface (split DNS) and never touches the
  global file; regular network DNS stays under NetworkManager.
- **Retired client**: Microsoft discontinued the Linux client on 2026-08-31.
  It remains in use for OpenVPN + Microsoft Entra ID authentication, which
  the plain OpenVPN client cannot perform.
- **VPN interface**: after Connect an interface named after the profile
  (e.g. `Mase-hub-italyn`) appears with an IP in the `198.18.x.x` range and
  its own DNS server.
