# Keycloak Professionnel

Base propre et prête à l'emploi pour déployer Keycloak avec PostgreSQL, un realm préconfiguré, un thème de connexion personnalisé et une documentation claire.

## Ce que contient le projet

- Keycloak `26.5.3` construit via une image Docker dédiée
- PostgreSQL `16-alpine` avec persistance locale
- Import automatique d'un realm `company`
- Thème de connexion sobre et personnalisable
- Health checks et métriques activés côté Keycloak
- Variables d'environnement externalisées avec `.env.example`

## Structure

```text
.
├── Dockerfile
├── docker-compose.yml
├── realm/
│   └── company-realm.json
├── themes/
│   └── company/
│       └── login/
│           ├── resources/
│           │   └── css/
│           │       └── styles.css
│           └── theme.properties
└── README.md
```

## Démarrage rapide

1. Copier le fichier d'exemple:

```bash
cp .env.example .env
```

2. Ajuster les secrets dans `.env`.

3. Lancer la stack:

```bash
docker compose up -d --build
```

4. Accéder aux interfaces:

- Keycloak: `http://localhost:8080`
- Console d'administration: `http://localhost:8080/admin`
- Management / health: `http://localhost:9000/health/ready`

## Identifiants initiaux

- Admin Keycloak: définis dans `.env` via `KC_BOOTSTRAP_ADMIN_USERNAME` et `KC_BOOTSTRAP_ADMIN_PASSWORD`
- Utilisateur de démonstration importé:
  - Login: `owner@company.local`
  - Mot de passe initial: `ChangeMe123!`
  - Rôles: `platform-admin`, `realm-admin`

Pense à changer immédiatement ce mot de passe dans un vrai environnement.

## Realm importé

Le realm `company` est importé au premier démarrage avec:

- rôles métier `platform-admin`, `manager`, `user`
- groupes `admins`, `managers`, `employees`
- un client public `company-portal`
- un client confidentiel `company-api`
- durcissement de base sur les mots de passe et la détection brute-force
- thème de login `company`

## Commandes utiles

Arrêter la stack:

```bash
docker compose down
```

Réinitialiser complètement les données:

```bash
docker compose down -v
```

Voir les logs Keycloak:

```bash
docker compose logs -f keycloak
```

## Recommandations production

- Remplacer les secrets de démonstration
- Publier Keycloak derrière un reverse proxy TLS
- Sauvegarder le volume PostgreSQL
- Connecter un SMTP réel
- Brancher vos vrais clients OIDC/SAML et vos fournisseurs d'identité
