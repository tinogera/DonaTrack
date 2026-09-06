# DonaTrack — Dependencias y Setup

Documento de referencia técnica. Qué se instala, dónde, y en qué entrega hace falta.

> **Principio general**: una dependencia se agrega **cuando se usa**, no antes. Meterlas
> por adelantado hace que el proyecto tarde más en arrancar, agranda el árbol de
> dependencias y hace que alguien las use sin querer antes de tiempo.

---

## Índice

1. [Herramientas de sistema](#1-herramientas-de-sistema)
2. [Estructura de los POM](#2-estructura-de-los-pom)
3. [Dependencias por entrega](#3-dependencias-por-entrega)
4. [Dependencias por módulo](#4-dependencias-por-módulo)
5. [Servicios externos y credenciales](#5-servicios-externos-y-credenciales)
6. [Plugins de IntelliJ](#6-plugins-de-intellij)
7. [Verificación del setup](#7-verificación-del-setup)

---

## 1. Herramientas de sistema

Se instalan en la máquina, no en el proyecto.

| Herramienta | Versión | Cuándo | Cómo instalarla |
|---|---|---|---|
| **JDK** | 21 (LTS) | Ya | IntelliJ: `File → Project Structure → SDKs → + → Download JDK`. Elegir Temurin o Corretto |
| **IntelliJ IDEA** | Última | Ya | La edición Community alcanza; con mail de la UTN se puede pedir la Ultimate gratis |
| **Git** | Cualquiera reciente | Ya | git-scm.com. Configurar `user.name` y `user.email` |
| **Maven** | — | Ya | **No hace falta instalarlo**: IntelliJ trae uno embebido |
| **Docker Desktop** | Última | Entrega 4 | docker.com. Conviene instalarlo ya para no perder tiempo después |
| **n8n** | Última | Entrega 2 | Vía Docker, o cuenta en n8n.cloud (tiene capa gratuita) |
| **PlantUML** | — | Entrega 1 | No se instala: es el plugin de IntelliJ (ver sección 6) |

**Sobre la versión de Spring Boot.** La rama actual es la **4.0.x** (salió en noviembre
de 2025, sobre Spring Framework 7). Aviso honesto: casi todos los tutoriales y respuestas
de StackOverflow están escritos para **3.x**. Si el grupo va a googlear mucho, 3.5.x da
menos fricción. Para saber el patch exacto vigente: generar un proyecto suelto en
start.spring.io y copiar el número que ponga.

---

## 2. Estructura de los POM

Tres niveles: padre → agregador → módulos.

### 2.1 POM padre (`/pom.xml`)

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.x</version>   <!-- patch exacto de start.spring.io -->
    <relativePath/>
  </parent>

  <groupId>ar.edu.utn.frba.dds.donatrack</groupId>
  <artifactId>donatrack</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>pom</packaging>

  <modules>
    <module>server</module>
  </modules>

  <properties>
    <java.version>21</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <opencsv.version>5.9</opencsv.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.opencsv</groupId>
        <artifactId>opencsv</artifactId>
        <version>${opencsv.version}</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

Dos cosas importantes:

- `<packaging>pom</packaging>` significa "esto no genera un jar, solo agrupa".
- `<dependencyManagement>` fija la versión **sin activar** la dependencia. Los módulos la
  declaran sin `<version>` y heredan el número. Así nadie termina con dos versiones
  distintas de la misma librería.

### 2.2 Agregador (`/server/pom.xml`)

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>ar.edu.utn.frba.dds.donatrack</groupId>
    <artifactId>donatrack</artifactId>
    <version>1.0.0-SNAPSHOT</version>
  </parent>

  <artifactId>server</artifactId>
  <packaging>pom</packaging>

  <modules>
    <module>donaciones</module>
    <module>notificaciones</module>
  </modules>
</project>
```

> ⚠️ Un módulo declarado en `<modules>` que no existe **hace fallar el build**. Se agregan
> a medida que se crean.

### 2.3 Módulo (`/server/donaciones/pom.xml`)

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>ar.edu.utn.frba.dds.donatrack</groupId>
    <artifactId>server</artifactId>
    <version>1.0.0-SNAPSHOT</version>
  </parent>

  <artifactId>donaciones</artifactId>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
      <groupId>com.opencsv</groupId>
      <artifactId>opencsv</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

> ⚠️ El `spring-boot-maven-plugin` va en **cada módulo booteable**, nunca en el POM padre.
> Es el error más común en Maven multimódulo.

---

## 3. Dependencias por entrega

### Entrega 1 — Modelado y dominio

| Artifact | Grupo | Para qué |
|---|---|---|
| `spring-boot-starter-web` | `org.springframework.boot` | El endpoint `GET` requerido, y toda la API REST posterior |
| `spring-boot-starter-validation` | `org.springframework.boot` | Validar mail obligatorio, cantidades positivas, etc. |
| `opencsv` | `com.opencsv` | Importación masiva de 20.000 filas |
| `spring-boot-starter-test` | `org.springframework.boot` | JUnit 5 + AssertJ + Mockito |

**Sobre Lombok: no se usa en el dominio.** `@Data` y `@Getter/@Setter` empujan al modelo
anémico, que es justamente lo que la pregunta de discusión de esta entrega pone en debate.
Las clases de dominio deben tener comportamiento, no accesores generados. En DTOs se puede
evaluar más adelante.

### Entrega 2 — REST, matchmaking, incentivos, notificaciones reales

| Artifact | Grupo | Para qué |
|---|---|---|
| `spring-boot-starter-mail` | `org.springframework.boot` | Envío real de correo por SMTP |
| `commons-text` | `org.apache.commons` | Trae Levenshtein implementado; también se puede escribir a mano |
| SDK del proveedor de SMS/WhatsApp | según proveedor | Twilio, por ejemplo |

El scheduling (`@Scheduled`) para los algoritmos y la tarea de inactividad **ya viene** en
Spring Boot: se activa con `@EnableScheduling`, sin dependencia extra.

### Entrega 3 — Logística, cola de mensajes

| Artifact | Grupo | Para qué |
|---|---|---|
| `spring-boot-starter-amqp` | `org.springframework.boot` | RabbitMQ, para la integración asincrónica con Notificaciones |

El cliente HTTP hacia el planificador externo (`RestClient`) viene en `starter-web`.
Los Server-Sent Events del dashboard también: son parte de Spring MVC.

### Entrega 4 — Persistencia

| Artifact | Grupo | Para qué |
|---|---|---|
| `spring-boot-starter-data-jpa` | `org.springframework.boot` | El **ORM** exigido, en todos los servicios menos Logística |
| `postgresql` | `org.postgresql` | Driver, con `<scope>runtime</scope>` |
| `spring-boot-starter-data-mongodb` | `org.springframework.boot` | El **ODM** exigido, solo en Logística |
| `flyway-core` | `org.flywaydb` | Migraciones versionadas del esquema |
| `flyway-database-postgresql` | `org.flywaydb` | Soporte de Flyway para Postgres |

> ⚠️ **No usar `ddl-auto: update`** en un proyecto grupal. Sin migraciones versionadas, la
> base de cada integrante termina distinta y nadie sabe cuál es la correcta.

### Entrega 5 — Front SSR y autenticación

| Artifact | Grupo | Para qué |
|---|---|---|
| `spring-boot-starter-thymeleaf` | `org.springframework.boot` | Motor de plantillas (SSR) |
| `spring-boot-starter-security` | `org.springframework.boot` | Autenticación y autorización |
| `jjwt-api` / `jjwt-impl` / `jjwt-jackson` | `io.jsonwebtoken` | Emisión y validación de tokens JWT |
| `thymeleaf-extras-springsecurity6` | `org.thymeleaf.extras` | Mostrar/ocultar en las vistas según rol |

Recursos del navegador (no son dependencias Maven, van en `static/`):

| Recurso | Para qué |
|---|---|
| **Leaflet** | Mapa interactivo, sobre OpenStreetMap. Sin API key ni costo |
| **HTMX** *(opcional)* | Filtros y formularios sin recargar la página, sin necesidad de una SPA |

> **CSS**: escribir uno propio con variables (`--color-primario`, `--espaciado-md`), que es
> literalmente lo que el enunciado llama *design tokens*. Si se usa Bootstrap, la
> accesibilidad y el responsive se heredan en vez de demostrarse, que es lo que se evalúa.

### Entrega 6 — Despliegue y observabilidad

| Artifact | Grupo | Para qué |
|---|---|---|
| `spring-boot-starter-actuator` | `org.springframework.boot` | Health checks y métricas |
| `micrometer-registry-prometheus` | `io.micrometer` | Exportar métricas a Prometheus |
| `micrometer-tracing-bridge-otel` | `io.micrometer` | Trazas distribuidas |
| `bucket4j-core` | `com.bucket4j` | Rate limiting |
| `spring-boot-starter-graphql` | `org.springframework.boot` | Solo si se elige la Opción 2 |
| `grpc-spring-boot-starter` | según implementación | Solo si se elige la Opción 1 |

---

## 4. Dependencias por módulo

Vista cruzada: qué lleva cada servicio al final del TP.

| Dependencia | donaciones | notificaciones | incentivos | logistica | auth | front |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `starter-web` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `starter-validation` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `starter-test` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `opencsv` | ✅ | — | — | — | — | — |
| `commons-text` | ✅ | — | — | — | — | — |
| `starter-mail` | — | ✅ | — | — | — | — |
| SDK SMS/WhatsApp | — | ✅ | — | — | — | — |
| `starter-amqp` | ✅ | ✅ | ✅ | ✅ | — | — |
| `starter-data-jpa` | ✅ | ✅ | ✅ | — | ✅ | — |
| `postgresql` | ✅ | ✅ | ✅ | — | ✅ | — |
| `flyway-core` | ✅ | ✅ | ✅ | — | ✅ | — |
| `starter-data-mongodb` | — | — | — | ✅ | — | — |
| `starter-thymeleaf` | — | — | — | — | — | ✅ |
| `starter-security` | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| `jjwt-*` | ✅ | — | ✅ | ✅ | ✅ | ✅ |
| `starter-actuator` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `micrometer-*` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

Fijate que **Logística no lleva JPA ni Postgres**, y es el único con Mongo. Eso es la
persistencia políglota hecha efectiva.

---

## 5. Servicios externos y credenciales

Todo lo de esta sección va en el `.env` real, **nunca** en el repositorio.

| Servicio | Para qué | Entrega | Costo |
|---|---|---|---|
| **SMTP** (Gmail o Mailtrap) | Correos reales. Gmail requiere contraseña de aplicación; Mailtrap intercepta los envíos en un buzón de prueba | 2 | Gratis |
| **Twilio** | SMS y WhatsApp. Tiene sandbox de WhatsApp para pruebas | 2 | Capa gratuita |
| **n8n** | Flujo de difusión de insignias | 2 | Gratis (self-hosted o cloud) |
| **Generación de imágenes** | Imagen de la insignia dentro del flujo n8n | 2 | Depende del proveedor |
| **Red social** | Publicación automática del logro | 2 | Gratis, requiere app registrada |
| **Planificador de rutas** | Provisto por la cátedra | 3 | — |
| **Proveedor cloud** | Despliegue público | 6 | Buscar capa gratuita |

### `.env.example`

Este sí se commitea: documenta qué variables hacen falta, sin valores.

```dotenv
# Base de datos relacional
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=donatrack
POSTGRES_USER=
POSTGRES_PASSWORD=

# Base de datos documental
MONGO_URI=

# Cola de mensajes
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USER=
RABBITMQ_PASSWORD=

# Correo
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASSWORD=

# SMS y WhatsApp
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=

# Planificador externo de rutas
PLANIFICADOR_URL=
PLANIFICADOR_CALLBACK_URL=

# JWT
JWT_SECRET=
JWT_EXPIRATION_MS=

# n8n
N8N_WEBHOOK_URL=
```

---

## 6. Plugins de IntelliJ

| Plugin | Para qué | Cuándo |
|---|---|---|
| **PlantUML Integration** | Previsualizar los `.puml` de `Docs/diagramas/` | Entrega 1 |
| **Graphviz** | Requerido por PlantUML para algunos tipos de diagrama | Entrega 1 |
| **Database Tools** | Explorar Postgres desde el IDE (viene en Ultimate) | Entrega 4 |
| **Docker** | Gestionar contenedores desde el IDE | Entrega 4 |
| **.env files support** | Resaltado de sintaxis en `.env` | Ya |
| **SonarLint** *(opcional)* | Detecta code smells mientras escribís | Ya |

---

## 7. Verificación del setup

Desde la raíz del proyecto:

```bash
mvn clean install
```

Tiene que compilar todos los POM sin errores. En la Entrega 1 todavía puede no haber
ninguna clase Java, y está bien: primero se verifica que el andamiaje esté sano.

Para levantar un servicio en particular:

```bash
mvn -pl server/donaciones spring-boot:run
```

Y para verificar el endpoint requerido por el enunciado:

```bash
curl http://localhost:8081/
# Hola desde el servicio de Donaciones.
```

### Checklist de que todo está bien

- [ ] `mvn clean install` verde desde la raíz
- [ ] Todos los integrantes clonaron y les compila
- [ ] IntelliJ reconoce los módulos como proyectos Maven
- [ ] El JDK del proyecto es 21 en todos los módulos
- [ ] El `.env` real **no** aparece en `git status`
- [ ] Cada servicio levanta en su puerto asignado sin conflictos
