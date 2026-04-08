# Intégration Grafana avec Keycloak

## Objet

Ce document décrit comment intégrer `Grafana` à `Keycloak` en `OpenID Connect` afin de mettre en place un SSO pour une application tierce.

L'objectif n'est pas de faire de Grafana une partie de la plateforme IAM, mais de montrer comment une application cliente se raccorde proprement à Keycloak.

## Principe

Dans un fonctionnement classique:

- Keycloak porte l'identité
- Grafana consomme cette identité
- les rôles et groupes sont gérés dans Keycloak
- Grafana applique ensuite ses propres droits en fonction des informations reçues

## Références du dépôt

- [Checklist d'administration Keycloak](../keycloak-admin-checklist.md)
- [Déploiement Grafana séparé](../../deployments/grafana/docker-compose.yml)
- [Variables d'exemple Grafana](../../deployments/grafana/.env.example)

## Préparation dans Keycloak

Avant de configurer Grafana, vérifier dans le realm cible:

- le realm existe
- les utilisateurs existent
- les rôles existent
- les groupes existent
- le client `grafana-oauth` est créé

Exemple de structure:

- rôles: `platform-admin`, `manager`, `user`
- groupes: `admins`, `managers`, `employees`

Mapping recommandé:

- `admins` -> `platform-admin`
- `managers` -> `manager`
- `employees` -> `user`

## Configuration du client Grafana dans Keycloak

Dans `Clients`:

1. créer un client `OpenID Connect`
2. définir `Client ID`: `grafana-oauth`
3. activer `Client authentication`
4. laisser `Standard flow` activé
5. désactiver les flows non nécessaires

Valeurs usuelles:

- `Root URL`: `http://localhost:3000`
- `Home URL`: `http://localhost:3000`
- `Valid redirect URIs`: `http://localhost:3000/login/generic_oauth`
- `Valid post logout redirect URIs`: `http://localhost:3000`
- `Web origins`: `http://localhost:3000`
- `Admin URL`: `http://localhost:3000`

Dans `Credentials`, récupérer ensuite le `Client secret`.

## Configuration de Grafana

La configuration du SSO Grafana peut être faite:

- directement dans l'interface Grafana
- ou, à titre indicatif, par variables d'environnement

L'approche la plus lisible pour comprendre le fonctionnement reste la configuration dans l'interface web Grafana.

## Paramétrage manuel dans Grafana

Dans Grafana:

1. ouvrir `Administration`
2. ouvrir `Authentication`
3. ouvrir `Generic OAuth`
4. activer `Generic OAuth`

Capture de référence:

![Configuration Generic OAuth dans Grafana](../images/grafana-generic-oauth-settings.png)

Renseigner les champs suivants:

`Display name`

- `Keycloak SSO`

`Client ID`

- `grafana-oauth`

`Client secret`

- valeur récupérée dans Keycloak

`Auth style`

- `AutoDetect`

`Scopes`

- `openid profile email offline_access roles`

`Auth URL`

- `http://localhost:8080/realms/Grafana/protocol/openid-connect/auth`

`Token URL`

- si Grafana tourne dans Docker: `http://host.docker.internal:8080/realms/Grafana/protocol/openid-connect/token`
- sinon: `http://localhost:8080/realms/Grafana/protocol/openid-connect/token`

`API URL`

- si Grafana tourne dans Docker: `http://host.docker.internal:8080/realms/Grafana/protocol/openid-connect/userinfo`
- sinon: `http://localhost:8080/realms/Grafana/protocol/openid-connect/userinfo`

`Allow sign up`

- `ON`

`Auto login`

- `OFF` au démarrage du projet

`Sign out redirect URL`

- `http://localhost:8080/realms/Grafana/protocol/openid-connect/logout?post_logout_redirect_uri=http://localhost:3000`

Dans `User mapping`, renseigner:

`Login field path`

- `preferred_username`

`Name field path`

- `name`

`Email field path`

- `email`

`Role attribute path`

```text
contains(realm_access.roles[*], 'platform-admin') && 'Admin' || contains(realm_access.roles[*], 'manager') && 'Editor' || 'Viewer'
```

Sauvegarder ensuite la configuration.

## Répartition des responsabilités

| Sujet | Keycloak | Grafana | Variable d'environnement |
| --- | --- | --- | --- |
| Realm | oui | non | indirectement via les URL |
| Client ID | oui | oui | possible |
| Client secret | oui | oui | possible |
| Redirect URI | oui | non | non |
| Web origins | oui | non | non |
| Auth URL | non | oui | possible |
| Token URL | non | oui | possible |
| API URL | non | oui | possible |
| Scopes | non | oui | possible |
| Display name SSO | non | oui | possible |
| Mapping login / nom / email | non | oui | possible |
| Mapping des rôles Grafana | non | oui | possible |
| Utilisateurs / groupes / rôles | oui | non | non |

Règle de lecture:

- Keycloak porte la configuration d'identité
- Grafana porte la configuration de consommation OAuth
- les variables d'environnement servent uniquement à automatiser ce paramétrage si nécessaire

## Variables d'environnement

Le déploiement fourni dans `deployments/grafana/` reste volontairement minimal.

Les variables conservées servent uniquement au fonctionnement local de Grafana:

```env
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=ChangeThisGrafanaAdminPassword!
GF_SERVER_ROOT_URL=http://localhost:3000
```

La configuration OAuth peut être injectée par variables si besoin, mais ce dépôt la présente avant tout dans l'interface Grafana.

## Validation

Après configuration:

1. ouvrir Grafana
2. cliquer sur `Sign in with Keycloak SSO`
3. s'authentifier dans Keycloak
4. vérifier le rôle obtenu dans Grafana

Capture de référence:

![Ecran de connexion Grafana](../images/grafana-login-screen.png)

## Mapping des rôles

Le mapping proposé est:

- `platform-admin` -> `Admin`
- `manager` -> `Editor`
- autre utilisateur authentifié -> `Viewer`

## Dépannage

Si Grafana redirige vers un mauvais realm:

- vérifier le nom exact du realm dans Grafana

Si Keycloak renvoie `Page not found`:

- vérifier que le realm existe
- vérifier la casse du nom du realm

Si `User sync failed` apparaît:

- vérifier que l'utilisateur existe bien dans le realm cible
- vérifier qu'il dispose d'un mot de passe
- vérifier son groupe ou ses rôles
- si nécessaire, repartir d'une instance Grafana neuve pour éliminer un état local incohérent

Si l'authentification réussit mais que les droits sont incorrects:

- vérifier le `Role mapping` dans Keycloak
- vérifier l'expression `Role attribute path` dans Grafana
