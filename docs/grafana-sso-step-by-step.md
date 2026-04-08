# Mise en place pas à pas du SSO Keycloak pour Grafana

## Objectif

Déployer `Grafana` avec Docker Compose et configurer manuellement l'intégration SSO dans `Keycloak` afin de découvrir l'interface d'administration, le paramétrage du client OpenID Connect et la gestion des droits.

## Prérequis

- Docker et Docker Compose installés
- Ports libres: `3000`, `8080`, `5432`, `9000`
- Accès au dépôt local

## Etape 1 - Préparer les variables

Copie le fichier d'exemple:

```bash
cp .env.example .env
```

Vérifie au minimum les variables suivantes dans `.env`:

```env
KC_BOOTSTRAP_ADMIN_USERNAME=admin
KC_BOOTSTRAP_ADMIN_PASSWORD=ChangeThisAdminPassword!
KC_DB_PASSWORD=ChangeThisDatabasePassword!

GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=ChangeThisGrafanaAdminPassword!
GRAFANA_ROOT_URL=http://localhost:3000

KEYCLOAK_REALM=company
KEYCLOAK_PUBLIC_URL=http://localhost:8080
KEYCLOAK_INTERNAL_URL=http://keycloak:8080
GRAFANA_OAUTH_CLIENT_ID=grafana-oauth
GRAFANA_OAUTH_CLIENT_SECRET=ChangeThisGrafanaClientSecret!
```

## Etape 2 - Démarrer la stack

```bash
docker compose up -d --build
```

## Etape 3 - Vérifier les services

Interfaces attendues:

- Keycloak: `http://localhost:8080`
- Admin Keycloak: `http://localhost:8080/admin`
- Grafana: `http://localhost:3000`
- Health Keycloak: `http://localhost:9000/health/ready`

## Etape 4 - Se connecter à l'administration Keycloak

Connecte-toi à l'admin Keycloak sur `http://localhost:8080/admin` avec le compte bootstrap défini dans `.env`.

Sélectionne ensuite le realm `company`.

## Etape 5 - Créer le client Grafana manuellement

Dans l'admin Keycloak:

1. Ouvre `Clients`
2. Clique sur `Create client`
3. Renseigne `Client type`: `OpenID Connect`
4. Renseigne `Client ID`: `grafana-oauth`
5. Clique sur `Next`

Sur l'écran de capacité:

1. Active `Client authentication`
2. Laisse `Standard flow` activé
3. Désactive les autres flows si non nécessaires
4. Clique sur `Next`

Sur l'écran de login:

1. `Root URL`: `http://localhost:3000`
2. `Home URL`: `http://localhost:3000`
3. `Valid redirect URIs`: `http://localhost:3000/login/generic_oauth`
4. `Valid post logout redirect URIs`: `http://localhost:3000`
5. `Web origins`: `http://localhost:3000`
6. Clique sur `Save`

## Etape 6 - Récupérer le secret du client

Une fois le client créé:

1. Ouvre `Clients` -> `grafana-oauth`
2. Va dans l'onglet `Credentials`
3. Copie le `Client secret`
4. Mets cette valeur dans `.env` pour `GRAFANA_OAUTH_CLIENT_SECRET`

Si tu veux utiliser un autre `Client ID`, mets aussi la même valeur dans `GRAFANA_OAUTH_CLIENT_ID`.

Redémarre ensuite Grafana:

```bash
docker compose up -d grafana
```

## Etape 7 - Comprendre comment Grafana lit les droits

Le rôle Grafana est déterminé à partir des rôles de realm Keycloak déjà présents dans le dépôt:

- `platform-admin` donne `Admin`
- `manager` donne `Editor`
- sinon l'utilisateur devient `Viewer`

Autrement dit:

- si un utilisateur possède le rôle `platform-admin`, Grafana lui donne les droits d'administration
- si un utilisateur possède le rôle `manager`, Grafana lui donne les droits d'édition
- si l'utilisateur n'a aucun de ces rôles, il reste en lecture seule

## Etape 8 - Gérer les droits manuellement dans Keycloak

Le dépôt importe déjà des rôles et groupes de base pour t'aider à découvrir le fonctionnement:

- rôles de realm: `platform-admin`, `manager`, `user`
- groupes: `admins`, `managers`, `employees`

### Option 1 - Gérer les droits par groupes

Dans Keycloak:

1. Ouvre `Groups`
2. Choisis un groupe existant ou crée un nouveau groupe
3. Va dans `Role mapping`
4. Assigne le rôle de realm souhaité

Exemple conseillé:

- groupe `admins` -> rôle `platform-admin`
- groupe `managers` -> rôle `manager`
- groupe `employees` -> rôle `user`

### Option 2 - Gérer les droits utilisateur par utilisateur

Dans Keycloak:

1. Ouvre `Users`
2. Choisis un utilisateur
3. Va dans `Role mapping`
4. Assigne directement les rôles de realm

Cette approche est utile pour découvrir, mais en pratique les groupes sont plus propres à maintenir.

## Etape 9 - Créer ou modifier un utilisateur de test

Tu peux utiliser l'utilisateur déjà importé:

- login: `owner@company.local`
- mot de passe initial: `ChangeMe123!`

Ou créer un nouvel utilisateur à la main:

1. `Users` -> `Add user`
2. Renseigne le `Username` et l'email
3. Sauvegarde
4. Va dans `Credentials` pour définir un mot de passe
5. Va dans `Groups` ou `Role mapping` pour lui donner les droits

## Etape 10 - Tester la connexion SSO

1. Ouvre `http://localhost:3000`
2. Clique sur `Sign in with Keycloak SSO`
3. Authentifie-toi sur Keycloak
4. Valide le retour vers Grafana

Après connexion, vérifie dans Grafana si le profil obtenu correspond bien au rôle attendu.

## Etape 11 - Dépannage rapide

Si le bouton SSO apparaît mais que la connexion échoue:

- vérifie que `KEYCLOAK_PUBLIC_URL` est accessible depuis ton navigateur
- vérifie que `KEYCLOAK_INTERNAL_URL` est résolu depuis le conteneur Grafana
- vérifie que le secret Grafana dans `.env` correspond exactement au secret du client Keycloak
- vérifie que `Valid redirect URIs` contient bien `http://localhost:3000/login/generic_oauth`
- vérifie que l'utilisateur est bien dans le realm `company`
- consulte les logs:

```bash
docker compose logs -f keycloak
docker compose logs -f grafana
```

## Etape 12 - Passage en production

Pour un environnement réel:

- remplace toutes les valeurs de démonstration
- publie Grafana et Keycloak derrière HTTPS
- remplace `localhost` par tes vrais noms DNS
- ajoute les URI de redirection et origines exactes dans Keycloak
- sauvegarde les volumes Docker

## Exemple production

Exemple de valeurs:

```env
KC_HOSTNAME=sso.example.com
KEYCLOAK_PUBLIC_URL=https://sso.example.com
KEYCLOAK_INTERNAL_URL=http://keycloak:8080
GRAFANA_ROOT_URL=https://grafana.example.com
```

Dans ce cas, mets à jour le client `grafana-oauth`:

- `Valid redirect URIs`: `https://grafana.example.com/login/generic_oauth`
- `Valid post logout redirect URIs`: `https://grafana.example.com`
- `Web origins`: `https://grafana.example.com`
- `Base URL`: `https://grafana.example.com`

## Résultat attendu

Une fois la configuration en place:

- l'utilisateur accède à Grafana sans compte local séparé
- l'authentification est centralisée dans Keycloak
- les rôles Grafana sont pilotés manuellement depuis Keycloak via les rôles ou les groupes du realm `company`
