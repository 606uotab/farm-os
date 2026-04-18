# farm-os — Installation base

Guide d'installation du socle du homelab. À la fin de ce document :

- Hôte **Void Linux** minimal avec stack KVM, Wi-Fi configuré, Firefox dédié aux captive portals.
- VM **atelier** Debian XFCE qui sert de cockpit d'administration.
- VM **OpenBSD-router** routant trois bridges internes (`br-wan`, `br-vpn`, `br-direct`) avec pf + dhcpd + unbound.
- Tunnel **Mullvad WireGuard** multihop (Norvège → Islande) actif sur `br-vpn` avec kill-switch.
- Accès distant via **WireGuard inbound** + **SSH ProxyJump** + **virt-manager remote** : pilotage GUI de toutes les VMs depuis n'importe où.

La suite (déploiement des VMs applicatives — routine, banque, Bitcoin, Windows, Whonix, Tor-gw, I2P-gw, anon-browse, VMs éphémères) est dans `APPLICATIVES.md`.

---

## Conventions

- Blocs de code = copy-paste direct (sauf mention `[côté client]` / `[dans la VM X]`).
- Chaque phase a une section **Vérifications** à la fin ; si KO, voir **Troubleshooting**.
- Les valeurs à personnaliser sont déclarées comme variables shell en début de section, à éditer UNE fois.
- Secrets (passwords, clés, numéros Mullvad) : jamais commités. Voir `SECRETS.md.example` (à créer à part).

## Prérequis matériel

- Machine x86_64 avec CPU supportant VT-x (Intel) ou AMD-V
- RAM : 16 Go recommandé (8 Go minimum)
- Disque : 500 Go recommandé (200 Go minimum)
- Wi-Fi RTL8821CE ou Ethernet
- Clé USB 4 Go+ pour installation Void

## Prérequis comptes/services

- Compte **Mullvad** payé (numéro de compte 16 chiffres) — phase 5
- Box ISP avec capacité de port forwarding UDP (sinon voir section CGNAT phase 6)
- Téléchargement préalable des ISOs :
  - Void live x86_64 (glibc)
  - Debian stable netinst
  - OpenBSD install amd64 (`.iso`)
  - Alpine virt (pour VM de test phase 4)

## Conventions de nommage

- Hostname hôte : `voidhv`
- Domaine interne : `farm.lan`

## Plan des 6 phases

| # | Objectif | Durée |
|---|---|---|
| 1 | Void Linux hôte + KVM + Wi-Fi + Firefox captive | 1-2 h |
| 2 | VM Debian atelier (cockpit d'admin) | 30 min |
| 3 | VM OpenBSD-router : base + pf + dhcpd + unbound | 1-2 h |
| 4 | Bridges internes + OpenBSD-router complet | 2 h |
| 5 | Mullvad WireGuard (multihop NO→IS + kill-switch) | 1-2 h |
| 6 | Accès distant (WG inbound + SSH + virt-manager remote) | 1-2 h |

Chaque phase laisse le système dans un état stable et testable. Interruption possible entre deux phases sans casser le setup.

---

# Phase 01 — Void Linux hôte + stack KVM

**Objectif** : machine hôte Void Linux fonctionnelle avec Wi-Fi connecté, stack de virtualisation opérationnelle (KVM + libvirt + virt-manager), desktop minimaliste, Firefox dédié captive portals.

**Durée estimée** : 1-2 h (hors téléchargements)

**Prérequis** :
- Clé USB 4 Go+ disponible
- Machine x86_64 avec VT-x/AMD-V activés dans le BIOS/UEFI
- Accès à un autre ordinateur pour télécharger l'ISO
- SSID et password Wi-Fi sous la main

---

## A. Préparer la clé USB d'installation

Depuis un autre ordinateur (Linux) :

```bash
# Télécharger l'ISO Void live glibc (PAS musl)
cd ~/Downloads
curl -LO https://repo-default.voidlinux.org/live/current/void-live-x86_64-$(date +%Y%m%d)-base.iso

# Si la date du jour n'existe pas, lister les builds disponibles :
# curl -s https://repo-default.voidlinux.org/live/current/ | grep -oP 'void-live-x86_64-\d{8}-base\.iso' | sort -u | tail -5
```

Vérifier la signature SHA256 :
```bash
curl -LO https://repo-default.voidlinux.org/live/current/sha256sum.txt
sha256sum -c sha256sum.txt --ignore-missing
```

Identifier la clé USB :
```bash
lsblk
# Repérer /dev/sdX (PAS /dev/sda si c'est le disque système !)
```

Écrire l'ISO (remplacer `sdX` par la vraie lettre) :
```bash
sudo dd if=void-live-x86_64-*-base.iso of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

## B. Boot et installation

1. Brancher la clé USB sur la machine cible.
2. Booter en UEFI (F12/F2/Esc selon constructeur, puis sélectionner la clé USB).
3. Login au prompt :
   - user : `root`
   - password : `voidlinux`

Lancer l'installateur :
```bash
void-installer
```

Réponses à donner (dans l'ordre du menu) :

- **Keyboard** : `fr` (ou ta disposition)
- **Network** : `skip` (on configurera iwd proprement après)
- **Source** : `Local` (rapide) puis on upgradera juste après
- **Hostname** : `voidhv`
- **Locale** : `fr_FR.UTF-8` (ou `en_US.UTF-8` si tu préfères anglais système)
- **Timezone** : `Europe/Paris`
- **Root password** : choisir un solide
- **User account** :
  - Login : ton choix (ex: `toi`)
  - Groups : `wheel,users,audio,video,storage,network,input`
  - Password solide
- **Bootloader** : `GRUB` sur le disque entier (ex: `/dev/nvme0n1` ou `/dev/sda`)
- **Partitions** → `Filesystems`, choisir **Custom** :

**Partitionnement recommandé** (UEFI + BTRFS, **sans LUKS sur Void**) :

| Partition | Taille | Type | Point de montage |
|---|---|---|---|
| EFI | 512 MB | vfat | `/boot/efi` |
| /boot | 1 GB | ext4 | `/boot` |
| / | reste du disque | btrfs | `/` |

Dans l'installer, créer les partitions avec `cfdisk`, puis :
- Formater `/boot/efi` en `vfat`
- Formater `/boot` en `ext4`
- Formater `/` en `btrfs`

Sous-volumes BTRFS (à créer après install ou ignorer pour l'instant) : `@`, `@home`, `@var`.

**Choix explicite** : Void est installé **sans chiffrement disque** (pas de LUKS). Le chiffrement est poussé au niveau des qcow2 des VMs sensibles (`APPLICATIVES.md`), avec passphrases protégées par un trousseau GPG. Rationale : permet un reboot Void sans interaction (utile pour récupération à distance via WireGuard) tout en conservant la confidentialité des données sensibles contre les fuites numériques de qcow2. Voir `ARCHITECTURE.md` section « Chiffrement et modèle de menace ».

**Confirmer** et lancer l'install.

Une fois fini : **Reboot**, retirer la clé USB.

## C. Premier boot : sync, firmware, upgrade

Login avec ton user, puis `su -` avec le mot de passe root.

```bash
# Sync des dépôts + upgrade
xbps-install -Suy

# Dépôts nonfree (pour firmware Wi-Fi propriétaire)
xbps-install -y void-repo-nonfree
xbps-install -Suy

# Firmware réseau + intel (ou amd selon CPU)
xbps-install -y linux-firmware-network linux-firmware-intel

# Vérifier présence firmware RTL8821CE
ls /lib/firmware/rtw88/ | grep 8821
# Doit afficher : rtw8821c_fw.bin (et éventuellement wow variant)
```

## D. Wi-Fi avec iwd

```bash
# Variables à éditer :
WIFI_SSID="ton_ssid"
WIFI_PASSWORD="ton_password_wifi"

# Installation
xbps-install -y iwd
```

Désactiver l'ASPM du driver rtw88 (stabilise le RTL8821CE) :
```bash
cat > /etc/modprobe.d/rtw88.conf <<'EOF'
options rtw_8821ce disable_aspm=1
EOF
```

Configuration iwd avec randomisation MAC :
```bash
mkdir -p /etc/iwd
cat > /etc/iwd/main.conf <<'EOF'
[General]
EnableNetworkConfiguration=true
AddressRandomization=once

[Network]
NameResolvingService=resolvconf

[DriverQuirks]
DefaultPowerSave=off
EOF
```

Activer les services runit :
```bash
ln -sf /etc/sv/dbus /var/service/
ln -sf /etc/sv/iwd /var/service/

# Attendre ~5 sec que les services démarrent
sleep 5
sv status dbus iwd
# Doit afficher "run" pour les deux
```

Se connecter au Wi-Fi :
```bash
iwctl
```

Puis dans le shell iwd :
```
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect <ton_ssid>
# (entrer le password au prompt)
exit
```

Vérifier :
```bash
ip -br addr show wlan0
ping -c 3 9.9.9.9
ping -c 3 voidlinux.org
```

## E. Stack virtualisation

Vérifier que KVM est disponible :
```bash
grep -Eo 'vmx|svm' /proc/cpuinfo | sort -u
# Doit afficher "vmx" (Intel) ou "svm" (AMD)

ls /dev/kvm
# Doit exister. Si absent → activer VT-x/AMD-V dans le BIOS et redémarrer
```

Installer la stack :
```bash
xbps-install -y \
  qemu libvirt virt-manager bridge-utils dnsmasq \
  iptables-nft swtpm polkit
```

Activer les services :
```bash
ln -sf /etc/sv/libvirtd /var/service/
ln -sf /etc/sv/virtlogd /var/service/
ln -sf /etc/sv/virtlockd /var/service/
ln -sf /etc/sv/polkitd /var/service/ 2>/dev/null || true

sleep 5
sv status libvirtd virtlogd virtlockd
```

Ajouter ton user aux groupes KVM/libvirt :
```bash
# Variable à éditer :
MYUSER="toi"

usermod -aG libvirt,kvm "$MYUSER"
```

**Déconnexion/reconnexion obligatoire pour que les groupes s'appliquent** :
```bash
exit        # sort du su -
exit        # se déconnecte du user
# Se re-loguer
```

Vérifier (en user normal, pas root) :
```bash
groups
# doit contenir libvirt et kvm

virsh list --all
# doit répondre sans demander de mot de passe
# (aucune VM listée au début, c'est normal)

virsh net-list --all
# doit voir un réseau "default" (NAT libvirt)
```

Démarrer et rendre persistant le réseau default :
```bash
sudo virsh net-start default 2>/dev/null
sudo virsh net-autostart default
```

## F. Desktop minimal (Sway recommandé)

Installer Sway (Wayland tiling) + dépendances :
```bash
sudo xbps-install -y \
  sway foot wofi waybar swaylock swayidle \
  xdg-desktop-portal xdg-desktop-portal-wlr \
  elogind grim slurp wl-clipboard \
  noto-fonts-ttf noto-fonts-emoji
```

Activer elogind (session manager) :
```bash
sudo ln -sf /etc/sv/elogind /var/service/
sleep 3
sudo sv status elogind
```

Lancer Sway manuellement depuis tty (pas de display manager) :
```bash
# Depuis le tty après login :
exec sway
```

**Config Sway minimale** (à créer UNE fois, côté user) :
```bash
mkdir -p ~/.config/sway
cp /etc/sway/config ~/.config/sway/config
# Éditer ensuite selon goût
```

Alternative si Sway ne plaît pas : `i3` + X11 (`xbps-install -y xorg-minimal i3 dmenu alacritty`).

## G. Firefox-ESR pour captive portals + outils trousseau

```bash
sudo xbps-install -y firefox-esr bubblewrap password-store gnupg2
```

`password-store` (`pass`) et `gnupg2` sont installés maintenant ; l'initialisation du trousseau et la génération de la clé GPG se font dans la partie préambule de `APPLICATIVES.md`.

Créer le profil dédié :
```bash
firefox -CreateProfile "captive $HOME/.mozilla/firefox/captive"
```

Lancer une fois pour initialiser et faire les réglages :
```bash
firefox -P captive --no-remote
```

Dans Firefox → `about:preferences` → régler :
- **Privacy & Security** :
  - Enhanced Tracking Protection : **Strict**
  - Cookies and Site Data : cocher "Delete cookies and site data when Firefox is closed"
  - History : "Never remember history"
  - Logins and Passwords : décocher "Ask to save logins"
  - Address Bar suggestions : tout décocher
  - Firefox Data Collection : tout décocher
- **Search** : DuckDuckGo, désactiver les suggestions

Dans `about:config` (confirmer le warning) :
```
privacy.resistFingerprinting = true
privacy.trackingprotection.enabled = true
network.cookie.lifetimePolicy = 2
browser.safebrowsing.malware.enabled = false
browser.safebrowsing.phishing.enabled = false
beacon.enabled = false
```

Fermer Firefox.

Ajouter l'alias dans `~/.bashrc` :
```bash
cat >> ~/.bashrc <<'EOF'

# Firefox captive portal (jamais pour browsing normal)
alias captive='firefox -P captive --no-remote --private-window http://neverssl.com &'
EOF
source ~/.bashrc
```

## H. Durcissement sysctl minimal

```bash
sudo tee /etc/sysctl.d/99-hardening.conf > /dev/null <<'EOF'
# Kernel hardening
kernel.dmesg_restrict=1
kernel.kptr_restrict=2
kernel.unprivileged_bpf_disabled=1
net.core.bpf_jit_harden=2

# Désactiver IPv6 (à commenter si tu l'utilises)
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1

# Network hardening générique
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.all.send_redirects=0
net.ipv4.icmp_echo_ignore_broadcasts=1
EOF

sudo sysctl --system
```

**Note SSH** : on ne l'installe PAS encore. On le configurera en phase 06 quand les bridges internes existeront, pour ne binder que sur l'IP interne.

## I. Vérifications finales phase 01

```bash
# Wi-Fi up et routé
ip -br addr show wlan0                       # doit avoir une IP
ping -c 2 9.9.9.9                            # doit répondre

# KVM dispo
ls /dev/kvm                                   # doit exister
lsmod | grep kvm                              # kvm_intel ou kvm_amd

# libvirt fonctionnel
virsh list --all                              # répond sans sudo
virsh net-list --all | grep -q active         # default actif

# Stack complète
which qemu-system-x86_64 virt-manager swtpm   # tous présents

# Sway lance
# (tester depuis tty : exec sway)

# Firefox captive
which firefox                                 # présent
ls ~/.mozilla/firefox/ | grep captive         # profil existe

# Aucun service à l'écoute externe
ss -tlnp 2>&1 | grep -v 127.0.0.1 | grep -v ::1
# Doit être vide ou ne contenir que des sockets de bind locaux
```

Si **toutes** les vérifications passent → phase 01 validée, passer à `02-vm-atelier.md`.

---

## Troubleshooting

### Wi-Fi ne se connecte pas

```bash
# Vérifier que le firmware est chargé
dmesg | grep -i rtw88
# Doit voir "rtw_8821ce" sans erreur

# Redémarrer iwd
sudo sv restart iwd

# Si ASPM reste problématique, essayer aussi :
echo "options rtw_8821ce disable_msi=1" | sudo tee -a /etc/modprobe.d/rtw88.conf
sudo reboot
```

### Captive portal à la bibliothèque/hôtel

Lancer la commande après connexion Wi-Fi :
```bash
captive
```
Valider le portail, puis **fermer** Firefox. Discipline : ne pas l'utiliser pour autre chose.

### /dev/kvm absent

- Entrer dans le BIOS/UEFI au démarrage
- Activer "Intel Virtualization Technology" (VT-x) ou "SVM Mode" (AMD)
- Sauvegarder, redémarrer

### virsh demande un password

- Le user n'est pas dans le groupe `libvirt`
- Vérifier : `groups | grep libvirt`
- Si absent : `sudo usermod -aG libvirt,kvm $USER` puis déconnexion complète + reconnexion (pas juste `su`)

### Sway ne démarre pas

```bash
# Vérifier elogind
sudo sv status elogind

# Vérifier session
loginctl list-sessions

# Fallback i3 :
sudo xbps-install -y xorg-minimal i3 dmenu alacritty
# puis : startx
```

### Pas de son dans les VMs plus tard

(Sera traité en phase 07 — ignorer pour l'instant)

---

## État attendu en fin de phase 01

- Machine boot sur Void, login user fonctionne
- Wi-Fi connecté automatiquement au reboot (iwd persiste la config)
- `virt-manager` lance une fenêtre sans erreur
- Firefox `captive` dispo via alias, jamais utilisé pour autre chose
- Aucune VM créée encore
- Aucun service à l'écoute externe
- Void non chiffré (boot sans interaction) ; `pass` + `gnupg2` installés pour la suite


---

# Phase 02 — VM Debian atelier (cockpit d'administration)

**Objectif** : créer la VM Debian qui servira de **station de pilotage** pour tout le reste du homelab. Depuis cette VM, on lance `virt-manager` connecté à l'hôte via SSH, on écrit les scripts `spawn-*`, on navigue la doc. L'hôte Void reste minimal, tout le confort d'admin vit dans cette VM.

**Durée estimée** : 30 min (hors téléchargement ISO)

**Prérequis** :
- Phase 01 validée
- ISO Debian stable netinst téléchargée (`debian-XX.X.X-amd64-netinst.iso` depuis [debian.org](https://www.debian.org/CD/netinst/))

**Référence archi** : `ARCHITECTURE.md` §3 (VM Atelier, destinée à `br-vpn` plus tard, pour l'instant sur `default` NAT).

---

## A. Préparer le stockage des ISOs

Depuis ton user sur l'hôte Void :
```bash
mkdir -p ~/isos
# Déposer debian-XX.X.X-amd64-netinst.iso dans ~/isos/
ls -lh ~/isos/
```

Rendre le dossier accessible à libvirt-qemu :
```bash
sudo chmod +x ~
sudo chmod o+r ~/isos/*.iso
```

## B. Créer la VM via virt-manager (UI)

Lancer :
```bash
virt-manager &
```

Dans virt-manager :
1. **File → New Virtual Machine**
2. **Local install media (ISO image or CDROM)** → Forward
3. Browse → naviguer vers `~/isos/debian-*-netinst.iso`
4. Choose the operating system : `debian12` (ou le plus proche)
5. **Memory** : 4096 MB — **CPUs** : 2
6. **Enable storage** : 40 GB, format qcow2, Manage → "Select or create custom storage" → nom `atelier.qcow2`
7. **Name** : `atelier`
8. **Network selection** : `default` (NAT libvirt, seule option pour l'instant)
9. **Customize configuration before install** : ✅ cocher
10. **Finish**

Dans la fenêtre "Customize" avant install :
- **Overview** → Firmware : `UEFI x86_64: /usr/share/OVMF/OVMF_CODE.fd`
- **CPUs** → Copy host CPU configuration : ✅
- **Disk 1** → Advanced → Disk bus : `VirtIO`, Cache mode : `none`, Discard : `unmap`
- **NIC** → Device model : `virtio`
- **Display Spice** → gardé par défaut
- **Sound ich9** → gardé (politique audio : atelier = ✅)
- **Add Hardware → Channel** → Name: `org.qemu.guest_agent.0` (si pas déjà là)
- Appliquer → **Begin Installation**

## C. Installer Debian (graphique ou texte)

L'installateur Debian démarre dans la console virt-manager.

Réponses principales :
- **Language** : French (ou anglais selon préférence)
- **Country** : France
- **Keymap** : French
- **Hostname** : `atelier`
- **Domain** : `farm.lan`
- **Root password** : solide
- **Full name + username** : ton user habituel
- **Partitioning** : `Use entire disk` → `All files in one partition`
- **Software selection** :
  - Debian desktop environment : **décocher**
  - Xfce : décocher (on installera manuellement après, plus léger)
  - web server, print server, etc. : décocher
  - **SSH server** : ✅ cocher
  - **Standard system utilities** : ✅ cocher

Attendre l'install. Quand terminé → Continue, la VM redémarre.

## D. Post-install : système minimal

Login avec ton user. Puis `su -` (root).

```bash
# Mise à jour complète
apt update && apt upgrade -y

# Paquets de base
apt install -y \
  sudo curl wget gnupg ca-certificates \
  git vim tmux htop tree jq \
  net-tools iputils-ping dnsutils \
  qemu-guest-agent spice-vdagent
```

Ajouter ton user à sudo :
```bash
usermod -aG sudo TON_USER
# Remplacer TON_USER par le user créé à l'install
```

Activer qemu-guest-agent et spice-vdagent (pour clipboard et intégration virt-manager) :
```bash
systemctl enable --now qemu-guest-agent
systemctl enable --now spice-vdagentd
```

## E. Environnement de bureau léger : XFCE

```bash
apt install -y xfce4 xfce4-terminal xfce4-goodies \
  lightdm network-manager-gnome \
  firefox-esr \
  file-roller
```

Rendre lightdm le display manager par défaut :
```bash
systemctl enable lightdm
```

Redémarrer la VM :
```bash
reboot
```

Au reboot, login graphique via LightDM → XFCE.

## F. Outils de pilotage dans l'atelier

Depuis un terminal XFCE (user normal, `sudo` quand nécessaire) :

```bash
sudo apt install -y \
  virt-manager libvirt-clients \
  openssh-client \
  remmina remmina-plugin-rdp \
  keepassxc \
  thunderbird
```

- `virt-manager` + `libvirt-clients` : client pour piloter libvirt sur l'hôte Void à distance (phase 06)
- `remmina` : client RDP pour la future VM Windows 11
- `keepassxc` : gestionnaire de mots de passe (pas obligatoire, mais utile dès maintenant)
- `thunderbird` : client mail, optionnel à ce stade

## G. Configuration SSH prête pour l'archi finale

Générer une clé SSH dédiée à l'admin farm-os :
```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/farm_os_admin -C "atelier-admin"
# Passphrase obligatoire
```

Préparer `~/.ssh/config` (les hôtes n'existent pas encore, mais le fichier sera complété en phase 06) :
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat > ~/.ssh/config <<'EOF'
# farm-os SSH config — phases ultérieures compléteront avec IPs réelles

# Hôte Void (sera accessible en phase 06 via br-wan interne)
Host voidhv
    # Hostname 10.0.0.1      # à décommenter phase 04
    User TON_USER_HOTE
    IdentityFile ~/.ssh/farm_os_admin
    IdentitiesOnly yes

# VM OpenBSD-router (phase 03 -> sur default NAT, phase 04 -> IP interne)
Host obsd
    # Hostname 10.0.0.2      # à décommenter phase 04
    User TON_USER_OBSD
    IdentityFile ~/.ssh/farm_os_admin
    IdentitiesOnly yes

# Les VMs futures (routine, dev, banque, etc.) : via ProxyJump
Host vm-*
    ProxyJump voidhv
    User TON_USER_DEFAUT
    IdentityFile ~/.ssh/farm_os_admin
    IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
```

## H. Snapshot de la VM atelier

Retour sur l'hôte Void (terminal hôte) :
```bash
# Arrêter proprement la VM atelier via virt-manager UI, puis :
virsh snapshot-create-as atelier \
  --name "phase02-install-clean" \
  --description "Atelier fraîchement installée, avant configs farm-os"
```

Ça permet de revenir à un atelier propre si on casse quelque chose plus tard.

## I. Vérifications finales phase 02

Dans la VM atelier :
```bash
# Réseau fonctionnel
ping -c 2 9.9.9.9
ping -c 2 debian.org

# Outils présents
which virt-manager virsh ssh remmina keepassxc firefox-esr

# Services invité
systemctl is-active qemu-guest-agent spice-vdagentd
# doit répondre "active" pour les deux

# Clé SSH générée
ls -l ~/.ssh/farm_os_admin ~/.ssh/farm_os_admin.pub

# Intégration clipboard fonctionne
# Test : copier du texte de l'hôte Void et coller dans la VM via terminal xfce
```

Sur l'hôte Void :
```bash
# La VM existe et est shut off après le snapshot
virsh list --all | grep atelier

# Snapshot créé
virsh snapshot-list atelier
```

---

## Troubleshooting

### Pas d'internet dans la VM

- Vérifier que `default` network est actif : `sudo virsh net-list`
- Si inactif : `sudo virsh net-start default`

### virt-manager trop lent dans la VM

- OK à ce stade : on s'en servira vraiment en phase 06 en remote vers l'hôte
- À ce stade, virt-manager dans l'atelier peut être utilisé en local pour se connecter à un libvirt dans la VM (pas le cas ici)

### Clipboard ne marche pas

- Installer `spice-vdagent` dans la VM : `sudo apt install spice-vdagent`
- Redémarrer : `sudo systemctl restart spice-vdagentd`
- Dans virt-manager : View → Resize to VM

### XFCE ne se lance pas

```bash
# Depuis tty dans la VM
sudo systemctl status lightdm
sudo systemctl restart lightdm
# Si échec persistant : sudo journalctl -u lightdm -n 50
```

---

## État attendu en fin de phase 02

- VM `atelier` existe, disque qcow2 ~5-10 Go utilisés
- Snapshot `phase02-install-clean` disponible
- XFCE fonctionnel, outils de base installés
- Clé SSH `farm_os_admin` générée (la publique sera distribuée aux serveurs dans les phases suivantes)
- `~/.ssh/config` stub prêt, à compléter en phase 06
- VM attachée au réseau `default` NAT libvirt (sera migrée sur `br-vpn` en phase 04)


---

# Phase 03 — VM OpenBSD-router (base)

**Objectif** : installer la VM OpenBSD qui deviendra le **point unique réseau** du homelab — routeur, firewall (pf), DHCP, DNS, plus tard VPN Mullvad et WireGuard inbound. Cette phase pose la base : install propre, durcissement minimal, paquets nécessaires. Les bridges multiples et la configuration routeur complète arrivent en phase 04.

**Durée estimée** : 1-2 h (hors téléchargement)

**Prérequis** :
- Phases 01 et 02 validées
- ISO OpenBSD amd64 téléchargée depuis [cdn.openbsd.org](https://cdn.openbsd.org/pub/OpenBSD/) (fichier `install7X.img` pour USB ou `install7X.iso` pour CD ; **préférer `.iso` pour VM**)
- Vérification signature avec `signify` depuis Void (ou au minimum SHA256)

**Référence archi** : `ARCHITECTURE.md` §3 (VM OpenBSD-router), §2 (rôle réseau).

---

## A. Télécharger et vérifier l'ISO OpenBSD

Depuis l'hôte Void (ou l'atelier, peu importe) :
```bash
cd ~/isos/
# Variables (ajuster selon version courante)
OBSD_VER="7.X"   # remplacer par la version stable actuelle (consulter openbsd.org)
OBSD_URL="https://cdn.openbsd.org/pub/OpenBSD/${OBSD_VER}/amd64"

curl -LO "${OBSD_URL}/install${OBSD_VER//./}.iso"
curl -LO "${OBSD_URL}/SHA256"
curl -LO "${OBSD_URL}/SHA256.sig"

# Vérif SHA256
sha256sum -c SHA256 --ignore-missing

# Vérif signature (optionnel mais recommandé)
# Nécessite signify et la clé publique de la release :
# xbps-install -y signify
# curl -LO https://ftp.openbsd.org/pub/OpenBSD/${OBSD_VER}/openbsd-${OBSD_VER//./}-base.pub
# signify -C -p openbsd-${OBSD_VER//./}-base.pub -x SHA256.sig install*.iso
```

## B. Créer la VM

Dans `virt-manager` :
1. File → New Virtual Machine → Local install media (ISO)
2. Browse → `install7X.iso`
3. Choose OS : chercher `openbsd` (prendre la version la plus proche)
4. Memory : **2048 MB** — CPUs : **2**
5. Storage : **20 GB** qcow2, nom `obsd-router.qcow2`
6. Name : `obsd-router`
7. Network : `default` (NAT pour l'install ; ajoutera br-wan en phase 04)
8. Customize before install : ✅

Dans Customize :
- **Overview → Firmware** : `UEFI x86_64: /usr/share/OVMF/OVMF_CODE.fd`
- **CPUs → Copy host CPU configuration** : ✅
- **Disk → Bus** : `VirtIO`, Cache `none`, Discard `unmap`
- **NIC → Model** : `virtio`
- **Sound** : **Remove** (politique audio : router = ❌, headless)
- **Display Spice → Graphics** : garder (utile pour installer, headless après)
- Appliquer → Begin Installation

## C. Installer OpenBSD (texte)

Dans la console de la VM (texte) :

```
Welcome to the OpenBSD/amd64 7.X installation program.
(I)nstall, (U)pgrade, (A)utoinstall or (S)hell?
```

Répondre `I` (Install), puis dans l'ordre :

- **Keyboard layout** : `fr` (ou `us` selon goût)
- **System hostname** : `obsd-router`
- **Network interfaces** : `vio0` (l'interface virtio)
  - IPv4 : `autoconf` (DHCP pour l'install ; changera en phase 04)
  - IPv6 : `none`
- **Password for root** : solide (distinct de tous les autres)
- **Start sshd(8) by default?** : `yes`
- **Setup a user?** : `yes`
  - Login : ton user (ex: `admin`)
  - Full name + password
- **Allow root ssh login?** : `no`
- **Timezone** : `Europe/Paris`
- **Disk** : `sd0` (le disque virtio)
  - Use the (W)hole disk MBR, (G)PT or (E)dit? : `G` (GPT)
  - Use (A)uto layout, (E)dit or (C)ustom layout : `A` (auto)
- **Location of sets** : `cd0`
- **Set name(s)** : accepter les défauts (tous sauf jeux/X si on veut plus léger), mais laisser **X par défaut** (utile pour debug initial ; on peut retirer après)
- Après install : **Reboot**

Éjecter l'ISO (virt-manager → View → Details → IDE CDROM → Disconnect).

## D. Premier boot OpenBSD

Login en user normal (`admin` ou celui que tu as créé), puis `su -`.

Vérifier la version et patcher :
```sh
# Info système
uname -a
sysctl kern.version

# Appliquer tous les patches de sécurité disponibles
syspatch

# Si des patches ont été appliqués, reboot
# Sinon, continuer
```

Mettre à jour les paquets :
```sh
# Définir la variable PKG_PATH si besoin (généralement auto-configuré)
pkg_add -u
```

## E. Installer les paquets nécessaires pour la suite

```sh
# Outils d'admin et futures briques
pkg_add -D snap vim-no_x11 curl git rsync wireguard-tools
```

Paquets notables :
- `vim-no_x11` : éditeur sans dépendances X
- `wireguard-tools` : pour Mullvad en phase 05 (WireGuard est natif OpenBSD, mais les outils userland sont dans ce package)
- **`dhcpd`** et **`unbound`** sont déjà **inclus dans le base system OpenBSD**, rien à installer

## F. Hygiène et durcissement minimal

### Activer doas pour l'user admin

```sh
# Créer /etc/doas.conf
cat > /etc/doas.conf <<'EOF'
permit persist keepenv :wheel
EOF
chmod 600 /etc/doas.conf

# Vérifier que ton user est dans le groupe wheel
grep wheel /etc/group
# Si absent : usermod -G wheel TON_USER
```

Tester en revenant user normal :
```sh
exit
doas whoami
# Doit afficher root après entrée du mot de passe
```

### Durcir sshd

Éditer `/etc/ssh/sshd_config` (en root) :
```sh
doas vi /etc/ssh/sshd_config
```

Régler ces lignes (décommenter/modifier) :
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers TON_USER
ClientAliveInterval 300
ClientAliveCountMax 2
```

**AVANT** de désactiver PasswordAuthentication : déposer ta clé publique. Depuis la VM atelier :
```bash
# Récupérer l'IP de la VM obsd-router depuis l'hôte Void
virsh net-dhcp-leases default
# Copier la clé publique
ssh-copy-id -i ~/.ssh/farm_os_admin.pub TON_USER@IP_OBSD
```

Tester la connexion par clé **avant de désactiver le password** :
```bash
ssh -i ~/.ssh/farm_os_admin TON_USER@IP_OBSD
```

Si OK, retourner sur la console OpenBSD et redémarrer sshd :
```sh
doas rcctl restart sshd
```

### Bannière pf minimale (juste loopback + DHCP sortant, en attendant phase 04)

Éditer `/etc/pf.conf` :
```sh
doas vi /etc/pf.conf
```

Remplacer le contenu par un squelette très permissif pour la phase 03 (on durcit en phase 04) :
```
# /etc/pf.conf — phase 03 (temporaire, minimal)
# Les règles complètes de routing/firewall arrivent en phase 04.

set skip on lo

# Bloquer par défaut
block all

# Accepter toutes les connexions sortantes initiées par l'hôte lui-même
pass out quick keep state

# Accepter SSH entrant depuis la VM atelier (LAN libvirt default)
pass in quick proto tcp from 192.168.122.0/24 to any port ssh

# Bloquer tout autre entrant explicitement
block in log all
```

Charger :
```sh
doas pfctl -f /etc/pf.conf
doas pfctl -si | head -5
# Doit afficher "Status: Enabled"
```

Activer pf au boot (normalement déjà fait par défaut) :
```sh
doas rcctl enable pf
```

### Désactiver les services non nécessaires

```sh
# Lister ce qui tourne
doas rcctl ls started

# Services pas utiles pour un router :
# (xenodm = display manager, on n'en a pas besoin après install)
doas rcctl disable xenodm 2>/dev/null
```

### Synchronisation horaire

```sh
doas rcctl enable ntpd
doas rcctl start ntpd
doas rcctl check ntpd
```

## G. Préparer les outils pour la phase 04

Vérifier que tout est présent pour la suite :
```sh
# WireGuard (support noyau natif + outils userland)
which wg wg-quick                    # /usr/local/bin/wg et wg-quick
ifconfig wg0 create 2>&1 | head -1   # doit créer l'interface sans erreur
ifconfig wg0 destroy                 # cleanup

# dhcpd (base system)
which dhcpd                          # /usr/sbin/dhcpd
ls /etc/examples/dhcpd.conf          # exemple à référencer

# unbound (base system)
which unbound unbound-control
ls /var/unbound/etc/                 # dossier config
```

## H. Snapshot

Arrêt propre de la VM OpenBSD :
```sh
doas shutdown -hp now
```

Sur l'hôte Void :
```bash
virsh snapshot-create-as obsd-router \
  --name "phase03-base-install" \
  --description "OpenBSD fraîchement installée, patches à jour, doas + sshd durci, pf minimal"
```

## I. Vérifications finales phase 03

Depuis la VM atelier, relancer obsd-router puis :
```bash
# SSH par clé (après avoir défini Hostname dans ~/.ssh/config, ou via IP directe)
ssh -i ~/.ssh/farm_os_admin TON_USER_OBSD@IP_OBSD

# Doit se connecter sans demander de password
# Une fois connecté, vérifier sur OpenBSD :
uname -a                              # OpenBSD, bonne version
syspatch -l                           # liste des patches appliqués (ou vide si tout est appliqué)
doas pfctl -si | grep Status          # Status: Enabled
doas rcctl ls started                 # ntpd, sshd au minimum
ping -c 2 9.9.9.9                     # sortie réseau OK
```

Sur l'hôte Void :
```bash
virsh snapshot-list obsd-router       # phase03-base-install présent
virsh list --all                      # obsd-router visible (running ou shut off)
```

---

## Troubleshooting

### `pkg_add` échoue

```sh
# Vérifier PKG_PATH
echo $PKG_PATH
# Si vide ou faux :
export PKG_PATH="https://cdn.openbsd.org/pub/OpenBSD/$(uname -r)/packages/$(machine -a)/"
pkg_add -u
```

### ssh-copy-id refuse "Too many authentication failures"

```bash
ssh-copy-id -o PubkeyAuthentication=no -i ~/.ssh/farm_os_admin.pub TON_USER@IP_OBSD
```

### pf bloque ta session SSH après `pfctl -f`

- Console virt-manager toujours accessible (pas affectée par pf)
- Ajouter une règle `pass in proto tcp from <IP_atelier> to port ssh` avant de reload

### `syspatch` ne trouve rien

Normal si tu as téléchargé l'ISO récemment. Sinon vérifier connexion internet + `/etc/installurl`.

### `ifconfig wg0 create` échoue

- `which wg-quick` pour confirmer l'install
- Noyau OpenBSD >= 6.8 supporte wg natif, pas besoin de module

---

## État attendu en fin de phase 03

- VM `obsd-router` installée, patches appliqués, snapshot `phase03-base-install` créé
- Access SSH par clé uniquement (plus de password), user non-root
- pf chargé avec règles minimales
- ntpd actif
- wireguard-tools + dhcpd + unbound prêts à être configurés
- VM toujours sur `default` NAT (sera reconfigurée en phase 04 pour multi-bridges)


---

# Phase 04 — Bridges internes et OpenBSD-router en mode routeur complet

**Objectif** : faire passer obsd-router de « VM sur NAT libvirt » à **vrai routeur central**. On crée les bridges côté hôte (`br-wan`, `br-vpn`, `br-direct`), on attache les interfaces correspondantes à obsd-router, on configure OpenBSD pour faire du routing + NAT + DHCP + DNS. On termine par un test avec une VM jetable et la migration de l'atelier sur `br-vpn`.

**Durée estimée** : 2 h

**Prérequis** :
- Phases 01, 02, 03 validées
- obsd-router installée, accessible en SSH par clé

**Référence archi** : `ARCHITECTURE.md` §2 (Réseau, bridges, IP plan), §3 (VMs), §5 (Tor-gw/I2P-gw futurs — leurs bridges arrivent en phases 09/10).

---

## A. Plan réseau (rappel)

| Bridge | Sujet | IP network | Gateway (OpenBSD) | DHCP |
|---|---|---|---|---|
| `br-wan` | Vers WAN via NAT Void | 192.168.100.0/24 | 192.168.100.1 (Void bridge) | Static côté OpenBSD |
| `br-vpn` | VMs via VPN Mullvad | 10.0.10.0/24 | 10.0.10.1 | OpenBSD dhcpd |
| `br-direct` | VMs sans VPN | 10.0.20.0/24 | 10.0.20.1 | OpenBSD dhcpd |

Les autres bridges (`br-whonix`, `br-tor`, `br-i2p`, etc.) seront ajoutés dans les phases dédiées.

---

## B. Créer les bridges libvirt (hôte Void)

**Sur l'hôte Void**, en user normal (utilise `sudo` au besoin) :

### br-wan (NAT vers WAN)

```bash
cat > /tmp/br-wan.xml <<'EOF'
<network>
  <name>br-wan</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr-wan' stp='off' delay='0'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'/>
</network>
EOF

sudo virsh net-define /tmp/br-wan.xml
sudo virsh net-autostart br-wan
sudo virsh net-start br-wan
```

Note : pas de bloc `<dhcp>` → libvirt ne distribue pas d'IP sur ce bridge. OpenBSD s'attribuera une IP statique.

### br-vpn (isolé, OpenBSD gère tout)

```bash
cat > /tmp/br-vpn.xml <<'EOF'
<network>
  <name>br-vpn</name>
  <bridge name='virbr-vpn' stp='off' delay='0'/>
</network>
EOF

sudo virsh net-define /tmp/br-vpn.xml
sudo virsh net-autostart br-vpn
sudo virsh net-start br-vpn
```

### br-direct (isolé, OpenBSD gère tout)

```bash
cat > /tmp/br-direct.xml <<'EOF'
<network>
  <name>br-direct</name>
  <bridge name='virbr-direct' stp='off' delay='0'/>
</network>
EOF

sudo virsh net-define /tmp/br-direct.xml
sudo virsh net-autostart br-direct
sudo virsh net-start br-direct
```

### Vérifier

```bash
sudo virsh net-list --all
# Doit montrer :
#  br-wan      active     yes (autostart)   yes (persistent)
#  br-vpn      active     yes               yes
#  br-direct   active     yes               yes
#  default     active     yes               yes   (libvirt par défaut)

ip link show | grep -E 'virbr'
# Doit afficher virbr-wan, virbr-vpn, virbr-direct
```

---

## C. Reconfigurer les NICs d'obsd-router

**Sur l'hôte Void** : arrêter la VM si elle tourne.
```bash
virsh shutdown obsd-router
# attendre l'arrêt propre
```

Éditer la définition :
```bash
virsh edit obsd-router
```

Dans l'XML, localiser la section `<interface>` actuelle (celle sur `default`) et la remplacer par trois interfaces :

```xml
    <interface type='network'>
      <source network='br-wan'/>
      <model type='virtio'/>
      <alias name='net-wan'/>
    </interface>
    <interface type='network'>
      <source network='br-vpn'/>
      <model type='virtio'/>
      <alias name='net-vpn'/>
    </interface>
    <interface type='network'>
      <source network='br-direct'/>
      <model type='virtio'/>
      <alias name='net-direct'/>
    </interface>
```

Supprimer l'ancienne `<interface>` sur `default` si elle existait. Sauver et quitter.

Démarrer la VM :
```bash
virsh start obsd-router
virsh console obsd-router
# Login quand le prompt apparaît
# (Ctrl+] pour quitter la console après)
```

Alternativement, utiliser virt-manager UI pour ce changement si l'édition XML est inconfortable.

---

## D. Configuration OpenBSD : interfaces réseau

Une fois connecté sur obsd-router (console ou SSH si encore possible) :

```sh
# Identifier les nouvelles interfaces (doivent être vio0, vio1, vio2)
ifconfig | grep -E '^vio'
# Exemple attendu :
# vio0: flags=...
# vio1: flags=...
# vio2: flags=...
```

L'ordre vio0/1/2 correspond à l'ordre dans l'XML (WAN, VPN, DIRECT).

Créer les fichiers `hostname.X` pour rendre les configs persistantes :

```sh
# vio0 → WAN (br-wan), static IP dans le réseau NAT Void
doas tee /etc/hostname.vio0 > /dev/null <<'EOF'
inet 192.168.100.10 255.255.255.0
!route add default 192.168.100.1
EOF

# vio1 → br-vpn, gateway interne
doas tee /etc/hostname.vio1 > /dev/null <<'EOF'
inet 10.0.10.1 255.255.255.0
EOF

# vio2 → br-direct, gateway interne
doas tee /etc/hostname.vio2 > /dev/null <<'EOF'
inet 10.0.20.1 255.255.255.0
EOF

# Protéger les fichiers
doas chmod 640 /etc/hostname.vio*
```

Appliquer la nouvelle config réseau :
```sh
doas sh /etc/netstart
```

Vérifier :
```sh
ifconfig vio0 | grep inet          # 192.168.100.10
ifconfig vio1 | grep inet          # 10.0.10.1
ifconfig vio2 | grep inet          # 10.0.20.1

# Route par défaut via br-wan
route -n show | head -10            # default via 192.168.100.1

# Ping internet doit passer
ping -c 2 9.9.9.9
```

Si `ping` ne passe pas : vérifier que `/etc/resolv.conf` pointe sur `9.9.9.9` ou `1.1.1.1` en attendant unbound (section G).

---

## E. Activer l'IP forwarding

```sh
# Activation immédiate
doas sysctl net.inet.ip.forwarding=1

# Persistance au boot
echo "net.inet.ip.forwarding=1" | doas tee -a /etc/sysctl.conf
```

Vérifier :
```sh
sysctl net.inet.ip.forwarding
# doit afficher : net.inet.ip.forwarding=1
```

---

## F. Configuration pf complète

Éditer `/etc/pf.conf` :
```sh
doas vi /etc/pf.conf
```

Remplacer tout le contenu par :

```pf
# /etc/pf.conf — phase 04 (routeur de base, pas encore VPN Mullvad)
#
# Interfaces :
#   vio0 = WAN (br-wan, 192.168.100.10, vers NAT Void -> Internet)
#   vio1 = br-vpn (10.0.10.0/24), gateway 10.0.10.1
#   vio2 = br-direct (10.0.20.0/24), gateway 10.0.20.1
#
# VPN Mullvad arrive en phase 05 : pour l'instant br-vpn sort direct comme br-direct.

wan_if  = "vio0"
vpn_if  = "vio1"
dir_if  = "vio2"

vpn_net = "10.0.10.0/24"
dir_net = "10.0.20.0/24"

# --- Options ---
set skip on lo
set block-policy drop
set loginterface $wan_if

# --- Normalisation ---
match in all scrub (no-df random-id max-mss 1440)

# --- NAT sortant ---
# Tout ce qui sort vers WAN est NAT'é sur l'IP WAN d'OpenBSD
match out on $wan_if inet from ($vpn_if:network) nat-to ($wan_if:0)
match out on $wan_if inet from ($dir_if:network) nat-to ($wan_if:0)

# --- Politique par défaut : block all ---
block log all

# --- Trafic local/sortant depuis OpenBSD ---
pass out quick keep state

# --- SSH entrant (pour admin depuis atelier, routes internes uniquement) ---
pass in quick on { $vpn_if $dir_if } proto tcp to ($wan_if) port ssh keep state
pass in quick on { $vpn_if $dir_if } proto tcp to self port ssh keep state

# --- Services réseau internes (DHCP et DNS) ---
pass in quick on { $vpn_if $dir_if } proto udp from any port 68 to port 67 keep state
pass in quick on { $vpn_if $dir_if } proto { tcp udp } to self port domain keep state

# --- Routing inter-VM interdit par défaut ---
# br-vpn et br-direct ne peuvent PAS se parler
block quick on $vpn_if from $dir_net
block quick on $dir_if from $vpn_net

# --- Autoriser clients internes à sortir sur Internet via WAN ---
pass in quick on $vpn_if inet from $vpn_net to any keep state
pass in quick on $dir_if inet from $dir_net to any keep state

# --- Ping depuis internes vers OpenBSD ---
pass in quick on { $vpn_if $dir_if } inet proto icmp icmp-type echoreq keep state
```

Charger et vérifier :
```sh
doas pfctl -nf /etc/pf.conf       # parse sans charger (dry-run)
doas pfctl -f /etc/pf.conf         # charger
doas pfctl -si                     # status
doas pfctl -sr                     # règles actives
```

---

## G. Configuration dhcpd (base system OpenBSD)

Éditer `/etc/dhcpd.conf` :
```sh
doas vi /etc/dhcpd.conf
```

Contenu :

```
# /etc/dhcpd.conf — DHCP pour bridges internes

# Globals
option domain-name "farm.lan";
default-lease-time 3600;
max-lease-time 86400;

# --- br-vpn (10.0.10.0/24) ---
subnet 10.0.10.0 netmask 255.255.255.0 {
    option routers 10.0.10.1;
    option domain-name-servers 10.0.10.1;
    # Plage dynamique pour tests/VMs éphémères
    range 10.0.10.100 10.0.10.200;

    # Static leases (à compléter au fur et à mesure de la création des VMs)
    # Exemple :
    # host atelier {
    #     hardware ethernet aa:bb:cc:dd:ee:ff;
    #     fixed-address 10.0.10.50;
    # }
}

# --- br-direct (10.0.20.0/24) ---
subnet 10.0.20.0 netmask 255.255.255.0 {
    option routers 10.0.20.1;
    option domain-name-servers 10.0.20.1;
    range 10.0.20.100 10.0.20.200;
}
```

Spécifier sur quelles interfaces dhcpd écoute (pas sur vio0 !) :
```sh
doas tee /etc/rc.conf.local > /dev/null <<'EOF'
dhcpd_flags="vio1 vio2"
EOF
```

Activer et démarrer :
```sh
doas rcctl enable dhcpd
doas rcctl start dhcpd
doas rcctl check dhcpd
# doit répondre : dhcpd(ok)
```

Si échec : `doas tail -f /var/log/daemon` pour voir les erreurs.

---

## H. Configuration unbound (DNS résolveur local)

Éditer `/var/unbound/etc/unbound.conf` :
```sh
doas vi /var/unbound/etc/unbound.conf
```

Remplacer le contenu par :

```
server:
    # Écouter sur IPs internes + localhost
    interface: 127.0.0.1
    interface: 10.0.10.1
    interface: 10.0.20.1

    # Autoriser les réseaux internes uniquement
    access-control: 127.0.0.0/8 allow
    access-control: 10.0.10.0/24 allow
    access-control: 10.0.20.0/24 allow
    access-control: 0.0.0.0/0 refuse

    # Hygiène
    hide-identity: yes
    hide-version: yes
    minimal-responses: yes
    qname-minimisation: yes
    aggressive-nsec: yes

    # Cache raisonnable
    cache-min-ttl: 300
    cache-max-ttl: 3600

    # Ne pas résoudre des RFC1918 au hasard
    private-address: 10.0.0.0/8
    private-address: 172.16.0.0/12
    private-address: 192.168.0.0/16

    # Logs minimaux
    verbosity: 1

forward-zone:
    name: "."
    # DoT vers Quad9 (temporaire — phase 05 basculera sur Mullvad)
    forward-tls-upstream: yes
    forward-addr: 9.9.9.9@853#dns.quad9.net
    forward-addr: 149.112.112.112@853#dns.quad9.net
```

Activer :
```sh
doas rcctl enable unbound
doas rcctl start unbound
doas rcctl check unbound
# doit répondre : unbound(ok)
```

Test depuis OpenBSD :
```sh
dig @127.0.0.1 voidlinux.org +short
# doit répondre avec une IP
```

Mettre à jour `/etc/resolv.conf` pour que OpenBSD lui-même utilise son unbound :
```sh
doas tee /etc/resolv.conf > /dev/null <<'EOF'
nameserver 127.0.0.1
lookup file bind
EOF
```

---

## I. Test avec une VM jetable

**Sur l'hôte Void**, lancer une Alpine éphémère attachée à `br-vpn` (sans créer de VM persistante) :

```bash
# Télécharger Alpine si pas déjà fait
cd ~/isos/
[ -f alpine-virt-latest.iso ] || \
  curl -L -o alpine-virt-latest.iso \
    "https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/alpine-virt-3.21.0-x86_64.iso"
# (Ajuster la version selon dispo)

# Lancer une VM live temporaire
virt-install \
  --name test-br-vpn \
  --memory 512 --vcpus 1 \
  --cdrom ~/isos/alpine-virt-latest.iso \
  --disk size=2,format=qcow2,path=/tmp/test-br-vpn.qcow2 \
  --network network=br-vpn,model=virtio \
  --os-variant alpinelinux3.21 \
  --graphics spice \
  --noautoconsole

virt-viewer test-br-vpn
```

Dans la console Alpine live (login `root` sans password) :
```sh
# Vérifier que DHCP a marché
ip -br addr show eth0
# doit montrer une IP dans 10.0.10.100-200

# DNS fonctionnel
cat /etc/resolv.conf
# doit contenir 10.0.10.1

# Routing fonctionnel
ping -c 2 10.0.10.1          # OpenBSD router OK
ping -c 2 9.9.9.9             # Internet via OpenBSD NAT OK
nslookup voidlinux.org        # DNS via unbound OK

# Isolation inter-bridge
# (rien à tester ici, pas de VM sur br-direct encore)
```

Si tout est OK → détruire la VM de test :
```bash
virsh destroy test-br-vpn
virsh undefine test-br-vpn --remove-all-storage
rm -f /tmp/test-br-vpn.qcow2
```

Si KO :
- Vérifier sur OpenBSD : `doas tcpdump -i vio1 -n` pendant que le test ping 9.9.9.9
- Vérifier pf : `doas pfctl -sr | head -20` et `doas pfctl -ss | head -20`
- Vérifier IP forwarding : `sysctl net.inet.ip.forwarding`
- Vérifier dhcpd sert bien : `doas rcctl check dhcpd` et `doas tail /var/log/daemon`

---

## J. Migrer la VM atelier sur br-vpn

### Récupérer la MAC de l'atelier

**Sur Void** :
```bash
virsh dumpxml atelier | grep -A1 'source network' | grep mac
# Noter l'adresse MAC aa:bb:cc:...
```

### Déclarer un static lease sur OpenBSD

Sur obsd-router :
```sh
doas vi /etc/dhcpd.conf
```

Ajouter dans le bloc `subnet 10.0.10.0` :
```
    host atelier {
        hardware ethernet AA:BB:CC:DD:EE:FF;   # MAC réelle de l'atelier
        fixed-address 10.0.10.50;
    }
```

Recharger dhcpd :
```sh
doas rcctl restart dhcpd
```

### Changer le réseau de l'atelier

**Sur Void** (atelier éteinte) :
```bash
virsh shutdown atelier
# attendre...

virsh edit atelier
# Modifier l'interface :
#   <source network='default'/> → <source network='br-vpn'/>

virsh start atelier
```

### Vérifier dans l'atelier

Login XFCE atelier → terminal :
```bash
ip -br addr show
# Doit afficher 10.0.10.50/24 sur enp1s0 ou ens3

ping -c 2 10.0.10.1           # OpenBSD router
ping -c 2 9.9.9.9              # Internet via OpenBSD
nslookup debian.org            # DNS via unbound
```

### Mettre à jour ~/.ssh/config dans l'atelier

```bash
sed -i 's|# Hostname 10.0.0.1|Hostname 10.0.0.1|' ~/.ssh/config
sed -i 's|# Hostname 10.0.0.2|Hostname 10.0.10.1|' ~/.ssh/config
# (Adapter voidhv plus tard quand on lui aura donné une IP sur br-vpn, phase 06)
```

À ce stade, l'atelier peut SSH vers obsd-router :
```bash
ssh obsd
# ou : ssh -i ~/.ssh/farm_os_admin TON_USER_OBSD@10.0.10.1
```

---

## K. Snapshot

Snapshot d'obsd-router post-configuration :
```bash
# Sur Void, VM éteinte propre :
virsh shutdown obsd-router
# (ou doas shutdown -hp now depuis l'intérieur)

virsh snapshot-create-as obsd-router \
  --name "phase04-router-complet" \
  --description "OpenBSD routeur : 3 NICs, pf NAT, dhcpd, unbound, IP forwarding"
```

Snapshot atelier (nouvelle config réseau) :
```bash
virsh shutdown atelier

virsh snapshot-create-as atelier \
  --name "phase04-on-br-vpn" \
  --description "Atelier migrée sur br-vpn, 10.0.10.50, SSH vers obsd OK"
```

---

## L. Vérifications finales phase 04

Sur obsd-router (via SSH depuis atelier) :
```sh
# 3 interfaces up
ifconfig | grep -E 'vio[012]' | head -3

# IP forwarding
sysctl net.inet.ip.forwarding

# pf actif + règles chargées
doas pfctl -si | grep Status
doas pfctl -sr | wc -l    # devrait afficher > 10

# dhcpd + unbound tournent
doas rcctl check dhcpd unbound

# OpenBSD lui-même résout via unbound
dig @127.0.0.1 voidlinux.org +short
```

Depuis l'atelier :
```bash
ip -br addr                                       # IP 10.0.10.50
route -n | grep default                           # via 10.0.10.1
nslookup debian.org                               # résolu via 10.0.10.1
ping -c 2 9.9.9.9                                 # sortie OK
ssh obsd uname -a                                 # SSH par clé OK
```

Sur Void :
```bash
virsh net-list --all                              # 4 réseaux actifs
virsh snapshot-list obsd-router                   # 2 snapshots
virsh snapshot-list atelier                       # 2 snapshots
```

---

## Troubleshooting

### L'atelier n'obtient pas d'IP sur br-vpn

- Vérifier que la MAC dans `/etc/dhcpd.conf` correspond bien à celle de la VM : `virsh dumpxml atelier | grep mac`
- Relancer dhcpd : `doas rcctl restart dhcpd`
- Dans l'atelier : `sudo dhclient -v enp1s0` (ou `ens3`)
- Sur OpenBSD : `doas tail -f /var/log/messages` pendant la requête

### Internet inaccessible depuis l'atelier

- Vérifier le NAT pf : `doas pfctl -sn`
- Tracer : `doas tcpdump -ni vio0 icmp` pendant `ping 9.9.9.9` depuis atelier
- Vérifier route par défaut OpenBSD : `route -n show | head -5`
- Vérifier que Void a lui-même internet (wlan0 up)

### Ping fonctionne mais pas DNS

- Unbound écoute sur les bonnes IPs : `netstat -an | grep 53`
- Test direct : `dig @10.0.10.1 voidlinux.org` depuis atelier
- Log unbound : `doas tail /var/log/messages | grep unbound`

### pfctl: "Syntax error" au parse

- Lire l'erreur : la ligne exacte est indiquée
- `doas pfctl -nvf /etc/pf.conf` pour voir le parse verbose
- Bug courant : un nom d'interface mal orthographié, ou `self` non reconnu sur vieilles versions → remplacer par IP explicite

### La VM atelier ne démarre pas après migration br-vpn

- Revenir à l'ancienne conf : `virsh edit atelier` → remettre `default`
- Ou rollback via snapshot : `virsh snapshot-revert atelier phase02-install-clean`
- Puis redémarrer le process de migration pas à pas

---

## État attendu en fin de phase 04

- Bridges libvirt : `br-wan`, `br-vpn`, `br-direct` actifs et persistants
- obsd-router : 3 NICs, IPs 192.168.100.10 / 10.0.10.1 / 10.0.20.1
- pf routant avec NAT, isolation inter-bridge
- dhcpd servant br-vpn et br-direct
- unbound résolveur interne avec DoT Quad9 (upstream)
- Atelier sur br-vpn (10.0.10.50), SSH vers obsd-router via clé
- Une VM jetable sur br-vpn sort sur Internet via OpenBSD avec succès
- Snapshots `phase04-router-complet` et `phase04-on-br-vpn` créés


---

# Phase 05 — Mullvad WireGuard sur obsd-router

**Objectif** : monter sur obsd-router les tunnels WireGuard vers Mullvad (multihop Norvège→Islande pour `br-vpn`), mettre en place le kill-switch pf, et faire pointer unbound vers les DNS Mullvad. `wg1` Paris pour la banque est mentionné mais son activation complète est reportée à la phase 07 (VMs utilisateur). `br-direct` reste direct.

**Durée estimée** : 1-2 h

**Prérequis** :
- Phases 01 à 04 validées
- **Compte Mullvad actif** avec numéro de compte (16 chiffres)
- obsd-router démarrée et accessible via SSH

**Référence archi** : `ARCHITECTURE.md` §2 (VPN Mullvad, kill-switch), §4 (routage par bridge).

---

## A. Générer la clé WireGuard et la déclarer à Mullvad

Sur obsd-router (SSH depuis atelier) :

```sh
# Générer la paire de clés
cd /tmp
umask 077
wg genkey > wg_privkey
wg pubkey < wg_privkey > wg_pubkey

# Afficher la publique pour upload vers Mullvad
cat wg_pubkey
```

Sur le portail Mullvad (depuis ton navigateur, VM atelier ou téléphone) :
1. Login avec ton numéro de compte sur [mullvad.net/account](https://mullvad.net/account)
2. **WireGuard configuration** → **Manage keys**
3. Coller la clé publique affichée ci-dessus → **Add key**
4. Noter l'**IP attribuée** par Mullvad (format `10.64.X.Y/32`) — sera utilisée dans `/etc/hostname.wg0`

Tu peux **aussi** télécharger un fichier `.conf` généré automatiquement par Mullvad (WireGuard configuration → Download configuration files → choisir entry Norvège, exit Islande) et t'en servir comme référence des valeurs à mettre. Pour OpenBSD on n'utilise pas ce `.conf` directement (format wg-quick Linux), on en extrait les champs.

## B. Récupérer endpoint et clé publique du serveur (multihop NO→IS)

Le fichier `.conf` téléchargé depuis Mullvad (ou l'API) contient :
```
[Peer]
PublicKey = <CLE_PUBLIQUE_SERVEUR_NORVÈGE>
AllowedIPs = 0.0.0.0/0
Endpoint = <IP_SERVEUR_NORVÈGE>:<PORT_MULTIHOP_VERS_ISLANDE>
```

Le **port** n'est pas le port WG standard (51820) : c'est un port spécifique qui encode le serveur de sortie (Islande) côté Mullvad.

Alternative programmatique (récupérer la liste des relais) :
```sh
ftp -o /tmp/relays.json https://api.mullvad.net/public/relays/wireguard/v1/
# Parser avec jq (à installer : pkg_add jq)
doas pkg_add jq
# Exemples de requêtes :
jq '.countries[] | select(.code=="no") | .cities[].relays[] | {hostname, ipv4_addr_in, public_key, multihop_port}' /tmp/relays.json | head -30
```

**Note** : les ports multihop dans l'API changent parfois. La méthode la plus fiable reste de télécharger le `.conf` multihop depuis le portail web Mullvad (il génère un endpoint/port valide à l'instant T) et d'y extraire les valeurs.

Variables à noter :
- `WG_PRIVKEY` : contenu de `/tmp/wg_privkey`
- `WG_LOCAL_IP` : ex `10.64.123.45/32` (attribuée par Mullvad)
- `PEER_PUBKEY` : clé publique du serveur d'entrée NO
- `PEER_ENDPOINT` : IP serveur NO
- `PEER_PORT` : port multihop pour sortie IS

## C. Configurer wg0 sur OpenBSD

Créer `/etc/hostname.wg0` :
```sh
doas vi /etc/hostname.wg0
```

Contenu (remplacer les valeurs par celles notées en B) :
```
wgkey <WG_PRIVKEY>
wgpeer <PEER_PUBKEY> \
    wgendpoint <PEER_ENDPOINT> <PEER_PORT> \
    wgaip 0.0.0.0/0
inet 10.64.123.45 255.255.255.255
mtu 1420
up
```

Protéger le fichier (contient la clé privée) :
```sh
doas chmod 600 /etc/hostname.wg0
```

Démarrer le tunnel :
```sh
doas sh /etc/netstart wg0
```

Vérifier :
```sh
ifconfig wg0
# Doit afficher : UP, inet 10.64.X.Y, wgpeer...

# Handshake réussi ?
doas wg show
# Doit afficher "latest handshake" < 3 minutes ago

# Test connectivité via tunnel
ping -I wg0 -c 3 10.64.0.1           # Mullvad DNS interne, via tunnel
curl --interface wg0 -s https://am.i.mullvad.net/connected
# Doit répondre "You are connected to Mullvad" et mentionner l'exit Islande
```

Si le handshake ne se fait pas : voir **Troubleshooting** en bas.

## D. wg1 (Paris) — structure préparée, activation en phase 07

On prépare le fichier mais on ne l'active que plus tard (la VM banque n'existe pas encore) :

```sh
doas vi /etc/hostname.wg1
```

```
wgkey <WG_PRIVKEY_PARIS>          # peut être la même clé ou en regénérer une
wgpeer <PEER_PUBKEY_PARIS> \
    wgendpoint <PEER_ENDPOINT_PARIS> 51820 \
    wgaip 0.0.0.0/0
inet 10.64.X.Z 255.255.255.255
mtu 1420
!# Activé en phase 07 — retirer le # pour activer
```

Même chose : `chmod 600`. On y revient en phase 07.

## E. pf : kill-switch et routage br-vpn via wg0

Éditer `/etc/pf.conf` :
```sh
doas vi /etc/pf.conf
```

**Remplacer** le contenu de phase 04 par la version avec WG :

```pf
# /etc/pf.conf — phase 05 (VPN Mullvad pour br-vpn, br-direct reste direct)

# --- Interfaces ---
wan_if  = "vio0"
vpn_if  = "vio1"
dir_if  = "vio2"
vpn_out = "wg0"          # tunnel Mullvad NO->IS

# --- Réseaux ---
vpn_net  = "10.0.10.0/24"
dir_net  = "10.0.20.0/24"
void_ip  = "192.168.100.1"    # Void host côté br-wan

# --- Options ---
set skip on lo
set block-policy drop
set loginterface $wan_if

# --- Normalisation ---
match in all scrub (no-df random-id max-mss 1440)

# --- NAT sortant ---
# br-vpn sort via wg0 (Mullvad)
match out on $vpn_out inet from $vpn_net nat-to ($vpn_out:0)
# br-direct sort via wan direct
match out on $wan_if  inet from $dir_net nat-to ($wan_if:0)

# --- Politique par défaut ---
block log all

# --- Trafic local et sortant depuis OpenBSD ---
pass out quick keep state

# --- Services internes (DHCP, DNS) ---
pass in quick on { $vpn_if $dir_if } proto udp from any port 68 to port 67 keep state
pass in quick on { $vpn_if $dir_if } proto { tcp udp } to self port domain keep state
pass in quick on { $vpn_if $dir_if } proto tcp to self port ssh keep state
pass in quick on { $vpn_if $dir_if } inet proto icmp icmp-type echoreq keep state

# --- Isolation inter-bridges ---
block quick on $vpn_if from $dir_net
block quick on $dir_if from $vpn_net

# --- br-vpn : accès local à Void host autorisé AVANT kill-switch ---
pass in quick on $vpn_if inet from $vpn_net to $void_ip keep state

# --- br-vpn : TOUT le reste doit passer par wg0 (kill-switch) ---
pass in quick on $vpn_if inet from $vpn_net to any route-to $vpn_out keep state

# --- br-vpn : bloquer toute sortie qui n'a pas été routée via wg0 (fail-closed) ---
block quick on $wan_if from $vpn_net

# --- br-direct : sortie clearnet normale ---
pass in quick on $dir_if inet from $dir_net to any keep state
```

Charger :
```sh
doas pfctl -nf /etc/pf.conf        # dry-run
doas pfctl -f /etc/pf.conf
doas pfctl -si | grep Status
```

**Test du kill-switch** (optionnel mais recommandé) :
```sh
# Depuis l'atelier (sur br-vpn) : vérifier qu'Internet marche
# puis sur obsd-router, couper wg0 :
doas ifconfig wg0 down
# Depuis l'atelier : ping 9.9.9.9 doit ÉCHOUER (plus de route valide)
# Remonter wg0 :
doas ifconfig wg0 up
doas sh /etc/netstart wg0
# Internet revient
```

## F. Unbound : forward vers DNS Mullvad via wg0

Mullvad expose un résolveur DNS privé accessible **uniquement via le tunnel** à `10.64.0.1`. Cela évite toute fuite DNS vers un résolveur externe.

Éditer `/var/unbound/etc/unbound.conf` :
```sh
doas vi /var/unbound/etc/unbound.conf
```

Remplacer le bloc `forward-zone` par :

```
forward-zone:
    name: "."
    forward-addr: 10.64.0.1
    # Mullvad DNS — en clair dans le tunnel WG (pas de DoT nécessaire,
    # la confidentialité vient du tunnel lui-même)
```

Recharger :
```sh
doas rcctl restart unbound
doas rcctl check unbound

# Test : depuis OpenBSD
dig @127.0.0.1 voidlinux.org +short
# Doit répondre. Si timeout : wg0 peut-être down.
```

**Note** : DNS pour br-direct passe aussi par Mullvad DNS (via le fait que unbound→10.64.0.1 sort via wg0 côté OpenBSD lui-même). C'est un compromis acceptable : la résolution est privée, la connexion TCP/HTTPS des VMs br-direct reste elle en direct. Si tu veux absolument séparer, voir section **Alternative DNS séparée** plus bas.

## G. Test depuis une VM sur br-vpn

Depuis l'atelier :
```bash
# Vérifier que ton IP publique vue depuis Internet est celle de Mullvad Islande
curl -s https://am.i.mullvad.net/connected
# Doit afficher "You are connected to Mullvad (...) Exit: Reykjavik, Iceland..."

curl -s https://ifconfig.me
# Doit afficher une IP Islandaise (Mullvad IS)

# Latence raisonnable (multihop NO->IS ajoute ~30-80ms)
ping -c 5 9.9.9.9

# DNS via Mullvad (pas de leak)
dig voidlinux.org @10.0.10.1 +short
```

Test de fuite DNS : [dnsleaktest.com](https://dnsleaktest.com) depuis un navigateur dans la VM atelier → doit lister uniquement un résolveur Mullvad.

## H. Snapshot

```bash
# Sur Void
virsh shutdown obsd-router
virsh snapshot-create-as obsd-router \
  --name "phase05-wg-mullvad" \
  --description "WireGuard Mullvad wg0 multihop NO->IS actif, pf kill-switch, unbound via Mullvad DNS"
```

## I. Vérifications finales phase 05

Sur obsd-router :
```sh
ifconfig wg0 | grep UP                # UP présent
doas wg show | grep handshake         # récent (< 3 min)
doas pfctl -si | grep Status          # Enabled
doas pfctl -sr | grep wg0 | head -3   # règles route-to wg0
dig @127.0.0.1 voidlinux.org +short   # résolu
```

Depuis l'atelier :
```bash
curl -s https://am.i.mullvad.net/connected | head -1   # Connected via Mullvad
curl -s https://ifconfig.me                             # IP islandaise
```

Test kill-switch (facultatif) :
```sh
# Sur obsd-router :
doas ifconfig wg0 down
# Atelier : plus d'accès Internet (timeouts)
doas sh /etc/netstart wg0
# Atelier : Internet revient
```

---

## Alternative : DNS séparés br-vpn / br-direct

Si tu veux que `br-direct` garde un résolveur qui ne dépend pas de Mullvad (ex: DoT Quad9 via WAN), deux options :

### Option 1 — Deux instances unbound
Compliqué à mettre en place proprement sur OpenBSD (unbound n'est pas multi-instance natif via rc).

### Option 2 — Un seul unbound, mais double `forward-zone` avec tags (views)
Unbound supporte des "views" conditionnelles par client source :

```
server:
    # ... (config commune) ...

view:
    name: "vpn-clients"
    view-first: yes
    forward-zone:
        name: "."
        forward-addr: 10.64.0.1       # via wg0

view:
    name: "direct-clients"
    view-first: yes
    forward-zone:
        name: "."
        forward-tls-upstream: yes
        forward-addr: 9.9.9.9@853#dns.quad9.net

access-control: 10.0.10.0/24 allow
access-control-view: 10.0.10.0/24 vpn-clients
access-control: 10.0.20.0/24 allow
access-control-view: 10.0.20.0/24 direct-clients
```

À creuser si tu veux cette granularité.

---

## Troubleshooting

### wg0 ne fait pas de handshake

```sh
# Test connectivité de base vers l'endpoint Mullvad
ping <PEER_ENDPOINT>

# Port UDP accessible depuis Void ? (NAT libvirt fait passer)
# Vérifier les logs :
doas dmesg | grep -i wg

# Regénérer la clé et la redéclarer sur le portail Mullvad (la précédente est peut-être
# expirée ou la clé locale ne correspond pas à celle uploadée)
```

### Handshake OK mais pas d'Internet via wg0

```sh
# Vérifier qu'il y a bien une route par défaut via wg0 ou que pf route-to fonctionne
doas route -n show | grep wg0

# Sniffer le trafic sortant
doas tcpdump -ni wg0 icmp &
ping -I wg0 10.64.0.1
```

### "am.i.mullvad.net" dit "not connected"

- Vérifier que la VM qui teste (atelier) est bien sur br-vpn : `ip -br addr`
- Vérifier les règles pf `route-to wg0` : `doas pfctl -sr | grep route-to`
- Vérifier que wg0 est UP : `ifconfig wg0 | head -3`

### Latence énorme ou packet drops

- Multihop NO→IS : ~30-80ms attendu, RTT < 200ms OK
- Si beaucoup plus : changer le serveur d'entrée (autre ville norvégienne)
- MTU : tester avec `ping -M do -s 1400 -c 3 9.9.9.9`, ajuster `mtu 1380` dans hostname.wg0 si fragmentation

### Kill-switch ne bloque pas

- Règles pf mal ordonnées : le `block quick on wan_if from vpn_net` doit venir APRÈS les `pass route-to wg0`
- `pass route-to` continue à matcher des paquets même si l'interface cible est down → mais ils sont dropés par route-to failure. Si fuite, ajouter explicitement `block return quick on $wan_if from $vpn_net` (avec `quick` en priorité haute)

### DNS timeout

- wg0 down → unbound ne peut plus forwarder → timeout attendu (cohérent avec le kill-switch)
- Pour avoir un fallback (moins sûr) : ajouter un second `forward-addr` vers Quad9 en DoT, mais ça crée une fuite potentielle si wg0 down

---

## État attendu en fin de phase 05

- `wg0` UP, handshake Mullvad récent, tunnel NO→IS fonctionnel
- `hostname.wg1` préparé mais inactif (attend phase 07)
- pf route br-vpn via wg0 avec kill-switch fail-closed
- br-direct sort toujours en direct via `vio0` WAN
- Unbound forward vers DNS Mullvad `10.64.0.1` (via wg0)
- Atelier voit Internet depuis une IP islandaise, pas de leak DNS
- Snapshot `phase05-wg-mullvad` créé


---

# Phase 06 — Accès distant : WireGuard inbound + SSH + virt-manager remote

**Objectif** : pouvoir piloter le homelab depuis n'importe où. Une interface WireGuard "inbound" sur obsd-router accepte les connexions depuis l'extérieur (un seul port UDP exposé). Une fois dans le tunnel, SSH vers OpenBSD + ProxyJump vers Void + virt-manager qemu+ssh donne un accès graphique complet à toutes les VMs. SSH sur Void est durci et ne binde que sur une IP interne.

**Durée estimée** : 1-2 h

**Prérequis** :
- Phases 01 à 05 validées
- Accès admin à ta box ISP (pour port forwarding)
- Un client WireGuard sur ton laptop / téléphone d'usage distant
- Si CGNAT (pas d'IP publique) : voir section **Cas CGNAT** en fin

**Référence archi** : `ARCHITECTURE.md` §9 (Accès distant), §1 (SSH sur Void bindé interne).

---

## A. Plan de l'accès distant

```
[Laptop/phone dehors]
      |
      | UDP 51820 (WireGuard, seul port exposé Internet)
      V
[Box ISP] --- port forward UDP 51820 ---> [Void wlan0] --- DNAT ---> [OpenBSD 192.168.100.10]
                                                                             |
                                                                        wg_inbound
                                                                       (10.99.0.1/24)
                                                                             |
[Tu es maintenant 10.99.0.2] ---ssh---> obsd
                             ---ssh via ProxyJump---> voidhv (192.168.100.1)
                             ---ssh via ProxyJump Void---> VMs
                             ---virt-manager qemu+ssh://voidhv/system---> GUI complète
```

Variables à noter pour la suite :
- `WG_IN_PORT=51820` (le port UDP qu'on exposera)
- `WG_IN_NET=10.99.0.0/24` (le subnet du tunnel inbound)
- `WG_IN_SERVER=10.99.0.1` (OpenBSD côté tunnel)
- `WG_IN_CLIENT=10.99.0.2` (toi, côté tunnel)

## B. Générer les clés WireGuard pour le tunnel inbound

Sur obsd-router :
```sh
cd /tmp
umask 077

# Clés serveur (OpenBSD)
wg genkey > wg_in_server_priv
wg pubkey < wg_in_server_priv > wg_in_server_pub

# Clés client (ton laptop)
wg genkey > wg_in_client_priv
wg pubkey < wg_in_client_priv > wg_in_client_pub

# Pre-shared key (optionnel mais recommandé : protection post-quantum)
wg genpsk > wg_in_psk

# Afficher pour copie
echo "=== SERVER PRIVATE ==="
cat wg_in_server_priv
echo "=== SERVER PUBLIC ==="
cat wg_in_server_pub
echo "=== CLIENT PRIVATE ==="
cat wg_in_client_priv
echo "=== CLIENT PUBLIC ==="
cat wg_in_client_pub
echo "=== PSK ==="
cat wg_in_psk
```

**Noter toutes les valeurs**. Les clés privées ne sortent jamais de la machine qui en a besoin (serveur OU client, pas les deux). La clé publique du client va dans la config serveur, et inversement.

## C. Configurer wg_inbound sur OpenBSD

Créer `/etc/hostname.wg1` — attention, `wg1` était réservé pour Mullvad Paris en phase 05. Utilisons plutôt **`wg2`** pour l'inbound, pour éviter tout conflit.

```sh
doas vi /etc/hostname.wg2
```

```
wgkey <WG_IN_SERVER_PRIV>
wgport 51820
wgpeer <WG_IN_CLIENT_PUB> \
    wgpsk <WG_IN_PSK> \
    wgaip 10.99.0.2/32
inet 10.99.0.1 255.255.255.0
mtu 1420
up
```

Protéger :
```sh
doas chmod 600 /etc/hostname.wg2
```

Activer :
```sh
doas sh /etc/netstart wg2
ifconfig wg2 | head -5
# Doit montrer inet 10.99.0.1 et listen port 51820
```

## D. Règles pf pour accepter le trafic inbound et autoriser le tunnel

Éditer `/etc/pf.conf` et ajouter :

```pf
# --- Interfaces additionnelles (phase 06) ---
wg_in = "wg2"
wg_in_net = "10.99.0.0/24"
void_ip  = "192.168.100.1"
void_ssh_port = "22"

# --- Accepter le port WG inbound sur WAN ---
pass in quick on $wan_if inet proto udp to port 51820 keep state

# --- Tunnel inbound : autoriser l'accès aux ressources internes ---
# SSH vers OpenBSD depuis le tunnel
pass in quick on $wg_in proto tcp from $wg_in_net to self port ssh keep state

# SSH vers Void via OpenBSD (ProxyJump)
pass in quick on $wg_in proto tcp from $wg_in_net to $void_ip port $void_ssh_port keep state

# SSH vers VMs internes (br-vpn, br-direct)
pass in quick on $wg_in proto tcp from $wg_in_net to { $vpn_net $dir_net } port ssh keep state

# Ping des ressources internes
pass in quick on $wg_in inet proto icmp from $wg_in_net keep state

# Forward IP pour routage tunnel ↔ internes
pass out quick on { $vpn_if $dir_if } from $wg_in_net keep state
pass out quick on $wan_if from $wg_in_net to $void_ip keep state
```

**Important** : ces règles sont à ajouter **avant** la section de policy générale. Structure ordonnée suggérée :
1. Options, scrub
2. NAT
3. Block all (default)
4. Pass out local
5. Services internes (DHCP, DNS)
6. **wg inbound (nouvelles règles ici)**
7. Règles br-vpn / br-direct
8. Isolation inter-bridges

Dry-run et charger :
```sh
doas pfctl -nf /etc/pf.conf
doas pfctl -f /etc/pf.conf
```

## E. Port forwarding sur Void (DNAT wlan0 → OpenBSD)

Le port UDP 51820 arrive sur Void via la box ISP. Il faut le rediriger vers OpenBSD (192.168.100.10).

Installer nftables (normalement déjà là) :
```bash
sudo xbps-install -y nftables
```

Créer `/etc/nftables.conf` :
```bash
sudo tee /etc/nftables.conf > /dev/null <<'EOF'
#!/usr/sbin/nft -f

flush ruleset

table inet nat {
  chain prerouting {
    type nat hook prerouting priority dstnat;
    iifname "wlan0" udp dport 51820 dnat to 192.168.100.10:51820
  }
  chain postrouting {
    type nat hook postrouting priority srcnat;
    oifname "wlan0" masquerade
  }
}

table inet filter {
  chain forward {
    type filter hook forward priority filter; policy accept;
    iifname "wlan0" oifname "virbr-wan" udp dport 51820 accept
  }
}
EOF

sudo chmod 600 /etc/nftables.conf
```

**Note** : libvirt gère déjà une partie du MASQUERADE pour ses réseaux NAT. Vérifier qu'il n'y a pas de conflit :
```bash
sudo nft list ruleset | head -40
```

Activer au boot (service runit custom) :
```bash
sudo mkdir -p /etc/sv/nftables
sudo tee /etc/sv/nftables/run > /dev/null <<'EOF'
#!/bin/sh
exec /usr/sbin/nft -f /etc/nftables.conf
EOF
sudo chmod +x /etc/sv/nftables/run

# Activer
sudo ln -sf /etc/sv/nftables /var/service/
sleep 3
sudo sv status nftables
```

Charger immédiatement :
```bash
sudo nft -f /etc/nftables.conf
sudo nft list ruleset | grep 51820
```

## F. Port forwarding sur la Box ISP

Interface admin de ta box (IP type `192.168.1.1` ou `192.168.0.1`, login admin) :
- Section "NAT", "Port forwarding", ou "Redirection de ports"
- Ajouter une règle :
  - **Protocol** : UDP
  - **External port** : 51820
  - **Internal IP** : IP de ta machine Void sur ton LAN domestique (ex: `192.168.1.42`)
  - **Internal port** : 51820
- Sauvegarder, appliquer

**Vérifier** depuis l'extérieur (après quelques minutes) :
```bash
# Depuis un réseau différent (4G phone en hotspot par exemple) :
nc -u -v -z <TON_IP_PUBLIQUE> 51820
# Ou : yougetsignal.com / canyouseeme.org (port UDP test)
```

## G. Config client WireGuard (laptop/phone)

Sur ton laptop (Linux/macOS/Windows) ou phone (Android/iOS avec l'app WireGuard) :

```
[Interface]
PrivateKey = <WG_IN_CLIENT_PRIV>
Address = 10.99.0.2/24
DNS = 10.0.10.1       # unbound sur obsd-router (DNS via Mullvad, privacy)

[Peer]
PublicKey = <WG_IN_SERVER_PUB>
PresharedKey = <WG_IN_PSK>
Endpoint = <TON_IP_PUBLIQUE_OU_DYNDNS>:51820
AllowedIPs = 10.0.0.0/8, 192.168.100.0/24
# AllowedIPs = 0.0.0.0/0    # pour full tunnel si tu veux tout le trafic via le tunnel
PersistentKeepalive = 25
```

Sauvegarder comme `farm-home.conf`, importer dans WireGuard, activer.

**Pour IP publique dynamique** : utiliser un DynDNS (DuckDNS, No-IP, ou un script cron qui met à jour un record DNS). L'`Endpoint` peut être un hostname.

## H. Installer et durcir sshd sur Void

```bash
sudo xbps-install -y openssh
```

Générer les host keys (si pas auto-créés) :
```bash
sudo ssh-keygen -A
```

Configuration `/etc/ssh/sshd_config` (Void) — remplacer/régler :
```bash
sudo tee /etc/ssh/sshd_config.d/farm-os.conf > /dev/null <<'EOF'
# Bind interne uniquement (jamais sur wlan0)
ListenAddress 192.168.100.1

# Auth
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers TON_USER_VOID

# Durcissement
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
UsePAM yes

# Algos modernes
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
HostKeyAlgorithms ssh-ed25519
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com
EOF
```

**Avant d'activer** : copier ta clé publique sur Void. Depuis l'atelier :
```bash
ssh-copy-id -i ~/.ssh/farm_os_admin.pub TON_USER_VOID@192.168.100.1
# Si route-to wg0 bloque cette connexion, temporairement la pass à travers
# (ou utiliser la console virt-manager pour copier la clé manuellement)
```

Activer le service runit :
```bash
sudo ln -sf /etc/sv/sshd /var/service/
sleep 3
sudo sv status sshd
# Tester binding :
ss -tlnp | grep ':22'
# Doit montrer 192.168.100.1:22 UNIQUEMENT (pas 0.0.0.0)
```

## I. Compléter ~/.ssh/config dans l'atelier

Depuis l'atelier, éditer `~/.ssh/config` :

```ssh
# farm-os — config finale phase 06

Host obsd
    Hostname 10.0.10.1
    User TON_USER_OBSD
    IdentityFile ~/.ssh/farm_os_admin
    IdentitiesOnly yes

Host voidhv
    Hostname 192.168.100.1
    User TON_USER_VOID
    IdentityFile ~/.ssh/farm_os_admin
    IdentitiesOnly yes
    ProxyJump obsd

# VMs internes via ProxyJump voidhv
Host vm-*
    User TON_USER_DEFAUT
    IdentityFile ~/.ssh/farm_os_admin
    IdentitiesOnly yes
    ProxyJump voidhv
```

Test depuis l'atelier :
```bash
ssh obsd hostname          # → obsd-router
ssh voidhv hostname        # → voidhv (via ProxyJump)
```

## J. Même ~/.ssh/config sur le laptop (pour accès distant)

Sur ton laptop (après connexion WireGuard au tunnel farm-home) :

```ssh
# farm-os — client distant

Host obsd
    Hostname 10.99.0.1
    User TON_USER_OBSD
    IdentityFile ~/.ssh/farm_os_admin
    IdentitiesOnly yes

Host voidhv
    Hostname 192.168.100.1
    User TON_USER_VOID
    IdentityFile ~/.ssh/farm_os_admin
    IdentitiesOnly yes
    ProxyJump obsd

Host vm-*
    User TON_USER_DEFAUT
    IdentityFile ~/.ssh/farm_os_admin
    IdentitiesOnly yes
    ProxyJump voidhv
```

**Note** : la clé privée `~/.ssh/farm_os_admin` doit être copiée sur le laptop (via clé USB ou scp sécurisé). **Pour le max de sécurité, utiliser une YubiKey** :
```bash
# Générer sur la YubiKey :
ssh-keygen -t ed25519-sk -O resident -f ~/.ssh/farm_os_yubikey -C "farm-os-yk"
# Copier la publique sur tous les serveurs, la privée reste dans la YubiKey
```

## K. Test virt-manager remote (depuis l'atelier)

Dans l'atelier, lancer :
```bash
virt-manager -c qemu+ssh://TON_USER_VOID@voidhv/system
```

La fenêtre doit s'ouvrir, afficher la liste des VMs (obsd-router, atelier — ou plus selon l'avancement). Double-clic sur une VM → console SPICE s'ouvre.

**Depuis le laptop distant** (après avoir activé le tunnel WG) :
```bash
virt-manager -c qemu+ssh://TON_USER_VOID@voidhv/system
```

Même expérience, tunnelée dans SSH, tunnelé dans WireGuard.

## L. Snapshot

```bash
# Sur Void
virsh shutdown obsd-router
virsh snapshot-create-as obsd-router \
  --name "phase06-remote-access" \
  --description "wg_inbound (wg2) actif, pf autorise tunnel, SSH Void binding interne, virt-manager remote testé"
```

## M. Vérifications finales phase 06

Sur obsd-router :
```sh
ifconfig wg2 | head -3                            # UP, 10.99.0.1
doas wg show wg2                                  # liste des peers
doas pfctl -sr | grep 51820                       # règle d'acceptation port WG
```

Sur Void :
```bash
sudo ss -tlnp | grep ':22'                        # binding 192.168.100.1 uniquement
sudo nft list ruleset | grep 51820                # DNAT actif
```

Depuis l'atelier :
```bash
ssh obsd hostname
ssh voidhv hostname
virt-manager -c qemu+ssh://TON_USER@voidhv/system  # lance la GUI
```

Depuis le laptop distant (en hotspot 4G par exemple) :
1. Activer WireGuard (farm-home)
2. `ping 10.99.0.1` répond
3. `ssh obsd hostname` répond
4. `virt-manager -c qemu+ssh://TON_USER@voidhv/system` ouvre la GUI avec les VMs visibles

---

## Cas CGNAT (pas d'IP publique)

Si ton FAI utilise le CGNAT (pas d'IP publique fixe, impossible de port-forward) :

**Option 1 — Mullvad port forwarding** : Mullvad offre le port forwarding sur comptes payants. Tu activates le port dans ton compte, et les connexions entrantes vers ce port (via l'IP de sortie Mullvad) arrivent dans ton tunnel Mullvad. Complique l'archi parce qu'il faut monter un tunnel Mullvad dédié pour l'inbound (à côté du multihop NO→IS). Faisable mais non trivial.

**Option 2 — VPS relais** (recommandé) : louer un VPS à 3-5€/mois (Hetzner CX11, Contabo, Scaleway). Sur le VPS :
- WireGuard avec IP publique fixe
- Un tunnel WG "ssortant" de chez toi vers le VPS
- Port forwarding sur le VPS : UDP 51820 public → UDP 51820 local (tunnel vers chez toi) → obsd-router

Archi plus riche qu'il faut documenter à part si tu empruntes cette voie — demandera un `06-bis-cgnat-vps.md` dédié si besoin.

**Option 3 — Tailscale/Headscale** : mesh VPN qui gère naturellement le NAT traversal. Headscale (backend self-hosted) sur un VPS + tailscaled sur tes devices. Moins "DIY pur" mais réduit énormément la complexité NAT.

---

## Troubleshooting

### Le client WG ne handshake pas avec l'extérieur

- Tester depuis la maison (même LAN) : sur atelier, importer la config client, se connecter vers `192.168.100.10:51820` — doit marcher. Confirme que la stack WG est OK.
- Tester port forwarding box : depuis un téléphone en 4G, `nc -zuv <IP_PUB> 51820`
- Inspecter sur Void : `sudo tcpdump -i wlan0 'udp port 51820'` pendant tentative client
- Inspecter sur OpenBSD : `doas tcpdump -i vio0 'udp port 51820'`
- Vérifier DNAT : `sudo nft list ruleset | grep dnat`

### SSH vers voidhv depuis atelier échoue

- Route OpenBSD vers 192.168.100.1 : `route -n show | grep 192.168.100`
- pf règle br-vpn → void_ip : `doas pfctl -sr | grep void_ip`
- Void bind SSH : `sudo ss -tlnp | grep 22` (doit montrer 192.168.100.1)
- `ssh -vvv voidhv` pour voir où ça bloque

### virt-manager remote affiche "authentication failed"

- User Void doit être dans groupe `libvirt` : `ssh voidhv groups`
- SSH agent forwarding ou clé correcte : `ssh-add -l`
- Test `virsh -c qemu+ssh://voidhv/system list` depuis l'atelier → mêmes logs

### SPICE console noire ou déconnexions

- virt-manager utilise `-4` par défaut ; si tu as désactivé IPv4 quelque part, forcer
- Upgrade virt-viewer sur le client : `sudo apt install --reinstall virt-viewer`
- Essayer VNC à la place : `virsh edit <vm>` → `<graphics type='vnc'/>`

---

## État attendu en fin de phase 06

- Interface `wg2` UP sur obsd-router, port UDP 51820 accepte clients autorisés
- Port 51820 UDP forwardé de la box ISP vers Void vers OpenBSD
- SSHd sur Void binde `192.168.100.1`, pas wlan0
- SSH par clé partout : atelier → obsd, atelier → voidhv (ProxyJump), atelier → VMs
- Même chose depuis un laptop distant via WireGuard farm-home
- `virt-manager -c qemu+ssh://voidhv/system` affiche la GUI depuis atelier ET depuis distant
- Snapshot `phase06-remote-access` créé


---

## Suite

Passer à `APPLICATIVES.md` pour le déploiement des VMs applicatives (Routine, Dev/Claude, Banque, Bitcoin-node/Sparrow, Windows 11), l'intégration Whonix, les passerelles Tor-gw / I2P-gw, la VM anon-browse (Kicksecure), et le système de VMs éphémères torifiées.
