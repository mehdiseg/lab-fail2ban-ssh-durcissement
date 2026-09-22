# Lab sécurité : durcir SSH et bannir les attaques avec fail2ban

> **Statut : à réaliser.** Ce guide est préparé à partir de la documentation officielle et de mes cours ; **je ne l'ai pas encore rejoué de bout en bout**. Les commandes sont à valider en le faisant, et le journal en bas de page sera complété avec mes résultats réels (captures, erreurs rencontrées, corrections).
>
> **Commandes vérifiées :** ce guide a été rejoué dans des conteneurs Debian 12 et 13 (22 septembre 2026) avec les vrais `sshd` et fail2ban : `sshd -t` valide la configuration, une connexion par clé fonctionne pour l'administrateur mais est refusée pour `root` et pour un compte hors `AllowUsers`, et un mot de passe est refusé même correct. `fail2ban-client -t` valide la jail, son filtre reconnaît 5 vrais messages `Failed password` sans confondre une connexion réussie, et un vrai serveur fail2ban a banni l'adresse au 5ᵉ échec, épargné l'adresse du réseau protégé, et accepté le débannissement. C'est ce test qui a révélé qu'il manquait `backend = systemd` dans `jail.local` sur une Debian minimale : la ligne a été ajoutée. Vérifié ne veut pas dire réalisé : c'est l'assistant IA qui a préparé ce guide qui a rejoué ces commandes dans un conteneur jetable, pas moi sur mon propre lab. Le journal ci-dessous reste à remplir une fois que je l'aurai fait moi-même.

## Objectif

Protéger l'accès SSH d'un serveur Debian exposé à internet :

1. **durcir** `sshd` : plus de connexion root, plus de mot de passe, authentification par **clé** ;
2. **bannir automatiquement** les adresses qui essaient de deviner un mot de passe avec **fail2ban**.

Ce lab complète [serveur-debian-lemp-securise](https://github.com/mehdiseg/serveur-debian-lemp-securise), qui met en place le pare-feu UFW.

## Prérequis

- Une VM **Debian 12** avec un utilisateur non-root disposant de `sudo`, et une seconde machine pour les tests.
- **Garder une session SSH ouverte pendant toute la manipulation** : une erreur de configuration peut verrouiller l'accès, et il faut pouvoir la corriger sans console.

## Étapes

### 1. Créer une clé et la copier sur le serveur (sur le poste client)

```bash
ssh-keygen -t ed25519 -C "poste-lab"
ssh-copy-id admin@192.168.50.10
ssh admin@192.168.50.10          # doit se connecter sans mot de passe (avec la phrase secrète de la clé)
```

Protéger la clé privée par une **phrase secrète**. Ne jamais copier la clé **privée** (`id_ed25519`) hors du poste.

### 2. Durcir `sshd` (sur le serveur)

Créer `/etc/ssh/sshd_config.d/durcissement.conf` :

Fichier du dépôt : [`configs/durcissement.conf`](configs/durcissement.conf)

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
AllowUsers admin
X11Forwarding no
```

Tester la configuration **avant** de recharger, puis recharger sans couper les sessions existantes :

```bash
sudo sshd -t && sudo systemctl reload ssh
```

Ouvrir alors un **deuxième** terminal et vérifier qu'une nouvelle connexion par clé fonctionne avant de fermer le premier.

### 3. Installer fail2ban

```bash
sudo apt install -y fail2ban
```

Créer `/etc/fail2ban/jail.local` (ne jamais modifier `jail.conf`, qui est écrasé par les mises à jour) :

Fichier du dépôt : [`configs/jail.local`](configs/jail.local)

```text
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 192.168.50.0/24

[sshd]
enabled = true
backend = systemd
```

```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
```

`ignoreip` protège son propre réseau d'un auto-bannissement.

**`backend = systemd` est indispensable sur une Debian 12 « minimale ».** Sans `rsyslog`, le fichier `/var/log/auth.log` n'existe pas, et fail2ban refuse de démarrer la jail avec l'erreur `Have not found any log file for sshd jail` (constaté en testant cette configuration dans un conteneur Debian 12 : la ligne `backend = systemd` fait passer `fail2ban-client -t` de l'échec au succès). Avec cette ligne, fail2ban lit les échecs de connexion directement dans le journal systemd. Vérifier ensuite avec `sudo fail2ban-client status sshd` et `sudo journalctl -u ssh`.

## Vérifications

```bash
sudo sshd -T | grep -Ei "permitrootlogin|passwordauthentication|maxauthtries"
sudo fail2ban-client status sshd        # jail active, nombre d'échecs et d'adresses bannies
```

Test de bannissement, sur une **VM de test uniquement** : les échecs que fail2ban sait lire sont les « Failed password ». Réactiver donc **temporairement** `PasswordAuthentication yes` (puis `sudo sshd -t && sudo systemctl reload ssh`), et depuis une **autre** machine, hors de `ignoreip` :

```bash
ssh -o PubkeyAuthentication=no utilisateur-bidon@192.168.50.10    # 5 essais avec un mauvais mot de passe
```

- `sudo fail2ban-client status sshd` liste ensuite l'adresse dans `Banned IP list`, et la connexion est bloquée.
- Vérifier ce qui est détecté, sans attendre : `sudo fail2ban-regex systemd-journal sshd` (nombre de lignes reconnues par le filtre).
- Débannir : `sudo fail2ban-client set sshd unbanip <adresse>`.
- Suivre en direct : `sudo journalctl -u fail2ban -f`.
- **Remettre `PasswordAuthentication no`** et recharger `sshd` une fois le test terminé.

## Pièges fréquents

- **Se verrouiller dehors** : `PasswordAuthentication no` avant d'avoir copié la clé, ou `AllowUsers` sans son propre compte. D'où la session gardée ouverte et `sshd -t`.
- Le service s'appelle `ssh` sous Debian (`sshd` sous d'autres distributions).
- Fail2ban qui ne démarre pas (« Have not found any log file ») ou qui ne détecte rien : `backend = systemd` manquant (voir plus haut), ou `rsyslog` absent alors que le backend attend `/var/log/auth.log`.
- Bannir sa propre adresse en testant : prévoir `ignoreip` ou tester depuis une machine dédiée.

## Pour aller plus loin

- Changer de port (`Port 2222`) : un peu de bruit en moins, **pas une sécurité** ; ne remplace pas ce qui précède.
- Exiger une seconde authentification (clé + code TOTP avec `libpam-google-authenticator`).
- Envoyer une notification à chaque bannissement (`action = %(action_mwl)s`).
- Détecter les scans de ports plutôt que seulement les échecs SSH : [lab-suricata-ids-detection](https://github.com/mehdiseg/lab-suricata-ids-detection).

## Références

- [Wiki de fail2ban](https://github.com/fail2ban/fail2ban/wiki)
- [Page de manuel de sshd_config](https://man.openbsd.org/sshd_config)

## Journal de réalisation

_Lab pas encore réalisé : cette section sera remplie au fur et à mesure._

| Date | Ce que j'ai fait | Résultat | Difficultés et solutions |
|---|---|---|---|
|  |  |  |  |

## Feuille de route

Ce lab fait partie de ma [feuille de route réseau](https://github.com/mehdiseg/roadmap-reseau-bts-sio).
