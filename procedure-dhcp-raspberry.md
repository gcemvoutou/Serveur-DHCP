# Partie A — Création d'un serveur DHCP sur Raspberry Pi

[⬅ Retour au sommaire du projet](../README.md)

## Objectif

Mettre en place un serveur DHCP fonctionnel sur un Raspberry Pi, capable de distribuer automatiquement des paramètres IP à des clients Windows et Linux, tout en assurant le routage de ces clients vers Internet via NAT.

---

## Sommaire

1. [Rappels théoriques](#1-rappels-théoriques)
2. [Architecture réseau](#2-architecture-réseau)
3. [Mise à jour du système](#3-mise-à-jour-du-système)
4. [Installation du serveur DHCP](#4-installation-du-serveur-dhcp)
5. [Activation du routage (NAT)](#5-activation-du-routage-nat)
6. [Configuration du fichier dhcpd.conf](#6-configuration-du-fichier-dhcpdconf)
7. [Démarrage et vérification du service](#7-démarrage-et-vérification-du-service)
8. [Test côté client Windows](#8-test-côté-client-windows)
9. [Vérification des baux (fichier leases)](#9-vérification-des-baux-fichier-leases)
10. [Renouvellement du bail](#10-renouvellement-du-bail)
11. [Test côté client Linux](#11-test-côté-client-linux)
12. [Modification de la plage d'adresses](#12-modification-de-la-plage-dadresses)
13. [Réservation d'une adresse IP fixe](#13-réservation-dune-adresse-ip-fixe)
14. [Récapitulatif de la configuration](#14-récapitulatif-de-la-configuration)
15. [Schéma récapitulatif](#15-schéma-récapitulatif)

---

## 1. Rappels théoriques

Un serveur DHCP permet de distribuer aux clients d'un réseau les paramètres IP nécessaires à leur connexion : adresse IP, masque de sous-réseau, passerelle et serveur DNS, pour une durée déterminée appelée **bail**. Il est également possible de réserver une adresse IP fixe pour une adresse MAC donnée.

Sous Linux, le serveur DHCP par défaut est **isc-dhcp-server**, une solution open-source fournie par l'ISC (Internet Software Consortium).

### Fonctionnement (échange en 4 temps)

| Étape | Description |
|---|---|
| `DHCPDISCOVER` | Le client demande un bail, avec une adresse source `0.0.0.0` et une destination en broadcast `255.255.255.255`, accompagnée de son adresse MAC. |
| `DHCPOFFER` | Un serveur DHCP répond en proposant une offre d'adresse. |
| `DHCPREQUEST` | Le client sélectionne une offre et la réserve auprès du serveur correspondant. |
| `DHCPACK` | Le serveur confirme l'attribution de l'adresse. |

---

## 2. Architecture réseau

Le Raspberry Pi fait ici office à la fois de **serveur DHCP** et de **routeur** entre le réseau local (interface `eth0`) et Internet (interface `wlan0`, connectée en Wi-Fi automatique).

> `images/01-schema-reseau.png`

> [!IMPORTANT]
> Comme le Raspberry Pi assure le routage entre deux réseaux, l'activation du routage IP (`ip_forward`) et la configuration du NAT via `iptables` sont **obligatoires**. Ce ne sera plus le cas dans les parties suivantes, où le routage est assuré directement par l'infrastructure réseau.

---

## 3. Mise à jour du système

```bash
sudo apt update
sudo apt upgrade -y
```

> `images/02-maj-systeme.png`

---

## 4. Installation du serveur DHCP

```bash
sudo apt install isc-dhcp-server
```

On édite ensuite le fichier `/etc/default/isc-dhcp-server` afin d'y déclarer l'interface réseau sur laquelle le serveur doit écouter les requêtes DHCP, ici `eth0`.

---

## 5. Activation du routage (NAT)

### 5.1 Activation de l'IP forwarding

On active le routage IP entre l'interface Wi-Fi et l'interface filaire en ajoutant la ligne suivante dans `/etc/sysctl.conf` :

```bash
net.ipv4.ip_forward=1
```

Puis on recharge la configuration :

```bash
sudo sysctl -p /etc/sysctl.conf
```

> `images/03-sysctl-ip-forward.png`

### 5.2 Règles de NAT (iptables)

```bash
sudo apt install iptables
```

**Masquerading** — réécrit les adresses IP des paquets provenant du réseau local (`eth0`) afin qu'ils puissent sortir sur Internet via l'interface Wi-Fi (`wlan0`) :

```bash
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
```

**Autorisation du trafic de retour** — permet aux paquets des connexions déjà établies de revenir depuis Internet vers le réseau local :

```bash
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

**Autorisation des connexions sortantes** — permet aux machines du réseau local d'initier de nouvelles connexions vers Internet :

```bash
sudo iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
```

> [!TIP]
> Ces trois règles forment un jeu minimal permettant un partage de connexion complet : NAT en sortie, autorisation du trafic établi en retour, et autorisation des nouvelles connexions sortantes.

---

## 6. Configuration du fichier dhcpd.conf

Deux fichiers sont utilisés par le serveur DHCP :

| Fichier | Rôle |
|---|---|
| `/etc/dhcp/dhcpd.conf` | Configuration du serveur (plage d'adresses, options réseau) |
| `/var/lib/dhcp/dhcp.leases` | Journal des baux attribués aux clients |

Configuration retenue pour `dhcpd.conf` :

```conf
# Notre réseau
subnet 192.168.100.0 netmask 255.255.255.0 {
  range 192.168.100.10 192.168.100.20;
  option routers 192.168.100.254;
  option domain-name-servers 8.8.8.8;
  option domain-name "mondomaine.org";
  option broadcast-address 192.168.100.255;
  default-lease-time 86400;
  max-lease-time 604800;
}
```

> `images/04-dhcpd-conf.png`

---

## 7. Démarrage et vérification du service

```bash
sudo systemctl start isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

> `images/05-status-service.png`

Une fois le service démarré, on relie le Raspberry Pi à un switch (sans autre serveur DHCP actif dessus) et on y connecte un poste client, afin qu'il obtienne automatiquement une adresse IP par DHCP.

Les échanges DHCP (`DHCPDISCOVER`, `DHCPOFFER`, `DHCPREQUEST`, `DHCPACK`) sont visibles dans les journaux du service, confirmant le bon déroulement de l'attribution.

---

## 8. Test côté client Windows

> `images/06-client-windows-ipconfig.png`

| Paramètre | Valeur obtenue |
|---|---|
| Suffixe DNS | `mondomaine.org` |
| Adresse IPv4 | `192.168.100.12` |
| Masque de sous-réseau | `255.255.255.0` |
| Passerelle par défaut | `192.168.100.254` |

Le client a bien obtenu une adresse comprise dans la plage définie sur le serveur DHCP.

---

## 9. Vérification des baux (fichier leases)

```bash
cd /var/lib/dhcp
cat dhcpd.leases
```

> `images/07-dhcpd-leases.png`

On y retrouve l'adresse `192.168.100.12`, associée à l'adresse MAC de la machine cliente, avec le statut `binding state active`.

---

## 10. Renouvellement du bail

### Côté client Windows

**Libération de l'adresse IP actuelle :**

```powershell
ipconfig /release
```

**Renouvellement du bail auprès du serveur DHCP :**

```powershell
ipconfig /renew "Ethernet"
```

> `images/08-renew-windows.png`

Le client récupère la même adresse IP (`192.168.100.12`), le bail précédent n'ayant pas encore expiré.

---

## 11. Test côté client Linux

Une machine Linux connectée au switch obtient automatiquement une adresse comprise dans la plage configurée sur le serveur DHCP.

```bash
ip a
```

> `images/09-client-linux-ip-a.png`

L'adresse obtenue (`192.168.100.13/24`) est confirmée côté serveur dans le fichier des baux, aux côtés de la machine Windows.

> `images/10-leases-client-linux.png`

---

## 12. Modification de la plage d'adresses

On fait évoluer la plage distribuée par le serveur, de `.10 - .20` à `.40 - .50` :

```conf
range 192.168.100.40 192.168.100.50;
```

Le service est ensuite redémarré pour appliquer la nouvelle configuration :

```bash
sudo systemctl restart isc-dhcp-server
```

Côté client, on libère puis on redemande un bail :

```powershell
ipconfig /release
ipconfig /renew
```

> `images/11-nouvelle-plage-client.png`

Le client obtient bien une adresse comprise dans la nouvelle plage (`192.168.100.40 - 192.168.100.50`).

> `images/12-leases-nouvelle-plage.png`

---

## 13. Réservation d'une adresse IP fixe

Il est possible de réserver une adresse IP fixe pour une machine identifiée par son adresse MAC. On récupère au préalable l'adresse MAC de la machine Linux (`34:17:eb:a9:2d:7e`) via la commande `ip a`.

On ajoute le bloc suivant dans `dhcpd.conf` :

```conf
subnet 192.168.100.0 netmask 255.255.255.0 {
  range 192.168.100.40 192.168.100.50;
  option routers 192.168.100.254;
  option domain-name-servers 8.8.8.8;
  option domain-name "mondomaine.org";
  option broadcast-address 192.168.100.255;
  default-lease-time 86400;
  max-lease-time 604800;

  group {
    use-host-decl-names true;
    host linux1 {
      hardware ethernet 34:17:eb:a9:2d:7e;
      fixed-address 192.168.100.5;
    }
  }
}
```

On force ensuite le client à redemander son adresse :

```bash
sudo dhclient -r    # libère le bail en cours
sudo dhclient       # redemande une adresse au serveur DHCP
```

> `images/13-client-linux-ip-fixe.png`

La machine obtient bien l'adresse IP fixe configurée (`192.168.100.5`).

> [!NOTE]
> Cette adresse **n'apparaît pas** dans le fichier des baux dynamiques (`/var/lib/dhcp/dhcpd.leases`). Une adresse réservée par adresse MAC est définie de manière statique dans `dhcpd.conf` et ne génère pas de bail dynamique classique.

> `images/14-leases-sans-fixe.png`

---

## 14. Récapitulatif de la configuration

| Paramètre | Valeur |
|---|---|
| Adresse du sous-réseau | `192.168.100.0` |
| Plage d'adresses distribuées | `192.168.100.10` – `192.168.100.20` (puis `192.168.100.40` – `192.168.100.50`) |
| Adresse de la passerelle | `192.168.100.254` |
| Adresse du serveur DNS | `8.8.8.8` |
| Domaine DNS | `mondomaine.org` |
| Durée du bail par défaut | 1 jour (86 400 s) |
| Durée maximale du bail | 7 jours (604 800 s) |

---

## 15. Schéma récapitulatif

> `images/15-schema-recapitulatif.png`

Ce schéma synthétise l'architecture complète : le Raspberry Pi assure à la fois la distribution des adresses IP sur le réseau local (`192.168.100.0/24`) et le routage vers Internet via son interface Wi-Fi.

---

[⬅ Retour au sommaire du projet](../README.md) · [Partie suivante : Redondance DHCP ➡](../B%20-%20Redondance%20DHCP/procedure-redondance-dhcp.md)
