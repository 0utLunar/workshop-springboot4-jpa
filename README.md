# 🛠️ workshop-springboot4-jpa

Workshop project on **Spring Boot 4** + **Spring Data JPA** — a REST e-commerce API covering
`User`, `Product`, `Category`, `Order`, `OrderItem` and `Payment`, backed by **PostgreSQL** and
deployed on **Railway**.

_Projeto de workshop com **Spring Boot 4** + **Spring Data JPA** — uma API REST de e-commerce
com `User`, `Product`, `Category`, `Order`, `OrderItem` e `Payment`, usando **PostgreSQL** e
deploy no **Railway**._

[![Java](https://img.shields.io/badge/Java-25-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](https://adoptium.net/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![JPA](https://img.shields.io/badge/JPA-Hibernate-59666C?style=for-the-badge&logo=hibernate&logoColor=white)](https://hibernate.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Maven](https://img.shields.io/badge/Maven-C71A36?style=for-the-badge&logo=apachemaven&logoColor=white)](https://maven.apache.org/)
[![Railway](https://img.shields.io/badge/deploy-Railway-111111?style=for-the-badge&logo=railway&logoColor=white)](https://railway.app/)
![Status](https://img.shields.io/badge/status-educational-informational?style=for-the-badge)
![License](https://img.shields.io/badge/license-MIT-brightgreen?style=for-the-badge)

---

The goal of the project is the JPA object-relational mapping layer: every entity mapping,
association and persistence technique was written by hand. There is no security layer, no DTO
layer and no frontend — the API returns entities directly as JSON.

_O objetivo do projeto é a camada de mapeamento objeto-relacional do JPA: cada mapeamento de
entidade, associação e técnica de persistência foi escrito à mão. Não há camada de segurança,
nem camada de DTO, nem frontend — a API retorna as entidades diretamente como JSON._

> **Status:** educational / workshop code. See [Known limitations](#known-limitations)
> before exposing any deployment publicly.
>
> **Status:** código educacional / de workshop. Veja as
> [limitações conhecidas](#known-limitations) antes de expor qualquer deploy publicamente.

## 📦 Stack — _Tech Stack_

| Component | Version |
|---|---|
| Java | 25 |
| Spring Boot | 4.1.1 |
| Spring WebMVC | `spring-boot-starter-webmvc` |
| Spring Data JPA | `spring-boot-starter-data-jpa` (Hibernate) |
| PostgreSQL | 17 (local via Docker, production on Railway) |
| H2 | in-memory, `test` profile only |
| Build | Maven Wrapper 3.3.4 (Maven 3.9.16) |
| Deploy | Railway |

## 🗺️ Domain Model — _Modelo de Domínio_

The core of the workshop — the association strategies used and why:

_O núcleo do workshop — as estratégias de associação usadas e por quê:_

```
┌───────────┴──────────┐   tb_user
│User                  │
│id, name, email,      │
│phone, password       │
└───────────┬──────────┘
           │
         1 │ N
           │
┌───────────┴──────────┐        ┌──────────────────────┐
│Order                 │───1:1──├Payment               │
│tb_order              │ @MapsId├moment                │
│id, moment, status    │        ├order_id (PK)         │
└───────────┬──────────┘        └──────────────────────┘
           │
         1 │ N
           │
┌───────────┴───────────────────────────────────┐   tb_order_item
│OrderItem                                      │
│@EmbeddedId OrderItemPK (order_id,             │
│product_id), quantity, price                   │
│→ getSubTotal()                                │
└───────────┬───────────────────────────────────┘
           │
         N │ 1
           │
┌───────────┴─────────────────┐   tb_product
│Product                      │
│id, name, description,       │
│price, imgUrl                │
└───────────┬─────────────────┘
           │
         N │ M   (join table: tb_product_category)
           │
┌───────────┴─────────────────┐   tb_category
│Category                     │
│id, name                     │
└─────────────────────────────┘
```

| Association | Mapping | Notes |
|---|---|---|
| User → Order | `@OneToMany(mappedBy = "client")` | `orders` is `@JsonIgnore`d to avoid infinite recursion |
| Order → User | `@ManyToOne` + `@JoinColumn(name = "client_id")` | Owning side of the relation |
| Product ↔ Category | `@ManyToMany` + join table `tb_product_category` | `Product` is the owning side; `Category.products` is `@JsonIgnore`d |
| Order ↔ OrderItem | `@OneToMany` / `@ManyToOne` | `Order.items` is owning side (`mappedBy = "id.order"`) |
| OrderItem ↔ Product | `@ManyToOne` inside the composite key | The "many-to-many with extra attributes" (`quantity`, `price`) pattern |
| OrderItem (composite PK) | `@EmbeddedId OrderItemPK` | `OrderItemPK` = `@ManyToOne Order` + `@ManyToOne Product` |
| Order ↔ Payment | `@OneToOne` + `@MapsId` | Shared primary key: `tb_payment` has no `id` column — its PK is `order_id`, mirroring `tb_order.id` |
| Order status | Enum stored as `Integer` | `OrderStatus` with custom `valueOf(int)`; invalid codes throw `IllegalArgumentException` |

Computed values are derived, never persisted:

_Valores computados são derivados, nunca persistidos:_

- `OrderItem.getSubTotal()` = `price * quantity`
- `Order.getTotal()` = sum of the order's item subtotals

Entities implement `Serializable` and override `equals`/`hashCode` based on `id` only.

_As entidades implementam `Serializable` e sobrescrevem `equals`/`hashCode` baseados apenas no `id`._

## 🔌 API Endpoints — _Endpoints da API_

Base path: none — the resources are mapped at the root.

_Caminho base: nenhum — os resources são mapeados na raiz._

### Users (full CRUD)

| Method | Path | Response |
|---|---|---|
| `GET` | `/users` | `200` — `List<User>` |
| `GET` | `/users/{id}` | `200` — `User` · `404` if not found |
| `POST` | `/users` | `201` — created `User` + `Location` header |
| `PUT` | `/users/{id}` | `200` — updated `User` (`name`, `email`, `phone` only) · `404` if not found |
| `DELETE` | `/users/{id}` | `204` · `404` if not found · `400` on constraint violation |

### Products, Categories, Orders (read-only)

| Method | Path | Response |
|---|---|---|
| `GET` | `/products` | `200` — `List<Product>` |
| `GET` | `/products/{id}` | `200` — `Product` |
| `GET` | `/categories` | `200` — `List<Category>` |
| `GET` | `/categories/{id}` | `200` — `Category` |
| `GET` | `/orders` | `200` — `List<Order>` |
| `GET` | `/orders/{id}` | `200` — `Order` including `items` and computed `total` |

Example — `GET /orders/1`:

_Exemplo — `GET /orders/1`:_

```json
{
  "id": 1,
  "moment": "2019-06-20T19:53:07Z",
  "orderStatus": "PAID",
  "client": {
    "id": 1,
    "name": "Maria Brown",
    "email": "maria@gmail.com",
    "phone": "988888888",
    "password": "123456"
  },
  "items": [
    { "product": { "id": 1, "name": "The Lord of the Rings", "price": 90.5,
                   "imgUrl": "", "categories": [ { "id": 2, "name": "Books" } ] },
      "quantity": 2, "price": 90.5, "subTotal": 181.0 },
    { "product": { "id": 3, "name": "Macbook Pro", "price": 1250.0,
                   "imgUrl": "", "categories": [ { "id": 3, "name": "Computers" } ] },
      "quantity": 1, "price": 1250.0, "subTotal": 1250.0 }
  ],
  "total": 1431.0,
  "payment": { "id": 1, "moment": "2019-06-20T21:53:07Z" }
}
```

> `OrderItem` exposes no `id` in JSON: the entity has no `getId()`, so `product` (read from
> `OrderItemPK`) is serialized at the top level and `order` is dropped by `@JsonIgnore`.
>
> `password` is serialized in plain text — this is one of the
> [known limitations](#known-limitations).

## ⚠️ Error Handling — _Tratamento de Erros_

`ResourceExceptionHandler` (`@ControllerAdvice`) maps service exceptions to a `StandardError` body:

| Exception | Status | Message |
|---|---|---|
| `ResourceNotFoundException` | `404` | `Resource not found` |
| `DatabaseException` | `400` | `Database error` |

```json
{
  "timestamp": "2026-09-25T21:45:10Z",
  "status": 404,
  "error": "Resource not found",
  "message": "Resource not found with id 99",
  "path": "/users/99"
}
```

> `StandardError.timestamp` is formatted with `@JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'")`, so it
> has second precision and no milliseconds.

## 🚀 Running Locally — _Rodando Localmente_

### 1. Database (Docker)

`docker-compose.yml` provides PostgreSQL 17 and pgAdmin:

_O `docker-compose.yml` fornece PostgreSQL 17 e pgAdmin:_

```bash
docker compose up -d
```

| Service | URL | Credentials |
|---|---|---|
| PostgreSQL | `localhost:5432` | db `course`, user `postgres`, password `1234567` |
| pgAdmin | http://localhost:5050 | `admin@admin.com` / `admin` |

### 2. Application

`src/main/resources/application.properties` defaults to `spring.profiles.active=prod`, so pick a
profile explicitly:

_O `application.properties` usa `spring.profiles.active=prod` por padrão, então escolha um perfil
explicitamente:_

```bash
# dev → PostgreSQL, ddl-auto=update, SQL logging
export PGHOST=localhost PGPORT=5432 PGDATABASE=course
export POSTGRES_USER=postgres POSTGRES_PASSWORD=1234567
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# test → in-memory H2, seeded automatically, H2 console at /h2-console
./mvnw spring-boot:run -Dspring-boot.run.profiles=test
```

### Profiles

| Profile | Database | `ddl-auto` | SQL logging | Seed data |
|---|---|---|---|---|
| `dev` | PostgreSQL | `update` | on (formatted) | none |
| `prod` | PostgreSQL | `none` | off | none |
| `test` | H2 in-memory | not set (`create-drop`) | on (formatted) | yes — via `config/TestConfig` |

The `test` profile seeds 3 categories, 5 products, 2 users, 3 orders, 4 order items and 1
payment on startup.

_O perfil `test` popula 3 categorias, 5 produtos, 2 usuários, 3 pedidos, 4 itens de pedido e
1 pagamento na inicialização._

### Environment Variables

Read by the `dev` and `prod` profiles — the app will not start without them:

_Lidas pelos perfis `dev` e `prod` — a aplicação não sobe sem elas:_

| Variable | Description |
|---|---|
| `PGHOST` | PostgreSQL host |
| `PGPORT` | PostgreSQL port |
| `PGDATABASE` | Database name |
| `POSTGRES_USER` | Database user |
| `POSTGRES_PASSWORD` | Database password |
| `JWT_SECRET` | Unused — leftover config, see [known limitations](#known-limitations) |
| `JWT_EXPIRATION` | Unused — leftover config |

No `.env` file is committed. Keep local credentials in an untracked `.env` (it is gitignored) or
export the variables in your shell.

_Nenhum arquivo `.env` é versionado. Mantenha as credenciais locais em um `.env` não rastreado
(ele está no `.gitignore`) ou exporte as variáveis no seu shell._

## ☁️ Deploying to Railway (PostgreSQL)

This project is deployed on Railway. The steps used:

_Este projeto está no Railway. Os passos usados foram:_

1. Create a Railway project and add the **PostgreSQL** plugin.
2. Add the **Java** (Maven) service — Railway auto-detects the root `pom.xml` and `mvnw`. No
   `Dockerfile` or `Procfile` is required.
3. Link the PostgreSQL service to the app service, so Railway injects the connection variables.
4. Set the variables the Spring profiles expect — Railway's PostgreSQL plugin exposes
   `PGHOST`, `PGPORT`, `PGDATABASE`, `POSTGRES_USER` and `POSTGRES_PASSWORD`, which map 1:1 to the
   placeholders in `application-prod.properties`.
5. **Run `script.sql` against the database once.** `prod` uses `ddl-auto=none`, so Hibernate
   creates nothing — the schema comes from the checked-in `script.sql` (tables, identity
   sequences, primary keys and foreign keys).
6. Deploy. The default profile is already `prod`.

> `script.sql` contains DDL only, no `INSERT` statements — a fresh production database starts
> empty, so the read endpoints return empty lists until data is inserted.
>
> Não existe ferramenta de migração (Flyway/Liquibase) neste projeto: mudanças de schema são
> aplicadas reexecutando `script.sql` manualmente.

## 🧪 Tests — _Testes_

```bash
./mvnw test -Dspring.profiles.active=test
```

The suite is intentionally minimal — `CourseApplicationTests` only asserts that the Spring context
loads. There are no repository, service or endpoint tests yet.

_A suíte é intencionalmente mínima — `CourseApplicationTests` apenas verifica que o contexto do
Spring carrega. Ainda não há testes de repository, service ou endpoint._

## 📁 Project Structure — _Estrutura_

```
src/main/java/com/educandoweb/course/
├── CourseApplication.java          # @SpringBootApplication
├── config/TestConfig.java          # @Profile("test") data seeder (CommandLineRunner)
├── entities/                       # JPA entities
│   ├── enums/OrderStatus.java
│   └── pk/OrderItemPK.java         # @Embeddable composite key
├── repositories/                   # Spring Data JPA repositories
├── resources/                      # @RestController classes
│   └── exceptions/                 # @ControllerAdvice + StandardError
└── services/                       # service layer
    └── exceptions/

src/main/resources/
├── application.properties          # spring.profiles.active=prod
├── application-dev.properties
├── application-prod.properties
└── application-test.properties

script.sql                          # PostgreSQL DDL for prod (ddl-auto=none)
docker-compose.yml                  # PostgreSQL 17 + pgAdmin
```

Layering: `resources` → `services` → `repositories` → `entities`.

_Camadas: `resources` → `services` → `repositories` → `entities`._

## 💻 Useful Commands — _Comandos Úteis_

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev   # run locally
./mvnw clean package                                  # build the jar
./mvnw test -Dspring.profiles.active=test              # run tests
docker compose up -d                                  # start PostgreSQL + pgAdmin
docker compose down -v                                 # stop and drop the volume
```

<a id="known-limitations"></a>

## 🚨 Known Limitations — _Limitações Conhecidas_

This is workshop code, not production code. Deliberately out of scope, but worth knowing:

_Este é código de workshop, não de produção. Fora de escopo de propósito, mas vale saber:_

- **No authentication or authorization.** Every endpoint is public, including `POST /users`,
  `PUT /users/{id}` and `DELETE /users/{id}`. Do not publish the deployment URL — anyone who finds
  it can read and modify the database.
  _**Sem autenticação nem autorização.** Todos os endpoints são públicos, incluindo `POST /users`,
  `PUT /users/{id}` e `DELETE /users/{id}`. Não publique a URL do deploy — qualquer pessoa que a
  encontrar consegue ler e modificar o banco._
- **Passwords are stored and returned in plain text.** `User.password` has no encoder and is
  serialized in the JSON responses.
  _**Senhas são armazenadas e retornadas em texto puro.** `User.password` não tem encoder e é
  serializado nas respostas JSON._
- **The `jwt.*` properties are dead configuration.** `jwt.secret` and `jwt.expiration` exist in the
  `dev` and `prod` property files, but no JWT implementation was ever added.
  _**As propriedades `jwt.*` são configuração morta.** `jwt.secret` e `jwt.expiration` existem nos
  arquivos de properties `dev` e `prod`, mas nenhuma implementação JWT foi adicionada._
- **No migrations.** The schema lives in `script.sql` and is applied by hand.
  _**Sem migrações.** O schema fica no `script.sql` e é aplicado manualmente._
- **No DTO layer, no validation, no pagination, no OpenAPI/Swagger.** Entities are returned
  directly, and `spring.jpa.open-in-view` is enabled.
  _**Sem camada de DTO, sem validação, sem paginação, sem OpenAPI/Swagger.** As entidades são
  retornadas diretamente e `spring.jpa.open-in-view` está habilitado._
- **Inconsistent 404 handling.** `UserService` throws `ResourceNotFoundException`, but
  `CategoryService`, `ProductService` and `OrderService` call `Optional.get()` directly, which
  results in a `500` instead of a `404` for a missing id.
  _**Tratamento de 404 inconsistente.** `UserService` lança `ResourceNotFoundException`, mas
  `CategoryService`, `ProductService` e `OrderService` chamam `Optional.get()` direto, o que
  resulta em `500` em vez de `404` para um id inexistente._
- **Seed data only exists in the `test` profile.** `dev` and `prod` start with an empty database.
  _**Os dados de seed só existem no perfil `test`.** `dev` e `prod` sobem com o banco vazio._

Known code issues left in place:

_Problemas de código conhecidos mantidos no lugar:_

- `OrderItemRepository extends JpaRepository<OrderItem, Long>`, but the entity's id is the
  `OrderItemPK` composite key, not a `Long`.
  _O `OrderItemRepository` estende `JpaRepository<OrderItem, Long>`, mas o id da entidade é a
  chave composta `OrderItemPK`, não um `Long`._
- `org.postgresql:postgresql` is declared twice in `pom.xml` (once as `runtime`, once without a
  scope).
  _`org.postgresql:postgresql` está declarado duas vezes no `pom.xml` (uma como `runtime`, outra
  sem scope)._

## 📄 License — _Licença_

MIT — see [`LICENSE`](./LICENSE).

_MIT — veja [`LICENSE`](./LICENSE)._
