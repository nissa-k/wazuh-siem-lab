# wazuh-siem-lab

Déploiement d'une plateforme **SIEM open source** basée sur [Wazuh](https://wazuh.com/) en conditions réelles, avec détection d'intrusions réseau via **Suricata** et simulation d'attaques depuis **Kali Linux**.

> Réalisé par **Nissa Karadag** — Février–Mars 2026

---

## Architecture

```
192.168.175.0/24 — Réseau NAT-SIEM (VirtualBox)

┌─────────────────────────────┐
│  Serveur Wazuh              │  192.168.175.3
│  Ubuntu 24.04 LTS           │  Wazuh Manager + Indexer + Dashboard
└─────────────────────────────┘
          ▲ agents / logs
┌─────────────────────────────┐
│  Machine cible DVWA         │  192.168.175.6
│  Ubuntu 24.04 LTS           │  Agent Wazuh + Apache2 + Suricata + DVWA
└─────────────────────────────┘
          ▲ attaques simulées
┌─────────────────────────────┐
│  Machine attaquante         │  192.168.175.7
│  Kali Linux                 │  Nmap, Nikto
└─────────────────────────────┘
```

---

## Stack technique

| Composant | Version | Rôle |
|---|---|---|
| Wazuh | 4.14.3 | SIEM — collecte, indexation, dashboard |
| Suricata | latest | IDS — analyse du trafic réseau |
| DVWA | latest | Application web vulnérable (cible) |
| Ubuntu | 24.04 LTS | OS serveur & machine cible |
| VirtualBox | — | Hyperviseur, réseau NAT interne |

---

## Fonctionnalités

- Déploiement **all-in-one** du stack Wazuh (Indexer → Manager → Dashboard)
- Enregistrement sécurisé des agents via clé d'authentification unique
- Intégration des alertes **Suricata** (`eve.json`) dans le pipeline Wazuh
- Détection en temps réel des scans **Nmap** (règles ET SCAN, SID 2024364)
- Visualisation des alertes dans le **Dashboard Wazuh** (module Discover)
- Décodeur Suricata personnalisé (`local_decoder.xml`)

---

## Installation rapide

### 1. Prérequis

- VirtualBox avec un réseau NAT `192.168.175.0/24` (DHCP activé)
- 3 VMs : Serveur Wazuh, Machine DVWA, Kali Linux

### 2. Déploiement du serveur Wazuh

```bash
# Modifier l'IP dans config.yml (Indexer, Manager, Dashboard → 192.168.175.3)
sudo nano config.yml

# Générer les certificats
sudo bash wazuh-install.sh --generate-config-files

# Installer dans l'ordre
sudo bash wazuh-install.sh --wazuh-indexer node-1
sudo bash wazuh-install.sh --start-cluster
sudo bash wazuh-install.sh --wazuh-server wazuh-1
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

Le dashboard est accessible sur : `https://192.168.175.3:443`

### 3. Machine cible DVWA

```bash
# Pile LAMP + DVWA
sudo apt install apache2 mariadb-server php php-mysqli php-gd libapache2-mod-php git unzip -y
sudo git clone https://github.com/digininja/DVWA.git /var/www/html/DVWA

# Suricata IDS
sudo apt install suricata -y
# Éditer /etc/suricata/suricata.yaml → af-packet: enp0s3
sudo systemctl enable suricata && sudo systemctl restart suricata
```

### 4. Agent Wazuh sur DVWA

```bash
# Configurer ossec.conf pour pointer vers 192.168.175.3 (TCP:1514)
# Ajouter le bloc localfile pour Suricata :
# <localfile>
#   <log_format>json</log_format>
#   <location>/var/log/suricata/eve.json</location>
# </localfile>

# Enregistrer l'agent sur le serveur via manage_agents (ID 002)
sudo /var/ossec/bin/manage_agents
```

---

## Simulation d'attaques

```bash
# Depuis Kali Linux — scan SYN stealth
sudo nmap -sS 192.168.175.6
```

Les alertes Suricata (`ET SCAN Possible Nmap User-Agent Observed`) remontent automatiquement dans le Dashboard Wazuh avec classification `Web Application Attack`, priorité 1.

---

## Problèmes connus & résolutions

| Problème | Résolution |
|---|---|
| Verrou `dpkg` (unattended-upgrades) | Arrêt du service avant installation |
| Timeout du Manager au démarrage | `systemctl daemon-reload && systemctl restart wazuh-manager` |
| Agent déconnecté / clé expirée | Réimportation de la clé via `manage_agents` |
| Règles Nmap désactivées dans Suricata | `suricata-update` + activation des règles ET SCAN |
| Répertoire `/var/log/wazuh-indexer/` manquant | Création manuelle + `chown wazuh-indexer` |

---

## Résultats

- **571 alertes** de faible sévérité détectées sur les dernières 24h
- **284 alertes** de sévérité moyenne
- Détection Nmap en temps réel validée end-to-end (Kali → Suricata → Wazuh → Dashboard)

---

## Licence

Projet réalisé dans un cadre pédagogique. Environnement isolé — ne pas déployer en production sans durcissement de la configuration.
