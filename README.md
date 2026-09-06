# DonaTrack

Sistema de gestión y trazabilidad de donaciones para una ONG.
Trabajo Práctico Integrador de **Diseño de Sistemas de Información** — UTN FRBA, 2026.

Una donación entra al depósito, el sistema la segmenta por subcategoría, unos algoritmos
de matchmaking la cruzan contra las necesidades de las entidades beneficiarias, una
administradora confirma el destino, se planifica una ruta de camión, la entidad confirma
la recepción y se notifica a todos los involucrados. En paralelo corre un sistema de
ludificación con misiones, insignias y ranking mensual.

## Arquitectura

Arquitectura distribuida en servicios. Cada uno es un **proceso independiente**, con su
propio `main`, su puerto y su base de datos: no comparten objetos ni tablas, solo se
hablan por red.

| Servicio | Puerto | Responsabilidad | Entrega |
|---|---|---|---|
| `server/donaciones` | 8081 | Donantes, donaciones, segmentación, entidades beneficiarias, necesidades, matchmaking | 1 |
| `server/notificaciones` | 8082 | Envío por correo, SMS y WhatsApp | 1 |
| `server/incentivos` | 8083 | Analítica, misiones, insignias, ranking mensual | 2 |
| `server/logistica` | 8084 | Flota, rutas, entregas, tracking en tiempo real | 3 |
| `server/auth` | 8085 | Tokens entre el front y los servicios | 5 |
| `front` | 8080 | Cliente SSR que consume las APIs | 5 |

> **Estado actual: Entrega 1.** Los cuatro primeros módulos compilan y levantan, pero
> todavía son andamiaje: solo tienen su clase `@SpringBootApplication`. `incentivos` y
> `logistica` están creados a modo de ejemplo; se implementan en las Entregas 2 y 3.

## Stack

Java 21 (LTS) · Spring Boot 4.0.8 · Maven multimódulo · PostgreSQL + MongoDB (Entrega 4)
· RabbitMQ (Entrega 3) · Thymeleaf + Leaflet (Entrega 5) · Docker Compose (Entrega 4).

## Requisitos

- **JDK 21.** En IntelliJ: `File → Project Structure → SDKs → + → Download JDK` (Temurin
  o Corretto).
- **Maven**: no hace falta instalarlo, IntelliJ trae uno embebido. Si preferís tenerlo en
  la terminal, `sudo dnf install maven` (o el equivalente de tu distro).
- **Git**, con `user.name` y `user.email` configurados.

Si usás el Maven embebido de IntelliJ y querés invocarlo desde la terminal:

```bash
export JAVA_HOME=/app/jbr
export PATH="/app/jbr/bin:/app/plugins/maven-plugin/lib/maven3/bin:$PATH"
```

## Cómo levantarlo

```bash
git clone git@github.com:tinogera/DonaTrack.git
cd DonaTrack
cp .env.example .env      # completar con las credenciales propias
mvn clean install         # tiene que quedar verde antes de seguir
```

Cada servicio se levanta por separado, en su propia terminal:

```bash
mvn -pl server/donaciones spring-boot:run      # http://localhost:8081
mvn -pl server/notificaciones spring-boot:run  # http://localhost:8082
```

O ejecutando el jar ya empaquetado:

```bash
java -jar server/donaciones/target/donaciones-1.0.0-SNAPSHOT.jar
```

### Tests

```bash
mvn test                                   # todos los módulos
mvn -pl server/donaciones test             # un módulo
mvn -pl server/donaciones test -Dtest=SegmentacionTest                       # una clase
mvn -pl server/donaciones test -Dtest=SegmentacionTest#separaPorSubcategoria # un test
```

## Estructura del repositorio

```
donatrack/
├── pom.xml           # POM padre — packaging: pom
├── .env.example      # nombres de variables, sin valores
├── Docs/             # enunciado, decisiones, diagramas y justificaciones
├── server/
│   ├── pom.xml       # agregador — packaging: pom
│   └── <servicio>/   # un módulo Maven por servicio
└── front/            # Entrega 5
```

Dentro de cada servicio, el paquete raíz es el nombre del servicio
(`server/donaciones/src/main/java/donaciones/`) y adentro va el esqueleto habitual:
`domain/`, `service/`, `repository/`, `controller/`, `dto/` y `config/`. Las carpetas se
crean cuando tienen contenido.

## Documentación

Todo el detalle está en `Docs/`:

- **`Docs/donatrack-tp.md`** — enunciado de cátedra completo, decisiones de arquitectura
  del grupo y los checkpoints de las seis entregas. Es la fuente de verdad del alcance.
- **`Docs/donatrack-dependencias.md`** — qué dependencia se agrega en qué entrega y en qué
  módulo, y el setup de herramientas y servicios externos.

Los diagramas van en `Docs/diagramas/` como `.puml` (PlantUML), y las justificaciones de
diseño en `Docs/justificaciones/entrega-N.md`.

## Cómo trabajamos

Git Flow liviano:

- `main` — solo lo que se entrega, con un tag por entrega (`entrega-1`, `entrega-2`, …).
- `develop` — integración.
- `feature/<descripcion>` — una rama por tarea.

El merge a `develop` es **siempre por Pull Request**, revisado por otro integrante.

Un par de convenciones que conviene conocer antes de escribir código:

- **Nada de Lombok en el dominio.** Las clases de dominio tienen comportamiento propio, no
  treinta accesores generados: es justamente la pregunta de discusión de la Entrega 1.
- **Las dependencias se agregan cuando se usan**, no por adelantado.
- **El `.env` real nunca se commitea.** Desde la Entrega 2 hay credenciales de correo, SMS
  y WhatsApp en juego.
