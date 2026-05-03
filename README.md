# 🛡️ Pi-hole + Tailscale — Home Lab sur Raspberry Pi Zero 2 W

<div align="center">

**Projet d'infrastructure réseau personnelle · Hébergement auto-géré · Sécurité & confidentialité**

![Raspberry Pi](https://img.shields.io/badge/Hardware-Raspberry%20Pi%20Zero%202%20W-c51a4a?style=for-the-badge&logo=raspberry-pi&logoColor=white)
![Pi-hole](https://img.shields.io/badge/Pi--hole-DNS%20Sinkhole-96060c?style=for-the-badge)
![Tailscale](https://img.shields.io/badge/Tailscale-VPN%20Mesh-246bfd?style=for-the-badge)
![OS](https://img.shields.io/badge/Raspberry%20Pi%20OS-Lite%2064--bit-4a9e4e?style=for-the-badge&logo=linux&logoColor=white)

</div>

---

## 🎯 Résumé du projet

Ce projet met en place une **infrastructure réseau domestique sécurisée et autonome** sur un Raspberry Pi Zero 2 W. Il combine deux outils complémentaires :

- **Pi-hole** : serveur DNS filtrant qui bloque publicités, trackers et domaines malveillants pour tous les appareils du réseau local — sans installer d'extension sur chaque navigateur.
- **Tailscale** : réseau VPN mesh basé sur WireGuard qui permet d'accéder à l'ensemble de l'infrastructure depuis n'importe où dans le monde, de manière chiffrée.

> 💼 **Compétences démontrées** : administration Linux, réseaux TCP/IP & DNS, déploiement de services auto-hébergés, sécurisation d'accès distants, gestion d'infrastructure embarquée.

---

## 🧱 Stack technique

| Couche | Technologie | Rôle |
|--------|-------------|------|
| Matériel | Raspberry Pi Zero 2 W (ARM Cortex-A53, 512 Mo RAM) | Serveur compact basse consommation |
| OS | Raspberry Pi OS Lite 64-bit (Debian Bookworm) | Système headless optimisé |
| DNS Filtering | Pi-hole v5.x + Lighttpd | Blocage DNS, interface d'administration web |
| VPN | Tailscale (WireGuard) | Accès sécurisé & chiffré à distance |
| Réseau | DHCP statique, configuration DNS sur routeur | Intégration réseau local |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                    INTERNET                         │
└────────────────────┬────────────────────────────────┘
                     │
              ┌──────▼──────┐
              │   Routeur   │  ◄── DNS primaire → Pi-hole
              └──────┬──────┘
                     │ réseau local (192.168.1.254/24)
         ┌───────────▼────────────┐
         │   Raspberry Pi Zero 2W │  IP fixe : 192.168.1.74
         │                        │
         │  ┌──────────────────┐  │
         │  │    Pi-hole       │  │  Port 53 (DNS)
         │  │  DNS Sinkhole    │  │  Interface web :80/admin
         │  └──────────────────┘  │
         │  ┌──────────────────┐  │
         │  │   Tailscale      │  │  IP VPN : 100.x.x.x
         │  │  WireGuard VPN   │  │  Accès chiffré distant
         │  └──────────────────┘  │
         └────────────────────────┘
                     │
        ┌────────────┴─────────────┐
        │                          │
   ┌────▼─────┐             ┌──────▼──────┐
   │ Appareils │             │  Appareil   │
   │  locaux   │             │  distant    │
   │(filtrés)  │             │ (via VPN)   │
   └──────────┘             └─────────────┘
```

**Flux DNS** : Chaque requête DNS des appareils → Pi-hole → liste de blocage → si autorisé, résolution via Cloudflare (1.1.1.1) ou Google (8.8.8.8).

---

## 🚀 Installation & déploiement

### Étape 1 — Préparation du système

Flasher la carte SD avec **Raspberry Pi Imager** (Raspberry Pi OS Lite 64-bit), en activant via les paramètres avancés :

```
✅ SSH activé (authentification par mot de passe)
✅ Identifiants configurés
✅ Wi-Fi 2.4 GHz renseigné
✅ Locale : Europe/Paris
```

Connexion initiale et mise à jour système :

```bash
ssh pi@pihole.local
sudo apt update && sudo apt upgrade -y
```

Attribution d'une **IP statique** (obligatoire pour un serveur DNS stable) :

```bash
# /etc/dhcpcd.conf
interface wlan0
static ip_address=192.168.1.74/24
static routers=192.168.1.254
static domain_name_servers=192.168.1.74
```

---

### Étape 2 — Déploiement de Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

Paramètres d'installation retenus :

| Paramètre | Valeur |
|-----------|--------|
| Interface réseau | `wlan0` |
| DNS upstream | Cloudflare `1.1.1.1` |
| Interface web admin | Activée (Lighttpd) |
| Query logging | Activé |

Changement du mot de passe admin :

```bash
pihole -a -p <mot_de_passe>
```

Interface accessible sur : `http://192.168.1.74/admin`

---

### Étape 3 — Déploiement de Tailscale

```bash
# Installation
curl -fsSL https://tailscale.com/install.sh | sh

# Authentification (génère une URL à ouvrir dans le navigateur)
sudo tailscale up

# Activation au démarrage
sudo systemctl enable tailscaled
```

**Configuration DNS sur Tailscale Admin Console** :

```
DNS → Add nameserver → Custom → 100.x.x.x (IP Tailscale du Pi)
✅ Override local DNS activé
```

→ Tous les appareils du réseau Tailscale profitent du filtrage Pi-hole, y compris en mobilité.

---

### Étape 4 — Intégration réseau local

Dans l'interface de la box/routeur, définir le **serveur DNS DHCP primaire** à `192.168.1.74`.

Vérification du filtrage DNS :

```bash
nslookup doubleclick.net 192.168.1.74
# Réponse attendue : 0.0.0.0 (domaine bloqué ✅)
```

---

## 🔧 Commandes de référence

### Pi-hole

```bash
pihole status              # État du service
pihole -up                 # Mise à jour Pi-hole
pihole -g                  # Mise à jour des listes (gravity)
pihole -t                  # Logs DNS en temps réel
pihole disable 300         # Désactivation temporaire (300s)
pihole enable              # Réactivation
pihole -w domaine.com      # Ajouter à la whitelist
pihole -b domaine.com      # Ajouter à la blacklist
pihole -a teleporter       # Export de la configuration
```

### Tailscale

```bash
tailscale status             # Appareils connectés au réseau VPN
tailscale ip                 # Adresse IP Tailscale du nœud
sudo tailscale up            # Connexion / ré-authentification
sudo tailscale down          # Déconnexion
journalctl -u tailscaled -f  # Logs en temps réel
```

### Système

```bash
sudo apt update && sudo apt upgrade -y   # Mise à jour complète
vcgencmd measure_temp                    # Température CPU
df -h                                    # Espace disque
htop                                     # Monitoring CPU/RAM
sudo systemctl disable bluetooth         # Désactiver Bluetooth (optimisation)
```

---

## 📊 Résultats & performances

> Chiffres observés après quelques jours d'utilisation sur un réseau domestique standard.

| Métrique | Valeur |
|----------|--------|
| Requêtes DNS bloquées | ~25–35% du trafic total |
| Temps de réponse DNS | < 5 ms sur le réseau local |
| Consommation RAM (Pi-hole + Tailscale) | ~180–220 Mo / 512 Mo |
| Charge CPU moyenne | < 15% |
| Consommation électrique | ~2,5 W |

---

## 📚 Références

- [Documentation Pi-hole](https://docs.pi-hole.net)
- [Documentation Tailscale](https://tailscale.com/kb)
- [Listes de blocage — Firebog](https://firebog.net)
- [WireGuard Protocol](https://www.wireguard.com/papers/wireguard.pdf)

---

<div align="center">

*Projet personnel · Infrastructure auto-hébergée · 2026*

</div>
