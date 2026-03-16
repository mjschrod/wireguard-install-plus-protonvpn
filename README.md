# WireGuard installer

![Lint](https://github.com/angristan/wireguard-install/workflows/Lint/badge.svg)
[![Say Thanks!](https://img.shields.io/badge/Say%20Thanks-!-1EAEDB.svg)](https://saythanks.io/to/angristan)

**This project is a bash script that aims to setup a [WireGuard](https://www.wireguard.com/) VPN on a Linux server, as easily as possible!**

WireGuard is a point-to-point VPN that can be used in different ways. Here, we mean a VPN as in: the client will forward all its traffic through an encrypted tunnel to the server.
The server will apply NAT to the client's traffic so it will appear as if the client is browsing the web with the server's IP.

The script supports both IPv4 and IPv6. Please check the [issues](https://github.com/angristan/wireguard-install/issues) for ongoing development, bugs and planned features! You might also want to check the [discussions](https://github.com/angristan/wireguard-install/discussions) for help.

WireGuard does not fit your environment? Check out [openvpn-install](https://github.com/angristan/openvpn-install).

## Requirements

Supported distributions:

- AlmaLinux >= 8
- Alpine Linux
- Arch Linux
- CentOS Stream >= 8
- Debian >= 10
- Fedora >= 32
- Oracle Linux
- Rocky Linux >= 8
- Ubuntu >= 18.04

## Usage

Download and execute the script. Answer the questions asked by the script and it will take care of the rest.

```bash
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
chmod +x wireguard-install.sh
./wireguard-install.sh
```

It will install WireGuard (kernel module and tools) on the server, configure it, create a systemd service and a client configuration file.

Run the script again to add or remove clients!

## Providers

I recommend these cheap cloud providers for your VPN server:

- [Vultr](https://www.vultr.com/?ref=8948982-8H): Worldwide locations, IPv6 support, starting at \$5/month
- [Hetzner](https://hetzner.cloud/?ref=ywtlvZsjgeDq): Germany, Finland and USA. IPv6, 20 TB of traffic, starting at 4.5€/month
- [Digital Ocean](https://m.do.co/c/ed0ba143fe53): Worldwide locations, IPv6 support, starting at \$4/month

## Contributing

Contributions are welcome! Here's how you can help:

### Discuss changes

Please open an issue before submitting a PR if you want to discuss a change, especially if it's a big one.

### Code formatting

We use [shellcheck](https://github.com/koalaman/shellcheck) and [shfmt](https://github.com/mvdan/sh) to enforce bash styling guidelines and good practices. They are executed for each commit / PR with GitHub Actions, so you can check the configuration [here](https://github.com/angristan/wireguard-install/blob/master/.github/workflows/lint.yml).

## Say thanks

You can [say thanks](https://saythanks.io/to/angristan) if you want!

## Credits & Licence

This project is under the [MIT Licence](https://raw.githubusercontent.com/angristan/wireguard-install/master/LICENSE)

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=angristan/wireguard-install&type=Date)](https://star-history.com/#angristan/wireguard-install&Date)


XXXXXXXXXXXXXXXXXXXXXXXXXXXXX
NEW BY FORK
XXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# WireGuard-Remotezugang mit ProtonVPN-Uplink und WAN-Bypass für Heimdienste

## Ziel

Dieses Setup beschreibt einen Debian-LXC, der gleichzeitig:

- **WireGuard-Server** für den externen Zugang ist,
- **ProtonVPN per WireGuard** als Uplink nutzt,
- **normalen Internetverkehr** von Remote-Clients über **ProtonVPN** leitet,
- aber **Zugriffe auf das eigene Heimnetz** sowie auf die **öffentliche WAN-IP des Heimanschlusses** **nicht** über ProtonVPN schickt.

Damit bleibt zum Beispiel folgender Zugriff auf dem **normalen Heimanschluss**:

- `https://dienst.schlumpf.gleeze.com:44385`

während sonstiger Internetverkehr des Android-Clients über **ProtonVPN** läuft.

---

## Architektur

### Komponenten

- **Debian-LXC** im Heimnetz
  - `wg0`: eingehender WireGuard-Server für externe Clients
  - `proton0`: ausgehender WireGuard-Tunnel zu ProtonVPN
- **Router** mit Portweiterleitung für `wg0`
- **Android** mit WireGuard-App als Client

### Routing-Ziel

Der externe Client schickt den Verkehr zunächst über `wg0` an den LXC. Der LXC entscheidet dann:

- Ziel `192.168.0.0/22` → **lokal ins Heimnetz**
- Ziel **aktuelle WAN-IP** des Heimanschlusses → **lokal über den Router**
- alles andere → **über `proton0` zu ProtonVPN**

### Wichtige Voraussetzung

Dieses Setup setzt voraus, dass der Router **Hairpin NAT / NAT Loopback** für Zugriffe auf die **eigene WAN-IP** aus dem LAN sauber beherrscht.

---

## Beispielwerte

Diese Anleitung nutzt folgende Beispielwerte:

- WireGuard-Servernetz: `10.7.0.0/24`
- WireGuard-Server-IP: `10.7.0.1/24`
- Android-Client-IP: `10.7.0.2/32`
- Heimnetz: `192.168.0.0/22`
- LXC-LAN-Interface: `eth0`
- DynDNS-Name für die WAN-IP: `wg.schlumpf.gleeze.com`
- Beispiel-Dienst: `dienst.schlumpf.gleeze.com:44385`
- Policy-Routing-Tabelle für ProtonVPN: `proton`

---

## Voraussetzungen

### 1. ProtonVPN-WireGuard ist vorhanden

`proton0` muss bereits funktionieren.

Beispielprüfung:

```bash
wg show proton0
ip addr show proton0
```

### 2. IP-Forwarding ist aktiv

```bash
sysctl net.ipv4.ip_forward
```

Erwartet:

```text
net.ipv4.ip_forward = 1
```

Falls nicht:

```bash
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-ipforward.conf
sysctl --system
```

### 3. Routing-Tabelle `proton` existiert

```bash
grep -q '^200 proton$' /etc/iproute2/rt_tables || echo '200 proton' >> /etc/iproute2/rt_tables
```

---

## Grundidee des Setups

Es gibt drei Bausteine:

1. `wg-policy-up.sh`
   - setzt die festen Routing- und Firewall-Regeln beim Start von `wg0`
2. `wg-policy-down.sh`
   - entfernt diese Regeln beim Stoppen von `wg0`
3. `update-wan-bypass.sh`
   - löst regelmäßig `wg.schlumpf.gleeze.com` auf
   - pflegt daraus die aktuelle `/32`-Bypass-Regel für die WAN-IP

Dadurch ist die WAN-IP **nicht hart im Setup kodiert**.

---

## Skript 1: `/usr/local/sbin/wg-policy-up.sh`

Dieses Skript setzt die festen Regeln für:

- Heimnetz lokal
- Standardverkehr über ProtonVPN
- NAT und Forwarding
- und ruft am Ende den WAN-Updater auf

```bash
#!/usr/bin/env bash
set -euo pipefail

WG_NET="10.7.0.0/24"
LAN_NET="192.168.0.0/22"
LAN_IF="eth0"
PROTON_TABLE="proton"
WAN_UPDATER="/usr/local/sbin/update-wan-bypass.sh"

# proton0 muss existieren
ip link show proton0 >/dev/null 2>&1

# Proton-Default-Route in eigene Tabelle
ip route replace default dev proton0 table "${PROTON_TABLE}"

# Feste Policy-Regeln sauber neu setzen
ip rule del from "${WG_NET}" to "${LAN_NET}" lookup main priority 100 2>/dev/null || true
ip rule del from "${WG_NET}" lookup "${PROTON_TABLE}" priority 200 2>/dev/null || true

ip rule add from "${WG_NET}" to "${LAN_NET}" lookup main priority 100
ip rule add from "${WG_NET}" lookup "${PROTON_TABLE}" priority 200

# NAT für LAN
iptables -t nat -C POSTROUTING -s "${WG_NET}" -d "${LAN_NET}" -o "${LAN_IF}" -j MASQUERADE >/dev/null 2>&1 || \
iptables -t nat -A POSTROUTING -s "${WG_NET}" -d "${LAN_NET}" -o "${LAN_IF}" -j MASQUERADE

# NAT für Proton
iptables -t nat -C POSTROUTING -s "${WG_NET}" -o proton0 -j MASQUERADE >/dev/null 2>&1 || \
iptables -t nat -A POSTROUTING -s "${WG_NET}" -o proton0 -j MASQUERADE

# Forwarding WG -> Proton
iptables -C FORWARD -i wg0 -o proton0 -j ACCEPT >/dev/null 2>&1 || \
iptables -A FORWARD -i wg0 -o proton0 -j ACCEPT

iptables -C FORWARD -i proton0 -o wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT >/dev/null 2>&1 || \
iptables -A FORWARD -i proton0 -o wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT

# Forwarding WG -> LAN
iptables -C FORWARD -i wg0 -o "${LAN_IF}" -d "${LAN_NET}" -j ACCEPT >/dev/null 2>&1 || \
iptables -A FORWARD -i wg0 -o "${LAN_IF}" -d "${LAN_NET}" -j ACCEPT

iptables -C FORWARD -i "${LAN_IF}" -o wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT >/dev/null 2>&1 || \
iptables -A FORWARD -i "${LAN_IF}" -o wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT

# Dynamische WAN-Bypass-Regel setzen/aktualisieren
"${WAN_UPDATER}"
```

Danach ausführbar machen:

```bash
chmod +x /usr/local/sbin/wg-policy-up.sh
```

---

## Skript 2: `/usr/local/sbin/wg-policy-down.sh`

Dieses Skript entfernt beim Stoppen von `wg0` alle gesetzten Regeln wieder.

```bash
#!/usr/bin/env bash
set -euo pipefail

WG_NET="10.7.0.0/24"
LAN_NET="192.168.0.0/22"
LAN_IF="eth0"
PROTON_TABLE="proton"
STATE_FILE="/var/lib/wg-wan-bypass/current_wan_ip"

# Dynamische WAN-Regel entfernen
if [ -f "${STATE_FILE}" ]; then
  WAN_IP="$(cat "${STATE_FILE}")"

  ip rule del from "${WG_NET}" to "${WAN_IP}/32" lookup main priority 110 2>/dev/null || true
  iptables -t nat -D POSTROUTING -s "${WG_NET}" -d "${WAN_IP}/32" -o "${LAN_IF}" -j MASQUERADE 2>/dev/null || true
  iptables -D FORWARD -i wg0 -o "${LAN_IF}" -d "${WAN_IP}/32" -j ACCEPT 2>/dev/null || true
fi

# Feste Policy-Regeln entfernen
ip rule del from "${WG_NET}" to "${LAN_NET}" lookup main priority 100 2>/dev/null || true
ip rule del from "${WG_NET}" lookup "${PROTON_TABLE}" priority 200 2>/dev/null || true
ip route del default dev proton0 table "${PROTON_TABLE}" 2>/dev/null || true

# NAT entfernen
iptables -t nat -D POSTROUTING -s "${WG_NET}" -d "${LAN_NET}" -o "${LAN_IF}" -j MASQUERADE 2>/dev/null || true
iptables -t nat -D POSTROUTING -s "${WG_NET}" -o proton0 -j MASQUERADE 2>/dev/null || true

# Forwarding entfernen
iptables -D FORWARD -i wg0 -o proton0 -j ACCEPT 2>/dev/null || true
iptables -D FORWARD -i proton0 -o wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT 2>/dev/null || true
iptables -D FORWARD -i wg0 -o "${LAN_IF}" -d "${LAN_NET}" -j ACCEPT 2>/dev/null || true
iptables -D FORWARD -i "${LAN_IF}" -o wg0 -m state --state ESTABLISHED,RELATED -j ACCEPT 2>/dev/null || true
```

Danach ausführbar machen:

```bash
chmod +x /usr/local/sbin/wg-policy-down.sh
```

---

## Skript 3: `/usr/local/sbin/update-wan-bypass.sh`

Dieses Skript löst den DynDNS-Namen auf und pflegt die aktuelle `/32`-Bypass-Regel.

```bash
#!/usr/bin/env bash
set -euo pipefail

WG_NET="10.7.0.0/24"
LAN_IF="eth0"
WAN_HOST="wg.schlumpf.gleeze.com"
STATE_DIR="/var/lib/wg-wan-bypass"
STATE_FILE="${STATE_DIR}/current_wan_ip"

mkdir -p "${STATE_DIR}"

NEW_IP="$(getent ahostsv4 "${WAN_HOST}" | awk '{print $1; exit}')"

if [ -z "${NEW_IP}" ]; then
  echo "Konnte ${WAN_HOST} nicht auflösen" >&2
  exit 1
fi

OLD_IP=""
if [ -f "${STATE_FILE}" ]; then
  OLD_IP="$(cat "${STATE_FILE}")"
fi

# Alte, abweichende Regel entfernen
if [ -n "${OLD_IP}" ] && [ "${OLD_IP}" != "${NEW_IP}" ]; then
  ip rule del from "${WG_NET}" to "${OLD_IP}/32" lookup main priority 110 2>/dev/null || true
  iptables -t nat -D POSTROUTING -s "${WG_NET}" -d "${OLD_IP}/32" -o "${LAN_IF}" -j MASQUERADE 2>/dev/null || true
  iptables -D FORWARD -i wg0 -o "${LAN_IF}" -d "${OLD_IP}/32" -j ACCEPT 2>/dev/null || true
fi

# Regel für aktuelle WAN-IP sauber neu setzen
ip rule del from "${WG_NET}" to "${NEW_IP}/32" lookup main priority 110 2>/dev/null || true
ip rule add from "${WG_NET}" to "${NEW_IP}/32" lookup main priority 110

iptables -t nat -C POSTROUTING -s "${WG_NET}" -d "${NEW_IP}/32" -o "${LAN_IF}" -j MASQUERADE >/dev/null 2>&1 || \
iptables -t nat -A POSTROUTING -s "${WG_NET}" -d "${NEW_IP}/32" -o "${LAN_IF}" -j MASQUERADE

iptables -C FORWARD -i wg0 -o "${LAN_IF}" -d "${NEW_IP}/32" -j ACCEPT >/dev/null 2>&1 || \
iptables -A FORWARD -i wg0 -o "${LAN_IF}" -d "${NEW_IP}/32" -j ACCEPT

echo "${NEW_IP}" > "${STATE_FILE}"
```

Danach ausführbar machen:

```bash
chmod +x /usr/local/sbin/update-wan-bypass.sh
```

---

## WireGuard-Serverkonfiguration `/etc/wireguard/wg0.conf`

Im `wg0`-Interface werden die Skripte per `PostUp` und `PostDown` eingebunden.

```ini
# Do not alter the commented lines
# They are used by wireguard-install
# ENDPOINT wg-pesthund.duckdns.org

[Interface]
Address = 10.7.0.1/24
PrivateKey = <SERVER_PRIVATE_KEY>
ListenPort = 51828
PostUp = /usr/local/sbin/wg-policy-up.sh
PostDown = /usr/local/sbin/wg-policy-down.sh

# BEGIN_PEER android
[Peer]
PublicKey = <ANDROID_PUBLIC_KEY>
PresharedKey = <OPTIONAL_PRESHARED_KEY>
AllowedIPs = 10.7.0.2/32
```

---

## ProtonVPN einrichten (`proton0`)

Dieser Abschnitt beschreibt die vollständige Einrichtung des ausgehenden WireGuard-Tunnels zu ProtonVPN.

### 1. ProtonVPN-WireGuard-Konfiguration beschaffen

Im ProtonVPN-Konto wird eine **manuelle WireGuard-Konfiguration** für einen gewünschten Server erzeugt und heruntergeladen.

Benötigt werden daraus insbesondere:

- Private Key
- Interface-Adresse(n)
- Peer Public Key
- Endpoint
- optional DNS-Server

### 2. Konfigurationsdatei anlegen

Datei: `/etc/wireguard/proton0.conf`

```ini
[Interface]
PrivateKey = <PROTON_PRIVATE_KEY>
Address = 10.x.x.x/32
Address = 2a07:xxxx::x:x/128
Table = off

[Peer]
PublicKey = <PROTON_SERVER_PUBLIC_KEY>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <PROTON_ENDPOINT>:51820
PersistentKeepalive = 25
```

### Bedeutung der wichtigsten Optionen

- `Table = off`
  - verhindert, dass `wg-quick` automatisch die globale Default-Route des Systems umbiegt
  - das Routing für Client-Traffic wird in dieser Anleitung bewusst über **Policy Routing** gesteuert
- `AllowedIPs = 0.0.0.0/0, ::/0`
  - bedeutet: ProtonVPN ist grundsätzlich als Standard-Uplink für IPv4 und IPv6 geeignet
- `PersistentKeepalive = 25`
  - hält NAT-Mappings stabil, was besonders hinter Heimroutern sinnvoll ist

### 3. DNS-Zeile bewusst weglassen

In dieser Anleitung wird in `proton0.conf` **keine** `DNS = ...`-Zeile verwendet.

Grund:

- `wg-quick` verarbeitet `DNS = ...` über `resolvconf`
- auf vielen minimalistischen Debian-/LXC-Installationen ist `resolvconf` nicht vorhanden
- für dieses Setup ist die DNS-Konfiguration des Hosts für `proton0` nicht entscheidend

Falls trotzdem eine `DNS = ...`-Zeile gewünscht ist, muss zusätzlich ein passender Resolver-Handler installiert werden, zum Beispiel `openresolv`.

### 4. Tunnel manuell testen

```bash
wg-quick up proton0
wg show proton0
ip addr show proton0
```

Wenn `proton0` sauber hochkommt, sollte ein Handshake mit dem Proton-Server sichtbar sein.

### 5. Autostart aktivieren

```bash
systemctl enable --now wg-quick@proton0
systemctl status wg-quick@proton0 --no-pager -l
```

### 6. Funktion prüfen

```bash
wg show proton0
ip route show table main
ip addr show proton0
```

Optional kann testweise geprüft werden, ob ein explizit über `proton0` gerouteter Request den Proton-Uplink nutzt. In dieser Anleitung erfolgt die eigentliche Nutzung von `proton0` jedoch gezielt über die später beschriebenen Policy-Routing-Regeln.

### 7. Typische Fehler

#### `resolvconf: command not found`

Wenn `wg-quick up proton0` mit diesem Fehler abbricht, ist fast immer eine `DNS = ...`-Zeile in `proton0.conf` vorhanden.

Lösung:

- `DNS = ...` entfernen
- oder `openresolv` / `resolvconf` installieren

#### `Table = off` vergessen

Wenn `Table = off` fehlt, kann `wg-quick` automatisch die globale Default-Route umstellen. Das kollidiert mit dem hier beschriebenen Policy-Routing-Ansatz.

#### Kein Handshake sichtbar

Dann sind meist diese Punkte zu prüfen:

- falscher Private/Public Key
- falscher Proton-Endpoint
- lokale Firewall blockiert ausgehenden UDP-Verkehr
- `PersistentKeepalive` fehlt bei NAT-sensiblen Umgebungen

---

## Android-Client-Profil

In der Android-WireGuard-App wird ein Profil angelegt, das allen Verkehr zunächst an `wg0` sendet.

```ini
[Interface]
PrivateKey = <ANDROID_PRIVATE_KEY>
Address = 10.7.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
PresharedKey = <OPTIONAL_PRESHARED_KEY>
Endpoint = wg.schlumpf.gleeze.com:51828
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

---

## systemd-Abhängigkeit: `wg0` erst nach `proton0`

Damit `wg0` nicht vor `proton0` startet:

```bash
systemctl edit wg-quick@wg0
```

Eintragen:

```ini
[Unit]
After=wg-quick@proton0.service
Requires=wg-quick@proton0.service
```

Danach:

```bash
systemctl daemon-reload
```

---

## Regelmäßige WAN-IP-Aktualisierung per systemd-Timer

### `/etc/systemd/system/update-wan-bypass.service`

```ini
[Unit]
Description=Update WireGuard WAN bypass rule from DynDNS
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/update-wan-bypass.sh
```

### `/etc/systemd/system/update-wan-bypass.timer`

```ini
[Unit]
Description=Run WAN bypass updater regularly

[Timer]
OnBootSec=30s
OnUnitActiveSec=5min
Unit=update-wan-bypass.service

[Install]
WantedBy=timers.target
```

Aktivieren:

```bash
systemctl daemon-reload
systemctl enable --now update-wan-bypass.timer
```

---

## Inbetriebnahme

### 1. ProtonVPN starten

```bash
systemctl enable --now wg-quick@proton0
```

### 2. WireGuard-Server starten

```bash
systemctl enable --now wg-quick@wg0
```

### 3. WAN-Bypass aktualisieren

```bash
/usr/local/sbin/update-wan-bypass.sh
```

### 4. Prüfen

```bash
wg show
ip rule
ip route show table proton
cat /var/lib/wg-wan-bypass/current_wan_ip
```

---

## Erwartetes Verhalten

### Internetverkehr vom Android-Client

- läuft über **ProtonVPN**
- sichtbar über eine externe IP-Prüfung wie `ifconfig.me`

### Zugriff auf Heimnetz

- `192.168.0.0/22` bleibt lokal

### Zugriff auf eigenen Dienst über WAN-Namen

- `https://dienst.schlumpf.gleeze.com:44385`
- läuft **nicht** über ProtonVPN
- sondern über den normalen Heimrouterpfad zur aktuellen WAN-IP

---

## Debugging

### Prüfen, ob die WAN-IP-Ausnahme greift

```bash
ip route get $(cat /var/lib/wg-wan-bypass/current_wan_ip) from 10.7.0.2 iif wg0
```

Erwartet sinngemäß:

```text
<wan-ip> from 10.7.0.2 via 192.168.0.1 dev eth0
```

### Mitschnitt für den Dienstpfad

```bash
tcpdump -ni eth0 host $(cat /var/lib/wg-wan-bypass/current_wan_ip) and port 44385
```

### Mitschnitt für ProtonVPN

```bash
tcpdump -ni proton0
```

### Status der Dienste

```bash
systemctl status wg-quick@proton0 --no-pager -l
systemctl status wg-quick@wg0 --no-pager -l
journalctl -u wg-quick@wg0 -b --no-pager | tail -n 100
```

---

## Hinweise und Fallstricke

### 1. Hairpin NAT muss funktionieren

Wenn der Router keine Zugriffe auf die eigene WAN-IP aus dem LAN korrekt zurück ins Heimnetz leitet, funktioniert der Bypass für `dienst.schlumpf.gleeze.com` nicht zuverlässig.

### 2. IPv6 beachten

Wenn `dienst.schlumpf.gleeze.com` zusätzlich einen AAAA-Record liefert, kann ein Client über IPv6 zugreifen. Diese Anleitung behandelt den WAN-Bypass **explizit über IPv4**. Falls nötig, muss ein analoger IPv6-Bypass ergänzt werden.

### 3. Keine zweite allgemeine Firewalllösung parallel einmischen

Diese Anleitung geht davon aus, dass die relevanten Regeln über die Skripte gesetzt werden. Zusätzliche Firewall-Tools oder persistente iptables-Dumps können das Verhalten verändern.

---

## Kurzfassung

Dieses Setup macht aus einem Debian-LXC gleichzeitig:

- einen **WireGuard-Server für Android-Clients**,
- einen **Policy-Routing-Knoten**,
- und einen **ProtonVPN-Uplink**.

Der zentrale Trick ist:

- **Internetverkehr** des Remote-Clients → über **`proton0`**
- **Heimnetz und eigene WAN-IP** → über **`eth0` / lokalen Router**
- die **aktuelle WAN-IP** wird automatisch aus einem DynDNS-Namen aufgelöst und als `/32`-Bypass-Regel gepflegt.

Dadurch bleibt ein Heimdienst wie `dienst.schlumpf.gleeze.com:44385` über den normalen Heimanschluss erreichbar, während sonstiger Verkehr über ProtonVPN läuft.
