# Proxmox VE + Samba Active Directory

Déploiement d'une infrastructure virtualisée avec **Proxmox VE 9.1** et intégration d'un domaine **Samba Active Directory** (`domaine.local`) avec postes clients Windows 10, partages sécurisés, stratégies de groupe, pare-feu et sauvegarde/restauration.

> Projet réalisé dans le cadre du mémoire de fin de cycle (Licence Professionnelle Réseaux Informatiques, Institut Supérieur d'Informatique, Dakar, 2025-2026).
> Lab de test : l'environnement a été supprimé après validation. Le rapport complet avec captures est disponible dans ce dépôt (`docs/`).

## Objectif

Mettre en place une infrastructure centralisée et à coût maîtrisé reposant sur des solutions open source : virtualisation, annuaire, authentification, contrôle d'accès, sécurité et sauvegarde.

## Architecture

| Machine | Rôle | Système | Adresse IP |
|---|---|---|---|
| pve | Hyperviseur Proxmox VE 9.1 | Proxmox VE | 192.168.123.128/24 |
| DEBIAN-SERV (VM 100) | Contrôleur de domaine Samba AD-DC | Debian 13 | 192.168.123.129/24 |
| client1 (VM 101) | Poste Direction | Windows 10 Pro | 192.168.123.130/24 |
| client2 (VM 102) | Poste Comptabilité | Windows 10 Pro | 192.168.123.131/24 |

- Réseau : 192.168.123.0/24, passerelle 192.168.123.2, pont `vmbr0`
- Domaine : `domaine.local` (NetBIOS : `DOMAINE`)
- Le DNS interne du domaine est le serveur DNS des clients ; il redirige les noms externes vers la passerelle.

**Note honnête sur le lab :** faute de serveur physique dédié, Proxmox VE a été installé dans une machine virtuelle VMware Workstation (virtualisation imbriquée) sur un portable Dell Latitude E5450 (8 Go de RAM). C'est un environnement de laboratoire, pas un déploiement bare-metal de production.

## Ce qui a été réalisé

1. **Plateforme de virtualisation** : installation de Proxmox VE, bascule vers les dépôts `no-subscription`, vérification réseau.
2. **Contrôleur de domaine** : VM Debian 13, IP statique, installation de Samba, Kerberos, Winbind, provisioning du domaine (`samba-tool domain provision`), DNS interne `SAMBA_INTERNAL`.
3. **Annuaire** : unités d'organisation (Utilisateurs, Groupes, Ordinateurs), groupes `Direction`, `Informatique`, `Compta` et comptes associés.
4. **Partages SMB sécurisés** : un dossier par groupe sous `/serv/partage`, permissions Unix `2770` (setgid) et directive `valid users` par groupe dans `smb.conf`.
5. **Intégration des clients Windows 10** : IP statique avec le DC comme DNS, jonction au domaine, authentification des comptes de domaine.
6. **Sécurité** : politique de mots de passe, verrouillage de compte à 5 tentatives, pare-feu UFW, GPO d'écran de veille verrouillé (300 s).
7. **Sauvegarde et continuité** : sauvegarde `vzdump` manuelle et planifiée (tous les jours à 21h00, 3 dernières conservées), test de restauration sur une nouvelle VM.

## Extraits de configuration

Partage Samba (`/etc/samba/smb.conf`) :

```ini
[Direction]
    path = /serv/partage/Direction
    read only = no
    valid users = @"DOMAINE\Direction"
```

Permissions du dossier partagé :

```bash
mkdir -p /serv/partage/Direction
chown root:"DOMAINE\\Direction" /serv/partage/Direction
chmod 2770 /serv/partage/Direction
```

Verrouillage de compte :

```bash
samba-tool domain passwordsettings set --account-lockout-threshold=5
samba-tool domain passwordsettings show
```

Pare-feu UFW (refus par défaut des connexions entrantes, accès autorisé depuis le réseau local) :

```bash
ufw default allow outgoing
ufw allow from 192.168.123.0/24
ufw enable
ufw status verbose
```

Vérifications utiles :

```bash
kinit administrator@DOMAINE.LOCAL && klist
samba-tool dbcheck --cross-ncs
samba-tool gpo listall
smbclient //localhost/Direction -U directeur
```

## Tests de validation

| Test | Résultat |
|---|---|
| Ticket Kerberos pour `administrator` | Réussi |
| Intégrité de l'annuaire (`dbcheck`) | 3841 objets vérifiés, 0 erreur |
| Résolution DNS interne (`domaine.local`) | Réussi |
| Authentification de 4 comptes sur 2 postes clients | Réussi |
| Accès de chaque utilisateur à son propre partage | Autorisé |
| Accès croisés entre groupes | Refusé (`NT_STATUS_ACCESS_DENIED`) |
| Verrouillage du compte après dépassement du seuil | Constaté sur un poste client |
| Application de la GPO (`gpresult /r`, écran de veille après 5 min) | Constaté |
| Sauvegarde `vzdump` de la VM du DC | Réussie (archive d'environ 1 Go) |
| Restauration sur la VM 200, IP d'origine conservée | Réussie |

## Limites

- Un seul contrôleur de domaine (pas de réplication ni de redondance).
- Sauvegardes stockées uniquement sur l'hôte Proxmox (pas de copie externalisée).
- Seuls des paramètres GPO de base ont été testés.
- Test limité à 2 postes clients et 4 comptes ; pas de test de charge.
- Hôte unique, sans cluster : haute disponibilité et migration à chaud non testées.

## Pistes d'amélioration

- Second contrôleur de domaine en réplication
- Sauvegardes externalisées (NAS ou cloud)
- Automatisation du déploiement avec Ansible
- Supervision (Prometheus/Grafana ou Zabbix)
- Segmentation réseau (VLAN) et IDS/IPS

## Technologies

Proxmox VE, Debian 13, Samba AD-DC, Kerberos, LDAP, DNS, SMB, GPO, UFW, vzdump, Windows 10, VMware Workstation.

## Structure du dépôt

```
proxmox-samba-AD/
├── README.md
├── docs/
│   └── memoire.pdf        # rapport complet (captures masquées)
├── configs/
│   └── smb.conf           # sans mot de passe
└── images/
    └── architecture.png   # schéma d'architecture
```

## Auteur

Daouda Diop, Licence Professionnelle Réseaux Informatiques, ISI 
