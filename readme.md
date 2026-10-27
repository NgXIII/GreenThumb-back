# GreenThumb — Backend

API REST + WebSocket du site **GreenThumb**, une plateforme pour suivre l'entretien de ses plantes, lire des fiches/ressources de jardinage et échanger sur un forum communautaire.

## Stack

- Java 21, Spring Boot 3.5.6, Gradle
- Spring Modulith (monolithe modulaire, événements entre modules)
- PostgreSQL + Liquibase (migrations et seeders)
- Redis (sessions / tokens)
- Spring Security + JWT (hash des mots de passe en Argon2)
- WebSocket STOMP (forum en temps réel, notifications)
- Services externes : Cloudinary (images), Trefle (données plantes), Gemini (modération IA du forum), SMTP Gmail (mails)

## Modules

Code dans `src/main/java/com/GreenThumb/api/` :

| Module | Rôle |
|---|---|
| `apigateway` | Contrôleurs REST, DTO, sécurité (filtre JWT, CORS), validation |
| `user` | Comptes, rôles, vérification e-mail, reset de mot de passe, paliers (tiers) |
| `plant` | Plantes de l'utilisateur, API Trefle, tâches d'entretien et planification |
| `tracking` | Génération des tâches selon la saison à la création d'une plante |
| `resources` | Articles/ressources, catégories, likes |
| `forum` | Channels, threads, messages, réactions, signalements, modération (Gemini) |
| `notification` | Notifications in-app (WebSocket) et e-mails |
| `infrastructure` | Config Redis, WebSocket, Cloudinary, tokens |

L'architecture modulaire est vérifiée par `src/test/java/com/GreenThumb/api/ArchTest.java`.

## Prérequis

- JDK 21
- PostgreSQL (une base `GreenThumb`)
- Redis
- Comptes/clés pour Cloudinary, Trefle, Gemini et un compte Gmail (mot de passe d'application)

## Configuration

Les valeurs sont lues depuis des variables d'environnement :

| Variable | Usage | Défaut (dev) |
|---|---|---|
| `DB_HOST`, `DB_PORT`, `DB_NAME` | PostgreSQL | `localhost`, `5432`, `GreenThumb` |
| `DB_USERNAME`, `DB_PASSWORD` | PostgreSQL | `postgres`, `rootroot` |
| `REDIS_HOST`, `REDIS_PORT`, `REDIS_USERNAME`, `REDIS_PASSWORD`, `REDIS_SSL` | Redis | — (`REDIS_SSL=false`) |
| `MAIL_USERNAME`, `MAIL_PASSWORD` | Expéditeur principal | — |
| `NOREPLY_MAIL_USERNAME`, `NOREPLY_MAIL_PASSWORD` | Expéditeur no-reply | — |
| `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` | Stockage des images | — |
| `TREFLE_API_KEY` | Données botaniques | — |
| `GOOGLE_API_KEY` | Modération Gemini | — |
| `FRONTEND_URL` | Liens dans les e-mails | `http://localhost:5173` |

Profils Spring : `dev` (par défaut avec `bootRun`), `staging`, `prod`, `test`. Voir `src/main/resources/application-*.properties`. Les origines CORS autorisées sont définies par profil (`greenthumb.cors.allowed-origins`).

## Lancer le projet

1. Cloner le dépôt
2. Créer la base `GreenThumb` dans PostgreSQL et démarrer Redis
3. Définir les variables d'environnement ci-dessus
4. Lancer :

```bash
./gradlew bootRun
```

Au démarrage, Liquibase applique les migrations et les seeders (`src/main/resources/db/changelog/`). L'API écoute sur `http://localhost:8080`.

Autres profils :

```bash
./gradlew bootRunStanging
```

```bash
./gradlew bootRunProd
```

### Installer PostgreSQL (Windows)

1. Télécharger PostgreSQL : https://www.postgresql.org/download/ (version 18)
2. Fermer la fenêtre Stack Builder proposée à la fin de l'installation
3. Ouvrir pgAdmin 4 et créer une base `GreenThumb`
4. (Optionnel) Ajouter la datasource dans IntelliJ pour y accéder depuis l'IDE

## Tests

```bash
./gradlew test
```

Les tests tournent sur SQLite (profil `test`). Un rapport de couverture JaCoCo est généré dans `build/reports/jacoco/`.

## API

Préfixe `/api`. Principales routes :

| Route | Contenu |
|---|---|
| `/api/users`, `/api/sessions` | Inscription, connexion, refresh, vérification e-mail, profil |
| `/api/codes` | Codes de réinitialisation de mot de passe |
| `/api/plants`, `/api/tasks` | Plantes et tâches d'entretien |
| `/api/resources`, `/api/resourceCategories` | Articles et catégories |
| `/api/channels`, `/api/threads`, `/api/messages` | Forum |
| `/api/reports` | Signalements de messages |
| `/api/notifications` | Notifications |
| `/api/tiers` | Paliers utilisateur |
| `/api/admin/users`, `/api/admin/moderation` | Administration et modération |

WebSocket STOMP sur `/ws` (broker `/topic` et `/queue`) : `/forum/send`, `/forum/reaction`, `/notifications/subscribe`.

Healthcheck : `GET /actuator/health`.

## Docker

```bash
docker build -t greenthumb-back .
```

```bash
docker run -p 8080:8080 --env-file .env -e SPRING_PROFILES_ACTIVE=prod greenthumb-back
```

## CI/CD

GitLab CI (`.gitlab-ci.yml`) : lance les tests sur chaque push, puis sur `main` construit l'image Docker et la pousse dans le registry GitLab.
