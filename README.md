# API Users

API REST développée avec **Spring Boot** permettant de gérer les utilisateurs d'application (utilisé notamment pour https://github.com/Natsumi26/API_square-game et https://github.com/Natsumi26/TrailTrack-api .

Cette API est utilisée en complément d'autre API. Elle permet notamment de vérifier qu'un utilisateur existe avant de lui permettre d'accéder aux fonctionnalités.

---

## Technologies

- Java
- Spring Boot
- Spring Security
- JWT avec JJWT
- BCrypt
- Spring Data JPA
- MySQL
- Maven
- Swagger / OpenAPI

### Architecture du projet

```text
API Users
│
├── configuration
│   └── SecurityConfig
│
├── controllers
│   ├── AuthController
│   └── UserController
│
├── dao
│   ├── JpaUserDao
│   └── UserDao
│
├── dto
│   └── LoginRequest
│
├── models
│   ├── UserEntity
│   └── UserEntityRepository
│
├── filter
│   └── JwtAuthenticationFilter
│
├── models
│   └── User
│
├── services
│   ├── CustomUserDetails
│   ├── JwtService
│   ├── UserDetailsServiceImpl
│   ├── UserServiceImpl
│   └── UserService
│
│
└── resources
    ├── application.properties 
    └──application-local.properties (Dans le gitIgnore : concerne les données sensibles de clés JWT et bdd)
```

---


## ⚙️ Prérequis

Avant de lancer le projet, installer :

* **Java**
* **Maven**

---

## 🗄️ Base de données

L'application utilise une base de données **MySQL**.

La base de données est gérée sur l'API tiers et doit être lancée avant de lancer API users.


---

## 🔧 Configuration

L'API Users utilise le port **8081** afin de pouvoir fonctionner en parallèle de l'API tiers qui utilise le port **8080**.

Exemple de configuration :

```properties
server.port=8081
```
La clé secrète JWT est configurée dans `application-local.properties` :

```properties
jwt.secret=VOTRE_SECRET
```

Cette clé doit être **strictement identique** à celle configurée dans l'API tiers.

⚠️ Ne pas versionner une vraie clé secrète dans Git.

Les informations de connexion à la base de données sont définies dans `application-local.properties` :

```properties
spring.datasource.url=jdbc:mysql://localhost:<port>/<nom-database>
spring.datasource.username=utilisateur
spring.datasource.password=mot-de-passe

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

> ⚠️ Les mots de passe et informations sensibles ne doivent pas être publiés sur GitHub.

---

## ▶️ Lancer l'application

### 1. Démarrer MySQL (API tiers)

```bash
docker compose up -d
```

### 2. Lancer l'application API users

Avec Maven :

```bash
./mvnw spring-boot:run
```

Sous Windows, il est également possible d'utiliser :

```bash
mvnw.cmd spring-boot:run
```

L'application démarre alors sur :

```text
http://localhost:8081
```

---

## Gestion des utilisateurs

### Créer un utilisateur

```http
POST http://localhost:8081/users
Content-Type: application/json
```

Exemple :

```json
{
  "username": "Alice",
  "password": "motdepasse",
  "email": "alice@example.com",
  "role": "ROLE_USER"
}
```

Le mot de passe est enregistré sous forme de hash BCrypt.

Un administrateur peut utiliser :

```json
{
  "username": "Admin",
  "password": "motdepasse",
  "email": "admin@example.com",
  "role": "ROLE_ADMIN"
}
```

### Lister les utilisateurs

```http
GET http://localhost:8081/users
Authorization: Bearer <JWT>
```

L'accès est réservé au rôle `ROLE_ADMIN`.

### Récupérer un utilisateur

```http
GET http://localhost:8081/users/{id}
Authorization: Bearer <JWT>
```

Un utilisateur `ROLE_USER` peut consulter son propre profil.

Un utilisateur `ROLE_ADMIN` peut consulter les profils.

### Supprimer un utilisateur

```http
DELETE http://localhost:8081/users/{id}
Authorization: Bearer <JWT>
```

L'accès est réservé au rôle `ROLE_ADMIN`.

### Vérifier la validité d'un utilisateur

```http
GET http://localhost:8081/users/{id}/valid
```

Cet endpoint faisait partie de la communication initiale avec l'API Games. Dans la nouvelle architecture JWT, l'API Games n'effectue plus cet appel à chaque requête.

## Authentification JWT

### Connexion

```http
POST http://localhost:8081/auth/login
Content-Type: application/json
```

Exemple :

```json
{
  "username": "Alice",
  "password": "motdepasse"
}
```

Si les identifiants sont valides, l'API retourne un JWT.

Le payload contient notamment :

```json
{
  "sub": "Alice",
  "userId": "UUID_DE_L_UTILISATEUR",
  "role": "ROLE_USER"
}
```

Le JWT permet aux autres applications de vérifier l'identité et les droits de l'utilisateur.

## Fonctionnement de la sécurité

Spring Security est configuré en mode **stateless**.

Le JWT est envoyé dans le header :

```http
Authorization: Bearer <JWT>
```

Le `JwtAuthenticationFilter` :

1. récupère le header `Authorization` ;
2. vérifie la présence du préfixe `Bearer` ;
3. extrait le token ;
4. vérifie sa signature et sa validité ;
5. récupère les informations nécessaires ;
6. place l'authentification dans le `SecurityContext`.

## Rôles

Deux rôles sont utilisés :

```text
ROLE_USER
ROLE_ADMIN
```

Les restrictions sont appliquées avec `@PreAuthorize`.

Exemple :

```java
@PreAuthorize("hasRole('ADMIN')")
```

Pour l'accès à son propre profil :

```java
@PreAuthorize("hasRole('ADMIN') or (hasRole('USER') and #id == authentication.principal.id)")
```

## Communication avec l'API tiers

L'API Users fournit le JWT.

```text
              POST /auth/login
Client ─────────────────────────▶ API Users
Client ◀───────────────────────── JWT
       │
       │ Authorization: Bearer JWT
       ▼
   API Games
```

Les deux APIs utilisent la même clé secrète JWT afin que l'API tiers puisse vérifier localement les tokens générés par l'API Users.

L'API tiesr récupère directement le `userId` contenu dans le JWT.

Il n'est donc plus nécessaire de contacter l'API Users pour chaque requête tiers.

## Swagger / OpenAPI

Une fois l'application démarrée :

```text
http://localhost:8081/swagger-ui/index.html
```

Documentation OpenAPI :

```text
http://localhost:8081/v3/api-docs
```

## Persistance

Les utilisateurs sont persistés dans MySQL avec JPA.

La structure comprend notamment :

- `User`
- `UserEntity`
- `UserDao`
- `GameEntityRepository` / repository utilisateur selon la structure du projet

Le mot de passe est protégé avec BCrypt avant son enregistrement.
