# TP2 — Architectures 2-tiers et 3-tiers comparées

**Étudiantes :** Nene Salematou DIALLO et Zita Eliane Mame Ndela SENE
**Professeur :** Dr Keba GUEYE — UCAD / École Supérieure Polytechnique, Département Génie Informatique

Rapport complet avec captures d'écran : [rapport_TP2.pdf](rapport_TP2.pdf) · Énoncé : [enonce_TP2.pdf](enonce_TP2.pdf)

## Fichiers du TP

| Fichier | Rôle | Où l'utiliser |
|---|---|---|
| `scolarite.sql` | Base `scolarite` (identique au TP1) | VM1 : `sudo mariadb < scolarite.sql` |
| `utilisateurs_tp2.sql` | Compte `app_scolarite`, utilisable uniquement depuis VM2 | VM1 : `sudo mariadb < utilisateurs_tp2.sql` |
| `client_lourd.py` | Application **2-tiers** (présentation + logique métier sur le poste client) | Poste client : `pip install pymysql` puis `python client_lourd.py` |
| `app3tiers/` | Application **3-tiers** en PHP : `index.php`, `releve.php`, `classement.php`, `style.css`, `inc/config.php`, `inc/metier.php`, `inc/gabarit.php` | VM2 : `/var/www/html/scolarite/` |
| `volume.sql` | Ajoute 20 000 étudiants fictifs et 120 000 notes (Partie 4) | VM1 : `sudo mariadb scolarite < volume.sql` |
| `volume_annuler.sql` | Supprime les données ajoutées par `volume.sql` | VM1 : `sudo mariadb scolarite < volume_annuler.sql` |

## Environnement utilisé

| Machine | Rôle | Adresse IP |
|---|---|---|
| VM1 | Serveur de base de données MariaDB (tier 3 du 3-tiers) | 192.168.204.133 |
| VM2 | Serveur d'application Apache + PHP, et deuxième poste client du 2-tiers | 192.168.204.132 |
| Windows | Poste client (client lourd, navigateur) | 192.168.204.1 |

Notre réseau VMware utilise les adresses `192.168.204.x`. Les fichiers fournis (`192.168.56.x`) sont déposés tels quels : lors du TP, `SERVEUR_BDD` (dans `client_lourd.py`), `BDD_HOTE` (dans `inc/config.php`) et l'adresse d'origine du compte dans `utilisateurs_tp2.sql` ont été adaptés à ce réseau.

## Comment tester

**2-tiers :** installer `pymysql` sur le poste client, adapter `SERVEUR_BDD`, puis `python client_lourd.py` (menu : liste des étudiants, relevé, classement).

**3-tiers :**
1. Sur VM2 : installer Apache, PHP et le module MySQL pour PHP, puis copier `app3tiers/` dans `/var/www/html/scolarite/`.
2. Sur VM1 : `sudo mariadb < utilisateurs_tp2.sql`.
3. Depuis un navigateur : `http://192.168.204.132/scolarite/`.

---

# Réponses aux questions

## Partie 1 — L'application en 2-tiers (client lourd)

### Q1. Lignes de présentation, de logique métier et d'accès aux données. Sur quelle machine s'exécutent-elles ?

`client_lourd.py` mélange les trois couches dans un seul programme :

| Couche | Éléments du code | Lignes |
|---|---|---|
| Présentation | Menu, saisie du matricule, affichage et mise en forme des listes, du relevé et du classement (`print`, `input`, formatage) | L54–55 |
| Logique métier | Calcul de la moyenne pondérée, décision de validation (admis / ajourné), décompte des admis, tri pour le classement | L29–30 et L39–43 |
| Accès aux données | Connexion `pymysql` (adresse, port, utilisateur, mot de passe, base), requêtes SQL (`SELECT`) et lecture des résultats | L22–26, L33–37, L45–49 et L52–58 |

Les trois couches s'exécutent **toutes sur la même machine : le poste client** (Windows ou VM2). Seules les données sont stockées sur le serveur VM1.

### Q2. Quelles informations un étudiant qui récupère `client_lourd.py` obtient-il sur le serveur de base de données ?

Le fichier contient **en clair** tous les paramètres de connexion : adresse IP du serveur, port (3306), nom de la base (`scolarite`), identifiant et mot de passe du compte MariaDB. Avec ces informations, l'étudiant peut se connecter au SGBD depuis n'importe quel autre outil (client `mysql`/`mariadb`, DBeaver, HeidiSQL, script Python…) et exécuter ses propres requêtes avec les privilèges du compte : lire toutes les données de la base (notes de tous les étudiants) et, selon les droits, les modifier ou les supprimer, **sans passer par les règles de gestion du programme**.

### Q3. Qu'aurait-il fallu installer sur les 300 postes d'une salle informatique ?

Sur chacun des 300 postes : Python 3, la bibliothèque `pymysql`, le programme `client_lourd.py` (avec ses paramètres de connexion), et vérifier l'accès réseau au port 3306 de VM1. Soit **300 installations et configurations**, à refaire à chaque nouvelle version.

## Partie 2 — La même application en 3-tiers

L'application est organisée ainsi : `inc/config.php` (paramètres de connexion), `inc/metier.php` (règles de gestion et requêtes), `inc/gabarit.php` (gabarit HTML commun), puis une page par écran (`index.php`, `releve.php`, `classement.php`).

**Depuis quelle machine le compte créé par `utilisateurs_tp2.sql` peut-il se connecter ?** Uniquement depuis **VM2** (192.168.204.132), le serveur d'application : `app_scolarite` est créé avec une adresse d'origine restreinte, donc MariaDB refuse toute connexion de ce compte depuis un autre poste (notamment Windows).

### Q4. Les deux architectures donnent-elles la même moyenne pour ESP260001 et le même nombre d'admis ?

| Résultat | 2-tiers (client lourd) | 3-tiers (web) |
|---|---|---|
| Moyenne de ESP260001 | 11,23 / 20 | 11,23 / 20 |
| Nombre d'admis | 18 / 30 | 18 / 30 |

Oui : les deux utilisent les mêmes données (base `scolarite`) et la même règle de gestion (version 1.0). C'est indispensable car l'expérience doit comparer **uniquement l'architecture**. Si les résultats différaient, on ne saurait pas si les écarts observés ensuite (déploiement, volume, sécurité) viennent de l'architecture ou d'une application différente.

### Q5. Qu'a dû installer le poste Windows pour utiliser la version 3-tiers ?

**Rien** : un navigateur web, déjà présent. Toute l'installation (Apache, PHP, application) se fait une seule fois sur VM2. En 2-tiers (Q3), il fallait installer Python, `pymysql` et le programme sur chacun des 300 postes ; en 3-tiers, aucune installation sur les postes et un seul serveur à configurer.

### Q6. Que renvoie la requête vers `config.php` ? Pourquoi le mot de passe n'est-il jamais envoyé au navigateur ?

Observation (`curl.exe -i http://192.168.204.132/scolarite/inc/config.php`) : `HTTP/1.1 200 OK`, `Server: Apache/2.4.52 (Ubuntu)`, `Content-Length: 0`, `Content-Type: text/html; charset=UTF-8` — **corps vide**.

Le serveur ne renvoie pas le contenu du fichier : `config.php` est un script PHP, **exécuté** par Apache (module PHP) sur VM2 avant toute réponse. Il ne fait que définir des variables et n'affiche rien ; la réponse est le résultat de l'exécution (corps vide), non le code source. Le mot de passe reste sur le serveur d'application et n'est utilisé que pour la connexion PHP → MariaDB, jamais transmis au client. (Cette protection suppose que PHP est correctement interprété par Apache.)

### Q7. Combien de comptes MariaDB faut-il en 3-tiers, quel que soit le nombre d'utilisateurs ? Et en 2-tiers ?

En 3-tiers, **un seul compte** suffit (`app_scolarite`), utilisé uniquement par VM2, quel que soit le nombre d'utilisateurs : ceux-ci s'authentifient auprès de l'application, pas du SGBD. En 2-tiers, chaque poste se connecte directement à MariaDB : il faut soit **un compte par utilisateur** (N comptes pour N utilisateurs), soit un compte partagé dont le mot de passe est distribué à tous les postes, avec une autorisation de connexion depuis tous ces postes.

### Q8. Le code source de la page (Ctrl+U) contient-il du SQL ou la règle de calcul ? Que reçoit exactement le navigateur ?

Non : aucune requête SQL, aucune règle de calcul, aucun mot de passe. Le navigateur reçoit uniquement le **résultat final** : du HTML (et du CSS) généré par PHP sur VM2, c'est-à-dire le texte et les tableaux à afficher. SQL, calculs et accès aux données restent sur les serveurs.

## Partie 3 — Faire évoluer une règle métier

Nouvelle règle (version 1.1) : une note **strictement inférieure à 8** dans un EC empêche la validation, quelle que soit la moyenne (`NOTE_ELIMINATOIRE = 8`).

- **2-tiers :** `client_lourd.py` modifié sur VM2 uniquement (relevé et décompte des admis, `VERSION = "1.1"`). La version 1.0 reste sur Windows, comme un poste non mis à jour.
- **3-tiers :** `/var/www/html/scolarite/inc/metier.php` modifié sur VM2 (relevé et classement, `VERSION_APPLI = '1.1'`).

### Q9. Décision pour ESP260011 et nombre d'admis sur chaque poste en 2-tiers.

| 2-tiers | Version | Décision ESP260011 | Nombre d'admis |
|---|---|---|---|
| Windows (non mis à jour) | 1.0 | VALIDÉ (moyenne 12,55) | 18 / 30 |
| VM2 (mis à jour) | 1.1 | AJOURNÉ (MAT = 7,50 < 8) | 16 / 30 |

Les deux postes, connectés à la même base, donnent des **résultats différents** pour le même étudiant et pour le nombre d'admis. Pour un service de scolarité : décisions contradictoires selon le poste utilisé (étudiant admis ici, ajourné ailleurs), procès-verbaux incohérents, contestations, risques d'erreur ou d'injustice, perte de confiance dans les résultats officiels.

### Q10. Combien de fichiers et de machines modifiés en 3-tiers ? Combien d'utilisateurs ont bénéficié de la règle, et quand ?

**Un seul fichier** (`inc/metier.php`) sur **une seule machine** (VM2). **Tous** les utilisateurs en ont bénéficié, **immédiatement** : dès le rechargement de la page suivante, sans rien installer sur leurs postes (Windows comme VM2 affichent la version 1.1).

### Q11. Où avez-vous dû intervenir pour la règle éliminatoire dans chaque cas ?

- **3-tiers :** dans `inc/metier.php`, sur le serveur : dans les requêtes SQL (le SGBD calcule la moyenne, il faut y ajouter le test de la note minimale) et dans le code PHP de décision, pour le relevé et pour le classement.
- **2-tiers :** dans le code Python de `client_lourd.py` (calcul de la moyenne et décision de validation dans le relevé, puis décompte des admis), **sur chaque poste** qui exécute le programme.

Dans les deux cas, la règle est à modifier à plusieurs endroits (relevé et admis/classement), mais en 3-tiers ces endroits sont centralisés sur un seul serveur.

## Partie 4 — Volume de données et réseau

Sur VM1, `volume.sql` a ajouté 20 000 étudiants fictifs et 120 000 notes (20 030 étudiants et 120 180 notes au total), puis `volume_annuler.sql` a rétabli l'état initial (180 notes).

### Q12. Tableau comparatif. D'où vient l'écart de volume ? Et pour 300 postes à 10 Mbit/s, ou sur connexion mobile ?

| Mesure | 2-tiers | 3-tiers |
|---|---|---|
| Lignes ou octets reçus par le poste client | 120 180 lignes (≈ 2,9 Mo) | 1 page HTML (796 octets mesurés) |
| Temps de calcul du classement | *à re-mesurer avec le volume (voir note)* | 0,35 s (`curl.exe`, 796 octets) |
| Machine qui a effectué le calcul | Poste client (Windows ou VM2) | SGBD (VM1) et serveur d'application (VM2) |

> **Note :** la capture 2-tiers du rapport affiche « 180 lignes rapatriées du serveur en 9 ms (4,2 Kio) », soit une mesure faite sans le volume ajouté ; le temps du classement sur 120 180 lignes est donc à relever à nouveau.

**Origine de l'écart :** en 2-tiers, le client doit rapatrier toutes les données brutes (étudiants et notes) pour calculer lui-même les moyennes et le classement. En 3-tiers, le calcul est fait côté serveur et le poste client ne reçoit que la page HTML du résultat. Le volume reçu par le client est donc très inférieur.

**Conséquences :** avec 300 postes sur une liaison à 10 Mbit/s, le trafic est multiplié par 300 (V × 300 octets). Pour un poste, t = V × 8 / débit ; avec V ≈ 2 870 000 octets (≈ 2,9 Mo), t ≈ 2,3 s par poste, soit ≈ 690 s (environ 11,5 minutes) pour 300 postes si la liaison est partagée. Sur connexion mobile (débit variable, latence élevée, forfait limité), le client lourd serait quasiment inutilisable, alors qu'une page web de quelques kilo-octets reste acceptable.

### Q13. Que mesure `Bytes_sent` ? Pourquoi est-ce une mesure honnête du trafic ?

`Bytes_sent` est une variable d'état de MariaDB qui compte le nombre total d'octets **envoyés par le serveur** à ses clients. La différence entre la valeur après et avant la requête donne le nombre d'octets envoyés en réponse à cette requête. C'est une mesure honnête car elle est faite **par le serveur lui-même**, indépendamment du code du client, et compte les octets réellement émis (données et protocole), sans estimation faite par le programme.

## Partie 5 — Sécurité : fermer l'accès direct à la base

En 3-tiers, seul le serveur d'application a besoin de joindre MariaDB. Sur VM1, le port 3306 a été autorisé uniquement depuis VM2 (l'accès SSH est autorisé d'abord pour ne pas perdre la main sur la VM). L'état du TP1 a été rétabli après la manipulation.

### Q14. Que se passe-t-il pour chaque architecture ? Pourquoi ne peut-on pas appliquer cette protection en 2-tiers ?

| Architecture | Constat |
|---|---|
| 2-tiers (Windows) | Connexion à MariaDB impossible (refus / délai dépassé) : `client_lourd.py` ne fonctionne plus. |
| 3-tiers | L'application web continue de fonctionner : seul VM2 (192.168.204.132) est autorisé à joindre le port 3306. |

En 2-tiers, chaque poste client doit joindre directement le port 3306 : le fermer aux postes casse l'application, et l'ouvrir à tous (postes nombreux, parfois à adresses changeantes) revient à laisser la base exposée. En 3-tiers, **un seul client légitime** du SGBD existe (le serveur d'application), ce qui permet de n'autoriser que lui.

### Q15. Deux autres risques propres au client lourd et protections du 3-tiers.

- **Identifiants de la base embarqués dans le client :** le mot de passe MariaDB est distribué avec le programme et peut être extrait (Q2). En 3-tiers, il reste sur le serveur d'application (`config.php` n'est jamais transmis, Q6) et un compte unique limité à VM2 est utilisé (Q7).
- **Contournement des règles métier, client modifiable :** un utilisateur peut modifier le client ou se connecter avec un autre outil, et ainsi lire ou altérer des données en ignorant les règles de gestion et en exécutant ses propres requêtes SQL. En 3-tiers, le client n'envoie que des requêtes HTTP vers l'application ; règles et requêtes SQL sont exécutées côté serveur, ce qui réduit fortement la surface d'attaque.

## Synthèse et comparaison

| Critère | 2-tiers | 3-tiers |
|---|---|---|
| Logiciel à installer sur chaque poste | Python, `pymysql` et `client_lourd.py` sur chaque poste | Un navigateur web (déjà présent) ; rien à installer |
| Déploiement d'une nouvelle règle | À refaire sur chaque poste (300) ; risque de postes non mis à jour | Un seul fichier sur un seul serveur ; effet immédiat pour tous |
| Cohérence entre utilisateurs | Non garantie : versions différentes, résultats différents (Q9) | Garantie : une seule version de la règle, côté serveur |
| Trafic réseau côté client | Élevé : données brutes rapatriées (proportionnel au volume) | Faible : seule la page HTML est reçue |
| Exposition du SGBD et des mots de passe | Forte : port 3306 ouvert aux postes, mot de passe dans le client | Réduite : un seul compte, port 3306 limité à VM2, mot de passe côté serveur |
| Charge des serveurs | Faible sur le serveur : le calcul est fait par les postes clients | Plus forte : le serveur d'application et le SGBD font les calculs |
| Simplicité de développement | Plus simple : un seul programme, pas de serveur d'application | Plus complexe : serveur d'application, HTTP, séparation des couches |

### Q16. Conditions d'acceptabilité du 2-tiers, et justification du 3-tiers pour un service de scolarité.

Le **2-tiers est acceptable** lorsque : le nombre de postes est faible et maîtrisé ; le réseau est local et de confiance ; la base est proche des postes et le volume transféré reste modeste ; les utilisateurs sont connus et de confiance (techniciens, administrateurs) ; les évolutions sont rares ; et le coût d'un serveur d'application supplémentaire n'est pas justifié.

Pour un **service de scolarité utilisé par toute l'université, le 3-tiers s'impose** : des milliers d'utilisateurs sur des postes non maîtrisés rendent le déploiement par poste impossible, alors qu'avec un navigateur il n'y a rien à installer ; les règles de gestion (validation, moyennes) doivent être identiques pour tous et modifiables en un seul endroit ; la base contient des données sensibles (notes) qu'il ne faut pas exposer aux postes ; le trafic reste faible, même à distance ou sur mobile ; enfin, le serveur d'application permet de centraliser sécurité, journalisation et montée en charge.

## Conclusion

Ce TP a permis de déployer la même application de scolarité en 2-tiers puis en 3-tiers et de les comparer. Le 2-tiers est simple à développer mais impose une installation sur chaque poste, des mises à jour difficiles à maîtriser, un trafic proportionnel au volume de données et une exposition directe de la base. Le 3-tiers centralise la logique métier sur un serveur d'application : déploiement et évolutions immédiats, résultats cohérents, trafic réduit, base protégée, au prix d'une architecture un peu plus complexe.
