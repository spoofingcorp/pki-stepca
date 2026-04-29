# TP : Déploiement d'une PKI et d'un Serveur Web Sécurisé (LXC sur Proxmox)

## Architecture et Prérequis

Vous allez déployer 3 conteneurs LXC sur Proxmox. Ces conteneurs seront connectés à un Bridge Linux (qui simule votre switch) et placés derrière un firewall pfSense virtuel.

⚠️ **Important : Adaptation réseau**
Dans ce guide, le segment réseau d'exemple utilisé est `192.168.33.0/24`

*   **CT 1 (DNS)** : IP `192.168.33.20`
*   **CT 2 (PKI)** : IP `192.168.33.21`
*   **CT 3 (WEB)** : IP `192.168.33.22`
*   **VM 1 (Client)** : IP `192.168.33.10`

---

## Étape 1 : Création des Conteneurs LXC sur Proxmox

Pour chaque service (DNS, PKI, WEB), créez un conteneur LXC en suivant cette procédure depuis l'interface Proxmox :

1.  Cliquez sur **Create CT** en haut à droite.
2.  **General** :
    *   Hostname : `dns`, `pki` ou `web` (selon le conteneur).
    *   Password : Définissez un mot de passe pour l'utilisateur `root`.
3.  **Template** : Sélectionnez un template récent de distribution Linux (ex: Debian 12).
4.  **Disks** : 8 Go (par défaut) sont amplement suffisants.
5.  **CPU / Memory** : 1 Core et 512 Mo de RAM par conteneur suffisent pour ces services.
6.  **Network** :
    *   **Bridge** : Sélectionnez le bridge correspondant au LAN de votre pfSense (ex: `vmbr1`).
    *   **IPv4** : Statique. Entrez l'IP prévue avec son masque (ex: `192.168.33.20/24`).
    *   **Gateway** : Entrez l'adresse IP LAN de votre pfSense.
7.  **DNS Server** : Laissez par défaut ou pointez vers votre pfSense pour le moment (vous les modifierez plus tard).
8.  Démarrez vos conteneurs.

*(Note : Assurez-vous que le paquet `ssh` est bien installé et démarré sur vos conteneurs pour la suite de l'exercice).*

---

## Étape 2 : Exécution des scripts d'installation

Connectez-vous en `root` sur chacun de vos conteneurs et exécutez les scripts correspondants. 

### 1. Sur le conteneur DNS (`192.168.33.20`)

Créez le script `install_dns.sh`, rendez-le exécutable (`chmod +x install_dns.sh`) et lancez-le :

```bash
#!/bin/bash

# --- VARIABLES A ADAPTER ---
IP_DNS="192.168.33.20"
IP_PKI="192.168.33.21"
IP_WEB="192.168.33.22"
RESEAU_AUTORISE="192.168.0.0/16" # Adaptez à votre sous-réseau
# ---------------------------

# Ajout des entrées dans /etc/hosts
echo "$IP_DNS dns.simplon.local" | tee -a /etc/hosts
echo "$IP_PKI pki.simplon.local" | tee -a /etc/hosts
echo "$IP_WEB web.simplon.local" | tee -a /etc/hosts

# Mise à jour des paquets et installation de Bind9
apt update
apt install -y bind9 bind9utils bind9-doc

# Configuration des options de Bind9
cat << EOF | tee /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    listen-on { any; };
    allow-query { 10.0.0.0/8; 172.16.0.0/12; $RESEAU_AUTORISE; };
    listen-on-v6 { none; };
    recursion yes;
};
EOF

# Configuration de la zone simplon.local
echo 'zone "simplon.local" {
    type master;
    file "/etc/bind/db.simplon.local";
};' | tee /etc/bind/named.conf.local

# Création du fichier de zone
touch /etc/bind/db.simplon.local

echo "\$TTL    604800
@       IN      SOA     dns.simplon.local. root.simplon.local. (
                          2         ; Serial
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.simplon.local.
@       IN      A       $IP_DNS
dns     IN      A       $IP_DNS
pki     IN      A       $IP_PKI
web     IN      A       $IP_WEB" | tee /etc/bind/db.simplon.local

# Vérification et redémarrage
named-checkconf
named-checkzone simplon.local /etc/bind/db.simplon.local
systemctl restart bind9
systemctl enable bind9.service
systemctl status bind9.service --no-pager
```

### 2. Sur le conteneur PKI (`192.168.33.21`)

Créez le script `install_pki.sh`, rendez-le exécutable et lancez-le :

```bash
#!/bin/bash

# --- VARIABLES A ADAPTER ---
IP_DNS="192.168.33.20"
IP_PKI="192.168.33.21"
IP_WEB="192.168.33.22"
# ---------------------------

echo "$IP_DNS dns.simplon.local" | tee -a /etc/hosts
echo "$IP_PKI pki.simplon.local" | tee -a /etc/hosts
echo "$IP_WEB web.simplon.local" | tee -a /etc/hosts

set -e
set -x

DOMAIN="simplon.local"
HOSTNAME="pki.simplon.local"

echo 'START of Install_stepCA.sh script' "$HOSTNAME" "$(hostname -I | awk '{print $1}')"

# Update resolv.conf
echo "[PKI 1] Updating resolv.conf"
{
    cat > /etc/resolv.conf << EOF
domain $DOMAIN
search $DOMAIN
nameserver $IP_DNS
nameserver 8.8.8.8
EOF
} || { echo "Failed to update resolv.conf"; exit 1; }

# Install step-ca
echo "[PKI 2] Installing step-ca"
{
    apt update && apt install -y wget
    wget -q https://dl.step.sm/gh-release/cli/doc-ca-install/v0.19.0/step-cli_0.19.0_amd64.deb && dpkg -i step-cli_0.19.0_amd64.deb
    wget -q https://dl.step.sm/gh-release/certificates/doc-ca-install/v0.19.0/step-ca_0.19.0_amd64.deb && dpkg -i step-ca_0.19.0_amd64.deb
} || { echo "Failed to install step-ca"; exit 1; }

# Configure step-ca
echo "[PKI 3] Configuring step-ca"
{
    echo "password" > /root/password.txt
    step ca init --ssh --deployment-type=standalone --name="Pki" --dns=$HOSTNAME --address=:8443 --provisioner=admin@$DOMAIN --password-file=/root/password.txt
} || { echo "Failed to configure step-ca"; exit 1; }

# Update the step path
mv $(step path) /etc/step-ca
sed -i "s|/root/.step|/etc/step-ca|g" /etc/step-ca/config/defaults.json
sed -i "s|/root/.step|/etc/step-ca|g" /etc/step-ca/config/ca.json

# Create a system user for step-ca
useradd --system --home /etc/step-ca --shell /bin/false step
chown -R step:step /etc/step-ca

# Setup systemd and start service
echo "[PKI 4] Configuring systemd and starting the service"
wget -q https://raw.githubusercontent.com/smallstep/certificates/master/systemd/step-ca.service
mv step-ca.service /etc/systemd/system/
export STEPPATH=/etc/step-ca
echo STEPPATH=/etc/step-ca > /etc/environment
systemctl daemon-reload
systemctl enable --now step-ca
mv /root/password.txt /etc/step-ca/

# Add ACME provisioner
step ca provisioner add acme --type ACME
systemctl restart step-ca.service

echo "step-CA installation and configuration complete."
```

### 3. Sur le conteneur WEB (`192.168.33.22`)

Créez le script `install_web.sh`, rendez-le exécutable et lancez-le :

```bash
#!/bin/bash

# --- VARIABLES A ADAPTER ---
IP_DNS="192.168.33.20"
IP_PKI="192.168.33.21"
IP_WEB="192.168.33.22"
# ---------------------------

echo "$IP_DNS dns.simplon.local" | tee -a /etc/hosts
echo "$IP_PKI pki.simplon.local" | tee -a /etc/hosts
echo "$IP_WEB web.simplon.local" | tee -a /etc/hosts

# Résolution DNS locale pointant vers le conteneur DNS
cat > /etc/resolv.conf << EOF
domain simplon.local
search simplon.local
nameserver $IP_DNS
nameserver 8.8.8.8
EOF

# Mise à jour des paquets et installation d'Apache
apt-get update
apt-get install -y apache2 wget ssh

# Création du dossier de log pour Apache2
mkdir -p /var/log/apache2

# Création de la configuration Apache
cat << EOF | tee /etc/apache2/sites-available/web.simplon.local.conf
<VirtualHost *:80>
    ServerName web.simplon.local
    ServerAlias www.web.simplon.local
    DocumentRoot /var/www/html

    ErrorLog \${APACHE_LOG_DIR}/error.log
    CustomLog \${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
EOF

# Activation du site et modules
a2ensite web.simplon.local.conf
a2dissite 000-default.conf
a2enmod rewrite
systemctl reload apache2

# Installation de Certbot
apt-get install -y certbot python3-certbot-apache

echo "Apache server setup completed."
```

---

## Étape 3 : Exercice Pratique

### A. Informations sur l'infrastructure

*   **PKI** : L'autorité de certification exécute Smallstep. Elle est configurée pour émettre des certificats SSL (ACME). Le certificat racine est dans `/etc/step-ca/certs`.
*   **WEB** : Exécute Apache2, configuré pour répondre sur `[http://web.simplon.local](http://web.simplon.local)`.

### B. Configuration de la VM Client (hôte)

Une fois les conteneurs déployés, vous devez pouvoir accéder au site web depuis une VM Debian ou Windows dans le même segment réseau que l'infra PKI.
⚠️ **Action requise** : Changez la configuration de votre carte réseau pour utiliser **l'IP de votre conteneur DNS** comme DNS primaire (ex: `192.168.33.20`).

### C. Test HTTP et Capture

1.  Ouvrez **Wireshark** sur votre VM Client et lancez une capture sur l'interface réseau connectée au réseau du lab.
2.  Ouvrez votre navigateur et allez sur `[http://web.simplon.local](http://web.simplon.local)`.
3.  Observez le trafic en clair dans Wireshark (Filtre : `http`).

### D. Mise en place du SSL

Votre objectif : Mettre en place SSL sur le serveur web à l'aide de Certbot et de votre autorité PKI interne.
*   **Que constatez-vous lors du premier test en HTTPS ?**
*   **Comment résoudre le problème lié au certificat auto-signé / autorité inconnue ?**

### 1. Importer l'autorité de certification (CA) sur le serveur WEB

Pour que le serveur Web (et Certbot) fasse confiance à notre PKI interne, il faut lui transmettre le certificat racine.

**Sur le serveur PKI :**
Vérifiez la présence des certificats, puis copiez le certificat racine vers le serveur Web :
```bash
cd /etc/step-ca/certs/
ls -l

# On autorise temporairement la connexion SSH root sur le serveur Web (si bloquée par défaut)
# Note: Si scp bloque, faites cette modification sur le serveur WEB d'abord :
# sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/g' /etc/ssh/sshd_config
# systemctl restart ssh

# Envoi du certificat (Adaptez l'IP du serveur web)
scp root_ca.crt root@192.168.33.22:/root/
```

**Sur le serveur WEB :**
Déplacez le certificat reçu et mettez à jour le magasin de confiance de Debian :
```bash
mv /root/root_ca.crt /usr/local/share/ca-certificates/pki_root_ca.crt
update-ca-certificates
```

### 2. Obtenir et déployer le certificat SSL avec Certbot

**Toujours sur le serveur WEB :**
Lancez Certbot en lui indiquant l'URL ACME de votre PKI interne :
```bash
certbot --apache --server https://pki.simplon.local:8443/acme/acme/directory
```
*   **Email** : Renseignez un email (ex: `admin@simplon.local`).
*   **Terms of Service** : Acceptez (A).
*   **EFF** : Refusez (N).
*   **Names** : Sélectionnez `simplon.local` (ou tapez 1).
*   **Redirect** : Choisissez `2` (Redirect) pour forcer tout le trafic HTTP vers HTTPS.

### 3. Test HTTPS et Validation Client

1.  Lancez une nouvelle capture Wireshark.
2.  Allez sur `[https://web.simplon.local/](https://web.simplon.local/)`.
3.  **Analyse** : Le trafic est désormais chiffré. Quel protocole est utilisé ? (Regardez dans Wireshark, vous devriez voir du `TLS 1.3`).
Normalement vous devez voire du TLS 1.3. Si ce n’est pas le cas, votre navigateur n’est pas a jour ou est mal configuré
Dans Chrome, taper > chrome://flags/#tls13-variant
4.  **Alerte de sécurité du navigateur** : Votre navigateur affichera un avertissement de sécurité. C'est normal, votre VM Client ne connaît pas encore la PKI interne !
5.  **Résolution côté client** :
    *   Depuis votre VM Client, rendez-vous sur l'URL : `[https://pki.simplon.local:8443/roots.pem](https://pki.simplon.local:8443/roots.pem)`
    *   Le certificat racine se télécharge.
    *   Importez-le dans le magasin de certificats de votre système d'exploitation ou directement dans les paramètres de votre navigateur (Autorités de certification de confiance).
    *   Rechargez la page web : le cadenas vert doit s'afficher.
