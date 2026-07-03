# Genie in a Box · 25 minutos — Prompts de la demo (Grupo Cóndor)

> Demo 100% dentro del universo Genie: **Genie Code** (command center) → **Genie One / Genie
> Agent Chat mode** → **Genie Agent mode** (informe profundo). Sin notebooks, sin SQL editor,
> sin CLI, sin DABs.
>
> Todos los prompts usan el catálogo `main` — está disponible por defecto en todo workspace
> de Databricks, no hace falta crearlo. El schema `grupo_condor` se crea automáticamente
> dentro de `main`. Usa cualquier SQL Warehouse Serverless disponible.
>
> Empresa ficticia: **Grupo Cóndor** — conglomerado LATAM con 3 unidades de negocio:
> 🛒 **Cóndor Retail** (tiendas) · 💳 **Cóndor Pagos** (crédito de consumo) · 🚚 **Cóndor Cargo**
> (logística). Cualquier empresa de la audiencia se reconoce en al menos una unidad.
>
> Todos los prompts y tiempos de este documento fueron validados de punta a punta antes de
> publicarse.

---

## Regla de oro para que la demo no se alargue

Genie Code se detiene a pedir confirmación cada vez que encuentra una ambigüedad de
infraestructura o de alcance. En una demo de tiempo acotado **no hay margen para negociar con
el agente en vivo** — cada prompt debe cerrar de antemano toda ambigüedad que pueda generar
una pausa:

- **Catálogo**: la mayoría de los workspaces no dan permiso para crear catálogos nuevos desde
  cero. Usa siempre `main` (viene por defecto en todo workspace) en vez de pedirle al agente
  que cree uno.
- **Modo de escritura**: al generar tablas nuevas, indica explícitamente "sin overwrite" —
  de lo contrario el agente puede escribir código con `mode("overwrite")`, lo cual activa un
  chequeo de seguridad que bloquea la ejecución y detiene el flujo.
- **Decisiones de diseño**: si le pides algo que puede no ser posible tal cual (por ejemplo,
  combinar tablas de distinta granularidad en una sola Metric View), incluye una cláusula de
  salida en el mismo prompt ("si no se puede combinar en una, crea varias"). Sin esa cláusula,
  el agente se detiene a preguntar en vez de decidir solo.

Cuando el prompt cierra estas rutas de escape, Genie Code corre de principio a fin sin
preguntar nada.

---

## BLOQUE 1 — Genie Code (command center) · pre-armado antes del evento

> **Decisión de formato**: construir el dataset completo en vivo toma varios minutos. Para
> una demo corta, el dataset se arma **antes** del evento (estilo "cooking show") y en vivo
> solo se muestra un prompt corto de extensión — así el público ve al agente trabajar sin
> gastar minutos valiosos esperando la carga inicial de datos.

### Prompt 1 · Dataset base (ejecutar ANTES del evento)

```
Crea el schema `main.grupo_condor` (usa el catálogo `main`, que ya existe por defecto en el
workspace — NO crees catálogos nuevos) y genera datos sintéticos realistas para Grupo
Cóndor, un conglomerado latinoamericano
(países: MX, CO, PE, CL, AR, EC; moneda USD; fechas entre 2025-01-01 y 2026-06-30; usa seed
fija para reproducibilidad; las tablas no existen aún — créalas nuevas, sin mode overwrite):

1. dim_clientes — 2000 clientes (cliente_id, nombre, pais, ciudad, segmento B2C/B2B, fecha_alta)
2. dim_tiendas — 120 tiendas de Cóndor Retail (tienda_id, pais, ciudad, formato SUPER/EXPRESS/ONLINE)
3. fact_ventas — 50000 ventas retail (venta_id, cliente_id, tienda_id, fecha, categoria entre
   ELECTRO/ALIMENTOS/HOGAR/MODA, monto_usd, canal TIENDA/ONLINE)
4. fact_creditos — 8000 créditos de consumo de Cóndor Pagos (credito_id, cliente_id,
   fecha_originacion, monto_usd 100-5000, plazo_meses 3-24, status ACTIVO/CANCELADO/MOROSO,
   dias_atraso 0-120 coherente con status)
5. fact_envios — 30000 envíos de Cóndor Cargo (envio_id, venta_id, fecha_despacho,
   fecha_entrega, estado ENTREGADO/EN_RUTA/RETRASADO, costo_envio_usd, pais_destino)

Reglas de negocio: el monto total de ventas ELECTRO en Chile en Q1-2026 debe ser ~35% menor
que en Q4-2025, con las demás categorías y países estables (anomalía para la demo); clientes
morosos concentrados en el segmento B2C con créditos > USD 3000; ~8% de envíos RETRASADO,
más frecuentes en Argentina. Valida al final con un SELECT de conteo por tabla.
```

**Resultado esperado**: 2.000 / 120 / 50.000 / 8.000 / 30.000 filas. La regla de ELECTRO en
Chile queda validada por el propio agente al final del proceso.

### Prompt 2 · Capa gold

```
Sobre main.grupo_condor crea la capa gold del Grupo Cóndor (tablas nuevas, sin
overwrite):
1. gold_ingresos_diarios — ingresos por fecha, pais, unidad_negocio (RETAIL = fact_ventas.monto_usd;
   PAGOS = interés mensual estimado de fact_creditos activos = monto_usd * 0.03; CARGO =
   fact_envios.costo_envio_usd)
2. gold_cartera_creditos — snapshot por crédito: saldo estimado, dias_atraso, rango_mora
   (Al dia / 1-29 / 30-59 / 60-89 / 90+)
3. gold_otif_envios — % de entregas a tiempo (fecha_entrega <= fecha_despacho + 5 días) por
   mes, pais_destino y estado
4. gold_ventas_categoria — ventas de Cóndor Retail agregadas por fecha, pais, categoria
   (ELECTRO/ALIMENTOS/HOGAR/MODA): SUM(monto_usd) as ingresos_usd, COUNT(*) as num_ventas
Valida cada tabla con un SELECT de 5 filas y no me pidas aprobaciones intermedias.
```

> ⚠️ La tabla `gold_ventas_categoria` es imprescindible: sin la dimensión `categoria`, el
> Genie Space no puede responder preguntas por línea de producto (ver Bloque 2, pregunta 2).

### Prompt 3 · Metric Views

```
Crea las Metric Views de main.grupo_condor usando CREATE VIEW ... WITH METRICS
LANGUAGE YAML:
1. metrics_grupo sobre gold_ingresos_diarios — dimensions pais, unidad_negocio, mes (yyyy-MM);
   measure ingresos_usd = SUM(ingreso)
2. metrics_cartera sobre gold_cartera_creditos — measure tasa_morosidad_pct = saldo con
   dias_atraso >= 30 / saldo total * 100
3. metrics_envios sobre gold_otif_envios — measure otif_pct = % entregas a tiempo
4. metrics_retail sobre gold_ventas_categoria — dimensions pais, categoria, mes (yyyy-MM);
   measures ingresos_usd = SUM(ingresos_usd), ticket_promedio_usd = SUM(ingresos_usd)/SUM(num_ventas)
No combines todo en una sola vista — la granularidad de cada tabla gold es distinta. Valida
cada una con SELECT ... GROUP BY pais usando MEASURE(), y confirma que metrics_retail muestra
la caída de ELECTRO en Chile en Q1-2026 vs Q4-2025. No me pidas aprobaciones intermedias.
```

**Resultado esperado**: 4 Metric Views (el agente decide solo dividir en varias vistas por
granularidad, gracias a la cláusula de fallback del prompt). Validación esperada: ELECTRO
Chile cae ~35-38% de Q4-2025 a Q1-2026.

### Prompt EN VIVO · lo único que se ejecuta frente al público

```
Sobre main.grupo_condor, agrega a gold_otif_envios una columna que calcule el costo
total de envíos retrasados por país (RETRASADO), y actualiza metrics_envios con una nueva
measure costo_retrasos_usd = SUM(costo_envio_usd) filtrado por estado='RETRASADO'. Valida con
un SELECT agrupado por pais_destino. No me pidas aprobaciones intermedias.
```

**Mensaje al presentar mientras corre**: "esto que están viendo ahora — un agente escribiendo,
ejecutando y validando SQL sobre Databricks, extendiendo un modelo de datos ya en producción —
es Genie Code. Todo lo que van a preguntar después ya estaba armado así, minutos antes de
subir al escenario."

### Prompt 4 · Crear el Genie Space (ejecutar ANTES del evento)

> Validado: Genie Code SÍ puede crear un Genie Space nuevo por prompt — título, warehouse y
> fuentes de datos (Metric Views) quedan configurados correctamente sin intervención manual.
> ⚠️ Limitación real encontrada: **las General Instructions NO se pueden fijar por prompt** —
> las escrituras del agente caen en un campo interno (`description`) que no es el que usa el
> motor NL→SQL. Después de este prompt, pega el texto de la sección "Instructions del Genie
> Space" (más abajo) manualmente en **Configure → Instructions → Text → General Instructions**,
> y verifica visualmente que quedó ahí — no confíes en que el agente lo haya hecho.

```
Crea un nuevo Genie Space en Databricks (un AI/BI Genie Space real, no un chat de Genie Code)
llamado "Grupo Cóndor - KPIs Corporativos", con descripción "Asistente analítico del Grupo
Cóndor". Agrega como fuentes de datos las Metric Views main.grupo_condor.metrics_grupo,
metrics_cartera, metrics_envios y metrics_retail. Usa el SQL Warehouse Serverless disponible.
Al final, dime explícitamente si pudiste configurar las General Instructions vía API o si
quedó pendiente de configuración manual en la UI.
```

**Resultado esperado**: Space creado con las 4 Metric Views como fuente (verificable en
Configure → Data). El propio agente debería reportar que las Instructions quedaron
pendientes — si dice lo contrario, verifícalo tú igual.

---

## BLOQUE 2 — Genie Space en modo Chat (rápido, NL→SQL directo)

> Usar el modo **Chat** (no Agent) para estas dos preguntas — es el modo NL→SQL estándar,
> con respuestas típicas en menos de un minuto. El modo Agent (razonamiento multi-step) es
> notablemente más lento incluso para preguntas simples — resérvalo solo para el informe
> profundo del Bloque 3. Cambiar de modo es manual, con el selector Agent/Chat junto al
> cuadro de preguntas.

1. `¿Cuáles fueron los ingresos totales del Grupo Cóndor por unidad de negocio?`
   Responde con tabla + gráfico de barras usando metrics_grupo (RETAIL domina ampliamente
   sobre PAGOS y CARGO).
2. `¿Cómo evolucionaron las ventas de ELECTRO en Chile mes a mes en el último año?`
   Responde con serie mensual, mostrando la caída de enero-febrero 2026 con claridad — la
   anomalía queda visualmente evidente.

**Mensaje al hacer la pregunta 2**: dejar que la caída "aparezca sola" en el gráfico — es el
gancho narrativo hacia el informe del Bloque 3.

---

## BLOQUE 3 — Genie Agent mode: el informe profundo (el momento "wow")

> Cambiar manualmente a modo **Agent**. Esta pregunta puede tardar varios minutos —
> **planificar para hablar encima de la espera**: explicar Genie Ontology, contar el caso de
> uso, mostrar el "Thinking..." en pantalla como prueba de que está razonando de verdad, no
> cacheado.

```
Analiza en profundidad la caída de ventas de ELECTRO en Chile en Q1-2026: identifica si se
correlaciona con la morosidad de créditos de Cóndor Pagos o los retrasos de envíos de Cóndor
Cargo en el mismo país y período. Da hipótesis de causas probables y recomienda 3 acciones
concretas. Redacta un informe ejecutivo breve para el comité del Grupo Cóndor.
```

**Resultado esperado — informe completo** (ver `informe_ejemplo_electro_chile.md` para el
texto de referencia):
- Cruza las 3 Metric Views (retail, cartera, envíos) sin que se le pida explícitamente
- Descarta morosidad como causa (dato concreto, dentro de rango histórico)
- Descarta logística como causa (OTIF por encima del promedio del período anterior)
- Aísla el problema a la categoría (ticket promedio estable → es caída de volumen, no de precio)
- 3 hipótesis (saturación de mercado, competencia, macro) + 3 recomendaciones con
  responsable y plazo asignados
- Botones nativos "Download PDF" / "Share" al pie — el documento ejecutivo real de Genie One

**Mensaje al revelar el informe**: "esto no es un dashboard con reglas fijas — el agente
decidió solo qué mirar, descartó dos hipótesis con datos, y armó un informe con acciones y
responsables. Esto es Genie Ontology funcionando: contexto certificado + razonamiento."

---

## BLOQUE 4 — Cierre

- Mostrar "Crear agente" desde la conversación → 1 clic transforma el chat en un
  **Genie Agent** compartible (fka Genie Space): *"esta misma lógica queda disponible para
  todo el equipo, sin que tengan que saber SQL ni volver a explicarle el contexto al agente."*
- Narrativa Ontology: "acertó porque conoce el negocio — Metric Views certificadas +
  contexto aprendido. 84,5% de acierto a la primera vs 52,4% de agentes genéricos (dato
  público de Databricks)."
- CTA: pay-as-you-go desde 1-jul-2026, USD 10 gratis por usuario/mes (~80-100 sesiones
  Genie One) · Free Edition disponible para probar hoy mismo.

---

## BLOQUE OPCIONAL — Dashboard AI/BI hiper-personalizado con Genie Code

> Módulo adicional (no forma parte del guion base de 25 min) para audiencias que quieran ver
> a Genie Code construyendo un **dashboard Lakeview real** de punta a punta — datasets,
> widgets y hasta la identidad visual — sin tocar el editor manualmente. Prepararlo antes del
> evento; se puede mostrar como "bonus" si sobra tiempo, o mencionar solo el resultado final.

### Prompt A · Construir el dashboard base

```
Crea un dashboard AI/BI (Lakeview) real en Databricks (no un notebook) llamado "Grupo Cóndor
- Dashboard Ejecutivo", usando datos de main.grupo_condor. El dashboard debe tener:
1. Una fila superior con 4 KPI counters: ingresos totales del grupo, tasa de morosidad
   promedio, OTIF promedio, ticket promedio de ELECTRO
2. Un gráfico de barras horizontal: ingresos totales por país
3. Un gráfico de líneas: evolución mensual de ingresos por unidad de negocio (RETAIL/PAGOS/CARGO)
4. Un gráfico de dona: distribución de cartera de créditos por rango_mora
Usa las Metric Views existentes (metrics_grupo, metrics_cartera, metrics_envios,
metrics_retail) como fuente de datos. Aplica un layout de grid de 12 columnas, prolijo y con
buen espaciado. Publica el dashboard al finalizar. No me pidas aprobaciones intermedias.
```

> ⚠️ Lección del test: el primer resultado suele venir **genérico** — números sin formatear,
> dimensiones ricas del dataset sin usar (ej. categoría de producto), y gráficos que comparan
> magnitudes muy distintas en la misma escala (una unidad de negocio 15x más grande que otra
> vuelve ilegibles a las demás). No es falta de datos — es que el primer prompt no pidió
> explotar esas dimensiones. Un segundo prompt de refinamiento es casi siempre necesario:

```
Corrige y enriquece el dashboard:
1. Verifica que los counters numéricos muestren formato compacto legible (ej: $7,8M), sin
   decimales de más ni errores de punto flotante — envuelve las expresiones en ROUND().
2. Agrega widgets que usen la dimensión categoría de producto de metrics_retail: evolución
   mensual de la categoría con mayor variación, y una tabla comparativa por categoría y país
   ordenada por variación % ascendente.
3. Si algún gráfico compara magnitudes muy distintas entre series (ej. unidades de negocio de
   tamaño muy dispar), conviértelo a porcentaje/composición en vez de valores absolutos.
4. Excluye del análisis mensual el mes en curso si está incompleto (evita barras/puntos
   distorsionados al final de la serie).
Verifica cada cambio con una query antes de darlo por hecho. Publica al finalizar.
```

**Nota operativa importante**: en el test, el botón "Publish" del editor de dashboards (beta)
resultó poco confiable — a veces requiere más de un intento, y en un caso Genie Code intentó
publicar vía API y fue bloqueado por guardrails de seguridad internos (protección contra
mutaciones no autorizadas de assets del workspace). **Siempre verifica manualmente la URL
publicada antes de la demo en vivo** — no confíes en el mensaje de confirmación del agente.

### Prompt B · Identidad visual 100% personalizada

> Validado: Lakeview soporta un **tema completamente custom** (no solo paletas predefinidas) —
> color de fondo del canvas y widgets, bordes, ejes/grid, color de selección, paleta
> categórica ilimitada, tipografía (familia/tamaño/peso/color) por título y valor, padding/
> radius/shadow de widgets, y gradientes continuos para heatmaps. Lo único que NO soporta es
> CSS arbitrario o imágenes de fondo. Se configura en **Settings → Theme** del dashboard.

Ejemplo genérico (reemplaza colores por la identidad de marca del cliente o del evento):

```
Personaliza la identidad visual de este dashboard aplicando estos colores en Settings > Theme
> Color palette:
- Color primario / títulos de widgets: {color 1, ej. #E52521}
- Bordes de widgets: {color 2, ej. #049CD8}
- Fondo del canvas: {color 3 claro, ej. #E4F6FF}
- Paleta categórica de gráficos: {lista de 4-8 colores hex}
Aplica también tipografía consistente (ej. Poppins) y esquinas redondeadas en los widgets.
No uses CSS custom ni imágenes externas — solo las opciones nativas del editor de tema.
```

Como prueba de concepto se validó con una paleta temática de **Super Mario Bros** (rojo
`#E52521`, azul `#049CD8`, amarillo `#FBD000`, verde `#43B047`, fondo cielo `#E4F6FF`) — el
resultado fue un dashboard completamente re-skinned (fondo, bordes, títulos y los 4 colores
reflejados en cada gráfico) sin tocar una sola línea de CSS. Para una demo con clientes,
reemplaza esa paleta por los colores corporativos del cliente o del evento — el mecanismo es
idéntico.

---

## Troubleshooting

| Síntoma | Causa raíz | Fix |
|---|---|---|
| Genie Code se detiene pidiendo confirmación | Intentó crear un catálogo nuevo y no tiene permiso | Nombrar siempre el catálogo existente en el prompt |
| Bloqueo "Code execution blocked" | El código generado usa `mode("overwrite")` | Especificar "tablas nuevas, sin overwrite" en el prompt |
| El Genie Space responde "no existe columna X" | La capa gold no incluye la dimensión que la pregunta necesita | Revisar que todas las dimensiones de negocio relevantes estén en alguna tabla gold / Metric View |
| Las respuestas no siguen las reglas de negocio esperadas | Las instructions no se guardaron en el lugar correcto | Verificar en **Configure → Instructions → Text → General Instructions** — no confundir con el campo "Description" de la pestaña About |
| Una pregunta en modo Agent tarda varios minutos incluso siendo simple | El modo Agent hace razonamiento multi-step por diseño — no es para Q&A rápido | Usar Chat mode para preguntas rápidas; reservar Agent solo para el informe de cierre |
| Genie Code crea el Genie Space pero las Instructions "no pegaron" | Solo puede escribir el campo `description` (About), no el campo real de Instructions | Pegar las Instructions manualmente en Configure → Instructions → Text — SIEMPRE verificar visualmente |
| Counters/tablas del dashboard muestran números con muchos decimales | `SUM()` sobre columnas DOUBLE genera errores de punto flotante | Envolver la expresión en `ROUND(..., n)` o `CAST(... AS DECIMAL(18,2))` en el dataset SQL |
| El botón "Publish" del editor de dashboards no responde al primer clic | Comportamiento inconsistente del editor (beta) | Reintentar el clic; verificar la URL publicada manualmente antes de confiar en la confirmación del agente |
| Genie Code reporta "blocked by IP access list" o "blocked by safety guardrails" al intentar publicar/mutar un asset vía API | Protecciones internas contra mutaciones no autorizadas de infraestructura compartida | Es un límite esperado, no un bug a resolver — publica manualmente desde la UI cuando esto ocurra |

---

## Instructions del Genie Space (texto listo para copiar/pegar)

Ubicación correcta: **Configure → pestaña Instructions → sub-pestaña Text → "General
Instructions"**. No es lo mismo que el campo "Description" de la pestaña "About".

```
Eres el analista corporativo del Grupo Cóndor (conglomerado LATAM: Cóndor Retail, Cóndor
Pagos, Cóndor Cargo). Moneda siempre USD, nunca conviertas.
Convenciones:
- "morosidad" = tasa_morosidad_pct de la metric view metrics_cartera (dias_atraso >= 30)
- "OTIF" o "entregas a tiempo" = otif_pct de metrics_envios
- "ingresos" a nivel corporativo/unidad de negocio = ingresos_usd de metrics_grupo
- preguntas sobre categorías de producto (ELECTRO, ALIMENTOS, HOGAR, MODA) = usa
  metrics_retail (dimensiones pais, categoria, mes)
Periodos en formato YYYY-MM; si no se especifica, usa los últimos 12 meses disponibles.
Prioriza SIEMPRE las Metric Views (metrics_grupo, metrics_cartera, metrics_envios,
metrics_retail) para cualquier KPI; nunca calcules manualmente desde las tablas gold u
origen si la metric view ya lo cubre.
```
