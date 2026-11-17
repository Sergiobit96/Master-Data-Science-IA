# Datos · Capa raw (Cercanías Madrid)

Guía para quien vaya a hacer la **limpieza y el EDA** a partir de lo capturado. Explica qué hay en `data/raw/`, cómo leerlo, qué significa cada campo y qué trampas conviene conocer.

> **Leyenda**
> - ✅ **Verificado**: comprobado con capturas reales (17/11/2026).
> - ⚠️ **Hipótesis**: encaja con lo visto, pero hay que confirmarlo antes de basarse en ello.
> - 📄 **Según especificación**: tomado del estándar; no lo hemos comprobado aún en una muestra de Renfe.

---

## 1. Regla principal: el manifiesto manda

**Solo son datos los ficheros registrados en `data/raw/manifest.csv` con `ok = True`.**

No recorráis las carpetas con `glob` para leer todo lo que haya. En `data/raw/` hay ficheros de prueba que no están en el manifiesto (ver [sección 8](#8-ficheros-de-prueba-del-17112026)). Partid siempre del manifiesto y abrid los ficheros de su columna `fichero`.

Además, el manifiesto distingue tres situaciones que en los datos parecen iguales:

| Situación | En disco | En el manifiesto |
|---|---|---|
| Se capturó y no había incidencias | Fichero con trenes sin retraso | Fila `ok = True` |
| Falló Renfe, la red o la validación | Nada | Fila `ok = False` con `error` |
| El colector no estaba funcionando | Nada | **Ninguna fila** |

**Un hueco sin datos no significa "no hubo incidencias".** Si se trata como tal, el modelo aprende que la red funciona mejor de lo que funciona.

---

## 2. Estructura de carpetas

```
data/raw/
├── manifest.csv
├── gtfs_rt/
│   ├── trip_updates/<AAAA-MM-DD>/trip_updates_<AAAAMMDD>T<HHMMSS>Z.pb
│   └── alerts/<AAAA-MM-DD>/alerts_<AAAAMMDD>T<HHMMSS>Z.pb
├── gtfs_static/fomento_transit_<AAAAMMDD>T<HHMMSS>Z_<hash12>.zip
└── meteo/open_meteo_madrid_<AAAAMMDD>T<HHMMSS>Z.json
```

- **La hora del nombre es la de captura, en UTC** (la `Z` final). No es la hora del feed.
- **La subcarpeta de día** también es la fecha UTC de captura. Entre las 00:00 y las 01:00 de Madrid (02:00 en verano), los ficheros caen en la carpeta del día anterior.
- **El nombre de cada GTFS estático** lleva al final los 12 primeros caracteres del SHA-256 del ZIP. Mismo hash significa mismo contenido.

`data/raw/` está en el `.gitignore` (~2,5 GB/mes a 60 s). Se genera con el notebook o el colector de adquisición; no viene en el repositorio.

---

## 3. `manifest.csv`

Una fila por intento de descarga. Las columnas son fijas para todas las fuentes; lo que no aplica queda vacío.

| Columna | Tipo | Descripción |
|---|---|---|
| `fuente` | texto | `gtfs_rt/trip_updates`, `gtfs_rt/alerts`, `gtfs_static`, `meteo/open_meteo` |
| `capturado_utc` | ISO 8601 con zona | Momento de la descarga |
| `ok` | bool | `True` si se descargó, validó y guardó |
| `error` | texto | Tipo y mensaje del error (`HTTPError: 404 ...`, `ReadTimeout: ...`). Vacío si `ok` |
| `feed_utc` | ISO 8601 con zona | Solo GTFS-RT: hora a la que Renfe generó el feed (`header.timestamp`) |
| `n_entidades` | entero | GTFS-RT: entidades del feed. Meteo: horas de la respuesta |
| `bytes` | entero | Tamaño descargado |
| `fichero` | ruta relativa a la raíz, con `/` | Dónde está guardado |
| `nota` | texto | Extras por fuente: `nueva_version=True/False` (GTFS estático), `desde=...; hasta=...` (meteo) |

**Dos horas, dos significados.** `capturado_utc` es cuándo descargamos; `feed_utc` es cuándo generó Renfe el contenido. ✅ Normalmente se diferencian en menos de 20 s. Si la diferencia crece, el feed de Renfe estaba congelado aunque nuestra descarga fuese bien: esa captura no aporta información nueva.

**GTFS estático en el manifiesto:** se registra en cada descarga, pero solo se guarda un fichero nuevo cuando cambia el contenido. Varias filas pueden apuntar al mismo ZIP; la suma de `bytes` mide lo descargado, no lo que ocupa en disco.

---

## 4. GTFS-RT `trip_updates`: estado de los trenes en circulación

### 4.1 Formato y lectura

Protobuf binario según la especificación [GTFS Realtime](https://gtfs.org/documentation/realtime/reference/). No es legible directamente: se lee con `gtfs-realtime-bindings`.

```python
from google.transit import gtfs_realtime_pb2
from google.protobuf.json_format import MessageToDict

feed = gtfs_realtime_pb2.FeedMessage()
feed.ParseFromString(ruta.read_bytes())

# Para trabajar como diccionario (útil para pasar a DataFrame):
d = MessageToDict(feed, preserving_proto_field_name=True)
```

Usad `preserving_proto_field_name=True` para mantener los nombres originales (`trip_id`, `stop_time_update`). Sin él, `MessageToDict` los pasa a camelCase (`tripId`, `stopTimeUpdate`).

**Ojo con `MessageToDict`:** omite los campos con valor por defecto. Un `delay: 0` puede **no aparecer** en el diccionario. Al pasarlo a tabla, un `delay` ausente en una entidad no cancelada significa 0, no "desconocido". Alternativa: leer los campos directamente del objeto protobuf, que sí devuelve 0.

### 4.2 Estructura

✅ Campos que aparecen realmente en el feed de Renfe (captura de 271 entidades del 17/11/2026, 18:06 UTC):

```
FeedMessage
├── header
│   ├── gtfs_realtime_version   "2.0"
│   └── timestamp               segundos Unix UTC (hora de generación del feed)
└── entity[]                    una por tren
    ├── id                      "TUUPDATE_<trip_id>" o "TUCANCEL_<trip_id>"
    └── trip_update
        ├── trip
        │   ├── trip_id                 "1020M19722C5"
        │   └── schedule_relationship   SCHEDULED | CANCELED
        ├── stop_time_update[]          0, 1 o varias (ver 4.4)
        │   ├── stop_id                 "35606"
        │   ├── arrival
        │   │   ├── delay               segundos (puede ser negativo)
        │   │   └── time                segundos Unix UTC: llegada prevista
        │   └── schedule_relationship   SKIPPED (si falta, es SCHEDULED)
        ├── vehicle
        │   └── wheelchair_accessible   ACCESSIBLE | INACCESSIBLE | NO_VALUE
        └── delay                       segundos; igual al de la parada
```

**Campos que NO trae Renfe** (✅ en la muestra), aunque existan en el estándar:
- `trip.start_date`, `trip.start_time`, `trip.route_id`, `trip.direction_id`. **No hay fecha de servicio** (ver trampa en 4.5).
- `departure` en las paradas: solo hay `arrival`.
- Identificador del vehículo físico.

### 4.3 Significado de los campos clave

| Campo | Significado | Uso previsto |
|---|---|---|
| `entity.id` | Identificador del mensaje dentro del feed. ✅ Siempre es prefijo + `trip_id` | Ninguno: redundante con `trip_id` |
| `trip.trip_id` | Viaje programado. Clave para cruzar con `trips.txt` del GTFS estático | Unir con línea, sentido y horario |
| `trip.schedule_relationship` | `SCHEDULED`: circula, con o sin retraso. `CANCELED`: cancelado | **Campo para detectar cancelaciones.** ✅ Coincide siempre con el prefijo del `id`, pero el estándar es este campo |
| `stop_time_update.stop_id` | Estación a la que se refiere la previsión | ⚠️ Parece ser la próxima parada del tren. Unir con `stops.txt` |
| `arrival.delay` | Retraso previsto en esa parada, en segundos. **Negativo = adelanto** | Variable principal para la etiqueta |
| `arrival.time` | Hora prevista de llegada (Unix UTC) | Hora programada ≈ `time − delay` |
| `trip_update.delay` | Retraso del viaje. ✅ En la muestra, idéntico a `arrival.delay` en el 100 % de los casos | Puede usarse cualquiera de los dos |

### 4.4 Tipos de entidad

✅ Distribución en la muestra:

| Tipo | Nº | Cómo se reconoce |
|---|---|---|
| Cancelado | 30 | `schedule_relationship = CANCELED`. **Sin `stop_time_update` ni `delay`** |
| Circula, una parada | 232 | Una parada `SCHEDULED` con `arrival` |
| Circula, con paradas saltadas | 9 | Primera parada `SCHEDULED` con `arrival`, seguida de 3 a 28 paradas `SKIPPED` sin `arrival` |

Las paradas `SKIPPED` indican estaciones en las que el tren **no va a parar** (servicio recortado, desvío, estaciones sin servicio). **Decisión pendiente de equipo:** si cuentan como incidencia. Para el viajero que esperaba en esa estación, lo son.

### 4.5 Trampas para la limpieza

1. **El mismo tren aparece en muchas capturas.** Un tren que circula 1 hora sale en ~60 capturas (a 60 s), con su retraso evolucionando. No contéis filas como incidencias: **agrupad por viaje** y decidid qué valor representa al viaje (retraso máximo, último retraso conocido…).
2. **El `trip_id` no basta como clave del viaje.** En GTFS, un `trip_id` se repite cada día que opera según su calendario, y Renfe no envía `start_date`. La clave de un viaje concreto es **`trip_id` + fecha de servicio**, y la fecha hay que deducirla de la hora de captura. Cuidado con los trenes que cruzan la medianoche y con la conversión UTC → hora de Madrid.
3. **Retrasos negativos.** ✅ Hay adelantos de hasta −5 min (5 casos en la muestra). No son errores, pero tampoco incidencias. Decidid si se recortan a 0.
4. **Cancelados sin retraso.** Un `delay` ausente en un cancelado no es 0: el campo no aplica.
5. **Capturas duplicadas.** Si dos capturas tienen el mismo `feed_utc`, son el mismo contenido. ✅ El feed se regenera cada ~20 s (en los segundos :04, :25 y :44 de cada minuto), así que a 60 s no debería pasar salvo que Renfe se congele.
6. **Los `trip_id` cambian con cada versión del GTFS estático.** Cada captura debe cruzarse con la versión estática vigente en su fecha (ver sección 6), no con la última.

### 4.6 Formato del `trip_id` y filtro de Madrid

✅ Los 271 `trip_id` de la muestra siguen el patrón:

```
1020M19722C5
││││ │    └── línea: C5 (puede llevar letra: C4a, C8b, R2N, RG1…)
││││ └─────── 5 dígitos: número de tren (⚠️)
│││└───────── "M" fija
││└────────── "20" fijo en toda la muestra
└┴─────────── núcleo: 10 = Madrid (⚠️)
```

⚠️ **Los dos primeros dígitos parecen ser el núcleo:** `10` = Madrid, `30` = Sevilla, `51` = Barcelona (Rodalies), `60` = Valencia… Los `trip_id` que empiezan por `10` tienen todos sufijo de línea madrileña: C1, C2, C3, C4a, C4b, C5, C7, C8a, C8b, C10.

**Antes de filtrar por el prefijo `10`**, confirmadlo cruzando con `routes.txt`/`trips.txt` del GTFS estático. Es más robusto filtrar por las rutas de Madrid del estático que por un prefijo deducido a ojo.

### 4.7 Cifras de referencia (una sola captura)

✅ 17/11/2026, 19:06 hora de Madrid (martes, hora punta de tarde), núcleo de Madrid (prefijo `10`):
- 64 trenes en el feed: 8 cancelados y 56 circulando.
- 21 con retraso ≥ 5 min, máximo 40 min.

**Es una foto de un momento.** Sirve para confirmar que la etiqueta es viable, no para estimar tasas.

---

## 5. GTFS-RT `alerts`: avisos e incidencias

✅ Verificado: ~85 entidades por feed, ~40 KB (pesa el doble que `trip_updates` con un tercio de entidades, por el texto). Se genera con un ciclo distinto al de `trip_updates`, así que **las horas de ambos feeds nunca coinciden**: al cruzarlos, emparejad cada captura con la más cercana del otro feed.

📄 **Estructura según la especificación (pendiente de verificar con una muestra de Renfe):**

```
entity[]
└── alert
    ├── active_period[]         start / end en segundos Unix UTC
    ├── informed_entity[]       a qué afecta: route_id, stop_id, trip, agency_id…
    ├── cause                   UNKNOWN_CAUSE, TECHNICAL_PROBLEM, MAINTENANCE, WEATHER…
    ├── effect                  NO_SERVICE, REDUCED_SERVICE, SIGNIFICANT_DELAYS, DETOUR…
    ├── url
    ├── header_text             TranslatedString → translation[] {text, language}
    └── description_text        TranslatedString → translation[] {text, language}
```

**Lo primero que hay que mirar en la EDA:**
- Qué campos rellena realmente Renfe, y si `cause` y `effect` vienen informados o siempre `UNKNOWN`.
- Cómo indica la línea afectada: `route_id` en `informed_entity` o solo en el texto libre.
- **¿85 avisos activos es lo normal?** Parece mucho. Probablemente muchos son permanentes (obras, accesibilidad de estaciones) y no incidencias del momento. Habrá que distinguirlos, por ejemplo por la duración de `active_period` o porque aparecen en todas las capturas.
- En qué idiomas viene el texto.

Se lee igual que `trip_updates` (mismo `FeedMessage`).

---

## 6. GTFS estático: horario programado

ZIP con ficheros CSV (`.txt`) según el estándar [GTFS Schedule](https://gtfs.org/documentation/schedule/reference/). ✅ ~15,6 MB. Contiene **todos los núcleos** de Cercanías.

- **Vigencia aproximada de un mes**, y los `trip_id` cambian entre versiones. Cada versión se guarda aparte (el hash va en el nombre).
- ✅ Validación hecha al descargar: el ZIP contiene `agency.txt`, `routes.txt`, `trips.txt`, `stops.txt` y `stop_times.txt`.
- **Elegir versión para cada captura:** la que tenga la fecha de descarga anterior más reciente a la captura, comprobando además que la fecha de la captura cae dentro de la vigencia de `calendar.txt`.

Ficheros relevantes para el proyecto:

| Fichero | Para qué | Clave de unión |
|---|---|---|
| `routes.txt` | Línea (C1, C2…) y núcleo | `route_id` |
| `trips.txt` | Ruta, sentido y calendario de cada viaje | `trip_id` ↔ RT; `route_id` ↔ routes; `service_id` ↔ calendar |
| `stop_times.txt` | Horario programado por parada | `trip_id` + `stop_id` |
| `stops.txt` | Nombre y coordenadas de estaciones | `stop_id` ↔ RT |
| `calendar.txt` / `calendar_dates.txt` | Qué días opera cada servicio | `service_id` |

**Pendiente de comprobar (✅ en la actividad 0 solo para una muestra de Sevilla):**
- Qué porcentaje de `trip_id` de Madrid en tiempo real existe en `trips.txt`.
- Cómo identifica el estático el núcleo de Madrid (`route_id`, `agency_id`…), para filtrar sin depender del prefijo del `trip_id`.
- Que los `stop_id` del tiempo real (5 dígitos) coinciden con los de `stops.txt`.

**Datos de la actividad 0 (versión del 04/11/2026):** 12 líneas en Madrid; un jueves tipo tiene 1.291 viajes, 20.201 paradas programadas y 93 estaciones.

---

## 7. Meteorología (Open-Meteo)

JSON de la API `https://api.open-meteo.com/v1/forecast`. Cada fichero cubre **16 días horarios**: los 14 anteriores, hoy y mañana (384 horas).

📄 Estructura de la respuesta (verificad con vuestro fichero):

```json
{
  "latitude": 40.4,
  "longitude": -3.69,
  "timezone": "Europe/Madrid",
  "utc_offset_seconds": 3600,
  "hourly_units": { "temperature_2m": "°C", "precipitation": "mm", "...": "..." },
  "hourly": {
    "time":           ["2026-11-03T00:00", "2026-11-03T01:00", "..."],
    "temperature_2m": [9.1, 8.7, "..."],
    "precipitation":  [0.0, 0.2, "..."]
  }
}
```

Variables pedidas: `temperature_2m` (°C), `precipitation`, `rain` y `snowfall` (precipitación total, lluvia y nieve), `wind_speed_10m` y `wind_gusts_10m` (viento medio y rachas), `weather_code` (código WMO del estado del tiempo). Comprobad las unidades exactas en `hourly_units`.

**Trampas:**
1. **Las horas están en hora local de Madrid y sin zona** (`"2026-11-03T00:00"`). Los feeds de tiempo real están en UTC. Hay que convertir una de las dos antes de cruzar. En el cambio de hora de octubre la hora local se repite, así que en la meteo esa hora podría aparecer duplicada o ambigua.
2. **Mucho solapamiento entre ficheros.** Cada descarga repite 14 días ya descargados. Para cada hora hay que quedarse con un valor; lo razonable es el de la **descarga más reciente** para horas pasadas.
3. **Las horas pasadas son salidas de modelo, no observaciones de estación.** Para datos observados haría falta AEMET OpenData (requiere clave).
4. **Las horas futuras son previsión.** Para el modelo de ML esto importa: al predecir hay que usar la previsión que existía en ese momento, no el valor que se conoció después (fuga de información).
5. **Un solo punto (Madrid-Atocha) para toda la red**, que cubre unos 100 km. La lluvia puede ser muy local.

---

## 8. Ficheros de prueba del 17/11/2026

Durante el desarrollo de la adquisición (17/11/2026, entre las 18:06 y las 18:51 UTC) se hicieron capturas de prueba que **están en `data/raw/` pero no en `manifest.csv`**: el manifiesto se recreó al añadir la columna `nota`.

- `gtfs_rt/trip_updates/2026-11-17/` y `gtfs_rt/alerts/2026-11-17/`: los ficheros anteriores a la captura de las 18:51:50 UTC.
- `meteo/`: el fichero anterior al de las 18:51:52 UTC.
- **`trip_updates_20261117T182959Z.pb`: su nombre no corresponde a su contenido.** Contiene el feed de las 18:06:45 UTC, guardado 23 minutos después.

Se conservan como evidencia de la exploración. **No forman parte del histórico.** Si se lee a partir del manifiesto (sección 1), quedan fuera automáticamente.

---

## 9. Licencias y privacidad

| Fuente | Licencia | Estado |
|---|---|---|
| GTFS-RT Renfe (`trip_updates`, `alerts`) | CC BY 4.0: permite reutilizar y transformar citando a Renfe | Según la ficha en data.renfe.com |
| GTFS estático Renfe | — | **Pendiente de confirmar** |
| Open-Meteo | — | **Pendiente de revisar** las condiciones de uso de la API y de los datos |

**Privacidad:** datos operativos de la red, sin información personal.

---

## 10. Decisiones pendientes de equipo

1. **Definición de incidencia.** Propuesta de la actividad 0: retraso ≥ 5 min o cancelación. Falta decidir sobre las paradas `SKIPPED` y los avisos de `alerts`.
2. **Valor que representa a cada viaje:** retraso máximo, último retraso conocido, retraso en una franja…
3. **Tratamiento de los adelantos** (retrasos negativos).
4. **Filtro de Madrid:** por el prefijo `10` del `trip_id` o por las rutas del GTFS estático (recomendado).
5. **Meteo:** seguir con Open-Meteo o pedir la clave de AEMET para tener datos observados.