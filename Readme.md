# Microservices with Spring Boot and OpenFeign

![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.5.13-brightgreen?logo=springboot&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.0-blue?logo=mysql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![OpenFeign](https://img.shields.io/badge/OpenFeign-Spring_Cloud_2025.0.0-6DB33F?logo=spring&logoColor=white)

Educational project composed of two independent Spring Boot applications. The consumer service exposes a REST API and delegates operations to the provider through **Spring Cloud OpenFeign**. The provider implements the business logic and persists clients, products, and purchases in **MySQL** with **Spring Data JPA**.

## Contents

- [Architecture](#architecture)
- [Technology stack](#technology-stack)
- [Project structure](#project-structure)
- [Data model](#data-model)
- [DTOs](#dtos)
- [API reference](#api-reference)
- [Getting started](#getting-started)
- [Testing](#testing)
- [Current implementation notes](#current-implementation-notes)

## Architecture

```text
HTTP Client
    |
    v
ms-openfeign-client (:8081)
    |
    | OpenFeign
    v
ms-products-api (:8080)
    |
    | Spring Data JPA / JDBC
    v
MySQL 8 (:3306)
```

| Service | Port | Responsibility |
|---|---:|---|
| `ms-openfeign-client` | `8081` | Receives external requests and delegates them to the provider through OpenFeign. It does not access the database. |
| `ms-products-api` | `8080` | Executes business logic and accesses MySQL through Spring Data JPA. |
| MySQL | `3306` | Stores clients, products, purchases, and their associations. |

The consumer reads the provider URL from its `application.yml`:

```yaml
microservicio:
  url: http://localhost:8080
```

## Technology stack

| Technology | Version | Purpose |
|---|---|---|
| Java | 21 | Backend language |
| Spring Boot | 3.5.13 | Application framework |
| Spring Cloud | 2025.0.0 | OpenFeign dependency management |
| Spring Cloud OpenFeign | Managed by Spring Cloud | Service-to-service communication |
| Spring Data JPA / Hibernate | Managed by Spring Boot | Persistence and ORM |
| MySQL | 8.0 | Relational database |
| Lombok | Managed by Spring Boot | Boilerplate reduction |
| JUnit 5 and Mockito | Spring Boot Test | Unit testing |
| Docker Compose | Compose specification | Local MySQL infrastructure |

## Project structure

```text
microservicios_springboot/
├── ms-openfeign-client/              # Consumer — port 8081
│   └── src/
│       ├── main/java/com/java/
│       │   ├── client/feign/         # OpenFeign interfaces
│       │   ├── config/               # CORS configuration
│       │   ├── controller/           # Consumer REST API
│       │   ├── model/                # HTTP serialization models
│       │   └── service/              # OpenFeign delegation layer
│       └── test/java/com/java/service/
├── ms-products-api/                  # Provider — port 8080
│   └── src/
│       ├── main/java/com/bd/
│       │   ├── config/               # Jackson/Hibernate configuration
│       │   ├── controller/           # Provider REST API
│       │   ├── dto/                  # ClientDTO and PurchaseDTO
│       │   ├── exception/            # Global exception handling
│       │   ├── model/                # JPA entities
│       │   ├── repository/           # Spring Data JPA repositories
│       │   └── service/              # Business logic and DTO conversion
│       └── test/java/com/bd/service/
├── docker-compose.yml                # MySQL container
├── mysql_query.sql                   # Database and table creation script
├── .env.example                      # Docker environment template
└── Readme.md
```

## Data model

```text
Client (1) ────────────── (N) Purchase
                                  |
                                  | @OneToMany + @JoinTable
                                  v
                               Product
```

- One client can have many purchases.
- Each purchase belongs to one client.
- `Purchase.products` is mapped as a unidirectional `@OneToMany` association through `purchase_products`.
- The `product_order` column preserves product order and allows the same product to appear more than once in a purchase.
- Purchase quantity is calculated from the number of selected products.
- Creating a purchase decreases each selected product's stock by one.
- Updating a purchase restores the previous stock before applying the new product list.
- Deleting a purchase restores the stock of its products.

### Tables

| Table | Description |
|---|---|
| `clients` | Client data |
| `products` | Product catalog and stock |
| `purchases` | Client reference and purchase quantity |
| `purchase_products` | Products and their order within each purchase |

Spanish database column names are mapped to English Java fields:

| Database column | Java field |
|---|---|
| `nombre` | `name` |
| `telefono` | `phoneNumber` |
| `precio` | `price` |
| `cantidad` | `quantity` |

## DTOs

The provider uses DTOs for Clients and Purchases. Products continue to use the `Product` entity directly.

### `ClientDTO`

| Field | Type | Usage |
|---|---|---|
| `idClient` | `Integer` | Identifier returned by the provider |
| `name` | `String` | Input and output |
| `email` | `String` | Input and output |
| `phone` | `String` | Input and output |
| `purchases` | `List<PurchaseDTO>` | Read-only through `@JsonProperty(READ_ONLY)` |

### `PurchaseDTO`

| Field | Type | Current usage |
|---|---|---|
| `idPurchase` | `Integer` | Generated when the purchase is saved |
| `idClient` | `Integer` | Identifies the client making the purchase |
| `clientName` | `String` | Obtained from the `Client` entity |
| `products` | `List<Product>` | Only each product ID is required in requests |
| `quantity` | `int` | Calculated with `products.size()` |

`PurchaseDTO` does not mark its calculated fields as `READ_ONLY`; the service builds those values from persisted entities instead.

## API reference

The consumer and provider do not expose the same paths. The consumer uses English paths and delegates to the provider's Spanish paths.

### Consumer API

Base URL: `http://localhost:8081`

#### Clients

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/client/allClients` | Get all clients |
| `GET` | `/client/{id}` | Get a client |
| `POST` | `/client/create` | Create a client |
| `PUT` | `/client/update/{id}` | Update a client |
| `DELETE` | `/client/delete/{id}` | Delete a client |

#### Products

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/product/allProducts` | Get all products |
| `GET` | `/product/{id}` | Get a product |
| `POST` | `/product/create` | Create a product |
| `PUT` | `/product/update/{id}` | Update a product |
| `DELETE` | `/product/delete/{id}` | Delete a product |

#### Purchases

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/purchase/allPurchases` | Get all purchases |
| `GET` | `/purchase/{id}` | Get a purchase |
| `POST` | `/purchase/create` | Create a purchase |
| `PUT` | `/purchase/update/{id}` | Update a purchase |
| `DELETE` | `/purchase/delete/{id}` | Delete a purchase |

### Provider API

Base URL: `http://localhost:8080`

| Resource | Get all | Get by ID | Create | Update | Delete |
|---|---|---|---|---|---|
| Clients | `GET /api/clientes` | `GET /api/clientes/{id}` | `POST /api/clientes` | `PUT /api/clientes/{id}` | `DELETE /api/clientes/{id}` |
| Products | `GET /api/productos` | `GET /api/productos/{id}` | `POST /api/productos` | `PUT /api/productos/{id}` | `DELETE /api/productos/{id}` |
| Purchases | `GET /api/compras` | `GET /api/compras/{id}` | `POST /api/compras` | `PUT /api/compras/{id}` | `DELETE /api/compras/{id}` |

### Create a client through the consumer

```bash
curl -X POST http://localhost:8081/client/create \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "5551234567"
  }'
```

### Create a product through the consumer

```bash
curl -X POST http://localhost:8081/product/create \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop",
    "price": 1500.00,
    "stock": 10
  }'
```

### Create a purchase through the consumer

The referenced client and products must already exist:

```bash
curl -X POST http://localhost:8081/purchase/create \
  -H "Content-Type: application/json" \
  -d '{
    "idClient": 1,
    "products": [
      { "id": 1 }
    ]
  }'
```

The provider calculates `idPurchase`, `clientName`, and `quantity`, and returns the products with their updated stock.

## Getting started

### Prerequisites

- Git
- Java 21
- Maven 3.9 or newer
- Docker and Docker Compose

### 1. Clone the repository

```bash
git clone https://github.com/maubmz/microservicios_springboot.git
cd microservicios_springboot
```

### 2. Configure MySQL

```bash
cp .env.example .env
```

Edit `.env` and provide your own values:

```env
MYSQL_ROOT_PASSWORD=a_secure_password
MYSQL_DATABASE=client_product
CONTAINER_NAME=bd-openfeign
```

Start the container:

```bash
docker compose up -d
```

The container creates the database configured by `MYSQL_DATABASE`, but `docker-compose.yml` does **not** automatically mount or execute `mysql_query.sql`. Initialize the tables with:

```bash
docker compose exec -T mysql sh -c 'mysql --force -uroot -p"$MYSQL_ROOT_PASSWORD"' < mysql_query.sql
```

The `--force` option allows the script to continue if `client_product` was already created by the container.

### 3. Start the provider

Docker Compose reads `.env`, but a separately started Spring Boot process does not load it automatically. Export the same password before starting the provider:

```bash
export MYSQL_ROOT_PASSWORD='a_secure_password'
cd ms-products-api
mvn spring-boot:run
```

The provider will run at `http://localhost:8080`.

Optional datasource variables:

```bash
export DB_URL='jdbc:mysql://localhost:3306/client_product'
export DB_USER='root'
```

### 4. Start the consumer

From another terminal opened at the repository root:

```bash
cd ms-openfeign-client
mvn spring-boot:run
```

The consumer will run at `http://localhost:8081`.

## Testing

After updating the tests described in the next section, run them with Maven:

```bash
cd ms-products-api
mvn test

cd ../ms-openfeign-client
mvn test
```

The classes under `src/test/java/.../service` use JUnit 5 and Mockito. Their purpose is to test the service layer in isolation without starting MySQL or the other microservice.

## Current implementation notes

- The unit tests are not synchronized with the current production classes:
  - Tests in `ms-openfeign-client` use `getId()` and `setId()` for `Client` and `Purchase`, but the current models expose `idClient` and `idPurchase`.
  - Tests in `ms-products-api` call services with `Client` and `Purchase` entities, while the current services receive and return `ClientDTO` and `PurchaseDTO`.
  - Provider tests use `PurchasesRepository`, while `PurchaseService` uses `PurchaseRepository`.
- The provider contains two nearly identical repositories: `PurchaseRepository` and `PurchasesRepository`. Keep the one used by the application and update the tests accordingly.
- The provided tree contains `.mvn/wrapper/maven-wrapper.properties`, but it does not show `mvnw` or `mvnw.cmd`; the instructions therefore use the installed `mvn` command.
- `application-local.yml` and the default value of `MYSQL_ROOT_PASSWORD` in `application.yml` contain a database password. Remove committed credentials and require environment variables before publishing or deploying the project.
- `mysql_query.sql` uses `CREATE DATABASE client_product` without `IF NOT EXISTS`, so the initialization command uses `--force`.
- The method name `deletePRoductById` in `ms-openfeign-client` contains a capitalization typo. It works internally because the controller and service use the same name, but it should be renamed to `deleteProductById`.

## Author

**maubmz** — [GitHub](https://github.com/maubmz)
