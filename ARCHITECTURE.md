# farm-os — Architecture cible

Une architecture d'OS et de réseaux virtuels qui mêle Linux et OpenBSD sagement. Void Linux en « dom0 », KVM/QEMU et virt-manager pour la virtualisation et l'accès à distance. Résultat : un homelab fonctionnel, sécurisé, discret quand il le faut, pilotable de n'importe où.

---

**Résumé technique** : hôte minimaliste + hyperviseur + VMs cloisonnées, VPN sélectif, passerelles d'anonymat spécialisées, VMs éphémères torifiées, accès distant sécurisé.

---

## Vue d'ensemble

```
                    [Internet]
                         │
                    [Box ISP]
                         │
                    Wi-Fi / Ethernet
                         │
┌────────────────────────▼────────────────────────────────┐
│  HÔTE : Void Linux minimal                              │
│  ├─ iwd (Wi-Fi RTL8821CE, MAC random)                   │
│  ├─ KVM + QEMU + libvirt + virt-manager                 │
│  ├─ Firefox-ESR (captive portals uniquement)            │
│  ├─ SSH (bindé interne, accessible via tunnel)          │
│  └─ NAT : wlan0 ──► br-wan                              │
└────────────┬────────────────────────────────────────────┘
             │ br-wan
┌────────────▼────────────────────────────────────────────┐
│  VM1 : OpenBSD-router (CLI, headless)                   │
│  ├─ em0 (WAN, sur br-wan)                               │
│  ├─ wg0 : Mullvad multihop NO→IS                        │
│  ├─ wg1 : Mullvad single-hop Paris                      │
│  ├─ wg_inbound : tunnel entrant depuis extérieur        │
│  ├─ pf : firewall + routage par bridge + kill-switch    │
│  ├─ dhcpd : baux statiques par MAC (IPs fixes)          │
│  ├─ unbound : DNS local anti-fuite (DoT)                │
│  └─ sshd (OpenSSH natif)                                │
└─┬────┬────┬────┬────┬────┬────┬────┬─────────────────────┘
  │    │    │    │    │    │    │    │
  ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼
br-vpn br-direct br-whonix br-tor-gw br-i2p-gw br-browse br-ephem br-wg-in
  │     │         │         │         │         │        │
  │     │         │     ┌───▼───┐ ┌───▼───┐     │        │
  │     │         │     │Tor-gw │ │I2P-gw │     │        │
  │     │         │     │OpenBSD│ │OpenBSD│     │        │
  │     │         │     │tor    │ │i2pd   │     │        │
  │     │         │     └───┬───┘ └───┬───┘     │        │
  │     │         │         │         │          │        │
  │     │         │       br-tor    br-i2p       │        │
  │     │         │         │         │          │        │
  │     │     ┌───▼────┐    │         │          │        │
  │     │     │WhonixGW│    │         │          │        │
  │     │     └───┬────┘    │         │          │        │
  │     │         │         │         │          │        │
  │     │       br-ws       │         │          │        │
  │     │         │         │         │          │        │
  │     │    WhonixWS  non-browser    │    anon-browse    │
  │     │  (TorBrowser)  apps         │   (Kicksecure     │
  │     │               torifiées     │   TBB+Chromium-I2P)│
  │     │                             │                    │
  │     ├─ Banque        VMs éphémères torifiées           │
  │     ├─ Bitcoin-node  (spawn-torbox.sh, transient)      │
  │     └─ Sparrow                                          │
  │
  ├─ Routine
  ├─ Dev/Claude
  ├─ Windows 11
  └─ Atelier
```

---

## 1. Hôte : Void Linux

**Base** : Void glibc minimal (`base-minimal`), LUKS + BTRFS, UEFI, GRUB.

**Paquets installés** :
```
linux-firmware-network linux-firmware-intel
iwd elogind dbus polkit
qemu libvirt virt-manager bridge-utils dnsmasq iptables-nft swtpm
firefox-esr bubblewrap
openssh
sway foot wofi   # ou i3/openbox selon goût
```

**Services runit activés** : `dbus`, `iwd`, `elogind`, `libvirtd`, `virtlogd`, `virtlockd`, `sshd`.

**Durcissement** :
- `/etc/modprobe.d/rtw88.conf` : `options rtw_8821ce disable_aspm=1`
- `iwd` : `AddressRandomization=once`, `DisableANQP=true`
- sysctl durcis (`dmesg_restrict`, `kptr_restrict`, `unprivileged_bpf_disabled`, `bpf_jit_harden`)
- IPv6 désactivé si non utilisé
- `sshd` bindé sur IP du bridge interne uniquement, pas sur `wlan0`
- Pas de service à l'écoute externe (`ss -tlnp` propre)
- Firefox lancé uniquement via `alias captive=...` avec profil séparé

---

## 2. Réseau

**Bridges** :
- `br-wan` : NAT depuis l'hôte → WAN pour OpenBSD-router
- `br-vpn` : VMs sortent via Mullvad (wg0)
- `br-direct` : VMs sortent sans VPN
- `br-whonix` : segment Whonix (GW + WS)
- `br-tor-gw` : lien router ↔ VM Tor-gw (OpenBSD)
- `br-tor` : VMs qui veulent torification réseau (non-navigateurs)
- `br-i2p-gw` : lien router ↔ VM I2P-gw (OpenBSD)
- `br-i2p` : VMs qui accèdent à I2P
- `br-browse` : VM anon-browse (Kicksecure)
- `br-ephem` : VMs éphémères (spawn-* scripts)
- `br-wg-in` : accès distant WireGuard entrant

**IPs fixes** via DHCP static leases OpenBSD (baux par MAC). Exemple :
- `br-vpn` : `10.0.10.0/24` — routine `.10`, dev `.20`, claude `.30`, win11 `.40`, atelier `.50`
- `br-direct` : `10.0.20.0/24` — banque `.10`, bitcoin-node `.20`, sparrow `.30`
- `br-whonix` : `10.0.30.0/24` — whonix-gw `.10`, whonix-ws `.20`
- `br-tor` : `10.0.31.0/24` — clients Tor gateway
- `br-i2p` : `10.0.32.0/24` — clients I2P gateway
- `br-browse` : `10.0.33.0/24` — anon-browse `.10`
- `br-ephem` : `10.0.34.0/24` — VMs éphémères (DHCP dynamique)

**VPN Mullvad** (sur la VM OpenBSD-router, pas sur l'hôte) :
- `wg0` : entry Norvège → exit Islande (routine + dev + atelier)
- `wg1` : Paris single-hop (banque — à tester, fallback br-direct)
- Kill-switch pf : `block out on egress from br-vpn:network` sauf via `wg0`
- Script périodique de refresh des endpoints via l'API Mullvad

**DNS** : `unbound` sur OpenBSD-router, DoT vers Mullvad ou Quad9. Les VMs pointent vers OpenBSD comme résolveur. Whonix et anon-browse utilisent leurs propres résolveurs (Tor Browser intègre DNS via Tor).

---

## 3. VMs

### Cœur infra

| VM | Base | Bridges | Usage |
|---|---|---|---|
| **OpenBSD-router** | OpenBSD CLI | br-wan + tous les br-* internes | Routeur, firewall pf, VPN Mullvad, DHCP, DNS, WG inbound |
| **Tor-gw** | OpenBSD CLI | br-tor-gw (WAN), br-tor | Daemon Tor, transproxy TCP + DNS pour VMs non-navigateurs |
| **I2P-gw** | OpenBSD CLI | br-i2p-gw (WAN), br-i2p | i2pd, proxy SOCKS 4447 / HTTP 4444 pour navigateur I2P |

### VMs utilisateur

| VM | Base | Bridge | Usage |
|---|---|---|---|
| **Atelier** | Debian stable | br-vpn | Cockpit d'admin, virt-manager remote, scripts spawn-* |
| **Routine** | Debian stable | br-vpn | Social, mail, factures EDF, etc. |
| **Banque** | Kicksecure ou Slackware | br-direct | Banque + vault PGP/SSH |
| **Bitcoin-node** | Debian stable | br-direct | bitcoind headless |
| **Sparrow** | Debian stable | br-direct | Sparrow → Bitcoin-node via RPC |
| **Dev / Claude** | Debian stable + KDE/Sway | br-vpn | Claude CLI, GitHub, dev |
| **Windows 11** | Windows | br-vpn | TPM swtpm, OVMF, virtio |
| **Whonix-GW** | Whonix officiel | br-wan + br-whonix | Tor gateway pour Whonix-WS |
| **Whonix-WS** | Whonix officiel | br-whonix | Tor Browser officiel (browsing Tor sérieux) |
| **anon-browse** | Kicksecure | br-browse | Tor Browser bundle officiel + Chromium-I2P |

### VMs éphémères

- Instanciées à la demande via `spawn-torbox.sh` / `spawn-i2pbox.sh` / `spawn-directbox.sh`
- Basées sur templates Alpine/Debian minimaux (~120-800 Mo)
- `virt-install --transient` + disque qcow2 COW dans `/tmp` → auto-destruction à l'arrêt
- Bridge selon le script : `br-tor`, `br-i2p`, ou `br-ephem`
- Usage : inspection fichier douteux, URL suspecte, test binaire non vérifié, téléchargement jetable

**Isolation inter-VM** : pf bloque le trafic entre bridges par défaut. Règles explicites uniquement (ex: Sparrow → bitcoind TCP:8332, anon-browse → I2P-gw SOCKS 4447).

---

## 4. anon-browse (Kicksecure)

**Choix de base** : Kicksecure (Debian durci) plutôt qu'OpenBSD, pour que le Tor Browser soit le **bundle officiel Linux** — fingerprint blend-in optimal. OpenBSD aurait été philosophiquement pur mais aurait fait un build port avec différences subtiles qui rendent la signature TBB plus rare/identifiable.

**Setup** :
- Kicksecure minimal + XFCE (ou LXQt)
- **Tor Browser bundle officiel** (télécharger depuis torproject.org, vérifier signature GPG)
- **Chromium** avec profil dédié I2P (proxy pointant sur I2P-gw `10.0.32.1:4444`)
- Discipline : Tor Browser ne parle qu'à son Tor embarqué (pas via Tor-gw — pas de Tor-over-Tor)
- Chromium configuré pour ne router **que** les `.i2p` via proxy I2P-gw, tout le reste est bloqué par pf côté router

**Règles pf côté OpenBSD-router** :
```
# Tor Browser : TCP direct vers Internet pour atteindre les relais Tor
pass in on br-browse proto tcp from 10.0.33.10 to any
# Chromium I2P : SOCKS vers I2P-gw
pass in on br-browse proto tcp from 10.0.33.10 to <i2p-gw> port { 4444 4447 }
# Bloquer UDP et ICMP, pas de fuite DNS clearnet
block in on br-browse proto { udp icmp }
```

---

## 5. Tor-gw (OpenBSD, non-navigateur)

**Rôle** : torifier des VMs qui ne sont pas des navigateurs (CLI tools, IRC, scripts, VMs éphémères).

`/etc/tor/torrc` (extrait) :
```
SOCKSPort 10.0.31.1:9050
DNSPort 10.0.31.1:5353
TransPort 10.0.31.1:9040
TransProxyType default
ClientOnly 1
AvoidDiskWrites 1
SafeSocks 1
# Optionnel : obfs4 bridges si FAI ou Wi-Fi public filtre Tor
# UseBridges 1
# ClientTransportPlugin obfs4 exec /usr/local/bin/obfs4proxy
# Bridge obfs4 <IP:PORT> <FP> cert=<...> iat-mode=0
```

pf sur Tor-gw (transproxy) :
```
pass in on br-tor inet proto tcp from 10.0.31.0/24 to any \
    divert-to 127.0.0.1 port 9040
pass in on br-tor inet proto udp from 10.0.31.0/24 to any port 53 \
    rdr-to 127.0.0.1 port 5353
block out on br-tor
```

Fail-closed si Tor crash.

---

## 6. I2P-gw (OpenBSD)

`pkg_add i2pd`

`/etc/i2pd/i2pd.conf` (extrait) :
```ini
[socksproxy]
enabled = true
address = 10.0.32.1
port = 4447

[httpproxy]
enabled = true
address = 10.0.32.1
port = 4444
```

I2P ne se transproxifie pas facilement → les apps clientes (Chromium sur anon-browse) doivent configurer explicitement le proxy. Moins magique que Tor, acceptable.

---

## 7. VMs éphémères torifiées

### Template

- **Alpine** ~120 Mo (`alpine-standard-3.xx.iso`) ou **Debian** cloud-init ~800 Mo
- Stocké dans `/var/lib/libvirt/images/torbox-template.qcow2` (read-only)

### Script `spawn-torbox.sh` (sur VM atelier, pilote libvirt via SSH)

```bash
#!/bin/bash
NAME="torbox-$(date +%s)"
BASE=/var/lib/libvirt/images/torbox-template.qcow2
DISK=/tmp/$NAME.qcow2

qemu-img create -f qcow2 -F qcow2 -b $BASE $DISK

virsh -c qemu+ssh://user@voidhv/system <<EOF
# Création transient
virt-install --connect qemu+ssh://user@voidhv/system \
  --name $NAME --memory 2048 --vcpus 2 \
  --disk $DISK,device=disk,bus=virtio \
  --network bridge=br-tor,model=virtio \
  --os-variant alpinelinux3.xx \
  --graphics spice --transient --noautoconsole --import
EOF

virt-viewer --connect qemu+ssh://user@voidhv/system $NAME
rm -f $DISK
```

### Variantes

- `spawn-i2pbox.sh` : `--network bridge=br-i2p`, Chromium préconfiguré
- `spawn-directbox.sh` : `--network bridge=br-ephem`, pour tests clearnet sans pollution
- Option disque `-snapshot` QEMU : tout en RAM, zéro écriture disque hôte

---

## 8. Captive portals

Firefox-ESR sur l'hôte, profil `captive` dédié :
- Delete cookies on close, private mode par défaut, pas de password manager
- `privacy.resistFingerprinting=true`, tracking protection strict
- Lancé via `alias captive='firefox -P captive --no-remote --private-window http://neverssl.com'`
- Optionnel : wrap dans `bwrap` avec `--tmpfs /home` pour zéro persistance

---

## 9. Accès distant

**Un seul port exposé sur Internet** : UDP WireGuard inbound sur OpenBSD-router.

**Workflow depuis l'extérieur** :
1. Activer WireGuard client → entrée dans `10.99.0.0/24`
2. SSH vers OpenBSD-router, ProxyJump vers Void/VMs
3. GUI : `virt-manager -c qemu+ssh://user@voidhv/system` → consoles VM graphiques tunnelisées
4. Windows : RDP sur IP interne, via tunnel
5. Partage d'écran live : RustDesk self-hosted si besoin spécifique

**`~/.ssh/config` côté client** :
```
Host obsd
    Hostname 10.99.0.1
Host voidhv
    Hostname 10.0.0.1
    ProxyJump obsd
Host vm-*
    ProxyJump voidhv
```

---

## 10. Presse-papier (politique)

Le presse-papier inter-VM passe par **le laptop client** quand virt-manager est utilisé en remote — il devient donc le maillon de confiance de tout flux clipboard. `spice-vdagent` dans l'invité est ce qui active le canal.

**Règle : activation sélective, désactivation par défaut sur toute VM sensible.**

| VM | spice-vdagent | Note |
|---|---|---|
| Atelier | ✅ | Cockpit, besoin de confort |
| Routine | ✅ | Mails, URLs, quotidien |
| Dev / Claude | ✅ | Copier-coller code |
| Windows 11 | ⚠ RDP | RDP clipboard natif + granulaire (texte/image/fichier) |
| **Banque** | ❌ | Pas de passwords/RIB hors VM |
| **Bitcoin-node** | ❌ | Headless |
| **Sparrow** | ❌ | Cible prioritaire clipboard swappers |
| **Whonix-WS** | ❌ | Désactivé par défaut dans Whonix |
| **anon-browse** | ❌ | Anti-fingerprinting + isolation |
| **VMs éphémères** | ❌ | Principe d'isolation |

**Discipline complémentaire** :
- KeePassXC (VM banque) avec auto-clear clipboard après 10 secondes
- Ne **jamais** copier seed phrase ou clé privée via presse-papier, même entre VMs de confiance — saisie manuelle ou QR code
- Une seule fenêtre SPICE active à la fois lors de manipulation sensible
- Relire systématiquement une adresse crypto collée avant signature (clipboard swappers)
- Pour Windows, privilégier RDP avec canal clipboard limité au texte uniquement

**Installer / retirer** :
```bash
# Debian/Kicksecure : installer
apt install spice-vdagent

# Debian/Kicksecure : retirer
apt purge spice-vdagent
systemctl disable spice-vdagentd
```

---

## 11. Audio (politique)

Audio VM ↔ client via canal SPICE, tunnelé dans le même SSH que l'affichage et le presse-papier. Aucun port réseau à ouvrir côté hôte.

**Activation par VM** (cohérente avec la politique clipboard) :

| VM | Audio | Note |
|---|---|---|
| Atelier | ✅ | Notifications, vidéos doc |
| Routine | ✅ | Media, visio |
| Dev / Claude | ✅ | Tutos, notifications |
| Windows 11 | ✅ | Drivers virtio-win, ou RDP audio |
| **Banque** | ❌ | Inutile, surface d'attaque en moins |
| **Bitcoin-node** | ❌ | Headless |
| **Sparrow** | ❌ | Pas de besoin, side-channel potentiel |
| **Whonix-WS** | ❌ | Désactivé par défaut (anti-fingerprinting) |
| **anon-browse** | ❌ | Anti-fingerprinting par défaut |
| **VMs éphémères** | ❌ | Isolation par défaut |

**Configuration libvirt** (XML VM) :
```xml
<sound model='ich9'>
  <audio id='1'/>
</sound>
<audio id='1' type='spice'/>
```

Ou via virt-manager → Add Hardware → Sound → `ich9`.

**Côté invité** :
- Linux : PipeWire ou PulseAudio (inclus dans tout desktop Debian standard)
- Windows : installer virtio-win pour les drivers audio SPICE

**Latence** : imperceptible en LAN, ~50-200 ms via WireGuard depuis l'extérieur (OK pour media, insuffisant pour temps réel).

**Side-channels** : l'audio peut révéler du fingerprinting (codecs, latences, hardware émulé) — raison pour laquelle Whonix le désactive. Garder en tête qu'une VM oubliée en visio continue d'émettre.

---

## 12. Chiffrement et modèle de menace

**Choix structurant** : Void **n'est pas chiffré** (pas de LUKS host). Le chiffrement est poussé au niveau des qcow2 des VMs sensibles. Toutes les VMs sauf `atelier` sont en **qcow2-LUKS natif QEMU**. Les passphrases sont stockées dans un trousseau **`pass`** chiffré par **GPG**, sur Void.

### Ce que ça protège

- **Fuite de qcow2** (backup offsite, clone SSD, snapshot envoyé pour debug) : blobs LUKS illisibles sans la passphrase.
- **Partition Void montée par tiers** : les secrets libvirt sont `ephemeral='yes'`, ils ne sont jamais persistés sur disque ; seul `~/.password-store/` est accessible, chiffré par GPG.
- **Compromission remote sans accès à la clé GPG** : un malware qui copie les qcow2 à distance ne peut pas les déchiffrer sans la passphrase GPG.

### Ce que ça ne protège pas (trade-offs assumés)

- **Vol physique + bruteforce passphrase GPG** : si la passphrase est faible, bruteforce possible. Mitigation : passphrase **forte** (entropie 100+ bits, type diceware 8 mots), gérée par l'utilisateur.
- **Host Void compromis en runtime (root)** : libvirtd a les secrets en RAM après unlock, attaquant root peut les dumper. Idem pour la RAM des VMs. Inhérent au modèle KVM.
- **Atelier non chiffrée** : contient la clé SSH `farm_os_admin` en clair. Mitigation : passphrase forte sur la clé SSH ; au-delà, la clé seule ne donne pas accès aux VMs chiffrées, qui ne démarrent pas sans unlock-farm.

### Workflow de déverrouillage

```
Boot Void (sans prompt, instant)
    ↓
libvirtd démarre, seule VM atelier autostart
    ↓
SSH distant vers Void (via WireGuard inbound, fonctionne tout de suite)
    ↓
./bin/unlock-farm.sh → prompt passphrase GPG
    ↓
pass décrypte chaque entrée farm/<VM>
    ↓
Secrets libvirt ephemeral définis + injectés (RAM uniquement)
    ↓
VMs chiffrées démarrées dans l'ordre (routeur d'abord)
```

### Sauvegardes cryptographiques

- **Clé GPG privée** : backup via `paperkey`, imprimé sur papier, rangé dans un lieu physique sûr hors machine. Sans ce papier ET perte du disque, les VMs chiffrées sont définitivement inaccessibles.
- **Passphrases qcow2** : **jamais en dehors** de `pass` et de la tête de l'utilisateur. Pas de sauvegarde séparée — le trousseau `pass` (chiffré GPG) est la source unique.
- **UUIDs des secrets libvirt** : listés dans `~/bin/unlock-farm.sh`, ce script est sauvegardable en clair (pas de secret).

### Politique par VM

| VM | qcow2-LUKS | Autostart |
|---|---|---|
| atelier | ❌ | ✅ |
| obsd-router, obsd-torgw, obsd-i2pgw | ✅ | ❌ |
| routine, dev/claude, banque, bitcoin-node, sparrow | ✅ | ❌ |
| Whonix-GW, Whonix-WS | ✅ | ❌ |
| anon-browse | ✅ | ❌ |
| Windows 11 | ✅ (qcow2) + BitLocker interne optionnel | ❌ |
| Templates et VMs éphémères | ❌ | — |

### Limites acceptées

- **Bitcoin-node encrypté** coûte du CPU sur la sync (~600 Go à re-lire et re-chiffrer lors des réorgs profondes). Acceptable car le CPU moderne + AES-NI rend le surcoût marginal.
- **Autostart désactivé** pour toutes les VMs chiffrées → après reboot Void, intervention utilisateur obligatoire (SSH + `unlock-farm.sh`). Pas de full headless.

---

## 13. Ordre de chantier

1. Install Void minimal (sans LUKS) + stack KVM + Wi-Fi iwd + Firefox captive + pass/gnupg
2. VM Debian atelier (sur réseau libvirt `default` NAT pour commencer) — **reste en clair**, sert de cockpit
3. Préambule chiffrement : GPG keyring + `pass` init + paperkey backup + script `unlock-farm.sh` stub
4. VM OpenBSD-router (WAN seul, puis `br-lan` isolé) → à chiffrer via qcow2-LUKS après install
5. pf de base + dhcpd static leases + DNS unbound
6. Tests avec VM jetable sur `br-lan`
7. WireGuard Mullvad sur OpenBSD-router (wg0, wg1)
8. Multi-bridges (br-vpn / br-direct)
9. WireGuard inbound + reconfig client distant
10. Déploiement progressif : routine, dev, windows, banque, bitcoin-node, sparrow (tous qcow2-LUKS sauf atelier)
11. Whonix (GW + WS) sur `br-whonix` → qcow2-LUKS
12. **Tor-gw** (OpenBSD) sur `br-tor-gw`/`br-tor` → qcow2-LUKS
13. **I2P-gw** (OpenBSD) sur `br-i2p-gw`/`br-i2p` → qcow2-LUKS
14. **anon-browse** (Kicksecure + TBB + Chromium I2P) sur `br-browse` → qcow2-LUKS
15. **Template Alpine + scripts spawn-*** pour VMs éphémères (pas chiffré)

---

## 14. Décisions écartées (pour mémoire)

- **Qubes OS** : trop restrictif, workflow imposé
- **OpenBSD en hôte** : mauvais support VM Linux desktop (vmm headless only, pas de GPU/USB/SPICE)
- **Mullvad sur l'hôte Void** : forcerait VPN sur toutes les VMs (banque bloque IP VPN, Bitcoin node impact)
- **Passthrough PCI de la Wi-Fi interne vers OpenBSD** : RTL8821CE non supporté par OpenBSD
- **AnyDesk** : fermé, télémétrie, opposé au modèle
- **OpenBSD pour anon-browse** : port tor-browser OpenBSD ≠ bundle officiel Linux, fingerprinting plus rare/identifiable, updates décalés
- **Monero infra dédiée** (node + wallet séparés) : trop lourd pour l'usage, Feather Wallet dans Whonix-WS suffit
- **Tor + I2P sur la même VM OpenBSD** : isolation accrue en gardant Tor-gw et I2P-gw séparées
- **LUKS sur Void (disque hôte)** : remplacé par qcow2-LUKS par VM + trousseau GPG, pour permettre un reboot Void sans interaction (accès distant post-reboot via WireGuard sans console physique)
- **LUKS côté invité dans chaque VM** : remplacé par qcow2-LUKS natif QEMU — OS invité vierge, chiffrement géré à l'extérieur, plus simple à automatiser
