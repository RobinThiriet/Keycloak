# Mise en place pas à pas du SSO Keycloak pour Grafana

## Objectif

Déployer `Grafana` avec Docker Compose et apprendre à tout configurer manuellement dans `Keycloak`:

- créer un realm
- créer les rôles
- créer les groupes
- créer les utilisateurs
- créer le client OpenID Connect Grafana
- tester le SSO et les droits

Après avoir suivi ce guide, tu dois pouvoir refaire l'intégration sans dépendre du fichier `realm/company-realm.json`.

## Ce que le dépôt contient déjà

Le dépôt contient un realm d'exemple dans [company-realm.json](/root/Keycloak/realm/company-realm.json), mais dans ce guide nous allons volontairement faire la configuration à la main dans l'interface Keycloak pour comprendre chaque étape.

Tu peux donc utiliser ce guide de deux manières:

- mode apprentissage: tu crées tout manuellement
- mode accéléré: tu t'inspires du realm JSON déjà fourni

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

Au premier accès, tu arrives en général sur le realm `master`.

## Etape 5 - Créer le realm manuellement

Dans Keycloak:

1. Ouvre le sélecteur de realm en haut à gauche
2. Clique sur `Create realm`
3. Renseigne `Realm name`: `company`
4. Clique sur `Create`

Tu peux ensuite ajuster les paramètres généraux du realm.

Réglages recommandés pour rester proche de l'exemple du dépôt:

- `User registration`: désactivé
- `Login with email`: activé
- `Duplicate emails`: désactivé
- `Verify email`: activé
- `Forgot password`: activé
- `Remember me`: activé

Contrôle attendu:

- le realm actif devient `company`

## Etape 6 - Créer les rôles du realm

Dans Keycloak:

1. Ouvre `Realm roles`
2. Clique sur `Create role`
3. Crée les rôles suivants:

- `platform-admin`
- `manager`
- `user`

Descriptions conseillées:

- `platform-admin`: administration de la plateforme
- `manager`: droits avancés sur les applications métier
- `user`: utilisateur standard

Pourquoi ces rôles:

- Grafana lira ces rôles pour attribuer ses propres droits

## Etape 7 - Créer les groupes

Dans Keycloak:

1. Ouvre `Groups`
2. Crée les groupes suivants:

- `admins`
- `managers`
- `employees`

Ensuite, pour chaque groupe:

1. Ouvre le groupe
2. Va dans `Role mapping`
3. Assigne le rôle correspondant

Mapping recommandé:

- groupe `admins` -> rôle `platform-admin`
- groupe `managers` -> rôle `manager`
- groupe `employees` -> rôle `user`

Pourquoi utiliser les groupes:

- c'est plus propre que d'affecter les rôles utilisateur par utilisateur
- ça facilite la maintenance quand tu ajoutes du monde

## Etape 8 - Créer un utilisateur de test

Dans Keycloak:

1. Ouvre `Users`
2. Clique sur `Add user`
3. Renseigne par exemple:

- `Username`: `owner@company.local`
- `Email`: `owner@company.local`
- `First name`: `Platform`
- `Last name`: `Owner`

4. Sauvegarde

Ensuite:

1. Va dans `Credentials`
2. Défini un mot de passe, par exemple `ChangeMe123!`
3. Choisis si le mot de passe est temporaire ou non

Puis donne les droits à l'utilisateur avec l'une des deux méthodes suivantes:

- méthode recommandée: onglet `Groups` puis ajout au groupe `admins`
- méthode découverte: onglet `Role mapping` puis ajout direct du rôle `platform-admin`

## Etape 9 - Créer le client Grafana manuellement

Dans l'admin Keycloak:

1. Ouvre `Clients`
2. Clique sur `Create client`
3. Renseigne `Client type`: `OpenID Connect`
4. Renseigne `Client ID`: `grafana-oauth`
5. Clique sur `Next`

Sur l'écran de capacité:

1. Active `Client authentication`
2. Laisse `Standard flow` activé
3. Désactive `Direct access grants`
4. Désactive `Implicit flow`
5. Désactive `Service accounts roles`
6. Clique sur `Next`

Sur l'écran de login:

1. `Root URL`: `http://localhost:3000`
2. `Home URL`: `http://localhost:3000`
3. `Valid redirect URIs`: `http://localhost:3000/login/generic_oauth`
4. `Valid post logout redirect URIs`: `http://localhost:3000`
5. `Web origins`: `http://localhost:3000`
6. Clique sur `Save`

## Etape 10 - Récupérer le secret du client

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

## Etape 11 - Comprendre le mapping des droits Grafana

Le rôle Grafana est déterminé à partir des rôles Keycloak par la configuration Docker de Grafana.

Le mapping actuellement prévu dans le dépôt est:

- `platform-admin` donne `Admin`
- `manager` donne `Editor`
- tout autre utilisateur authentifié devient `Viewer`

Concrètement:

- un utilisateur du groupe `admins` devient `Admin`
- un utilisateur du groupe `managers` devient `Editor`
- un utilisateur du groupe `employees` devient `Viewer`

## Etape 12 - Créer d'autres utilisateurs pour tester les droits

Crée au moins trois comptes de test pour bien comprendre:

- un compte admin dans `admins`
- un compte manager dans `managers`
- un compte standard dans `employees`

Exemple:

- `admin1@company.local` -> groupe `admins`
- `manager1@company.local` -> groupe `managers`
- `user1@company.local` -> groupe `employees`

## Etape 13 - Tester la connexion SSO

1. Ouvre `http://localhost:3000`
2. Clique sur `Sign in with Keycloak SSO`
3. Authentifie-toi sur Keycloak
4. Valide le retour vers Grafana

Après connexion, vérifie dans Grafana si le profil obtenu correspond bien au rôle attendu.

Fais le test avec plusieurs utilisateurs pour bien visualiser la différence entre `Admin`, `Editor` et `Viewer`.

## Etape 14 - Dépannage rapide

Si le bouton SSO apparaît mais que la connexion échoue:

- vérifie que `KEYCLOAK_PUBLIC_URL` est accessible depuis ton navigateur
- vérifie que `KEYCLOAK_INTERNAL_URL` est résolu depuis le conteneur Grafana
- vérifie que le secret Grafana dans `.env` correspond exactement au secret du client Keycloak
- vérifie que `Valid redirect URIs` contient bien `http://localhost:3000/login/generic_oauth`
- vérifie que le client a bien `Client authentication` activé
- vérifie que l'utilisateur testé est bien dans le realm `company`
- vérifie que l'utilisateur a bien un rôle ou un groupe cohérent avec ce que tu attends

Consulte les logs:

```bash
docker compose logs -f keycloak
docker compose logs -f grafana
```

## Etape 15 - Passage en production

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

- tu sais créer un realm Keycloak à la main
- tu sais créer des rôles, groupes et utilisateurs
- tu sais créer un client OIDC pour Grafana
- tu sais piloter les droits Grafana depuis Keycloak
- tu peux reproduire l'intégration sans dépendre du realm d'exemple du dépôt
