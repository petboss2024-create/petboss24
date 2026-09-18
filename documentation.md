# PetBoss 24 — Live Walk Tracking

Documentación viva del sistema de tracking GPS del pet sitter durante un paseo.
Se actualiza en cada paso. Este archivo es la fuente de verdad del proyecto.

**Última actualización:** 2026-09-18
**Estado actual:** Fase 3 — creación del plugin en el Bubble Plugin Editor

---

## 1. Objetivo

Que el pet sitter registre un paseo en vivo (recorrido GPS, duración, distancia,
eventos) y que el cliente pueda ver ese recorrido — en tiempo real durante el
paseo, y como resumen al finalizar.

---

## 2. Punto de partida (lo que existe hoy)

Un archivo HTML único (`petboss_live_walk.html`) pegado en un **HTML element**
de Bubble, que se comunica con la app vía **3 elementos JS2B del plugin Toolbox**.

### Comunicación actual con Bubble

| Función | Cuándo dispara | Payload | Config JS2B |
|---|---|---|---|
| `bubble_fn_walkStarted(json)` | 1 vez, al tocar Play | `{lat, lng, start_time}` | text, single output |
| `bubble_fn_updateLocation(json)` | cada 15s + al loguear evento | `{lat, lng, miles, elapsed_s, events{}, ts}` | text, single output |
| `bubble_fn_walkCompleted({...})` | 1 vez, al tocar Stop | `output1`=end lat, `output2`=end lng, `output3`=miles | multiple outputs, number |

### Lifecycle del walk (state machine)

```
'idle' → 'running' → 'finalizing' → 'done'
```

- **idle**: GPS activo, mapa centrado, esperando Play
- **running**: timer + polyline + distancia + ping de tracking cada 15s
- **finalizing**: Stop tocado, overlay "Saving walk…", `walkCompleted` en camino
- **done**: summary visible

### Estructura del summary

1. Header "Walk Summary" + dot verde "Service completed"
2. Stats grid — Duration (HH:MM:SS) | Distance (X.XX mi)
3. Start / End times — HH:MM
4. Events breakdown — chips 🐕 💩 💧 ⚠️ con contadores
5. Route Map — polyline naranja + marker verde "S" + marker rojo "E", con `fitBounds`
6. Done button

### Limitaciones conocidas del punto de partida

- La ruta (polyline) vive **solo en memoria del cliente**: se pierde al cerrar.
- El JSON de `updateLocation` se guarda crudo y hay que parsearlo en el cliente.
- API key y Map ID **hardcodeados** en el HTML.
- Hay que copiar/pegar el HTML en cada página donde se use.
- Dependencia de Toolbox: 3 elementos JS2B + 3 workflows de parseo.

---

## 3. Decisión: migrar a plugin de Bubble

**Fecha:** 2026-09-16
**Veredicto:** viable y recomendado. El JS core (haversine, `watchPosition`,
polyline, markers, timer, route map, formatters) es ~90% reutilizable.

### Qué se gana

| Hoy (HTML element + Toolbox) | Como plugin |
|---|---|
| 3 elementos JS2B + 3 workflows de parseo | 0 dependencias de Toolbox |
| JSON crudo parseado en cliente | States tipados (number, date, text) usables directo en expresiones Bubble |
| `start_time` como ISO string | State tipo **date** nativo |
| `output1/output2/output3` sin nombre | Events con nombre semántico |
| `KEY` y `MAP_ID` hardcodeados | Shared keys en Settings → Plugins, por app |
| `WALK_ID = 'demo-walk-id'` | Field dinámico `walk_id` |
| Polyline se pierde | State `route_polyline` (encoded) → persistible en la DB |
| Play/Stop solo dentro del HTML | Element actions llamables desde cualquier workflow |
| Copiar/pegar HTML en cada página | Versionado del plugin, rollback, un solo lugar |

### El argumento decisivo: customer view sin polling

Si el elemento recibe la posición como **field dinámico** apuntando a
`Current Walk's ...`, Bubble re-ejecuta `update(instance, properties, context)`
automáticamente cuando ese dato cambia en la DB, vía su websocket.
**Sin `DoInterval`, sin polling.** Un mismo elemento con un field
`mode = sitter | customer` cubre las dos vistas.

### Fricciones aceptadas

1. **No hay git en el Plugin Editor.** Son textareas en el browser, sin source
   maps. Mitigación: el fuente vive en este repo, se pega al publicar.
2. **CSS hay que reescribirlo scopeado.** La hoja actual toca `html, body`, usa
   `100vh` e `inset:0` — nada de eso sobrevive dentro de `instance.canvas`.
3. **Altura del elemento** necesita `instance.setHeight()`.
4. **Loader de Google Maps** debe guardarse con flag global para no cargar el
   script dos veces si la app tiene otro plugin de mapas.
5. **Versionado**: publicar versión nueva y actualizar la app.

### Lo que el plugin NO resuelve

`watchPosition` se congela cuando el browser suspende la pestaña o se apaga la
pantalla. Es el techo real de un walk tracker en web. Solo lo resuelve una app
nativa o un wrapper tipo Capacitor. **Tenerlo presente antes de invertir más.**

---

## 4. Arquitectura objetivo del plugin

### Plugin

**Nombre:** `PetBoss Live Walk`
**Tipo:** privado (sin publicar al marketplace)

### Shared keys (Settings → Plugins, por app)

- `Google Maps API Key` — expuesta al cliente, protegida por referrer restrictions
- `Map ID (dark)`
- `Map ID (light)`

### Element `LiveWalkMap` (visual)

**Fields:**
- `walk_id` (text, dinámico)
- `mode` (dropdown: `sitter` | `customer`)
- `tracking_interval_s` (number, default 15)
- `theme` (dropdown: `dark` | `light`)
- `auto_start` (yes/no)
- campos de posición del sitter (solo modo customer)

**States:**
`walk_state`, `current_lat`, `current_lng`, `start_lat`, `start_lng`,
`start_time`, `end_lat`, `end_lng`, `end_time`, `miles`, `elapsed_s`,
`count_walk`, `count_poop`, `count_pee`, `count_alert`, `route_polyline`,
`last_event_type`

**Events:**
`walk_started`, `location_updated`, `event_logged`, `walk_completed`, `gps_error`

**Element actions:**
`Start walk`, `Stop walk`, `Log event` (field `type`), `Recenter`

### Reemplazo del núcleo de comunicación

```js
// antes
bubble_fn_updateLocation(JSON.stringify(payload))

// después
instance.publishState('current_lat', pos.lat)
instance.publishState('current_lng', pos.lng)
instance.publishState('miles', Number(total.toFixed(3)))
instance.publishState('elapsed_s', elapsed)
instance.publishState('count_poop', counts.poop)
instance.triggerEvent('location_updated')
```

---

## 5. Fases

- [x] **Fase 1 — Google Cloud.** API key, restricciones, Map IDs. *Ya estaba
      configurado por el usuario antes de esta sesión.*
- [ ] **Fase 2 — Modelo de datos en Bubble.** Data type `Walk`, separación de
      campos live vs finales, decisión sobre `WalkEvent` como data type propio.
- [ ] **Fase 3 — Esqueleto del plugin.** Crear plugin, shared keys, elemento,
      fields, states, events, actions. ← **EN CURSO**
- [ ] **Fase 4 — Código del elemento.** `initialize` / `update` / actions + CSS
      scopeado.
- [ ] **Fase 5 — Página del sitter.** Workflows que persisten en la DB.
- [ ] **Fase 6 — Customer view y tests de campo.**

---

## 6. Configuración y credenciales

| Ítem | Valor | Notas |
|---|---|---|
| Google Cloud project | PetBoss 24 | |
| Map ID (light, "iOS") | `8b19bc19c481b43a42b68e89` | Los Map IDs no son secretos |
| Map ID (dark) | *pendiente de confirmar* | |
| Google Maps API key | **no se guarda en el repo** | Vive en Settings → Plugins de Bubble |
| APIs requeridas | Maps JavaScript API | Geocoding ya no se usa |
| Intervalo de tracking | 15 s (máx. 240 pings en un paseo de 60 min) | |

---

## 7. Changelog

### 2026-09-18
- Se crea este documento.
- Usuario confirma que la Fase 1 (Google Cloud) ya está completa. Se salta.
- Se define el nombre del plugin: **PetBoss Live Walk**.

### 2026-09-16
- Análisis de factibilidad de migrar el HTML element a un plugin de Bubble.
- Veredicto: viable, recomendado. Se define la arquitectura objetivo (sección 4).

### Sesión previa (HTML)
- Lifecycle completo de 4 estados implementado.
- Tres canales de comunicación con Bubble vía Toolbox JS2B.
- Summary genérico "Walk Summary" sin datos de mascota.
- Se removieron: rating de experiencia, cards de Pace/Calories, header de
  mascota con avatar, constantes `PET_NAME`/`PET_PHOTO_URL`/`PET_WEIGHT_KG`,
  Geocoding API, `addressStart` del payload, botón manual "Send to Bubble".
