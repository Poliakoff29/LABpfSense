# 🔧 pfSense — Maîtrise des fonctionnalités avancées

> VPN, haute disponibilité réseau, filtrage web, threat intelligence et supervision du trafic — un pare-feu open-source poussé dans ses retranchements pour couvrir ce qu'une PME attend d'une solution réseau d'entreprise, sans le budget d'un NGFW propriétaire.

Ce projet fait suite à mon [lab d'attaque/défense pfSense](./lab-cybersecurite-guide.md) : une fois le pare-feu éprouvé contre des attaques réelles, j'ai voulu explorer l'ensemble de ses fonctionnalités de niveau entreprise — celles qu'on attend concrètement d'un poste réseau ou SOC N2.

## Architecture

| Élément | Détail |
|---|---|
| pfSense | WAN (pont, réseau physique) · LAN 192.168.1.0/24 |
| Ubuntu Server | Hôte applicatif de test — 192.168.1.103 |
| Windows 10 | Poste client de test — 192.168.1.104 |
| Contrainte matérielle | Ressources limitées → lab volontairement réduit à 2 machines clientes plutôt que l'ensemble du parc (Windows Server/AD traité dans un projet séparé) |

> **Note de méthode** — Chaque fonctionnalité a été configurée, testée avec un scénario concret (pas juste activée dans l'interface), puis validée par une vérification mesurable : un test qui échoue avant la configuration et réussit après, ou l'inverse selon le cas.

## Sommaire

1. [Sauvegarde & mise à jour](#1-sauvegarde--mise-à-jour)
2. [Alias & IP virtuelles](#2-alias--ip-virtuelles)
3. [NAT & Port Forwarding](#3-nat--port-forwarding)
4. [DDNS](#4-ddns)
5. [Multi-WAN (failover + load balancing)](#5-multi-wan-failover--load-balancing)
6. [Traffic Shaper (QoS)](#6-traffic-shaper-qos)
7. [Squid & SquidGuard](#7-squid--squidguard)
8. [OpenVPN](#8-openvpn)
9. [IPsec (site-à-site)](#9-ipsec-site-à-site)
10. [pfBlockerNG](#10-pfblockerng)
11. [ntopng](#11-ntopng)
12. [Test de charge (Stress Test)](#12-test-de-charge-stress-test)

---

## 1. Sauvegarde & mise à jour

Avant de toucher à quoi que ce soit : exporter la configuration (*Diagnostics ▸ Backup & Restore*) et vérifier l'état des mises à jour (*System ▸ Update*). Un filet de sécurité indispensable avant d'enchaîner des changements sur un pare-feu en service — une config qui casse se restaure en un clic plutôt qu'en reconfigurant tout à la main.

![Écran de sauvegarde de la configuration pfSense](pfsense-backup-config-screen.png)

## 2. Alias & IP virtuelles

**Alias** : un nom réutilisable représentant un groupe d'IP, de réseaux ou de ports (ex. `LAN_Servers`, `Web_Ports`). Centralise la maintenance — modifier l'alias une fois met à jour toutes les règles qui l'utilisent, au lieu de les retrouver une par une.

**IP virtuelle (VIP)** : permet à une interface physique unique de porter plusieurs adresses IP. Utile pour exposer plusieurs services sur des IP publiques distinctes malgré un seul pare-feu, ou pour la haute disponibilité (CARP) entre deux pfSense.

Mis en pratique : un alias `LAN_Servers` regroupant les machines internes, un alias de ports `Web_Ports`, et une VIP de type IP Alias sur le WAN — réutilisée directement à l'étape suivante (NAT).

![Liste des alias et de l'IP virtuelle configurés](pfsense-alias-list.png)

## 3. NAT & Port Forwarding

Le NAT sortant masque les machines du LAN derrière l'IP WAN. Le **Port Forwarding** fait l'inverse : il expose un service interne à l'extérieur en redirigeant un port entrant vers une IP du LAN.

Deux redirections mises en place pour démontrer l'intérêt des VIP : un service web (port 8080 → Ubuntu:80) sur l'IP WAN principale, et un accès RDP (port 3389) sur la VIP dédiée vers Windows 10 — deux services distincts, chacun sur sa propre IP publique, sans collision de ports.

![Règle de redirection de port testée avec succès](nat-port-forward-web-test.png)

## 4. DDNS

Les connexions grand public ont une IP publique qui change régulièrement. Le **Dynamic DNS** maintient un nom de domaine fixe (ex. `rock-lab.duckdns.org`) toujours à jour avec l'IP actuelle, via un client intégré à pfSense qui pousse les changements vers un fournisseur DDNS (ici DuckDNS).

> Point technique rencontré : avec une connexion en double NAT, pfSense ne voit pas directement sa vraie IP publique sur son interface WAN — le client DDNS a été configuré pour utiliser un service de détection d'IP externe plutôt que l'IP brute de l'interface.

![Statut du client DDNS sur pfSense, à jour](pfsense-ddns-status.png)

## 5. Multi-WAN (failover + load balancing)

Brancher deux accès Internet sur pfSense permet soit de **basculer automatiquement** si l'un tombe (failover), soit de **répartir la charge** entre les deux en continu (load balancing) — ou les deux à la fois via un seul Gateway Group avec les deux passerelles au même niveau (Tier).

> Faute d'un deuxième accès Internet réel, le second WAN a été simulé avec un réseau NAT VirtualBox indépendant du pont utilisé pour le WAN principal — suffisant pour que pfSense gère deux passerelles réellement distinctes et teste la bascule en conditions quasi réelles.

La détection de panne et le retrait/réintégration automatique d'une passerelle sont gérés par `dpinger`, qui surveille chaque gateway en continu (ping régulier) et bascule en quelques secondes en cas de coupure.

![Basculement automatique détecté sur Status ▸ Gateways](pfsense-multiwan-failover.png)

## 6. Traffic Shaper (QoS)

Deux mécanismes distincts : les **Limiters** plafonnent un débit strict (indépendamment de la charge du réseau), les **Queues** gèrent une priorité relative quand le lien sature (sans rien brider tant qu'il y a de la bande disponible). Les deux répondent à des besoins différents — l'un garantit un plafond, l'autre garantit un ordre de passage.

Mis en pratique : un Limiter à 1 Mbit/s appliqué à Ubuntu Server, validé par un test de débit avant/après — passage de plusieurs dizaines de Mbit/s à un plafond net autour de 1 Mbit/s.

![Test de débit avant et après l'application du limiteur](speedtest-after-limiter.png)

## 7. Squid & SquidGuard

**Squid** est un proxy web : le trafic HTTP du LAN transite par pfSense avant d'aller sur Internet, ce qui permet la journalisation et sert de point d'ancrage au filtrage. **SquidGuard** s'y greffe pour bloquer l'accès à des domaines ou catégories entières (listes noires).

Configuré en **mode proxy explicite** (le client configure l'adresse du proxy) plutôt qu'en mode transparent, pour filtrer le HTTPS proprement sans avoir à déployer de certificat d'interception SSL sur chaque poste.

> Limite technique importante documentée : ni Squid ni SquidGuard ne peuvent identifier un **service applicatif précis** (ex. YouTube) de façon fiable — ce niveau de classification ("App-ID") est une fonctionnalité propre aux pare-feux nouvelle génération (Palo Alto, Fortinet...), qui inspectent les métadonnées TLS plutôt que le simple port/IP.

![Page de blocage SquidGuard sur un domaine filtré](squidguard-blocked-page.png)

## 8. OpenVPN

VPN d'accès distant (« road warrior ») : un poste externe se connecte à pfSense via un tunnel chiffré et rejoint virtuellement le réseau interne, comme s'il y était physiquement branché — utile pour le télétravail ou l'administration à distance sécurisée.

Configuré via l'assistant OpenVPN de pfSense (génération de l'autorité de certification, du certificat serveur, définition du réseau de tunnel et du réseau local accessible), avec export de la configuration client via le package OpenVPN Client Export, import sur le poste distant et validation de l'accès aux ressources du LAN à travers le tunnel.

![Client OpenVPN connecté avec une IP de tunnel attribuée](openvpn-client-connected.png)

## 9. IPsec (site-à-site)

Contrairement à OpenVPN (accès distant pour un utilisateur), IPsec en mode site-à-site relie **deux réseaux entiers** en permanence via un tunnel chiffré au niveau IP — transparent pour les utilisateurs et les applications des deux côtés, typiquement utilisé pour interconnecter deux sites d'une même entreprise.

Négociation en deux phases : la Phase 1 authentifie et chiffre la communication entre les deux pare-feux, la Phase 2 définit les sous-réseaux autorisés à communiquer à travers le tunnel (le « domaine de chiffrement »).

![Tunnel IPsec établi, statut actif](ipsec-tunnel-status.png)

## 10. pfBlockerNG

Filtrage réseau **sans configuration côté client** (contrairement à Squid) : pfBlockerNG bloque au niveau du pare-feu et du DNS, à partir de listes de réputation IP (sources connues de malware, botnets) et de listes DNS (domaines publicitaires, de tracking, malveillants), avec en complément la possibilité de bloquer par pays entier (GeoIP) — utile pour couper tout le trafic en provenance de régions sans activité légitime pour l'entreprise.

![Tableau de bord pfBlockerNG avec les listes actives](pfblockerng-dashboard.png)

## 11. ntopng

Outil de visibilité réseau en temps réel : contrairement à Wazuh (événements de sécurité) ou Zabbix (disponibilité/ressources), ntopng se concentre sur l'**analyse du trafic lui-même** — qui communique avec qui, quels protocoles, quels volumes, les principaux consommateurs de bande passante (« top talkers »). Un allié précieux pour diagnostiquer un ralentissement réseau ou repérer un comportement anormal.

![Tableau de bord ntopng montrant le trafic en temps réel](ntopng-traffic-dashboard.png)

## 12. Test de charge (Stress Test)

Validation du comportement de l'ensemble sous contrainte : génération d'un trafic volumineux (outils de type hping3/iperf3) contre les services exposés, pour observer la tenue de la table d'états de pfSense, la réaction des règles de pare-feu et des Limiters sous charge, ainsi que la consommation CPU/mémoire du pare-feu lui-même — une vérification que la configuration tient la route au-delà d'un test ponctuel isolé.

![Résultats du test de charge et comportement de pfSense sous contrainte](stress-test-results.png)

---

## Compétences démontrées

Pare-feu & routage · NAT/PAT · haute disponibilité réseau (Multi-WAN) · QoS · VPN (accès distant et site-à-site) · filtrage web et proxy · threat intelligence réseau (GeoIP, listes de réputation) · supervision du trafic · tests de résilience.

## Voir aussi

- [`laboratoire-reseau-entreprise.md`](./laboratoire-reseau-entreprise.md) — vue d'ensemble de l'architecture complète du lab
- [`lab-cybersecurite-guide.md`](./lab-cybersecurite-guide.md) — volet attaque/défense (DoS, brute-force SSH, injection SQL/WAF, SIEM Wazuh)

---

*Les adresses IP et identifiants de démonstration sont propres à cet environnement de lab et à adapter à ton propre contexte.*
