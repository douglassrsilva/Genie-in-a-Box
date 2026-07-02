# Genie in a Box

Una demo guiada de **Genie Code**, **Genie One / Genie Agents** y **Genie Ontology** en
Databricks — construida enteramente con prompts en lenguaje natural, sin escribir SQL a mano,
sin notebooks y sin scripts de infraestructura.

En menos de media hora vas a:

1. Generar un dataset sintético completo (clientes, ventas, créditos, envíos) con **Genie
   Code**, el asistente agéntico de Databricks.
2. Construir capas gold y **Metric Views** — la capa semántica gobernada de Databricks.
3. Hacerle preguntas de negocio en español a un **Genie Space** (chat NL→SQL) y ver cómo
   descubre una anomalía real en los datos.
4. Pedirle a **Genie Agent** (modo de razonamiento profundo) que cruce varias fuentes y
   redacte un informe ejecutivo con hipótesis y recomendaciones.

Todo el contenido de la demo — datos, empresa ficticia, escenario de negocio — es sintético.

## Qué hay en este repo

| Archivo | Contenido |
|---|---|
| [`prompts_demo.md`](./prompts_demo.md) | Los prompts exactos, en orden, listos para copiar/pegar en Genie Code y en el Genie Space. Incluye los resultados esperados, una tabla de troubleshooting y el texto de las *General Instructions* del Space. |

## Prerrequisitos

- **Un workspace de Databricks con Unity Catalog habilitado.** Si tu organización ya usa
  Databricks, pídele a tu administrador acceso a un workspace con permisos para crear
  schemas, tablas y Genie Spaces.
- **Un SQL Warehouse Serverless** disponible (cualquiera sirve; se crea uno por defecto en
  la mayoría de los workspaces).
- **Genie Code** y **Genie Spaces / Genie Agents** habilitados en el workspace (son parte de
  la plataforma estándar de Databricks; en algunos workspaces pueden requerir que el
  administrador los active desde la consola de administración).

No hace falta instalar nada localmente ni usar la CLI de Databricks — toda la demo corre
dentro del navegador.

### No tenés un workspace de Databricks todavía

Databricks ofrece dos formas de empezar gratis:

**Opción A — Free Edition (recomendada para explorar esta demo)**
- Sin tarjeta de crédito, sin límite de expiración, pensada para aprendizaje y uso personal.
- Andá a [databricks.com/try-databricks](https://www.databricks.com/try-databricks) y elegí
  **"Get Free Edition"**.
- Registrate con tu email personal y seguí el asistente de creación de cuenta.
- Incluye un workspace serverless — suficiente para correr todos los prompts de este repo.
- Tiene límites diarios de uso y no incluye algunas funciones enterprise; si algo de la
  demo no está disponible en tu cuenta, probá la Opción B.

**Opción B — Trial de 14 días (plataforma completa)**
- Acceso completo a Databricks por 2 semanas, con crédito de evaluación incluido, conectado
  a tu propia cuenta de nube (AWS, Azure o GCP).
- Andá a [databricks.com/try-databricks](https://www.databricks.com/try-databricks) y elegí
  **"Try Databricks"**.
- Usá tu email corporativo para el registro (da mejores condiciones de evaluación).
- Vas a necesitar una cuenta activa en AWS, Azure o GCP para desplegar el workspace (el
  asistente de Databricks te guía paso a paso).

## Cómo usar este repo

1. Abrí [`prompts_demo.md`](./prompts_demo.md).
2. Seguí los bloques en orden: primero los prompts de **Genie Code** (Bloque 1), después las
   preguntas al **Genie Space en modo Chat** (Bloque 2), y por último la pregunta en **modo
   Agent** que genera el informe ejecutivo (Bloque 3).
3. Todos los prompts usan el catálogo `main`, disponible por defecto en cualquier workspace
   — no hace falta crear nada de infraestructura antes de empezar.
4. Si algo no responde como se espera, revisá la sección de **Troubleshooting** al final del
   documento — cubre los problemas más comunes (permisos de catálogo, modo de escritura,
   dimensiones faltantes en el modelo de datos).

## Licencia / uso

Contenido de demo con fines educativos y de evaluación de producto. Los datos son 100%
sintéticos.
