 TP1 — Le modèle client-serveur

**Étudiantes :** Nene Salematou DIALLO et Zita Eliane Mame Ndela SENE
**Professeur :** Dr Keba GUEYE — UCAD / École Supérieure Polytechnique, Département Génie Informatique

Rapport complet avec captures d'écran : [rapport_TP1.pdf](rapport_TP1.pdf) · Énoncé : [enonce_TP1.pdf](enonce_TP1.pdf)

## Fichiers du TP

| Fichier | Rôle | Où l'utiliser |
|---|---|---|
| `scolarite.sql` | Création et remplissage de la base `scolarite` (données fictives) | VM1 : `sudo mariadb < scolarite.sql` |
| `utilisateurs_tp1.sql` | Comptes distants `etudiant` (lecture seule) et `secretariat` (saisie des notes) | VM1 : `sudo mariadb < utilisateurs_tp1.sql` |
| `index.html` | Page servie par Apache (port 80) | VM1 : `/var/www/html/index.html` |

## Environnement utilisé

| Machine | Rôle | Adresse IP |
|---|---|---|
| VM1 | Serveur : MariaDB (port 3306) et Apache (port 80) | 192.168.204.133 |
| VM2 | Client (`mariadb`, `curl`) | 192.168.204.132 |
| Windows | Client (HeidiSQL, navigateur, Wireshark) | 192.168.204.1 |

Notre réseau VMware utilise les adresses `192.168.204.x`. Les fichiers fournis (`192.168.56.%`) sont déposés tels quels ; lors du TP, les adresses de `utilisateurs_tp1.sql` ont été adaptées à ce réseau, et le compte `secretariat` a été autorisé pour l'adresse réelle de VM2 (`192.168.204.132`).

## Comment tester

1. Sur VM1 : `sudo mariadb < scolarite.sql` puis `sudo mariadb < utilisateurs_tp1.sql`.
2. Dans `/etc/mysql/mariadb.conf.d/50-server.cnf`, mettre `bind-address = 0.0.0.0`, puis `sudo systemctl restart mariadb`.
3. Ouvrir le port : `sudo ufw allow 3306/tcp`.
4. Depuis un client : `mariadb -h 192.168.204.133 -u etudiant -p scolarite`.
5. Serveur web : installer Apache, copier `index.html` dans `/var/www/html/`, puis ouvrir `http://192.168.204.133/`.

---

# Réponses aux questions

## Partie 1 — Préparer le serveur (VM1)

### Q1. Combien d'étudiants, d'EC et de notes ? Schéma des quatre tables.

- Étudiants : **30**
- EC : **6**
- Notes : **180**

| Table | Clé primaire | Clés étrangères / colonnes |
|---|---|---|
| `filiere` | `code` | `libelle` |
| `etudiant` | `matricule` | `prenom`, `nom`, `email` ; `filiere` → `filiere.code` |
| `ec` | `code` | `intitule`, `credits` |
| `note` | `id` (auto-incrément) | `matricule` → `etudiant.matricule` ; `ec` → `ec.code` ; `valeur` ; couple (`matricule`, `ec`) unique |

### Q2. Le client `mariadb` et le serveur `mariadbd` sont-ils le même programme ?

Non. Ils tournent sur la même machine mais sont deux programmes distincts : `ps -ef | grep -i mariadb` montre deux processus avec des PID différents. Le serveur `mariadbd` tourne en permanence et attend les demandes ; le client `mariadb` est lancé par l'utilisateur, se connecte au serveur, envoie ses requêtes et affiche les réponses. Les rôles client et serveur sont donc des **rôles de programmes, pas de machines**.

## Partie 2 — Ouvrir le service au réseau

Première tentative depuis Windows : échec avec l'erreur 2002 (10061). Sur VM1, `ss` montrait le serveur à l'écoute uniquement sur `127.0.0.1:3306`. Nous avons remplacé `bind-address = 127.0.0.1` par `bind-address = 0.0.0.0`, redémarré MariaDB, créé les comptes distants (`utilisateurs_tp1.sql`), ouvert le port 3306 dans `ufw`, puis nous nous sommes reconnectées depuis Windows avec le compte `etudiant`.

### Q3. Que signifie l'erreur 10061 ? VM1 était-elle joignable ? MariaDB fonctionnait-il ?

L'erreur 10061 signifie que la connexion a été **refusée** : la machine a répondu, mais aucun programme n'acceptait de connexion sur `192.168.204.133`, port 3306. VM1 était donc joignable et MariaDB fonctionnait, mais il n'écoutait que sur `127.0.0.1` (sortie de `ss`), donc uniquement pour les clients locaux.

### Q4. Différence entre écouter sur 127.0.0.1 et sur 0.0.0.0 ? Pourquoi 127.0.0.1 par défaut ?

Sur `127.0.0.1` (boucle locale), le serveur n'accepte que les connexions venant de VM1 elle-même. Sur `0.0.0.0`, il écoute sur toutes les adresses de la machine, donc aussi sur `192.168.204.133`, et devient joignable depuis le réseau. Le défaut est `127.0.0.1` **par sécurité** : un service de base de données fraîchement installé n'est pas exposé au réseau tant que l'administrateur ne l'a pas décidé et n'a pas défini les comptes et droits distants.

### Q5. `SELECT USER(), CURRENT_USER();` — quelle différence entre les deux valeurs ?

Résultat : `USER()` = `etudiant@192.168.204.1` ; `CURRENT_USER()` = `etudiant@192.168.204.%`.

`USER()` indique **qui se connecte réellement** : le nom donné par le client et l'adresse de la machine d'origine (le poste Windows). `CURRENT_USER()` indique le **compte du serveur retenu** pour cette connexion, tel que défini dans `utilisateurs_tp1.sql` ; ce sont ses droits qui s'appliquent. Les deux diffèrent quand le compte est défini pour un ensemble de machines (`192.168.204.%`) plutôt que pour une adresse précise.

### Q6. Le poste Windows peut-il se connecter en `root` depuis le réseau ? Pourquoi est-ce une bonne chose ?

Non : `Access denied for user 'root'@'192.168.204.1' (using password: NO)`. Le compte `root` n'est autorisé qu'en local sur VM1 (`sudo mariadb`) et aucun compte `root` n'existe pour les machines du réseau. C'est une bonne chose : le compte qui a tous les droits ne peut pas être attaqué à distance ; les clients du réseau n'utilisent que des comptes limités (`etudiant`, `secretariat`).

## Partie 3 — Plusieurs clients, un seul serveur

Le client `mariadb` a été installé sur VM2. Une requête `SELECT SLEEP(20);` a été lancée en même temps depuis Windows (compte `etudiant`) et depuis VM2 (compte `secretariat`), et le serveur observé pendant ces 20 secondes (`PROCESSLIST`, `ss`).

### Q7. Adresse IP et port source de chaque client.

| Client | Adresse IP | Port source |
|---|---|---|
| Windows (`etudiant`) | 192.168.204.1 | 53663 |
| VM2 (`secretariat`) | 192.168.204.132 | 57334 |

Les ports source sont choisis automatiquement par le système d'exploitation de chaque client. Ils diffèrent de 3306, qui est le port du serveur, connu à l'avance ; le port source sert seulement à identifier la connexion côté client. C'est le couple (adresse, port) de chaque côté qui permet au serveur de distinguer les connexions simultanées.

### Q8. Messages d'erreur obtenus. Où sont enregistrés les droits et qui les vérifie ?

- `UPDATE` par `etudiant` (Windows) : **refusé**, erreur 1142 « UPDATE command denied to user 'etudiant'@'192.168.204.1' for table `scolarite`.`note` »
- `UPDATE` par `secretariat` (VM2) : **accepté**, « Query OK, 1 row affected »
- `DELETE` par `secretariat` (VM2) : **refusé**, erreur 1142 « DELETE command denied to user 'secretariat' for table `scolarite`.`note` »

Les droits sont enregistrés **sur le serveur** (créés par `utilisateurs_tp1.sql`) et c'est le serveur qui les vérifie à chaque requête, pas le client. **Conclusion :** la sécurité ne repose pas sur le logiciel client ; un utilisateur qui change de client ou le modifie ne peut pas obtenir plus de droits que ceux que le serveur lui accorde.

### Q9. Le client Windows voit-il immédiatement la note modifiée depuis VM2 ? Pourquoi ?

Oui : la valeur relue depuis Windows est **12**, celle modifiée depuis VM2. Les données n'existent qu'à un seul endroit, sur le serveur ; les clients n'en gardent pas de copie et interrogent le serveur à chaque requête. Toute modification est donc vue par tous les autres clients dès leur requête suivante : c'est cette centralisation qui garantit la **cohérence**.

## Partie 4 — Un deuxième service : le serveur web

Apache a été installé sur VM1 et `index.html` copié dans `/var/www/html/`. La page a été ouverte depuis le navigateur de Windows, puis avec `curl -v` depuis VM2, et le journal `access.log` consulté sur VM1.

### Q10. Requête du client et réponse du serveur dans la sortie de `curl -v`.

Les lignes commençant par `>` sont la requête du client ; celles commençant par `<` sont la réponse du serveur.

- **Requête :** `GET / HTTP/1.1` contient la méthode (`GET`), la ressource demandée (`/`) et la version du protocole. Les en-têtes indiquent la machine visée (`Host: 192.168.204.133`) et le logiciel client (`User-Agent: curl/…`).
- **Réponse :** `HTTP/1.1 200 OK` contient la version et le code d'état (200 = succès). Les en-têtes indiquent le logiciel serveur (`Server: Apache/…`) et le type du contenu (`Content-Type: text/html`), suivis de la page elle-même.

### Q11. Informations enregistrées dans `access.log` sur chacun des deux clients.

Pour chaque requête, le serveur enregistre l'adresse IP du client (`192.168.204.1` pour Windows, `192.168.204.132` pour VM2), la date et l'heure, la requête (`GET / HTTP/1.1`), le code de réponse (200), la taille de la réponse et le logiciel client (Firefox pour Windows, `curl/7.81.0` pour VM2). Le navigateur a aussi demandé automatiquement `/favicon.ico`, absent du serveur, d'où le code **404**.

### Q12. Comment le système sait-il à quel programme remettre un segment TCP ? Pourquoi le client doit-il connaître l'adresse ET le port ?

Le système regarde le **port de destination** du segment : 3306 est associé à `mariadbd`, 80 à Apache (visible avec `ss -tlnp`). L'adresse IP désigne la machine (VM1), le port désigne le programme sur cette machine. Comme c'est toujours le client qui prend l'initiative, il doit connaître les deux à l'avance : l'adresse pour atteindre VM1, le port pour atteindre le bon service parmi ceux qu'elle héberge.

### Q13. Connexion persistante (MariaDB) contre requête-réponse (HTTP).

| Mode | Avantage | Inconvénient |
|---|---|---|
| MariaDB (connexion persistante) | Le client s'identifie une seule fois puis enchaîne ses requêtes rapidement ; le serveur connaît le contexte de la session. | Chaque client occupe une connexion pendant toute la session, même inactif : le nombre de clients simultanés est limité. |
| HTTP (requête-réponse) | Le serveur ne garde rien entre deux requêtes : il peut servir un très grand nombre de clients. | Chaque requête est indépendante : il faut renvoyer à chaque fois les informations nécessaires, et le serveur ne « se souvient » pas du client. |

Pour se souvenir d'un utilisateur connecté, un site web lui donne après la connexion un **identifiant de session**, que le navigateur renvoie automatiquement à chaque requête suivante (cookie). Le serveur retrouve ainsi l'utilisateur à chaque requête.

## Partie 5 — Les limites du modèle

### 5.1 Le serveur s'arrête — Q14

MariaDB a été arrêté sur VM1 (`sudo systemctl stop mariadb`), puis une requête a été tentée depuis les deux clients avant de redémarrer le service.

- **Windows (HeidiSQL) :** la requête échoue et HeidiSQL affiche une fenêtre d'erreur « An error occurred in the application » (capture dans le rapport).
- **VM2 :** `ERROR 2002 (HY000): Can't connect to server on '192.168.204.133' (115)`

**Tous** les clients sont affectés : sans le serveur, aucun ne peut accéder aux données, puisqu'elles ne sont qu'à cet endroit et que les clients ne communiquent jamais entre eux. Ce défaut d'architecture s'appelle le **point unique de défaillance**.

### 5.2 Ce qui circule sur le réseau — Q15

Le trafic sur le port 3306 a été capturé avec Wireshark sur Windows (carte VMnet8, filtre `mysql`) pendant que Windows exécutait la requête sur l'étudiant `ESP260011`.

- Nom du compte retrouvé : `etudiant` (paquet « Login Request user=etudiant »)
- Requête retrouvée : `SELECT nom, valeur FROM note JOIN etudiant USING(matricule) WHERE matricule = 'ESP260011'`
- Données de la réponse retrouvées : le nom `SECK` et ses notes 9.50, 16.50, 7.50, 15.50, 14.75 et 11.75
- Mot de passe en clair : **non** — `Etudiant@2026` n'apparaît nulle part dans la capture

Le nom du compte, le texte de la requête et les données de la réponse circulent **en clair** : n'importe qui capable de capturer le trafic peut les lire. Pour protéger ces échanges, il faut chiffrer la connexion entre client et serveur, par exemple avec **TLS** (connexion MariaDB chiffrée, et HTTPS pour le serveur web).

## Synthèse

| Caractéristique | MariaDB | Apache (HTTP) |
|---|---|---|
| Machine et port du serveur | VM1 (192.168.204.133), port 3306 | VM1 (192.168.204.133), port 80 |
| Logiciel client utilisé | HeidiSQL ou `mariadb` (Windows), `mariadb` (VM2) | Navigateur (Windows), `curl` (VM2) |
| Qui prend l'initiative de la connexion ? | Le client | Le client |
| Durée de vie d'une connexion | Toute la session du client | Une requête et sa réponse |
| Où sont vérifiés les droits ? | Sur le serveur (comptes et droits MariaDB) | Sur le serveur |
| Effet d'un arrêt du serveur | Plus aucun client ne peut accéder aux données | Plus aucun client ne peut afficher le site |

### Q16. Définition du modèle client-serveur, deux avantages et deux inconvénients.

Le modèle client-serveur répartit les rôles entre deux types de programmes. Le **serveur** (ici `mariadbd` ou Apache sur VM1) attend des demandes sur un port connu d'une machine connue. Les **clients** (HeidiSQL, `mariadb`, navigateur, `curl`) prennent l'initiative de la connexion, envoient une requête et attendent la réponse. Ce sont des rôles de programmes et non de machines : client et serveur peuvent tourner sur la même machine, comme en Partie 1, et une même machine peut héberger plusieurs serveurs distingués par leur port. Le serveur centralise les données et décide des droits d'accès ; les clients ne communiquent jamais entre eux. Il peut servir plusieurs clients en même temps, chacun identifié par son adresse et son port source.

**Avantages**
- **Cohérence des données :** centralisées sur le serveur, toute modification est vue immédiatement par tous les clients (Q9).
- **Sécurité centralisée :** les droits sont définis et vérifiés sur le serveur, indépendamment du logiciel client (Q8).

**Inconvénients**
- **Point unique de défaillance :** si le serveur s'arrête, tous les clients sont bloqués (Q14).
- **Trafic en clair par défaut :** comptes, requêtes et données peuvent être lus sur le réseau sans chiffrement (Q15).
