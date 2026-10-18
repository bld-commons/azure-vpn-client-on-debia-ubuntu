# Azure VPN Client 3.0.0 su Debian / Ubuntu

Installazione del client ufficiale Microsoft **Azure VPN Client 3.0.0** (Linux)
su Debian e Ubuntu, con le modifiche necessarie per far funzionare **Connect**.

Microsoft ha **ritirato il client per Linux il 31/08/2026** (era ufficialmente
supportato solo su Ubuntu 20.04/22.04), ma il pacchetto funziona ancora ed è
attualmente l'unico modo per usare la VPN point-to-site con autenticazione
**OpenVPN + Microsoft Entra ID** (il semplice OpenVPN non sa fare il login
Entra ID).

## Versioni testate / supportate

| Sistema | Stato |
|---|---|
| **Ubuntu 24.04.5 LTS** | **Macchina di origine** (`francesco-home`): client 3.0.0 installato e funzionante senza modifiche — Ubuntu ha `systemd-resolved` attivo di default |
| **Debian 13 (trixie)** | **Configurato e testato** — questa guida è stata scritta e verificata qui, GNOME/Wayland; richiede la correzione di `systemd-resolved` (passaggi 5–6) |
| Ubuntu 22.04 LTS | Supportata dalla documentazione Microsoft (pre-ritiro); stessa generazione di questo pacchetto 3.0.0, attesa funzionante così com'è |
| Ubuntu 20.04 LTS | Supportata da versioni precedenti del client; questo `.deb` 3.0.0 richiede `libssl3` e **non** si installa |
| Debian 12 (bookworm) | Atteso che funzioni (`libssl3` presente); non testato |
| Predecessore di Ubuntu 22.04 / Debian 12 | Non compatibile (mancano `libssl3`, `libstdc++6 >= 12`) |

Dettagli del pacchetto: dipende da `libssl3`, `libstdc++6 (>= 12)`,
`libcurl4`, `libgtk-3-0` → generazione Ubuntu 22.04+; su Ubuntu 24.04 /
Debian 13 le varianti `t64` (`libssl3t64`, `libcurl4t64`, ...) dichiarano
`Provides:` per i vecchi nomi, quindi `dpkg -i` risolve le dipendenze senza
problemi (verificato con `dpkg --audit` su entrambi i sistemi).

## Contenuto di questa cartella

| File | Scopo |
|---|---|
| `microsoft-azurevpnclient_3.0.0_amd64.deb` | Il pacchetto del client |
| `49-azurevpnclient-dns.rules` | Regola polkit per il DNS senza password |
| `README.md` | Questo documento in inglese (versione principale) |

---

## Passaggi

### 1. Recupero del pacchetto (se non ce l'hai)

Sulla macchina che ha già il client installato:

```bash
dpkg-repack microsoft-azurevpnclient
scp microsoft-azurevpnclient_3.0.0_amd64.deb <user>@<questa-macchina>:~/
```

### 2. Installazione

```bash
sudo dpkg -i microsoft-azurevpnclient_3.0.0_amd64.deb
```

La `postinst` del pacchetto esegue automaticamente
`setcap cap_net_admin+eip /opt/microsoft/microsoft-azurevpnclient/microsoft-azurevpnclient`.
È questa capability che permette al client di creare l'interfaccia tun e le
route **senza root**. Verifica con:

```bash
getcap /opt/microsoft/microsoft-azurevpnclient/microsoft-azurevpnclient
# atteso: ... cap_net_admin=eip
```

Su Debian 13 / Ubuntu 24.04 i nomi dei pacchetti dipendenti sono stati
ridenominati nelle varianti `t64` (`libssl3t64`, `libcurl4t64`,
`libgtk-3-0t64`, ...), ma quei pacchetti dichiarano `Provides:` per i vecchi
nomi: le dipendenze si risolvono quindi automaticamente (verificato con
`dpkg --audit` su entrambi i sistemi). Se `dpkg -i` segnalasse comunque un
pacchetto mancante:

```bash
sudo apt -f install
```

### 3. Profilo VPN

Copia il file XML del profilo dalla macchina che lo ha:

```bash
mkdir -p ~/vpn
scp <host-origine>:<percorso>/azurevpnconfig.xml ~/vpn/
```

Poi nel client: **Import** → seleziona il file XML → profilo creato.

### 4. Problema iniziale: Connect fallisce sul DNS

Senza le modifiche successive, Connect fallisce con:

```
Connection failed: Couldn't set DNS server/domains.
```

**Causa**: il client imposta il DNS della VPN chiamando `SetLinkDNS` su
**systemd-resolved** via D-Bus. Su Debian 13 (e su qualsiasi installazione
senza `systemd-resolved`) il DNS lo gestisce solo NetworkManager, quindi la
chiamata fallisce. Ubuntu ha `systemd-resolved` attivo di default (verificato
su Ubuntu 24.04, la macchina di origine), ed è per questo che lì Connect
funziona senza modifiche. **Eseguire il client come root non risolve il
problema**: il servizio D-Bus semplicemente non esiste.

Nota: in precedenza `/etc/resolv.conf` era stato reso immutable
(`chattr +i`) per impedirne la riscrittura; con `systemd-resolved` attivo
non serve più (vedi Note).

### 5. Installazione e configurazione di systemd-resolved

```bash
# 1. rimuovi l'eventuale attributo immutable da /etc/resolv.conf
sudo chattr -i /etc/resolv.conf

# 2. installa e attiva systemd-resolved
sudo apt install systemd-resolved
sudo systemctl enable --now systemd-resolved

# 3. NetworkManager delega il DNS a resolved
sudo mkdir -p /etc/NetworkManager/conf.d
printf '[main]\ndns=systemd-resolved\n' | sudo tee /etc/NetworkManager/conf.d/10-systemd-resolved.conf
sudo systemctl restart NetworkManager

# 4. resolv.conf diventa link fisso allo stub resolver (127.0.0.53)
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

### 6. Regola polkit per il DNS senza password

Senza questa regola, Connect può bloccarsi su un prompt di autenticazione
polkit (il file `.pkla` incluso nel pacchetto non è più letto da polkit
moderno, e la `.rules` inclusa ha un path sbagliato):

```bash
sudo cp 49-azurevpnclient-dns.rules /etc/polkit-1/rules.d/
```

Contenuto della regola (i membri del gruppo `sudo` possono impostare server
e domini DNS via resolved senza password):

```js
polkit.addRule(function(action, subject) {
    if ((action.id == "org.freedesktop.resolve1.set-dns-servers" ||
         action.id == "org.freedesktop.resolve1.set-domains") &&
        subject.isInGroup("sudo")) {
        return polkit.Result.YES;
    }
});
```

### 7. Avvio e verifica

```bash
# avvia il client (o dalla launcher GNOME: "Azure VPN Client")
/opt/microsoft/microsoft-azurevpnclient/microsoft-azurevpnclient
```

Verifiche di sistema:

```bash
ls -la /etc/resolv.conf     # -> /run/systemd/resolve/stub-resolv.conf
resolvectl status           # resolved attivo, DNS upstream visibili
resolvectl query <host-interno-della-vpn>   # test end-to-end DNS via VPN
```

Log del client:

```
~/.config/microsoft-azurevpnclient/logs/AzureVPNClientUI.log
~/.config/microsoft-azurevpnclient/profiles/<nome-profilo>
```

Nel log ci si aspetta `= Connected` e l'assenza di
`Couldn't set DNS server/domains`.

---

## Note

- **Perché non serve root**: la `postinst` applica `cap_net_admin` al
  binario; l'unica operazione privilegiata era la chiamata DNS, gestita
  dalla regola polkit del passaggio 6.
- **Perché non serve più `chattr +i`**: con resolved, `/etc/resolv.conf` è
  un symlink stabile a `127.0.0.53` che nessuno riscrive. Il DNS della VPN
  viene assegnato solo all'interfaccia della VPN (split DNS) e non tocca il
  file globale; il DNS di rete resta gestito da NetworkManager.
- **Client ritirato**: Microsoft ha dismesso il client Linux il 31/08/2026.
  Resta in uso per l'autenticazione OpenVPN + Microsoft Entra ID, che con il
  semplice client OpenVPN non si può fare.
- **Interfaccia VPN**: dopo la connect appare un'interfaccia con il nome del
  profilo (es. `Mase-hub-italyn`) con IP nella range `198.18.x.x` e un DNS
  dedicato.
