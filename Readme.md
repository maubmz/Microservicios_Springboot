# Microservicios con Spring Boot y OpenFeign

![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.5.13-brightgreen?logo=springboot&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.0-blue?logo=mysql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![OpenFeign](https://img.shields.io/badge/OpenFeign-Spring_Cloud_2025.0.0-6DB33F?logo=spring&logoColor=white)

Proyecto educativo compuesto por dos aplicaciones Spring Boot independientes. El servicio consumidor expone una API REST y delega las operaciones al servicio proveedor mediante **Spring Cloud OpenFeign**. El proveedor implementa la lógica de negocio y persiste clientes, productos y compras en **MySQL** mediante **Spring Data JPA**.

## Contenido

- [Arquitectura](#arquitectura)
- [Tecnologías](#tecnologías)
- [Estructura](#estructura)
- [Modelo de datos](#modelo-de-datos)
- [DTO](#dto)
- [Endpoints](#endpoints)
- [Primeros pasos](#primeros-pasos)
- [Pruebas](#pruebas)
- [Consideraciones de la versión actual](#consideraciones-de-la-versión-actual)

## Arquitectura

```text
Cliente HTTP
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

| Servicio | Puerto | Responsabilidad |
|---|---:|---|
| `ms-openfeign-client` | `8081` | Recibe las solicitudes externas y las delega al proveedor mediante OpenFeign. No accede a la base de datos. |
| `ms-products-api` | `8080` | Ejecuta la lógica de negocio y accede a MySQL mediante Spring Data JPA. |
| MySQL | `3306` | Almacena clientes, productos, compras y sus asociaciones. |

El consumidor obtiene la URL del proveedor desde su `application.yml`:

```yaml
microservicio:
  url: http://localhost:8080
```

## Tecnologías

| Tecnología | Versión | Propósito |
|---|---|---|
| Java | 21 | Lenguaje del backend |
| Spring Boot | 3.5.13 | Framework de aplicación |
| Spring Cloud | 2025.0.0 | Gestión de dependencias de OpenFeign |
| Spring Cloud OpenFeign | Administrada por Spring Cloud | Comunicación entre servicios |
| Spring Data JPA / Hibernate | Administrada por Spring Boot | Persistencia y ORM |
| MySQL | 8.0 | Base de datos relacional |
| Lombok | Administrada por Spring Boot | Reducción de código repetitivo |
| JUnit 5 y Mockito | Spring Boot Test | Pruebas unitarias |
| Docker Compose | Compose specification | Ejecución local de MySQL |

## Estructura

```text
microservicios_springboot/
├── ms-openfeign-client/              # Consumidor, puerto 8081
│   └── src/
│       ├── main/java/com/java/
│       │   ├── client/feign/         # Interfaces OpenFeign
│       │   ├── config/               # Configuración CORS
│       │   ├── controller/           # API pública del consumidor
│       │   ├── model/                # Modelos para serialización HTTP
│       │   └── service/              # Delegación hacia OpenFeign
│       └── test/java/com/java/service/
├── ms-products-api/                  # Proveedor, puerto 8080
│   └── src/
│       ├── main/java/com/bd/
│       │   ├── config/               # Configuración Jackson/Hibernate
│       │   ├── controller/           # API del proveedor
│       │   ├── dto/                  # ClientDTO y PurchaseDTO
│       │   ├── exception/            # Manejo global de excepciones
│       │   ├── model/                # Entidades JPA
│       │   ├── repository/           # Repositorios Spring Data JPA
│       │   └── service/              # Lógica de negocio y conversión DTO
│       └── test/java/com/bd/service/
├── docker-compose.yml                # Contenedor MySQL
├── mysql_query.sql                   # Creación de base de datos y tablas
├── .env.example                      # Plantilla de configuración de Docker
└── Readme.md
```

## Modelo de datos

```text
Client (1) ────────────── (N) Purchase
                                  |
                                  | @OneToMany + @JoinTable
                                  v
                               Product
```

- Un cliente puede tener muchas compras.
- Cada compra pertenece a un cliente.
- `Purchase.products` está mapeado como una asociación unidireccional `@OneToMany` mediante la tabla `purchase_products`.
- `product_order` conserva la posición de los productos y permite repetir un producto dentro de una compra.
- La cantidad de una compra se calcula con el número de productos seleccionados.
- Al crear una compra, el stock de cada producto disminuye una unidad.
- Al actualizar una compra, primero se restaura el stock anterior y después se aplica la nueva lista.
- Al eliminar una compra, se restaura el stock de sus productos.

### Tablas

| Tabla | Descripción |
|---|---|
| `clients` | Datos de clientes |
| `products` | Catálogo y stock de productos |
| `purchases` | Cliente y cantidad de cada compra |
| `purchase_products` | Productos incluidos y orden dentro de cada compra |

Los nombres de columnas en español se mapean a campos Java en inglés:

| Columna | Campo Java |
|---|---|
| `nombre` | `name` |
| `telefono` | `phoneNumber` |
| `precio` | `price` |
| `cantidad` | `quantity` |

## DTO

El proveedor utiliza DTO para Clientes y Compras. Productos continúa usando directamente la entidad `Product`.

### `ClientDTO`

| Campo | Tipo | Uso |
|---|---|---|
| `idClient` | `Integer` | Identificador devuelto por el proveedor |
| `name` | `String` | Entrada y salida |
| `email` | `String` | Entrada y salida |
| `phone` | `String` | Entrada y salida |
| `purchases` | `List<PurchaseDTO>` | Solo lectura mediante `@JsonProperty(READ_ONLY)` |

### `PurchaseDTO`

| Campo | Tipo | Uso actual |
|---|---|---|
| `idPurchase` | `Integer` | Se genera al guardar la compra |
| `idClient` | `Integer` | Identifica al cliente de la compra |
| `clientName` | `String` | Se obtiene de la entidad `Client` |
| `products` | `List<Product>` | En la entrada basta con proporcionar el ID de cada producto |
| `quantity` | `int` | Se calcula con `products.size()` |

`PurchaseDTO` no marca sus campos calculados como `READ_ONLY`; el servicio simplemente construye esos valores a partir de las entidades persistidas.

## Endpoints

El consumidor y el proveedor no exponen las mismas rutas. El consumidor usa rutas en inglés y delega a las rutas en español del proveedor.

### API del consumidor

Base URL: `http://localhost:8081`

#### Clientes

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/client/allClients` | Obtener todos los clientes |
| `GET` | `/client/{id}` | Obtener un cliente |
| `POST` | `/client/create` | Crear un cliente |
| `PUT` | `/client/update/{id}` | Actualizar un cliente |
| `DELETE` | `/client/delete/{id}` | Eliminar un cliente |

#### Productos

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/product/allProducts` | Obtener todos los productos |
| `GET` | `/product/{id}` | Obtener un producto |
| `POST` | `/product/create` | Crear un producto |
| `PUT` | `/product/update/{id}` | Actualizar un producto |
| `DELETE` | `/product/delete/{id}` | Eliminar un producto |

#### Compras

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/purchase/allPurchases` | Obtener todas las compras |
| `GET` | `/purchase/{id}` | Obtener una compra |
| `POST` | `/purchase/create` | Crear una compra |
| `PUT` | `/purchase/update/{id}` | Actualizar una compra |
| `DELETE` | `/purchase/delete/{id}` | Eliminar una compra |

### API del proveedor

Base URL: `http://localhost:8080`

| Recurso | GET todos | GET por ID | POST | PUT | DELETE |
|---|---|---|---|---|---|
| Clientes | `/api/clientes` | `/api/clientes/{id}` | `/api/clientes` | `/api/clientes/{id}` | `/api/clientes/{id}` |
| Productos | `/api/productos` | `/api/productos/{id}` | `/api/productos` | `/api/productos/{id}` | `/api/productos/{id}` |
| Compras | `/api/compras` | `/api/compras/{id}` | `/api/compras` | `/api/compras/{id}` | `/api/compras/{id}` |

### Ejemplo: crear un cliente mediante el consumidor

```bash
curl -X POST http://localhost:8081/client/create \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Juan Pérez",
    "email": "juan@ejemplo.com",
    "phone": "5551234567"
  }'
```

### Ejemplo: crear un producto mediante el consumidor

```bash
curl -X POST http://localhost:8081/product/create \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop",
    "price": 1500.00,
    "stock": 10
  }'
```

### Ejemplo: crear una compra mediante el consumidor

Primero deben existir el cliente y los productos referenciados:

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

El proveedor calcula `idPurchase`, `clientName` y `quantity`, y devuelve los productos con el stock actualizado.

## Primeros pasos

### Requisitos

- Git
- Java 21
- Maven 3.9 o posterior
- Docker y Docker Compose

### 1. Clonar el repositorio

```bash
git clone https://github.com/maubmz/microservicios_springboot.git
cd microservicios_springboot
```

### 2. Configurar MySQL

```bash
cp .env.example .env
```

Edita `.env` y define valores propios:

```env
MYSQL_ROOT_PASSWORD=una_contraseña_segura
MYSQL_DATABASE=client_product
CONTAINER_NAME=bd-openfeign
```

Inicia el contenedor:

```bash
docker compose up -d
```

El contenedor crea la base indicada por `MYSQL_DATABASE`, pero `docker-compose.yml` no monta automáticamente `mysql_query.sql`. Inicializa las tablas con:

```bash
docker compose exec -T mysql sh -c 'mysql --force -uroot -p"$MYSQL_ROOT_PASSWORD"' < mysql_query.sql
```

`--force` permite que el script continúe si la base `client_product` ya fue creada por el contenedor.

### 3. Ejecutar el proveedor

El archivo `.env` es leído por Docker Compose, pero no se carga automáticamente en un proceso Spring Boot iniciado desde la terminal. Exporta la misma contraseña:

```bash
export MYSQL_ROOT_PASSWORD='una_contraseña_segura'
cd ms-products-api
mvn spring-boot:run
```

El proveedor se ejecutará en `http://localhost:8080`.

También puedes configurar:

```bash
export DB_URL='jdbc:mysql://localhost:3306/client_product'
export DB_USER='root'
```

### 4. Ejecutar el consumidor

Desde otra terminal y a partir de la raíz del repositorio:

```bash
cd ms-openfeign-client
mvn spring-boot:run
```

El consumidor se ejecutará en `http://localhost:8081`.

## Pruebas

Una vez actualizadas las pruebas señaladas en la siguiente sección, podrán ejecutarse con Maven:

```bash
cd ms-products-api
mvn test

cd ../ms-openfeign-client
mvn test
```

Las clases bajo `src/test/java/.../service` usan JUnit 5 y Mockito; su objetivo es probar los servicios de manera aislada, sin iniciar MySQL ni el otro microservicio.

## Consideraciones de la versión actual

- Las pruebas unitarias no están sincronizadas con las clases de producción actuales:
  - Las pruebas de `ms-openfeign-client` usan `getId()` y `setId()` para `Client` y `Purchase`, pero los modelos actuales exponen `idClient` e `idPurchase`.
  - Las pruebas de `ms-products-api` invocan servicios con entidades `Client` y `Purchase`, mientras que los servicios actuales reciben y devuelven `ClientDTO` y `PurchaseDTO`.
  - Las pruebas del proveedor usan `PurchasesRepository`, mientras que `PurchaseService` utiliza `PurchaseRepository`.
- Existen dos repositorios casi duplicados: `PurchaseRepository` y `PurchasesRepository`. Conviene conservar solo el utilizado por la aplicación y actualizar las pruebas.
- El árbol proporcionado contiene `.mvn/wrapper/maven-wrapper.properties`, pero no muestra `mvnw` ni `mvnw.cmd`; por eso las instrucciones usan Maven instalado (`mvn`).
- `application-local.yml` y el valor predeterminado de `MYSQL_ROOT_PASSWORD` en `application.yml` contienen una contraseña. Elimina credenciales del repositorio y exige variables de entorno antes de publicar o desplegar.
- `mysql_query.sql` usa `CREATE DATABASE client_product` sin `IF NOT EXISTS`; por eso el comando de inicialización utiliza la opción `--force`.
- El nombre `deletePRoductById` en `ms-openfeign-client` contiene un error de mayúsculas. Funciona internamente porque controlador y servicio usan el mismo nombre, pero conviene renombrarlo a `deleteProductById`.

## Autor

**maubmz** — [GitHub](https://github.com/maubmz)
