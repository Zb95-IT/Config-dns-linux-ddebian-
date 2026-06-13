

# Mise en place d'un serveur DNS sous Linux Debian avec Bind9

## Objectif

Installer et configurer un serveur DNS faisant autorité sur la zone `wilders.lan` sous Debian 12, créer un enregistrement de type A et un enregistrement de type CNAME, puis valider la résolution depuis le serveur et depuis un client du réseau local.

Le DNS joue le rôle d'annuaire du réseau. Il traduit un nom comme `web.wilders.lan` en adresse IP. À la différence du DHCP, il ne distribue aucune adresse, il répond simplement à la question "quelle est l'adresse de ce nom".

## Environnement

| Rôle | Système | Adresse IP | Réseau |
|------|---------|-----------|--------|
| Serveur DNS | Debian 12 | 192.168.1.50 | Réseau interne |
| Client de test | Ubuntu 24.04 | 192.168.1.60 | Réseau interne |

## Prérequis

1. Une VM Debian 12 installée
2. La carte réseau configurée en Réseau interne
3. Un accès root

## 1. Configuration de l'adresse IP fixe

Le réseau interne ne possède pas de serveur DHCP, l'adresse est donc fixée manuellement.

Fichier `/etc/network/interfaces` :

```
auto ens33
iface ens33 inet static
    address 192.168.1.50
    netmask 255.255.255.0
```

```bash
systemctl restart networking
ip a
```

## 2. Installation de Bind9

```bash
apt update
apt install bind9 bind9utils dnsutils
```

Bind9 est le logiciel serveur DNS le plus répandu sous Linux.

## 3. Déclaration de la zone

Fichier `/etc/bind/named.conf.local` :

```
zone "wilders.lan" {
    type master;
    file "/etc/bind/db.wilders.lan";
};
```

Cette déclaration indique à Bind9 qu'il fait autorité sur la zone `wilders.lan` et où se trouvent ses enregistrements.

## 4. Création des enregistrements

Fichier `/etc/bind/db.wilders.lan` :

```
$TTL    604800
@       IN      SOA     ns1.wilders.lan. admin.wilders.lan. (
                        2
                        604800
                        86400
                        2419200
                        604800 )
;
@       IN      NS      ns1.wilders.lan.
ns1     IN      A       192.168.1.50
srv     IN      A       192.168.1.51
web     IN      CNAME   srv.wilders.lan.
```

Détail des enregistrements :

| Nom | Type | Rôle |
|------|------|------|
| ns1 | A | Le serveur DNS local |
| srv | A | Associe le nom srv à l'adresse 192.168.1.51 |
| web | CNAME | Alias qui pointe vers srv |

Un enregistrement A associe un nom à une adresse IP. Un enregistrement CNAME associe un nom à un autre nom.

## 5. Validation et démarrage

```bash
named-checkconf
named-checkzone wilders.lan /etc/bind/db.wilders.lan
systemctl restart named
systemctl status named
```

La commande `named-checkzone` doit afficher `OK` et le service doit être `active (running)`. Les requêtes locales `dig @127.0.0.1` confirment que la zone répond correctement.

![Validation et tests depuis le serveur](screenshots/01-validation-serveur.png)

## 6. Test depuis le client

Le client Ubuntu reçoit lui aussi une adresse fixe puisque le réseau est isolé.

```bash
sudo ip addr add 192.168.1.60/24 dev ens33
sudo ip link set ens33 up
ping 192.168.1.50
```

Résolution de l'enregistrement A en interrogeant le serveur :

```bash
dig @192.168.1.50 srv.wilders.lan
```

![Test de l'enregistrement A depuis le client](screenshots/02-test-client-enregistrement-A.png)

Résolution de l'enregistrement CNAME :

```bash
dig @192.168.1.50 web.wilders.lan
```

La réponse renvoie d'abord le CNAME `web vers srv`, puis l'adresse de srv. La résolution traverse donc bien l'alias.

![Test de l'enregistrement CNAME depuis le client](screenshots/03-test-client-enregistrement-CNAME.png)

## Critères d'acceptation

* [x] La zone se nomme `wilders.lan`
* [x] La résolution de l'enregistrement A fonctionne
* [x] La résolution de l'enregistrement CNAME fonctionne
* [x] Les tests valident la résolution depuis le serveur et depuis un client

## Remarque

Sur le serveur, des messages `network unreachable` peuvent apparaître dans les logs de Bind9. Ils proviennent des tentatives de contact des serveurs racines d'Internet, impossibles sur un réseau interne isolé. Ils n'affectent pas la résolution de la zone `wilders.lan`, qui reste pleinement fonctionnelle.
