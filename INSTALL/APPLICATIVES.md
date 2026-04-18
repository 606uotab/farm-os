# farm-os — Installation applicatives

Seconde partie du guide d'installation : déploiement des VMs qui donnent réellement de l'usage au homelab. Suit `BASE.md` qui a posé l'hôte, le routeur, les bridges, le VPN et l'accès distant.

À la fin de ce document :

- VMs applicatives : **Routine**, **Dev/Claude**, **Banque** (avec `wg1` Mullvad Paris activé), **Bitcoin-node** + **Sparrow**, **Windows 11**
- **Whonix-GW** + **Whonix-WS** sur `br-whonix`
- Seconde VM OpenBSD **Tor-gw** avec transproxy pour torifier des VMs non-navigateurs
- Troisième VM OpenBSD **I2P-gw** avec i2pd
- **anon-browse** (Kicksecure + Tor Browser + Chromium-I2P) sur `br-browse`
- Template **Alpine** + scripts `spawn-torbox.sh` / `spawn-i2pbox.sh` / `spawn-directbox.sh` pour VMs éphémères

## Conventions (rappel)

- Blocs de code = copy-paste direct (sauf mention `[côté X]`).
- Toutes les opérations VM se font depuis l'atelier via `virt-manager -c qemu+ssh://voidhv/system` ou en CLI via `virsh --connect qemu+ssh://user@voidhv/system`.
- Chaque nouvelle VM : static lease dhcpd ajouté sur obsd-router, clé SSH déposée, `~/.ssh/config` mis à jour, snapshot de référence créé.
- Politique clipboard/audio : voir `ARCHITECTURE.md` §10 et §11, appliquée par VM.

## Plan des phases

| # | Objectif | Durée |
|---|---|---|
| 0 | Préambule : trousseau GPG + politique de chiffrement qcow2 | 30 min |
| 7 | VMs utilisateur (Routine, Dev, Banque, Bitcoin, Sparrow, Windows) | 3-4 h |
| 8 | Whonix-GW + Whonix-WS | 1 h |
| 9 | Tor-gw OpenBSD (transproxy) | 1-2 h |
| 10 | I2P-gw OpenBSD (i2pd) | 1 h |
| 11 | anon-browse (Kicksecure + TBB + Chromium-I2P) | 1 h |
| 12 | Template Alpine + scripts spawn-* | 1 h |

---

# Phase 00 — Préambule : trousseau GPG et chiffrement qcow2

**Objectif** : poser l'infrastructure cryptographique utilisée par toutes les VMs suivantes. Une clé GPG protège un trousseau `pass` qui stocke les passphrases qcow2-LUKS. Au boot Void, un script `unlock-farm.sh` déverrouille le trousseau (une interaction utilisateur) puis démarre les VMs.

**Modèle de menace** : voir `ARCHITECTURE.md` section « Chiffrement et modèle de menace ». Protection contre fuites numériques de qcow2 (backups, clones), pas contre bruteforce offline infaisable d'une passphrase GPG forte.

**Politique de chiffrement** :

| VM | qcow2-LUKS ? |
|---|---|
| atelier | ❌ (reste en clair, sert de cockpit à distance) |
| Toutes les autres (obsd-router, obsd-torgw, obsd-i2pgw, routine, dev, banque, bitcoin-node, sparrow, Whonix-GW, Whonix-WS, anon-browse, Windows 11) | ✅ |
| Templates et VMs éphémères | ❌ (jetables par nature) |

## 0.A. Générer la clé GPG sur Void

```bash
# [sur voidhv, user normal]
gpg --full-generate-key
```

Réponses :
- Type : `(9) ECC (sign and encrypt)` (ou `(1) RSA and RSA` et `4096` si l'installer GPG de Void ne propose pas ECC)
- Curve : `Curve 25519`
- Validity : `0` (jamais expire)
- Nom : `farm-os keyring`
- Mail : `farm-keyring@local` (ou équivalent, n'a pas besoin d'être réel)
- **Passphrase** : **forte**, gérée par tes soins — type diceware 8 mots ou équivalent 100+ bits d'entropie. **À ne jamais inscrire numériquement**.

Récupérer l'ID de la clé :
```bash
gpg --list-secret-keys --keyid-format long
# Noter la ligne "sec   ed25519/XXXXXXXXXXXXXXXX" — XX... est ton KEYID
```

## 0.B. Initialiser pass

```bash
pass init <KEYID>
# Crée ~/.password-store/, chiffré par ta clé GPG
pass ls
# Doit afficher : Password Store (vide)
```

## 0.C. Configurer GPG pour un KDF solide

Éditer `~/.gnupg/gpg.conf` :
```bash
cat >> ~/.gnupg/gpg.conf <<'EOF'
personal-digest-preferences SHA512
cert-digest-algo SHA512
s2k-digest-algo SHA512
s2k-cipher-algo AES256
s2k-mode 3
s2k-count 65011712
EOF
```

Régénérer la clé avec le nouveau KDF (sinon les options ci-dessus ne s'appliqueront qu'aux nouvelles clés) :
```bash
gpg --edit-key <KEYID>
gpg> passwd
# Re-taper la passphrase, GPG ré-encode la clé avec le nouveau KDF
gpg> save
```

## 0.D. Backup paperkey de la clé privée

**Une seule fois**, sur papier, dans un endroit physique sûr (coffre, chez un proche de confiance) :

```bash
sudo xbps-install -y paperkey
gpg --export-secret-keys --armor <KEYID> | paperkey --output /tmp/farm-paperkey.txt

# Imprimer /tmp/farm-paperkey.txt
# Ranger le papier, PUIS :
shred -uvz /tmp/farm-paperkey.txt
```

**Si tu perds le disque ET la passphrase** : les VMs chiffrées sont perdues. Si tu as le papier + la passphrase, tu peux tout reconstituer.

## 0.E. Procédure générique : chiffrer un qcow2

Cette procédure se répète pour **chaque VM à chiffrer**. À utiliser après l'installation de la VM (install en clair, snapshot, puis chiffrement du qcow2 après arrêt).

### Pour une VM existante (post-install)

Arrêter la VM :
```bash
virsh shutdown <VM_NAME>
```

Générer passphrase + secret libvirt :
```bash
UUID=$(uuidgen)
PASS=$(openssl rand -base64 32)

# Stocker dans pass (chiffré par GPG)
echo -n "$PASS" | pass insert -m -e "farm/<VM_NAME>"
# Saisir le password noté par openssl, Ctrl-D pour finir

# Définir le secret libvirt (ephemeral : jamais persisté sur disque)
cat > /tmp/${UUID}-secret.xml <<EOF
<secret ephemeral='yes' private='yes'>
  <uuid>${UUID}</uuid>
  <description>LUKS passphrase for <VM_NAME>.qcow2</description>
  <usage type='volume'>
    <volume>/var/lib/libvirt/images/<VM_NAME>.qcow2</volume>
  </usage>
</secret>
EOF
sudo virsh secret-define /tmp/${UUID}-secret.xml
rm /tmp/${UUID}-secret.xml

# Injecter la valeur (mémoire uniquement, secret ephemeral)
echo -n "$PASS" | base64 | sudo virsh secret-set-value ${UUID} --base64 /dev/stdin
```

Convertir le qcow2 en blob LUKS :
```bash
sudo qemu-img convert -p \
  --object secret,id=sec0,data="$PASS" \
  -O qcow2 \
  -o encrypt.format=luks,encrypt.key-secret=sec0 \
  /var/lib/libvirt/images/<VM_NAME>.qcow2 \
  /var/lib/libvirt/images/<VM_NAME>-enc.qcow2

unset PASS   # important !

sudo mv /var/lib/libvirt/images/<VM_NAME>.qcow2 /var/lib/libvirt/images/<VM_NAME>-plain.bak
sudo mv /var/lib/libvirt/images/<VM_NAME>-enc.qcow2 /var/lib/libvirt/images/<VM_NAME>.qcow2
```

Éditer le XML VM :
```bash
virsh edit <VM_NAME>
```

Dans `<disk type='file' device='disk'>`, modifier `<source>` :
```xml
<source file='/var/lib/libvirt/images/<VM_NAME>.qcow2'>
  <encryption format='luks'>
    <secret type='passphrase' uuid='${UUID}'/>
  </encryption>
</source>
```

Mémoriser l'UUID pour le script `unlock-farm.sh` (section 0.F).

Démarrer et valider :
```bash
virsh start <VM_NAME>
# La VM doit démarrer normalement, voir /dev/vda comme disque non chiffré en interne
```

Une fois confirmé, effacer le backup clair :
```bash
sudo shred -uvz /var/lib/libvirt/images/<VM_NAME>-plain.bak
```

### Pour une nouvelle VM (à la création)

Créer directement un disque qcow2-LUKS vide, puis lancer l'install dessus :
```bash
UUID=$(uuidgen)
PASS=$(openssl rand -base64 32)
echo -n "$PASS" | pass insert -m -e "farm/<VM_NAME>"

# Créer le disque chiffré
sudo qemu-img create \
  --object secret,id=sec0,data="$PASS" \
  -f qcow2 \
  -o encrypt.format=luks,encrypt.key-secret=sec0 \
  /var/lib/libvirt/images/<VM_NAME>.qcow2 \
  <TAILLE>G

# Définir le secret libvirt (voir bloc ci-dessus)
# Créer la VM dans virt-manager avec ce fichier comme disque existant
# Dans l'XML, ajouter <encryption> comme ci-dessus

unset PASS
```

## 0.F. Script unlock-farm.sh

À déposer sur Void dans `~/bin/unlock-farm.sh` après avoir créé les VMs chiffrées et noté leurs UUIDs.

```bash
mkdir -p ~/bin
cat > ~/bin/unlock-farm.sh <<'SCRIPT'
#!/bin/bash
# unlock-farm.sh — déverrouille et démarre les VMs chiffrées après reboot Void.
# Prérequis : GPG agent actif (prompt passphrase automatique), pass initialisé.

set -euo pipefail

# Map VM -> UUID du secret libvirt associé.
# À compléter au fur et à mesure que tu crées les VMs chiffrées.
declare -A VM_SECRETS=(
  # [obsd-router]="UUID-À-REMPLIR"
  # [obsd-torgw]="UUID-À-REMPLIR"
  # [obsd-i2pgw]="UUID-À-REMPLIR"
  # [routine]="UUID-À-REMPLIR"
  # [dev]="UUID-À-REMPLIR"
  # [banque]="UUID-À-REMPLIR"
  # [bitcoin-node]="UUID-À-REMPLIR"
  # [sparrow]="UUID-À-REMPLIR"
  # [Whonix-Gateway-XFCE]="UUID-À-REMPLIR"
  # [Whonix-Workstation-XFCE]="UUID-À-REMPLIR"
  # [anon-browse]="UUID-À-REMPLIR"
  # [windows11]="UUID-À-REMPLIR"
)

# Ordre de démarrage (routeur en premier, applicatives ensuite, anonymat en dernier)
START_ORDER=(
  obsd-router
  obsd-torgw
  obsd-i2pgw
  Whonix-Gateway-XFCE
  Whonix-Workstation-XFCE
  routine
  dev
  banque
  bitcoin-node
  sparrow
  anon-browse
  windows11
)

echo "==> Déverrouillage du trousseau GPG (une passphrase demandée)"
# Force une décryption triviale pour déclencher le prompt GPG
pass ls > /dev/null

echo "==> Définition + injection des secrets libvirt (ephemeral, RAM uniquement)"
for VM in "${!VM_SECRETS[@]}"; do
  UUID="${VM_SECRETS[$VM]}"
  PASS=$(pass show "farm/$VM")
  
  cat > /tmp/$UUID-secret.xml <<EOF
<secret ephemeral='yes' private='yes'>
  <uuid>$UUID</uuid>
  <usage type='volume'>
    <volume>/var/lib/libvirt/images/$VM.qcow2</volume>
  </usage>
</secret>
EOF
  sudo virsh secret-define /tmp/$UUID-secret.xml > /dev/null
  rm /tmp/$UUID-secret.xml
  
  echo -n "$PASS" | base64 | sudo virsh secret-set-value "$UUID" --base64 /dev/stdin > /dev/null
  unset PASS
  echo "  - $VM : secret injecté"
done

echo "==> Démarrage des VMs dans l'ordre"
for VM in "${START_ORDER[@]}"; do
  if [[ -n "${VM_SECRETS[$VM]:-}" ]]; then
    echo "  - $VM : virsh start..."
    sudo virsh start "$VM" 2>/dev/null || echo "    (déjà active ou erreur — vérifier avec virsh list)"
    sleep 3
  fi
done

echo "==> Terminé. VMs sensibles démarrées."
echo "    Le trousseau GPG reste en mémoire le temps de la session ; purge manuelle :"
echo "    echo RELOADAGENT | gpg-connect-agent"
SCRIPT

chmod +x ~/bin/unlock-farm.sh
```

## 0.G. Désactiver l'autostart des VMs chiffrées

Après création des VMs chiffrées :
```bash
for VM in obsd-router obsd-torgw obsd-i2pgw routine dev banque bitcoin-node sparrow Whonix-Gateway-XFCE Whonix-Workstation-XFCE anon-browse windows11; do
  sudo virsh autostart --disable "$VM" 2>/dev/null || true
done

# Atelier garde son autostart (non chiffrée)
sudo virsh autostart atelier
```

Au boot Void : seule **atelier** démarre automatiquement. Le reste attend ton `unlock-farm.sh`.

## 0.H. Workflow de reboot

1. Reboot Void (panne EDF, kernel update, etc.)
2. Void boot sans prompt, SSH répond dès que sshd est up, WireGuard inbound actif
3. Depuis ton laptop (via WG inbound) : `ssh voidhv`
4. Lancer `./bin/unlock-farm.sh`
5. GPG prompt → taper la passphrase
6. Les 11 VMs chiffrées démarrent en ordre (routeur, gateways, applicatives)
7. Terminer la session : `echo RELOADAGENT | gpg-connect-agent` pour purger la passphrase de la mémoire

Durée totale : 30-60 secondes. Une seule interaction.

## 0.I. Vérifications

```bash
# Clé GPG présente
gpg --list-secret-keys

# Trousseau pass opérationnel
pass ls

# paperkey installé (pour backup éventuel)
which paperkey

# Script unlock-farm.sh présent
ls -l ~/bin/unlock-farm.sh
```

## État attendu en fin de phase 00

- Clé GPG Ed25519 (ou RSA 4096) sur voidhv, chiffrée par passphrase forte
- Paperkey sauvegardée physiquement hors machine
- `pass` initialisé avec la clé
- Script `unlock-farm.sh` en place, tableau `VM_SECRETS` vide (à remplir au fil des VMs)
- Politique : toutes les VMs sauf atelier seront créées avec qcow2-LUKS

---

# Phase 07 — VMs utilisateur

**Objectif** : déployer les VMs qui portent les usages réels : routine quotidienne, développement, banque, noeud Bitcoin + wallet Sparrow séparés, Windows 11.

**Prérequis** : `BASE.md` entièrement validé. Atelier SSH vers obsd-router et voidhv OK. Mullvad `wg0` actif.

## 7.A. Procédure commune pour toute nouvelle VM Debian

Cette procédure se répète pour chaque VM Linux classique. Une fois maîtrisée, les sections 7.B à 7.E la référencent sans la répéter.

### Création de la VM dans virt-manager

Depuis l'atelier (`virt-manager -c qemu+ssh://voidhv/system`) ou directement sur l'hôte :

1. **File → New Virtual Machine** → Local install media → ISO Debian netinst
2. RAM et CPU selon usage (cf. sections dédiées)
3. Disque : qcow2 dans `/var/lib/libvirt/images/<nom>.qcow2`
4. Network selection : le bridge cible (`br-vpn` ou `br-direct`)
5. Customize before install : ✅
6. Dans Customize :
   - **Firmware** : `UEFI x86_64: /usr/share/OVMF/OVMF_CODE.fd`
   - **CPU → Copy host CPU configuration** : ✅
   - **Disk bus** : VirtIO, cache `none`, discard `unmap`
   - **NIC** : virtio
   - **Sound** : garder ou retirer selon politique audio (cf. ARCHITECTURE §11)
   - **Spice channel qemu-ga** ajouté si absent
7. Begin installation

### Choix d'install Debian standard

- Langue, keymap standard
- Hostname : nom de la VM (ex: `routine`)
- Domain : `farm.lan`
- Partitioning : "Use entire disk, all files in one partition"
- Software selection : décocher desktop (on installe manuellement après), cocher **SSH server** et **Standard system utilities**

### Post-install : configuration commune

Récupérer l'IP dynamique via dhcpd :
```sh
# [sur obsd-router] : regarder les baux DHCP
doas cat /var/db/dhcpd.leases | tail -20
# ou : doas dhcpctl show all (si dispo)
```

Depuis l'atelier, SSH vers la VM (password initial) :
```bash
ssh TON_USER@<IP_ATTRIBUEE>
```

Dans la VM (root via `su -`) :
```bash
# Mise à jour complète
apt update && apt upgrade -y

# Paquets de base (ajuster selon usage)
apt install -y \
  sudo curl wget gnupg ca-certificates \
  git vim tmux htop \
  net-tools iputils-ping dnsutils \
  qemu-guest-agent

# Ajouter user à sudo
usermod -aG sudo TON_USER

# Activer qemu-guest-agent (pour virt-manager communication)
systemctl enable --now qemu-guest-agent
```

### Static lease sur obsd-router

Récupérer la MAC :
```bash
# [sur voidhv ou atelier]
virsh dumpxml <NOM_VM> | grep -A1 'source network' | grep mac
```

Sur obsd-router :
```sh
doas vi /etc/dhcpd.conf
```

Ajouter dans le subnet approprié (ex: `subnet 10.0.10.0`) :
```
    host <NOM_VM> {
        hardware ethernet AA:BB:CC:DD:EE:FF;
        fixed-address 10.0.10.XX;
    }
```

Recharger :
```sh
doas rcctl restart dhcpd
```

Forcer un renew dans la VM :
```bash
sudo dhclient -r && sudo dhclient
ip -br addr show
```

### Déposer la clé SSH depuis l'atelier

```bash
# Depuis l'atelier (user normal)
ssh-copy-id -i ~/.ssh/farm_os_admin.pub TON_USER@<IP_STATIQUE>
```

Désactiver le password login dans la VM :
```bash
# Dans la VM
sudo tee /etc/ssh/sshd_config.d/farm-os.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
EOF
sudo systemctl restart ssh
```

### Ajouter dans `~/.ssh/config` de l'atelier

```ssh
Host vm-<NOM_VM>
    Hostname 10.0.X.XX
    User TON_USER
    IdentityFile ~/.ssh/farm_os_admin
    ProxyJump voidhv
```

### Politique clipboard + audio (cf. ARCHITECTURE §10 et §11)

```bash
# Si VM = confort (atelier, routine, dev, windows) : installer
sudo apt install -y spice-vdagent

# Si VM = sensible (banque, bitcoin-node, sparrow) : NE PAS installer
# Et ajuster le XML pour retirer sound device :
#   virsh edit <vm> -> supprimer <sound> et <audio>
```

### Snapshot de référence

Sur voidhv :
```bash
virsh shutdown <NOM_VM>
virsh snapshot-create-as <NOM_VM> \
  --name "phase07-install-clean" \
  --description "VM installée, stabilisée, prête à usage (avant chiffrement)"
```

### Chiffrement qcow2 (sauf atelier)

**Sauf pour la VM atelier elle-même**, appliquer ensuite la procédure 0.E de la phase 00 (Préambule) :
- Générer passphrase + secret libvirt `ephemeral='yes'`
- Stocker dans `pass show farm/<VM_NAME>`
- Convertir le qcow2 en blob LUKS via `qemu-img convert`
- Éditer le XML VM pour pointer sur le secret
- Valider démarrage, effacer le backup clair
- Ajouter l'UUID dans `~/bin/unlock-farm.sh` (tableau `VM_SECRETS`)
- Désactiver l'autostart : `virsh autostart --disable <VM_NAME>`

---

## 7.B. VM Routine (social, mail, factures)

**Cible** : activités quotidiennes banales — mail perso, factures EDF / télécom / femme de ménage, réseaux sociaux, shopping, actualités.

**Specs** :
- Base : Debian stable
- RAM : 4 Go, CPU : 2
- Disque : 40 Go qcow2
- Bridge : `br-vpn` (sortie via Mullvad IS)
- IP fixe : `10.0.10.10`
- Clipboard + audio : ✅ activés

**Création** : suivre 7.A avec ces specs.

**Post-install : environnement XFCE léger**
```bash
sudo apt install -y \
  xfce4 xfce4-goodies lightdm \
  network-manager-gnome \
  firefox-esr thunderbird \
  libreoffice-calc libreoffice-writer \
  keepassxc file-roller \
  evince eog
```

```bash
sudo systemctl enable lightdm
sudo reboot
```

Au redémarrage, login graphique → XFCE.

**Config Firefox de cette VM** :
- Profil unique, history OK (c'est la VM normale)
- Container extensions si tu veux isoler les domaines sensibles
- uBlock Origin
- Mots de passe stockés dans KeePassXC (base dans `~/Documents/routine.kdbx`)

**Vérifications** :
```bash
curl -s https://am.i.mullvad.net/connected | head -1
# Doit confirmer sortie Mullvad Islande
```

Snapshot `phase07-routine-ready`.

---

## 7.C. VM Dev / Claude / GitHub

**Cible** : développement, Claude CLI, GitHub, editors lourds.

**Specs** :
- Debian stable
- RAM : 8 Go, CPU : 4
- Disque : 80 Go qcow2
- Bridge : `br-vpn`
- IP fixe : `10.0.10.20`
- Clipboard + audio : ✅

**Création** : 7.A.

**Post-install : outils dev**
```bash
sudo apt install -y \
  build-essential git \
  python3 python3-pip python3-venv \
  nodejs npm \
  docker.io docker-compose \
  neovim tmux ripgrep fd-find fzf \
  jq yq \
  xfce4 xfce4-terminal lightdm firefox-esr \
  chromium
```

```bash
sudo usermod -aG docker TON_USER
sudo systemctl enable lightdm
```

**Claude CLI** :
```bash
# Installation npm globale
sudo npm install -g @anthropic-ai/claude-cli
# Ou selon l'outil choisi ; authentification via navigateur au premier lancement
```

**GitHub** :
```bash
ssh-keygen -t ed25519 -f ~/.ssh/github -C "dev-vm-github"
# Ajouter ~/.ssh/github.pub dans https://github.com/settings/keys

# Dans ~/.ssh/config de la VM
cat >> ~/.ssh/config <<'EOF'

Host github.com
    IdentityFile ~/.ssh/github
    IdentitiesOnly yes
EOF
```

**Snapshot** `phase07-dev-ready`.

---

## 7.D. VM Banque + activation wg1 Mullvad Paris

**Cible** : accès banque en ligne, stockage vault PGP / SSH keys. **Hygiène stricte** : pas de clipboard, pas d'audio, pas de navigation hors sites bancaires.

**Specs** :
- Base : **Kicksecure** (Debian durci — choisi pour vault + hygiène stricte)
- RAM : 4 Go, CPU : 2
- Disque : 40 Go qcow2
- Bridge : `br-direct` (puis routage pf via `wg1` Paris)
- IP fixe : `10.0.20.10`
- Clipboard : ❌ désactivé
- Audio : ❌ désactivé

### 7.D.1 Activer wg1 Paris sur obsd-router

Générer/récupérer la config Mullvad single-hop Paris :
1. Sur le portail Mullvad → WireGuard Configuration → single-hop → exit **France (Paris)**
2. Télécharger le `.conf`, extraire `PublicKey`, `Endpoint` (IP:port), `Address` (IP attribuée)

Éditer `/etc/hostname.wg1` sur obsd-router :
```sh
doas vi /etc/hostname.wg1
```

```
wgkey <TA_CLE_PRIVEE_MULLVAD>    # peut être la même que wg0
wgpeer <PEER_PUB_PARIS> \
    wgendpoint <PEER_IP_PARIS> 51820 \
    wgaip 0.0.0.0/0
inet <IP_ATTRIBUEE_MULLVAD> 255.255.255.255
mtu 1420
up
```

```sh
doas chmod 600 /etc/hostname.wg1
doas sh /etc/netstart wg1
ifconfig wg1 | head -3
doas wg show wg1
```

### 7.D.2 Créer un bridge dédié `br-banque`

Le routage par bridge est plus propre que par IP. On crée un sous-segment dans `br-direct` via un nouveau bridge dédié.

Sur voidhv :
```bash
cat > /tmp/br-banque.xml <<'EOF'
<network>
  <name>br-banque</name>
  <bridge name='virbr-banque' stp='off' delay='0'/>
</network>
EOF
sudo virsh net-define /tmp/br-banque.xml
sudo virsh net-autostart br-banque
sudo virsh net-start br-banque
```

Ajouter une interface sur obsd-router pour ce bridge :
```bash
# Arrêter obsd-router, ajouter vio3 sur br-banque
virsh edit obsd-router
# <interface type='network'><source network='br-banque'/><model type='virtio'/></interface>
virsh start obsd-router
```

Sur obsd-router :
```sh
doas tee /etc/hostname.vio3 > /dev/null <<'EOF'
inet 10.0.21.1 255.255.255.0
EOF
doas sh /etc/netstart vio3
```

### 7.D.3 pf : router `br-banque` via `wg1`

Éditer `/etc/pf.conf` sur obsd-router, ajouter :
```
bnq_if  = "vio3"
bnq_net = "10.0.21.0/24"
vpn_fr  = "wg1"

match out on $vpn_fr inet from $bnq_net nat-to ($vpn_fr:0)

pass in quick on $bnq_if inet from $bnq_net to any route-to $vpn_fr keep state
block quick on $wan_if from $bnq_net
```

Recharger : `doas pfctl -f /etc/pf.conf`.

Static lease dhcpd pour `br-banque` : ajouter un `subnet 10.0.21.0` dans `/etc/dhcpd.conf`, redémarrer dhcpd avec `dhcpd_flags="vio1 vio2 vio3"`.

### 7.D.4 Installation Kicksecure

Télécharger Kicksecure KVM depuis [kicksecure.com](https://www.kicksecure.com/wiki/KVM) :
```bash
# [sur voidhv, dans ~/isos/]
curl -LO https://download.kicksecure.com/libvirt/Kicksecure-XFCE-X.Y.Z.Intel_AMD64.qcow2.libvirt.xz
# + signature .sig
xz -d Kicksecure-*.qcow2.libvirt.xz
```

Importer les XMLs libvirt fournis par Kicksecure pour les réseaux et la VM, puis adapter le network à `br-banque`.

Alternative : installer Debian stable normale en VM et appliquer la config Kicksecure via leurs scripts (plus manuel).

**Après premier boot** :
- Créer utilisateur
- Mettre à jour : `sudo apt update && sudo apt full-upgrade`
- Vérifier sortie IP : `curl -s ifconfig.me` → doit être une IP **française Mullvad**
- Tester login banque : si blocage, documenter le message et tester fallback via `br-direct` sans VPN (changer bridge dans virt-manager)

**Vault PGP / SSH** :
- Installer KeePassXC : `sudo apt install keepassxc`
- Importer/Générer clé GPG maîtresse dans `~/.gnupg/`
- Clés SSH stockées cryptées, sauvegardées sur support physique externe (clé USB chiffrée) uniquement — **jamais sur le réseau**

**Clipboard/audio** :
```bash
sudo apt purge spice-vdagent
# Dans virt-manager : Remove Hardware → Sound
```

**Snapshot** `phase07-banque-ready`.

### 7.D.5 Test de blocage banque (optionnel mais recommandé)

Tenter le login à ta banque habituelle depuis la VM banque via Paris. Trois scénarios possibles :
1. **Tout passe** : parfait, garde le setup.
2. **Challenge renforcé** (SMS, QCM) : acceptable si cohérent avec tes habitudes.
3. **Compte bloqué** : appeler le support banque pour débloquer. Basculer la VM sur `br-direct` sans VPN en attendant (via `virsh edit` ou virt-manager → retirer NIC br-banque et ajouter NIC br-direct).

Documenter le résultat dans `SECRETS.md.example` (template, pas le vrai) :
```
# Banque XXX : VPN Paris = KO, utilise br-direct
# Banque YYY : VPN Paris = OK
```

---

## 7.E. Bitcoin-node (bitcoind headless) + Sparrow (wallet)

Deux VMs séparées, philosophie "full node sur une VM, clés privées sur une autre" :
- **bitcoin-node** : bitcoind headless, parle au réseau BTC, pas de clés
- **sparrow** : GUI, connecte à bitcoin-node via RPC, contient les clés privées chiffrées

**Les deux sur `br-direct`** (clearnet car bitcoin-node gagne à être joignable, Sparrow parle au node local).

### 7.E.1 VM bitcoin-node

**Specs** :
- Debian stable
- RAM : 4 Go, CPU : 2
- Disque : **600 Go** qcow2 (blockchain BTC ≈ 550 Go + marge)
- Bridge : `br-direct`
- IP fixe : `10.0.20.20`
- Clipboard ❌, audio ❌, headless (pas de desktop)

Installation via 7.A, mode serveur (pas de XFCE).

Installer bitcoind :
```bash
sudo apt install -y bitcoind

sudo mkdir -p /var/lib/bitcoind
sudo chown bitcoin:bitcoin /var/lib/bitcoind

sudo tee /etc/bitcoin/bitcoin.conf > /dev/null <<'EOF'
# bitcoin.conf

datadir=/var/lib/bitcoind

# Réseau
listen=1
bind=10.0.20.20
port=8333

# RPC (pour Sparrow uniquement, accessible via LAN interne)
rpcbind=10.0.20.20
rpcallowip=10.0.20.30   # IP de la VM sparrow
rpcuser=bitcoinrpc
rpcpassword=GENERER_UN_PASSWORD_FORT_32_CHARS

# Full node, pas de pruning (si tu veux pruner: prune=5500 pour 5.5GB)
txindex=1

# Performance
dbcache=2048
maxconnections=40
EOF

sudo chown root:bitcoin /etc/bitcoin/bitcoin.conf
sudo chmod 640 /etc/bitcoin/bitcoin.conf

sudo systemctl enable --now bitcoind
```

**Sync initial** : plusieurs jours (200-500 Go à télécharger selon la vitesse de ton lien). Surveiller :
```bash
sudo tail -f /var/log/bitcoin/debug.log
# Ou avec bitcoin-cli :
bitcoin-cli -rpcwait getblockchaininfo | jq .verificationprogress
```

Snapshot **après sync** : `phase07-bitcoin-synced` (énorme, ~600 Go, à prévoir si possible sur disque dédié hôte avec BTRFS).

### 7.E.2 VM sparrow (wallet GUI)

**Specs** :
- Debian stable
- RAM : 4 Go, CPU : 2
- Disque : 20 Go qcow2
- Bridge : `br-direct`
- IP fixe : `10.0.20.30`
- Clipboard ❌ (éviter clipboard swappers sur adresses BTC)
- Audio ❌

Installation via 7.A.

Installer XFCE + Sparrow :
```bash
sudo apt install -y xfce4 lightdm firefox-esr
sudo systemctl enable lightdm

# Sparrow : télécharger .deb depuis sparrowwallet.com
# Vérifier signature GPG :
curl -LO https://github.com/sparrowwallet/sparrow/releases/download/X.Y.Z/sparrow_X.Y.Z-1_amd64.deb
curl -LO https://github.com/sparrowwallet/sparrow/releases/download/X.Y.Z/sparrow-X.Y.Z-manifest.txt
curl -LO https://github.com/sparrowwallet/sparrow/releases/download/X.Y.Z/sparrow-X.Y.Z-manifest.txt.asc
gpg --keyserver hkps://keys.openpgp.org --recv-keys D4D0D3202FC06849A257B38DE94618334C674B40
gpg --verify sparrow-X.Y.Z-manifest.txt.asc
sha256sum -c sparrow-X.Y.Z-manifest.txt --ignore-missing
sudo apt install ./sparrow_*.deb
```

Lancer Sparrow via menu XFCE :
- Preferences → Server → Bitcoin Core
- URL : `http://10.0.20.20:8332`
- User : `bitcoinrpc`
- Password : celui du bitcoin.conf
- Test Connection → OK

Créer un wallet :
- **Hardware wallet recommandé** : Ledger, Trezor, Coldcard (device physique)
- Si software wallet : générer seed dans Sparrow, **écrire la seed sur papier**, **jamais photographier**, stocker dans un lieu physique sûr
- **Ne jamais coller** la seed dans le presse-papier (cf. ARCHITECTURE §10)

**Règle pf** pour autoriser Sparrow → bitcoin-node :
```
# Sur obsd-router /etc/pf.conf :
pass in quick on $dir_if proto tcp from 10.0.20.30 to 10.0.20.20 port 8332 keep state
```

**Snapshot** `phase07-sparrow-ready` (avant de créer le wallet — si créé, le snapshot contient la seed chiffrée mais pas la passphrase, acceptable).

---

## 7.F. Windows 11

**Specs** :
- Windows 11 Pro ou Enterprise (ISO officielle Microsoft)
- RAM : 8 Go, CPU : 4
- Disque : 80 Go qcow2
- Bridge : `br-vpn`
- IP fixe : `10.0.10.40`
- Clipboard : ✅ RDP natif préféré
- Audio : ✅

### 7.F.1 Préparer virtio-win et swtpm

Télécharger virtio-win ISO : [fedoraproject.org virtio-win](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso).

Déposer sur voidhv dans `~/isos/virtio-win.iso`.

Vérifier swtpm installé (phase 01) : `which swtpm`.

### 7.F.2 Créer la VM

`virt-manager` → New VM avec :
- ISO : `Windows11.iso`
- RAM 8192 / CPU 4 / Disk 80 GB
- Customize before install :
  - **Firmware** : `UEFI x86_64 secboot: /usr/share/OVMF/OVMF_CODE.secboot.fd` (Secure Boot)
  - **CPUs** : Copy host CPU
  - **Add Hardware → TPM** :
    - Type : Emulated
    - Model : CRB
    - Version : 2.0
  - **Add Hardware → Storage** : second CD-ROM avec `virtio-win.iso`
  - **Disk Bus** : VirtIO (important sinon Windows ne verra pas le disque sans drivers)
  - **NIC** : virtio
- Begin Installation

### 7.F.3 Installation Windows 11

- **Entrée de clé** : entrer la licence ou "I don't have a key" pour eval (peut être activé plus tard)
- **Custom install** : au moment du choix du disque, Windows ne verra rien. Cliquer **Load driver** → parcourir le CD virtio-win → `amd64\w11` → installer `viostor` (stockage). Puis NetKVM (réseau) aussi.
- Suite de l'install classique
- Au premier boot : laisser finir l'OOBE (Out Of Box Experience)

### 7.F.4 Post-install Windows

Dans Windows :
1. Installer tous les virtio drivers restants depuis `D:\` (ou E:\ selon CD-ROM) : balloon, vioinput, vioserial, qxldod pour video
2. Installer **SPICE Guest Tools** depuis [spice-space.org](https://www.spice-space.org/download.html#windows-binaries) — pour audio et clipboard
3. Activer **RDP** (plus fluide que SPICE pour Windows) :
   - Settings → System → Remote Desktop → **On**
   - Noter le nom de la machine ou utiliser l'IP 10.0.10.40
4. Updates Windows complètes

### 7.F.5 Accès depuis atelier

Depuis l'atelier :
```bash
# RDP via Remmina ou ligne de commande :
remmina rdp://USER@10.0.10.40
# ou : xfreerdp /v:10.0.10.40 /u:USER /p:PASSWORD /clipboard:mode=allow-text +fonts
```

**Configuration RDP conseillée** :
- Clipboard : texte uniquement (pas d'images/fichiers) — côté Remmina Preferences
- Audio : redirigé local
- Keyboard : FR si tu es sur clavier FR

**Snapshot** `phase07-windows-ready`.

---

## État attendu en fin de phase 07

- 6 VMs applicatives en marche : atelier, routine, dev/claude, banque, bitcoin-node, sparrow, windows11 (7 avec l'atelier déjà là)
- `wg1` Mullvad Paris actif, banque route via Paris
- bitcoin-node sync (ou sync en cours)
- sparrow connecté à bitcoin-node via RPC interne
- Windows 11 accessible en RDP depuis atelier et distant
- Snapshots pour chaque VM

---

# Phase 08 — Whonix (GW + WS)

**Objectif** : ajouter le couple Whonix-Gateway (Tor) + Whonix-Workstation (Tor Browser durci) pour le browsing Tor sérieux et amnésique optionnel.

**Prérequis** : phase 07 OK. Bridge `br-whonix` à créer.

**Référence archi** : `ARCHITECTURE.md` §3 (Whonix-GW/WS).

## 8.A. Créer le bridge `br-whonix`

```bash
# [sur voidhv]
cat > /tmp/br-whonix.xml <<'EOF'
<network>
  <name>br-whonix</name>
  <bridge name='virbr-whonix' stp='off' delay='0'/>
</network>
EOF
sudo virsh net-define /tmp/br-whonix.xml
sudo virsh net-autostart br-whonix
sudo virsh net-start br-whonix
```

## 8.B. Télécharger les images Whonix officielles

Depuis [whonix.org/wiki/KVM](https://www.whonix.org/wiki/KVM) :

```bash
# [sur voidhv, dans ~/isos/whonix/]
mkdir -p ~/isos/whonix && cd ~/isos/whonix

# Télécharger les 4 fichiers :
# - Whonix-Gateway-XFCE-XX.X.X.X.Intel_AMD64.qcow2.libvirt.xz
# - Whonix-Workstation-XFCE-XX.X.X.X.Intel_AMD64.qcow2.libvirt.xz
# - les .xml libvirt correspondants
# - les signatures .asc

# Vérifier signatures
gpg --keyserver hkps://keys.openpgp.org --recv-keys 916B8D99C38EAF5E8ADC7A2A8D66066A2EEACCDA
gpg --verify *.asc

# Décompresser les images
xz -d *.qcow2.libvirt.xz
```

## 8.C. Importer dans libvirt

Les fichiers XML fournis par Whonix définissent :
- Un réseau isolé interne (`Whonix-Internal`) — on peut le garder ou le remplacer par notre `br-whonix`
- Un réseau externe (`Whonix-External`) — à remplacer par `br-wan`

Adapter les XMLs :
```bash
# Remplacer les références réseau par nos bridges :
sed -i 's|Whonix-External|br-wan|g' Whonix-Gateway*.xml
sed -i 's|Whonix-Internal|br-whonix|g' Whonix-Gateway*.xml Whonix-Workstation*.xml
```

Déplacer les qcow2 vers le pool libvirt :
```bash
sudo mv Whonix-Gateway-XFCE-*.qcow2 /var/lib/libvirt/images/whonix-gw.qcow2
sudo mv Whonix-Workstation-XFCE-*.qcow2 /var/lib/libvirt/images/whonix-ws.qcow2
sudo chown libvirt-qemu:libvirt-qemu /var/lib/libvirt/images/whonix-*.qcow2
```

Éditer les XMLs VM pour pointer sur les nouveaux chemins, puis :
```bash
sudo virsh define Whonix-Gateway-XFCE-*.xml
sudo virsh define Whonix-Workstation-XFCE-*.xml
```

## 8.D. Premier boot

```bash
virsh start Whonix-Gateway-XFCE
virt-viewer Whonix-Gateway-XFCE
```

Accepter les conditions d'usage Whonix, laisser l'assistant setup.

Ensuite :
```bash
virsh start Whonix-Workstation-XFCE
virt-viewer Whonix-Workstation-XFCE
```

Dans chaque : update (via `whonix-repository` puis `sudo apt update && sudo apt full-upgrade`).

## 8.E. Intégration avec obsd-router

Ajouter une règle pf pour autoriser Whonix-GW à sortir sur Internet via WAN :
```
# /etc/pf.conf
pass in quick on $wan_if inet from 192.168.100.0/24 to any keep state
# (Whonix-GW reçoit une IP via dhcpd de libvirt ? Non — br-wan a libvirt NAT,
#  on peut laisser libvirt DHCP gérer Whonix-GW côté WAN)
```

Par défaut, Whonix-GW tire son IP WAN via DHCP libvirt (192.168.100.X). Vérifier dans Whonix-GW :
```sh
sudo ip -br addr
# eth0 sur 192.168.100.X/24
# eth1 sur 10.152.152.10 (IP interne Whonix par défaut)
```

Whonix-WS, elle, parle uniquement à Whonix-GW via `br-whonix` et reçoit `10.152.152.11`.

## 8.F. Vérifications

Dans Whonix-WS :
- Ouvrir Tor Browser (raccourci bureau)
- Vérifier sur `check.torproject.org` → "Congratulations. This browser is configured to use Tor."
- Aucun accès direct Internet hors Tor

Dans Whonix-GW :
- `systemctl status tor` → active
- `nyx` pour monitoring en ligne de commande

## 8.G. Snapshots

```bash
virsh shutdown Whonix-Gateway-XFCE
virsh shutdown Whonix-Workstation-XFCE
virsh snapshot-create-as Whonix-Gateway-XFCE --name "phase08-gw-ready"
virsh snapshot-create-as Whonix-Workstation-XFCE --name "phase08-ws-ready"
```

## État attendu en fin de phase 08

- Whonix-GW + Whonix-WS opérationnels sur `br-whonix`
- Tor Browser dans Whonix-WS navigue via le réseau Tor
- Zéro fuite clearnet depuis Whonix-WS

---

# Phase 09 — Tor-gw (OpenBSD, transproxy)

**Objectif** : seconde VM OpenBSD dédiée à la torification réseau des VMs non-navigateurs (scripts, clients CLI, VMs éphémères). Complémentaire à Whonix qui reste pour le browsing GUI.

**Prérequis** : phases précédentes OK. ISO OpenBSD sous la main.

**Référence archi** : `ARCHITECTURE.md` §3 (Tor-gw).

## 9.A. Créer les bridges `br-tor-gw` et `br-tor`

```bash
# [sur voidhv]
for name in br-tor-gw br-tor; do
  cat > /tmp/${name}.xml <<EOF
<network>
  <name>${name}</name>
  <bridge name='virbr-${name#br-}' stp='off' delay='0'/>
</network>
EOF
  sudo virsh net-define /tmp/${name}.xml
  sudo virsh net-autostart ${name}
  sudo virsh net-start ${name}
done
```

## 9.B. Créer la VM OpenBSD Tor-gw

Procédure identique à la phase 03 de BASE.md (install OpenBSD base), avec ces différences :
- Nom : `obsd-torgw`
- RAM 1 Go, 1 CPU, disque 8 Go
- **Deux interfaces** : `br-tor-gw` (WAN-like, communique avec obsd-router) et `br-tor` (côté clients)
- Install headless, doas, sshd par clé, patches

## 9.C. Configuration réseau sur obsd-torgw

```sh
# /etc/hostname.vio0 (br-tor-gw, vers obsd-router)
doas tee /etc/hostname.vio0 > /dev/null <<'EOF'
inet 10.0.40.10 255.255.255.0
!route add default 10.0.40.1
EOF

# /etc/hostname.vio1 (br-tor, vers clients)
doas tee /etc/hostname.vio1 > /dev/null <<'EOF'
inet 10.0.31.1 255.255.255.0
EOF

doas sh /etc/netstart
```

Sur **obsd-router** (le routeur principal) : ajouter une interface sur `br-tor-gw` :
```bash
# [sur voidhv]
virsh shutdown obsd-router
virsh edit obsd-router
# Ajouter <interface type='network'><source network='br-tor-gw'/><model type='virtio'/></interface>
virsh start obsd-router
```

Dans obsd-router :
```sh
doas tee /etc/hostname.vio4 > /dev/null <<'EOF'
inet 10.0.40.1 255.255.255.0
EOF
doas sh /etc/netstart vio4
```

## 9.D. Installer et configurer Tor sur obsd-torgw

```sh
doas pkg_add tor
doas vi /etc/tor/torrc
```

```
# /etc/tor/torrc — Tor-gw transproxy
User _tor
DataDirectory /var/tor

SOCKSPort 10.0.31.1:9050
DNSPort 10.0.31.1:5353
TransPort 10.0.31.1:9040
TransProxyType default

ClientOnly 1
AvoidDiskWrites 1
SafeSocks 1

# Décommenter si FAI/Wi-Fi bloque Tor :
# UseBridges 1
# ClientTransportPlugin obfs4 exec /usr/local/bin/obfs4proxy
# Bridge obfs4 ADDR:PORT FINGERPRINT cert=... iat-mode=0
```

```sh
doas rcctl enable tor
doas rcctl start tor
doas rcctl check tor
```

Vérifier :
```sh
netstat -an | grep -E '9040|9050|5353'
# Doit montrer bindings sur 10.0.31.1
```

## 9.E. pf transproxy sur obsd-torgw

```sh
doas vi /etc/pf.conf
```

```pf
# /etc/pf.conf — obsd-torgw

ext_if  = "vio0"    # vers obsd-router
int_if  = "vio1"    # vers clients br-tor
cli_net = "10.0.31.0/24"

set skip on lo
set block-policy drop

# Transproxy TCP clients -> Tor
pass in on $int_if inet proto tcp from $cli_net to any \
    divert-to 127.0.0.1 port 9040
# Redirect DNS -> Tor DNSPort
pass in on $int_if inet proto udp from $cli_net to any port 53 \
    rdr-to 127.0.0.1 port 5353

# Fail-closed : pas de UDP/ICMP sortant
block quick on $int_if proto { icmp udp }
block out on $int_if

# Tor lui-même peut sortir
pass out quick on $ext_if proto tcp from self keep state
pass out quick on $ext_if proto udp from self to any port { 53 123 } keep state
```

Activer IP forwarding :
```sh
doas sysctl net.inet.ip.forwarding=1
echo "net.inet.ip.forwarding=1" | doas tee -a /etc/sysctl.conf
```

Charger pf :
```sh
doas pfctl -f /etc/pf.conf
```

## 9.F. obsd-router : autoriser br-tor-gw ↔ WAN

Dans `/etc/pf.conf` sur obsd-router :
```
# Autoriser obsd-torgw (10.0.40.10) à sortir directement sur WAN (pas via wg0)
pass in quick on vio4 inet from 10.0.40.10 to any keep state
match out on $wan_if inet from 10.0.40.0/24 nat-to ($wan_if:0)
```

Recharger : `doas pfctl -f /etc/pf.conf`.

## 9.G. Test avec une VM jetable

```bash
# [sur voidhv]
virt-install \
  --name test-tor \
  --memory 512 --vcpus 1 \
  --cdrom ~/isos/alpine-virt-latest.iso \
  --disk size=2,format=qcow2,path=/tmp/test-tor.qcow2 \
  --network network=br-tor,model=virtio \
  --os-variant alpinelinux3.21 \
  --graphics spice
```

Dans la console Alpine (configurer IP manuellement ou utiliser une autre distro avec DHCP auto-configurée sur br-tor — il faut un dhcpd sur obsd-torgw ou configurer manuellement) :

```sh
# Alpine live : config IP manuelle
ip addr add 10.0.31.50/24 dev eth0
ip link set eth0 up
ip route add default via 10.0.31.1
echo "nameserver 10.0.31.1" > /etc/resolv.conf

# Tester transproxy
wget -qO- https://check.torproject.org | grep -i 'Congratulations'
# Doit dire "Congratulations. This browser is configured to use Tor"
```

Si OK → cleanup : `virsh destroy test-tor; virsh undefine test-tor --remove-all-storage`.

**Option** : installer un dhcpd sur obsd-torgw pour éviter la config manuelle à chaque VM cliente.

## 9.H. Snapshot

```bash
virsh shutdown obsd-torgw
virsh snapshot-create-as obsd-torgw \
  --name "phase09-torgw-ready" \
  --description "Tor daemon avec transproxy TCP + DNS, pf fail-closed"
```

## État attendu en fin de phase 09

- VM `obsd-torgw` sur `br-tor-gw`/`br-tor`
- Tor daemon écoute TransPort/DNSPort/SOCKSPort
- Une VM cliente sur `br-tor` sort sur Internet **uniquement via Tor**
- Fail-closed si Tor crash

---

# Phase 10 — I2P-gw (OpenBSD, i2pd)

**Objectif** : passerelle I2P pour browsing eepsite et services I2P.

## 10.A. Bridges `br-i2p-gw` et `br-i2p`

Même procédure que 9.A en remplaçant `tor` par `i2p`.

## 10.B. Créer la VM `obsd-i2pgw`

Identique à 9.B (OpenBSD base, 1 Go RAM, 1 CPU, 8 Go disque, deux NICs `br-i2p-gw` et `br-i2p`).

## 10.C. Configuration réseau

```sh
# /etc/hostname.vio0 (br-i2p-gw vers obsd-router)
inet 10.0.41.10 255.255.255.0
!route add default 10.0.41.1
```

```sh
# /etc/hostname.vio1 (br-i2p vers clients)
inet 10.0.32.1 255.255.255.0
```

Sur obsd-router : ajouter interface vio5 sur `br-i2p-gw` avec IP `10.0.41.1/24`, même logique que 9.C et 9.F.

## 10.D. Installer et configurer i2pd

```sh
doas pkg_add i2pd
doas vi /etc/i2pd/i2pd.conf
```

Ajuster (minimaux) :
```ini
ipv4 = true
ipv6 = false
daemon = true

# SOCKS proxy pour clients
[socksproxy]
enabled = true
address = 10.0.32.1
port = 4447

# HTTP proxy pour clients
[httpproxy]
enabled = true
address = 10.0.32.1
port = 4444

# Bande passante (ajuster)
bandwidth = L
```

Activer :
```sh
doas rcctl enable i2pd
doas rcctl start i2pd
doas rcctl check i2pd
```

## 10.E. pf sur obsd-i2pgw

```pf
# /etc/pf.conf — obsd-i2pgw

ext_if = "vio0"
int_if = "vio1"
cli_net = "10.0.32.0/24"

set skip on lo
set block-policy drop

# Clients peuvent parler au proxy SOCKS/HTTP
pass in on $int_if proto tcp from $cli_net to self port { 4444 4447 } keep state

# Tout le reste bloqué depuis clients (pas de pass through)
block quick on $int_if from $cli_net

# i2pd sort vers réseau I2P
pass out quick on $ext_if proto { tcp udp } from self keep state
```

Pas besoin de forwarding ici : les clients ne routent pas à travers i2pgw, ils utilisent i2pd comme proxy applicatif. Les requêtes sont encapsulées par le protocole I2P et sortent depuis i2pgw vers le réseau I2P mondial.

## 10.F. Test basique

Depuis une VM sur `br-i2p`, configurer le navigateur avec proxy HTTP `10.0.32.1:4444`, charger `http://stats.i2p` → doit répondre (la première résolution prend 30-60s, le temps que i2pd établisse des tunnels).

## 10.G. Snapshot

```bash
virsh shutdown obsd-i2pgw
virsh snapshot-create-as obsd-i2pgw --name "phase10-i2pgw-ready"
```

## État attendu en fin de phase 10

- `obsd-i2pgw` opérationnel avec i2pd
- Proxy SOCKS 4447 / HTTP 4444 accessible depuis `br-i2p`
- Clients résolvent des eepsites via i2pd

---

# Phase 11 — anon-browse (Kicksecure + TBB + Chromium-I2P)

**Objectif** : VM navigateur spécialisée — Tor Browser officiel (bundle Linux blend-in) **en parallèle** de Chromium dédié I2P. Base Kicksecure pour hygiène système.

**Référence archi** : `ARCHITECTURE.md` §3 (anon-browse) et §4 (politique fingerprint).

## 11.A. Bridge `br-browse`

```bash
cat > /tmp/br-browse.xml <<'EOF'
<network>
  <name>br-browse</name>
  <bridge name='virbr-browse' stp='off' delay='0'/>
</network>
EOF
sudo virsh net-define /tmp/br-browse.xml
sudo virsh net-autostart br-browse
sudo virsh net-start br-browse
```

Sur obsd-router : ajouter NIC vio6 sur `br-browse` avec IP `10.0.33.1/24`, subnet dhcpd correspondant.

## 11.B. Importer Kicksecure KVM

```bash
# [sur voidhv dans ~/isos/kicksecure/]
# Télécharger depuis https://www.kicksecure.com/wiki/KVM :
# - Kicksecure-XFCE-XX.X.X.X.Intel_AMD64.qcow2.libvirt.xz
# - .xml libvirt
# - signature

gpg --keyserver hkps://keys.openpgp.org --recv-keys <KEY_KICKSECURE>
gpg --verify *.asc
xz -d *.qcow2.libvirt.xz
```

Adapter le XML : network → `br-browse`, renommer la VM en `anon-browse`.

```bash
sudo mv Kicksecure-*.qcow2 /var/lib/libvirt/images/anon-browse.qcow2
sudo chown libvirt-qemu:libvirt-qemu /var/lib/libvirt/images/anon-browse.qcow2
sudo virsh define Kicksecure-*.xml   # après adaptation XML
```

## 11.C. Premier boot + config

```bash
virsh start anon-browse
virt-viewer anon-browse
```

Créer utilisateur, update Kicksecure :
```bash
sudo apt update && sudo apt full-upgrade -y
```

Static lease dhcpd sur obsd-router pour `anon-browse` → `10.0.33.10`.

## 11.D. pf sur obsd-router pour `br-browse`

Règles spécifiques :
```
brs_if = "vio6"
brs_net = "10.0.33.0/24"

# Tor Browser a son Tor embarqué, doit pouvoir atteindre Internet en TCP
pass in quick on $brs_if inet proto tcp from $brs_net to any keep state

# Chromium-I2P : proxy vers obsd-i2pgw
pass in quick on $brs_if inet proto tcp from $brs_net to 10.0.32.1 port { 4444 4447 } keep state

# Pas d'UDP/ICMP (évite fuite DNS et fingerprinting)
block quick on $brs_if proto { udp icmp }

# NAT sortant via WAN direct (Tor Browser gère son anonymat via Tor interne)
match out on $wan_if inet from $brs_net nat-to ($wan_if:0)
```

Recharger pf.

## 11.E. Installer Tor Browser (bundle officiel)

Dans la VM anon-browse :
```bash
# Télécharger depuis torproject.org (lien stable)
cd ~/Downloads
TOR_VER=$(curl -s https://www.torproject.org/download/ | grep -oP 'tor-browser-linux-amd64-\K[0-9.]+(?=.tar\.xz)' | head -1)
curl -LO "https://dist.torproject.org/torbrowser/${TOR_VER}/tor-browser-linux-amd64-${TOR_VER}.tar.xz"
curl -LO "https://dist.torproject.org/torbrowser/${TOR_VER}/tor-browser-linux-amd64-${TOR_VER}.tar.xz.asc"

# Vérifier signature
gpg --auto-key-locate nodefault,wkd --locate-keys torbrowser@torproject.org
gpg --verify tor-browser-linux-amd64-*.tar.xz.asc

# Extraire
tar xf tor-browser-linux-amd64-*.tar.xz
mv tor-browser /opt/tor-browser

# Lancer une fois pour enregistrer le desktop entry
/opt/tor-browser/start-tor-browser.desktop --register-app
```

## 11.F. Installer Chromium et configurer pour I2P

```bash
sudo apt install -y chromium
```

Créer un profil Chromium dédié I2P :
```bash
mkdir -p ~/.config/chromium-i2p
```

Script de lancement `~/bin/chromium-i2p.sh` :
```bash
mkdir -p ~/bin
cat > ~/bin/chromium-i2p.sh <<'EOF'
#!/bin/bash
exec chromium \
  --user-data-dir="$HOME/.config/chromium-i2p" \
  --proxy-server="http=10.0.32.1:4444;https=10.0.32.1:4444" \
  --proxy-bypass-list="<-loopback>" \
  --new-window \
  "$@"
EOF
chmod +x ~/bin/chromium-i2p.sh
```

Créer un raccourci bureau (XFCE) :
```bash
cat > ~/.local/share/applications/chromium-i2p.desktop <<'EOF'
[Desktop Entry]
Name=Chromium (I2P)
Exec=/home/USER/bin/chromium-i2p.sh %U
Icon=chromium
Type=Application
Categories=Network;WebBrowser;
EOF
```

Remplacer `USER` par ton nom user.

## 11.G. Hygiène : pas de Firefox générique, pas de sound, pas de spice-vdagent

```bash
sudo apt purge firefox* spice-vdagent -y
```

Sur voidhv, retirer sound de l'XML :
```bash
virsh edit anon-browse
# Supprimer <sound>, <audio>
```

## 11.H. Snapshot

```bash
virsh shutdown anon-browse
virsh snapshot-create-as anon-browse --name "phase11-browse-ready"
```

## État attendu en fin de phase 11

- `anon-browse` lance Tor Browser via son icône desktop → navigue sur le réseau Tor
- `chromium-i2p.sh` ouvre Chromium qui route uniquement les `.i2p` via `obsd-i2pgw`
- Aucun autre navigateur, aucun son, pas de clipboard partagé

---

# Phase 12 — VMs éphémères torifiées

**Objectif** : système de VMs jetables instanciées à la demande, branchées automatiquement sur un bridge (`br-tor`, `br-i2p`, ou `br-ephem` clearnet), qui se détruisent à la fermeture. Scripts `spawn-*` lancés depuis l'atelier.

**Référence archi** : `ARCHITECTURE.md` §7 (VMs éphémères).

## 12.A. Préparer le template Alpine

Télécharger Alpine standard :
```bash
# [sur voidhv]
cd ~/isos/
curl -LO https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/alpine-virt-3.21.0-x86_64.iso
```

Créer la VM template :
```bash
virt-install \
  --name torbox-template \
  --memory 1024 --vcpus 2 \
  --cdrom ~/isos/alpine-virt-3.21.0-x86_64.iso \
  --disk size=4,format=qcow2,path=/var/lib/libvirt/images/torbox-template.qcow2 \
  --network network=br-tor,model=virtio \
  --os-variant alpinelinux3.21 \
  --graphics spice
```

Dans la console Alpine (login `root`) :
```sh
setup-alpine
# Hostname: torbox-template
# Network: eth0 dhcp (obsd-torgw doit servir DHCP, sinon IP statique)
# Root password: <random>
# Timezone: UTC
# Proxy: none
# NTP: chrony
# Mirror: 1 (ou f pour fastest)
# User: none (root seulement, c'est un jetable)
# SSH: openssh
# Disk: sda sys
```

Reboot Alpine.

Post-install :
```sh
# Configurer pour DHCP sur eth0 automatique
echo "auto eth0" > /etc/network/interfaces
echo "iface eth0 inet dhcp" >> /etc/network/interfaces

# Installer outils de base
apk update
apk add curl wget tor-browser-launcher firefox-esr torsocks \
        xfce4 xfce4-terminal lightdm \
        chromium
```

**Note** : tor-browser-launcher sur Alpine est optionnel. Tu peux à la place fournir une image template avec Tor Browser déjà installé dans `/opt/tor-browser/` si tu préfères (moins de runtime).

Configurer auto-poweroff après 60 min d'inactivité (option) :
```sh
echo '*/15 * * * * [ -z "$(who)" ] && [ $(awk -F. "{print \$1}" /proc/uptime) -gt 3600 ] && poweroff' | crontab -
```

Arrêter proprement :
```sh
poweroff
```

Sur voidhv, sauvegarder le template en lecture seule :
```bash
sudo chmod 400 /var/lib/libvirt/images/torbox-template.qcow2
virsh undefine torbox-template   # on garde juste le qcow2 comme base
```

## 12.B. Scripts spawn-* sur la VM atelier

Dans la VM atelier (qui a le client virt-manager/virsh en qemu+ssh:// vers voidhv) :

```bash
mkdir -p ~/bin
```

### `spawn-torbox.sh`
```bash
cat > ~/bin/spawn-torbox.sh <<'EOF'
#!/bin/bash
# Lance une VM Alpine éphémère branchée sur br-tor (torifiée).
# La VM est détruite à sa fermeture (--transient).

set -euo pipefail

CONN="qemu+ssh://USER@voidhv/system"
TEMPLATE="/var/lib/libvirt/images/torbox-template.qcow2"
NAME="torbox-$(date +%s)"
DISK="/tmp/${NAME}.qcow2"

ssh voidhv "qemu-img create -f qcow2 -F qcow2 -b ${TEMPLATE} ${DISK}"

virt-install \
  --connect "${CONN}" \
  --name "${NAME}" \
  --memory 2048 --vcpus 2 \
  --disk path=${DISK},device=disk,bus=virtio \
  --network bridge=virbr-tor,model=virtio \
  --os-variant alpinelinux3.21 \
  --graphics spice \
  --transient \
  --noautoconsole \
  --import

# Ouvrir la console
virt-viewer --connect "${CONN}" "${NAME}" &

echo "VM ${NAME} lancée. Elle sera détruite à l'arrêt."
echo "Pour arrêter : virsh --connect ${CONN} destroy ${NAME}"
EOF
chmod +x ~/bin/spawn-torbox.sh
```

### `spawn-i2pbox.sh`
Variante : `--network bridge=virbr-i2p`. Le template peut être le même ou un template dédié avec Chromium-I2P préconfiguré.

### `spawn-directbox.sh`
Variante : `--network bridge=virbr-ephem` (ajouter ce bridge si besoin, ou router via `br-direct`). Pour tests clearnet jetables.

## 12.C. Option amnésique extrême : `-snapshot` QEMU

Pour que **rien** ne soit persistant (même pas temporairement sur disque hôte), utiliser l'option QEMU `-snapshot` : les écritures vont en RAM et disparaissent à l'arrêt.

Dans le script, ajouter au lieu du `qemu-img create` :
```bash
# Au lieu de COW disk :
virt-install \
  --connect "${CONN}" \
  --name "${NAME}" \
  ...
  --disk path=${TEMPLATE},device=disk,bus=virtio,snapshot=on \
  ...
```

**Attention** : consomme la RAM selon les écritures. Pour tests courts uniquement.

## 12.D. Cleanup des disques résiduels

Script cron dans la VM atelier :
```bash
cat > ~/bin/cleanup-ephem.sh <<'EOF'
#!/bin/bash
# Supprime les disques qcow2 de VMs éphémères non utilisées

ssh voidhv 'find /tmp -maxdepth 1 -name "torbox-*.qcow2" -o -name "i2pbox-*.qcow2" -o -name "directbox-*.qcow2" | while read f; do
  NAME=$(basename "$f" .qcow2)
  if ! virsh list | grep -q "$NAME"; then
    rm -f "$f"
    echo "Cleaned $f"
  fi
done'
EOF
chmod +x ~/bin/cleanup-ephem.sh

# Planifier : quotidien
(crontab -l 2>/dev/null; echo "0 2 * * * ~/bin/cleanup-ephem.sh") | crontab -
```

## 12.E. Test

Depuis l'atelier :
```bash
~/bin/spawn-torbox.sh
```

Dans la VM Alpine qui s'ouvre :
- Vérifier IP : `ip -br addr` → sur `br-tor` (10.0.31.X)
- Tester Tor : `wget -qO- https://check.torproject.org | grep -i congrat`

Fermer la VM (poweroff depuis la console ou `virsh destroy`) → elle disparaît complètement.

Sur voidhv :
```bash
ls /tmp/torbox-*.qcow2 2>/dev/null
# Vide si cleanup lancé, sinon résidu jusqu'au prochain cron
```

## État attendu en fin de phase 12

- Template `torbox-template.qcow2` en lecture seule sur voidhv
- Scripts `spawn-torbox.sh` / `spawn-i2pbox.sh` / `spawn-directbox.sh` dans la VM atelier
- Lancement d'une VM jetable = une commande, fermeture = destruction automatique
- Cleanup planifié des disques temporaires résiduels

---

# Fin du guide d'installation

Le homelab est complet. Ce que tu as :

- Hôte Void minimal, Wi-Fi géré, KVM complet, Firefox captive, durcissement sysctl
- VM atelier Debian XFCE comme cockpit (virt-manager remote vers voidhv)
- VM obsd-router avec 3 bridges principaux + pf + dhcpd + unbound
- VMs utilisateur : routine, dev/claude, banque (wg1 Paris), bitcoin-node + sparrow, windows 11
- VMs anonymat : Whonix-GW + Whonix-WS, obsd-torgw, obsd-i2pgw, anon-browse (Kicksecure)
- VMs éphémères sur demande
- Accès distant complet via WireGuard inbound + SSH ProxyJump + virt-manager remote
- Mullvad actif sur `br-vpn` (multihop NO→IS) et `br-banque` (Paris), kill-switch pf
- Presse-papier et audio activés ou désactivés par VM selon sensibilité

## Maintenance courante

À faire régulièrement :
- **Void hôte** : `xbps-install -Suy` (hebdomadaire)
- **Debian VMs** : `apt update && apt upgrade` (hebdomadaire)
- **OpenBSD VMs** : `syspatch && pkg_add -u` (mensuel ou à chaque annonce errata)
- **Whonix** : laisser la VM se maintenir via son système d'update automatique, sinon `whonix-repository` + `apt`
- **Kicksecure** : `apt update && apt upgrade`
- **Snapshots de référence** : refaire `phaseXX-ready` après update majeure
- **Mullvad keys** : renouveler annuellement (regénérer et reuploader clé publique)
- **Tor bridges obfs4** : renouveler si blocage FAI évolue
- **Backup des secrets** : clés SSH admin, seed BTC, vault PGP — sur support physique hors-ligne, chiffré

## Checklist de sécurité trimestrielle

- [ ] Vérifier services à l'écoute externe : `ss -tlnp` sur Void + `netstat -an` sur OpenBSDs
- [ ] Tester le kill-switch Mullvad (couper wg0, vérifier fail-closed)
- [ ] Rotation des clés SSH admin si compromission suspectée
- [ ] Vérifier port forwarding ISP : pas d'autres ports ouverts que 51820 UDP
- [ ] Audit des règles pf : `pfctl -sr | wc -l` (comparer à baseline)
- [ ] Tester WG inbound depuis device externe
- [ ] Refaire `am.i.mullvad.net` check depuis routine et banque

## Extensions futures possibles

- **Monitoring** : Prometheus + Grafana dans une VM sur `br-direct`
- **Backup automatisé** : Restic + Backblaze B2 via la VM atelier (disque qcow2 tous les bridges)
- **Mesh** : Tailscale/Headscale pour multi-machines (bureau + homelab)
- **YubiKey SSH** : migration de la clé SSH admin vers une YubiKey FIDO2
- **GPU passthrough** : pour une VM gaming ou ML, si ton hôte a un second GPU
- **DNS anti-ads** : AdGuard Home dans une VM dédiée, point d'entrée DNS remplaçant unbound upstream

---

Pour toute question ou bug durant l'install, ce document + `ARCHITECTURE.md` + `BASE.md` sont self-contained : une IA de debug peut les ingérer comme contexte complet.
