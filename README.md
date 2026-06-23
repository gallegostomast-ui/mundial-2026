# Ruta al Mundial 2026 — seguidor + simulador

Página de un solo archivo (`mundial-2026.html`) que sigue el Mundial 2026 **en vivo**:
trae los resultados desde una API pública, calcula la clasificación con el reglamento de
desempate 2026 (mano a mano primero), arma la tabla de los 8 mejores terceros, resuelve la
fase eliminatoria con la matriz del **Anexo C** y la dibuja como un **bracket póster**, lista
los **goleadores** y muestra los **horarios en la zona del que mira**.

Abrir con doble clic en cualquier navegador. No necesita servidor ni build. Solo Google Fonts
(CDN, opcional) y conexión a internet para los datos en vivo; sin conexión cae a una semilla.

---

## Qué muestra hoy

- **Franja "Hoy"** (arriba de Grupos): resumen de la jornada del día actual *según tu zona
  horaria* — en vivo (marcador + minuto), finalizado (resultado) o la hora si no empezó,
  ordenado por hora. Si hoy no hay partidos, muestra los del próximo día con partidos.
- **Grupos**: tabla calculada de cada grupo + el fixture de sus 6 partidos en **solo lectura**
  (estado: final / en vivo / por jugar), con **sede** y horario. Un grupo se marca *en curso*,
  *en vivo · parcial* o *cerrado*.
- **Mejores terceros**: los 12 terceros en una sola tabla con la línea de corte en el 8°.
- **Llave**: **bracket póster de dos mitades** (izquierda y derecha convergiendo a la final en
  el centro), con sede y horario por llave. En pantalla angosta colapsa a una vista por **mitad**
  (selector Izq/Der/Final) con los pares conectados a su partido hijo. La asignación de terceros
  se resuelve por **Anexo C** en cada refresco. Cada slot se marca **provisorio** (punto dorado)
  o **definitivo**: un cruce deja de ser provisorio solo cuando la posición del equipo es
  *matemáticamente inamovible* (calculado, ver más abajo), no por número de fecha.
- **Goleadores**: tabla ordenada por goles, parseada del campo de goleadores de la API
  (con sus limitaciones — ver abajo). Penales marcados; sin asistencias.
- **Simulador**: **sandbox aislado** que parte del estado real en vivo pero no lo toca. Cargás
  los marcadores de los partidos de grupos **no finalizados** (los finalizados vienen reales y
  bloqueados), recalcula las 12 tablas y los terceros, resuelve el **Anexo C** del escenario y
  arma el **árbol interactivo** completo. En cada llave elegís al ganador con un clic y se propaga
  solo hasta coronar campeón; si cambiás un resultado o un ganador aguas arriba, todo lo de abajo
  que dependía se limpia (nunca queda un campeón huérfano). Botón para reiniciar al estado real.

Los datos vienen en vivo de la API; si falla, se siembra con los resultados conocidos al
20/6/2026 (`SEED_RESULTS`, parciales).

---

## Fuente de datos en vivo: `worldcup26.ir`

API pública open source (proyecto de fans, no oficial). Endpoints usados (GET, **sin token**,
con `Access-Control-Allow-Origin: *` → se llaman directo desde el navegador, sin proxy):

- `GET /get/teams` → 48 equipos (`id`, `fifa_code`, `groups`, …). Sirve para el mapa
  `team_id → fifa_code`.
- `GET /get/games` → 104 partidos. Grupos (`type:"group"`, ids 1-72) y eliminatorias
  (`r32`…`final`, ids 73-104). Campos clave: `home_team_id`, `away_team_id`, `home_score`,
  `away_score`, `home_scorers`/`away_scorers`, `group`, `matchday`, `local_date`, `stadium_id`,
  `finished`, `time_elapsed`, `type`.
- `GET /get/stadiums` → 16 sedes (cruce de `stadium_id`). En el código las sedes están
  embebidas en `VENUES` (ciudad, estadio y huso), así que no se vuelve a pedir en vivo.

Detalles del enganche (todo en la capa de datos):

- **Refresco automático adaptativo**: 30 s si hay algún partido en vivo, 60 s si no
  (`REFRESH_LIVE_MS` / `REFRESH_IDLE_MS`, con `setTimeout` reprogramado).
- **Mapeo de códigos** (`CODE_MAP`): los 48 `fifa_code` coinciden con las claves de `TEAMS`
  salvo **2** — `ALG→DZA` (Argelia) y `HAI→HTI` (Haití). Validado contra los 48.
- **Estado del partido** (`gameStatus`): `played` / `live` / `upcoming`, a partir de
  `finished` (señal primaria; `time_elapsed` viene con casing inconsistente).
- **Fallback**: en el arranque se siembra `SEED_RESULTS` y recién después se pide la API; si el
  fetch falla y no hay nada cargado, queda la semilla. Indicador "datos en vivo / sin conexión"
  en el header.

---

## Arquitectura (3 capas, todo en el `<script>` de `mundial-2026.html`)

1. **CAPA DE DATOS** — equipos, metadatos, estructura y enganche a la API.
   - `TEAMS`: equipos por grupo. `FLAG` / `NAME`: display.
   - `SEED_RESULTS`: semilla de fallback, formato `{ group, home, away, hs, as }`.
   - `API_BASE`, `TEAMS_URL`, `GAMES_URL`, `REFRESH_*`, `CODE_MAP`.
   - `VENUES`: las 16 sedes (`{ city, stadium, off }`, `off` = huso fijo durante el torneo).
   - `R32_TIES`, `R16_TIES`, `QF_TIES`, `SF_TIES`, `FINAL_TIE`, `THIRD_TIE`, `KO_LEADERS`:
     estructura oficial de la llave (partidos 73-104).
   - `ANNEX_C`: las **495 combinaciones** del Anexo C (`grupos-tercero → asignación a líderes`).
2. **MOTOR** — funciones puras, sin DOM.
   - `buildMatches()`, `applyResults()`, `statsFor()`, `standings()`, `rankThirds()`,
     `groupClosed()`, `counts()` / `isLive()` / `isPlayed()` (cuentan el parcial en vivo).
3. **DATOS EN VIVO + RENDER**
   - `matchesFromGames()` / `koMetaFromGames()`: API → modelo (grupos + metadata de KO).
   - `parseLocalDate()` + `kickoffUTC()`: hora de sede → instante UTC.
   - `thirdsAssignment()`: resuelve los terceros contra `ANNEX_C` (provisorio).
   - `scorerRanking()` / `parseScorers()`: goleadores desde el campo crudo.
   - `renderToday`, `renderGroups`, `renderThirds`, `renderKO`, `renderScorers`, `renderMeta`.

El estado es `state.matches` (lista de partidos) + `state.koMeta` (sede/hora de eliminatorias).
**Todo se deriva de los partidos**, no de totales por equipo — lo que permite el mano a mano.

---

## Reglamento de desempate 2026 (implementado en `standings()`)

Cambio importante de esta edición: **el enfrentamiento directo va antes que la diferencia de gol
general** (sistema tipo UEFA). Orden, cuando dos o más equipos del grupo empatan en puntos:

1. Puntos en los partidos entre los equipos empatados (mano a mano)
2. Diferencia de gol en esos enfrentamientos directos
3. Goles a favor en esos enfrentamientos directos
4. Diferencia de gol en todo el grupo
5. Goles a favor en todo el grupo
6. Fair play (tarjetas) — *no implementado: falta el dato*
7. Ranking FIFA — *no implementado: falta el dato*

Los parciales **en vivo** entran al cálculo (vía `counts()`), marcados como provisorios.

**Terceros** (`rankThirds()`): como son de grupos distintos y nunca se enfrentaron, **no hay mano
a mano**. Orden: puntos → dif. de gol general → goles → fair play → ranking FIFA. Top 8 entran.

### Nota fina sobre el mano a mano
La implementación ordena cada bloque de empatados por `H2H pts → H2H dif. gol → H2H goles →
general`. La FIFA, si tras aplicar el mano a mano un subconjunto sigue idéntico, **reaplica** el
criterio solo entre ese subconjunto. Para los casos prácticos el orden actual alcanza; para
fidelidad total al pie de la letra, conviene volver `standings()` recursiva sobre los subconjuntos
empatados en los tres criterios de H2H.

---

## Bracket y matriz de los terceros (Anexo C) — implementado

Los cruces de los líderes de **A, B, D, E, G, I, K, L** contra los terceros **no son a dedo ni por
sorteo**: están predefinidos en el **Anexo C** del reglamento FIFA, una matriz de **495
combinaciones** (todas las formas de elegir 8 de los 12 grupos que aportan tercero).

- La tabla completa está embebida en `ANNEX_C` (`clave de 8 grupos → 8 terceros en el orden de
  líderes A,B,D,E,G,I,K,L`). Fuente: template oficial de Wikipedia que reproduce el Anexo C;
  validada (495 combinaciones únicas = C(12,8), y cada asignación respeta los grupos candidatos
  por slot que da la API).
- `thirdsAssignment()` toma los 8 mejores terceros **de hoy**, arma la clave y resuelve contra
  `ANNEX_C` en **cada refresco**, así que el rival de cada líder se ve siempre — marcado como
  provisorio (punto dorado) hasta que cierren los 12 grupos.
- La estructura de los 16 partidos de dieciseisavos y el árbol hacia la final salen de los labels
  oficiales de la API y de Wikipedia (`R32_TIES` … `FINAL_TIE`).

---

## Horarios (zona del navegador)

- La API entrega `local_date` en **hora local de la sede** (verificado contra horarios oficiales
  en varias sedes/husos).
- `VENUES` tiene el huso fijo de cada sede durante el torneo (Eastern −4, Central −5, Pacific −7,
  México −6; el Mundial entero cae dentro del horario de verano y sin transiciones DST, así que el
  offset es constante). `kickoffUTC()` convierte a un **instante absoluto** (epoch ms).
- En la UI los horarios se muestran en **tu zona** (detectada con `Intl.DateTimeFormat()…timeZone`),
  en formato **24 h**, con la **hora local de la sede** como referencia. La conversión puede
  cambiar el día y eso se ve en la fecha mostrada.

---

## Goleadores: limitaciones de la fuente

No hay endpoint de goleadores; se parsean los campos `home_scorers`/`away_scorers` de cada
partido (`parseScorers` → `scorerRanking`), que vienen **sin normalizar**:

- Nombres **abreviados** (`H. Kane`) mezclados con **completos**, y algunos **con caracteres
  corruptos** (charset). Se muestran tal cual: no se inventan correcciones.
- Merge **conservador**: se junta a un mismo jugador solo cuando coinciden **apellido + selección**
  (puede quedar algún conteo repartido).
- Goles en contra **excluidos** del ranking; penales **marcados** `(p)`.
- **Sin asistencias**: la API no las trae.

---

## Simulador (sandbox aislado)

Una pestaña aparte para proyectar escenarios sin tocar los datos reales ni las otras pestañas.

- **Datos paralelos**: `simInit()` toma un snapshot del estado real (`simState.matches`). Los
  partidos **finalizados** quedan bloqueados con su resultado real; el resto, editables y vacíos.
  El criterio de "editable" es el **estado** (no finalizado), nunca la jornada.
- **Reusa el motor sobre esos datos**: `withSim(fn)` intercambia `state.matches` por los del
  simulador, corre `standings` / `rankThirds` / `thirdsAssignment` (las **mismas** funciones) y
  restaura siempre (`try/finally`). No se modifica ninguna función del motor ni del bracket.
- **Recalcula** las 12 tablas (1°/2° con desempate 2026), la tabla de terceros con su línea de
  corte y los cruces de 16avos vía **Anexo C**, a medida que cargás resultados.
- **Árbol interactivo** (reusa `koCard` y el bracket responsive): clic en el equipo que avanza →
  se propaga al partido siguiente hasta el campeón. `simResolve()` recalcula los participantes de
  cada llave de arriba hacia abajo y **purga los ganadores elegidos que dejaron de ser válidos**,
  así nunca queda un equipo avanzando desde una llave que cambió (sin "campeón huérfano").
- **Reiniciar al estado real**: re-snapshotea desde la API y borra hipótesis y picks.

---

## Limitaciones conocidas

- Dato anómalo de la fuente: el partido **inaugural** viene 1 h corrido respecto al oficial
  (único caso; se deja tal cual para no inventar).
- Fair play y ranking FIFA no están como criterio de desempate final (no hay dato).
- Sin persistencia: al recargar se vuelve a pedir la API (o a la semilla).
- Nombres de goleadores con las limitaciones de arriba.

---

## Pendientes (no aplicados todavía)

1. **Partidos de cada grupo en un desplegable**: que el fixture de los 6 partidos de cada grupo
   esté colapsado por defecto (`<details>` o similar) para dejar la tabla más limpia.
2. **Notas contextuales por pestaña**: hoy las notas largas están todas juntas al pie; pasarlas
   a notas contextuales por pestaña (cada vista con su nota propia, expandible).

---

## Cómo correr / retomar

- Abrir `mundial-2026.html` con doble clic (necesita internet para los datos en vivo).
- Todo es un solo archivo; no hay build ni dependencias.
- Para modularizar a futuro (`data.js` / `engine.js` / `ui.js`) hace falta un dev server, porque
  con módulos ES el archivo deja de abrirse con `file://` (`python -m http.server`, `vite`, etc.).
