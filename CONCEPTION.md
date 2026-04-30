# Conception

Notre conception de l'API REST pour le jeu de Bataille Navale.

## Ressources principales

On a identifié 4 ressources métier :

- les **utilisateurs** (joueurs)
- les **navires de référence** (la flotte type, ex: porte-avion = 5 cases)
- les **parties** (une partie entre 2 joueurs)
- les **navires placés** dans une partie + les **tirs** effectués

## Liste des fonctionnalités

Côté client, on imagine avoir besoin de :

- se connecter / vérifier qu'on est bien connecté
- récupérer la liste des navires (pour savoir ce qu'on a à placer)
- créer une partie
- récupérer l'état d'une partie
- placer un navire / le déplacer
- récupérer le placement de ses navires
- tirer une torpille
- récupérer la liste des tirs déjà faits

## Liste des *Endpoints*

### Authentification

#### `POST /login`
Récupère un token JWT à partir d'un identifiant et d'un mot de passe.

- **en-têtes** : `Authorization: Basic <base64(login:mdp)>`
- **corps** : aucun
- **réponses** :
  - `200` → `{ "token": "<jwt>" }`
  - `401` si identifiants incorrects → `{ "error": "Identifiants invalides" }`

#### `GET /auth`
Vérifie la validité du JWT envoyé.

- **en-têtes** : `Authorization: Bearer <jwt>`
- **réponses** :
  - `200` → `{ "valid": true, "payload": { ... } }`
  - `401` si le token est invalide ou expiré

---

### Flotte de référence

#### `GET /ships`
Liste des types de navires disponibles dans une partie (porte-avion, croiseur, etc.).

- **en-têtes** : aucun particulier (pas besoin d'être connecté pour voir ça)
- **réponse** :
  - `200` → tableau JSON
  ```json
  [
    { "id": 1, "name": "aircraft-carrier", "length": 5 },
    { "id": 2, "name": "cruiser", "length": 4 },
    ...
  ]
  ```

---

### Parties

#### `POST /games`
Crée une nouvelle partie.

- **en-têtes** : `Authorization: Bearer <jwt>`
- **corps** : pour l'instant rien (on part du principe que la partie sera entre les 2 joueurs déjà en BDD)
- **réponses** :
  - `201` → `{ "id": 1, "status": "placement", "players": [...] }`
  - `401` si pas authentifié

#### `GET /games/{id}`
Récupère les infos d'une partie (statut, joueurs, à qui le tour…).

- **en-têtes** : `Authorization: Bearer <jwt>`
- **paramètres URL** : `id` = identifiant de la partie
- **réponses** :
  - `200` → `{ "id": 1, "status": "placement|en_cours|fini", "currentPlayer": ..., "winner": ... }`
  - `404` si la partie n'existe pas

#### `GET /games`
Liste des parties (utile plus tard pour le matchmaking).

- **réponse** : `200` + tableau de parties

---

### Placement des navires

On place les navires d'un joueur dans une partie donnée.
Pour l'instant on identifie le joueur via le JWT (ou en dur si pas encore prêt).

#### `POST /games/{id}/ships`
Place un nouveau navire sur la grille.

- **en-têtes** : `Authorization: Bearer <jwt>`
- **paramètres URL** : `id` = id de la partie
- **corps** :
  ```json
  {
    "shipId": 1,
    "x": "D",
    "y": 5,
    "orientation": "H"
  }
  ```
  (`shipId` correspond à un navire de la flotte, `x`/`y` la coord en haut à gauche, orientation H ou V)
- **réponses** :
  - `201` → le navire placé
  - `400` si placement invalide (hors grille, chevauchement, navires qui se touchent…)
  - `401` si pas authentifié
  - `409` si le navire a déjà été placé

#### `PUT /games/{id}/ships/{shipId}`
Déplace ou réoriente un navire déjà placé.

- **corps** : même format que pour le POST
- **réponses** :
  - `200` → navire mis à jour
  - `400` si placement invalide

#### `GET /games/{id}/ships`
Récupère le placement des navires du joueur connecté pour cette partie.

- **réponses** :
  - `200` → tableau de navires placés
  - `404` si la partie n'existe pas

---

### Tirs (en cours de partie)

#### `POST /games/{id}/shots`
Tire une torpille sur la grille adverse.

- **en-têtes** : `Authorization: Bearer <jwt>`
- **corps** :
  ```json
  { "x": "B", "y": 6 }
  ```
- **réponses** :
  - `201` → résultat du tir
    ```json
    { "result": "manqué|touché|touché-coulé", "x": "B", "y": 6 }
    ```
  - `400` si coordonnées invalides ou si déjà tiré ici
  - `403` si ce n'est pas son tour
  - `404` si partie inconnue

#### `GET /games/{id}/shots`
Récupère la liste des tirs de la partie (les siens et ceux reçus).

- **réponse** : `200` + tableau des tirs

---

## Notes

- Tous les endpoints renvoient du **JSON** (en-tête `Content-Type: application/json`).
- On utilise un **JWT** transmis via l'en-tête `Authorization: Bearer …` pour identifier le joueur (sauf `/login` et `/ships`).
- Les codes de statut suivent la logique habituelle : `200` ok, `201` création, `400` requête invalide, `401` pas connecté, `403` interdit, `404` introuvable, `409` conflit.
- Le **matchmaking** plus poussé (chercher un adversaire, créer/rejoindre une partie ouverte…) sera ajouté plus tard. Pour le MVP on part avec une partie créée entre les 2 joueurs déjà en BDD (bean / elfo).

## MVP (à implémenter en priorité)

- `POST /login` + `GET /auth`
- `GET /ships`
- `POST /games`
- `POST /games/{id}/ships` + `PUT /games/{id}/ships/{shipId}`
- `POST /games/{id}/shots` (en dernier)
