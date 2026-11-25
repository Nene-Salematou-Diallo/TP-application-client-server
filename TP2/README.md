# TP2 — Architectures 2-tiers et 3-tiers comparées

Rapport complet : [rapport_TP2.pdf](rapport_TP2.pdf) (énoncé : [enonce_TP2.pdf](enonce_TP2.pdf)).

## Fichiers

| Fichier | Rôle | Où l'utiliser |
|---|---|---|
| `scolarite.sql` | Base `scolarite` (identique au TP1) | VM1 : `sudo mariadb < scolarite.sql` |
| `utilisateurs_tp2.sql` | Compte `app_scolarite`, utilisable uniquement depuis VM2 (`192.168.56.12`) | VM1 : `sudo mariadb < utilisateurs_tp2.sql` |
| `client_lourd.py` | Application 2-tiers (présentation + logique métier sur le poste client) | Poste client : `pip install pymysql` puis `python client_lourd.py` |
| `app3tiers/` | Application 3-tiers en PHP (`index.php`, `releve.php`, `classement.php`, `style.css`, `inc/`) | VM2 (serveur d'application Apache + PHP) |
| `volume.sql` | Ajoute 20 000 étudiants et 120 000 notes (partie 4) | VM1 : `sudo mariadb scolarite < volume.sql` |
| `volume_annuler.sql` | Supprime les données ajoutées par `volume.sql` | VM1 : `sudo mariadb scolarite < volume_annuler.sql` |

## Architecture

- **Poste client** : `client_lourd.py` se connecte directement à MariaDB (VM1, port 3306).
- **VM2** : `app3tiers/` contient la logique métier ; seul VM2 parle à la base.
- **VM1** : MariaDB, `192.168.56.11`.

## Réponses aux questions

Q1 à Q14 : voir le [rapport](rapport_TP2.pdf).
