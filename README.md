# Serveur DHCP — Mise en place, redondance et relais

## Présentation

Ce projet documente la mise en place d'un service **DHCP (Dynamic Host Configuration Protocol)** sous Linux, depuis une installation simple jusqu'à des architectures de haute disponibilité utilisées en environnement professionnel.

Le protocole DHCP permet de distribuer automatiquement aux clients d'un réseau les paramètres IP nécessaires à leur fonctionnement (adresse IP, masque de sous-réseau, passerelle, serveur DNS) pour une durée déterminée (le **bail**). Il repose sur un échange en quatre temps entre le client et le serveur : `DHCPDISCOVER` → `DHCPOFFER` → `DHCPREQUEST` → `DHCPACK`.

Le TP est découpé en trois parties, chacune correspondant à un niveau de complexité et à un objectif pédagogique distinct.

> [!NOTE]
> L'ensemble des manipulations a été réalisé sur des machines Linux (Raspberry Pi OS / Ubuntu Server) à l'aide du service **isc-dhcp-server**, la solution DHCP open-source de référence sous Linux, fournie par l'ISC (Internet Software Consortium).

---

## Objectifs du TP

- Comprendre le fonctionnement du protocole DHCP et le rôle de chacun de ses paramètres.
- Installer et configurer un serveur DHCP fonctionnel sur Raspberry Pi, avec routage NAT vers Internet.
- Mettre en évidence les limites d'une architecture DHCP non redondée (perte de service, conflits d'adresses).
- Mettre en place une solution de haute disponibilité DHCP conforme aux standards professionnels (**DHCP Failover**).
- Comprendre le rôle d'un agent relais DHCP sur les réseaux segmentés.

---

## Sommaire du dépôt

| Partie | Contenu | Lien |
|---|---|---|
| **A** | Création d'un serveur DHCP sur Raspberry Pi, avec routage NAT (partage de connexion) vers Internet | [Créer serveur DHCP sur Raspberry](./Partie%20A%20—%20Création%20d'un%20serveur%20DHCP%20sur%20Raspberry%20Pi/procedure-dhcp-raspberry.md) |
| **B** | Redondance DHCP « à plat » : deux serveurs actifs sur la même plage d'adresses, mise en évidence des risques de conflit | [Redondance DHCP](./Partie%20B%20—%20Redondance%20DHCP/procedure-redondance-dhcp.md) |](https://github.com/gcemvoutou/Serveur-DHCP/tree/main/Partie%20B%20%E2%80%94%20Redondance%20DHCP)
| **C** | Agent relais DHCP (théorie) et mise en place d'une grappe DHCP en haute disponibilité (**DHCP Failover**) | [Agent relais DHCP & Grappe DHCP](./Partie%20C%20—%20Agent%20relais%20DHCP/procedure-agent-relais.md)](https://github.com/gcemvoutou/Serveur-DHCP/tree/main/Partie%20C%20%E2%80%94%20Agent%20relais%20DHCP) |
---

## Progression pédagogique

```
Partie A                Partie B                    Partie C
Serveur DHCP     ─────▶  Deux serveurs DHCP  ─────▶  Solution professionnelle
unique                   sans synchronisation        (Failover + relais)
                          (démonstration du risque)
```

> [!TIP]
> Ces trois parties sont conçues pour être lues dans l'ordre : la partie B met volontairement en évidence les limites d'une redondance DHCP non maîtrisée, ce qui justifie la mise en place du mécanisme de **DHCP Failover** présenté en partie C.

---

## Environnement technique

- **OS serveurs :** Raspberry Pi OS (Debian) / Ubuntu Server
- **OS clients de test :** Windows 11, Linux
- **Service DHCP :** isc-dhcp-server (ISC)
- **Virtualisation :** VirtualBox (réseau NAT interne pour les parties B et C)
- **Outils complémentaires :** iptables, sysctl, ntpsec (synchronisation d'horloge pour le failover)
