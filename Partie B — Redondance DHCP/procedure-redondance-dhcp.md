# Partie B — Redondance DHCP

[⬅ Retour au sommaire du projet](../README.md)

## Objectif

Disposer d'un second serveur DHCP en cas de panne du premier, en configurant deux serveurs actifs sur la **même plage d'adresses**, sans mécanisme de synchronisation entre eux. Cette manipulation permet d'observer et de comprendre les limites d'une telle approche, avant la mise en place d'une solution de redondance maîtrisée en partie C.

---

## Sommaire

1. [Architecture à réaliser](#1-architecture-à-réaliser)
2. [Préparation des VM sous VirtualBox](#2-préparation-des-vm-sous-virtualbox)
3. [Configuration du réseau NAT VirtualBox](#3-configuration-du-réseau-nat-virtualbox)
4. [Adressage IP fixe des serveurs](#4-adressage-ip-fixe-des-serveurs)
5. [Installation et configuration du serveur 1](#5-installation-et-configuration-du-serveur-1)
6. [Réplication de la configuration sur le serveur 2](#6-réplication-de-la-configuration-sur-le-serveur-2)
7. [Tests côté clients](#7-tests-côté-clients)
8. [Vérification des baux sur les deux serveurs](#8-vérification-des-baux-sur-les-deux-serveurs)
9. [Analyse : risque de conflit d'adresses IP](#9-analyse--risque-de-conflit-dadresses-ip)
10. [Conclusion et limites de l'approche](#10-conclusion-et-limites-de-lapproche)

---

## 1. Architecture à réaliser

<img src="./images/1-architecture-cible.png" alt="architecture-cible" width="45%">

L'architecture repose sur un réseau NAT VirtualBox (`10.0.2.0/24`) regroupant quatre machines virtuelles :

| Machine | Rôle | Adresse IP |
|---|---|---|
| Serveur 1 | Serveur DHCP (Linux) | `10.0.2.10` (fixe) |
| Serveur 2 | Serveur DHCP (Linux) | `10.0.2.11` (fixe) |
| Client 1 | Poste Windows | Attribuée par DHCP |
| Client 2 | Poste Linux | Attribuée par DHCP |

La passerelle vers Internet (`10.0.2.1`) est assurée directement par VirtualBox.

> [!NOTE]
> Contrairement à la partie A, les serveurs et les clients se trouvent ici sur le **même réseau**. Le routage vers Internet étant assuré directement par VirtualBox, les serveurs DHCP n'ont ici pour seul rôle que d'attribuer des adresses IP : l'activation de `ip_forward` et la configuration d'`iptables` ne sont donc **pas nécessaires**.

---

## 2. Préparation des VM sous VirtualBox

- Préparation d'une VM Ubuntu (Serveur 1), mise à jour.
- Préparation d'une seconde VM Ubuntu (Serveur 2), mise à jour.
- Préparation d'une VM Windows 11 (Client 1) et d'une VM Linux (Client 2), mises à jour.

---

## 3. Configuration du réseau NAT VirtualBox

Dans **Fichier → Outils → Réseau**, onglet **Réseau NAT**, on crée une nouvelle carte virtuelle et on désactive son serveur DHCP intégré, afin que ce soit nos propres serveurs qui distribuent les adresses IP.

```
Nom          : DHCP
IPv4 Prefix  : 10.0.2.0/24
Enable DHCP  : décoché
```

<img src="./images/2-virtualbox-reseau-nat.png" alt="Configuration du réseau NAT VirtualBox" width="25%">

> [!IMPORTANT]
> Le DHCP intégré de VirtualBox **doit impérativement être désactivé** sur ce réseau NAT, sous peine d'entrer en conflit avec les serveurs DHCP mis en place dans ce TP.

Les quatre VM sont ensuite rattachées à cette carte réseau NAT (`10.0.2.0/24`, passerelle `10.0.2.1`) :

<img src="./images/3-virtualbox-adaptateur-reseau.png" alt="Configuration du réseau NAT VirtualBox bis" width="45%">

---

## 4. Adressage IP fixe des serveurs

> [!IMPORTANT]
> Un serveur DHCP ne peut pas dépendre du DHCP pour obtenir sa propre adresse IP. Les deux serveurs doivent donc être configurés avec une **adresse IP fixe**, avant même le démarrage du service DHCP.

| VM | IP | Masque | Passerelle |
|---|---|---|---|
| Serveur 1 | `10.0.2.10` | `/24` | `10.0.2.1` |
| Serveur 2 | `10.0.2.11` | `/24` | `10.0.2.1` |

---

## 5. Installation et configuration du serveur 1

### 5.1 Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

<img src="./images/4-maj-serveur1.png" alt="Mise à jour du système" width="45%">

### 5.2 Installation d'isc-dhcp-server

```bash
sudo apt install isc-dhcp-server
```

<img src="./images/5-install-isc-dhcp-server.png" alt="Installation d'isc-dhcp-server" width="30%">

### 5.3 Identification et déclaration de l'interface réseau

Avant de configurer le service, il est nécessaire d'identifier précisément l'interface réseau du serveur sur laquelle le service DHCP devra écouter les requêtes des clients.

**Identification de l'interface :**

```bash
ip a
```

> <img src="./images/6-ip-a-serveur1.png" alt="Identification de l'interface réseau" width="50%">

Cette commande liste l'ensemble des interfaces réseau de la machine. Sur cette VM, l'interface active correspondant à la carte réseau utilisée est `enp0s3` (nomenclature standard sous Debian/Ubuntu pour la première interface Ethernet détectée).

> [!NOTE]
> Le nommage des interfaces réseau (`enp0s3`, `eth0`, etc.) peut varier selon la distribution et la configuration matérielle/virtuelle de la machine. Il est donc indispensable de vérifier ce nom via `ip a` avant de configurer isc-dhcp-server, plutôt que de le déduire par défaut.

**Déclaration de l'interface dans la configuration du service :**

Le fichier `/etc/default/isc-dhcp-server` définit sur quelle(s) interface(s) le service `isc-dhcp-server` doit écouter les requêtes DHCP. Sans cette déclaration, le service ne sait pas sur quel réseau il doit distribuer des adresses IP et refuse de démarrer correctement.

```bash
sudo nano /etc/default/isc-dhcp-server
```

On renseigne l'interface identifiée précédemment :

```bash
INTERFACESv4="enp0s3"
```

<img src="./images/7-interfacesv4.png" alt="Déclaration de l'interface dans isc-dhcp-server" width="50%">

> [!IMPORTANT]
> Une interface mal renseignée (ou laissée vide) est une cause fréquente d'échec de démarrage du service `isc-dhcp-server`, ou d'un service démarré mais qui ne répond à aucune requête DHCP. C'est l'une des premières choses à vérifier en cas de dysfonctionnement.

### 5.4 Configuration de dhcpd.conf

Dans le fichier de configuration DHCP `/etc/dhcp/dhcpd.conf`, accessible avec la commande `sudo nano /etc/dhcp/dhcpd.conf`, on définit les paramètres nécessaires au fonctionnement du serveur DHCP : la plage d’adresses IP à distribuer, la passerelle par défaut, le serveur DNS ainsi que la durée du bail DHCP.

Cette configuration sera également appliquée au serveur DHCP 2 pour notre test.

```conf
subnet 10.0.2.0 netmask 255.255.255.0 {
  range 10.0.2.100 10.0.2.150;
  option routers 10.0.2.1;
  option domain-name-servers 8.8.8.8;
  default-lease-time 86400;
  max-lease-time 604800;
}
```

<img src="./images/8-dhcpd-conf-serveur1.png" alt="Configuration de dhcpd.conf" width="40%">

**Récapitulatif de la configuration :**

| Paramètre | Valeur |
|---|---|
| Adresse du sous-réseau | `10.0.2.0` |
| Plage d'adresses distribuées | `10.0.2.100` – `10.0.2.150` |
| Adresse de la passerelle | `10.0.2.1` |
| Adresse du serveur DNS | `8.8.8.8` |
| Durée du bail par défaut | 1 jour (86 400 s) |
| Durée maximale du bail | 7 jours (604 800 s) |

### 5.5 Passage en IP fixe

> [!CAUTION]
> Le service DHCP refuse de démarrer s'il ne dispose pas d'une adresse IP fixe et stable pour écouter le réseau. Il est donc nécessaire de conserver temporairement la carte réseau en mode **Automatique (DHCP)** le temps d'installer le paquet `isc-dhcp-server` (pour conserver l'accès à Internet), puis de basculer en mode **Manuel** avant d'activer le service.

Paramétrage : **Paramètres → Réseau → connexion « Filaire » → roue crantée**, puis configuration de l'IPv4 :

```
Méthode     : Manuel
Adresse     : 10.0.2.10
Masque      : 255.255.255.0
Passerelle  : 10.0.2.1
```

<img src="./images/9-ipv4-manuel-serveur1.png" alt="Passage en IP fixe" width="45%">

### 5.6 Activation du service

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

<img src="./images/10-status-serveur1.png" alt="Activation du service" width="45%">

---

## 6. Réplication de la configuration sur le serveur 2

On reproduit à l'identique la procédure d'installation et de configuration sur le serveur 2. Les seules différences sont :

- **Adresse IP fixe du serveur :** `10.0.2.11` (au lieu de `10.0.2.10`).
- **Configuration DHCP :** strictement identique à celle du serveur 1 (même plage `10.0.2.100 - 10.0.2.150`, même passerelle, même DNS).

Les deux serveurs DHCP sont ensuite démarrés **simultanément**, afin d'observer leur comportement lorsqu'ils distribuent la même plage d'adresses sans aucun mécanisme de synchronisation.

<img src="./images/11-status-serveur2.png" alt=" Réplication de la configuration" width="45%">

---

## 7. Tests côté clients

### Client Windows

```powershell
ipconfig
```
Le résultat de la commande ipconfig montre que le client Windows a bien reçu l'adresse IP dynamique 10.0.2.102. Cette adresse se trouve bien dans la plage définie (10.0.2.100 à .150).

<img src="./images/12-client-windows-ipconfig.png" alt=" Client Windows" width="45%">


| Paramètre | Valeur obtenue |
|---|---|
| Adresse IPv4 | `10.0.2.102` |
| Masque de sous-réseau | `255.255.255.0` |
| Passerelle par défaut | `10.0.2.1` |

### Client Linux

```bash
ip a
```

<img src="./images/13-client-linux-ip-a.png" alt=" Client Linux" width="45%">

Le client Linux obtient l'adresse `10.0.2.103/24`, avec la mention `dynamic`, confirmant une attribution automatique par un serveur DHCP.

---

## 8. Vérification des baux sur les deux serveurs

```bash
sudo cat /var/lib/dhcp/dhcpd.leases
```

### Sur le serveur 1

<img src="./images/14-leases-serveur1.png" alt="Sur le serveur 1" width="45%">

Le serveur 1 a attribué l'adresse `10.0.2.102` au client Windows.

### Sur le serveur 2

<img src="./images/15-leases-serveur2.png" alt="Sur le serveur 2" width="45%">

Le serveur 2 a attribué l'adresse `10.0.2.103` au client Linux, **mais** l'adresse `10.0.2.102` apparaît **également** dans son propre fichier de baux.

---

## 9. Analyse : risque de conflit d'adresses IP

> [!WARNING]
> Les deux serveurs DHCP utilisent la **même plage d'adresses sans aucune synchronisation**. Chacun peut donc attribuer indépendamment une adresse IP à un client, avec un risque réel qu'une même adresse soit distribuée à deux machines différentes.

**Que se passe-t-il lorsqu'un client émet une requête ?**
Les deux serveurs DHCP peuvent répondre simultanément par un `DHCPOFFER`. Le client sélectionne l'une des offres reçues et envoie un `DHCPREQUEST` au serveur correspondant ; l'autre offre est alors abandonnée.

**Qui répond en premier ?**
Le résultat est **imprévisible** : il dépend de la latence réseau et de la charge de chaque serveur au moment de la requête, et non d'une quelconque priorité configurée.

**Que se passerait-il si deux machines obtenaient la même adresse IP ?**
Un conflit d'adresse IP surviendrait : ARP se retrouverait avec deux réponses pour une même IP, provoquant des coupures réseau intermittentes. Windows affiche typiquement une alerte de type *« conflit d'adresse IP »* sur l'un des deux postes concernés.

---

## 10. Conclusion et limites de l'approche

Cette configuration permet d'observer concrètement le fonctionnement de deux serveurs DHCP actifs partageant la même plage d'adresses. Elle met en évidence un point essentiel : sans mécanisme de synchronisation, chaque serveur attribue des adresses de manière totalement indépendante, avec un risque avéré de conflit d'adresses IP en cas d'attribution simultanée à deux clients différents.

> [!CAUTION]
> Cette approche est adaptée à une **démonstration pédagogique** ou à un environnement de test, mais elle n'est **pas recommandée en production**.

Une solution de type **DHCP Failover** permettrait de synchroniser automatiquement les informations de baux entre les deux serveurs et de supprimer ce risque de conflit. C'est l'objet de la partie suivante.

---

[⬅ Partie précédente : Serveur DHCP sur Raspberry](../Partie%20A%20—%20Création%20d'un%20serveur%20DHCP%20sur%20Raspberry%20Pi/procedure-dhcp-raspberry.md) 

[Retour au sommaire du projet](../README.md) · 

[Partie suivante : Agent relais DHCP & Grappe DHCP ➡](../Partie%20C%20—%20Agent%20relais%20DHCP/procedure-agent-relais.md)
