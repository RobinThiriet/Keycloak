# Mise en place pas à pas du SSO Keycloak pour Grafana

## Objectif

Construire une plateforme `Keycloak + Grafana` depuis zéro, puis mettre en place une authentification SSO OpenID Connect pour Grafana en configurant Keycloak entièrement à la main.

Ce guide couvre:

- le démarrage de la stack Docker
- la création du realm `company`
- la création des rôles, groupes et utilisateurs
- la création du client OpenID Connect `grafana-oauth`
- le test fonctionnel du SSO et des droits Grafana

## Prérequis

- Docker et Docker Compose installés
- Ports libres: `3000`, `8080`, `5432`, `9000`
- Accès au dépôt local

## Etape 1 - Préparer l'environnement

Copie le fichier d'exemple:

```bash
cp .env.example .env
```

Vérifie les variables importantes:

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

## Etape 3 - Vérifier l'accès aux services

Interfaces attendues:

- Keycloak: `http://localhost:8080`
- Admin Keycloak: `http://localhost:8080/admin`
- Grafana: `http://localhost:3000`
- Health Keycloak: `http://localhost:9000/health/ready`

## Etape 4 - Se connecter à l'administration Keycloak

Connecte-toi à `http://localhost:8080/admin` avec le compte bootstrap défini dans `.env`.

Au premier démarrage, seul le realm `master` doit exister.

## Etape 5 - Créer le realm `company`

Dans Keycloak:

1. Ouvre le sélecteur de realm en haut à gauche
2. Clique sur `Create realm`
3. Saisis `company`
4. Valide la création

Réglages recommandés pour le realm:

- `User registration`: `OFF`
- `Login with email`: `ON`
- `Duplicate emails`: `OFF`
- `Verify email`: `ON`
- `Forgot password`: `ON`
- `Remember me`: `ON`

Résultat attendu:

- le realm actif devient `company`

## Etape 6 - Créer les rôles de realm

Dans `Realm roles`, crée les rôles suivants:

- `platform-admin`
- `manager`
- `user`

Usage recommandé:

- `platform-admin`: administration Grafana
- `manager`: édition Grafana
- `user`: accès standard

## Etape 7 - Créer les groupes

Dans `Groups`, crée les groupes suivants:

- `admins`
- `managers`
- `employees`

Pour chaque groupe, ouvre `Role mapping` et attribue:

- `admins` -> `platform-admin`
- `managers` -> `manager`
- `employees` -> `user`

Résultat attendu:

- les droits fonctionnels sont gérés par groupes plutôt que manuellement utilisateur par utilisateur

## Etape 8 - Créer les utilisateurs de test

Dans `Users`, crée au minimum trois comptes:

- un compte admin
- un compte manager
- un compte standard

Exemple:

- `admin1@company.local`
- `manager1@company.local`
- `user1@company.local`

Pour chaque utilisateur:

1. crée le compte
2. définis un mot de passe dans `Credentials`
3. assigne le bon groupe dans `Groups`

Exemple de répartition:

- `admin1@company.local` -> `admins`
- `manager1@company.local` -> `managers`
- `user1@company.local` -> `employees`

## Etape 9 - Créer le client Grafana

Dans `Clients`:

1. clique sur `Create client`
2. choisis `OpenID Connect`
3. saisis `grafana-oauth`
4. passe à l'étape suivante

Réglages de capacité:

- `Client authentication`: `ON`
- `Authorization`: `OFF`
- `Standard flow`: `ON`
- `Direct access grants`: `OFF`
- `Implicit flow`: `OFF`
- `Service accounts roles`: `OFF`

Réglages d'URL:

- `Root URL`: `http://localhost:3000`
- `Home URL`: `http://localhost:3000`
- `Valid redirect URIs`: `http://localhost:3000/login/generic_oauth`
- `Valid post logout redirect URIs`: `http://localhost:3000`
- `Web origins`: `http://localhost:3000`

## Etape 10 - Récupérer le secret du client

Dans `Clients` -> `grafana-oauth` -> `Credentials`:

1. copie le `Client secret`
2. colle-le dans `.env` à la variable `GRAFANA_OAUTH_CLIENT_SECRET`

Si tu changes le `Client ID` dans Keycloak, reporte la même valeur dans `GRAFANA_OAUTH_CLIENT_ID`.

Applique ensuite la configuration à Grafana:

```bash
docker compose up -d grafana
```

## Etape 11 - Comprendre le mapping des droits Grafana

Grafana attribue ses droits à partir des rôles de realm Keycloak.

Mapping configuré dans ce projet:

- `platform-admin` -> `Admin`
- `manager` -> `Editor`
- tout autre utilisateur authentifié -> `Viewer`

Conséquence pratique:

- un utilisateur du groupe `admins` devient `Admin`
- un utilisateur du groupe `managers` devient `Editor`
- un utilisateur du groupe `employees` devient `Viewer`

## Etape 12 - Tester le SSO

1. ouvre `http://localhost:3000`
2. clique sur `Sign in with Keycloak SSO`
3. connecte-toi avec un utilisateur Keycloak
4. valide le retour vers Grafana

Répète le test avec plusieurs profils pour valider le mapping des droits.

## Etape 13 - Vérifier les rôles dans Grafana

Contrôles conseillés:

- le compte admin doit obtenir un rôle `Admin`
- le compte manager doit obtenir un rôle `Editor`
- le compte standard doit obtenir un rôle `Viewer`

Si le rôle ne correspond pas:

- vérifie le groupe de l'utilisateur dans Keycloak
- vérifie le `Role mapping` du groupe
- vérifie que Grafana a bien redémarré après la mise à jour du secret

## Etape 14 - Dépannage rapide

Si la connexion SSO échoue:

- vérifie que `KEYCLOAK_PUBLIC_URL` est accessible depuis le navigateur
- vérifie que `KEYCLOAK_INTERNAL_URL` est joignable depuis le conteneur Grafana
- vérifie que le secret du client est identique dans Keycloak et dans `.env`
- vérifie que `http://localhost:3000/login/generic_oauth` est présent dans `Valid redirect URIs`
- vérifie que `Client authentication` est bien activé
- vérifie que l'utilisateur testé appartient au realm `company`

Logs utiles:

```bash
docker compose logs -f keycloak
docker compose logs -f grafana
```

## Etape 15 - Passage en production

Pour un environnement réel:

- remplace toutes les valeurs de démonstration
- publie Keycloak et Grafana derrière HTTPS
- remplace `localhost` par tes noms DNS réels
- ajuste les `redirect URIs`, `post logout redirect URIs` et `web origins`
- sauvegarde les volumes Docker

## Exemple production

```env
KC_HOSTNAME=sso.example.com
KEYCLOAK_PUBLIC_URL=https://sso.example.com
KEYCLOAK_INTERNAL_URL=http://keycloak:8080
GRAFANA_ROOT_URL=https://grafana.example.com
```

Dans ce cas, configure le client `grafana-oauth` avec:

- `Valid redirect URIs`: `https://grafana.example.com/login/generic_oauth`
- `Valid post logout redirect URIs`: `https://grafana.example.com`
- `Web origins`: `https://grafana.example.com`
- `Base URL`: `https://grafana.example.com`

## Résultat attendu

À la fin du guide:

- Keycloak est déployé proprement
- le realm `company` est créé à la main
- Grafana est intégré comme application tierce
- le SSO fonctionne
- les droits Grafana sont pilotés depuis Keycloak
