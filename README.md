# 🔧 pfSense — Maîtrise des fonctionnalités avancées

> VPN, haute disponibilité réseau, filtrage web, threat intelligence et supervision du trafic : un pare-feu open source poussé dans ses retranchements pour couvrir ce qu'une PME attend d'une solution réseau d'entreprise, sans le budget d'un NGFW propriétaire.

![pfSense](https://img.shields.io/badge/pfSense-CE-212121?logo=pfsense&logoColor=white)
![VirtualBox](https://img.shields.io/badge/Lab-VirtualBox-183A61?logo=virtualbox&logoColor=white)
![Niveau](https://img.shields.io/badge/Niveau-Interm%C3%A9diaire%20%E2%86%92%20Avanc%C3%A9-blue)
---

## Introduction

### Qui suis-je ?

Je m'appelle **Rock Dannon**, étudiant en **Master 2 Réseaux Informatiques et Télécommunications**. J'ai plus de deux ans d'expérience terrain : support utilisateurs N1/N2, réseaux d'entreprise (LAN/WAN, VLAN, Wi-Fi professionnel, téléphonie IP, câblage structuré) et encadrement d'équipes techniques. En parallèle, je construis mes propres laboratoires pour approfondir la **cybersécurité**, la **supervision réseau** Ce dépôt fait partie des travaux que je documente pour montrer concrètement ce que je sais faire.

### Ce projet

Ce laboratoire fait suite à mon [lab d'attaque/défense pfSense](https://github.com/Poliakoff29/LabCybersecurity). Une fois le pare-feu éprouvé contre des attaques réelles (DoS, brute-force SSH, injection SQL, SIEM Wazuh), j'ai voulu explorer l'ensemble de ses **fonctionnalités de niveau entreprise** : celles qu'on attend concrètement d'un poste d'administrateur réseau ou d'analyste SOC N2.

**12 fonctionnalités** sont couvertes, chacune sous forme de tutoriel reproductible :

| # | Fonctionnalité | Ce que ça apporte |
|---|---|---|
| 1 | Sauvegarde et mise à jour | Filet de sécurité avant tout changement |
| 2 | Alias et IP virtuelles | Règles maintenables, plusieurs IP sur une interface |
| 3 | NAT et Port Forwarding | Exposer un service interne de façon contrôlée |
| 4 | DDNS | Nom de domaine fixe malgré une IP publique changeante |
| 5 | Multi-WAN | Basculer ou répartir la charge entre deux accès Internet |
| 6 | Traffic Shaper (QoS) | Plafonner ou prioriser le débit |
| 7 | Squid et SquidGuard | Proxy web et filtrage par domaines |
| 8 | OpenVPN | Accès distant chiffré pour un utilisateur |
| 9 | IPsec site-à-site | Relier deux réseaux entiers |
| 10 | pfBlockerNG | Threat intelligence : listes IP/DNS et GeoIP |
| 11 | ntopng | Visibilité temps réel sur le trafic |
| 12 | Test de charge | Vérifier la tenue de l'ensemble sous contrainte |

### À qui s'adresse ce tutoriel ?

- Aux étudiants et débutants en réseau qui veulent monter un lab pfSense complet.
- Aux techniciens et administrateurs de PME qui cherchent une alternative open source aux pare-feux propriétaires.
- Aux recruteurs et responsables techniques qui veulent voir ma méthode de travail.

### Ce que vous saurez faire à la fin

- Sauvegarder, restaurer et faire évoluer une configuration pfSense en sécurité.
- Publier un service interne (NAT), avec une IP virtuelle dédiée et un nom DDNS.
- Mettre en place un second accès Internet avec bascule automatique et répartition de charge.
- Limiter le débit d'une machine et expliquer la différence entre *limiter* et *queue*.
- Mettre en place un proxy filtrant et en connaître les limites.
- Déployer un VPN d'accès distant (OpenVPN) et un VPN site-à-site (IPsec).
- Bloquer des menaces par listes de réputation, DNS et pays (pfBlockerNG).
- Observer et analyser le trafic (ntopng) et valider la résistance du pare-feu sous charge.

> **Note de méthode.** Chaque fonctionnalité a été configurée, testée avec un scénario concret (pas seulement activée dans l'interface), puis validée par une vérification mesurable : un test qui échoue avant la configuration et réussit après, ou l'inverse selon le cas.

> **Avertissement.** Ce lab est réalisé dans un environnement isolé, sur mes propres machines virtuelles. Les tests de charge et de sécurité ne doivent être faits que sur **votre propre lab**, jamais sur un réseau dont vous n'êtes pas propriétaire. Les adresses IP, noms et identifiants utilisés ici sont des valeurs de démonstration : remplacez-les par les vôtres.

---

## Architecture du lab

| Élément | Détail |
|---|---|
| **pfSense** | WAN en mode pont (réseau physique) · LAN `192.168.1.0/24` (passerelle `192.168.1.1`) |
| **Ubuntu Server** | Hôte applicatif de test, `192.168.1.103` |
| **Windows 10** | Poste client de test, `192.168.1.104` |
| **Hyperviseur** | Oracle VirtualBox |
| **Version de pfSense** | pfSense CE `[ 2.9.0]` |

```text
                    Réseau physique (box / routeur)
                                │
                         [ WAN : pont ]
                        ┌───────┴────────┐
                        │    pfSense     │
                        │ LAN 192.168.1.1│
                        └───────┬────────┘
                                │ Réseau interne VirtualBox 192.168.1.0/24
                   ┌────────────┴────────────┐
          Ubuntu Server 192.168.1.103   Windows 10 192.168.1.104
```

Pour la fonctionnalité Multi-WAN (section 5), une **troisième carte réseau** est ajoutée à la machine pfSense et simule un second accès Internet.

---

## Sommaire

0. [Préparation de l'environnement](#0-préparation-de-lenvironnement)
1. [Sauvegarde et mise à jour](#1-sauvegarde-et-mise-à-jour)
2. [Alias et IP virtuelles](#2-alias-et-ip-virtuelles)
3. [NAT et Port Forwarding](#3-nat-et-port-forwarding)
4. [DDNS (Dynamic DNS)](#4-ddns-dynamic-dns)
5. [Multi-WAN (failover et load balancing)](#5-multi-wan-failover-et-load-balancing)
6. [Traffic Shaper (QoS)](#6-traffic-shaper-qos)
7. [Squid et SquidGuard](#7-squid-et-squidguard)
8. [OpenVPN (accès distant)](#8-openvpn-accès-distant)
9. [IPsec (site-à-site)](#9-ipsec-site-à-site)
10. [pfBlockerNG](#10-pfblockerng)
11. [ntopng](#11-ntopng)
12. [Test de charge (stress test)](#12-test-de-charge-stress-test)
13. [Conclusion](#conclusion)
14. [Ressources](#ressources)

---

## 0. Préparation de l'environnement

### Prérequis

- Un pfSense déjà installé (voir le [lab d'attaque/défense](https://github.com/Poliakoff29/LabCybersecurity)) avec deux cartes réseau : WAN en accès par pont et LAN en réseau interne.
- Une VM **Ubuntu Server** et une VM **Windows 10** connectées au même réseau interne que le LAN de pfSense.
- Accès à l'interface web de pfSense depuis le poste Windows : `https://192.168.1.1`.
- Un compte No-IP gratuit (section 4).

### Réglages réseau du lab

1. Sur le **WAN** de pfSense (*Interfaces ▸ WAN*), décochez **Block private networks and loopback addresses** : dans un lab, le WAN est branché sur un réseau privé (votre box), et cette option bloquerait tout le trafic entrant de test.
2. Vérifiez que les VM Ubuntu et Windows obtiennent bien une IP dans `192.168.1.0/24` (DHCP de pfSense : *Services ▸ DHCP Server ▸ LAN*).
3. Changez le mot de passe administrateur par défaut de pfSense (*System ▸ User Manager*).
4. Sur Ubuntu Server, installez un serveur web de test et un outil de mesure :

```bash
sudo apt update
sudo apt install -y apache2 iperf3 curl
sudo systemctl enable --now apache2
```

5. Sur Windows 10 (édition Pro), activez le Bureau à distance (*Paramètres ▸ Système ▸ Bureau à distance*) pour le test de la section 3.

---

## 1. Sauvegarde et mise à jour

### Objectif
Avant de toucher à quoi que ce soit : exporter la configuration et vérifier l'état des mises à jour. Un filet de sécurité indispensable avant d'enchaîner des changements sur un pare-feu en service. Une configuration qui casse se restaure en un clic plutôt qu'en reconfigurant tout à la main.

### Procédure

1. Ouvrez **Diagnostics ▸ Backup & Restore**, onglet *Backup & Restore*.
2. Dans *Backup Configuration*, laissez **Backup area : All**.
3. (Recommandé) Cochez **Encrypt this configuration file** et saisissez un mot de passe : le fichier contient des secrets (clés, mots de passe).
4. Cliquez sur **Download configuration as XML** et conservez le fichier `config-pfSense-AAAAMMJJ.xml` **hors de la VM** (par exemple sur la machine hôte).
5. Ouvrez **System ▸ Update**, onglet *System Update* : vérifiez la version installée et la branche suivie. Si une version plus récente existe, appliquez la mise à jour après avoir fait la sauvegarde.
6. Prenez l'habitude d'exporter une nouvelle sauvegarde **avant chaque section** de ce tutoriel.

### Restaurer une configuration

*Diagnostics ▸ Backup & Restore ▸ Restore Configuration* : choisissez le fichier XML (et le mot de passe s'il est chiffré), cliquez sur **Restore Configuration**. pfSense redémarre avec l'ancienne configuration.

### Vérification
- Le fichier XML est bien téléchargé et s'ouvre dans un éditeur de texte.
- Test de restauration : modifiez un réglage sans importance, restaurez la sauvegarde et constatez le retour à l'état initial.

<img width="1193" height="520" alt="pfsense-backup-config-screen" src="https://github.com/user-attachments/assets/06ce4bb9-5910-4c64-a486-a9dff340d2ee" />

---

## 2. Alias et IP virtuelles

### Objectif
- **Alias** : un nom réutilisable représentant un groupe d'IP, de réseaux ou de ports (par exemple `LAN_Servers`, `Web_Ports`). Il centralise la maintenance : modifier l'alias une fois met à jour toutes les règles qui l'utilisent, au lieu de les retrouver une par une.
- **IP virtuelle (VIP)** : permet à une interface physique unique de porter **plusieurs adresses IP**. Utile pour exposer plusieurs services sur des IP publiques distinctes malgré un seul pare-feu, ou pour la haute disponibilité (CARP) entre deux pfSense.

### Procédure : alias

1. **Firewall ▸ Aliases ▸ IP ▸ Add**
   - Name : `LAN_Servers`
   - Type : *Host(s)*
   - IP or FQDN : `192.168.1.103` et `192.168.1.104` (un champ par entrée)
   - Description : *Machines internes du lab*
   - **Save**
2. **Firewall ▸ Aliases ▸ Ports ▸ Add**
   - Name : `Web_Ports`
   - Type : *Port(s)*
   - Port : `80`, `443`, `8080`
   - **Save**, puis **Apply Changes**.

> Les noms d'alias ne peuvent contenir que des lettres, des chiffres et des tirets bas.

### Procédure : IP virtuelle

1. **Firewall ▸ Virtual IPs ▸ Add**
   - Type : **IP Alias**
   - Interface : **WAN**
   - Address type : *Single address*
   - Address(es) : une IP **libre** de votre réseau WAN, par exemple `<IP_VIRTUELLE_WAN>` avec le même masque que le WAN (par exemple `/24`)
   - Description : *VIP RDP*
2. **Save**, puis **Apply Changes**.

> Vérifiez que l'IP choisie n'est utilisée par aucune autre machine du réseau physique (un `ping` doit rester sans réponse avant l'ajout).

### Vérification
- Les alias apparaissent dans *Firewall ▸ Aliases* et sont proposés dans les listes déroulantes des règles.
- La VIP répond au `ping` depuis une machine du réseau WAN après application.

<img width="1210" height="529" alt="pfsense-alias-list" src="https://github.com/user-attachments/assets/fc8feac8-60e2-4771-b86f-0426695c1c96" />

<img width="1224" height="553" alt="pfsense-virtual-ip" src="https://github.com/user-attachments/assets/588c242a-4fb2-4da4-9282-4cf5a5d646fd" />

---

## 3. NAT et Port Forwarding

### Objectif
Le **NAT sortant** masque les machines du LAN derrière l'IP WAN. Le **Port Forwarding** fait l'inverse : il expose un service interne à l'extérieur en redirigeant un port entrant vers une IP du LAN.

Deux redirections sont mises en place pour démontrer l'intérêt des VIP : un service web (port 8080 vers Ubuntu:80) sur l'IP WAN principale, et un accès RDP (port 3389) sur la VIP dédiée vers Windows 10. Deux services distincts, chacun sur sa propre IP, sans collision de ports.

### Procédure

**Redirection 1 : site web Ubuntu**

1. **Firewall ▸ NAT ▸ Port Forward ▸ Add**
   - Interface : **WAN**
   - Protocol : **TCP**
   - Destination : **WAN address**
   - Destination port range : *(other)* `8080`
   - Redirect target IP : `192.168.1.103` (ou l'alias `LAN_Servers` si vous ciblez une machine précise)
   - Redirect target port : **HTTP** (`80`)
   - Description : *Web Ubuntu*
   - Filter rule association : **Add associated filter rule**
2. **Save**.

**Redirection 2 : RDP sur la VIP**

1. **Firewall ▸ NAT ▸ Port Forward ▸ Add**
   - Interface : **WAN**
   - Protocol : **TCP**
   - Destination : *Address or Alias* ▸ la **VIP** créée à la section 2
   - Destination port range : **MS RDP** (`3389`)
   - Redirect target IP : `192.168.1.104`
   - Redirect target port : **MS RDP** (`3389`)
   - Description : *RDP Windows 10 via VIP*
   - Filter rule association : **Add associated filter rule**
2. **Save**, puis **Apply Changes**.

### Vérification
Les tests se font **depuis le côté WAN** (par exemple la machine hôte), pas depuis le LAN (le NAT reflection n'est pas activé) :

1. Ouvrez `http://<IP_WAN_PFSENSE>:8080` : la page par défaut d'Apache sur Ubuntu doit s'afficher.
2. Ouvrez une connexion Bureau à distance vers `<IP_VIRTUELLE_WAN>` : l'écran de connexion de Windows 10 doit apparaître.
3. Consultez **Status ▸ System Logs ▸ Firewall** : les connexions autorisées par les règles associées sont visibles.

> ⚠️ **Sécurité.** Publier directement le RDP (port 3389) sur Internet est une très mauvaise pratique en production : c'est une cible privilégiée des attaques par force brute. Ici, c'est une démonstration en lab isolé. En entreprise, on passe par un VPN (sections 8 et 9) ou par une passerelle d'accès sécurisée, et on restreint la source autorisée.

<img width="1356" height="458" alt="nat" src="https://github.com/user-attachments/assets/1a53ab3f-6c92-4e30-98b8-c047623834d7" />

---

## 4. DDNS (Dynamic DNS)

### Objectif
Les connexions grand public ont une IP publique qui change régulièrement. Le **Dynamic DNS** maintient un nom de domaine fixe (par exemple `<rocklab.ddns.net`) toujours à jour avec l'IP actuelle, grâce à un client intégré à pfSense qui pousse les changements vers un fournisseur DDNS (ici **DuckDNS**).

> **Point technique rencontré.** Avec une connexion en **double NAT**, pfSense ne voit pas directement sa vraie IP publique sur son interface WAN. Le client DDNS doit donc utiliser un **service de détection d'IP externe** plutôt que l'IP brute de l'interface. La documentation Netgate décrit ces *Check IP Services* comme destinés aux cas où le WAN se trouve derrière un équipement NAT en amont.

### Procédure

**Côté DuckDNS**

1. Connectez-vous sur [noip.com](https://www.noip.com/) et créez un sous-domaine (par exemple `<rocklab.ddns.net>`).

**Côté pfSense**

1. Vérifiez le service de détection d'IP : **Services ▸ Dynamic DNS ▸ Check IP Services**. Par défaut, un service externe est déjà défini ; vous pouvez en ajouter un autre au besoin.
2. **Services ▸ Dynamic DNS ▸ Dynamic DNS Clients ▸ Add**
   - Service Type : **DuckDNS** (si votre version le propose ; sinon **Custom**, voir ci-dessous)
   - Interface to monitor : **WAN**
   - Hostname : `<rocklab.ddns.net>`
   - Password / Token : votre jeton DuckDNS
   - Description : *DDNS DuckDNS*
   - Si la case ou l'option de détection d'IP est proposée, forcez l'utilisation du **service de détection externe** (*Check IP Service*, mode « toujours »).
3. **Save**.

**Variante « Custom »** si noip n'est pas dans la liste :
- Service Type : **Custom**
- Update URL : `https://www.noip.com/update?domains=<votre-sous-domaine>&token=<VOTRE_TOKEN>&ip=%IP%`
- Result Match : `OK`

### Vérification

1. **Status ▸ Dynamic DNS** : l'adresse en cache doit s'afficher **en vert** et correspondre à votre IP publique.
2. Depuis Windows 10, comparez avec votre IP publique réelle (un site « quelle est mon IP ») et résolvez le nom :

```powershell
nslookup rocklab.ddns.net
```

3. Les deux adresses doivent concorder.

<img width="1272" height="624" alt="DDNS" src="https://github.com/user-attachments/assets/fc6464e8-c6e1-46f2-8dd8-53eee50ffa59" />

---

## 5. Multi-WAN (failover et load balancing)

### Objectif
Brancher **deux accès Internet** sur pfSense permet soit de **basculer automatiquement** si l'un tombe (*failover*), soit de **répartir la charge** entre les deux en continu (*load balancing*). On peut aussi combiner les deux avec un seul *Gateway Group* dont les deux passerelles sont au même niveau (*Tier*).

C'est l'une des fonctions les plus utiles pour une entreprise à budget limité : pfSense, open source, offre un multi-WAN sans les licences habituellement nécessaires.

> **Simulation du second accès.** Faute d'un deuxième accès Internet réel, le second WAN est simulé avec un **réseau NAT VirtualBox** indépendant du pont utilisé pour le WAN principal. Cela suffit pour que pfSense gère deux passerelles réellement distinctes et teste la bascule en conditions quasi réelles. Limite : les deux sorties passent en réalité par la même connexion physique, donc l'IP publique observée est identique.

La détection de panne et le retrait/réintégration automatique d'une passerelle sont gérés par **`dpinger`**, qui surveille chaque passerelle en continu par des sondes régulières (ping vers une adresse de surveillance) et déclenche la bascule en quelques secondes.

### Procédure

**1. Ajouter la deuxième interface WAN**

1. Éteignez la VM pfSense. Dans VirtualBox, activez un **troisième adaptateur réseau** en mode **NAT** (ou *Réseau NAT*) et redémarrez.
2. **Interfaces ▸ Assignments** : dans *Available network ports*, ajoutez le nouveau port (il apparaît comme `OPT1`).
3. **Interfaces ▸ OPT1**
   - Enable : coché
   - Description : `WAN2`
   - IPv4 Configuration Type : **DHCP**
   - Décochez **Block private networks and loopback addresses** (le réseau NAT VirtualBox est privé)
   - **Save**, puis **Apply Changes**.

**2. Régler les passerelles**

1. **System ▸ Routing ▸ Gateways** : deux passerelles DHCP existent (`WAN_DHCP` et `WAN2_DHCP`).
2. Modifiez chacune et donnez-leur une **adresse de surveillance différente** (champ *Monitor IP*), par exemple :
   - `WAN_DHCP` : `8.8.8.8`
   - `WAN2_DHCP` : `1.1.1.1`
   Deux adresses distinctes évitent qu'une seule cible fasse croire à une panne des deux liens.
3. **Save**, puis **Apply Changes**.

**3. Créer les groupes de passerelles**

**System ▸ Routing ▸ Gateway Groups ▸ Add**

| Groupe | `WAN_DHCP` | `WAN2_DHCP` | Usage |
|---|---|---|---|
| `Failover_WAN` | Tier 1 | Tier 2 | Bascule : WAN2 ne sert que si WAN tombe |
| `LB_WAN` | Tier 1 | Tier 1 | Répartition de charge sur les deux liens |

Pour chaque groupe : Trigger Level = **Member Down** (ou *Packet Loss or High Latency* pour réagir aussi à la dégradation), puis **Save**.

**4. Utiliser le groupe dans une règle LAN**

1. **Firewall ▸ Rules ▸ LAN** : éditez la règle autorisant le trafic du LAN vers *any*.
2. **Advanced Options ▸ Gateway** : sélectionnez `Failover_WAN` (puis `LB_WAN` pour le test de répartition).
3. **Save**, puis **Apply Changes**.

**5. DNS et connexions persistantes**

1. **System ▸ General Setup** : ajoutez deux serveurs DNS en associant chacun à une passerelle différente (par exemple `8.8.8.8` avec `WAN_DHCP` et `1.1.1.1` avec `WAN2_DHCP`), pour que la résolution survive à la perte d'un lien.
2. (Optionnel) **System ▸ Advanced ▸ Miscellaneous** : *Use sticky connections* garde un même client sur la même passerelle pendant une session, et *State Killing on Gateway Failure* ferme les connexions qui passaient par le lien tombé.

<!-- 📸 Ajouter ici une capture de la page System ▸ Routing ▸ Gateway Groups -->

### Vérification

**Test de bascule (`Failover_WAN`)**

1. **Status ▸ Gateways** : les deux passerelles doivent être **Online**.
2. Depuis Ubuntu, lancez un ping continu :

```bash
ping 8.8.8.8
```

3. Dans VirtualBox, coupez le lien du WAN principal (*Périphériques ▸ Réseau ▸ décocher « Connecter le câble »*).
4. Observez *Status ▸ Gateways* : `WAN_DHCP` passe en **Offline**, et le groupe bascule sur `WAN2_DHCP`. Le ping reprend après quelques secondes.
5. Rebranchez le lien : la passerelle revient **Online** et le trafic retourne sur le WAN principal.

**Test de répartition (`LB_WAN`)**

1. Lancez plusieurs téléchargements en parallèle depuis Ubuntu.
2. Observez **Status ▸ Traffic Graph** (ou les compteurs de l'interface sur le tableau de bord) : du trafic doit apparaître sur **les deux** interfaces WAN.

<img width="1280" height="572" alt="Capture d&#39;écran 2026-11-08 193201" src="https://github.com/user-attachments/assets/ea5bfef6-d47e-46de-b9fb-fd126d040f33" />

---

## 6. Traffic Shaper (QoS)

### Objectif
Deux mécanismes distincts répondent à des besoins différents :

- Les **Limiters** plafonnent un débit strict, indépendamment de la charge du réseau : ils garantissent un **plafond**.
- Les **Queues** gèrent une priorité relative quand le lien sature, sans rien brider tant qu'il y a de la bande disponible : elles garantissent un **ordre de passage**.

Mis en pratique ici : un **Limiter à 1 Mbit/s** appliqué à Ubuntu Server, validé par un test de débit avant/après.

### Procédure

**1. Créer les deux limiters** (téléchargement et envoi)

**Firewall ▸ Traffic Shaper ▸ Limiters ▸ New Limiter**

| Champ | Limiter 1 | Limiter 2 |
|---|---|---|
| Enable | coché | coché |
| Name | `Ubuntu_Download` | `Ubuntu_Upload` |
| Bandwidth | `1` | `1` |
| Bw Type | Mbit/s | Mbit/s |
| Mask | None *(ou Destination address)* | None *(ou Source address)* |

> Bonne pratique de la documentation Netgate : nommez les limiters `Download` / `Upload` plutôt que `In` / `Out`, car le sens « in » ou « out » change selon l'interface où la règle s'applique.

**2. Créer la règle qui y envoie le trafic d'Ubuntu**

1. **Firewall ▸ Rules ▸ LAN ▸ Add** (en haut de la liste, **au-dessus** de la règle « LAN to any »)
   - Action : **Pass**
   - Protocol : **Any**
   - Source : **Address or Alias** ▸ `192.168.1.103`
   - Destination : **Any**
   - Description : *Limite Ubuntu 1 Mbit/s*
2. **Advanced Options ▸ In / Out pipe** (affichage de la section : bouton *Display Advanced*)
   - **In** : `Ubuntu_Upload`
   - **Out** : `Ubuntu_Download`
3. **Save**, puis **Apply Changes**.

> Sur une règle d'interface **LAN** : *In* correspond au trafic qui entre dans l'interface LAN depuis le client (donc l'**envoi**), *Out* à celui qui en sort vers le client (donc le **téléchargement**).

**3. Réinitialiser les états**

Les connexions déjà ouvertes gardent leur ancienne règle : **Diagnostics ▸ States ▸ Reset States** avant le test.

### Vérification (avant / après)

1. **Avant** de créer la règle : depuis Ubuntu, mesurez le débit.

```bash
sudo apt install -y speedtest-cli     # ou utilisez iperf3 / un gros téléchargement
speedtest-cli --simple
# ou : curl -o /dev/null -w "%{speed_download}\n" http://speedtest.tele2.net/100MB.zip
```

2. Appliquez la règle, réinitialisez les états, puis **refaites exactement le même test**.
3. Résultat attendu : passage de plusieurs dizaines de Mbit/s à un **plafond net autour de 1 Mbit/s**.
4. Vérifiez que **Windows 10** n'est pas limité (la règle ne cible qu'Ubuntu).

<img width="862" height="660" alt="avant" src="https://github.com/user-attachments/assets/de006ff2-f748-4fb3-b050-848f11b1ab31" />

<img width="684" height="688" alt="apres" src="https://github.com/user-attachments/assets/376b610d-c144-4faa-bb35-03aacce50e34" />

---

## 7. Squid et SquidGuard

> ⚠️ **Avertissement important.** La documentation Netgate indique que les paquets **Squid, SquidGuard et Lightsquid sont dépréciés** dans pfSense Plus et pfSense CE, en raison de nombreuses vulnérabilités amont non corrigées, et recommande de les désinstaller. Cette section est donc **pédagogique, pour un lab isolé**. En production, préférez un filtrage DNS (voir la section 10, pfBlockerNG) ou une solution de proxy maintenue.

### Objectif
**Squid** est un proxy web : le trafic HTTP du LAN transite par pfSense avant d'aller sur Internet, ce qui permet la journalisation et sert de point d'ancrage au filtrage. **SquidGuard** s'y greffe pour bloquer l'accès à des domaines ou à des catégories entières (listes noires).

Il est configuré en **mode proxy explicite** (le client indique l'adresse du proxy) plutôt qu'en mode transparent, pour filtrer le HTTPS par nom de domaine **sans déployer de certificat d'interception SSL** sur chaque poste.

### Procédure

**1. Installer les paquets**

**System ▸ Package Manager ▸ Available Packages** : installez `squid`, puis `squidGuard`.

**2. Configurer Squid**

**Services ▸ Squid Proxy Server ▸ General**
- Enable Squid Proxy : coché
- Proxy Interface(s) : **LAN**
- Proxy Port : `3128`
- Transparent HTTP Proxy : **décoché** (mode explicite)
- Allow Users on Interface : coché
- **Save**

**3. Configurer SquidGuard**

1. **Services ▸ Proxy Filter ▸ General settings** : cochez **Enable**, puis **Save**.
2. Onglet **Target Categories ▸ Add** :
   - Name : `Reseaux_sociaux`
   - Domain List : un domaine par ligne, par exemple :

```text
facebook.com
tiktok.com
```

   - **Save**
3. Onglet **Common ACL** ▸ *Target Rules List* : pour `Reseaux_sociaux`, choisissez **deny** ; laissez **Default access [all]** sur **allow** ; **Save**.
4. Retournez dans **General settings** et cliquez sur **Apply**.

**4. Configurer le client (Windows 10)**

*Paramètres ▸ Réseau et Internet ▸ Proxy ▸ Configuration manuelle* : adresse `192.168.1.1`, port `3128`.

**5. (Optionnel) Forcer le passage par le proxy**

Pour empêcher de contourner le proxy, ajoutez dans **Firewall ▸ Rules ▸ LAN** une règle de **blocage** des ports TCP 80 et 443 depuis le LAN, **sauf** pour pfSense lui-même.

### Vérification

1. Depuis Windows 10, ouvrez `http://www.facebook.com` puis `https://www.facebook.com` : la page de blocage de SquidGuard doit apparaître.
2. Ouvrez un site non listé : il doit s'afficher normalement.
3. Consultez les journaux : **Status ▸ System Logs ▸ Packages ▸ Proxy Filter** et l'onglet *Real Time* du proxy Squid.

### Limite technique importante

Ni Squid ni SquidGuard ne peuvent identifier un **service applicatif précis** (par exemple YouTube) de façon fiable : ils raisonnent par domaine. Ce niveau de classification (« App-ID ») est propre aux pare-feux de nouvelle génération (Palo Alto, Fortinet…), qui inspectent les métadonnées TLS plutôt que le simple port ou la simple IP. Le blocage HTTPS par domaine peut aussi se montrer incohérent selon les sites : testez toujours.

<img width="1078" height="423" alt="Capture d&#39;écran 2026-11-07 200227" src="https://github.com/user-attachments/assets/2cff3d07-83d6-41c9-809b-7f2c71d9d7b9" />

---

## 8. OpenVPN (accès distant)

### Objectif
VPN d'accès distant (« *road warrior* ») : un poste externe se connecte à pfSense via un **tunnel chiffré** et rejoint virtuellement le réseau interne, comme s'il y était physiquement branché. Utile pour le télétravail ou l'administration à distance sécurisée.

### Procédure

**1. Installer l'outil d'export des configurations clientes**

**System ▸ Package Manager ▸ Available Packages** : installez `openvpn-client-export`.

**2. Lancer l'assistant OpenVPN**

**VPN ▸ OpenVPN ▸ Wizards**

1. **Type of Server** : *Local User Access*.
2. **Certificate Authority** : *Add new CA* ▸ nom (par exemple `CA-Lab-pfSense`), longueur de clé 2048 bits ou plus, durée de validité, informations d'organisation. **Add new CA**.
3. **Server Certificate** : *Add new Certificate* ▸ nom (par exemple `OpenVPN-Server-Cert`). **Create new Certificate**.
4. **Server Setup** :
   - Interface : **WAN**
   - Protocol : **UDP on IPv4 only**
   - Local Port : `1194`
   - TLS Authentication : coché, clé TLS générée automatiquement
   - DH Parameters Length : `2048`
   - Data Encryption Algorithms : **AES-256-GCM** (valeur par défaut)
   - Auth Digest Algorithm : **SHA256**
   - **Tunnel Network** : `10.8.0.0/24` (réseau virtuel réservé aux clients VPN)
   - **Local Network** : `192.168.1.0/24` (réseau interne rendu accessible)
   - Redirect Gateway : décoché (seul le trafic vers le LAN passe par le tunnel)
5. **Firewall Rule Configuration** : cochez **Firewall Rule** (autorise la connexion au serveur VPN sur le WAN) et **OpenVPN rule** (autorise le trafic dans le tunnel).
6. **Finish**.

**3. Créer l'utilisateur VPN**

**System ▸ User Manager ▸ Users ▸ Add** : nom d'utilisateur (`vpnuser`), mot de passe fort, puis cochez **Certificate : Click to create a user certificate**, choisissez la CA et **Save**.

**4. Exporter la configuration cliente**

1. **VPN ▸ OpenVPN ▸ Client Export**.
2. Remote Access Server : celui que vous venez de créer.
3. **Host Name Resolution** : l'IP du WAN, ou votre nom DDNS (section 4) si l'IP change.
4. Dans la liste des utilisateurs, téléchargez la configuration (*Inline Configurations*, fichier `.ovpn`, ou l'installateur Windows).

**5. Connecter le poste distant**

1. Installez le client **OpenVPN** sur la machine distante (une machine du réseau WAN, par exemple l'hôte).
2. Importez le fichier `.ovpn` et connectez-vous avec `vpnuser`.

### Vérification

1. Sur la machine distante, l'adresse du tunnel doit appartenir à `10.8.0.0/24`.
2. Testez l'accès aux ressources du LAN **à travers le tunnel** :

```bash
ping 192.168.1.103          # Ubuntu
ping 192.168.1.104          # Windows 10
```

3. Ouvrez `http://192.168.1.103` : la page d'Apache doit s'afficher sans redirection de port.
4. **Status ▸ OpenVPN** : le client doit apparaître comme connecté, avec son IP virtuelle et le volume de données.
5. Déconnectez le VPN : les mêmes tests doivent échouer (preuve que l'accès passait bien par le tunnel).

<img width="925" height="264" alt="Capture d&#39;écran 2026-11-09 105940" src="https://github.com/user-attachments/assets/ab234dfe-9212-4b7c-a368-2c8c4e20031b" />

---

## 9. IPsec (site-à-site)

### Objectif
Contrairement à OpenVPN (accès distant pour un utilisateur), IPsec en mode **site-à-site** relie **deux réseaux entiers** en permanence via un tunnel chiffré au niveau IP. C'est transparent pour les utilisateurs et les applications des deux côtés, et typiquement utilisé pour interconnecter deux sites d'une même entreprise.

La négociation se fait en deux phases :
- **Phase 1** : authentifie et chiffre la communication entre les deux pare-feux (tunnel de gestion).
- **Phase 2** : définit les sous-réseaux autorisés à communiquer à travers le tunnel (le « domaine de chiffrement »).

### Topologie

| | Site A (ce lab) | Site B |
|---|---|---|
| Équipement | pfSense du lab | **Second pfSense** (VM) `[à adapter à votre montage]` |
| Réseau LAN | `192.168.1.0/24` | `192.168.2.0/24` |
| WAN | `<IP_WAN_A>` | `<IP_WAN_B>` |

> Les deux sous-réseaux LAN doivent être **différents**, et les deux WAN doivent pouvoir communiquer entre eux (même réseau pont, par exemple).

### Procédure
À réaliser **sur les deux pfSense**, en inversant les adresses.

**1. Phase 1** : **VPN ▸ IPsec ▸ Tunnels ▸ Add P1**
- Key Exchange version : **IKEv2**
- Internet Protocol : **IPv4**
- Interface : **WAN**
- Remote Gateway : IP du WAN de l'**autre** site
- Description : *Tunnel Site A ↔ Site B*
- Authentication Method : **Mutual PSK**
- My identifier : *My IP address* ; Peer identifier : *Peer IP address*
- Pre-Shared Key : une clé longue et aléatoire (identique des deux côtés)
- Encryption Algorithm : **AES 256** ; Hash : **SHA256** ; DH Group : **14 (2048 bits)**
- Lifetime : valeur par défaut
- **Save**

**2. Phase 2** : dans le tunnel créé, **Show Phase 2 Entries ▸ Add P2**
- Mode : **Tunnel IPv4**
- Local Network : **LAN subnet** (ou `192.168.1.0/24`)
- Remote Network : le LAN de l'**autre** site (`192.168.2.0/24`)
- Protocol : **ESP**
- Encryption : **AES 256** ; Hash : **SHA256** ; PFS key group : **14**
- **Save**, puis **Apply Changes**.

**3. Règles de pare-feu**

**Firewall ▸ Rules ▸ IPsec ▸ Add** : règle **Pass**, protocole *Any*, source *LAN net* de l'autre site vers *LAN net* local. Sur le WAN, le trafic IKE (UDP 500/4500) et ESP doit être autorisé vers l'autre pare-feu ; rappelez-vous aussi de laisser décoché **Block private networks** sur le WAN en lab.

### Vérification

1. **Status ▸ IPsec** : le tunnel doit être **Established** (utilisez *Connect VPN* si besoin).
2. Depuis une machine du LAN du Site A, testez une machine du LAN du Site B :

```bash
ping 192.168.2.10
```

3. **Status ▸ IPsec ▸ SADs** : les associations de sécurité (SA) actives apparaissent avec les compteurs d'octets.
4. **Diagnostics ▸ Packet Capture** sur le WAN (protocole ESP) : le trafic doit apparaître **chiffré**, sans contenu lisible.

<img width="916" height="323" alt="Capture d&#39;écran 2026-11-09 104951" src="https://github.com/user-attachments/assets/426ca48c-04c3-4e0a-9f45-105cf5cbb5a4" />

---

## 10. pfBlockerNG

### Objectif
Filtrage réseau **sans configuration côté client** (contrairement à Squid) : pfBlockerNG bloque au niveau du pare-feu et du DNS, à partir de :
- **listes de réputation IP** (sources connues de malware, botnets) ;
- **listes DNS** (domaines publicitaires, de tracking, malveillants), le mécanisme appelé **DNSBL** ;
- **GeoIP** : blocage par pays entier, utile pour couper le trafic en provenance de régions sans activité légitime pour l'entreprise.

### Procédure

**1. Installer le paquet**

**System ▸ Package Manager ▸ Available Packages** : installez **`pfBlockerNG-devel`**.

**2. Compte MaxMind (pour GeoIP)**

pfBlockerNG exige un compte **MaxMind GeoLite2 gratuit** et une **clé de licence** pour télécharger les bases GeoIP. Créez le compte sur le site de MaxMind, générez une clé de licence, puis saisissez-la dans **Firewall ▸ pfBlockerNG ▸ IP** (champ *MaxMind License Key*).

**3. Configuration initiale**

1. **Firewall ▸ pfBlockerNG ▸ General** (ou l'assistant à la première ouverture) :
   - Enable pfBlockerNG : coché
   - Inbound Firewall Rules : **WAN**
   - Outbound Firewall Rules : **LAN**
2. **Save**.

**4. Activer DNSBL**

1. **Firewall ▸ pfBlockerNG ▸ DNSBL** : cochez **Enable DNSBL**.
2. DNSBL Virtual IP : conservez l'adresse proposée par défaut (`10.10.10.1`) ; Listening Interface : **LAN**.
3. Dans **DNSBL Feeds**, ajoutez un groupe de flux (exemple : *Pub_Tracking*) avec une liste de domaines publicitaires et de suivi (par exemple une liste « hosts » publique et maintenue), format *Auto*, action **Unbound**.
4. **Save**.

**5. Ajouter une liste de réputation IP**

**Firewall ▸ pfBlockerNG ▸ Feeds** propose des flux préconfigurés (par exemple des listes de réputation de type DROP ou Emerging Threats). Activez-en un, avec l'action **Deny Both**, puis **Save**.

**6. Blocage par pays (GeoIP)**

**Firewall ▸ pfBlockerNG ▸ IP**, onglets par continent : sélectionnez les pays à bloquer, action **Deny Both**, puis **Save**.
> Ne bloquez jamais votre propre pays ni les pays hébergeant les services dont vous dépendez (CDN, mises à jour).

**7. Appliquer**

**Firewall ▸ pfBlockerNG ▸ Update** : lancez *Run* avec l'option **Reload All** et attendez la fin du téléchargement.

### Vérification

1. Depuis Windows 10 (dont le DNS est pfSense), résolvez un domaine publicitaire présent dans la liste :

```

   La réponse doit être l'IP du DNSBL (`10.10.10.1`) au lieu de l'adresse réelle : le domaine est neutralisé.
2. Ouvrez le même domaine dans le navigateur : il ne doit pas se charger (désactivez le DNS sécurisé du navigateur, qui contourne le DNS de pfSense).
3. **Firewall ▸ pfBlockerNG ▸ Alerts** : les blocages DNSBL et IP/GeoIP apparaissent avec leur source, leur liste et leur horodatage.
4. Test GeoIP : accédez à un site hébergé dans un pays bloqué avant l'application des règles, puis après ; l'accès doit échouer et une alerte *GeoIP* doit apparaître.

<!-- ⚠️ À vérifier : la capture ci-dessous porte le même nom que celle de la section 7 (Squid). Remplacez-la par une capture de l'onglet Alerts ou DNSBL de pfBlockerNG si c'est la même image. -->
<img width="1078" height="423" alt="Capture d&#39;écran 2026-11-07 200227" src="https://github.com/user-attachments/assets/80af39f1-eb4a-49a2-92ca-772027e3dc0f" />

---

## 11. ntopng

### Objectif
Outil de visibilité réseau **en temps réel**. Contrairement à Wazuh (événements de sécurité) ou Zabbix (disponibilité et ressources), ntopng se concentre sur l'**analyse du trafic lui-même** : qui communique avec qui, quels protocoles, quels volumes, quels sont les principaux consommateurs de bande passante (« *top talkers* »). Un allié précieux pour diagnostiquer un ralentissement réseau ou repérer un comportement anormal.

### Procédure

1. **System ▸ Package Manager ▸ Available Packages** : installez `ntopng`.
2. **Diagnostics ▸ ntopng Settings** :
   - Enable ntopng : coché
   - Interface : **LAN**
   - Mot de passe administrateur : choisissez-en un
   - **Save**
3. Patientez quelques instants, puis ouvrez **Diagnostics ▸ ntopng** (lien *Access ntopng*). L'interface web s'ouvre sur le port `3000`.
4. Connectez-vous avec l'utilisateur `admin`.

> **Dépannage connu.** Sur certaines versions, le formulaire de mot de passe de pfSense n'est pas pris en compte. Solution : désinstaller le paquet **sans** conserver ses réglages, le réinstaller, se connecter avec `admin` / `admin` lors de la première connexion, puis **changer le mot de passe directement dans ntopng**.

### Génération de trafic de test

Depuis Windows 10 et Ubuntu, produisez du trafic varié :

```bash
# Ubuntu
curl -s https://example.com > /dev/null
sudo apt update
iperf3 -c <serveur-iperf> -t 20
```

et naviguez sur différents sites depuis Windows 10.

### Vérification

Dans ntopng, vérifiez les vues suivantes :

1. **Dashboard** : volume total, répartition par protocole.
2. **Hosts** : les hôtes `192.168.1.103` et `192.168.1.104` apparaissent avec leur trafic.
3. **Flows** : les connexions actives (source, destination, application détectée).
4. **Top talkers** : la machine qui consomme le plus de bande passante est identifiée.
5. Lancez un gros téléchargement depuis Ubuntu et constatez que l'hôte monte en tête des consommateurs.

> ntopng consomme de la mémoire et du processeur : sur une VM modeste, limitez l'analyse à l'interface LAN.

<img width="1365" height="664" alt="Capture d&#39;écran 2026-11-07 195839" src="https://github.com/user-attachments/assets/1c1b4a0e-4b07-4bcc-92a9-2a416ab66441" />

---

## 12. Test de charge (stress test)

### Objectif
Valider le comportement de l'ensemble **sous contrainte** : générer un trafic volumineux contre les services exposés, pour observer la tenue de la table d'états de pfSense, la réaction des règles de pare-feu et des limiters sous charge, ainsi que la consommation CPU et mémoire du pare-feu lui-même. C'est une vérification que la configuration tient la route au-delà d'un test ponctuel isolé.

> ⚠️ **À ne faire que sur votre propre lab isolé.** Générer du trafic massif vers un réseau ou un service qui ne vous appartient pas est illégal.

### Montage

- **Cible** : le service web d'Ubuntu, exposé par le port forward de la section 3 (`192.168.100.123:8080`).
- **Générateur de trafic** : une machine **côté WAN** (par exemple une VM Kali en pont), puisque le trafic doit **traverser** pfSense. Deux machines du même LAN ne passeraient pas par le pare-feu.

### Procédure

**1. Relever l'état de référence**

Sur pfSense, notez : charge CPU, mémoire et nombre d'états (tableau de bord), puis **Status ▸ Monitoring** pour les graphiques (*System ▸ Processor / Memory / States*).

**2. Générer la charge** (depuis la machine côté WAN)

Charge applicative (nombreuses requêtes web simultanées) :

```bash
sudo apt install -y apache2-utils
ab -n 20000 -c 200 http://<IP_WAN_PFSENSE>:8080/
```

Rafale de paquets SYN à débit contrôlé :

```bash
sudo apt install -y hping3
sudo hping3 -S -p 8080 -c 50000 -i u200 <IP_WAN_PFSENSE>
```

*(`-c 50000` limite le nombre de paquets et `-i u200` espace les envois de 200 microsecondes : on garde un test borné et reproductible.)*

**3. Observer pendant le test**

- **Tableau de bord** et **Status ▸ Monitoring** : évolution du CPU, de la mémoire et de la table d'états.
- **Diagnostics ▸ States ▸ States Summary** : sources qui génèrent le plus d'états.
- **Status ▸ System Logs ▸ Firewall** : paquets bloqués ou autorisés.
- Depuis une autre machine côté WAN, vérifiez si le service reste joignable :

```bash
curl -I http://<IP_WAN_PFSENSE>:8080
```

**4. Comparer avec et sans limiter** (section 6)

Refaites le test avec le limiter actif sur Ubuntu et comparez le débit servi, les états consommés et la charge de pfSense.

**5. Pour aller plus loin : limiter les abus au niveau de la règle**

Dans **Firewall ▸ Rules ▸ WAN** (règle du port forward) ▸ **Advanced Options**, des limites par source existent (nombre maximal d'états ou de nouvelles connexions par seconde). Elles permettent d'ajouter automatiquement une source trop agressive dans une table de blocage. À tester avec le même scénario pour constater la différence.

### Vérification

- Le pare-feu **ne plante pas** et reste administrable pendant le test.
- La table d'états monte puis redescend à la fin du test.
- Le service reste accessible (ou se dégrade de façon contrôlée).
- Les mesures (CPU, mémoire, états) avant/pendant/après sont consignées et comparées.

<img width="1298" height="552" alt="Capture d&#39;écran 2026-11-07 195550" src="https://github.com/user-attachments/assets/ccf6153b-ce10-44be-be92-600ce8a7fb5e" />

---

## Compétences démontrées

Pare-feu et routage · NAT/PAT · haute disponibilité réseau (Multi-WAN) · QoS · VPN (accès distant et site-à-site) · filtrage web et proxy · threat intelligence réseau (GeoIP, listes de réputation) · supervision du trafic · tests de résilience.

---

## Conclusion

### Ce que j'ai appris

Ce lab m'a fait passer d'un pare-feu « qui bloque des attaques » à un pare-feu utilisé comme une **plateforme réseau complète**. Ce que j'en retiens :

1. **La sauvegarde avant tout.** Exporter la configuration avant chaque section a transformé les erreurs en simples retours en arrière. C'est un réflexe que j'applique désormais à tout équipement.
2. **Nommer pour maintenir.** Les alias rendent les règles lisibles et évolutives ; les IP virtuelles permettent d'exposer plusieurs services sur une même interface sans collision.
3. **Les contraintes du terrain.** Le double NAT m'a appris pourquoi un client DDNS doit passer par un service de détection d'IP externe. La simulation du second WAN m'a montré comment pfSense surveille et bascule entre passerelles avec `dpinger`.
4. **Plafond contre priorité.** J'ai compris la différence entre un *limiter* (plafond strict) et une *queue* (ordre de passage), et l'importance du sens *In/Out* selon l'interface où la règle s'applique.
5. **Connaître les limites d'un outil.** Squid et SquidGuard filtrent par domaine mais ne reconnaissent pas une application précise ; ils sont de plus dépréciés par l'éditeur. Savoir dire « cet outil ne fait pas ça » et proposer une alternative (filtrage DNS avec pfBlockerNG, pare-feu nouvelle génération) est aussi une compétence.
6. **Deux VPN, deux usages.** OpenVPN pour un utilisateur nomade, IPsec pour relier deux sites : même objectif de confidentialité, architectures et cas d'usage différents.
7. **Voir pour comprendre.** ntopng donne la vision du trafic que ni un SIEM (Wazuh) ni une supervision de disponibilité (Zabbix) ne donnent seuls ; les trois sont complémentaires.
8. **Tester pour croire.** Chaque fonctionnalité a été validée par un test mesurable (avant/après), et le test de charge a mis la configuration à l'épreuve au lieu de la supposer solide.

### Difficultés rencontrées

- **Ressources matérielles limitées** : le lab a été réduit à deux machines clientes (Windows Server et Active Directory sont traités dans un projet séparé).
- **Un seul accès Internet réel** : le second WAN a dû être simulé avec un réseau NAT VirtualBox.
- **Double NAT** : adaptation du client DDNS.
- **Outils dépréciés** (Squid, SquidGuard) : documentés comme tels, avec leurs limites.

### Pistes pour la suite

- Haute disponibilité avec **deux pfSense en CARP** (les IP virtuelles de la section 2 en sont la première brique).
- Détection d'intrusion avec Suricata ou Snort, couplée à **Wazuh** pour la corrélation des alertes.
- Intégration d'**Active Directory** et d'un serveur Windows pour l'authentification des utilisateurs (projet séparé).
- Supervision de la disponibilité des passerelles et des tunnels avec **Zabbix**.

---

## Ressources

- [Documentation officielle pfSense (Netgate)](https://docs.netgate.com/pfsense/en/latest/) : rubriques *Multi-WAN*, *VPN ▸ OpenVPN*, *VPN ▸ IPsec*, *Firewall ▸ NAT*
- [Limiters (Netgate)](https://docs.netgate.com/pfsense/en/latest/trafficshaper/limiters.html)
- [IP Address Check Services pour le DDNS (Netgate)](https://docs.netgate.com/pfsense/en/latest/services/dyndns/check-services.html)
- [Configuration de SquidGuard et avertissement de dépréciation (Netgate)](https://docs.netgate.com/pfsense/en/latest/packages/cache-proxy/squidguard.html)

## Voir aussi

- [`ArchitectureLAB.md`](https://github.com/rdannon/HomeLAB) : vue d'ensemble de l'architecture complète du lab
- [`LabCybersecurity`](https://github.com/rdannon/LabCybersecurity) : volet attaque/défense (DoS, brute-force SSH, injection SQL/WAF, SIEM Wazuh)

---

## Me contacter

**Rock Dannon** · Master 2 Réseaux Informatiques et Télécommunications
💼 [LinkedIn](https://www.linkedin.com/in/rock-dannon) · 💻 [GitHub](https://github.com/rdannon)

*Les adresses IP, noms de domaine et identifiants présentés sont des valeurs de démonstration propres à ce lab : ne les réutilisez jamais telles quelles en production.*
