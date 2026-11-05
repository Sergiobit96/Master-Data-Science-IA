# Actividad 0 · Equipo e ideas de producto

## Equipo

| Integrante | Usuario de GitHub |
|---|---|
| Alejandro Ronda Navazo | [@arondax](https://github.com/arondax) |
| Sergio _(apellidos)_ | _@usuario_ |
| Susana Portela García de Blas | [_@Sportelagdb_](https://github.com/Sportelagdb) |

Repositorio: _enlace al repositorio privado_

## Resumen de las tres propuestas

| | 1 · Precio de la electricidad | 2 · Efectos adversos de fármacos | 3 · Incidencias en Cercanías |
|---|---|---|---|
| Usuario principal | Comercializadoras y agentes del mercado | Investigadores y profesionales farmacéuticos | Viajero habitual de Cercanías Madrid |
| Tarea de ML | Regresión de series temporales (D+1) | Clasificación multietiqueta | Clasificación binaria por línea y franja |
| Datos | Abiertos, con histórico desde 2019 | APIs públicas; el cruce entre fuentes es difícil | Abiertos, pero el histórico hay que capturarlo |
| Riesgo principal | Tema muy trabajado; el cambio a 15 min | Pocos ejemplos por grupo terapéutico | Profundidad del histórico propio |

---

## Idea 1 · Previsión del precio de la electricidad en el mercado diario

### Título y descripción

**Nombre provisional: previsión del precio del mercado diario español.**

Sistema que predice el precio del mercado diario español con 24–48 horas de antelación y compara modelos estadísticos, de machine learning y de deep learning. Tomamos como referencia el protocolo de evaluación de Lago et al. (2021, *Applied Energy*), que no se ha aplicado al mercado ibérico. El proyecto aprovecha dos cambios recientes que la literatura todavía no recoge:

- **Precio cada 15 minutos (MTU15):** desde el 1 de octubre de 2025 el mercado fija un precio por cuarto de hora, 96 valores al día en lugar de 24.
- **Precios cero y negativos:** son cada vez más frecuentes desde abril de 2024.

### Problema y usuario

El precio mayorista español es de los más volátiles de Europa: en un mismo día puede pasar de 0 €/MWh a mediodía a más de 200 €/MWh por la noche. Depende de la producción renovable, la demanda, el gas y el CO₂.

**Usuarios:**

- **Comercializadoras:** compran en el mercado diario y necesitan estimar su coste para fijar tarifas y cubrirse.
- **Productores renovables y operadores de baterías:** deciden cuándo vender o almacenar.
- **Grandes consumidores industriales:** desplazan los procesos intensivos a las horas baratas.
- **Hogares con PVPC:** su precio incorpora el del mercado mayorista.

**Cómo lo resuelven hoy:** con previsiones propias o de proveedores especializados. Los benchmarks públicos de referencia son horarios y no incluyen el mercado ibérico.

### Uso y valor

> Por la mañana, antes del cierre de la subasta del mercado diario, una analista de una comercializadora consulta la previsión de precio para cada cuarto de hora del día siguiente, con su intervalo de incertidumbre, y las franjas con riesgo de precio cero o negativo. Con ello ajusta las ofertas de compra y decide qué parte del consumo previsto cubrir.

**Valor:** una previsión más precisa reduce el coste de compra y el riesgo de cobertura. Además, explicamos qué factores mueven el precio en cada franja.

### Datos

| Fuente | Variables | Resolución | Acceso | Estado |
|---|---|---|---|---|
| OMIE | Precio del mercado diario (variable objetivo) | Horaria; 15 min desde oct. 2025 | Ficheros públicos; paquete `omie-market-data` | Acceso público documentado; pendiente de descargar una muestra |
| ESIOS (Red Eléctrica) | Previsión de demanda, eólica y solar; generación real por tecnología | Horaria / 15 min | API con token, que se solicita a consultasios@ree.es | Pendiente de obtener el token |
| REData (Red Eléctrica) | Mix, demanda y PVPC agregados | Horaria / diaria | API sin token | Acceso público documentado |
| MIBGAS | Precio del gas en el mercado ibérico | Diaria | Público | Acceso público documentado |
| Mercado de CO₂ (EUA) | Precio de los derechos de emisión | Diaria | Fuentes de mercado | **Hipótesis:** falta identificar una fuente abierta y sus condiciones de uso |
| Open-Meteo / AEMET | Temperatura, radiación, viento | Horaria | Gratuito; AEMET con clave | Acceso público documentado |
| Calendario | Festivos, día de la semana, horario de verano | — | Construcción propia | — |

**Periodo propuesto:** de enero de 2019 hasta la fecha, para cubrir varios regímenes de precios.

**Restricciones:** no hay datos personales. Falta revisar las condiciones de uso de cada fuente y los cambios de hora (días de 23 y 25 horas).

### Primera versión (MVP)

**Qué mostraríamos funcionando**

1. Un dataset integrado y reproducible de precio, demanda, previsiones renovables, gas y calendario.
2. Previsión D+1 a resolución horaria, con dos referencias (un modelo ingenuo y LEAR) y al menos un modelo de ML, evaluados con ventanas temporales.
3. Una vista con la previsión del día siguiente, su error histórico y las franjas con riesgo de precio cero o negativo.

**Qué dejamos fuera por ahora**

- Resolución de 15 minutos.
- Deep learning (DNN, N-HiTS, TFT).
- Predicción probabilística completa.
- Interpretabilidad detallada.

### Viabilidad y dudas

- **Datos:** abundantes y abiertos. La dependencia crítica es el token de ESIOS.
- **Fugas de información:** para no hacer trampa hay que usar las **previsiones** de renovables, demanda y meteorología que existían en su momento, no los valores observados. Las previsiones meteorológicas históricas son más difíciles de conseguir que las observaciones.
- **Cambio de resolución:** el paso a 15 minutos en octubre de 2025 parte la serie en dos regímenes. Hay que decidir si se predice en horario y se desagrega, o directamente en 15 minutos.
- **Originalidad:** es un tema muy trabajado. Lo que aporta el proyecto es el mercado ibérico, MTU15 y los precios negativos.
- **Modelos:** hay que comprobar qué modelos (LEAR, DNN, gradient boosting) se imparten en el módulo de ML, porque la actividad 4 pide modelos vistos en clase.

---

## Idea 2 · Predicción de efectos adversos de medicamentos a partir de información farmacológica

### Título y descripción

**Nombre provisional: previsión de efectos secundarios**

Sistema que predice los posibles efectos adversos asociados a un principio activo mediante técnicas de Machine Learning. Usaríamos la información de las fichas técnicas de medicamentos y otras fuentes farmacológicas públicas para entrenar un modelo que prediga varios efectos adversos a la vez.

### Problema y usuario

La información sobre los efectos secundarios de los medicamentos está repartida entre distintas bases de datos y fichas técnicas, así que comparar medicamentos y analizar sus posibles efectos adversos es laborioso.

**Usuario:** investigadores y profesionales del ámbito farmacéutico que necesiten analizar estas relaciones de forma más sencilla.

### Uso y valor

El usuario introduce el principio activo de un medicamento y obtiene una lista de posibles efectos adversos, ordenados según las puntuaciones del modelo. También puede consultar los efectos recogidos en la ficha técnica oficial para comparar las predicciones con la información disponible.

### Datos

Hay APIs públicas, documentadas y estables. Usaríamos tres fuentes principales:

- **CIMA (AEMPS):** medicamentos, principios activos y reacciones adversas de la sección 4.8 de las fichas técnicas.
- **ClinicalTrials.gov:** efectos adversos comunicados en ensayos clínicos.
- **PubChem:** características y estructuras químicas de los principios activos.

Aunque las fuentes son de acceso público, falta comprobar la disponibilidad y la calidad de los datos. Una dificultad será unificar los nombres de medicamentos y efectos adversos entre fuentes.

### Primera versión (MVP)

Acotaríamos el prototipo a un grupo terapéutico. El MVP incluiría:

- La extracción de los datos, su limpieza y su normalización.
- El entrenamiento de un modelo de clasificación multietiqueta.
- Una interfaz sencilla para consultar las predicciones.

Incorporaríamos fuentes complementarias si resulta viable.

### Viabilidad y dudas

La principal dificultad será relacionar correctamente los datos de las distintas fuentes y conseguir suficientes ejemplos para entrenar el modelo. Para comprobar la viabilidad haríamos una prueba inicial con cinco o diez principios activos: verificar que se pueden extraer sus efectos adversos y construir un conjunto de datos consistente. El resultado sería un prototipo de investigación, no una herramienta de diagnóstico ni de recomendación médica.

---

## Idea 3 · Previsión de incidencias en Cercanías Madrid

### Título y descripción

**Nombre provisional: Previsión de incidencias en Cercanías.**

Herramienta que estima, para cada línea y franja horaria del núcleo de Cercanías Madrid, la probabilidad de que haya retrasos o cancelaciones. El histórico de incidencias lo construimos capturando los feeds públicos en tiempo real de Renfe y lo cruzamos con el horario programado y la meteorología. Madrid es el primer alcance; el diseño permite ampliarlo después a otros núcleos y a la red nacional.

### Problema y usuario

**Usuario:** viajero habitual de Cercanías Madrid que hace el mismo trayecto en una franja fija (por ejemplo, Alcalá → Atocha entre 7:00 y 9:00).

**Necesidad:** saber con antelación si su línea y su franja tienen un riesgo alto de incidencias, para decidir si sale antes, cambia de ruta o de modo de transporte.

**Cómo lo resuelve hoy:** reacciona cuando la incidencia ya existe, con los avisos de Renfe (web o app) y las redes sociales, o tira de su experiencia. No conocemos ninguna fuente pública que ofrezca una estimación anticipada basada en el histórico.

### Uso y valor

> Martes, 22:00. Laura va mañana de Alcalá de Henares a Atocha en la C-2/C-7 a las 7:30. Abre la herramienta, elige línea, sentido y franja, y ve que la probabilidad estimada de incidencia es del 35 %, frente al 12 % habitual de esa franja, con lluvia fuerte prevista. También ve qué tramos han acumulado más retrasos este mes. Decide salir 20 minutos antes.

**Valor:** pasar de reaccionar a anticipar. Además, una visión histórica de la fiabilidad de su línea por franja que hoy no tiene en ningún sitio.

La herramienta no predice accidentes ni evalúa la seguridad de la infraestructura. Estima la probabilidad de incidencias operativas (retrasos y cancelaciones) a partir de datos públicos.

### Datos

| Fuente | Contenido | Formato y acceso | Estado |
|---|---|---|---|
| GTFS estático de Renfe Cercanías | Rutas, viajes, paradas, horarios y calendario de todos los núcleos | ZIP de descarga pública | **Confirmado.** Descargado e inspeccionado: 15 núcleos y 12 líneas en Madrid. Un jueves tipo tiene 1.291 viajes, 20.201 paradas programadas y 93 estaciones. Calendario vigente del 04/11/2026 al 03/12/2026. |
| Renfe GTFS-RT · *Horarios de viaje* (`trip_updates`) | Cancelaciones, cambios de horario, servicios añadidos y estaciones sin servicio, por viaje | `https://gtfsrt.renfe.com/trip_updates.pb` / `.json`; según la ficha se actualiza cada 20 s; CC BY 4.0 | **Acceso confirmado** (consulta manual del JSON el 04/11/2026). Incluye el campo `delay`, y el `trip_id` de una muestra (C1 de Sevilla) cruza con el GTFS estático. Pendiente: confirmar que aparecen viajes del núcleo de Madrid y que hay retrasos distintos de 0. |
| Renfe GTFS-RT · *Incidencias y avisos* (`alerts`) | Avisos de incidencias en vía, servicios por carretera, accesibilidad | `https://gtfsrt.renfe.com/alerts.pb` / `.json`; según la ficha se actualiza cada 20 s; CC BY 4.0 | **Acceso público documentado.** Pendiente: capturar una muestra real y comprobar la estructura de las líneas y tramos afectados. |
| Renfe GTFS-RT · *Ubicación de los vehículos* | Posición GPS y estado de cada tren | GTFS-RT / JSON en data.renfe.com | **Hipótesis.** Fuente secundaria si `trip_updates` no trae retrasos. |
| AEMET OpenData / Open-Meteo | Precipitación, viento y temperatura, observados y previstos | AEMET: API con clave gratuita. Open-Meteo: API abierta | **Hipótesis.** Pendiente: pedir la clave de AEMET y revisar las condiciones de uso. |
| CRTM · GTFS Red de Cercanías | Horario programado de Cercanías Madrid publicado por el Consorcio | Portal de datos abiertos del CRTM | **Revisado.** Es GTFS estático, no tiempo real. Sirve para contrastar el GTFS de Renfe, pero no aporta etiquetas. |

**Restricciones conocidas**

- **No hay histórico público de incidencias.** Las etiquetas salen de nuestra propia captura continua, que tiene que empezar cuanto antes.
- El GTFS estático tiene una vigencia de un mes y los `trip_id` cambian entre versiones. Hay que archivar cada versión con su fecha de descarga.
- Si el colector se cae, el hueco es **ausencia de datos**, no ausencia de incidencias. Registraremos el tiempo de actividad del colector.
- **Privacidad:** son datos operativos, sin información personal. Los feeds en tiempo real tienen licencia CC BY 4.0, que obliga a atribuir. La licencia del GTFS estático está pendiente de confirmar.

### Primera versión (MVP)

**Qué mostraríamos funcionando**

1. Un colector programado que guarda periódicamente `trip_updates` y `alerts` del núcleo de Madrid y registra los huecos de captura.
2. Una tabla de incidencias por **línea × sentido × franja de 1 h × día**. Incidencia (definición provisional): al menos una cancelación o un retraso de 5 minutos o más. El umbral se justificará con los datos.
3. Un baseline (la tasa histórica de incidencias por línea y franja) y un modelo que añada calendario y meteorología.
4. Una vista donde el usuario elige línea, sentido y franja, y ve la probabilidad estimada frente a la habitual y el histórico reciente.

**Qué dejamos fuera por ahora**

- Otros núcleos y la red nacional.
- Predicción por estación o por tramo entre estaciones.
- Notificaciones push.
- Cualquier análisis de causas de accidentes.

**Ampliación futura:** el GTFS estático ya incluye todos los núcleos de Cercanías, así que extender el colector a otros núcleos es una cuestión de configuración. Ampliarlo a Media y Larga Distancia exigiría otras fuentes de Renfe, cuyo contenido en tiempo real no hemos comprobado (hipótesis).

### Viabilidad y dudas

**Principal incertidumbre:** que `trip_updates` contenga retrasos distintos de 0 para los viajes de Madrid. Sin ellos, la etiqueta queda reducida a cancelaciones y avisos, que son menos frecuentes.

**Qué comprobaremos**

1. **Contenido del feed:** capturar muestras en hora punta y medir cuántas entidades son de Madrid, qué fracción tiene un `delay` distinto de 0 y qué porcentaje de `trip_id` cruza con el GTFS estático.
2. **Profundidad del histórico:** capturando desde noviembre, en la actividad 1 tendremos semanas de datos y en la actividad 4 unos pocos meses, sin estacionalidad anual. Mientras tanto buscaremos archivos de terceros con histórico de estos feeds (hipótesis).
3. **Desequilibrio de clases:** medir qué fracción de celdas línea × franja tiene incidencias, para fijar el umbral, la granularidad y las métricas.
4. **Asociación y causalidad:** la meteorología se tratará como variable asociada, no como causa.

**Encaje con el máster**

| Módulo | Qué aporta el proyecto |
|---|---|
| Python | Ingesta continua y limpieza |
| SQL | Modelo horario + tiempo real |
| Estadística | Tasas e intervalos por línea y franja |
| ML | Clasificación con validación temporal |
| Visualización | Vista por línea y franja |
| LLM | Resumen en lenguaje natural de avisos y riesgo |
| MLOps | Colector programado y pipeline |
| Big Data | Volumen de ingesta al ampliar a toda la red (posible vía A) |

---
