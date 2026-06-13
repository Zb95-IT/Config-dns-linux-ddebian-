[README-DNS-Bind9.md](https://github.com/user-attachments/files/28914754/README-DNS-Bind9.md)
# Mise en place d'un serveur DNS sous Linux Debian avec Bind9

## Objectif

Installer et configurer un serveur DNS faisant autorité sur la zone `wilders.lan` sous Debian 12, créer un enregistrement de type A et un enregistrement de type CNAME, puis valider la résolution depuis le serveur et depuis un client du réseau local.

Le DNS joue le rôle d'annuaire du réseau. Il traduit un nom comme `web.wilders.lan` en adresse IP. À la différence du DHCP, il ne distribue aucune adresse, il répond simplement à la question "quelle est l'adresse de ce nom".

## Environnement

| Rôle | Système | Adresse IP | Réseau |
|------|---------|-----------|--------|
| Serveur DNS | Debian 12 | 192.168.1.50 | Réseau interne |
| Client de test | Ubuntu | 192.168.1.60 | Réseau interne |

## Prérequis

1. Une VM Debian 12 installée
2. La carte réseau configurée en Réseau interne
3. Un accès root

## 1. Configuration de l'adresse IP fixe

Le réseau interne ne possède pas de serveur DHCP, l'adresse doit donc être fixée manuellement.

Fichier `/etc/network/interfaces` :

```
auto ens33
iface ens33 inet static
    address 192.168.1.50
    netmask 255.255.255.0
```

Application et vérification :

```bash
systemctl restart networking
ip a
```

![Adresse IP du serveur](screenshots/01-ip-serveur.png)

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

![Fichier de zone](screenshots/02-zone.png)

## 5. Validation de la configuration

```bash
named-checkconf
named-checkzone wilders.lan /etc/bind/db.wilders.lan
```

Le résultat attendu est `OK`. Cette étape évite de redémarrer le service avec une erreur de syntaxe.

![Validation de la zone](screenshots/03-checkzone.png)

## 6. Redémarrage du service

Sur Debian 12 le service se nomme `named`.

```bash
systemctl restart named
systemctl status named
```

Le service doit afficher `active (running)`.

![Service actif](screenshots/04-status.png)

## 7. Test depuis le serveur

```bash
dig @127.0.0.1 srv.wilders.lan
dig @127.0.0.1 web.wilders.lan
```

La section ANSWER doit renvoyer 192.168.1.51 pour srv, et le CNAME suivi de l'adresse pour web.

![Résolution de l'enregistrement A](screenshots/05-dig-srv.png)

![Résolution de l'enregistrement CNAME](screenshots/06-dig-web.png)

## 8. Test depuis le client

Le client Ubuntu reçoit lui aussi une adresse fixe puisque le réseau est isolé.

```bash
sudo ip addr add 192.168.1.60/24 dev ens33
sudo ip link set ens33 up
```

Vérification de la connectivité puis de la résolution en interrogeant le serveur :

```bash
ping 192.168.1.50
dig @192.168.1.50 web.wilders.lan
```

![Test depuis le client](screenshots/07-client.png)

## Critères d'acceptation

* [x] La zone se nomme `wilders.lan`
* [x] La résolution de l'enregistrement A fonctionne
* [x] La résolution de l'enregistrement CNAME fonctionne
* [x] Les tests valident la résolution depuis le serveur et depuis un client

## Remarque

Sur le serveur, des messages `network unreachable` peuvent apparaître dans les logs de Bind9. Ils proviennent des tentatives de contact des serveurs racines d'Internet, impossibles sur un réseau interne isolé. Ils n'affectent pas la résolution de la zone `wilders.lan`, qui reste pleinement fonctionnelle.
