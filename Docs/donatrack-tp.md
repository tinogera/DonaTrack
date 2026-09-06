# DonaTrack — Sistema de Gestión y Trazabilidad de Donaciones

**Trabajo Práctico Integrador — Diseño de Sistemas de Información**
UTN FRBA — 2026

> ⚠️ **Nota sobre fechas**: el documento de cátedra fue escrito cuando la materia era
> anual. Ahora es cuatrimestral y las fechas del enunciado original **no aplican**.
> Confirmar el cronograma real con el profesor y completar la tabla de la sección 4.

---

## Índice

1. [Introducción](#1-introducción)
2. [Arquitectura](#2-arquitectura)
   - [2.1 Principios de la arquitectura](#21-principios-de-la-arquitectura)
   - [2.2 Los servicios](#22-los-servicios)
   - [2.3 Cómo se comunican los servicios](#23-cómo-se-comunican-los-servicios)
   - [2.4 Persistencia políglota](#24-persistencia-políglota)
   - [2.5 Stack tecnológico](#25-stack-tecnológico)
   - [2.6 Estructura del repositorio](#26-estructura-del-repositorio)
   - [2.7 Estructura interna de un servicio](#27-estructura-interna-de-un-servicio)
   - [2.8 Metodología de trabajo con Git](#28-metodología-de-trabajo-con-git)
3. [Enunciado completo](#3-enunciado-completo)
   - [3.1 Contexto general](#31-contexto-general)
   - [3.2 Requerimientos mínimos de UI/UX](#32-requerimientos-mínimos-de-uiux)
   - [3.3 Entrega 1 — Arquitectura y Modelado en Objetos I](#33-entrega-1--arquitectura-y-modelado-en-objetos-i)
   - [3.4 Entrega 2 — Arquitectura y Modelado en Objetos II](#34-entrega-2--arquitectura-y-modelado-en-objetos-ii)
   - [3.5 Entrega 3 — Arquitectura y Modelado en Objetos III](#35-entrega-3--arquitectura-y-modelado-en-objetos-iii)
   - [3.6 Entrega 4 — Persistencia y Maquetado](#36-entrega-4--persistencia-y-maquetado)
   - [3.7 Entrega 5 — Arquitectura Web MVC](#37-entrega-5--arquitectura-web-mvc)
   - [3.8 Entrega 6 — Despliegue, Observabilidad y Seguridad](#38-entrega-6--despliegue-observabilidad-y-seguridad)
4. [Entregables y checkpoints](#4-entregables-y-checkpoints)
   - [4.0 Checkpoint 0 — Setup inicial](#40-checkpoint-0--setup-inicial)
   - [4.1 Entregable 1](#41-entregable-1)
   - [4.2 Entregable 2](#42-entregable-2)
   - [4.3 Entregable 3](#43-entregable-3)
   - [4.4 Entregable 4](#44-entregable-4)
   - [4.5 Entregable 5](#45-entregable-5)
   - [4.6 Entregable 6](#46-entregable-6)
5. [Decisiones pendientes](#5-decisiones-pendientes)
6. [Anexo — Links de interés](#6-anexo--links-de-interés)

---

## 1. Introducción

Una ONG dedicada a recolectar y distribuir donaciones de bienes materiales no tiene
un sistema unificado. Eso produce desorden, registros duplicados y poca claridad
sobre el destino de lo donado, lo que perjudica la transparencia y la eficiencia.

**DonaTrack** es la solución propuesta: una plataforma que organiza, registra y
monitorea las donaciones desde que entran al depósito hasta que se entregan a una
entidad beneficiaria.

El flujo central del sistema es:

```
Donante lleva bienes al depósito
        ↓
Se registra la donación y el sistema la segmenta por subcategoría
        ↓
Algoritmos de matchmaking la cruzan contra necesidades de entidades beneficiarias
        ↓
Un administrador confirma el destino final
        ↓
Se planifica una ruta de camión
        ↓
El camión entrega y la entidad confirma la recepción
        ↓
Se notifica a todos los involucrados y se actualizan los incentivos del donante
```

En paralelo corre un sistema de ludificación: misiones, insignias, categorías de
donante y un ranking mensual, para incentivar la donación sostenida.

---

## 2. Arquitectura

### 2.1 Principios de la arquitectura

Estas son las reglas que ordenan todo el resto. Si algo del diseño las contradice,
está mal el diseño.

**1. Cada servicio es un proceso independiente.**
Tiene su propio `main`, su propio puerto, su propio archivo de configuración y se
despliega por separado. No es un paquete dentro de una aplicación más grande.

**2. Cada servicio es dueño exclusivo de su base de datos.**
Ningún servicio consulta la base de otro. Ni siquiera tiene la conexión. La base es
un detalle interno: si fuera compartida, cambiar una columna rompería a otro
servicio sin enterarse.

**3. Los servicios se comunican solo por red.**
Nunca por objetos compartidos ni por base de datos común. Solo HTTP o mensajes.

**4. Las referencias entre servicios son por ID, no por objeto.**
En un monolito escribiríamos `donacion.getDonante().getNombre()`. Acá la clase
`Donacion` del Servicio de Logística tiene un `Long donanteId`, y si necesita el
nombre lo pide por HTTP.

**5. La duplicación controlada de datos es aceptable y esperada.**
El Servicio de Incentivos guarda su propia copia mínima de la actividad de donación
(`donanteId`, `fecha`, `cantidad`, `subcategoria`) porque la necesita para calcular
rachas. No es un error de normalización: es el precio de la independencia.

**6. Cada servicio tiene su propio modelo de dominio.**
Por eso el enunciado pide **un diagrama de clases por servicio**. Dos servicios
pueden tener una clase `Donacion` y ser cosas distintas, porque cada uno la mira
desde su responsabilidad.

### 2.2 Los servicios

| Servicio | Responsabilidad | Aparece en |
|---|---|---|
| **Donaciones** | Donantes (humanas/jurídicas), donaciones, segmentación, entidades beneficiarias, necesidades, estados, matchmaking | Entrega 1 |
| **Notificaciones** | Envío por correo, SMS y WhatsApp | Entrega 1 (simulado) / 2 (real) |
| **Incentivos** | Analítica de donantes, misiones, insignias, categorías, ranking mensual | Entrega 2 |
| **Logística** | Flota de camiones, planificación de rutas, entregas, tracking en tiempo real | Entrega 3 |
| **Autenticación** | Gestión de tokens entre el front y los servicios de aplicación | Entrega 5 |
| **Front (SSR)** | Cliente liviano desacoplado que consume las APIs | Entrega 5 |

Puertos asignados (definidos ahora para que nadie se pise):

| Servicio | Puerto |
|---|---|
| donaciones | 8081 |
| notificaciones | 8082 |
| incentivos | 8083 |
| logistica | 8084 |
| auth | 8085 |
| front | 8080 |

### 2.3 Cómo se comunican los servicios

Hay dos mecanismos, y la elección no es de gusto: depende de si podés esperar o no.

**Sincrónico — HTTP contra la API REST.**
Pregunto y espero la respuesta. Ejemplo: Logística necesita la dirección de una
entidad beneficiaria para armar la ruta, entonces hace `GET /entidades/42` contra el
Servicio de Donaciones. Si Donaciones está caído, Logística no puede avanzar. Hay
acoplamiento en el tiempo.

**Asincrónico — cola de mensajes.**
Aviso y sigo con lo mío. Donaciones publica "la donación 17 fue asignada" y sigue
respondiendo pedidos. Quien esté suscripto lo procesa cuando pueda. Si el consumidor
está caído, el mensaje espera en la cola.

**Lo que fija el enunciado:**

- La integración con el **Servicio de Notificaciones debe ser asincrónica**, vía cola
  de mensajes, para no afectar la disponibilidad ante picos de carga o fallas
  transitorias. Registrar una donación no puede fallar porque el proveedor de SMS
  está lento.
- Entre servicios de dominio **puede** ser sincrónico por API, y la **direccionalidad
  la define cada equipo**, priorizando bajo acoplamiento y trazabilidad. Es una
  decisión nuestra que hay que justificar por escrito.

### 2.4 Persistencia políglota

Como cada servicio es dueño exclusivo de su base, cada uno puede elegir el motor que
mejor le sirva sin coordinar con nadie.

**Lo que exige el enunciado:**

- Servicio de **Logística** → base de datos **documental**, con **ODM**.
- Los **demás servicios** → base de datos **relacional**, con **ORM**.
- Para lo relacional se puede **compartir un mismo motor** a fines de simplicidad,
  pero con **un esquema separado por servicio**.
- Hay que contemplar **desnormalizaciones** donde sirvan para optimizar lecturas.

**Por qué documental en Logística.**
Una ruta es un camión con una lista *ordenada* de paradas, y en cada parada un
conjunto de entregas. En relacional eso son tres o cuatro tablas y varios joins para
reconstruir una sola ruta. En documental es un único documento que se lee de una vez,
con todo anidado. Y siempre se consulta completa: nunca vas a querer "todas las
paradas sueltas sin su ruta". Sumale la posición GPS, que llega a alta frecuencia y
con estructura que puede variar según la fuente elegida.

**Por qué relacional en el resto.**
Donantes, donaciones, entidades y necesidades se cruzan permanentemente entre sí, y
el matchmaking hace consultas relacionales puras. La integridad referencial acá vale.

> La pregunta de discusión de la Entrega 4 es precisamente si esta decisión está bien.
> La cátedra pide cuestionarla, no aceptarla. Conviene llegar con opinión formada.

### 2.5 Stack tecnológico

La cátedra no impone nada. Estas son nuestras elecciones y hay que poder defenderlas.

| Elemento | Elección | Motivo |
|---|---|---|
| Lenguaje | **Java 21 (LTS)** | Rodaje y soporte de herramientas. `sealed` y pattern matching sirven para los estados de donación |
| Build | **Maven multimódulo** | Decisión del grupo; POM padre + agregador + un módulo por servicio |
| Framework | **Spring Boot 4.0.x** | Cubre las seis entregas sin cambiar de stack |
| ORM | **JPA / Hibernate** (Spring Data JPA) | Entrega 4 |
| ODM | **Spring Data MongoDB** | Entrega 4 |
| Relacional | **PostgreSQL** (un motor, esquema por servicio) | Entrega 4 |
| Documental | **MongoDB** | Entrega 4 |
| Cola | **RabbitMQ** (Spring AMQP) | Entrega 3 |
| Template engine | **Thymeleaf** | Entrega 5 — templates que son HTML válido |
| Mapa | **Leaflet + OpenStreetMap** | Sin API key ni costo |
| Tiempo real | **Server-Sent Events** | El flujo es unidireccional; encaja mejor que WebSockets |
| Low-code | **n8n** | Entrega 2, difusión de insignias |
| Contenedores | **Docker + Docker Compose** | Desde Entrega 4 |

**Decisión sobre Lombok: no se usa en el dominio.**
`@Data` y `@Getter/@Setter` empujan directo al modelo anémico, que es exactamente lo
que la pregunta de discusión de la Entrega 1 pone en debate. Las clases de dominio
tienen que tener comportamiento propio, no treinta accesores generados. En DTOs se
puede evaluar.

### 2.6 Estructura del repositorio

Monorepo con módulos Maven.

```
donatrack/
├── pom.xml                      # POM padre — packaging: pom
├── docker-compose.yml           # desde Entrega 4
├── .gitignore
├── .env.example                 # nombres de variables, sin valores reales
├── README.md
│
├── Docs/
│   ├── diagramas/               # .puml versionables
│   └── justificaciones/         # un documento por entrega
│
├── server/
│   ├── pom.xml                  # agregador — packaging: pom
│   ├── donaciones/              # Entrega 1
│   ├── notificaciones/          # Entrega 1
│   ├── incentivos/              # Entrega 2
│   ├── logistica/               # Entrega 3
│   └── auth/                    # Entrega 5
│
└── front/                       # Entrega 5 — Spring Boot + Thymeleaf
```

**Los módulos se crean cuando se usan.** Un módulo declarado en `<modules>` que no
existe hace fallar el build. Los que dicen "Entrega 2" en adelante todavía no van.

**Sin módulo `shared` por ahora.** Es la tentación obvia y la que más acopla. Cuando
llegue la integración de la Entrega 2/3 va a hacer falta compartir *contratos* (DTOs),
no dominio, y ahí se decide si va un módulo `contracts` o si cada servicio define su
propia vista del otro.

**Diagramas en PlantUML (`.puml`), no en binarios.** Se versionan como texto, se
diffean y se mergean. Cuando la Entrega 3 pida el diagrama de despliegue
"actualizado", se editan cuatro líneas en vez de rehacerlo.

### 2.7 Estructura interna de un servicio

Todos los servicios comparten el mismo esqueleto. Solo cambia el nombre del paquete
raíz, que es el del servicio.

```
server/donaciones/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/donaciones/
    │   │   ├── DonacionesApplication.java
    │   │   ├── domain/          # el modelo, con comportamiento
    │   │   ├── service/         # casos de uso / orquestación
    │   │   ├── repository/      # interfaces de acceso a datos
    │   │   ├── controller/      # endpoints REST
    │   │   ├── dto/             # lo que entra y sale por HTTP
    │   │   └── config/
    │   └── resources/
    │       └── application.yml
    └── test/java/donaciones/
```

Notificaciones es idéntico pero su paquete raíz es `notificaciones`. Eso hace que dos
clases con el mismo nombre en servicios distintos no choquen, y deja explícito que
son modelos diferentes.

**Las carpetas se crean cuando tienen contenido.** Son el esqueleto por defecto, no
una obligación. Notificaciones en la Entrega 1 probablemente arranque solo con
`domain/`, `service/` y `dto/`.

**`repository/` es la interfaz, no la implementación.** En la Entrega 1 la
implementación es en memoria. En la Entrega 4 entra JPA (o Mongo en Logística) y, si
la interfaz está bien definida, ni el service ni el controller se enteran del cambio.
Esto es material directo para el documento de justificaciones.

**Equivalencias, si venís de Node/Express:**

| Express | Spring |
|---|---|
| `routes/` | No existe: la ruta se declara con `@RequestMapping` en el propio controller |
| `schemas/` (Zod) | Bean Validation: `@NotBlank`, `@Email`, `@Positive` sobre los DTOs |
| `middlewares/` de error | `@RestControllerAdvice` |
| `middlewares/` transversales | `Filter` / `HandlerInterceptor` |
| `service/` con axios | `RestClient` en el front consumiendo las APIs |

### 2.8 Metodología de trabajo con Git

Git Flow liviano:

- `main` → solo lo que se entrega, con un tag por entrega.
- `develop` → integración.
- `feature/<descripcion>` → una por tarea.

Merge a `develop` **siempre por Pull Request**, revisado por otro integrante. En
grupo, esto es lo que evita que se rompa todo la noche antes.

**El `.gitignore` tiene que cubrir:**

```gitignore
target/
.idea/
*.iml
*.iws
out/
.env
*.log
.DS_Store
```

> ⚠️ El `.env` real **nunca** se commitea. Desde la Entrega 2 se manejan credenciales
> de correo, SMS y WhatsApp. Un token filtrado en un repo público es un problema serio.

**Reparto sugerido del trabajo (3 personas activas).** En la Entrega 1 casi todo el
peso está en Donaciones, así que conviene cortar por **caso de uso vertical**, no por
servicio: uno agarra donantes + importación CSV, otro donaciones + segmentación, otro
entidades + necesidades + notificador. Todos tocan el mismo módulo pero en paquetes
distintos, y nadie queda bloqueado. Desde la Entrega 3, cuando aparece Logística,
conviene separar por servicio.

La cuarta persona, aunque no programe: los diagramas y el documento de justificaciones
son entregables obligatorios en **todas** las entregas y son lo que más se descuida.

---

## 3. Enunciado completo

> Transcripción del documento de cátedra. Las fechas corresponden a la versión anual
> y no aplican al cursado cuatrimestral actual.

### 3.1 Contexto general

#### Marco institucional

UTN Solidaria es una iniciativa de la Subsecretaría de Asuntos Estudiantiles,
orientada a acompañar y asistir a personas en situación de vulnerabilidad a través de
acciones solidarias organizadas por estudiantes y voluntarios. Se constituye como un
espacio de articulación entre estudiantes, docentes y graduados, con el propósito de
canalizar acciones solidarias que generen un impacto positivo y sostenible.

Entre sus líneas de acción se destacan el reparto de viandas en los alrededores de la
sede Medrano, la recepción de donaciones de alimentos no perecederos, ropa, abrigo,
colchones y frazadas, el acompañamiento a instituciones como el Hospital Gandulfo y un
hogar de niños, y la generación de espacios de voluntariado.

En el marco de este trabajo práctico se propone analizar y modelar los procesos que
conforman UTN Solidaria, usándola como caso de estudio, y llevarla a la práctica
mediante el diseño y desarrollo de una solución tecnológica.

#### Contexto del problema

Una organización sin fines de lucro dedicada a la recolección y distribución de
donaciones de bienes materiales enfrenta dificultades para gestionar y dar
trazabilidad a los recursos que recibe. La falta de un sistema unificado provoca
desorden, duplicación de registros y poca claridad sobre el destino de las donaciones,
perjudicando la transparencia y la eficiencia.

#### Nuestro sistema

Se propone el diseño y desarrollo de **DonaTrack**: una solución digital orientada a
organizar, registrar y monitorear las donaciones desde su recepción en el depósito
hasta su entrega a entidades beneficiarias. El objetivo es mejorar la distribución de
recursos y fortalecer la confianza de donantes y beneficiarias.

Se diseñará e implementará una **arquitectura distribuida en servicios**:

- **Servicio de Donaciones**: alta, baja y modificación de personas donantes (humanas
  o jurídicas), registro de donaciones con trazabilidad y auditoría de estados, y
  gestión de entidades beneficiarias.
- **Servicio de Logística**: generación de rutas de los camiones para la entrega de
  las donaciones y trazabilidad durante ese proceso.
- **Servicio de Incentivos**: analítica de la actividad de donación, cálculo del
  progreso de los donantes y otorgamiento de incentivos simbólicos como insignias y
  ascensos de categoría.
- **Servicio de Notificaciones**: envío por distintos medios ante cambios de estado de
  las donaciones, asignaciones, entregas realizadas y recompensas desbloqueadas.
- **Servicio de Autenticación**.
- **Componente de front para server-side rendering** (se explica en la Entrega 5).

El diagrama de despliegue inicial es orientativo y sufrirá modificaciones conforme
avance el desarrollo. En él se omite la integración entre servicios; se darán detalles
adicionales a medida que avancen las entregas.

#### Cronograma original (versión anual — no vigente)

| Nro. | Título | Semana propuesta |
|---|---|---|
| 1 | Arquitectura y Modelado en Objetos I + Diseño de UI | Semana del 20 de abril |
| 2 | Arquitectura y Modelado en Objetos II | Semana del 25 de mayo |
| 3 | Arquitectura y Modelado en Objetos III | Semana del 29 de junio |
| 4 | Persistencia y Maquetado de Interfaz de Usuario | Semana del 7 de septiembre |
| 5 | Arquitectura Web MVC | Semana del 12 de octubre |
| 6 | Despliegue, Observabilidad y Seguridad | Semana del 16 de noviembre |

### 3.2 Requerimientos mínimos de UI/UX

#### Acceso

- Landing page pública que permita presentar el propósito y objetivos de DonaTrack,
  mostrar ejemplos destacados de donaciones del último mes, e incluir enlaces a:
  visualización de fotos de donaciones entregadas sin login, registro o inicio de
  sesión para donantes y beneficiarias, e información legal y de privacidad.
- Mostrar las donaciones ya entregadas en un **mapa interactivo**, con detalles al
  hacer clic en cada marcador.
- Permitir el registro de personas donantes y entidades beneficiarias.
- Permitir el inicio de sesión de donantes, beneficiarias y administradoras.

#### Persona donante

- Filtrar sus donaciones realizadas por estado y por categoría o subcategoría.
- Navegar por las entidades beneficiarias registradas.
- (Persona humana) Acceder a una sección de Incentivos y consultar misiones e
  insignias obtenidas.
- Recibir notificaciones cuando una donación suya es asignada, cuando cumple una
  misión, cuando sube de categoría y cuando la entidad beneficiaria recibe la donación.
- Seguir en un mapa las entregas activas de sus donaciones, viendo los camiones
  asociados con ubicación y última actualización.

#### Entidad beneficiaria

- Registrar necesidades materiales concretas.
- Ver el estado de las donaciones asignadas.
- Confirmar la recepción de una donación, cargando fotos.
- Recibir notificaciones por asignación de donaciones y confirmación de recepción.
- Seguir en un mapa las entregas activas que debe recibir, con ubicación y última
  actualización de los camiones.

#### Persona administradora

- Registrar personas donantes y donaciones recibidas en el depósito, asociándolas a la
  persona donante correspondiente.
- Actualizar el estado de las donaciones del depósito en caso de vencimiento.
- Seleccionar la entidad beneficiaria final de una donación pendiente, a partir del
  resultado de los algoritmos de selección.
- Administrar los camiones disponibles.
- Visualizar el ranking mensual de donantes más activos y el historial de rankings
  anteriores.
- Importar personas donantes ya registradas desde archivos CSV con más de 10.000 filas.

### 3.3 Entrega 1 — Arquitectura y Modelado en Objetos I

#### Objetivos

- Entrar en contacto con el dominio y sus principales abstracciones.
- Incorporar de forma paulatina conceptos y principios de diseño.
- Familiarizarse con el entorno de desarrollo y las tecnologías a aplicar.
- Familiarizarse con la arquitectura del sistema.
- Incorporar nociones de diseño UI/UX.

#### Unidades vinculadas

Unidad 2 (Herramientas de Concepción y Comunicación del Diseño), Unidad 3 (Diseño con
Objetos), Unidad 4 (Diseño de Interfaz de Usuario), Unidad 6 (Diseño de Arquitectura),
Unidad 8 (Validación del Diseño).

#### Alcance

- Servicio de Donaciones — Gestión de donantes y donaciones.
- Servicio de Donaciones — Importación masiva de donantes mediante CSV.
- Servicio de Donaciones — Gestión de entidades beneficiarias y necesidades.
- Servicio de Notificaciones — Primera iteración.
- Bocetos de interfaz de usuario.

#### Dominio

**Donantes.** Son personas humanas o jurídicas que desean aportar a las entidades
beneficiarias. Pueden registrarse antes de llevar una donación al depósito.

A las **personas humanas** se les solicita nombre, apellido, edad, número de documento,
género, dirección y al menos un medio de contacto (obligatorio: correo electrónico;
opcionales: teléfono y/o WhatsApp). Pueden determinar cuál será su medio de contacto
predeterminado para recibir notificaciones.

Las **personas jurídicas** deben indicar razón social, tipo (Gubernamental, ONG,
Empresa, Institución), rubro y al menos un medio de contacto. Cada organización tiene
personas representantes habilitadas a operar en su nombre.

**Registro de personas donantes.** Las donantes (o un representante, en el caso de las
jurídicas) se acercan al depósito para ingresar una donación. Si no tienen usuario, una
persona administradora les pide los datos de registro y se les envía un correo de
bienvenida para que accedan por primera vez.

**Donaciones y segmentación.** Una vez registrada la persona donante, la administradora
completa el formulario de la donación en su nombre. Se incluye una descripción general
y los bienes que contiene. De cada bien se conoce una descripción y, opcionalmente, una
foto. Los bienes pertenecen a una **categoría** (mobiliario, alimentos, vestimenta,
etc.) y cada categoría tiene múltiples **subcategorías**.

La subcategoría es la **unidad mínima de asignación** del sistema, y permite
identificar con precisión qué bien se necesita o se dona. En "Alimentos" pueden
definirse subcategorías como fideos secos, arroz, legumbres secas o aceite vegetal; en
"Vestimenta", camperas de abrigo, remeras, pantalones o ropa infantil.

Hay categorías en las que es necesario conocer si el estado es **usado o nuevo**, como
mobiliario o vestimenta. Si los bienes son **perecederos**, se debe ingresar la fecha
de vencimiento. En todos los casos se contabiliza la cantidad de un mismo producto en
una unidad determinada (kilogramos, unidades o según corresponda).

Se registra la totalidad de los bienes en una **única carga**, a partir de la cual el
sistema realiza automáticamente una **segmentación interna**, generando múltiples
**posibles donaciones independientes** agrupadas obligatoriamente por subcategoría. Cada
donación resultante queda asociada a una única subcategoría, garantizando coherencia en
el proceso automático de vinculación con las necesidades.

En bienes perecederos, el sistema puede generar donaciones separadas cuando hay
diferencias en la fecha de vencimiento. En los no perecederos cuyo estado es relevante,
debe consignarse si son nuevos o usados.

*Ejemplos:*
- La oficina corporativa de Arcos Plateados está en mudanza y quiere donar muebles
  usados: seis sillas y una mesa rectangular.
- Una planta industrial de pastas secas desea donar 100 paquetes de fideos y 50
  tetra-packs de tomate que vencen el 01/01/2027.

**Entidades beneficiarias y necesidades.** Son organizaciones sin fines de lucro que se
registran para recibir donaciones: escuelas rurales, comedores, espacios de tutoría,
entre otros. De cada una se conoce razón social, dirección completa, teléfono y correos
de las personas representantes.

Pueden registrar sus necesidades materiales indicando subcategoría y una breve
descripción. La plataforma distingue entre necesidades **recurrentes** y
**extraordinarias**.

- **Extraordinarias**: surgen ante situaciones excepcionales (mudanzas, inundaciones,
  incendios, vencimientos próximos). Ejemplo: la escuela rural N°10, tras una
  inundación, necesita 30 bancos y 30 sillas. La cantidad solicitada puede cubrirse con
  **donaciones parciales**: una persona dona 2 sillas, otra 20, y así hasta alcanzar las
  30. La necesidad se considera satisfecha cuando se recibe una cantidad **igual o
  superior** a la requerida.
- **Recurrentes**: vinculadas al funcionamiento habitual de la organización, con bienes
  de consumo periódico. Se satisfacen dentro del período correspondiente según la
  cantidad objetivo definida. Ejemplo: el comedor infantil "Escobar Sonrisas" requiere
  100 paquetes de fideos por semana.

**Importación masiva por CSV.** Dado que la ONG ya tiene un histórico de donantes, debe
permitirse migrar su información importando un archivo `.csv`. Cada línea representa una
persona donante (humana o jurídica) con los campos mínimos para identificarla y
contactarla. Si el registro ya existe (el correo ya está registrado), se **actualiza** su
información; si no, se **crea** el usuario y se le envían sus credenciales de acceso.

Formato del archivo:

| TipoPersona | TipoDoc | Documento | Nombre/Razón Social | Email | Teléfono |
|---|---|---|---|---|---|
| HUMANA | DNI | 12345678 | Ana Pérez | ana@mail.com | +54 11 5555-5555 |
| JURIDICA | CUIT | 30-12345678-9 | Arcos Plateados S.A. | contacto@empresa.com | +54 11 4444-4444 |

Se provee un archivo de prueba de 20.000 filas.

**Servicio de Notificaciones.** Se solicita exponer un componente notificador que, dado
un destinatario, un mensaje y un medio de notificación (correo electrónico, SMS o
WhatsApp), pueda realizar el envío. En esta iteración se **simula** la llamada a los
servicios externos y se marcan las notificaciones como completadas. La integración real
llega en la próxima entrega.

**Bocetos de interfaz.** El sistema tendrá una interfaz web para donantes,
beneficiarias y administradoras. Se solicita realizar bocetos de las principales
interfaces, partiendo de la especificación de requerimientos mínimos.

#### Requerimientos detallados

*De dominio:*
- Gestión de personas donantes.
- Gestión de las donaciones resultantes.
- Gestión de entidades beneficiarias y sus necesidades.
- Importación masiva de donantes en CSV.
- Envío de notificaciones simuladas por distintos medios.

*De implementación:*
- El sistema debe implementarse como una **aplicación multimódulo**.
- El Servicio de Donaciones debe exponer un endpoint web simple que ante un `GET`
  retorne: `Hola desde el servicio de Donaciones.`

#### Entregables

- **Modelo del Dominio**: diagrama de clases inicial, **uno por servicio**.
- **Diagramas de Arquitectura**: despliegue, componentes y/o cualquier otro que refleje
  la arquitectura física y lógica de la entrega.
- **Justificaciones de Diseño Iniciales.**
- **Diagrama General de Casos de Uso.**
- **Bocetos de interfaz de usuario** (se sugiere el uso de IA).
- **Implementación** de los requerimientos.

#### Pregunta de discusión

¿Un modelo de dominio rico es una inversión necesaria para capturar la complejidad del
negocio, o una sobreingeniería innecesaria frente a un modelo anémico más simple?

### 3.4 Entrega 2 — Arquitectura y Modelado en Objetos II

#### Objetivos

- Diseñar e implementar, de manera incremental, las nuevas funcionalidades.
- Incorporar nociones de ejecución de tareas asincrónicas y/o calendarizadas.
- Familiarizarse con las APIs como mecanismo de integración y exponer servicios a
  través de un protocolo de red.

#### Unidades vinculadas

Unidades 2, 3, 6, 7 (Integración de Sistemas) y 8.

#### Alcance

- Servicio de Donaciones — Trazabilidad de las donaciones.
- Servicio de Notificaciones — Integración concreta con distintos medios.
- Servicio de Donaciones — Asignación de necesidades materiales.
- Servicio de Incentivos — Primera iteración de analítica y recompensas.
- Exposición REST de los servicios.

#### Dominio

**Estados de las donaciones.** El sistema **debe garantizar** la trazabilidad y
auditoría de los estados de cada donación.

| Estado | Cuándo se alcanza |
|---|---|
| **En depósito** | Al registrarse; disponible para ser asignada |
| **Asignación realizada** | Al día siguiente, si el algoritmo de asignación la asigna a una entidad |
| **Lista para entregar** | Cuando se planificó una ruta que la incluye |
| **En traslado** | Cuando el camión asignado inició el recorrido |
| **Entregada** | Cuando la entidad beneficiaria confirmó la entrega |
| **Entrega fallida** | Si no se pudo entregar el día estipulado; vuelve al depósito |
| **Vencida** | Cuando una administradora del depósito lo determina |

Si la entrega falla, se debe registrar una justificación (por ejemplo: "Tocamos timbre
pero nadie respondió").

**Asignación de las donaciones.** Por cada donación en estado *En depósito*, el sistema
debe ejecutar un proceso de **matchmaking**: la evaluación y asignación entre donaciones
y necesidades materiales compatibles, para determinar el destino más adecuado.

Se deben implementar **dos algoritmos** con criterios distintos, que asignan un
**score normalizado entre 0 y 1** a cada cruce donación-necesidad.

Una donación solo puede evaluarse contra necesidades que soliciten **exactamente la
misma subcategoría**.

Calculados los puntajes, el sistema ordena los resultados y genera un ranking,
proponiendo como máximo las **diez** necesidades con mayor puntuación para cada
donación. El diseño debe contemplar la **incorporación de nuevos algoritmos a futuro**.

*Algoritmo 1 — Compatibilidad Semántica.* Analiza la correspondencia entre las
características del bien donado y las necesidades declaradas.

```
S_sem = a1·χ1² + a2·χ2 + a3·χ3
```

Las variables `χn` se calculan para cada par (Donación, Necesidad) y devuelven un valor
en `[0, 1]`:

- **χ1 — Similitud textual**: resultado de aplicar la Distancia de Levenshtein entre el
  nombre del bien y la descripción de la necesidad.
  ```
  χ1 = 1 − DistLevenshtein(Nombre, Desc) / max(Longitud(Nombre), Longitud(Desc))
  ```
  Si son idénticos, la distancia es 0 y χ1 = 1. Si son totalmente distintos, tiende a 0.

- **χ2 — Cobertura de volumen**: penaliza el desperdicio. Con `R = CantDonada / CantRequerida`:
  ```
  R ≤ 1  ⇒  χ2 = R
  R > 1  ⇒  χ2 = 1/R
  ```

- **χ3 — Tipo de necesidad**: Extraordinaria ⇒ 1; Recurrente ⇒ 0.5. Ajustable a futuro.

Las constantes `an` son **configurables por el administrador** y determinan qué le
importa más a la ONG. **La suma de todos los pesos debe dar 1.** Ejemplo: a1 = 0.40
(peso textual), a2 = 0.40 (peso del volumen), a3 = 0.20 (peso del tipo).

*Algoritmo 2 — Prioridad a sub-atendidos.* Prioriza a organizaciones que recibieron
menos donaciones en el último trimestre.

```
S_sub = 1 / (1 + α · CantDonacionesTrimestre)
```

Donde `CantDonacionesTrimestre` es la cantidad de donaciones recibidas durante el
trimestre por la entidad que publicó la necesidad, y `α` un factor de suavizado (ej.
0.1) para que el puntaje baje gradualmente.

A partir de los rankings, el componente puede filtrar automáticamente las necesidades
que aparecieron en el **Top 10 de ambos algoritmos**, para que una administradora
confirme el destino final. Si no hubo coincidencias, la interfaz muestra ambos rankings
por separado.

La ejecución de los algoritmos debe realizarse **en horarios de baja carga** para no
degradar el desempeño.

**Analítica de donantes.** El Servicio de Incentivos consolida y analiza los datos de
donaciones para generar métricas de impacto y comportamiento. Su finalidad es brindar a
las donantes información clara sobre su participación e incentivar el uso sostenido
mediante ludificación, y poner a disposición de las administradoras información agregada.

La persona donante puede observar en su perfil sus totales históricos, gráficos de
evolución por período, comparaciones mensuales, el número total de organizaciones que
ayudó, su posición en el ranking de donantes activos e indicadores de impacto acumulado.

**Recompensas.** Se establecen tres categorías de donante: **Colaborador, Sostenedor y
Transformador**, visibles públicamente junto al nombre de usuario.

Para subir de categoría hay que completar misiones **en orden secuencial**: al completar
una, se desbloquea la siguiente. Cada misión completada otorga una **insignia**, visible
en el perfil si la persona la configura como visible.

Ejemplos de misiones:

| Tipo | Condición |
|---|---|
| **Racha** | Donar durante X meses consecutivos |
| **Completitud** | Donar en X categorías distintas |
| **Hábil Donador** | Una donación que supere X cantidad de bienes |
| **Donaciones Exitosas** | Lograr X donaciones recibidas exitosamente |

El donante debe poder ver en todo momento el progreso de su misión actual y la distancia
restante. Ciertas misiones, como Racha, pueden **perder el progreso acumulado** ante
determinadas condiciones (por ejemplo, si no dona durante un mes completo).

En esta entrega se solicita la integración entre el Servicio de Donaciones y el de
Incentivos, para que las donaciones impacten en el cálculo de progreso.

**Difusión de insignias y ranking mensual.** Cada vez que una donante obtenga una
insignia, el sistema activa automáticamente un flujo de procesamiento que recibe un texto
descriptivo, **genera una imagen** asociada a la insignia y **publica el contenido en una
red social**, mencionando al usuario. Se solicita integrar una herramienta de
automatización de flujos low-code (por ejemplo, n8n).

Al cierre de cada mes, un flujo programado ejecuta el **ranking mensual de donantes más
activos**, calculado según la cantidad de misiones cumplidas en ese período (sin importar
la categoría). El sistema persiste el ranking del mes y destaca a los tres primeros.

**Eventos e integración con medios de notificación (Parte I).** Se requiere notificar en
los siguientes casos:

- A una persona donante, cuando no registre interacción con la plataforma durante más de
  **20 días** (Servicio de Donaciones).
- A una entidad beneficiaria, cuando se le asigna una donación en base a sus necesidades
  (Servicio de Donaciones).
- A una persona donante, cuando su donación acaba de ser asignada (Servicio de Donaciones).
- A una persona donante, cuando cumple una misión (Servicio de Incentivos).
- A una persona donante, cuando cambia de categoría (Servicio de Incentivos).

En esta iteración se solicita el envío de notificaciones **reales** por correo
electrónico, SMS y/o WhatsApp.

**Exposición REST — Parte I.**

*Gestión de personas donantes y donaciones:*
- Operaciones CRUD sobre las donaciones.
- Operaciones CRUD sobre las personas donantes (jurídicas y humanas).
- Cambios de estado de una donación, garantizando trazabilidad y auditoría.

*Gestión de entidades beneficiarias y necesidades:*
- Operaciones CRUD sobre las entidades beneficiarias.
- Operaciones CRUD sobre las necesidades materiales (recurrentes y extraordinarias).
- Obtención del ranking generado por los algoritmos y selección de entidad final.
- Ejecución a demanda de los algoritmos de asignación.

*Servicio de Incentivos:*
- Métricas de actividad de una donante (período actual y acumulado).
- Misiones completadas por una donante.
- Insignias de una donante.

#### Entregables

- **Modelo del Dominio**: diagrama de clases que contemple las funcionalidades requeridas.
- **Diagrama de despliegue y componentes.**
- **Justificaciones de Diseño**: documento y diagramas complementarios.
- **Implementación** de los requerimientos de la entrega.
- **Implementación** del flujo automatizado de publicación y difusión de insignias.

#### Pregunta de discusión

¿Procesar las asignaciones de donaciones en tiempo real mejora realmente el sistema, o
introduce complejidad y problemas de escalabilidad que justifican un enfoque batch
asincrónico?

### 3.5 Entrega 3 — Arquitectura y Modelado en Objetos III

#### Objetivos

- Diseñar e implementar, de manera incremental, las nuevas funcionalidades.
- Exponer un servicio a través de un protocolo de red.
- Incorporar flujos de trabajo asincrónicos.

#### Unidades vinculadas

Unidades 2, 3, 6, 7 y 8.

#### Alcance

- Servicio de Logística — Entrega y planificación de rutas.
- Servicio de Logística — Monitoreo de camiones en tiempo real.
- Exposición REST de los servicios.

#### Dominio

**Entrega y planificación de rutas.** La organización cuenta con una flota de camiones.
De cada uno se conoce la **patente**, la **capacidad en volumen (m³)**, la **altura (m)**
y la **capacidad de carga (kg)**. Todos pueden transportar cualquier tipo de bien y
parten siempre desde el depósito.

La plataforma integra un **componente externo** que genera las rutas de reparto del día
siguiente. Recibe, por ejecución, un conjunto de donaciones en estado *Asignación
Realizada* junto con la información de los camiones disponibles. Devuelve, por cada
camión, una lista **ordenada** de destinos (direcciones de las entidades beneficiarias)
con las entregas a realizar en cada una. Al completar la planificación, las rutas quedan
disponibles para los choferes en su aplicación.

**Trazabilidad de las entregas.** Antes de iniciar el recorrido, el chofer indica en el
sistema que da inicio a su ruta, y las entregas asignadas pasan a **En traslado**. Cuando
el camión entrega en la sede de la entidad, esta confirma la recepción: la entrega pasa a
**Entregada** y queda registrado qué camión la realizó. Luego la entidad carga fotos de
la donación recibida.

Si la entidad informa que no recibió la entrega en el día correspondiente, se marca como
**No recibida** y el caso es revisado por las administradoras. Si la donación regresa al
depósito, la entrega vuelve al estado **Pendiente**.

**Monitoreo de camiones en tiempo real.** El sistema debe mostrar en un **dashboard
administrativo** la posición actual de los camiones y su avance sobre la ruta asignada,
en tiempo real. El equipo de campo propuso **dos alternativas**, y hay que **elegir una**:

- **Dispositivo GPS configurable instalado en el camión**, que envía periódicamente
  ubicación y velocidad.
- **Aplicación móvil del conductor**, que reporta la geolocalización mientras la ruta
  esté activa.

La configuración de los dispositivos o de la app es responsabilidad del equipo externo.
La plataforma debe **recibir, validar y procesar** la información enviada, y definir el
**contrato de integración** correspondiente.

**Eventos — Parte II.** Se requiere notificar en:

- **Inicio de ruta**: a todas las entidades beneficiarias y donantes cuyas entregas
  formen parte de la ruta iniciada. Debe incluir un **enlace al mapa interactivo**.
- **Entrega realizada con éxito**: a la entidad y al donante. Debe incluir un
  **comprobante de entrega** con fecha, hora y camión responsable.
- **Entrega no satisfactoria**: a la entidad, a la donante y a las administradoras. Si la
  entrega puede replanificarse, se deja constancia del estado y puede generarse una nueva
  asignación de ruta.

**Exposición REST — Parte II.** Servicio de Logística: gestión de flota de camiones, y
operaciones CRUD sobre rutas y entregas.

**Integración entre servicios.** Se debe garantizar la integración entre todos los
servicios para obtener información relevante y orquestar los casos de uso. La interacción
entre servicios de dominio **puede resolverse mediante comunicación sincrónica (API)**, y
la direccionalidad de las interacciones **queda sujeta a discusión de cada equipo**,
priorizando bajo acoplamiento y trazabilidad.

La integración entre los servicios de dominio y el **Servicio de Notificaciones debe ser
asincrónica, a través de una cola de mensajes**, para no afectar la disponibilidad ante
picos de carga o fallas transitorias.

#### Requerimientos detallados

*De dominio:*
- Operaciones a través de los endpoints solicitados, mediante API REST.
- Generar, en horarios de baja carga, los planes de ruta para la siguiente jornada
  operativa, integrándose con el componente externo provisto.
- Permitir a los choferes informar el comienzo de su ruta.
- Mostrar en tiempo real, tanto a donantes como a beneficiarias, la localización de los
  camiones.
- Gestionar la recepción de la entrega por parte de las entidades en todos los casos.

*De implementación:*
- El sistema debe exponer una **URL de callback** donde el componente externo notifique el
  resultado de la planificación. La llamada de retorno permite registrar las rutas
  generadas y actualizar el estado de las entregas asignadas.
- Las solicitudes al proveedor deben realizarse **en lotes**: por restricciones del
  proveedor, cada ejecución procesa **como máximo 100 donaciones**.
- Es posible que el planificador no pueda asignar todas las donaciones. En ese caso
  devuelve las no asignadas en un campo aparte, y es responsabilidad de nuestro sistema
  **volver a planificarlas**.

#### Entregables

- **Modelo del Dominio**: diagrama de clases por servicio.
- **Justificaciones de Diseño**: documento y diagramas complementarios.
- **Implementación** de los requerimientos de la entrega.
- **Diagrama de despliegue actualizado**, incluyendo las integraciones concretas entre
  servicios.

#### Pregunta de discusión

¿La precisión y confiabilidad de dispositivos GPS dedicados justifican su costo frente a
la flexibilidad y menor inversión de una app móvil para el seguimiento de camiones?

### 3.6 Entrega 4 — Persistencia y Maquetado

#### Objetivos

- Incorporar nociones de persistencia de datos en un medio relacional.
- Incorporar nociones de la técnica de mapeo objeto-relacional.
- Incorporar nociones de desnormalización del modelo relacional.
- Incorporar nociones de diseño UI/UX.
- Incorporar nociones de maquetado web mediante HTML5.
- Incorporar nociones de aplicación de estilos mediante CSS.

#### Unidades vinculadas

Unidades 2, 4, 5 (Diseño de Datos y Estrategias de Persistencia), 6 y 8.

#### Alcance

- Persistencia del modelo de objetos por servicio.
- Estrategia de persistencia políglota.
- Diseño y maquetado de interfaces de usuario.

#### Requerimientos detallados

- Persistir las entidades del modelo planteado para cada servicio.
- El **Servicio de Logística** debe usar una **base de datos documental**; los servicios
  restantes, una **base de datos relacional**.
- Contemplar estrategias de **desnormalización** cuando resulte necesario para optimizar
  consultas de lectura.
- Implementar los **bocetos de interfaz** planteados en la Entrega 1.

#### Consideraciones para persistencia

Cada servicio debe tener persistencia en un **esquema relacional propio**, excepto
Logística que debe tener un esquema no relacional. Para la persistencia relacional se
puede **compartir un mismo motor** a fines de simplicidad.

Para la persistencia relacional se debe utilizar un **ORM**; para la documental, un **ODM**.

#### Consideraciones para el maquetado

| Requisito | Detalle |
|---|---|
| **UI orientada a la usabilidad** | Principios de usabilidad heurística, curva de aprendizaje mínima para usuarios sin experiencia |
| **Navegación optimizada** | Rutas y jerarquías con **profundidad máxima de tres niveles** hacia funcionalidades clave |
| **Familiaridad del diseño** | Adherir a design patterns establecidos (Material Design, Human Interface Guidelines) |
| **Microcopy y ayudas visuales** | Etiquetas, tooltips, placeholders y onboarding hints en flujos críticos |
| **Gestión de estados** | Toda acción acompañada de notificaciones no intrusivas (toasts, modals, banners) |
| **Indicadores asincrónicos** | Loaders, skeleton screens o spinners en procesos con latencia **mayor a 300 ms** |
| **Layout responsive** | Media queries, flexbox o grillas, con adaptabilidad entre breakpoints móvil, tablet y desktop |
| **Integridad cross-device** | Componentes interactivos accesibles y operables sin distorsión ni superposición |
| **Normas WCAG** | Mínimo **nivel AA**: soporte ARIA, foco visible, estructuras semánticas HTML correctas |
| **Interacción móvil** | Elementos clickeables con hit área mínima de **48×48 px** |
| **Estilo visual unificado** | Sistema de **design tokens**: colores, tipografías, espaciados y bordes |

#### Entregables

- **Modelo del Dominio**: actualización del diagrama de clases.
- **Justificaciones de Diseño**: documento y diagramas complementarios.
- **Modelo de datos**: **diagrama entidad-relación físico por servicio**.
- **Justificaciones y consideraciones de diseño relacional.**
- **Implementación** en código de los requerimientos.
- **Maquetado** e implementación en HTML de las interfaces requeridas, contemplando la
  navegabilidad entre interfaces y aplicando estilos.

#### Pregunta de discusión

¿El uso de una base de datos NoSQL en el Servicio de Logística es una decisión técnica
acertada por la naturaleza de los datos, o una elección innecesaria frente a la robustez
de un modelo relacional?

### 3.7 Entrega 5 — Arquitectura Web MVC

#### Objetivos

- Incorporar patrones de arquitectura MVC mediante la implementación de un cliente liviano.
- Aplicar principios de diseño centrado en el usuario (UI/UX).

#### Unidades vinculadas

Unidades 2, 4, 6, 7, 8 y 9 (Diseño y Seguridad).

#### Alcance

- Implementación de un cliente liviano desacoplado.
- Servicio de Autenticación.

**Cliente liviano desacoplado.** Se debe crear un **nuevo proyecto independiente** para
la interfaz gráfica. El cliente será **desacoplado de la lógica de negocio**, consumiendo
las APIs REST desarrolladas en entregas anteriores. Para generar las interfaces se debe
utilizar un **motor de plantillas (template engine)**. Las vistas deben renderizar
dinámicamente los datos obtenidos de las APIs.

**Servicio de Autenticación.** Encargado de la gestión de **tokens de autorización**
desde el servidor frontend hacia los servicios de aplicación donde reside la lógica de
negocio.

#### Entregables

- **Implementación** de un cliente liviano desacoplado, renderizando las vistas generadas.
- **Bonus**: implementación de un **SSO** con soporte de **social login**.

#### Pregunta de discusión

¿Adoptar SSR responde a una necesidad real del sistema, o limita la experiencia de
usuario frente a la flexibilidad de una SPA moderna?

### 3.8 Entrega 6 — Despliegue, Observabilidad y Seguridad

#### Objetivos

- Familiarizarse con técnicas y proveedores de despliegue.
- Incorporar herramientas de monitoreo, observabilidad y seguridad.
- Familiarizarse con diferentes estrategias de integración.
- Familiarizarse con la arquitectura de microservicios y sus componentes asociados.

#### Unidades vinculadas

Unidades 2, 6, 8 y 9.

#### Alcance

- Sistema desplegado en la nube.
- Herramientas de observabilidad, monitoreo, seguridad y control.
- Nuevas estrategias de diseño de APIs web.
- Nociones de microservicios.

#### Requerimientos detallados

- **Desplegar el sistema en la nube** para que pueda ser accedido por el público general.
- Implementar soluciones de **observabilidad** que permitan recolectar, visualizar y
  analizar **métricas, trazas y logs**, para obtener una visión integral y en tiempo real
  del estado y comportamiento de los servicios.
- Incorporar herramientas de **monitoreo proactivo** que supervisen la salud de los
  servicios y permitan configurar políticas de **autorestart** ante fallos.
- Implementar **rate limiting** para regular el tráfico entrante y proteger los recursos
  del sistema frente a abusos o sobrecarga.
- Implementar **una** de estas dos opciones:
  - **Opción 1**: el protocolo **gRPC** como reemplazo de HTTP para las comunicaciones
    entre servicios que requieren actualizaciones en tiempo real.
  - **Opción 2**: una interfaz **GraphQL** en el Servicio de Donaciones o en el de
    Logística, para dar soporte a filtros dinámicos y selección de campos específicos.
- **Bonus**: ajustar los servicios para incorporarlos a una **arquitectura de
  microservicios**, con registro en un **Service Registry** y consumo a través de un
  **API Gateway**.
- **Bonus**: implementar el patrón **Transactional Outbox** para asegurar la persistencia
  confiable de los eventos de notificación generados inmediatamente después de una
  operación de escritura transaccional.

#### Entregables

- **Diagrama de Despliegue** de la solución actual.
- **Implementación** de herramientas de observabilidad, monitoreo, seguridad y control.
- **Implementación** de un web service utilizando **gRPC o GraphQL**.
- **Propuesta en diagrama de despliegue** e implementación de la arquitectura de
  microservicios.

#### Pregunta de discusión

¿Priorizar eficiencia y tipado fuerte con gRPC es más adecuado que la flexibilidad en
consultas que ofrece GraphQL, o depende estrictamente del contexto del sistema?

---

## 4. Entregables y checkpoints

Los checkpoints están **ordenados por dependencia**: cada uno se apoya en el anterior.
Marcar con `[x]` a medida que se completan.

**Regla general para todas las entregas**: los diagramas y el documento de
justificaciones **no se dejan para el final**. Se actualizan a medida que se toman las
decisiones, mientras están frescas. Es lo que más se descuida y lo que más se evalúa.

**Fechas reales del cursado cuatrimestral** (completar con el profesor):

| Entrega | Fecha límite |
|---|---|
| 1 | _por confirmar_ |
| 2 | _por confirmar_ |
| 3 | _por confirmar_ |
| 4 | _por confirmar_ |
| 5 | _por confirmar_ |
| 6 | _por confirmar_ |

### 4.0 Checkpoint 0 — Setup inicial

No es un entregable, pero sin esto no arranca nada.

- [ ] **CP0.1** — JDK 21 instalado y configurado en IntelliJ
- [ ] **CP0.2** — Repositorio creado en GitHub, con ramas `main` y `develop`
- [ ] **CP0.3** — `.gitignore` y `.env.example` en la raíz
- [ ] **CP0.4** — POM padre (`packaging: pom`) con `dependencyManagement` y `java.version`
- [ ] **CP0.5** — `server/pom.xml` agregador
- [ ] **CP0.6** — Módulo `donaciones` con su POM y estructura `src/main/java`
- [ ] **CP0.7** — `mvn clean install` verde desde la raíz
- [ ] **CP0.8** — Todo el grupo clonó y le compila
- [ ] **CP0.9** — README con instrucciones de levantado

---

### 4.1 Entregable 1

**Qué hay que entregar:**

- [ ] Diagrama de clases del dominio, **uno por servicio** (Donaciones y Notificaciones)
- [ ] Diagrama de casos de uso general
- [ ] Diagrama de despliegue y de componentes
- [ ] Documento de justificaciones de diseño iniciales
- [ ] Bocetos de interfaz de usuario
- [ ] Implementación funcionando

**Checkpoints:**

- [ ] **CP1.1 — Endpoint de humo.** `DonacionesApplication` levanta en el puerto 8081 y
      un `GET` devuelve `Hola desde el servicio de Donaciones.` Es el requerimiento
      textual del enunciado y confirma que el andamiaje Maven está bien.

- [ ] **CP1.2 — Dominio de donantes.** Jerarquía persona humana / jurídica, medios de
      contacto con uno predeterminado, dirección, representantes de las jurídicas.
      Decidir cómo modelar la jerarquía: ¿herencia, o composición con un rol? Esta
      decisión reaparece en la Entrega 4 con el mapeo ORM, así que conviene pensarla ya.

- [ ] **CP1.3 — Repositorio en memoria.** Definir la **interfaz** `DonanteRepository` y
      una implementación en memoria. La interfaz es lo que importa: en la Entrega 4 se
      cambia la implementación y nada más debería enterarse.

- [ ] **CP1.4 — Categorías, subcategorías y bienes.** Modelar `Categoria`,
      `Subcategoria` y `Bien` con descripción, foto opcional, cantidad + unidad, estado
      nuevo/usado cuando aplique, y fecha de vencimiento cuando sea perecedero.
      Pregunta a resolver: ¿qué determina si una categoría necesita el estado
      nuevo/usado, y qué determina si es perecedera?

- [ ] **CP1.5 — Donación y segmentación.** El caso más interesante de la entrega: una
      carga única entra y salen múltiples donaciones independientes, agrupadas por
      subcategoría, separando además por fecha de vencimiento en perecederos.
      Tests obligatorios con los dos ejemplos del enunciado: los muebles usados de Arcos
      Plateados y los fideos + tetra-packs de la planta de pastas.

- [ ] **CP1.6 — Entidades beneficiarias.** Razón social, dirección completa, teléfono,
      correos de representantes.

- [ ] **CP1.7 — Necesidades materiales.** Recurrentes y extraordinarias, con subcategoría,
      descripción y cantidad objetivo. Modelar la satisfacción por **donaciones parciales**
      (se satisface al igualar o superar lo requerido) y el concepto de **período** en las
      recurrentes. Otra decisión: ¿dos clases distintas, o una con una estrategia adentro?
      Lo que se elija acá condiciona el matchmaking de la Entrega 2.

- [ ] **CP1.8 — Importación CSV.** Parseo del archivo con las seis columnas del formato,
      lógica **upsert** por correo electrónico (existe → actualizar; no existe → crear y
      enviar credenciales), y manejo de errores **por fila** (una fila mal formada no
      puede abortar las 20.000).

- [ ] **CP1.9 — CSV a escala.** Probar con el archivo de 20.000 filas y **medir el tiempo**.
      Si tarda demasiado, ver por qué. Ese número es un dato concreto para el documento de
      justificaciones.

- [ ] **CP1.10 — Módulo notificaciones.** Segundo módulo Maven, puerto 8082. Componente
      que recibe destinatario, mensaje y medio (correo / SMS / WhatsApp), simula el envío
      y marca la notificación como completada. Diseñarlo **pensando en la Entrega 2**: la
      integración real no debería obligar a reescribir el modelo, solo a cambiar las
      implementaciones de cada medio.

- [ ] **CP1.11 — Tests.** Cobertura al menos de segmentación, satisfacción de necesidades
      y upsert del CSV. No hace falta cubrir todo, sí las reglas de negocio no triviales.

- [ ] **CP1.12 — Bocetos de UI.** Cubrir landing, registro/login, panel de donante, panel
      de entidad beneficiaria y panel de administrador. El enunciado sugiere usar IA. Estos
      bocetos se implementan en la Entrega 4, así que no conviene dibujar cosas imposibles.

- [ ] **CP1.13 — Diagramas.** Clases por servicio, casos de uso, despliegue y componentes.
      En `.puml` dentro de `Docs/diagramas/`.

- [ ] **CP1.14 — Justificaciones.** Documento en `Docs/justificaciones/entrega-1.md`.
      Debe cubrir al menos: por qué la separación en servicios, cómo se modeló la jerarquía
      de donantes y por qué, cómo se modelaron las necesidades, y la respuesta a la
      pregunta de discusión (modelo rico vs anémico) apoyada en el propio código.

- [ ] **CP1.15 — Cierre.** Merge a `main`, tag `entrega-1`, README actualizado.

---

### 4.2 Entregable 2

**Qué hay que entregar:**

- [ ] Diagrama de clases actualizado
- [ ] Diagrama de despliegue y componentes actualizado
- [ ] Documento de justificaciones + diagramas complementarios
- [ ] Implementación de los requerimientos
- [ ] Implementación del flujo automatizado de difusión de insignias

**Checkpoints:**

- [ ] **CP2.1 — Máquina de estados.** Los siete estados con sus transiciones válidas.
      Una transición inválida tiene que fallar explícitamente, no pasar silenciosamente.

- [ ] **CP2.2 — Auditoría de estados.** Historial completo: estado anterior, estado nuevo,
      momento, quién lo hizo y justificación cuando corresponde (obligatoria en *Entrega
      fallida*). El enunciado dice que el sistema **debe garantizar** esto.

- [ ] **CP2.3 — DTOs y validación.** Separar el modelo de dominio de lo que entra y sale
      por HTTP. Bean Validation sobre los DTOs. Nunca exponer entidades de dominio
      directamente.

- [ ] **CP2.4 — Manejo de errores centralizado.** `@RestControllerAdvice` con respuestas de
      error consistentes y códigos HTTP correctos. Hacerlo una vez, ahora, y no repetirlo
      en cada controller.

- [ ] **CP2.5 — REST de donantes y donaciones.** CRUD completo + endpoint de cambio de
      estado.

- [ ] **CP2.6 — REST de entidades y necesidades.** CRUD completo de ambas.

- [ ] **CP2.7 — Abstracción de matchmaking.** Interfaz común a todos los algoritmos, con
      score normalizado en `[0,1]`. El enunciado pide explícitamente que se puedan
      **agregar algoritmos nuevos a futuro**: si agregar un tercero obliga a tocar código
      existente, el diseño está mal.

- [ ] **CP2.8 — Filtro por subcategoría.** Una donación solo se evalúa contra necesidades
      de **exactamente** la misma subcategoría. Aplicar antes de calcular scores, porque
      además reduce muchísimo el trabajo.

- [ ] **CP2.9 — Algoritmo de compatibilidad semántica.** Levenshtein para χ1, cobertura de
      volumen para χ2, tipo de necesidad para χ3. Pesos `an` **configurables**, con
      validación de que sumen 1. Tests con casos borde: textos idénticos, textos sin
      relación, R exactamente 1, R muy mayor a 1.

- [ ] **CP2.10 — Algoritmo de sub-atendidos.** Fórmula con α configurable. Requiere contar
      donaciones recibidas por entidad en el trimestre.

- [ ] **CP2.11 — Ranking e intersección.** Top 10 por algoritmo, y filtrado de las
      necesidades que aparecen en ambos Top 10. Si la intersección es vacía, devolver ambos
      rankings por separado.

- [ ] **CP2.12 — Ejecución calendarizada.** Los algoritmos corren en horario de baja carga.
      Con `@Scheduled` alcanza. También hay que exponer la **ejecución a demanda** por API.

- [ ] **CP2.13 — Módulo incentivos.** Tercer módulo, puerto 8083.

- [ ] **CP2.14 — Misiones y progreso.** Los cuatro tipos (Racha, Completitud, Hábil
      Donador, Donaciones Exitosas), con progreso consultable y distancia al objetivo.
      Diseñar para que agregar un tipo nuevo no rompa nada. Contemplar la **pérdida de
      progreso** en Racha.

- [ ] **CP2.15 — Insignias y categorías.** Desbloqueo secuencial dentro de cada categoría,
      insignia por misión completada, visibilidad configurable, y ascenso entre
      Colaborador / Sostenedor / Transformador.

- [ ] **CP2.16 — Analítica de donantes.** Totales históricos, evolución por período,
      comparaciones mensuales, cantidad de organizaciones ayudadas, posición en el ranking.

- [ ] **CP2.17 — Integración Donaciones → Incentivos.** Que una donación registrada impacte
      en el progreso de misiones. Acá se decide la **direccionalidad**: ¿Donaciones avisa,
      o Incentivos consulta? Es una decisión que hay que justificar por escrito.

- [ ] **CP2.18 — REST de incentivos.** Métricas, misiones completadas e insignias.

- [ ] **CP2.19 — Notificaciones reales.** Correo por SMTP, SMS y WhatsApp por un proveedor
      (Twilio tiene sandbox gratuito). **Credenciales solo en `.env`.**

- [ ] **CP2.20 — Los cinco eventos de la entrega.** Inactividad de más de 20 días,
      asignación a entidad, asignación a donante, misión cumplida, cambio de categoría.
      El de inactividad requiere una tarea calendarizada.

- [ ] **CP2.21 — Flujo n8n.** Webhook que recibe el texto, genera la imagen y publica en
      una red social mencionando al usuario. Exportar el workflow como JSON y versionarlo
      en `Docs/`.

- [ ] **CP2.22 — Ranking mensual.** Proceso programado al cierre de mes que calcula el
      ranking por misiones cumplidas, lo **persiste** (hay que consultar históricos) y
      destaca a los tres primeros.

- [ ] **CP2.23 — Diagramas y justificaciones.** Incluir la respuesta a la pregunta de
      discusión (tiempo real vs batch), que conecta directo con la decisión del CP2.12.

- [ ] **CP2.24 — Cierre.** Merge, tag `entrega-2`.

---

### 4.3 Entregable 3

**Qué hay que entregar:**

- [ ] Diagrama de clases por servicio, con Logística incluido
- [ ] Diagrama de despliegue actualizado **con las integraciones concretas**
- [ ] Documento de justificaciones + diagramas complementarios
- [ ] Implementación de los requerimientos

**Checkpoints:**

- [ ] **CP3.1 — Módulo logística.** Cuarto módulo, puerto 8084.

- [ ] **CP3.2 — Flota de camiones.** Patente, volumen (m³), altura (m), capacidad de carga
      (kg). CRUD.

- [ ] **CP3.3 — Modelo de ruta y entrega.** Ruta = camión + lista **ordenada** de paradas;
      parada = destino + entregas. El orden importa. Modelarlo pensando que en la Entrega 4
      esto va a una base documental.

- [ ] **CP3.4 — Contrato con el planificador externo.** Leer la documentación del servicio
      provisto y definir el formato de request y response.

- [ ] **CP3.5 — Envío en lotes.** Máximo **100 donaciones por ejecución**. Partir el
      conjunto y manejar cada lote.

- [ ] **CP3.6 — URL de callback.** Endpoint donde el componente externo notifica el
      resultado. Registrar las rutas generadas y actualizar el estado de las entregas.
      Considerar qué pasa si llega un callback duplicado o fuera de orden.

- [ ] **CP3.7 — Replanificación.** Las donaciones que el planificador devuelve sin asignar
      tienen que volver al ciclo. Cuidado con el bucle infinito si nunca entran.

- [ ] **CP3.8 — Planificación calendarizada.** El plan del día siguiente se genera en
      horario de baja carga.

- [ ] **CP3.9 — Estados de la entrega.** Pendiente → En traslado → Entregada / No recibida
      → Pendiente. Inicio de ruta por parte del chofer, confirmación por parte de la
      entidad, registro de qué camión entregó y carga de fotos.

- [ ] **CP3.10 — Decidir GPS o app móvil.** Elegir **una** de las dos alternativas y
      dejarlo escrito con sus motivos: es la pregunta de discusión de la entrega.

- [ ] **CP3.11 — Contrato de ingesta de posición.** Endpoint que **recibe, valida y
      procesa** la posición. Validar de verdad: coordenadas fuera de rango, timestamps
      viejos, camiones inexistentes.

- [ ] **CP3.12 — Dashboard en tiempo real.** Posición actual de cada camión y avance sobre
      la ruta. Server-Sent Events para empujar las actualizaciones.

- [ ] **CP3.13 — REST de logística.** Gestión de flota + CRUD de rutas y entregas.

- [ ] **CP3.14 — Cola de mensajes.** RabbitMQ levantado y la integración con Notificaciones
      **migrada a asincrónica**. Es requisito explícito del enunciado. Definir qué pasa con
      los mensajes que fallan.

- [ ] **CP3.15 — Los tres eventos de la entrega.** Inicio de ruta (con enlace al mapa),
      entrega exitosa (con comprobante: fecha, hora y camión) y entrega no satisfactoria
      (a entidad, donante y administradoras).

- [ ] **CP3.16 — Integración entre los cuatro servicios.** Definir y documentar la
      direccionalidad de cada interacción, priorizando bajo acoplamiento.

- [ ] **CP3.17 — Diagramas y justificaciones.** El diagrama de despliegue de esta entrega
      tiene que mostrar las integraciones concretas, no cajas sueltas.

- [ ] **CP3.18 — Cierre.** Merge, tag `entrega-3`.

---

### 4.4 Entregable 4

**Qué hay que entregar:**

- [ ] Diagrama de clases actualizado
- [ ] **Diagrama entidad-relación físico por servicio**
- [ ] Documento de justificaciones y consideraciones de diseño relacional
- [ ] Implementación de la persistencia
- [ ] Maquetado HTML/CSS de las interfaces, con navegabilidad y estilos

**Checkpoints:**

- [ ] **CP4.1 — Docker Compose.** PostgreSQL + MongoDB levantando con un comando, y datos
      persistentes entre reinicios.

- [ ] **CP4.2 — Esquemas separados.** Un esquema Postgres por servicio dentro del mismo
      motor. Cada servicio se conecta **solo al suyo**.

- [ ] **CP4.3 — Mapeo ORM de Donaciones.** Acá aparece la decisión del CP1.2: cómo mapear
      la jerarquía donante humana/jurídica. Las tres estrategias (tabla única, tabla por
      clase concreta, tabla por subclase) tienen consecuencias distintas y hay que
      justificar la elegida.

- [ ] **CP4.4 — Mapeo ORM de Incentivos y Notificaciones.**

- [ ] **CP4.5 — Migraciones.** Esquema versionado con Flyway o Liquibase, no
      `ddl-auto: update`. En grupo, sin migraciones versionadas la base de cada uno
      termina distinta.

- [ ] **CP4.6 — ODM de Logística.** Rutas y entregas como documentos en MongoDB. Aprovechar
      el anidamiento: la ruta completa en un solo documento.

- [ ] **CP4.7 — Desnormalizaciones.** Identificar las consultas de lectura más pesadas
      (ranking mensual, analítica del donante, dashboard) y desnormalizar donde valga la
      pena. **Documentar cada una con su motivo**: el enunciado lo pide explícitamente.

- [ ] **CP4.8 — Migrar las implementaciones de repositorio.** Si las interfaces del CP1.3
      estaban bien definidas, nada fuera de la capa de repositorio debería cambiar. Si
      cambia mucho, es información valiosa para el documento.

- [ ] **CP4.9 — DER físico por servicio.** Con tipos, claves y restricciones reales.

- [ ] **CP4.10 — Sistema de design tokens.** Variables CSS para colores, tipografías,
      espaciados y bordes. Esto **es** lo que el enunciado llama design tokens. Escribirlo,
      no heredarlo de un framework.

- [ ] **CP4.11 — Maquetado de las pantallas.** Implementar los bocetos del CP1.12 en HTML
      semántico.

- [ ] **CP4.12 — Responsive.** Media queries con breakpoints móvil / tablet / desktop.
      Probar que ningún componente se superponga ni se distorsione.

- [ ] **CP4.13 — Accesibilidad WCAG AA.** Atributos ARIA, foco visible, contraste
      suficiente, HTML semántico, hit área mínima de 48×48 px. Verificar con una
      herramienta automática y dejar constancia.

- [ ] **CP4.14 — Navegación de máximo tres niveles.** Mapear cada funcionalidad clave y
      contar los clics desde el inicio. Si alguna queda a cuatro, hay que rediseñar.

- [ ] **CP4.15 — Microcopy y estados.** Tooltips, placeholders, onboarding hints, toasts
      para resultados y errores, y loaders o skeletons para todo lo que tarde más de 300 ms.

- [ ] **CP4.16 — Justificaciones.** Consideraciones de diseño relacional, desnormalizaciones
      con su motivo, y la respuesta a la pregunta de discusión (NoSQL en Logística).

- [ ] **CP4.17 — Cierre.** Merge, tag `entrega-4`.

---

### 4.5 Entregable 5

**Qué hay que entregar:**

- [ ] Implementación del cliente liviano desacoplado, renderizando las vistas
- [ ] *(Bonus)* SSO con social login

> ⚠️ **A confirmar con el profesor.** El enunciado pide motor de plantillas y SSR, y la
> pregunta de discusión es SSR vs SPA. El profesor dijo que el front es a elección. Hay que
> preguntar explícitamente: **¿se puede hacer SPA con React, o tiene que ser SSR con
> template engine?** Los checkpoints de abajo asumen SSR con Thymeleaf.

**Checkpoints:**

- [ ] **CP5.1 — Módulo front.** Proyecto Spring Boot + Thymeleaf, puerto 8080, **sin lógica
      de negocio propia**: solo consume las APIs.

- [ ] **CP5.2 — Cliente HTTP hacia los servicios.** `RestClient` con manejo de errores y
      timeouts. Un servicio caído no puede tirar abajo el front.

- [ ] **CP5.3 — Módulo auth.** Quinto módulo, puerto 8085. Gestión de tokens desde el
      servidor frontend hacia los servicios de aplicación.

- [ ] **CP5.4 — Validación de tokens en cada servicio.** Que los servicios de dominio
      rechacen pedidos sin token válido.

- [ ] **CP5.5 — Login y registro.** Para donantes, entidades beneficiarias y
      administradoras.

- [ ] **CP5.6 — Landing pública.** Propósito y objetivos, donaciones destacadas del último
      mes, enlaces a fotos sin login, a registro/login y a información legal y de privacidad.

- [ ] **CP5.7 — Mapa de donaciones entregadas.** Leaflet con marcadores clickeables que
      muestran detalles. Accesible sin login.

- [ ] **CP5.8 — Vistas de donante.** Filtro de donaciones por estado y categoría o
      subcategoría, navegación por entidades, sección de incentivos con misiones e
      insignias.

- [ ] **CP5.9 — Vistas de entidad beneficiaria.** Registro de necesidades, estado de
      donaciones asignadas, confirmación de recepción con carga de fotos.

- [ ] **CP5.10 — Vistas de administrador.** Registro de donantes y donaciones, marcado de
      vencidas, selección de entidad final a partir de los rankings, administración de
      camiones, ranking mensual e histórico, importación CSV.

- [ ] **CP5.11 — Seguimiento en vivo.** Mapa con camiones, ubicación y última
      actualización, para donante y entidad. Consume el SSE del CP3.12.

- [ ] **CP5.12 — Justificaciones.** Respuesta a la pregunta de discusión (SSR vs SPA).

- [ ] **CP5.13 — Cierre.** Merge, tag `entrega-5`.

---

### 4.6 Entregable 6

**Qué hay que entregar:**

- [ ] Diagrama de despliegue de la solución final
- [ ] Implementación de observabilidad, monitoreo, seguridad y control
- [ ] Implementación de un web service con gRPC **o** GraphQL
- [ ] Propuesta en diagrama e implementación de arquitectura de microservicios

**Checkpoints:**

- [ ] **CP6.1 — Dockerfile por servicio.** Imágenes que buildeen y corran.

- [ ] **CP6.2 — Compose completo.** Todos los servicios + Postgres + Mongo + RabbitMQ
      levantando juntos en local.

- [ ] **CP6.3 — Elegir proveedor cloud.** Comparar opciones con capa gratuita y dejar la
      decisión justificada.

- [ ] **CP6.4 — Desplegar.** Sistema accesible públicamente. Variables de entorno y
      secretos gestionados por el proveedor, **nunca en el repo**.

- [ ] **CP6.5 — Métricas.** Actuator + Micrometer, expuestas y recolectadas.

- [ ] **CP6.6 — Logs centralizados.** Logs estructurados de todos los servicios en un solo
      lugar.

- [ ] **CP6.7 — Trazas distribuidas.** Poder seguir un pedido que atraviesa varios
      servicios. Es lo que más se nota cuando algo falla en producción.

- [ ] **CP6.8 — Dashboard de observabilidad.** Métricas, trazas y logs visualizables.

- [ ] **CP6.9 — Health checks y autorestart.** Endpoints de salud y política de reinicio
      automático ante fallos.

- [ ] **CP6.10 — Rate limiting.** Límite de solicitudes con respuesta `429` correcta.

- [ ] **CP6.11 — gRPC o GraphQL.** Elegir uno e implementarlo. gRPC para comunicación entre
      servicios en tiempo real; GraphQL en Donaciones o Logística para filtros dinámicos y
      selección de campos. La pregunta de discusión es sobre esta elección.

- [ ] **CP6.12 — Bonus: microservicios.** Service Registry + API Gateway.

- [ ] **CP6.13 — Bonus: Transactional Outbox.** Persistencia confiable de los eventos de
      notificación generados tras una escritura transaccional.

- [ ] **CP6.14 — Diagrama de despliegue final.**

- [ ] **CP6.15 — Documento de cierre.** Recorrido por todas las decisiones de diseño del
      TP y las seis preguntas de discusión respondidas.

- [ ] **CP6.16 — Cierre.** Merge, tag `entrega-6`.

---

## 5. Decisiones pendientes

Cosas que hay que resolver y todavía no están definidas.

| # | Decisión | Cuándo se necesita | Estado |
|---|---|---|---|
| 1 | Fechas reales de cada entrega | Ya | ⬜ Preguntar al profesor |
| 2 | ¿SPA con React o SSR con template engine? | Entrega 5 (pero condiciona la 4) | ⬜ Preguntar al profesor |
| 3 | `groupId` y paquete base definitivos | Ya | ✅ `groupId` `ar.edu.utn.frba.dds.donatrack`; paquete base corto: `<servicio>` |
| 4 | Versión exacta de Spring Boot | Ya | ⬜ Ver start.spring.io |
| 5 | ¿Herencia o composición para donante humano/jurídico? | Entrega 1 (impacta en la 4) | ⬜ |
| 6 | ¿Necesidad recurrente y extraordinaria: dos clases o una con estrategia? | Entrega 1 | ⬜ |
| 7 | Direccionalidad Donaciones ↔ Incentivos | Entrega 2 | ⬜ |
| 8 | Proveedor de SMS/WhatsApp | Entrega 2 | ⬜ |
| 9 | ¿GPS dedicado o app móvil? | Entrega 3 | ⬜ |
| 10 | Direccionalidad Donaciones ↔ Logística | Entrega 3 | ⬜ |
| 11 | Estrategia de mapeo de herencia en JPA | Entrega 4 | ⬜ |
| 12 | ¿Módulo `contracts` compartido o DTOs propios por servicio? | Entrega 2/3 | ⬜ |
| 13 | Proveedor cloud | Entrega 6 | ⬜ |
| 14 | ¿gRPC o GraphQL? | Entrega 6 | ⬜ |

---

## 6. Anexo — Links de interés

**Modelado UML**
- Diagrama de Casos de Uso — sparxsystems.com
- Diagrama de Clases — plantuml.com/es/class-diagram
- Diagrama de Componentes — sparxsystems.com
- Diagrama de Despliegue — sparxsystems.com
- StarUML — staruml.io

**Metodología de trabajo**
- Git Flow (documento de cátedra)

**Diseño y maquetado web**
- Diseño y Maquetado Web, Partes I y II (material de cátedra)
- Seminario UX/UI (material de cátedra)
- Heurísticas de Nielsen — uifrommars.com
- Laws of UX — lawsofux.com/es
- Web Content Accessibility Guidelines — w3.org/TR/WCAG21
- Material Design — m2.material.io/components
- Human Interface Guidelines — developer.apple.com/design/human-interface-guidelines

**Leyes argentinas**
- Ley 26.653 de Accesibilidad en Páginas Web
- Ley de Protección de los Datos Personales

**Recursos del enunciado**
- Diagrama de despliegue inicial (Drive de la cátedra)
- Diagrama de estados de una donación (Drive de la cátedra)
- Archivo CSV de prueba, 20.000 filas (Drive de la cátedra)
- Servicio externo de planificación de rutas (Drive de la cátedra)

**Conceptos técnicos**
- Distancia de Levenshtein — Wikipedia
- CRUD — developer.mozilla.org
- Transactional Outbox — microservices.io
- gRPC — grpc.io
- GraphQL — graphql.org
