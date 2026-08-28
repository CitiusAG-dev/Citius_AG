# Citius Gestor de Registros — contexto del proyecto

## Qué es

`Citius_Gestor_Registros.html` es una aplicación **de un solo archivo** (vanilla HTML/CSS/JS, sin build,
sin dependencias externas de Node) para capturar y mantener datos de Propiedades/Naves, Parques y
Transacciones industriales, con un módulo adicional para generar una presentación (deck) imprimible/
exportable a PDF a partir de esos datos.

Embebe directamente en el archivo:
- **SheetJS (`XLSX` global)** — leer/escribir el Excel que funge como base de datos.
- **Leaflet** (cargado por CDN al final del archivo) — mapas (captura de polígonos y el mapa de la
  presentación).

No hay servidor: todo corre en el navegador (Chrome/Edge). El archivo Excel vinculado se lee/escribe con
la **File System Access API** (`showOpenFilePicker`/`showSaveFilePicker`/`FileSystemFileHandle`), por eso
solo funciona en Chrome/Edge (`const FSA = "showOpenFilePicker" in window`).

## Cómo editar este archivo

El archivo es enorme (~2MB) y varias líneas son extremadamente largas de forma individual (el literal
`BUNDLE` con toda la metadata de campos, y las imágenes embebidas en base64 del logo/portada/cierre).
**Nunca leas el archivo completo ni un rango de líneas a ciegas** — usa `grep -n` para ubicar la línea
exacta de la función/constante que necesitas y luego `Read` con `offset`/`limit` acotado, o edita
directamente con `Edit` usando un `old_string` único. Evita `sed -n` sobre rangos que puedan incluir la
línea del `BUNDLE` o las imágenes base64 (revienta la salida).

No hay ningún test runner ni navegador interactivo disponible en las sesiones de Claude Code sobre este
proyecto. La forma de verificar cambios visuales/de comportamiento es:
- **Chrome headless con `--dump-dom`** para verificar HTML/CSS renderizado, o
- **Chrome headless + Chrome DevTools Protocol** (`--remote-debugging-port`, `--remote-allow-origins=*`,
  websocket-client de Python) para simular clics, leer `document.querySelector(...)`, capturar screenshots
  (`Page.captureScreenshot`) e inspeccionar el estado en memoria (`state`, `fieldConfig`, `lists`, etc.)
  directamente vía `Runtime.evaluate`.
- Para chequear sintaxis JS rápido sin abrir Chrome: extraer el `<script>` más largo del archivo y
  parsearlo con `esprima` (Python). **Ojo:** esprima es viejo y no soporta optional chaining (`?.`) ni
  `for await...of` (iteración asíncrona, usada por `ensurePhotoSet()`) ni otras sintaxis ES2018+/ES2020+ —
  esos usos hacen que esprima marque error ahí; son falsos positivos conocidos, no bugs reales (Chrome sí
  los soporta). Si esprima falla justo en una de esas líneas, no hay nada que arreglar; para confirmar que
  el resto del archivo sí está sano, se puede neutralizar temporalmente esa línea en una copia (solo para
  el parseo) y volver a correr esprima — la verificación real y definitiva es siempre Chrome headless.

## Navegación entre pestañas (sidebar)

Las 8 "vistas" de la app (Resumen, las 4 tablas de entidad, Mapa, Presentación, Configuración) son
mutuamente excluyentes: cada `show*View()` (`showResumenView`/`showTableView`/`showMapView`/
`showPresentationView`/`showConfigView`) fija exactamente una de 4 banderas booleanas
(`resumenViewActive`/`mapViewActive`/`presentationViewActive`/`configViewActive` — la tabla no tiene
bandera propia, es "ninguna de las 4 arriba" + `current`), llama `hideAllViews()`, muestra su propio
contenedor y llama `renderTabs()` (deriva qué botón del `<nav id="tabs">` se ve activo a partir de esas 4
banderas + `current`). Los botones del nav se generan en JS (no hay `onclick` en el HTML estático) — ver
`fields()`/`ENTITIES` para las 4 entidades de tabla.

- **Se recuerda la última pestaña visitada entre recargas del navegador** (v3.16.0) — antes, recargar
  SIEMPRE mandaba a la tabla de Propiedades sin importar dónde estuviera el usuario. `saveLastView(view,
  entity)` (localStorage `citius_last_view_v1`, `{view:"table"|"map"|"presentation"|"config"|"resumen",
  entity}` — `entity` solo aplica a `"table"`) se llama al ENTRAR a cada una de las 5 `show*View()`, así que
  cualquier forma de navegar (clic de menú, el logo, `editFromMap`, etc.) deja este estado al día solo, sin
  tocar cada call-site aparte. Lo único que cambió de verdad fue el arranque: la llamada incondicional
  `showTableView(); renderTable();` al final del script se reemplazó por una IIFE que lee
  `loadLastView()` y despacha a la vista correspondiente (fijando `current=last.entity` ANTES de
  `showTableView()` cuando aplica) — sin nada guardado (usuario nuevo) o con un valor no reconocible, cae
  exactamente al comportamiento de siempre (Propiedades).
- **El logo de Citius (`.sidebar-logo`) navega a Resumen** (v3.16.0, `onclick="showResumenView()"` +
  `cursor:pointer`) — antes era un `<div>` inerte sin manejador.

## Modelo de datos

- **Entidades**: `PROPIEDADES` (Propiedades/Naves), `PARKS` (Parques), `LAND` (Terrenos), `TRANSACCIONES`
  (Transacciones) — definidas en `ENTITIES` (label, columnas de la tabla, campo de título, campo de
  estatus). `LAND` es un caso especial: vive en un Excel completamente independiente, no en el Excel de
  datos compartido de las otras 3 — ver `## Terrenos (módulo con Excel independiente)` más abajo antes de
  asumir que algo que aplica a las otras 3 entidades también aplica a `LAND`.
  - **Chip de color de la columna de estatus** (`ENTITIES[ent].statusField`, renderizado en `renderTable()`):
    `PARKS.statusField="STATUS"` sigue usando `statusPill(v)` (genérico, por substring — "avail"/"ready"/
    "already"→verde, "constr"/"plan"/"tbd"/"progress"→ámbar, el resto gris; valores reales "Ready"/"Under
    Construction"/"Under Development"/"Planned"). **`PROPIEDADES.statusField="MARKET_STATUS"` tiene su
    propia función `marketStatusPill(v)` (v3.31.0, colores específicos pedidos por el usuario)** —
    comparación EXACTA (`ciEq`, no substring) contra las 3 opciones reales de la lista `MARKET_STATUS`
    (`DEFAULT_LISTS`): `Vacant`→verde, `Available`→ámbar, `Occupied`→rojo (nueva variante `.pill.red`,
    `rgba(226,72,61,.14)`/`var(--red)`, mismo patrón que `.pill.green`/`.pill.amber`/`.pill.gray`) —
    cualquier valor que no matchee ninguna de las 3 cae a gris. El dispatch entre las dos funciones vive en
    el call-site (`renderTable()`: `c==="MARKET_STATUS"?marketStatusPill(r[c]):statusPill(r[c])`), no dentro
    de `statusPill` — si se agrega otra entidad/campo de estatus con su propio mapa de colores a futuro,
    replicar este mismo patrón (función nueva + rama por key exacta en el call-site), no intentar generalizar
    `statusPill` con más casos por substring.
- **`BUNDLE[ent].fields`** — metadata **original/hardcodeada** de campos por entidad: `{key, group}` (más
  `rows` con datos semilla). Es el fallback si nunca se generó configuración de campos.
  - **Quitar un campo de esta app (sin tocar el Excel del usuario)**: basta con quitar su `{key,group}` de
    `BUNDLE[ent].fields` — `pruneFieldConfigToKnownFields()` (guard ya existente, corre en cada carga)
    quita sola la entrada correspondiente de `fieldConfig[ent]`; y como `buildWorkbook()` solo escribe
    `r[key]` para keys que SÍ están en `fields(ent)` (cae a `r.__row[ci]`, el valor crudo tal como se leyó,
    para cualquier columna que ya no reconoce), la columna y los datos ya capturados en el Excel real
    **se preservan intactos** — el sistema simplemente deja de leerlos/mostrarlos/editarlos, no borra nada.
    Precedente real: se quitó `ADMIN` de `BUNDLE.PROPIEDADES.fields` (v3.10.0) y también de
    `BUNDLE.PARKS.fields` (v3.14.0, mismo pedido para esa entidad) — son entradas independientes aunque
    compartan key, cada una se quitó solo cuando se pidió para esa entidad específica.
  - **Etiqueta visible de un campo**: siempre pasa por `fieldLabel(key)` (no `key.replace(/_/g," ")`
    directo) — regresa `FIELD_LABEL_OVERRIDES[key]` si existe, si no cae al `.replace(...)` de siempre. Este
    mapa existe para corregir el TEXTO visible de un key que viene mal escrito del Excel origen (ej.
    `PROPERTY_TPYE`→"PROPERTY TYPE", `ADRESS`→"ADDRESS" (compartido por Propiedades y Parques, el fix
    aplica a ambas), `TRANSACCION_TYPE`→"TRANSACTION TYPE" (Transacciones)) **sin renombrar el key** —
    renombrar el key rompería la
    coincidencia exacta que `loadFromWorkbook()`/`detectHeaderRow()` hacen contra el header real del Excel
    (comparación literal, sin normalizar guiones bajos ni mayúsculas — ver también `## Persistencia y
    seguridad de datos` abajo). Si aparece otro key mal escrito en el futuro, agregarlo a este mapa en vez
    de tocar el key real.
- **`fieldConfig[ent]`** — la metadata de campos **real y editable en runtime**, un array de
  `{key, group, required, type, list, help}` por campo:
  - `required`: boolean, bloquea el guardado si el campo queda vacío.
  - `type`: `"text" | "int" | "decimal" | "date"` — cambia el control de captura real y se valida al
    guardar. `"decimal"` (v3.10.0) es un paralelo exacto de `"int"` pero permitiendo un punto decimal:
    mismo `<input type="number">` en la ficha pero con `step="any"` (en vez de `step="1"`), y en
    `saveBtn.onclick` valida con `/^-?\d+(\.\d+)?$/` + `parseFloat` (en vez de `/^-?\d+$/` + `parseInt`) —
    mismo bloqueo duro si no matchea. Los campos `type:"text"` YA aceptaban decimales de forma silenciosa
    (`maybeNum()`) antes de esto — lo que aporta `"decimal"` es la afectación explícita del campo (`step`
    correcto, sin poder escribir letras) y la validación estricta, igual que ya tenía `"int"`.
  - **`type:"date"` se guarda internamente como texto ISO `YYYY-MM-DD`** (para que ordene bien y viaje
    limpio al Excel), pero se **muestra y edita como `DD-MM-AAAA`** (`formatDateDisplay`/`dateOrDashHtml`)
    tanto en la tabla como en la ficha (vista y edición) — es un formato de presentación, el modelo interno
    no cambia.
    - **El campo de fecha en la ficha NO usa `<input type="date">` nativo** — se reemplazó por un
      componente propio (`dateFieldHtml`, en `buildForm`/`buildViewForm`): un `<input readonly>` que
      muestra `DD-MM-AAAA` con la **misma tipografía exacta** que cualquier otro campo (el date input
      nativo del navegador tiene su propia tipografía inconsistente con el resto — por eso se reemplazó),
      más un botoncito de calendario. Clic en cualquiera de los dos abre directo un mini-calendario hecho a
      mano (`openDatePicker`/`renderDatePicker`/`pickDate`, mismo patrón de popover flotante que
      `col-filter-dropdown`/`config-menu-dropdown`/etc., con navegación de mes y un atajo "Hoy")
      — a propósito, para no depender de los "segmentos" día/mes/año del `<input type="date">` nativo del
      navegador (que además no se pueden forzar a mostrar `DD-MM-AAAA` de forma confiable entre navegadores).
      **Este componente es UNO SOLO, compartido por todos los campos `type:"date"` de cualquier entidad**
      (no hay ningún hook por-campo) — el atajo "Sin fecha" que traía antes se quitó (v3.17.0, pedido
      puntual sobre `TRANSACTION_DATE` en Transacciones, pero al ser compartido el cambio aplica a
      cualquier otro campo de fecha). Consecuencia real: **ya no hay forma de vaciar una fecha ya
      capturada desde el picker**, solo elegir una nueva — si se necesita esa opción de vuelta en el
      futuro, agregarla ahí afecta a TODOS los campos de fecha por igual, no solo a uno.
    - El `data-key` real para guardar vive en un `<input type="hidden">` con id `dateValue_<key>` (valor
      ISO); el input visible (`dateDisplay_<key>`) es solo de presentación. `pickDate()` actualiza ambos y
      dispara un evento `input` en el hidden para que `updateFormNavBadges()` siga funcionando igual.
    - **Al leer del Excel**: Excel guarda fechas como número de serie (días desde 1899-12-30);
      `sheet_to_json` en `loadFromWorkbook()` las lee tal cual, sin convertir (no se pasa `cellDates`). Por
      eso, si un campo es `type:"date"`, `loadFromWorkbook()` detecta `fields(ent).filter(f=>f.type==="date")`
      y, si el valor leído es numérico, lo pasa por `excelSerialToISODate()` a texto ISO antes de guardarlo
      en `state` — sin este paso se ve como un entero gigante. Al escribir de vuelta, se guarda como texto
      ISO plano, no como celda de fecha nativa de Excel con formato `m/d/yyyy` (eso requeriría manipular el
      `.z`/número de formato de la celda, no implementado).
  - `list`: nombre de una lista desplegable (`lists[...]`) o `""` si no aplica.
  - **Pares valor+unidad con nombres irregulares**: la mayoría siguen el sufijo `_VALUE`/`_UNIT` (inglés)
    o `_VALOR`/`_UNIDAD` (español), pero algunos no siguen ningún patrón de sufijo — están declarados a
    mano en `IRREGULAR_VALUE_UNIT_PAIRS` (hoy solo `{"TRANSACTION_AREA":"TRANSACTION_UNIT"}`). Si aparece
    otro caso de un valor+unidad que no se agrupa visualmente, es casi seguro esto — agregarlo ahí.
    `pairValueLabel(key)` deriva la etiqueta correcta en ambos casos.
  - `help`: texto de ayuda opcional (columna `AYUDA` en la hoja `CONFIG_CAMPOS`), editable en
    Configuración > Campos con un botón "i" por fila (`editFieldHelp`, usa `prompt()` igual que las notas
    de ruta de Configuración). Si no está vacío, se muestra en la ficha (vista y edición) como un icono "i"
    circular junto al label del campo, con un tooltip accesible por hover **y** teclado (`fieldHelpIconHtml`
    — CSS puro con `:hover`/`:focus` + `aria-describedby`, sin JS para el tooltip en sí). El tooltip se abre
    **hacia abajo** del ícono (`top:calc(100% + 6px)`), no hacia arriba — a propósito, para que nunca se
    corte contra el borde superior del área con scroll de `.form-panels` cuando el campo está en la
    primera fila de un panel (o cualquier fila que quede pegada arriba al hacer scroll).
    - **Fix real (v3.14.0)**: el tooltip (`.field-help-tip`) podía verse por DEBAJO de `.form-nav` (el menú
      de pestañas propio de la ficha — Identificación/Ubicación/etc., no el menú lateral de la app) cuando
      se extendía hacia la izquierda más allá del borde de `.form-panels`. Causa: `.form-nav` y
      `.form-panels` son columnas HERMANAS con `overflow-y:auto` cada una (lo que en la práctica también
      fija `overflow-x` a `auto`) pero ninguna con `position`/`z-index` propio — sin una jerarquía de
      apilamiento explícita entre 2 hermanos con scroll, el navegador puede decidir el orden de forma
      ambigua para contenido que escapa sus límites. Fix: `.form-panels` ganó `position:relative;z-index:1`
      (prioridad explícita sobre `.form-nav`, que se queda en `z-index:auto`), más `.field-help-tip` subió
      de `z-index:50` a `z-index:300` (refuerzo, mismo nivel que los demás popovers flotantes de la app).
  - El **orden del array** determina tanto el orden de los campos dentro de su grupo como el orden de las
    pestañas (grupos) en la ficha — el primer grupo que aparece recorriendo el array es la primera
    pestaña, etc. (mismo criterio que usa `buildForm`/`buildViewForm` al agrupar).
  - `fields(ent)` es el único punto de acceso (`return fieldConfig[ent] || BUNDLE[ent].fields`) — todo el
    resto del código lee campos a través de esta función, nunca de `BUNDLE` directo.
  - Se edita desde **Configuración → Campos de ficha** (drag-and-drop de grupos y de campos dentro de un
    grupo, checkboxes/selects por campo). Se autoguarda en `localStorage` en cada cambio; el botón
    **"Guardar configuración de campos"** además la escribe al Excel (hoja `CONFIG_CAMPOS`) si hay archivo
    vinculado.
  - Se sincroniza con la hoja `CONFIG_CAMPOS` del Excel al abrir/reconectar/recargar el archivo
    (`loadFieldConfigFromSheet`), igual que las listas se sincronizan con la hoja de listas.
    - **La columna TIPO se valida contra una lista blanca hardcodeada al leer** (`["text","int","decimal","date"]`)
      — cualquier valor no reconocido cae a `"text"` por default. **Si se agrega un tipo nuevo en el
      futuro (`fcFieldRowHtml`'s `<select>` de tipo), hay que agregarlo TAMBIÉN aquí** — se
      olvidó hacerlo al agregar `"decimal"` en v3.10.0 (fix real en v3.15.1: el campo se guardaba bien en
      el Excel, pero volvía a "Texto" en cada recarga porque la lista blanca de lectura nunca se actualizó).
  - **Al quitar un campo de `BUNDLE[ent].fields`** (porque se eliminó esa columna del Excel y ya no
    aplica), no basta con editar `BUNDLE` — `fieldConfig` es una capa persistida aparte (localStorage +
    hoja `CONFIG_CAMPOS`) y puede seguir teniendo ese campo cacheado para usuarios que ya lo tenían. Por
    eso al arrancar se corre `pruneFieldConfigToKnownFields()`, que descarta de `fieldConfig[ent]`
    cualquier `key` que ya no exista en `BUNDLE[ent].fields` — así una columna eliminada del código no
    reaparece sola como columna vacía al guardar. Ejemplo real: la columna `PCT_COMPLETADO` (antes
    calculada a mano en Excel) se quitó de `BUNDLE.PROPIEDADES`/`BUNDLE.PARKS` y quedó reemplazada por la
    columna "% Completado" calculada por la app (ver abajo).
- **Columna "% Completado" en la tabla** (`completionStats`/`completionPill` en `renderTable()`, justo
  antes de "Acciones" en las 3 entidades): `# de campos required con valor / # total de campos required`
  de esa entidad, según `fieldConfig`. Pill verde en 100%, ámbar si es parcial, gris "—" si esa entidad no
  tiene ningún campo marcado como obligatorio todavía. Es un valor **derivado en cada render**, no se
  guarda en el Excel ni en `state`.
  - **Para Propiedades, Parques y Terrenos, tener la foto obligatoria también cuenta como obligatorio.**
    El sistema de fotos no vive en `state`/Excel — son archivos `<identificador>[_<tipo>].jpg/.png/.webp`
    en la carpeta de imágenes vinculada (`imagesDirHandle`). **`PHOTO_ID_FIELD`** —
    `{PROPIEDADES:"MAPPING_CODE", PARKS:"ID_PARK", LAND:"ID__LAND"}` — define qué campo de cada entidad se
    usa como base del nombre de archivo; si se agrega una entidad más con fotos a futuro, basta con
    sumarla a este mapa (la condición `if(PHOTO_ID_FIELD[ent])` que decide si mostrar el panel
    "Fotografía" ya es genérica, no hay que tocarla). Este mismo mapa también se reutiliza para la
    validación de IDs únicos al guardar (ver más abajo) — es el mismo campo identificador en ambos casos.
  - **Varias fotos por registro en Propiedades/Parques/Terrenos** (v3.28.0, tipos ajustados en v3.29.0;
    **Terrenos se sumó en v3.35.0**, antes tenía 1 sola foto sin sufijo) —
    `PHOTO_KINDS_PROPIEDADES_PARKS=[{key:"FACHADA",label:"Foto Fachada",required:true},
    {key:"EXT",label:"Foto Exterior",required:false}, {key:"INT",...}, {key:"AEREA",...}, {key:"DOCKS",...},
    {key:"LAYOUT",...}]` (solo Fachada obligatoria) y **`PHOTO_KINDS_LAND=[{key:"SATELITAL",label:"Foto
    Satelital",required:false}, {key:"PANORAMICA",label:"Foto Panorámica",required:false},
    {key:"LAYOUT",label:"Foto Layout",required:false}]`** (v3.35.0, pedido explícito del usuario — ninguna
    obligatoria; reemplazó la única foto sin sufijo que Terrenos tenía antes). `PHOTO_KINDS_BY_ENTITY`
    (`{PROPIEDADES,PARKS:PHOTO_KINDS_PROPIEDADES_PARKS, LAND:PHOTO_KINDS_LAND}`) es la fuente única;
    `MULTI_PHOTO_ENTITIES` se deriva de sus keys (`new Set(Object.keys(PHOTO_KINDS_BY_ENTITY))`, ya no una
    lista escrita a mano) y `photoKindsFor(ent)` solo hace el lookup ahí, cayendo a **una sola foto con
    `key:""`** (sin sufijo) para cualquier entidad que no esté en ese mapa (hoy ninguna — Transacciones no
    tiene campo de foto en absoluto, vía `PHOTO_ID_FIELD`).
    - **El `key:"LAYOUT"` de Terrenos es intencionalmente el MISMO key que el de Propiedades/Parques** —
      `confirmPhotoEditSave()` bifurca el algoritmo de recorte por `kindKey==="LAYOUT"` (no por entidad, ver
      `## Presentación (deck)`... en realidad ver la nota de fotos de Layout más abajo), así que Terrenos
      hereda automáticamente el mismo ajuste "fit por ancho sin recortar, fondo blanco" sin tocar ese código.
    - **Las fotos viejas sin sufijo de Terrenos NO se migran** a ninguno de los 3 tipos nuevos — a
      diferencia de la migración `_EXT→_FACHADA` (ver abajo), aquí no hay forma segura de saber a cuál de
      los 3 correspondía cada foto vieja (¿era satelital? ¿panorámica? ¿un layout?). El archivo viejo se
      queda intacto en la carpeta de imágenes, simplemente huérfano — el sistema deja de buscarlo, no lo
      borra. Si el usuario sabe a cuál tipo corresponde una foto vieja en particular, la puede renombrar a
      mano con el sufijo correcto, o volver a subirla desde la ficha.
    `photoFileBase(code,kindKey)` centraliza el nombre de archivo (`code` a secas, o
    `code+"_"+kindKey`) para que el resto del mecanismo no necesite saber cuántos tipos de foto tiene
    esa entidad. Todas las funciones del mecanismo (`buildImageBlock`/`buildImageViewBlock`,
    `initImageField`, `loadImagePreview`/`loadViewImagePreview`, `handleImageFileChosen`, `removeImage`,
    `unlockImagesAndPreview`) reciben/propagan un `kindKey`, y generan un `id` de DOM por tipo
    (`imagePreviewBox_FACHADA`, `imagePreviewBox_SATELITAL`, etc.). `buildImageBlock`/`buildImageViewBlock`
    solo muestran el título de cada foto (`.image-block-title`) cuando `photoKindsFor(ent).length>1` — con
    los 3 tipos nuevos, Terrenos ahora SÍ muestra título por bloque (antes, con 1 sola foto, no lo hacía).
    Cada bloque se dibuja en columna (etiqueta arriba, caja de foto abajo, botones debajo) y, con más de un
    tipo, en grid de 2 columnas (`.image-blocks-grid.multi`) — Terrenos ahora entra en ese grid también.
    **Migración automática de fotos viejas** (2 pasos, ambos corren en cada escaneo de la carpeta, ver
    `ensurePhotoSet()`, y ambos son naturalmente "una sola vez" — tras migrar no queda nada que volver a
    migrar):
    1. `migrateExtSuffixToFachada()` (v3.29.0) — para Propiedades y Parques: si existe
       `{id}_EXT.<ext>` (de cuando `_EXT` SÍ era la foto obligatoria, bajo el mecanismo de v3.28.0) y
       todavía no existe `{id}_FACHADA.<ext>`, lo renombra. Corre ANTES que el paso 2 para no dejar una
       ventana donde alguien pueda subir la nueva "Foto Exterior" (opcional) y sobrescribir por
       accidente el archivo de la fachada bajo el nombre viejo. **Itera un array fijo `["PROPIEDADES",
       "PARKS"]` escrito a mano, NO `MULTI_PHOTO_ENTITIES`** (desde v3.35.0, cuando Terrenos se sumó a
       `MULTI_PHOTO_ENTITIES` con sus propios 3 tipos) — a propósito, porque esta migración es
       específicamente sobre el sufijo `_EXT`, que Terrenos nunca usó; si iterara `MULTI_PHOTO_ENTITIES`
       recorrería Terrenos sin necesidad, buscando un sufijo que jamás existió ahí.
    2. `migrateLegacyPropertyPhotos()` (v3.28.0, solo Propiedades) — rescata fotos aún más viejas,
       guardadas sin ningún sufijo (`{MAPPING_CODE}.jpg`, de antes de v3.28.0), renombrándolas
       directamente a `_FACHADA` (conservando la extensión real del archivo — no siempre es `.jpg`).
    Ninguna de las dos pide permiso de escritura si no estaba ya concedido (no interrumpe un escaneo de
    solo lectura con un prompt); si no puede escribir, simplemente no migra ese ciclo y lo reintenta en
    el siguiente escaneo. Parques nunca tuvo fotos antes de v3.28.0 (nada que migrar por el paso 2);
    Terrenos no se tocó por ninguno de los dos pasos (su propia migración de fotos viejas, si hiciera
    falta, sería un paso 3 aparte — no se construyó, ver nota de v3.35.0 arriba sobre por qué no se
    migran automáticamente las fotos viejas sin sufijo de Terrenos).
  - **`renameImageFile(ent,oldCode,newCode)`** (ganó el parámetro `ent` en v3.28.0) recorre
    `photoKindsFor(ent)` y renombra el archivo de CADA tipo que exista (ignora silenciosamente los tipos
    sin archivo — no todos los registros van a tener las 6 fotos). `photoMissingCount(ent,record)` solo
    cuenta como faltante el tipo marcado `required` (Fachada) — el resto nunca bloquean ni suman al
    badge de la pestaña "Fotografía".
  - **Bug real corregido (v3.39.3)**: el usuario reportó una Propiedad con TODOS los campos obligatorios
    llenos y TODAS las fotos solicitadas cargadas (verificado dato por dato en la ficha), pero la columna
    "% Completado" de la tabla se quedaba fija en 98% en vez de 100%. Causa: `completionStats()` (la
    función detrás de esa columna) contaba la foto obligatoria llamando `hasPhoto(código SIN sufijo)` —
    lógica de ANTES de que la foto obligatoria pasara a guardarse como `{código}_FACHADA.ext` (v3.28.0/
    v3.29.0). Como ese archivo sin sufijo ya no existe (se renombra solo al migrar, ver arriba), esa
    condición siempre daba falso, así que la foto NUNCA contaba como presente para el % de cumplimiento —
    aunque sí estuviera, y aunque la pestaña "Fotografía" de la ficha (que sí usa `photoMissingCount()`,
    con el sufijo correcto) la mostrara bien. Con ~50 campos obligatorios + 1 foto = 51 "obligatorios"
    totales, 1 siempre fallando da justo 50/51≈98% — coincide exactamente con lo reportado. Fix:
    `completionStats()` ahora reusa `photoMissingCount(ent,rec)===0` en vez de duplicar (mal) la lógica de
    detección de foto — misma fuente de verdad que ya usaba correctamente la pestaña Fotografía, evita que
    ambos vuelvan a desincronizarse en el futuro.
  - **Fotos 50% más grandes (v3.28.0)**: `PHOTO_TARGET_W=600, PHOTO_TARGET_H=405` (antes 400×270,
    misma proporción exacta 1.481) — usados en `confirmPhotoEditSave` (antes v3.30.3: directo en
    `handleImageFileChosen`, ver punto siguiente) al llamar `processImageToCoverCrop` (el algoritmo de
    recorte "cover" en sí no cambió). La Presentación (`renderPresentationView()`) sigue mostrando solo la
    foto obligatoria de cada propiedad — el único ajuste ahí fue buscar `{MAPPING_CODE}_FACHADA.jpg` (antes
    `_EXT.jpg`, antes de eso sin sufijo).
  - **Drag-and-drop + girar antes de guardar (v3.30.3)**, 2 pedidos juntos del usuario:
    1. **Arrastrar un archivo sobre la caja de preview** (`.image-preview`) sube la foto, alternativa al
       botón "Subir foto" de siempre — mismo destino final, ambos caminos convergen en
       `startImageUpload(file,kindKey)` (`handleImageDragOver`/`handleImageDragLeave`/`handleImageDrop`
       en la caja; `handleImageFileChosen` en el `<input type="file">`, sin cambios de comportamiento
       propio más allá de delegar). Solo acepta `image/png|jpeg|webp` (mismo `accept` que el input de
       archivo); un tipo no soportado muestra un toast de error sin abrir nada.
    2. **Modal "Ajustar foto antes de guardar"** — antes de v3.30.3, elegir/soltar un archivo procesaba y
       ESCRIBÍA la foto de inmediato (`handleImageFileChosen` llamaba `processImageToCoverCrop` y
       guardaba directo). Ahora `startImageUpload` valida lo mismo de siempre (identificador capturado,
       carpeta vinculada, permiso de escritura) y abre `#photoEditOverlay` (mismo patrón `.overlay`/`.show`
       que `mapOverlay`/`pinOverlay` — un modal real, NO un popover flotante como los paneles de
       Columnas/Ordenar/filtro) con una vista previa y 2 botones para girar 90° a la izquierda/derecha
       (`rotatePhotoEdit(±90)`). Guardar (`confirmPhotoEditSave`) recién ahí corre
       `processImageToCoverCrop` sobre la imagen YA rotada y escribe el archivo — idéntico flujo de
       escritura que antes, solo que la fuente es el resultado rotado en vez del archivo crudo.
       `rotateImageToBlob(srcBlob,degrees)` dibuja la imagen en un canvas del tamaño correcto
       (intercambiando ancho/alto en 90°/270°) — función nueva, independiente del recorte "cover" que ya
       existía, así que `processImageToCoverCrop` no se tocó. La vista previa reutiliza la clase
       `.image-preview` tal cual (mismo aspect-ratio 40/27 y `object-fit:cover` que la caja real de la
       ficha), para que lo que se ve en el modal sea representativo de cómo va a quedar recortada la foto
       final. Cancelar cierra sin escribir nada; el archivo original nunca se toca hasta confirmar. El
       texto de ayuda (`.map-hint` dentro del modal) NO menciona el recorte automático (se quitó esa
       frase en v3.31.1 a pedido del usuario, quedó solo "Gira la foto si hace falta y confirma para
       guardarla.") — si se agrega otra explicación al modal a futuro, mismo criterio: texto breve, sin
       detalles de implementación interna.
       - **Fix real (v3.31.1): Escape cerraba la ficha completa mientras dejaba este modal abierto y
         "huérfano"** — el listener global de Escape (`document.addEventListener("keydown",...)`, al final
         del archivo) llamaba `closeModal()` (la ficha, `#overlay`) incondicionalmente sin saber que
         `#photoEditOverlay` seguía abierto ENCIMA, así que un Escape cerraba la ficha de abajo pero dejaba
         flotando el control de girar sin nada detrás. Fix: el listener ahora revisa PRIMERO si
         `#photoEditOverlay` tiene la clase `.show` — si sí, Escape solo llama `closePhotoEditModal()` y
         `return` (nada más se cierra ese Escape); recién un SEGUNDO Escape cierra la ficha normalmente,
         como siempre. Mismo criterio de "la capa de encima se cierra primero" que ya aplica entre un
         dropdown flotante (filtro de columna, menú de fila) y lo que esté debajo — si se agrega otro modal
         que pueda abrirse ENCIMA de la ficha en el futuro (no un popover, un `.overlay` real como
         `pinOverlay`/`mapOverlay`), replicar este mismo chequeo antes de la lista de `close*()` de siempre.
  - **"Layout" usa un ajuste distinto al resto: fit por ancho sin recortar, fondo blanco (v3.34.0)** —
    pedido explícito del usuario: para un plano/site-plan, el recorte "cover" de siempre (llena el
    recuadro, recorta lo que sobra) puede cortar información real del dibujo. `confirmPhotoEditSave`
    ahora bifurca por `kindKey`: **`"LAYOUT"`** usa `processImageFitWidthWhiteBg(blob,W,H,quality)` en vez
    de `processImageToCoverCrop` — escala la imagen COMPLETA para que su ancho quede exacto en
    `PHOTO_TARGET_W` (proporción real preservada, nunca se recorta ni se distorsiona) y la centra
    verticalmente sobre un lienzo blanco de `PHOTO_TARGET_W×PHOTO_TARGET_H`; el espacio que sobre
    arriba/abajo (el caso normal — un plano suele ser más panorámico que 40:27) queda relleno en blanco.
    Los demás tipos (Fachada/Exterior/Interior/Aérea/Docks de Propiedades/Parques, Satelital/Panorámica de
    Terrenos) siguen exactamente con `processImageToCoverCrop` de siempre, sin ningún cambio — la bifurcación
    es por **key exacta `"LAYOUT"`, no por entidad**, así que cuando Terrenos ganó su propio tipo "Foto
    Layout" en v3.35.0 (mismo key `"LAYOUT"`, ver `## Modelo de datos` arriba) heredó este mismo ajuste
    automáticamente, sin tocar `confirmPhotoEditSave`/`renderPhotoEditPreview` para nada. **Caso atípico
    confirmado con el usuario**: si un plano en particular,
    ya escalado a `PHOTO_TARGET_W`, resulta MÁS ALTO que `PHOTO_TARGET_H` (portrait, poco común) — se
    centra igual y el sobrante arriba/abajo simplemente cae fuera del lienzo (recorte centrado, no
    estirado); no es el camino esperado, el usuario debe girar la foto a horizontal desde el modal de
    girar (`rotatePhotoEdit`) antes de guardar — decisión explícita del usuario, no se intentó "arreglar"
    ese caso en el código (ej. creciendo el lienzo), delegado a que el usuario gire primero.
    - **La vista previa del modal de girar también distingue Layout**: `renderPhotoEditPreview()` agrega la
      clase `.fit-contain` a `#photoEditPreview` solo cuando `photoEditState.kindKey==="LAYOUT"` — CSS
      `object-fit:contain` + fondo blanco en vez de `object-fit:cover`, para que la vista previa (que
      muestra la imagen cruda, todavía sin el ajuste final horneado) se vea representativa de cómo va a
      quedar guardada. La imagen YA guardada (el JPG final, siempre exactamente
      `PHOTO_TARGET_W×PHOTO_TARGET_H`) se sigue mostrando con la `.image-preview` de siempre sin esta
      clase — no hace falta: como el archivo final ya tiene la proporción exacta del recuadro (40/27),
      `cover` y `contain` dan el mismo resultado visual ahí, solo importa la distinción en el modal de
      girar (donde la fuente es la imagen cruda, de proporción arbitraria).
    - Verificado con una simulación en Python de la geometría (escala, alto dibujado, offset vertical) para
      3 casos: panorámico típico (centra con margen blanco simétrico), ya exactamente 40:27 (sin margen,
      idéntico a cover en ese caso) y portrait atípico (confirma el recorte centrado esperado). Sin Chrome
      real disponible — pendiente que el usuario suba un plano real y confirme el resultado.
  - `ensurePhotoSet()` enumera la carpeta UNA VEZ (vía `for await...of imagesDirHandle.entries()`) y
    cachea el resultado en `photoSet` (un `Set` de identificadores con foto — con sufijo cuando aplica, sin
    importar de qué entidad); `hasPhoto(code)` consulta ese caché de forma síncrona. `photoSet===null`
    significa "todavía no lo sabemos" (sin carpeta vinculada, sin permiso, o aún sin escanear) — en ese
    estado **no se cuenta** el requisito de foto para nadie (evita falsos "falta foto" antes de poder
    verificar). `renderTable()` dispara `ensurePhotoSet()` en segundo plano si hace falta y se re-renderiza
    sola cuando resuelve; `openEdit`/`openView` la esperan (`await`) antes de armar la ficha para que el
    badge de la pestaña "Fotografía" salga bien desde el primer render. Subir/quitar/renombrar una foto
    parchea `photoSet` in-place (sin re-escanear la carpeta completa) y refresca el badge de la ficha
    abierta (`refreshPhotoNavBadge`) más la tabla. Cambiar de carpeta de imágenes (`chooseImagesDirBtn`)
    resetea `photoSet=null` para forzar un reescaneo de la carpeta nueva.
- **Columna "Días desde Actualización" en la tabla (v3.40.0, corregida en v3.40.1)**, pedido explícito del
  usuario, misma filosofía que "% Completado": valor **calculado en cada render**, nunca se guarda en el
  Excel ni en `state` (`daysSinceUpdate(rec)`/`daysSinceUpdateHtml(days)`, celda justo antes de "%
  Completado" en `renderTable()`). Ordenable como cualquier columna (`DAYS_SORT_KEY`, header + panel
  "Ordenar por"), sin fecha muestra "—" y se va siempre al final del orden (`null`, aprovechando el manejo
  genérico de "vacíos" de `applySort()` — a diferencia de `PCT_SORT_KEY`, que usa un sentinel `-1` porque
  `completionStats()` casi nunca regresa `null`). Ancho redimensionable con su propia key sintética
  `DAYS_COL_WIDTH_KEY` (mismo patrón que `PCT_COL_WIDTH_KEY`/`ACTIONS_COL_WIDTH_KEY`). **No es parte del
  panel "Columnas"** — es una columna fija del sistema, igual que "% Completado" y "Acciones".
  - **v3.40.0 (primer intento) agregó un campo NUEVO `LAST_UPDATE` a
    `BUNDLE.PROPIEDADES`/`BUNDLE.PARKS`/`BUNDLE.TRANSACCIONES` — esto estaba MAL y se revirtió por
    completo en v3.40.1.** El usuario reportó que la columna nunca calculaba nada, ni siquiera para un
    registro cuyo "Last Update" se veía correcto en la ficha. Compartió una captura de su Excel real: la
    columna se llama **`LAST_MODIFIED`**, no `LAST_UPDATE` — y resulta que `LAST_MODIFIED` **YA EXISTÍA**
    como campo de Propiedades/Parques/Transacciones desde antes de este cambio (ver
    `SYSTEM_TIMESTAMP_FIELD_KEYS` más abajo). El campo que el usuario vio "correcto" en la ficha era ese
    `LAST_MODIFIED` preexistente, NO el `LAST_UPDATE` que yo acababa de agregar — dos campos de fecha
    distintos, con nombre parecido, en la misma sección de la ficha, fácil de confundir. Como el Excel real
    nunca tuvo una columna literalmente llamada `LAST_UPDATE`, ese campo nuevo quedaba SIEMPRE vacío
    (`loadFromWorkbook()` solo puebla columnas cuyo encabezado hace match EXACTO —sensible a
    mayúsculas/guion bajo— con una key ya declarada en `fields(ent)`; sin ese encabezado en el Excel, nunca
    hay nada que leer). Fix real (v3.40.1): se revirtieron por completo los 3 campos `LAST_UPDATE` agregados
    a `BUNDLE` (de vuelta a 117/69/29 campos respectivamente) y el `ENTITY_DEFAULT_TYPES` que los
    acompañaba (de vuelta al ternario `ent==="LAND"` original en `defaultFieldConfig()`) — Terrenos, que sí
    tenía `LAST_UPDATE` desde antes, no se tocó. `daysSinceUpdate(rec)` ahora lee `rec.LAST_MODIFIED ||
    rec.LAST_UPDATE` — el campo correcto que YA existe en cada entidad (Propiedades/Parques/Transacciones
    usan `LAST_MODIFIED`; Terrenos usa `LAST_UPDATE`; nunca los 2 en la misma entidad, así que el operador
    `||` no tiene ambigüedad posible). `LAST_MODIFIED` se guarda como ISO completo con hora en UTC
    (`Date#toISOString()`) en vez de solo la fecha, pero el regex de `daysSinceUpdate` solo necesita el
    prefijo `YYYY-MM-DD`, así que ambos formatos funcionan igual sin cambiar el regex. **Lección para el
    futuro: antes de agregar un campo nuevo a una entidad porque "el usuario dice que ya existe en su
    Excel", verificar primero si ya existe algo equivalente con OTRO nombre en `BUNDLE`/
    `SYSTEM_TIMESTAMP_FIELD_KEYS`** — hubiera bastado revisar la definición de `SYSTEM_TIMESTAMP_FIELD_KEYS`
    (ya documentada más abajo en este mismo archivo) para notar que Propiedades/Parques/Transacciones ya
    tenían su propio campo de "última modificación" con otro nombre.
- **IDs duplicados bloquean el guardado (v3.28.0)** — `saveBtn.onclick`, justo después del bloque que
  aborta por `invalidKeys` (así un ID duplicado bloquea con la misma prioridad que un valor inválido, sin
  llegar a tocar `state[current]`): si `PHOTO_ID_FIELD[current]` tiene un valor capturado y algún OTRO
  registro de esa misma entidad (`r.__id!==editingId` — `editingId` es `null` para uno nuevo, así que
  compara contra todos los existentes) ya tiene ese mismo valor (comparado con `String(...)` en ambos
  lados, porque el valor puede llegar convertido a número por `maybeNum()` si parece numérico), se marca
  el campo con `.field-error`, se muestra un `toast` de error, y se sale sin guardar nada. Solo valida
  hacia adelante en cada guardado — no hay una herramienta aparte para auditar duplicados que ya existan
  hoy en los datos (decisión explícita del usuario, para no sumar alcance de más).
- **Columnas visibles de la tabla, personalizables** (`columnPrefs`, `visibleCols(ent)`, botón
  "Columnas" en el topbar de cada tabla): a diferencia de `fieldConfig`/`lists`, esto es una preferencia
  **personal por navegador** (localStorage `citius_column_prefs_v1`), no viaja en el Excel. El panel deja
  ocultar/mostrar y reordenar (drag-and-drop) columnas, y agregar **cualquier** campo de esa entidad como
  columna nueva (no solo los 7 de `ENTITIES[ent].listCols`), con buscador. Si no hay preferencia guardada
  (o quedó con claves que ya no existen — se filtran solas), cae de vuelta a `ENTITIES[ent].listCols`.
  `renderTable()` usa `visibleCols(current)` en vez de `cfg.listCols` directamente para encabezados,
  celdas y el cálculo del colspan del estado vacío.
- **Ancho de columna personalizable, arrastrando el borde derecho de cualquier `<th>`** (v3.14.0,
  `columnWidths` — localStorage `citius_column_widths_v1`, `{ent:{key:px}}`, mismo patrón que
  `columnPrefs` pero un mapa por key en vez de un array de orden — `getColWidth`/`setColWidth`/
  `clearColWidth`). Incluye también "% Completado" y "Acciones" (keys sintéticas `PCT_COL_WIDTH_KEY`/
  `ACTIONS_COL_WIDTH_KEY` = `"__PCT__"`/`"__ACTIONS__"`, ninguna de las 2 es un campo real de
  `fields(ent)`); el checkbox de selección para presentación (`.th-select`, 36px fijo) se dejó fuera a
  propósito, no tiene contenido que gane espacio.
  - **`#tbl` sigue en `table-layout:auto`** (no se cambió a `fixed`, para no arriesgar el ancho automático
    de columnas que el usuario nunca toca): un `width` solo no basta para fijar una columna en layout
    `auto` (el contenido la puede forzar a crecer), así que `colWidthStyle(ent,key)` también agrega
    `max-width` + `overflow:hidden` + `text-overflow:ellipsis` tanto al `<th>` como a cada `<td>` de esa
    columna — solo cuando hay un ancho guardado; sin ancho guardado no se agrega ningún `style`, cero
    cambio de comportamiento respecto a antes de este feature.
    - **Fix real (v3.31.0): angostar mucho una columna recortaba el ícono de orden y el botón de filtro**
      (`.th-cell-right`, a la derecha del header), en vez de solo el texto del label. Causa: el `<span>`
      del label, dentro de `.th-cell{display:flex}`, no tenía `min-width:0` — sin eso, un elemento flex NO
      se encoge por debajo del ancho de su contenido de texto (su "tamaño mínimo" por default es el ancho
      del texto completo en una sola línea), así que al angostar la columna el contenido de `.th-cell`
      terminaba siendo más ancho que el `<th>`, y el `overflow:hidden` de arriba recortaba parejo desde el
      borde derecho — cortando justo el grupo de sort/filtro, que `justify-content:space-between` empuja al
      final de la fila. Fix: `.th-cell.sortable span:first-child{flex:1 1 auto;min-width:0;overflow:hidden;
      text-overflow:ellipsis;white-space:nowrap}` — el label ahora SÍ se encoge y trunca con "…" en una sola
      línea; `.th-cell-right` (que ya tenía `flex-shrink:0` desde siempre) nunca vuelve a perder espacio, así
      que el ícono de orden y el botón de filtro quedan siempre completos y visibles sin importar qué tan
      angosta quede la columna. El selector es específico de `.th-cell.sortable` (los headers de campo, que
      envuelven el label en un `<span>`) — no afecta el header de "% Completado" (mismo `.th-cell.sortable`
      pero sin `<span>` propio para el texto, texto + ícono concatenados directo) ni el de "Acciones" (sin
      `.th-cell` en absoluto).
  - **El arrastre no toca la tabla real hasta soltar el mouse**: mueve solo una guía visual
    (`#colResizeGuide`, `position:fixed`, agregada a `document.body`) seguiendo el mouse — con ~450 filas,
    reflowar la tabla completa en cada pixel de arrastre sería costoso e innecesario. El ancho se calcula
    y persiste una sola vez al soltar (`setColWidth` + `renderTable()`). Patrón mousedown→adjunta
    mousemove+mouseup en `document`→mouseup los quita y aplica — mismo patrón que ya usaba el lazo de
    selección del Mapa (`onLassoMouseDown/Move/Up`), pero sobre `document` plano en vez de eventos de
    Leaflet.
  - El handle (`.col-resize-handle`, franja de 6px en el borde derecho, `cursor:col-resize`) es HERMANO de
    `.th-cell` (no descendiente) dentro del `<th>` — un mousedown ahí nunca burbujea a través de
    `toggleSort`/`toggleColumnFilter`, así que ordenar y filtrar columna no se ven afectados.
- **Celdas valor+unidad en la tabla** (`unitKeyFor(ent,key)`): si una columna visible es el lado "valor"
  de un par (mismo criterio de sufijos `_VALUE`/`_VALOR` + `_UNIT`/`_UNIDAD` que `pairFieldsByUnit`), la
  celda muestra "valor unidad" juntos (ej. "56,673 SF", la unidad en gris/`.cell-unit`) **sin** agregar una
  columna aparte — a propósito, el usuario no quiere una columna de unidad separada. Esas celdas se alinean
  a la derecha (`.num-cell`). La columna "% Completado" se alinea al centro (`.pct-col`).
  - **`buildFieldRenderMeta(ent)` (v3.25.3, optimización de rendimiento)** — dentro del loop de filas de
    `renderTable()`, cada celda necesita saber su `unitKey` (`unitKeyFor`), si es el lado "unidad" de algún
    par (`valueKeyForUnit`), y si es un campo de fecha (`isDateField`) — las 3 funciones, tal cual, barren
    TODOS los campos de la entidad en cada llamada (Propiedades tiene 117), así que llamarlas por celda
    costaba `O(filas×columnas×campos)` (y `valueKeyForUnit` peor, ~`campos²` por celda, porque internamente
    llama `unitKeyFor` por cada campo candidato) — con ~1200 filas esto era perceptible al filtrar/ordenar
    (reportado por el usuario). `buildFieldRenderMeta(ent)` calcula la MISMA lógica exacta de las 3
    funciones en una sola pasada por los campos, UNA vez por `renderTable()` (no por celda), devolviendo 2
    `Map` (`byKey`: key→`{unitKey,isDate}`; `valueKeyByUnitKey`: unitKey→valueKey) — el loop de filas hace
    lookups `O(1)` sobre esos mapas en vez de volver a llamar las funciones originales. **Verificado que da
    exactamente el mismo resultado que las funciones originales para los 117+ campos reales de las 4
    entidades, sin ninguna discrepancia**, antes de considerarlo terminado. `unitKeyFor`/`valueKeyForUnit`/
    `isDateField` NO se borraron — siguen usándose en otros lugares (ej. `resolveFilterValue` dentro de
    `applyFilters`, que es un cuello de botella aparte, todavía sin optimizar — ver nota en la memoria de
    versionado v3.25.3 sobre lo que quedó pendiente a propósito).
- **Orden de la tabla, single-click o multi-columna estilo Excel** (`sortState`, `toggleSort(key)`,
  `applySort(rows,ent)`, localStorage `citius_sort_state_v1`, por entidad). **`sortState[ent]` es SIEMPRE un
  array de `{key,dir}` ordenado por prioridad** (nivel 0 = 1er criterio; array vacío/ausente = sin orden) —
  desde v3.30.0, antes (hasta v3.29.0) era un solo objeto `{key,dir}`; `loadSortState()` normaliza ese
  formato viejo a un array de 1 elemento al cargar, así nadie pierde el sort ya guardado.
  - **Clic directo en un header** (incluida "% Completado", key sintética `PCT_SORT_KEY`) sigue siendo el
    "orden rápido" de siempre: ciclo asc → desc → sin orden. El `<div class="th-cell sortable">` entero es
    clickeable; el botón de filtro (embudo) sigue siendo un target de clic aparte gracias a que
    `toggleColumnFilter` ya hacía `evt.stopPropagation()`. **Diferencia desde v3.30.0**: si ya había un orden
    multi-columna de OTRAS columnas (o esta columna no era la única activa), el clic en un header SIEMPRE
    resetea a un único nivel — reemplaza cualquier orden personalizado previo, mismo criterio que el botón
    de orden rápido de Excel (que también descarta cualquier "Orden personalizado" ya aplicado).
  - **Panel "Ordenar por" (v3.30.0)** — botón "Ordenar" en el topbar de cada tabla (a la DERECHA de
    "Limpiar filtros", que va primero — orden de botones ajustado en v3.30.1 a pedido del usuario), abre un
    panel flotante (`toggleSortPanel`, reutiliza la clase `.columns-panel` + modificador `.sort-panel`,
    mismo patrón de popover que el panel "Columnas") para construir un orden de varios niveles explícito:
    "+ Agregar nivel" (`addSortLevel`) agrega una fila con un selector de columna y un `<select>` Asc/Desc;
    cada cambio de nivel (`setSortLevelKey`/`setSortLevelDir`) aplica en vivo (mismo criterio de "aplicar sin
    botón de confirmar aparte" que el panel "Columnas"); los niveles se pueden reordenar por drag-and-drop
    (`sortLevelDragStart/Over/Drop`, mismo patrón exacto que `fcFieldDragStart/Over/Drop`/
    `presItemDragStart/Over/Drop`) o quitar uno a uno (✕) o todos de golpe ("Quitar todo", `clearSortLevels`).
  - **Selector de columna por nivel, con buscador (v3.30.1)** — el primer intento (v3.30.0) usaba un
    `<select>` nativo con los ~120 campos de la entidad, incómodo de recorrer a ciegas (reportado por el
    usuario tras probar). Reemplazado por un botón (`.sort-level-key-btn`, muestra la etiqueta actual + un
    chevron) que abre un popover propio (`#sortLevelKeyPicker`, `toggleSortLevelKeyPicker`/
    `renderSortLevelKeyPickerList`/`pickSortLevelKey`) con el MISMO estilo que el buscador "Agregar
    columna…" del panel "Columnas": un `<input>` de búsqueda (`.columns-panel-search`, clase reutilizada
    tal cual) + una lista de botones filtrados en vivo (`.col-add-item`, reutilizada tal cual) —
    `sortableFieldOptions()` (array `{key,label}`, incluye "% Completado" vía `PCT_SORT_KEY`) es la fuente
    común tanto del buscador como de `sortFieldLabel(key)` (la etiqueta que se muestra en el botón del
    nivel). El popover vive FUERA de `#sortPanel` en el DOM (se posiciona pegado al botón de CADA nivel, no
    al panel completo), así que `onDocMouseDownForSortPanel` excluye explícitamente
    `#sortLevelKeyPicker` (`!e.target.closest(...)`) para que un clic ahí no cierre el panel "Ordenar por"
    por accidente. Cualquier mutación que cambie los índices de los niveles (`removeSortLevel`/
    `sortLevelDrop`/`clearSortLevels`) cierra el picker primero — si quedara abierto apuntando al índice
    viejo, el clic de una opción actualizaría el nivel equivocado.
  - **Badge de prioridad en el header** (`sortIconHtml`): con 2+ niveles activos, cada header que participa
    en el orden muestra su número de prioridad (1,2,3…) en un badge circular sobre la flecha
    (`.sort-icon-wrap`/`.sort-priority`) — con 1 solo nivel (el caso de siempre) se ve idéntico a como se
    veía antes de v3.30.0, cero regresión visual.
  - **`applySort(rows,ent)`** recorre los niveles en cadena: el primer nivel que no empate entre 2 filas
    decide el orden; los siguientes niveles solo desempatan DENTRO de cada grupo que empató en el nivel
    anterior (ej. ordenar por DEVELOPER primero y AVAILABLE_VALUE después deja los registros agrupados por
    DEVELOPER, y dentro de cada grupo ordenados por AVAILABLE_VALUE). Vacíos siguen yendo siempre al final
    en cada nivel, sin importar la dirección — mismo criterio de siempre, ahora aplicado nivel por nivel.
  - **"✕ Limpiar filtros" NO toca el orden** — se intentó lo contrario en v3.30.0 (el mismo botón también
    limpiaba `sortState[ent]`), pero el usuario pidió explícitamente separarlo de vuelta en v3.30.1:
    `clearAllFilters()` solo borra `filters[ent]`; `updateClearFiltersBtn()` solo depende del conteo de
    filtros de columna, sin mirar `sortState` en absoluto. El orden se quita desde su propio "Quitar todo"
    dentro del panel "Ordenar por", o ciclando un header hasta "sin orden" — nunca desde este botón.
- **Chip de "N en presentación"** (`#presSelectedChip`) vive en `.topbar-controls-left` (a la izquierda
  del topbar, aparte de `.topbar-controls-right` donde está el resto), solo visible en Propiedades — se
  actualiza dentro de `renderTable()` a partir de `presentationOrder.length`. `#count` usa `font-size:12px`.
  - **"×" para vaciar toda la selección de un clic (v3.23.0)**: el chip solo se muestra cuando
    `presentationOrder.length>0` (antes se mostraba siempre que `showSelect`, incluso en 0), y ahora incluye
    un `<span class="chip-x">` clickeable → `clearAllPresentationSelection()` — pide `confirm()` (acción
    masiva) y vacía `presentationOrder` por completo. Distinto de `toggleSelectAllPresentation(false)` (ya
    existente desde antes, el checkbox "seleccionar todas" del header de la tabla): ese solo desmarca las
    filas que coinciden con los filtros de columna ACTIVOS en ese momento, mientras que el "×" del chip
    vacía TODO sin importar qué esté filtrado.
- **"(Vacíos)" como última opción del dropdown de filtro tipo lista, estilo Autofiltro de Excel (v3.30.2)**
  — pedido explícito del usuario ("no me aparece la opción de que me muestre los valores vacíos... como el
  Excel que aparece hasta abajo como última opción"). Antes, `columnFilterValuesInUse(ent,key)` excluía por
  completo las filas con valor vacío/`null` al armar la lista de checkboxes — no había forma de aislar ni de
  excluir explícitamente los registros sin captura en ese campo. Fix: si al menos una fila (de las que ya
  pasan los DEMÁS filtros activos) tiene valor vacío, se agrega `""` (string vacío) como última entrada del
  array que regresa esa función — se renderiza como `(Vacíos)` en cursiva/gris (`colFilterValueLabel(v)`,
  clase `.col-filter-blank-label`) y se busca por esa etiqueta, no por el valor real, así escribir "vac" en
  el buscador también la encuentra. **Cero cambios en `fieldMatchesFilter()`/`applyColumnFilter()`** — ya
  soportaban un valor `""` dentro de `filter.values` de forma nativa (`rowVal==null?"":String(rowVal)` ya
  normaliza null/undefined/vacío real a la misma `""`), así que solo hizo falta exponer esa opción en la UI,
  no tocar la lógica de matching. Alcance: **solo el dropdown de checkboxes (tipo "list")** — la mayoría de
  los campos, que son `type:"text"` por default (ver `detectFieldType` en `## Listas condicionales`) — no se
  tocó el filtro numérico (histograma/slider Desde-Hasta) ni el de texto libre ("Contiene…"), que no tienen
  un mecanismo de "(Blancos)" equivalente en Excel tampoco. Verificado con una simulación en Python de la
  lógica real (`columnFilterValuesInUse`/`fieldMatchesFilter`/`applyColumnFilter`) antes de darlo por
  terminado: seleccionar solo "(Vacíos)" aísla las filas sin valor; seleccionar TODO (incluyendo Vacíos)
  quita el filtro por completo (sin restricción, mismo criterio ya establecido de "todo marcado = sin
  filtro"); deseleccionar solo "(Vacíos)" excluye las filas sin valor dejando el resto intacto.
- **Limpiar un filtro de columna**: cada dropdown de filtro de columna tiene un único botón secundario
  **icon-only** (ícono de bote de basura, sin texto, `title="Borrar filtro"`) que reemplaza tanto al viejo
  "Limpiar" como a "Cancelar": `clearColumnFilter()` borra el filtro de esa columna si ya existía uno, o si
  no había nada que borrar simplemente cierra el dropdown sin aplicar cambios (mismo efecto que tenía
  "Cancelar" antes).
- **Botón "✕ Limpiar filtros (N)" global** (`#clearFiltersBtn`, topbar de tabla, junto a "Columnas",
  v3.12.0) — quita de una todos los filtros de columna de la entidad activa. Comparten el mismo topbar/
  botón las 4 tablas (Propiedades/Parques/Terrenos/Transacciones, `current` decide a cuál le toca). Solo
  visible cuando `filters[current]` tiene algo; `clearAllFilters()` vacía `filters[current]`, cierra
  cualquier dropdown de filtro de columna abierto (`closeColumnFilterDropdown()` — evita un popover
  huérfano tras `renderTable()` reconstruir los headers) y re-renderiza; `updateClearFiltersBtn()`
  (llamado desde `renderTable()`) mantiene el texto/contador y el `display` sincronizados. Existió un botón
  parecido en v1.9.0, quitado en v1.9.1 por no usarse — reintroducido a pedido explícito del usuario.
- **Cierre del dropdown de filtro al hacer clic afuera, "consumiendo" ese clic**: `onDocClickForColFilter`
  es un listener de **`click` en fase de captura** (`document.addEventListener("click", fn, true)`, ya no
  `mousedown`) — si el clic cae fuera del dropdown y fuera de un `.col-filter-btn`, llama
  `e.stopPropagation()` **antes** de que el evento llegue al elemento real (ej. una fila de la tabla), y
  luego cierra el dropdown. Esto evita el bug de "clic afuera cierra el filtro pero también dispara lo que
  esté debajo" (ej. abrir la ficha de esa fila) — el primer clic solo cierra, hace falta un segundo clic
  para esa acción de abajo. Este mismo patrón podría replicarse para los otros menús flotantes (menú de
  fila, menú de Configuración, panel de Columnas) si algún día se pide — hoy solo se aplicó al filtro de
  columna porque fue lo único que se pidió.
- **`state[ent]`** — array en memoria de los registros reales de esa entidad; es la fuente de verdad
  mientras la app está corriendo. Cada registro tiene `__id` (y a veces `__row` con la fila cruda original
  para preservar columnas desconocidas al reescribir el Excel).
- **`layouts[ent]`** — `{preamble, headerKeys}` capturado del archivo original (filas antes del encabezado
  + orden de columnas), para que al guardar se preserve el layout tal cual venía, en vez de reordenar todo.
- **Lectura por nombre, no por posición**: el encabezado se detecta buscando la fila con más celdas que
  calcen con las `keys` conocidas (`detectHeaderRow`), y cada columna se mapea por el texto del header, no
  por índice fijo. Esto es lo que permite reordenar/mover columnas y filas de encabezado en el Excel sin
  romper nada, siempre que el texto del header no cambie.
- **`lists`** — mapa `{NOMBRE_LISTA: [valores...]}` para todos los selects de tipo lista. Se administra
  desde **Configuración → Listas**. Se guarda en `localStorage` (`saveLists()`) y se sincroniza con la hoja
  de Excel cuyo nombre contiene "LISTA" (hoy `"(LD) Lista Desp"`).

## Campos calculados de Parques (v3.11.0)

3 campos de `BUNDLE.PARKS.fields` (ya existían como campos manuales) pasaron a calcularse automáticamente y
quedaron de **solo lectura** en la ficha (nunca editables a mano, mismo tratamiento visual que
`LAST_MODIFIED`/el campo de polígono: `<input readonly disabled>` + `<input type="hidden" data-key="...">`
con el valor real, clase `.field-readonly`):

- **`AVAILABLE_LAND_VALUE = TOTAL_LAND_SIZE_VALUE - OCCUPIED_LAND_VALUE`** (redondeado con `round2`),
  **`AVAILABLE_LAND_UNIT = TOTAL_LAND_SIZE_UNIT`** (copiado tal cual, pedido explícito del usuario). Si
  `TOTAL_LAND_SIZE_VALUE` no es numérico/está vacío, queda en blanco (no hay de dónde calcularlo);
  `OCCUPIED_LAND_VALUE` vacío/no numérico se trata como `0` (un Parque sin nada capturado de ocupado se
  asume 100% disponible — juicio propio, no especificado por el usuario).
- **`NUM_BUILDINGS_IN_PARK`** = cantidad de registros de `state.PROPIEDADES` cuyo `ID_PARK` coincide
  (`ciEq`, insensible a mayúsculas) con el `ID_PARK` de ese Parque.
- **`AVAILABLE_BUILDINGS`** = igual, pero solo contando los que además tengan `MARKET_STATUS` igual a
  `"Available"` (`ciEq`).

`recomputeParkDerivedFields(park)` recalcula UN registro mutándolo in-place; `recomputeAllParkDerivedFields()`
recalcula todos los de `state.PARKS`. **A diferencia de `LAST_MODIFIED`/`LAST_UPDATE` (v3.9.x, que solo
dependían del mismo registro)**, los conteos de edificios dependen de `state.PROPIEDADES` — una entidad
distinta — así que se recalculan en TODOS estos puntos para nunca quedar obsoletos ni en pantalla ni en el
Excel exportado:
- `renderTable()`: si `current==="PARKS"`, recalcula TODOS antes de armar filas (tabla siempre fresca).
- `enterEditMode()`/`openView()`: recalcula el registro específico antes de construir la ficha.
- `saveBtn.onclick`: si se guardó un Parque, recalcula ESE registro (pudo cambiar `TOTAL_LAND_SIZE`/
  `OCCUPIED_LAND`); si se guardó una Propiedad, recalcula TODOS los Parques (su `ID_PARK`/`MARKET_STATUS`
  pudo cambiar a cuál Parque pertenece o si cuenta como disponible) — **antes** de `saveState()`, para que
  el Excel de ese mismo guardado ya lleve el valor fresco.
- El botón de eliminar: si se borra una Propiedad, recalcula todos los Parques antes de `saveState()`.

`PARK_COMPUTED_FIELD_KEYS = new Set(["AVAILABLE_LAND_VALUE","NUM_BUILDINGS_IN_PARK","AVAILABLE_BUILDINGS"])`
(`AVAILABLE_LAND_UNIT` no está en el Set porque se renderiza junto con `AVAILABLE_LAND_VALUE` como par
valor+unidad, intercepto por la key del lado "valor") — especialización por KEY exacta, mismo patrón que
`POLYGON_FIELD_KEYS`/`SYSTEM_TIMESTAMP_FIELD_KEYS`: intercepta el dispatch normal en `renderStandaloneField`
(campos sueltos) y `renderPairField` (el par valor+unidad) ANTES de generar inputs/selects editables. **La
vista (`buildViewForm`) no necesitó ningún cambio** — ya mostraba `fmt(record[key])` como texto plano para
cualquier campo, computado o no, por ser inherentemente de solo lectura.

Si se agrega otro campo calculado con dependencia cruzada entre entidades en el futuro, replicar este mismo
patrón de "recalcular en cada punto donde cualquiera de las entidades involucradas se guarda/borra/renderiza"
— no basta con recalcular solo al guardar el propio registro, como sí alcanzaba para `LAST_MODIFIED`.

## Persistencia y seguridad de datos (Excel vinculado)

- **Vincular archivo**: Abrir Excel / Nuevo Excel / Reconectar (en Configuración → Archivos Excel /
  Imágenes), usando el picker nativo y guardando el `FileSystemFileHandle` en IndexedDB para recordarlo
  entre sesiones (`saveHandle`/`getHandle`).
- **Escritura**: `writeToFile()` siempre serializa **todo** el estado en memoria (`buildWorkbook()`) y
  sobrescribe el archivo completo — no es un merge incremental. Se dispara automáticamente desde
  `saveState()` cada vez que se agrega/edita/borra un registro, o desde el botón "Guardar configuración de
  campos". **`writeToFile()` regresa `true`/`false`** (éxito real o no) — cualquier código que muestre un
  toast de "guardado" después de `await writeToFile()` **debe** condicionarlo a ese resultado, nunca
  mostrarlo a ciegas. Hubo un bug real por esto (corregido en 1.10.1): el botón "Guardar configuración de
  campos" mostraba "Configuración de campos guardada" pasara lo que pasara, tapando casi al instante el
  toast de error que `writeToFile()` ya mostraba internamente si fallaba o si no había `fileHandle` — el
  usuario nunca se enteraba de que en realidad no se había escrito nada en el Excel (veía los cambios en la
  app porque `fieldConfig`/`state` sí se guardan siempre en `localStorage` de inmediato, independientemente
  de si el archivo está vinculado). Si un usuario reporta "veo el cambio en el sistema pero no aparece en
  el Excel", **revisar primero** si el punto de guardado en cuestión muestra su mensaje de éxito
  incondicionalmente en vez de basado en el resultado de `writeToFile()`.
- **Protección contra ediciones concurrentes** (implementada porque el Excel puede estar abierto a la vez
  en Excel de escritorio/Excel Online vía OneDrive):
  1. Al cambiar de módulo/pestaña (`showTableView`, `showMapView`, `showPresentationView`,
     `showConfigView`) se revisa si el archivo cambió en disco (`refreshFromFileIfLinked()`, comparando
     `file.lastModified`) y si es así se vuelve a leer antes de mostrar la vista.
  2. Al abrir un registro para ver/editar (`openView`/`openEdit`) se hace la misma revisión antes de
     buscar el registro.
  3. Justo antes de escribir (`writeToFile()`), se compara la fecha de modificación actual del archivo
     contra la última conocida (`lastKnownFileModTime`); si cambió, se pide confirmación explícita
     (`confirm()`) antes de sobrescribir, avisando que se perderían cambios externos.
  - Esto es una mitigación, no una solución completa: sigue siendo un "último en escribir gana" si el
    usuario decide sobrescribir de todas formas. No hay merge de cambios concurrentes.

## Archivo de Configuración centralizado (Campos + Listas) — despliegue en equipo

Pensado para el caso de uso real del proyecto: ~10 personas cubriendo ~15 mercados, cada una con **su propio
Excel de datos** (Propiedades/Parques/Transacciones), pero compartiendo **un solo archivo de configuración**
(Campos + Listas) sincronizado vía SharePoint/OneDrive junto con el HTML. Es un tercer recurso vinculable en
Configuración → Archivos, junto al Excel de datos y la carpeta de imágenes.

- **Mismas 2 hojas que ya existían dentro del Excel de datos**: `CONFIG_CAMPOS` y `(LD) Lista Desp` — mismo
  formato de columnas exacto, así que `loadFieldConfigFromSheet(wb)` se reutiliza tal cual, y
  `loadListsFromSheet(wb)` (extraído de lo que antes vivía inline en `loadFromWorkbook`) también. Esto
  permite literalmente copiar esas 2 hojas de un Excel de datos existente a un archivo nuevo para arrancar
  la migración.
- **Vinculación**: `configFileHandle` (persistido en IndexedDB vía `saveConfigHandle`/`getConfigHandle`,
  clave `"config"` — mismo patrón que `saveImagesHandle`/`getImagesHandle`, no se generalizó `saveHandle`
  para no romper el estilo ya establecido de duplicar el patrón por recurso en vez de refactorizar
  prematuramente). Botones "Vincular archivo…"/"Crear archivo…" en su propia tarjeta de `.files-grid`.
- **Nivel de cuidado del Excel de datos, no el de la carpeta de imágenes**: tiene su propio
  `lastKnownConfigModTime`, se revisa en cada uno de los 7 call-sites que ya revisaban
  `refreshFromFileIfLinked()` (vía el nuevo `refreshConfigFileIfLinked()`), y `writeToConfigFile()` calca el
  diálogo de "el archivo cambió externamente, ¿guardar de todas formas?" de `writeToFile()`. La carpeta de
  imágenes NO tiene nada de esto (no importaba ahí); aquí sí importa porque el objetivo explícito es que los
  cambios se detecten y se propaguen a todo el equipo.
- **Prioridad total cuando está vinculado**: `loadFromWorkbook()` (el Excel de datos) deja de leer
  `CONFIG_CAMPOS`/`(LD) Lista Desp` de ahí, y `buildWorkbook()` deja de escribirlas — ambos casos gateados
  por `if(!configFileHandle){...}`. Así los 15 Excels de datos no quedan con copias desincronizadas de la
  configuración. **Sin archivo de configuración vinculado, todo sigue funcionando exactamente como antes**
  (fallback total, nadie se rompe por no haber configurado esto todavía).
- `fcSaveBtn` ("Guardar configuración de campos") guarda en el archivo de configuración si está vinculado,
  si no cae al Excel de datos (comportamiento de siempre). Nuevo botón "Guardar en archivo de configuración"
  en el panel de Listas (antes no existía ninguno — los cambios de listas antes solo llegaban al Excel de
  datos de rebote, la próxima vez que cualquier otra acción disparara `writeToFile()`); solo visible cuando
  hay archivo de configuración vinculado (`renderListsNav()` alterna `#listsSaveRow`).
- `localStorage` (`saveFieldConfigLocal`/`saveLists`) sigue siendo el respaldo/caché de siempre, sin
  cambios — es la fuente antes de vincular cualquier archivo.

## Terrenos (módulo con Excel independiente)

4ta pestaña/entidad (`LAND`, label "Terrenos"), agregada en v3.0.0. A diferencia de Propiedades/Parques/
Transacciones, que comparten un solo Excel de datos, Terrenos vive en **su propio Excel completamente
aparte** (ej. `Usuarios/LAND_CS.xlsx`, hoja "LAND", 36 columnas) — es un 4to recurso vinculable en
Configuración → Archivos, con la misma tarjeta/patrón de vinculación que los otros 3, pero totalmente
independiente de ellos.

- **`BUNDLE.LAND`**: 36 campos en 6 grupos (`IDENTIFICACIÓN`, `UBICACIÓN`, `MEDIDAS Y DIVISIBILIDAD`,
  `CARACT. GENERALES`, `SERVICIOS E INFRAESTRUCTURA`, `COMERCIAL`), `rows:[]` — no trae datos semilla
  porque los datos reales siempre vienen del archivo separado. `ENTITIES.LAND.statusField` es `null` (como
  Transacciones — no hay un campo de estatus equivalente en los datos reales).
  - **`LATITUDE`/`LONGITUDE`** (v3.22.0, grupo "UBICACIÓN", justo antes de `GEO_JSON`) — Terrenos no los
    tenía originalmente (a diferencia de Propiedades/Parques), se agregaron a pedido del usuario para poder
    usar ahí el pin picker del mapa (ver `## Pin picker` más abajo). Quedan con `type:"text"` por default
    (no están en `LAND_DEFAULT_TYPES`, mismo criterio que cualquier campo nuevo de Terrenos sin tipo
    especial asignado) — el usuario puede cambiarlos a "Decimal" desde Configuración > Campos si quiere.
    Cero cambio obligatorio en `LAND_CS.xlsx`: si esas columnas no existen ahí, el campo queda vacío hasta
    capturarse (a mano o con el picker) y se escribe igual al guardar, mismo patrón que cualquier campo
    nuevo agregado a una entidad ya existente (ej. `CATEGORY_INDUSTRIAL` en Propiedades, v3.13.0).
- **Tipos por defecto especiales** (`LAND_DEFAULT_TYPES`, usado solo dentro de `defaultFieldConfig()` para
  `ent==="LAND"`): `LAST_UPDATE`→fecha, `ASKING_PRICE_VALUE`→entero, el resto texto. Varios campos que
  parecen numéricos a simple vista se dejaron como texto a propósito porque los datos reales de
  `LAND_CS.xlsx` mezclan decimales o texto de estatus ("TBC", "In Project") en esa misma columna (ej.
  `TOTAL_LAND_SIZE_VALUE`) — forzarlos a `type:"int"` rechazaría/rompería esos valores reales al guardar.
  `ID__LAND` es el único campo `required` por defecto (es el identificador único del terreno).
- **9 cambios de campos juntos (v3.24.0)**, a pedido explícito del usuario, directo en `BUNDLE.LAND.fields`
  (sin ninguna UI de "agregar/quitar campo" propia — esa se evaluó pero el usuario decidió no construirla
  todavía, ver más abajo):
  - Quitados: `YEAR_DEVELOPED`, `MINIMUM_DIVISIBILITY_VALUE`/`_UNIT`/`_TBC` (las 3, no solo valor+unidad —
    la `_TBC` sola no tendría sentido sin su valor/unidad), `BUILDING_OCUPATION_RADIO_`,
    `NUMBER_OF_BUILDINGS_IN_THE_PARK`, `SECURITY`, `CONSTRUCTIONS_COMPANIES_ARE_WELCOME`, `ASKING_PRICE_UR`
    — junto con sus entradas en `FIELD_LIST_MAP.LAND` y `LAND_DEFAULT_TYPES` donde aplicaba (`SECURITY`→
    `LAND_SECURITY`, `CONSTRUCTIONS_COMPANIES_ARE_WELCOME`→`LAND_CONSTRUCTION_WELCOME`,
    `NUMBER_OF_BUILDINGS_IN_THE_PARK`→`"int"`). Las listas `LAND_SECURITY`/`LAND_CONSTRUCTION_WELCOME` NO
    se borraron de `DEFAULT_LISTS` — quedaron sin ningún campo usándolas, el usuario las puede borrar él
    mismo desde Configuración > Listas (v3.23.0) si quiere.
  - Agregados: `CONTACT_PHONE`/`CONTACT_EMAIL` (IDENTIFICACIÓN, junto a `CONTACT` que se dejó intacto),
    `REGION`/`STATE` (UBICACIÓN, insertados antes de `MARKET` para dejar los 4 campos de cascada contiguos
    en orden `REGION,STATE,MARKET,SUBMARKET` — Terrenos queda con la cascada completa automáticamente, sin
    tocar `CASCADE_ORDER`/`cascadeParentKeyFor`, mismo mecanismo que benefició a Transacciones en v3.17.0),
    `FENCE` (CARACT. GENERALES, sin lista/tipo asignado — mismo criterio de "no adivinar" que
    `CATEGORY_INDUSTRIAL` en v3.13.0), `TRANSACTION_TYPE` (COMERCIAL, reutiliza la lista ya existente
    `TRANSACTION_TYPE` vía `FIELD_LIST_MAP.LAND`, key limpio en inglés ya que es un campo nuevo sin columna
    de Excel real preexistente que igualar).
  - **Carga de foto habilitada para Terrenos** — ver `PHOTO_ID_FIELD` en `## Modelo de datos` arriba, la
    generalización completa del mecanismo de fotos que esto requirió.
  - **Antes de implementar**: el usuario preguntó por una protección de "no dejar borrar un campo con datos
    capturados", pensando en un futuro editor self-service de campos (agregar/quitar campos desde la UI, no
    solo su configuración). Se le explicó el riesgo real (el Excel se reconstruye completo en cada
    guardado — `buildWorkbook()`/`buildLandWorkbook()` escriben desde `fieldConfig`/`BUNDLE`, no parchean el
    archivo — así que borrar un campo con datos los pierde de forma irreversible en el siguiente guardado)
    pero decidió no construir esa UI por ahora. Estos 9 cambios se hicieron de la forma establecida en esta
    conversación: edición directa del código por mí, no autoservicio.
- **3 cambios de campos más (v3.35.0)**, a pedido explícito del usuario, mismo estilo que el batch de
  v3.24.0 (edición directa de `BUNDLE.LAND.fields`, no autoservicio):
  - **Quitado**: `LEASE_PURCHASE` (COMERCIAL) — junto con su entrada en `FIELD_LIST_MAP.LAND`
    (`LAND_LEASE_PURCHASE`). La lista `LAND_LEASE_PURCHASE` NO se borró de `DEFAULT_LISTS` (mismo criterio
    que el batch de v3.24.0) — quedó sin ningún campo usándola, el usuario la puede borrar él mismo desde
    Configuración > Listas si quiere.
  - **Cambio de lista**: `TRANSACTION_TYPE` (COMERCIAL) dejó de usar la lista compartida `TRANSACTION_TYPE`
    — ahora usa **`TRANSACTION_TYPE_LAND`** (`FIELD_LIST_MAP.LAND`), una lista propia de Terrenos. Esta
    lista NO se creó en `DEFAULT_LISTS` (a diferencia de las 10 `LAND_*` del batch de v3.24.0, que sí se
    poblaron con valores reales de `LAND_CS.xlsx`) — mismo criterio que `BROKERS_CAG` (v3.32.0/v3.32.1): si
    el usuario ya la tiene creada en su sistema (Configuración > Listas), el campo la usa tal cual apenas
    se recargue; si no existe todavía, el `<select>` simplemente sale vacío hasta que se cree — sin error,
    sin romper nada (mismo fallback ya documentado para cualquier nombre de lista inexistente).
  - **Agregado**: `MINIMUM_TRANSACTION_AREA_VALUE`/`_UNIT` (par valor+unidad, grupo "MEDIDAS Y
    DIVISIBILIDAD", justo después de `TOTAL_AVAILABLE_AREA_VALUE`/`_UNIT`) — sigue el sufijo estándar
    `_VALUE`/`_UNIT` (no hace falta tocar `IRREGULAR_VALUE_UNIT_PAIRS`, `pairFieldsByUnit` ya lo agrupa
    solo). `MINIMUM_TRANSACTION_AREA_VALUE`→`"decimal"` en `LAND_DEFAULT_TYPES`;
    `MINIMUM_TRANSACTION_AREA_UNIT`→`"HAS_M2"` en `FIELD_LIST_MAP.LAND` (reutiliza la lista ya existente,
    mismo criterio que `TOTAL_LAND_SIZE_UNIT`/`TOTAL_AVAILABLE_AREA_UNIT`, no se creó una lista nueva).
    Ninguno de los 2 es `required` (mismo criterio de "no adivinar obligatoriedad" que `CATEGORY_INDUSTRIAL`
    en Propiedades, v3.13.0).
  - **Fotos**: ver el rediseño completo del mecanismo de fotos de Terrenos (3 tipos nuevos, reemplazando la
    única foto sin sufijo de antes) en `## Modelo de datos` arriba, bullet "Varias fotos por registro en
    Propiedades/Parques/Terrenos".
- **Listas nuevas `LAND_*`** (`DEFAULT_LISTS`): no se reutilizaron listas existentes de
  Propiedades/Parques con nombres parecidos (`SECURITY`, `INFRASTRUCUTRE_STATUS`, `SEWER_SYSTEM`) porque
  tienen valores distintos a los reales de Terrenos — se crearon 10 listas `LAND_*` independientes, pobladas
  con los valores distintos reales encontrados en las 273 filas de `LAND_CS.xlsx` (incluye una
  inconsistencia ya presente en los datos, "Already Set"/"Alredy Set" en `INFRAESTRUCTURE_STATUS`, dejada
  tal cual — se puede limpiar después desde Configuración → Listas si se desea). Sí se reutilizan `HAS_M2`
  (los 3 pares tamaño/unidad) y `CON_COMERCIAL` (tarifa de venta, mismo formato "USD/SF" que Propiedades).
- **`POLYGON_FIELD_KEYS`** (antes `POLYGON_FIELD_KEY`, un solo string): generalizado a un `Set` con
  `.has()` en vez de `===`, para que el campo real `GEO_JSON` de Terrenos también reciba el botón "Dibujar
  polígono" sin renombrar esa columna (debía seguir coincidiendo exactamente con el encabezado real del
  Excel) — `GEOJSON_POLIGONO` (Propiedades) sigue funcionando igual.
- **Cascada Mercado→Submercado**: Terrenos no tiene Región/Estado, solo `MARKET`/`SUBMARKET`. Funciona
  automático sin ningún cambio de código — `cascadeParentKeyFor(ent,key)` ya buscaba el ancestro más
  cercano en `CASCADE_ORDER` que SÍ existiera como campo en esa entidad. **Transacciones sí tiene los 4**
  (`REGION`/`STATE`/`MARKET`/`SUBMARKET`, agrupados en su propia sección "Ubicación" — v3.17.0, antes solo
  tenía `SUBMARKET` suelto), así que participa completo en la cascada Región→Estado→Mercado→Submercado
  igual que Propiedades/Parques.
- **Archivo independiente**: mismo patrón exacto que el Excel de datos y el Archivo de Configuración, pero
  con su propio set de variables/funciones sin generalizar (mismo estilo ya establecido de duplicar el
  patrón por recurso): `landFileHandle`/`lastKnownLandModTime`/`writingLand`/`pendingWriteLand`,
  `saveLandHandle`/`getLandHandle` (IndexedDB clave `"land"`), `buildLandWorkbook()`/
  `loadLandFromWorkbook(wb)` (una sola hoja "LAND", mismo `detectHeaderRow` case-insensitive que ya usan las
  otras entidades), `refreshLandFileIfLinked()`/`writeToLandFile()` (mismo diálogo de "el archivo cambió
  externamente, ¿guardar de todas formas?"), banner `#reconnectLandBar` propio. Tarjeta "Archivo de
  Terrenos" en `.files-grid` con un solo botón "Abrir Terrenos…" (sin "Nuevo…" — mismo criterio que v2.1.2,
  el usuario siempre parte de un Excel ya existente).
- **Guardar/eliminar rama por entidad activa**: `saveState()` escribe en `landFileHandle` (vía
  `writeToLandFile()`) si `current==="LAND"`, si no en el `fileHandle` de siempre — antes escribía siempre
  al Excel de datos compartido sin importar la pestaña activa, lo que habría mezclado ahí los datos de
  Terrenos. `buildWorkbook()`/`loadFromWorkbook()` (Excel de datos compartido) excluyen explícitamente
  `LAND` de su loop `for(const ent in ENTITIES)` — Terrenos nunca se lee ni se escribe en ese archivo.
- **3 guards de migración** (necesarios porque `state`/`fieldConfig`/`lists` solo se re-siembran desde cero
  si no existe NADA guardado en `localStorage` — para cualquiera que ya usara el sistema antes de v3.0.0,
  se habrían quedado sin `LAND` para siempre): `if(!state.LAND) state.LAND=[]` (evita un crash real en el
  primer guardado), un loop que rellena `fieldConfig[ent]` para cualquier entidad de `BUNDLE` que falte, y
  un loop que trae a `lists` cualquier clave de `DEFAULT_LISTS` que falte. Si se agrega una 5ta entidad en
  el futuro, replicar este mismo patrón de 3 guards.

## Mapa (todos los polígonos capturados)

Pestaña "Mapa" del menú lateral (`showMapView`/`renderAllPolygonsMap`, contenedor `#allPolygonsMapEl`,
Leaflet + Esri World Imagery como basemap): dibuja en un solo mapa TODOS los polígonos ya capturados, sin
importar de qué entidad vengan, con un filtro de capas para prender/apagar cada tipo. Agregado en v3.1.0 —
antes solo mostraba Propiedades (única entidad con campo de polígono en ese momento).

- **`MAP_LAYERS`**: array de configuración, una entrada por entidad con polígono (`{ent, label, color,
  titleField, subField}` — `titleField`/`subField` son qué campos usar para el título y el subtítulo del
  popup, ej. `PROPERTY_NAME`/`MAPPING_CODE` para Propiedades). Es el único lugar que hay que tocar para
  agregar una 4ta capa en el futuro (ej. si Transacciones alguna vez capturara un polígono).
- **`polygonKeyFor(ent)`**: `fields(ent).find(f=>POLYGON_FIELD_KEYS.has(f.key))` — encuentra el campo de
  polígono real de esa entidad sin hardcodear su nombre (cada entidad puede llamarlo distinto:
  `GEOJSON_POLIGONO` en Propiedades/Parques, `GEO_JSON` en Terrenos). Si una entidad no tiene ningún campo
  en `POLYGON_FIELD_KEYS` (ej. se quitó desde Configuración > Campos), regresa `null` y esa capa
  simplemente no aporta polígonos, sin romper nada.
- **Filtro de capas** (`mapLayerFilter`, `{PROPIEDADES, PARKS, LAND}` todo `true` por defecto): checkboxes
  en `.map-layer-filters` (dentro de `.map-toolbar`), cada uno con un punto de color y el conteo de
  polígonos de esa capa (`mapLayerFilterHtml()`, recalculado en cada render). **En memoria, no persistido**
  — mismo criterio que `resumenYear`/`resumenMarket`, se resetea a "todas visibles" en cada carga de la
  página. `toggleMapLayer(ent, checked)` actualiza el filtro y vuelve a llamar
  `renderAllPolygonsMap(true)` — el `true` es `keepView` (v3.25.2): mostrar/ocultar una capa NUNCA debe
  mover el zoom/centro actual (antes sí lo hacía, recentrando como si el mapa se abriera de cero — bug
  reportado por el usuario, que notó que las capas complementarias de abajo NUNCA tenían este problema).
- **Colores por capa**: Propiedades `#f0b323` (el amarillo de acento de siempre — sin cambio, para no alterar
  cómo se veían los polígonos que ya existían de antes de v3.1.0), Parques `#2e78c9` (azul nuevo), Terrenos
  `#1e9e5a` (mismo valor que `--green`). El popup de cada polígono incluye una etiqueta con el nombre de la
  entidad en ese color, además del título/subtítulo y el botón "Editar registro".
- **Modo de visualización Polígonos/Pines (v3.25.0)** — selector "Mostrar" en `.map-toolbar` (junto al de
  basemap), `mapDisplayMode` (`"polygons"` default, persistido en `citius_map_display_mode_v1`). En modo
  Pines, `renderAllPolygonsMap()` dibuja un `L.marker` con el mismo ícono `.map-pin` (teardrop, coloreado
  por capa vía `style` inline) en `LATITUDE`/`LONGITUDE` de cada registro con coordenadas válidas
  (`hasValidLatLng(row)`) — independiente de si ese registro también tiene polígono. Mismo popup
  (nombre/subcampo/etiqueta de capa/"Editar registro") en ambos modos, mismo `boundsList`/`fitBounds` para
  el auto-zoom (para un pin se empuja un `L.latLngBounds` degenerado de un solo punto). El conteo
  (`#mapCount` y el de cada capa en `mapLayerFilterHtml()`) cambia de texto/criterio según el modo: "N
  polígono(s) capturado(s)" vs "N ubicación(es) con coordenadas".
  - **La herramienta de selección (lazo/radio/polígono) sigue al modo activo** — `handleMapSelection`/
    `showMapSelectPanel` antes SOLO sabían leer el centroide de un polígono; ahora, si
    `mapDisplayMode==="pins"`, usan directo el punto `LATITUDE`/`LONGITUDE` como centro (y dibujan un
    `L.circleMarker` de resalte en vez de reconstruir el polígono) — así seleccionar siempre actúa sobre lo
    que de verdad está dibujado en el mapa en ese momento, nunca selecciona a ciegas un registro sin
    representación visual en el modo activo. Esto se agregó de más (no se pidió explícitamente) para no
    dejar la herramienta de selección rota en modo Pines.
- **`editFromMap(ent, id)`**: antes solo recibía `id` y siempre fijaba `current="PROPIEDADES"` a ciegas —
  funcionaba porque solo existía esa capa. Ahora recibe también la entidad del polígono clickeado y navega
  ahí antes de abrir la ficha (`current=ent; showTableView(); openEdit(id)`).
- **Parques no tenía ningún campo de polígono** hasta v3.1.0 (a diferencia de Propiedades y Terrenos) — se
  agregó `GEOJSON_POLIGONO` a `BUNDLE.PARKS.fields` (grupo `UBICACIÓN`, justo después de `LONGITUDE`, mismo
  nombre de campo que usa Propiedades). El botón "Dibujar polígono" apareció solo en la ficha de Parques sin
  más cambios, porque ese mecanismo ya era genérico por `POLYGON_FIELD_KEYS`.
- **Guard de migración de `fieldConfig` generalizado** (ver `## Terrenos` arriba, que introdujo el primer
  guard): el guard de v3.0.0 solo cubría "la entidad completa falta en `fieldConfig`". Agregar un campo
  nuevo a una entidad que YA EXISTÍA (como `GEOJSON_POLIGONO` en Parques) es un caso distinto — se generalizó
  el loop para que, además, revise cada entidad que SÍ existe en `fieldConfig` y le agregue (con `push`, sin
  pisar lo que ya tenía el usuario) cualquier campo de `defaultFieldConfig()[ent]` que le falte por `key`.
  Este guard generalizado cubre cualquier campo nuevo agregado a cualquier entidad en el futuro, no solo
  este caso puntual.
- **Los 4 contenedores de mapas Leaflet de la app (`#allPolygonsMapEl` acá, `#polygonMapEl` y `#pinMapEl`
  en la ficha, `.map-page-canvas`/`#presMapCanvas` en Presentación) llevan `position:relative;z-index:0`
  explícito en su propia regla CSS** — fix real (v3.7.4): Leaflet asigna z-index altos a sus capas internas
  (hasta 700 en `leaflet-popup-pane`) pero su propio `.leaflet-container` solo trae `position:relative` de
  fábrica, SIN z-index — sin un z-index propio no forma contexto de apilamiento nuevo, así que esas capas
  internas no quedan contenidas: se comparan directo contra cualquier otro elemento posicionado de la página
  (un dropdown, un modal) y a veces ganan aunque estén en otro lugar de la pantalla — el mapa se veía
  "encima" de menús/modales que debían quedar por delante. `#pinMapEl` (v3.22.0, ver `## Pin picker` abajo)
  siguió este mismo patrón desde que se creó, sin necesitar el fix después. Si se agrega un 5to mapa Leaflet
  en el futuro, replicar el mismo `position:relative;z-index:0` en el contenedor que se le pase a `L.map(...)`.
- **La página "Mapa" del deck de Presentación (`.deck-page.map-page`) tiene el mismo `z-index:0` agregado**
  (v3.8.0, misma familia de bug que el punto anterior, un nivel más arriba en el árbol): el badge "Location
  Map" (`.map-page-badge`) y la leyenda de propiedades sin coordenadas (`.map-page .deck-note`) son markup
  PROPIO de la app (no de Leaflet), cada uno con su propio `z-index:1000` explícito, hermanos de
  `#presMapCanvas` dentro de `.deck-page.map-page` — como ese contenedor tenía `position:relative` pero sin
  z-index propio, tampoco formaba contexto de apilamiento, y ambos se seguían viendo por encima del modal de
  edición al abrir una ficha desde Presentación (el fix del punto anterior solo había contenido las capas
  internas de Leaflet, no este markup propio). Si se agrega algún otro overlay con z-index explícito a
  cualquier página del deck (`.deck-page`) en el futuro, revisar que su `.deck-page` contenedor tenga
  también su propio z-index — si no, se repite el mismo bug.

### Capas complementarias (Red Eléctrica, Pozos de Agua) — v3.2.0

5to recurso vinculable, y el **primero puramente de solo lectura** (sin ficha, sin `state`, sin escritura):
"Datos Complementarios (Mapa)" en Configuración > Archivos, un Excel externo con 2 hojas de referencia
(`Red_Electrica`, `Pozos_Agua`), cada una con una columna `geojson` — pensado para capas de contexto (líneas
de alta tensión, pozos de agua) que el usuario quiere ver en el mapa pero nunca captura/edita desde el
sistema.

- **Sin `writeToXFile`, sin `writing`/`pendingWriteX`, sin diálogo de conflicto**: como nunca se escribe, no
  hace falta nada de eso — `ensurePermission` se llama con `{mode:"read"}` en vez de `"readwrite"` (ya
  aceptaba cualquier mode string, no fue necesario tocarla). Si se agrega un 6to recurso de solo lectura en
  el futuro, replicar este patrón simplificado, no el patrón completo de lectura/escritura de los otros 4.
- **`loadComplementaryDataFromWorkbook(wb)`**: parsea ambas hojas a `complementaryData.{RED_ELECTRICA,
  POZOS}` — arrays de `{...columnas del Excel, __geom: geometría ya parseada}` (valores `"None"` literales
  del Excel se normalizan a `""`). `refreshComplementaryFileIfLinked()` se llama **solo desde
  `showMapView()`** — a diferencia de los otros 4 recursos vinculables, no se refresca en cada cambio de
  vista/ficha, porque su único consumidor es la pestaña Mapa (evita releer un Excel de ~21k filas en cada
  clic de la tabla).
- **Bug real durante la implementación, ya corregido**: se asumió al principio un tipo de geometría fijo
  por capa (líneas para Red Eléctrica, puntos para Pozos) — pero la hoja `Red_Electrica` en realidad MEZCLA
  `LineString` (las líneas, 13,749 filas) con `Point` (las subestaciones, "S.E ...", 5,345 filas) en la
  misma hoja. Con el supuesto fijo, esas 5,345 filas de tipo Point tronaban silenciosamente al intentar
  dibujarlas como línea. Fix: `buildComplementaryLeafletLayer(key)` usa `L.geoJSON` por fila (con
  `pointToLayer` para convertir puntos a `L.circleMarker`) en vez de asumir el tipo — lee la geometría real
  de cada fila, sin importar de qué hoja/capa venga. **Si se agrega una 3ra capa complementaria en el
  futuro, no asumir un solo tipo de geometría por hoja — verificar los tipos reales presentes.**
- **Rendimiento con ~21k features**: `L.canvas({padding:0.5})` **compartido** entre todas las features de
  una capa (un solo `<canvas>`, no miles de `<path>` SVG — clave para que no se congele el navegador).
  `complementaryLeafletLayers` cachea la capa Leaflet ya construida por key; togglear el checkbox después
  solo hace `addTo`/`removeLayer` (instantáneo) — nunca se reconstruye salvo que el archivo cambie en disco
  (`loadComplementaryDataFromWorkbook` invalida el caché). Medido con el archivo real: parseo ~85ms,
  construir Red Eléctrica (19,094 features) ~180ms, Pozos (2,064) ~18ms.
- **Ambas capas empiezan apagadas** (`complementaryLayerFilter={RED_ELECTRICA:false, POZOS:false}`) — son
  datasets de todo México, no acotados al área de trabajo de un usuario en particular; se decidió no
  imponerlas encendidas por default. En memoria, no persistido (mismo criterio que `mapLayerFilter`).
- **`renderComplementaryLayers()` es una función APARTE de `MAP_LAYERS`/`renderAllPolygonsMap`'s
  `boundsList`**, a propósito: sus capas se agregan/quitan directo del mapa (`allPolyMap.addLayer/
  removeLayer`), pero JAMÁS aportan al cálculo de `fitBounds` — ese sigue pensado para acercarse solo a las
  propiedades/parques/terrenos capturados por el usuario. Si aportaran al bounds, prender "Red Eléctrica"
  alejaría el zoom para mostrar todo México en vez de mantenerse sobre los registros del usuario. Si se
  agrega una 3ra capa complementaria, seguir el mismo patrón (nunca tocar `boundsList`).
- Colores: Red Eléctrica `#c0392b` (rojo), Pozos de Agua `#17a2b8` (teal — deliberadamente distinto del
  azul de Parques `#2e78c9` para no confundirse en la leyenda de checkboxes).

### Basemap y selección espacial (lazo/radio/polígono) — v3.4.0

Dos controles nuevos en `.map-toolbar`. **Cero dependencias nuevas** — `leaflet-draw@1.0.4` ya estaba
cargado desde antes (usado solo para dibujar UN polígono dentro de la ficha, en un mapa Leaflet aparte:
`polyMap`/`polyDrawnItems`/`openPolygonMap`) y se reutiliza para 2 de las 3 herramientas de selección.

- **`#mapBasemapSelect`**: `MAP_BASEMAPS` (6 entradas, cada una con 1-2 `L.tileLayer`, orden del `<select>`
  y del array siempre sincronizados) en este orden: Satelital, **Calles claro / Voyager**, Calles
  (`World_Street_Map`), Topográfico (`World_Topo_Map`), Escala de grises
  (`Canvas/World_Light_Gray_Base/Reference`), Oscuro (su par `Canvas/World_Dark_Gray_*`). Las últimas 4 más
  Satelital son servicios REST públicos de Esri, sin llave de API. "Calles claro" es la excepción: CARTO
  Voyager (`basemaps.cartocdn.com/rastertiles/voyager`, subdominios `abcd`, agregado en v3.6.2, movido a 2da
  posición en v3.7.0 a pedido del usuario) — el usuario pidió el estilo del ejemplo oficial de MapTiler/
  Leaflet, pero MapTiler requiere llave de API; se le preguntó y **eligió explícitamente** esta alternativa
  gratuita sin llave en vez de conectar su propia cuenta de MapTiler (visualmente muy similar: fondo crema,
  bosques verdes, calles en amarillo/naranja). Único basemap que necesita `subdomains` distinto del default
  de Leaflet — cada
  entrada de `layers` acepta un `subdomains` opcional, `setMapBasemap` lo pasa con fallback a `"abc"` para
  las demás capas (sus URLs no traen `{s}`, así que el valor no les afecta).
  `mapBasemapPref` persistido en `localStorage` (`citius_map_basemap_pref_v1`, default `"satellite"` — cero
  regresión visual). `setMapBasemap(key)` quita `currentBasemapLayers` y agrega las nuevas; reemplazó el
  bloque hardcodeado de tiles que antes vivía dentro de `renderAllPolygonsMap()`.
- **3 herramientas de selección** (`#mapToolLasso`/`#mapToolRadius`/`#mapToolPolygon`, toggle — un solo
  botón activo a la vez, `activateMapSelectTool(tool)`/`deactivateMapSelectTool()`):
  - Polígono y Radio: `new L.Draw.Polygon(allPolyMap, opts)`/`new L.Draw.Circle(allPolyMap, opts)`
    (confirmado en el bundle real de leaflet-draw, no adivinado: constructor `(map, options)`,
    `.enable()`/`.disable()`, disparan `L.Draw.Event.CREATED` con `e.layer`/`e.layerType`, y el layer NO se
    agrega solo al mapa — hay que agregarlo a mano, igual que ya hace `openPolygonMap`). `allPolyMap` (el
    mapa de la pestaña Mapa) nunca había tenido ningún draw handler — es una instancia Leaflet
    completamente aparte de `polyMap` (el de la ficha), así que no hay ningún conflicto entre ambos usos.
  - Lazo (freehand): leaflet-draw NO trae esta herramienta — implementado a mano con eventos crudos
    `mousedown`/`mousemove`/`mouseup` sobre `allPolyMap` (`onLassoMouseDown`/`Move`/`Up`,
    `startLassoDraw`/`stopLassoDraw`), deshabilitando `map.dragging` mientras se dibuja — mismo nivel de API
    que usa internamente `L.Draw.SimpleShape` para su propio drag-to-draw.
  - **`handleMapSelection(shape)` itera SOLO `MAP_LAYERS`** (Propiedades/Parques/Terrenos, ya existía desde
    v3.1.0) — **JAMÁS `COMPLEMENTARY_LAYERS`** (Pozos/Red Eléctrica), aclarado explícitamente por el
    usuario. También respeta `mapLayerFilter`: una capa oculta del mapa tampoco participa en la selección.
    Centro de cada polígono vía `L.geoJSON(gj).getBounds().getCenter()`; `pointInPolygon()` (ray-casting)
    para polígono/lazo (mismo test — ambos terminan siendo un anillo de puntos), `distanceTo()` nativo de
    Leaflet para radio.
  - El shape dibujado se queda visible (`selectionShapeLayer`, aparte de `allPolyLayerGroup` y de las capas
    complementarias) hasta la próxima selección o hasta cerrar el panel; lo seleccionado se resalta con
    contorno grueso en su color (`selectionHighlightLayer`). `renderAllPolygonsMap()` limpia cualquier
    selección activa al inicio de cada render (`clearMapSelection()`).
  - **Fix real durante la implementación**: a diferencia de polígono/radio (que restauran `dragging` y el
    estado del botón vía `deactivateMapSelectTool()` dentro del handler de `CREATED`), el lazo no lo hacía
    al soltar el mouse — el mapa se quedaba con el arrastre deshabilitado. Fix: `onLassoMouseUp()` también
    llama `deactivateMapSelectTool()`. Relacionado: el trazo final del lazo desaparecía al terminar (a
    diferencia de polígono/radio) — se ajustó para que se conserve como shape final, solo se borra si se
    cancela a medio dibujar.
  - Panel de resultados (`#mapSelectPanel`, hijo de `#allPolygonsMapEl` — Leaflet ya pone
    `position:relative` en su contenedor) agrupado por entidad, cada fila → `editFromMap(ent,id)`
    (reutilizado tal cual).

### Panel de filtros dinámicos del Mapa — v3.7.1

Panel a la **derecha** del mapa (`#mapFilterSidebar`, dentro de un nuevo wrapper `.map-body` insertado entre
`.map-toolbar` y `#allPolygonsMapEl` — `.mapview` sigue siendo `flex-direction:column` para que el toolbar
quede a todo lo ancho arriba; solo `.map-body` es `row`) para filtrar en vivo qué polígonos se dibujan/son
seleccionables en el mapa, anclando campos de cualquiera de las 3 entidades con polígono (`MAP_LAYERS`:
Propiedades/Parques/Terrenos). Completamente aparte de `filters[ent]` (el del dropdown de columna de la
tabla) — nunca lo toca ni es tocado por él.

- **Región/Estado/Mercado/Submercado son filtros COMPARTIDOS** (pedido explícito del usuario): anclar uno
  de estos 4 filtra las 3 entidades A LA VEZ con un solo selector, en vez de uno por entidad. Cualquier otro
  campo anclado es específico de una sola entidad. Mismo criterio ya establecido en el código para estos 4
  campos: `CASCADE_ORDER` ya los trataba como grupo transversal (cascada Región→Estado→Mercado→Submercado),
  y `resumenMarket`/`resumenMarketMatches` en el Resumen ya era "un filtro, varias entidades" con 1 campo.
- **Estado**: `mapFilters={shared:{}, perEntity:{PROPIEDADES:[],PARKS:[],LAND:[]}}` (localStorage
  `citius_map_filters_v1`, mismo shape que `filters[ent]` por entrada: `{key,type:"list",values}` /
  `{key,type:"number",min,max}` / `{key,type:"text",text}`) y `mapFilterPanelFields={shared:[],
  perEntity:{...}}` (qué campos están anclados, separado del valor — localStorage
  `citius_map_filter_panel_fields_v1`).
- **`rowMatchesMapFilters(ent,row)`**: recorre `CASCADE_ORDER` para los filtros compartidos — si la entidad
  no tiene ese campo (Terrenos no tiene `REGION`/`STATE`, solo `MARKET`/`SUBMARKET`) esa dimensión se
  **ignora** para esa entidad (nunca se trata como "no coincide"); luego recorre `mapFilters.perEntity[ent]`
  para los específicos. Reutiliza `fieldMatchesFilter` tal cual (misma función del filtro de columna).
  Conectado en 2 puntos ya existentes, solo agregando la condición sin tocar su lógica: el loop que arma
  `items` en `renderAllPolygonsMap()` y el loop que arma `matches` en `handleMapSelection()` (lazo/radio/
  polígono) — así lo filtrado tampoco es seleccionable, igual que ya se comporta `mapLayerFilter` (el
  checkbox de mostrar/ocultar capa entera) en esos mismos 2 lugares.
- **Panel (`renderMapFilterSidebar()`)**, mismo mecanismo de "aplicar en vivo sin perder foco" que cualquier
  filtro con inputs de texto/número en esta app: checkboxes de lista mutan el estado y llaman
  `renderAllPolygonsMap()` al toque; número/texto con debounce (~350ms) sin re-renderizar el panel completo
  desde ese flujo (solo estado + mapa + contador `<span>` de "N activo(s)"). Sección "Ubicación (Propiedades,
  Parques y Terrenos)" con los campos compartidos anclados (checkboxes, valores = unión real de las 3
  entidades vía `mapSharedFieldValuesInUse(key)`, siempre tratados como lista aunque no haya catálogo de
  cascada cargado — son categóricos por naturaleza), y una sección por entidad para sus campos específicos
  anclados. `renderMapFilterSidebar()` se llama explícito en `showMapView()`, al anclar/quitar un campo, y en
  "Limpiar filtros" — nunca enganchado dentro de `renderAllPolygonsMap()` mismo (mismo motivo que en
  cualquier otro panel de filtros de esta app: el propio panel dispara ese render con debounce mientras se
  escribe, y re-renderizarse a sí mismo desde ahí perdería el foco del input a media escritura).
- Reutiliza las clases `.filter-sidebar-*`/`.col-filter-*` (mismo look en toda la app, sin duplicar CSS) +
  `.filter-sidebar-group-label` para los encabezados de sección.
- **Cuidado real**: algunas keys de campo traen paréntesis (ej. `ELECTRIC_SERVICE_(kv)` en Terrenos) — todo
  acceso a los widgets usa `document.getElementById("mapFilterField_"+scope+"_"+key)` (no parsea selector),
  nunca un selector CSS armado a mano con la key embebida.
- **Un campo tipo lista con 0 valores marcados SÍ se guarda como filtro real** (`{key,type:"list",values:[]}`,
  "no mostrar nada") — a diferencia de `fieldMatchesFilter` (compartida con el dropdown de columna de la
  tabla, sin tocar), que trata una lista vacía como "sin restricción". `rowMatchesMapFilters` nunca delega
  ese caso puntual a `fieldMatchesFilter` — usa `mapFieldFilterMatches(rowVal,f)`, que intercepta
  `type==="list" && values.length===0` y regresa `false` antes de delegar. Fix real (v3.7.3): sin este
  intercepto, `setMapFilterListValues` colapsaba "0 seleccionados" al mismo estado que "sin filtro" (ambos
  `null`), así que el botón "Seleccionar todo" no podía deseleccionar todo cuando ya estaba completo — el
  panel se re-renderizaba mostrando todo marcado de nuevo en cada intento.
- **Nota histórica**: el primer intento de esta entrega (aún v3.7.0) puso este panel dentro de las TABLAS de
  Propiedades/Parques/Terrenos, editando `filters[ent]` — el usuario aclaró que lo quería en el Mapa, sobre
  qué polígonos se muestran, con Región/Estado/Mercado/Submercado compartidos entre las 3 entidades. Ese
  diseño intermedio fue revertido por completo antes de construir el de arriba.

## Listas condicionales (cascada Región→Estado→Mercado→Submercado)

Único caso de "lista condicional" en la app — construido específico para estos 4 campos (`REGION`, `STATE`,
`MARKET`, `SUBMARKET`, existentes en `PROPIEDADES`/`PARKS`/`TRANSACCIONES` — esta última los ganó en
v3.17.0, agrupados en su propia sección "Ubicación"; Terrenos es la única excepción, solo tiene
`MARKET`/`SUBMARKET`), no un motor genérico para cualquier lista futura.

- **Catálogo**: hoja `CATALOGO_UBICACION` (nombre exacto) dentro del Archivo de Configuración — mismo lugar
  y mismo patrón de prioridad que `CONFIG_CAMPOS`/`(LD) Lista Desp` (gana el archivo de configuración si está
  vinculado; si no, se intenta leer del Excel de datos). Encabezados de columna flexibles vía
  `CASCADE_HEADER_ALIASES` (acepta español `REGION/ESTADO/MERCADO/SUBMERCADO` o inglés
  `REGION/STATE/MARKET/SUBMARKET`) — el usuario puede copiar directamente las columnas de su catálogo real
  sin renombrar nada. **Es de solo lectura**: no hay UI para editar filas (no tiene sentido para miles de
  filas); se mantiene directamente en Excel y el sistema solo lo relee cuando detecta que el archivo cambió.
  - **Por ser de solo lectura, la app NUNCA debe reconstruir esta hoja desde un modelo interno reducido —
    debe preservar su AOA crudo tal cual.** Hubo un bug real de pérdida de datos (fix en v3.1.1): como
    `buildWorkbook()`/`buildConfigWorkbook()` reconstruyen su archivo desde cero en cada guardado, y
    ninguna sabía de esta hoja, `CATALOGO_UBICACION` se borraba del archivo en disco en el primer guardado
    después de vincularlo. Fix: `cascadeCatalogAoa` (el AOA completo, todas las columnas —incluso las que
    la app no modela, ej. `RESPONSABLE`— no solo las 4 de `cascadeCatalog`) se captura en
    `loadCascadeCatalogFromSheet` apenas se encuentra la hoja, y `appendCascadeCatalogSheet(wb)` la
    re-agrega tal cual en ambas funciones de construcción de workbook. Si se agrega alguna otra hoja
    de solo lectura en el futuro (sin editor propio dentro de la app), replicar este mismo patrón —
    guardar el AOA crudo al leer, no derivarla de nuevo del modelo en memoria al escribir.
- `cascadeCatalog` (array en memoria de `{REGION,STATE,MARKET,SUBMARKET}`), `CASCADE_ORDER`,
  `cascadeParentKeyFor(ent,key)` (busca el ancestro más cercano en la cadena que sí exista como campo en esa
  entidad — para `LAND` (Terrenos, sin `REGION`/`STATE`) regresa `null` en `MARKET`, así que ese campo se
  muestra sin filtrar; `TRANSACCIONES` tiene los 4 desde v3.17.0, así que encadena completo igual que
  Propiedades/Parques), `cascadeOptionsFor(key,parentKey,parentValue)` (solo filtra si el padre existe Y ya
  tiene valor; si el padre está vacío, muestra todas las opciones).
- **Autofill de ubicación en Transacciones desde `MAPPING_CODE`** (v3.18.0): al capturar/editar el código
  de mapeo, `autofillTransactionFromMappingCode(inputEl)` (`oninput` del campo, agregado en el fallback
  final de `renderStandaloneField`) busca la Propiedad correspondiente (`ciEq` exacto contra
  `state.PROPIEDADES`) y trae `PROPERTY_NAME` + los 4 campos de ubicación — **no son de solo lectura**, el
  usuario los puede corregir después; solo actúa con coincidencia exacta (código incompleto/typo no toca
  nada ya capturado), y sí sobreescribe si el código cambia a otra propiedad. Como los 4 campos de ubicación
  pueden ser `<select>` (cascada, con catálogo) o `<input>` de texto (sin catálogo), la función maneja
  ambos: para `<select>`, reconstruye su `innerHTML` con `cascadeOptionsFor` usando el valor YA asignado al
  padre en el MISMO ciclo (recorre `CASCADE_ORDER` en orden — Región primero, así Estado ya puede filtrar
  por la Región recién puesta) en vez de solo asignar `.value` (que fallaría en silencio si el valor
  buscado no está entre las `<option>` que el `<select>` tenía renderizadas de ANTES del cambio de
  Propiedad) — mismo patrón que `onCascadeFieldChange`, pero seleccionando el valor objetivo en vez de
  vaciar a `""`.
  - **`SIMPLE_AUTOFILL_MAP`** (v3.21.0): mapa genérico `{keyDestinoEnTransacciones: keyOrigenEnPropiedades}`
    para campos que se traen igual desde la Propiedad al capturar/editar el `MAPPING_CODE`, pero que —a
    diferencia de los 4 de ubicación— NO encadenan entre sí, así que no necesitan reconstrucción de
    `<select>` ni recorrido en orden: alcanza con `el.value=prop[sourceKey]||""` sea el destino `<select>`
    o `<input>` de texto. Hoy tiene 2 entradas: `TRANSACCION_TYPE` (Transacciones) ← `TRANSACTION_AVAILABLE`
    (Propiedades), y `BROKER_BUILDING` (Transacciones, texto) ← `LISTING_BROKER` (Propiedades, texto). Antes
    de agregar `TRANSACCION_TYPE` se verificó que su lista (`TRANSACTION_TYPE`) es superset de la lista de
    `TRANSACTION_AVAILABLE` (`TRANS_AVAILABLE`) — así el valor copiado siempre calza con una opción real del
    `<select>` destino, nunca queda un valor "huérfano" sin opción seleccionable. Si se agrega otro campo
    simple a futuro con la misma necesidad, agregarlo a este mapa en vez de escribir un caso especial nuevo;
    si en cambio el campo nuevo encadena con otro (como los 4 de ubicación), no calza aquí — necesita su
    propio manejo de reconstrucción como el de arriba.
- En la ficha (`renderStandaloneField`), estos 4 campos se renderizan como `<select
  onchange="onCascadeFieldChange(this)">` **solo si `cascadeCatalog.length>0`** — sin catálogo cargado
  siguen exactamente igual que siempre (`<input>` de texto con datalist), cero regresión.
  `onCascadeFieldChange` recorre la cadena hacia adelante desde el campo cambiado, reconstruye cada
  `<select>` descendiente y resetea su valor — leyendo el valor del padre directamente del DOM, ya que esta
  app **no mantiene un objeto "draft" en memoria durante la edición** (confirmado en `saveBtn.onclick`, línea
  ~2749 en el momento de este cambio): el DOM es la única fuente de verdad mientras se edita un registro.
- Valores históricos que ya no están en el catálogo se conservan seleccionados con el sufijo "(no está en la
  lista)" — mismo patrón que `listOptionsHtml` ya usaba para listas normales, para no perder datos reales de
  registros capturados antes de tener el catálogo.
- En Configuración > Campos, la celda "Lista" de estos 4 campos muestra un aviso ("Cascada
  Región/Estado/Mercado/Submercado") en vez del selector normal, solo cuando hay catálogo cargado.
- **`detectFieldType(ent,key)`** (usado por el filtro de columna de la tabla, línea ~1693) tiene su propio
  chequeo `CASCADE_ORDER.includes(key) && cascadeCatalog.length` para clasificar estos 4 campos como tipo
  `"list"` (filtro de checkboxes, igual que MARKET_STATUS) en vez de `"text"` (filtro de "contiene"). Es un
  chequeo INDEPENDIENTE del que usa `renderStandaloneField` para decidir el `<select>` de la ficha — mismo
  criterio, pero dos lugares distintos en el código. Si se agrega un tercer lugar que necesite saber "¿este
  campo es de cascada?", replicar el mismo chequeo ahí también en vez de asumir que hay un solo punto de
  verdad para esto.
  - **`detectFieldType` respeta `cfg.type` directamente** (fix real, v3.14.1): `"int"`/`"decimal"` → filtro
    numérico ("Desde/Hasta"); `"text"` → filtro de lista (checkboxes) — el muestreo de hasta 30 valores
    guardados para adivinar "número" vs "texto" por `typeof` es solo el ÚLTIMO recurso, cuando no hay
    `cfg` en absoluto. Antes de este fix, el muestreo corría SIEMPRE (ignorando `cfg.type` por completo)
    — un campo configurado como "Texto" (ej. `ID_PARK`) podía terminar con el filtro numérico solo porque
    sus valores capturados resultaron ser JS numbers (`maybeNum()`, en `saveBtn.onclick`, convierte
    cualquier texto con pinta de número a JS number sin importar el tipo configurado — eso NO se tocó, sigue
    igual, para preservar el orden numérico correcto al ordenar la columna; el fix es solo sobre qué widget
    usa el filtro). 2 excepciones a propósito, ya especiales por su propia semántica desde antes de este
    fix: `SYSTEM_TIMESTAMP_FIELD_KEYS` (siempre `"text"` — un timestamp casi nunca se repite, listarlo como
    checkboxes no serviría) y `PARK_COMPUTED_FIELD_KEYS` (siempre `"number"` — son conteos/montos
    genuinamente numéricos aunque su `cfg.type` por default también sea "text").
- **Filtros de columna en cascada, estilo Autofiltro de Excel** (v3.15.0): el dropdown de una columna tipo
  lista NO ofrece todos los valores distintos de toda la tabla — `columnFilterValuesInUse(ent,key)` (no
  confundir con `fieldValuesInUse(ent,key)`, la versión de uso general SIN cascada, usada por Resumen y el
  panel de filtros del Mapa, que a propósito NO se tocó) calcula los valores solo entre las filas que YA
  pasan los DEMÁS filtros de columna activos (`filters[ent].filter(f=>f.key!==key)` — nunca el de la
  columna misma, sería circular) reutilizando `fieldMatchesFilter`/`resolveFilterValue`. Ejemplo: con
  `MARKET="Saltillo"` filtrado, el dropdown de `SUBMARKET` solo ofrece los submercados que de verdad
  existen dentro de Saltillo. Usada en los 3 puntos del flujo del dropdown: `toggleColumnFilter` (al
  abrirlo), `colFilterVisibleValues()` (buscar/listar), `applyColumnFilter()` (decidir si "todo marcado"
  equivale a "sin filtro" para esa columna).
- **Buscador del filtro de columna, estilo Excel (v3.26.0)** — `onColFilterSearch(val)` ya no solo
  angosta la lista de checkboxes visibles: en CADA tecla (incluso al borrar la búsqueda por completo) hace
  `colFilterState.values=new Set(colFilterVisibleValues())` — lo que SÍ calza con la búsqueda queda
  seleccionado, lo que NO calza (oculto de la lista) se deselecciona. Con búsqueda vacía, "lo que se ve"
  vuelve a ser todo, así que todo se reselecciona — misma regla, sin caso especial. Pedido explícito del
  usuario ("el mismo comportamiento que tiene Excel"): antes, escribir en el buscador y darle "Aceptar" sin
  tocar ningún checkbox a mano dejaba la selección VIEJA intacta (normalmente "todo", sin filtro real) en
  vez de filtrar a lo que se veía en pantalla. El usuario puede seguir ajustando la selección a mano después
  de escribir — `onColFilterToggleValue` no cambió, sigue operando sobre el mismo `colFilterState.values`
  sin importar cómo se haya poblado.
- **Enter aplica el filtro (v3.26.1)** — los 3 `<input>` del dropdown (buscador de tipo lista, "Desde"/
  "Hasta" de tipo número, "Contiene…" de tipo texto), armados en `renderColumnFilterDropdown`, comparten un
  fragmento `onkeydown="if(event.key==='Enter') applyColumnFilter()"` (`applyOnEnter`, una sola vez) —
  Enter en cualquiera de ellos llama exactamente la misma función que el botón "Aceptar", además de poder
  seguir usando el botón como siempre.
- **Slicer con histograma para el filtro numérico, estilo Zillow (v3.27.0)** — cuando `detectFieldType`
  clasifica la columna como `"number"` (recordar: eso depende de `fieldConfig[ent][i].type==="int"|
  "decimal"`, configurado en Configuración > Campos — un campo con `type:"text"`, el default, SIEMPRE cae
  en `"list"` sin importar que sus valores parezcan números, ver nota de `detectFieldType` arriba), el
  dropdown "Desde"/"Hasta" de siempre gana un histograma de 20 barras + un control de rango de doble
  agarradera dibujado ENCIMA del histograma, pedido a partir de una captura de referencia de
  zillow.com/rentals.
  - **`numericColumnValuesInUse(ent,key)`**: misma cascada "respeta los demás filtros activos" que
    `columnFilterValuesInUse`, pero regresa números (vía `resolveFilterValue`, ya convertidos por
    Métrico/Imperial si aplica) en vez de strings.
  - **`buildColFilterHistogram(ent,key)`**: 20 bins (`COL_FILTER_HIST_BINS`) parejos entre el mínimo/máximo
    real de `numericColumnValuesInUse`, cada uno con su `count`/`pct` (normalizado al bin más alto).
    `hasRange:false` con 0 o 1 valor distinto — en ese caso el dropdown se ve exactamente como antes de
    v3.27.0 (solo los inputs simples), cero regresión para columnas sin variación suficiente. Se calcula
    UNA vez al abrir el dropdown (`toggleColumnFilter`, guardado en `colFilterState.hist`), no en cada
    interacción.
  - **Doble agarradera**: 2 `<input type="range" step="any">` superpuestos en `.col-filter-range-track`
    (mismo contenedor), con `pointer-events:none` en el input y `pointer-events:auto` solo en
    `::-webkit-slider-thumb`/`::-moz-range-thumb` — el truco estándar para que ambas agarraderas se puedan
    arrastrar de forma independiente sin que el "track" invisible de una bloquee el arrastre de la otra.
    `.track-fill` (un `<div>` absoluto entre las 2 agarraderas) marca visualmente el rango seleccionado.
  - **3 formas de mover el rango, siempre sincronizadas**: arrastrar una agarradera
    (`onColFilterRangeInput(which,val)`) o escribir directo en "Desde"/"Hasta"
    (`onColFilterNumberTyped(which,val)`) — cualquiera actualiza la otra Y llama
    `updateColFilterRangeVisuals()` (repinta `.track-fill` y agrega/quita `.in-range` a cada barra del
    histograma comparando sus límites `data-bin-min`/`data-bin-max` contra el rango actual). Arrastrar el
    mínimo más allá del máximo (o viceversa) se clampa contra la otra agarradera en el momento, nunca las
    deja cruzarse.
  - El dropdown crece de 240px a 288px SOLO cuando `hist.hasRange` es verdadero — el resto de los casos
    (lista, texto, número sin suficiente variación) se quedan en el ancho de siempre.
  - **`applyColumnFilter()`/`clearColumnFilter()` no cambiaron** — siguen leyendo `colFilterState.values.
    min/max` igual que siempre, sin que les importe si esos valores vinieron de arrastrar el slider o de
    escribir a mano.
  - **Formato de "Desde"/"Hasta": 2 decimales máximo + coma de miles (v3.27.1)** — `fmtColFilterNum(n)`
    (`toLocaleString("en-US",{maximumFractionDigits:2})`) se usa tanto ahí como en los labels de extremos
    del histograma (antes usaban `fmtNum`, sin tope de decimales). `onColFilterRangeInput` redondea con
    `round2()` (ya existente, reusado) antes de guardar — sin este redondeo, arrastrar el slider (`step=
    "any"`) dejaba valores con 10+ decimales. Los inputs "Desde"/"Hasta" son `type="text"` (no `"number"`,
    que NUNCA puede mostrar comas — restricción del navegador), con `onColFilterNumberTyped` limpiando
    comas antes de guardar (crítico: `colFilterState.values.min/max` nunca debe llevar comas, o
    `Number(...)` en `fieldMatchesFilter`/`applyColumnFilter` daría `NaN`) y un handler nuevo
    `onColFilterNumberBlur` que reformatea con coma SOLO al perder el foco (no en cada tecla, para no
    mandar el cursor al final mientras se está escribiendo/editando en medio del número).

## Vista de unidades en tabla (Métrico / Imperial / Como se registró)

El usuario captura con mezcla de unidades (ej. SF y m² para el mismo campo entre registros distintos) —
este selector, arriba de las tablas de Propiedades/Parques/Transacciones, normaliza el DISPLAY sin tocar el
dato guardado. **Una sola preferencia compartida** entre las 3 tablas (`unitViewPref`, `localStorage`
`citius_unit_view_pref_v1`, mismo patrón que `columnPrefs`) — decisión explícita del usuario, no es por
tabla. **Nunca afecta la ficha** — abrir/editar un registro siempre muestra el valor y unidad exactamente
como se guardaron, sin importar la vista de tabla activa.

- Solo convierte dimensiones físicas con lado métrico Y lado imperial definidos: área (`SF_M2`: SF↔m²,
  ×0.092903), longitud (`FT_M`: ft↔m ×0.3048; `CM_IN`: in↔cm ×2.54), peso (`KG_LB`: lb↔kg ×0.453592).
  **No convierte moneda** (`MXN_USD`) — decisión explícita del usuario, no hay tipo de cambio fijo que usar.
  `HAS_M2` (hectáreas/m², ambas métricas) nunca se toca bajo ninguna vista — no existe un lado imperial para
  ese campo.
- Tarifas compuestas moneda+área (lista `CON_COMERCIAL`: "USD/SF", "MXN/m2", etc.) — `convertRateUnit()`
  convierte SOLO la parte de área, la moneda queda intacta. Importante: una tarifa (\$ por unidad de área)
  escala AL REVÉS que un valor de área plano — `$/m² = $/SF ÷ factor` (no `× factor`), porque m² es una
  unidad de área más grande que SF. Si se agrega otra dimensión compuesta en el futuro, replicar ese cuidado
  con la dirección del factor, no asumir que siempre es multiplicar.
- `convertCellForView(ent,valueKey,unitKey,rawValue,rawUnit)` es el único punto de conversión, usado en los
  3 lugares que el usuario pidió explícitamente que respetaran la vista activa (no solo el display visual):
  la celda combinada "valor unidad" en `renderTable()`, `applySort()` (de paso arregla un bug ya existente:
  ordenar una columna con unidades mezcladas ordenaba por magnitud cruda sin sentido dimensional) y
  `applyFilters()` vía `resolveFilterValue()` (el filtro numérico mín/máx compara contra el valor ya
  convertido). El filtro tipo-lista de la propia columna de UNIDAD sigue sobre valores crudos — alcance
  consciente, no extendido ahí.

## Ficha de edición/vista de un registro

- **"Borrar fecha" en el calendario personalizado (v3.43.6)** — pedido explícito del usuario (reportado
  desde Transacciones, pero el campo de fecha es el mismo componente compartido por toda la app,
  `dateFieldHtml`/`.date-picker-popover`): antes, una vez elegida una fecha, no había forma de dejar el
  campo vacío de nuevo si se había seleccionado por error — solo se podía reemplazarla por OTRA fecha, nunca
  quitarla. `clearDatePicker()` (mismo mecanismo que `pickDate(iso)`, pero deja `dateValue_<key>`/
  `dateDisplay_<key>` en `""` en vez de un ISO) se agregó al pie del calendario (`.date-picker-foot`, junto
  al botón "Hoy" ya existente), como **"Borrar fecha"** — solo aparece si el campo YA tiene una fecha
  seleccionada (`selectedIso` en `renderDatePicker()`), nada que borrar si está vacío. Al guardar, como
  `saveBtn.onclick` reconstruye el registro entero desde cero leyendo `#mBody [data-key]` (ver
  `## Modelo de datos`), un valor vacío simplemente no se agrega al objeto nuevo — el campo queda
  efectivamente borrado del registro guardado, no solo visualmente vacío en la ficha.
- **Emparejado valor+unidad** (`pairFieldsByUnit`): junta un campo `..._VALUE`/`..._VALOR` con su
  `..._UNIT`/`..._UNIDAD` correspondiente en un solo control combinado (input + select), en vez de dos
  campos sueltos. La mayoría de los pares del Excel usan el sufijo en inglés (`_VALUE`/`_UNIT`), pero
  `ASKING_SALE_PRICE_VALOR`/`ASKING_SALE_PRICE_UNIDAD` (Propiedades y Parques) vino con sufijo en español
  — `pairFieldsByUnit` reconoce ambos pares de sufijos (los dos tienen 6 caracteres, así que el mismo
  `slice(0,-6)` para sacar el prefijo/label funciona igual para ambos). Si aparece otro par de campos que
  no se agrupa visualmente, es casi siempre por esto: revisar si sus sufijos calzan con alguno de los dos
  patrones reconocidos.
- `openView(id)` abre en modo **solo lectura** (`buildViewForm`); el botón "✎ Editar" en la esquina
  superior derecha del modal cambia a modo edición (`switchToEditMode`) preservando la pestaña activa.
- `openEdit(id)` abre directo en modo edición; en modo edición aparece en su lugar el botón
  **"↩ Salir modo edición"**, que regresa a modo vista sin cerrar la ficha (`exitEditMode`), también
  preservando la pestaña activa.
- **Guardar no cierra la ficha**: al hacer clic en "Guardar" el registro se persiste, la tabla se
  refresca, y la ficha se vuelve a montar en modo edición en la misma pestaña (no en modo vista, no
  cerrada). Si es un registro nuevo, `editingId` se fija al id recién creado para que un segundo "Guardar"
  actualice en vez de duplicar.
- **Validación al guardar — guardado parcial permitido**: un campo `required` vacío **ya no bloquea el
  guardado** — se persiste igual todo lo que sí se capturó (pensado para cuando no se tiene a la mano toda
  la información y se va llenando la ficha por partes). Tras guardar, si aún quedan campos obligatorios
  vacíos, se resaltan en rojo (`.field-error`) y sale un toast tipo `"Guardado. Aún falta información en N
  campo(s) obligatorio(s)"` (desaparece solo a los ~2.2s) — el badge de la sección (ver abajo) baja al
  número real de faltantes tras guardar. Lo que **sí sigue bloqueando el guardado por completo** (nada se
  persiste) es un valor con formato inválido (`type:"int"` con texto no numérico), porque eso es un error a
  corregir, no información pendiente; en ese caso se resalta el campo, se cambia a su pestaña, y el toast
  es "Hay un valor inválido en algún campo" (o combinado con el aviso de obligatorios si aplican ambos). A
  propósito el toast nunca enumera los nombres de los campos — el detalle de cuáles son se ve por el
  resaltado rojo en el propio formulario, no en el mensaje.
- **El borde rojo de obligatorio vacío ya no espera a un intento de guardado** (v3.8.0, pedido explícito del
  usuario): `markEmptyRequiredFields()` corre al final de `enterEditMode(rec)` (cubre tanto abrir un
  registro existente como "Nuevo registro", ambos pasan por esa misma función) y marca `.field-error` en
  cualquier campo `required` vacío apenas se monta el formulario — antes solo se veía en rojo después de dar
  clic en "Guardar" al menos una vez. `fieldInputIsEmpty(inp)` (checkbox: no marcado; el resto:
  `String(value).trim()===""`) es la misma regla de "vacío" que ya usaba `saveBtn.onclick` para calcular
  `missingKeys`, extraída a una función propia para no duplicar esa lógica.
- **Campo de fecha+hora calculado por el sistema (v3.9.0/v3.9.1)** — en las 4 entidades: `LAST_MODIFIED` en
  Propiedades/Parques/Transacciones (campo nuevo, v3.9.0), y `LAST_UPDATE` en Terrenos (campo que YA
  existía en `BUNDLE.LAND.fields`/el Excel de esa entidad como fecha normal editable a mano — se le cambió
  el comportamiento en v3.9.1 para que funcione igual que los otros 3, sin agregar un campo nuevo ni tocar
  su key). Calculado por el sistema, nunca editable: `saveBtn.onclick` pisa el valor de cada campo que
  coincida en cada guardado exitoso (después del bloque que aborta si hay `invalidKeys`), sin importar qué
  traiga el formulario. Cada uno es un campo real de `fields(ent)` (grupo "IDENTIFICACIÓN" en Propiedades/
  Parques/Terrenos, "DETALLES DE TRANSACCIÓN" en Transacciones — esa entidad no tiene grupo IDENTIFICACIÓN)
  — por serlo, ya es seleccionable desde el picker "Columnas" de la tabla sin trabajo extra.
  - **`SYSTEM_TIMESTAMP_FIELD_KEYS`** (`Set(["LAST_MODIFIED","LAST_UPDATE"])`, junto a `POLYGON_FIELD_KEYS`)
    — especialización por KEY exacta (no por `type`), sin colisión entre entidades: cada key vive en una
    sola entidad. Si se agrega este mismo comportamiento a otra entidad en el futuro, basta con agregar su
    key a este Set — el resto del mecanismo (`renderStandaloneField`/`buildViewForm`/`renderTable()`/
    `saveBtn.onclick`) ya itera genéricamente sobre `fields(current)`/`f.key`, sin mencionar ninguna entidad
    en particular.
  - **No es un campo `type:"date"`** (esos se guardan sin hora, `YYYY-MM-DD`) — se intercepta el dispatch
    normal en `renderStandaloneField`/`buildViewForm`/`renderTable()` ANTES de mirar `fieldConfig[ent].type`
    (irrelevante para este campo, sin usarse). `formatDateTimeDisplay(iso)`/`dateTimeOrDashHtml(iso)` (junto
    a `formatDateDisplay`/`dateOrDashHtml`) formatean `DD-MM-AAAA HH:MM` en hora LOCAL del navegador (el
    valor guardado es UTC, `toISOString()`).
  - En la ficha se muestra igual que el campo de polígono: `<input readonly disabled>` con el valor
    formateado + un `<input type="hidden" data-key="...">` con el ISO real (clase CSS
    `.field-readonly input:disabled`, mismo estilo apagado que `.polygon-cap input:disabled`). Nunca
    `required` (default `false`), así que nunca cuenta en el % Completado ni se marca en rojo.
  - **Cero cambios de Excel necesarios**: el guard ya existente en `loadFromWorkbook()`/`loadLandFromWorkbook()`
    que agrega a `layouts[ent].headerKeys` cualquier campo de `fields(ent)` que falte en el header real (el
    mismo mecanismo que en su momento agregó `GEOJSON_POLIGONO` a un Excel viejo de Parques) agrega esta columna
    sola la próxima vez que se abra un Excel existente — queda vacía en filas viejas, se llena sola
    conforme se van guardando registros.
- **Asterisco de obligatorio** (`.field-required`, span rojo tras el label) se muestra siempre, tanto en
  modo vista (`buildViewForm`) como en modo edición (`buildForm`) — no solo en edición.
- **Conteo de obligatorios vacíos por sección**: cada botón del menú izquierdo de la ficha (`.form-nav
  button`) muestra un badge rojo (`.form-nav-badge`) con cuántos campos `required` de esa sección/grupo
  están vacíos en el registro actual (`countMissingRequired`). En modo vista se calcula una sola vez al
  abrir; en modo edición se **recalcula en vivo** mientras se escribe (listener de `input` delegado en
  `#mBody`, función `updateFormNavBadges`), así que baja en tiempo real conforme se llenan los campos. Si
  una sección no tiene faltantes, no se muestra badge (no se pinta un "0").
- **Aviso de "valor no coincide con la lista"** (v3.5.0, `fieldValueMismatch(ent,f,record)` + clase CSS
  `.field-value-mismatch`, fondo rojo claro `rgba(226,72,61,.14)`): pensado para cargas masivas desde Excel
  que puedan traer valores que no coincidan con ninguna lista configurada. Se ve tanto en modo vista
  (`.view-val`/`.view-unit`) como en modo edición (el `<select>`, que además ya mostraba de por sí el
  sufijo "(no está en la lista)" — misma comparación exacta reutilizada, así los dos SIEMPRE coinciden).
  Aplica a campos de lista normal y a los 4 de cascada. **Es puramente informativo**: `countMissingRequired`
  y `completionStats` (ver arriba) solo miran si el campo tiene *algún* valor, nunca llaman a esta función
  — un campo con un valor "incorrecto" cuenta como lleno igual, no aparece en los badges del menú
  izquierdo ni baja el % de completado. Deliberadamente distinta de `.field-error` (borde rojo, sí bloquea
  guardar) — esta es solo fondo rojo claro, para no confundir "revisar este dato" con "error que impide
  guardar".
- En la tabla de registros (Propiedades/Parques/Transacciones), hacer clic en **cualquier parte de la
  fila** abre la ficha en modo vista; solo el **círculo amarillo** al inicio de la fila selecciona/quita el
  registro de la presentación (esto fue un cambio explícito del usuario — no restaurar el comportamiento
  anterior de "clic en el Mapping Code").

### Pin picker de `LATITUDE`/`LONGITUDE` — v3.22.0

Botón **"Ubicar en el mapa"** (mismo estilo que "Dibujar polígono", ver `## Mapa` arriba) que aparece justo
debajo del campo `LONGITUDE` en la ficha de Propiedades, Parques y Terrenos — un picker de pin estilo
"Uber Eats": el pin queda SIEMPRE fijo en el centro del contenedor, y el usuario arrastra el MAPA por
debajo para ubicarlo.

- **`openPinMap()`/`closePinMap()`/`initPinMapIfNeeded()`** (overlay `#pinOverlay`, mapa `#pinMapEl`) —
  calcado del mecanismo ya existente de `openPolygonMap()`/`polyMap`, mismo criterio de centro inicial: si
  el registro ya tiene `LATITUDE`/`LONGITUDE` válidos los usa como centro con zoom 16, si no cae al default
  `[25.5,-100.9]` zoom 13 que ya usa el modal de polígono. **Basemap Satelital desde v3.23.0** (antes CARTO
  Voyager/calles) — se cambió a los mismos 2 tile layers de Esri (World_Imagery + Boundaries_and_Places)
  que ya usa `initPolyMapIfNeeded()`, para que el picker de pin y el de polígono se vean consistentes entre
  sí (pedido explícito del usuario: "tiene que ser el mismo que se ve en la captura del polígono").
- **El pin central (`.pin-picker-marker`) NO es un `L.marker` de Leaflet** — es un `<div>` posicionado
  `absolute` (`left:50%;top:50%;transform:translate(-50%,-100%)`) por ENCIMA del mapa, hermano de
  `#pinMapEl` dentro de `.pin-map-wrap`, así que nunca se mueve al hacer pan — solo el mapa se mueve debajo.
  Reutiliza la forma visual de `.map-pin` (el mismo teardrop rojo que ya usan los pines numerados del mapa
  de Presentación) sin ningún cambio a esa clase.
- **Lectura de coordenadas en vivo** (`#pinMapCoords`, badge centrado abajo del mapa): `pinMap.on('move',
  updatePinMapCoords)` — se actualiza en cada frame mientras se arrastra, mostrando `lat, lng` a 6
  decimales (mismo evento nativo de Leaflet que ya usa `renderAllPolygonsMap`/`declutterMapPins` para otras
  cosas, sin polling ni debounce).
- **"Guardar ubicación"** escribe `pinMap.getCenter()` (redondeado a 6 decimales con `.toFixed(6)`, ~11cm de
  precisión) directo en `input[data-key="LATITUDE"]`/`input[data-key="LONGITUDE"]` de la ficha ABIERTA —
  igual que el polígono, es solo un pre-llenado: los campos siguen editables a mano después, y "Guardar
  ubicación" no persiste el registro por sí solo (eso sigue pasando solo con el botón "Guardar" de la
  ficha).
- **El botón solo se agrega si la entidad tiene AMBOS campos** (`renderStandaloneField`, al llegar a
  `f.key==="LONGITUDE"`: `flds.some(x=>x.key==="LATITUDE")`) — no requirió ningún cambio en
  `POLYGON_FIELD_KEYS` ni en otro mecanismo genérico, es un chequeo puntual solo para este botón.
  `.field-pinpicker{grid-column:1/-1}` (mismo patrón que `.field-polygon`) para que el botón ocupe todo el
  ancho de la grilla de campos, en vez de quedar en la misma celda angosta que el input de `LONGITUDE`.
- **Terrenos no tenía `LATITUDE`/`LONGITUDE`** antes de esta versión — se agregaron a `BUNDLE.LAND.fields`
  específicamente para poder ofrecer este picker ahí también (ver `## Terrenos` arriba para el detalle de
  esos 2 campos nuevos).
- **2da vía para llenar `LATITUDE`/`LONGITUDE`: centroide automático del polígono (v3.23.0)** — al darle
  "Guardar polígono" (`polySaveBtn.onclick`, mismo modal genérico de Propiedades/Parques/Terrenos), además
  de guardar el GeoJSON como siempre, ahora también llama `layers[0].getCenter()` y escribe el resultado
  (`.toFixed(6)`, mismo formato que el pin picker) en los `input[data-key="LATITUDE"/"LONGITUDE"]` de la
  ficha abierta. Confirmado con Chrome headless que Leaflet 1.9.4 (la versión cargada) calcula el centroide
  geométrico REAL del polígono (centro de masa), no el centro del bounding box — se verificó contra un
  triángulo de coordenadas conocidas, el resultado coincidió exactamente con el promedio de vértices
  esperado. **Solo se recalcula cuando el usuario entra activamente a "Dibujar polígono" y le da "Guardar
  polígono"** (registro nuevo o polígono redibujado) — abrir/guardar la ficha sin tocar el polígono nunca
  toca lat/long, a pedido explícito del usuario ("los registros que ya tienen datos no los modifiques, yo
  lo tendría que dar clic en editar los datos"). Ambas vías (este centroide y el pin picker de arriba)
  escriben a los mismos 2 `<input>`, así que cualquiera de las 2 puede corregir lo que dejó la otra — no
  hay conflicto ni orden de prioridad especial entre ellas, gana la que se use más recientemente.

### Pin numerado del mapa de Presentación, ahora clickeable — v3.22.0

El mapa de la página "Location Map" del deck (`renderPresMap`, pines `L.divIcon` con el número de cada
propiedad) tenía el número dentro de un `<span>` sin interacción. Ahora es un `<a href="https://
www.google.com/maps?q=lat,lng" target="_blank" rel="noopener">` — mismo mecanismo que el link de
`__LOCATION_PIN__` en la tabla comparativa (v3.20.0: Chrome preserva `<a href>` como hipervínculo real al
exportar a PDF). `.map-pin a{text-decoration:none}` (antes `.map-pin span`, sin ese estilo porque un
`<span>` no se subraya solo) — a pedido explícito del usuario, para que el número clickeable no se vea
distinto al pin de siempre (sin el azul/subrayado típico de link).

- **Recuadro gris alrededor de cada pin, SOLO en Vista Previa de macOS (v3.32.3, en investigación)** —
  reportado por el usuario con una captura: al abrir el PDF exportado directo en Vista Previa (Preview.app)
  cada pin numerado se ve con un recuadro/halo gris cuadrado detrás; el mismo PDF abierto en Chrome o
  Adobe Acrobat Reader se ve bien (solo la gota + su sombra, sin recuadro). **Diagnóstico**: `.map-pin` es
  un cuadrado de 26×26px con `border-radius:50% 50% 50% 0` + `transform:rotate(-45deg)` (el truco de
  "gota" hecho con CSS puro) más `box-shadow:0 2px 5px rgba(0,0,0,.4)` — el formato PDF no tiene un
  operador nativo que exprese "sombra difuminada sobre una forma rotada con esquinas redondeadas" de una
  sola vez, así que al exportar (Chrome "Imprimir → Guardar como PDF") es probable que Chrome rasterice el
  pin en una capa con transparencia (el cuadrado completo, con las esquinas fuera de la gota marcadas
  transparentes) en vez de vector puro. Chrome/Acrobat respetan esa transparencia; Vista Previa
  (Quartz/PDFKit) tiene un bug/limitación conocida con ese tipo de capa proveniente de PDFs de Chrome y
  pinta las zonas "transparentes" como gris sólido. **No es un problema del archivo ni de la app** — es una
  incompatibilidad de Vista Previa con cómo Chrome exporta esa combinación específica a PDF.
  - **Prueba aplicada (v3.32.3, pendiente de confirmación real)**: `@media print{.map-pin{box-shadow:none}}`
    — quita la sombra SOLO durante impresión/exportación a PDF (la pestaña Mapa en pantalla, fuera de
    `@media print`, conserva la sombra igual que siempre). La sospecha es que sin `box-shadow`, Chrome ya
    no necesite rasterizar esa capa con transparencia, y el recuadro gris desaparezca en Vista Previa. Sigue
    el mismo criterio de "un fix a la vez, confirmar con una impresión/PDF real antes de encadenar otro
    cambio" que el resto de los ajustes de impresión de esta app. **Si esto NO resuelve el recuadro**, el
    siguiente sospechoso sería el propio `border-radius`+`rotate` (tocarlo requeriría replantear el pin como
    SVG con un `<path>` real en vez del truco de cuadrado rotado — cambio más grande, no aplicado todavía).
- **Pedido explícitamente NO implementado, por inviable sin llave de API de pago**: mostrar TODOS los
  pines de la presentación a la vez al hacer clic en uno solo. El esquema de URLs gratuito de Google Maps
  no tiene un modo de "varios pines sueltos, sin ruta" — lo único que acepta más de 1 punto es el link de
  "Cómo llegar" (`/maps/dir/?api=1&origin=...&waypoints=lat,lng|...`), que traza una RUTA entre las paradas
  en orden y tiene un límite de ~9-10 paradas totales (menos en celular aún). El usuario decidió dejarlo
  como está (cada pin/link abre solo su propia ubicación) tras conocer esta limitación — si se retoma a
  futuro, esa es la única vía real, con el límite de paradas explicado arriba.
- **`target="_blank"` no garantiza pestaña nueva una vez exportado a PDF** — funciona en la vista previa en
  pantalla, pero dentro de un PDF ya generado el comportamiento del link (misma pestaña vs. nueva) lo decide
  el VISOR de PDF que lo abre (ej. el visor integrado de Chrome), no el HTML de origen — el formato PDF solo
  tiene una acción de URI, sin concepto de "target". No hay nada que cambiar en la app para forzar esto.

### Picker de registro genérico — v3.42.0 (Parque de Propiedades), generalizado v3.43.0 (Propiedad de Transacciones)

Pedido explícito del usuario: antes `ID_PARK`/`PARK_NAME` en la ficha de Propiedades eran 2 inputs de texto
libre, sin ninguna relación real garantizada con un registro de Parques — un typo o un nombre que no
calzara exacto rompía silenciosamente el match que ya usa el mapa aéreo de Layout (`relatedParkFor()`, ver
`## Presentación (deck)`). Se construyó un picker con tabla+búsqueda+orden para elegir el Parque real desde
un listado en vez de teclearlo. Cuando el usuario pidió "algo similar" para relacionar el `MAPPING_CODE` de
una Propiedad en la ficha de Transacciones, el picker se **generalizó** en vez de duplicarse (mismo modal,
mismo mecanismo de búsqueda/orden, solo cambian la entidad listada, las columnas mostradas y qué hacer con
la fila elegida) — ver `openRecordPicker(cfg)` en el JS, con 2 wrappers concretos hoy:

| | `openParkPicker()` (v3.42.0) | `openPropertyPickerForTransaction()` (v3.43.0) |
|---|---|---|
| Se abre desde | Ficha de Propiedades, campo `ID_PARK` | Ficha de Transacciones, campo `MAPPING_CODE` |
| Entidad listada | `state.PARKS` | `state.PROPIEDADES` |
| Columnas | ID Park, Park Name, Developer, Market, Submarket | Mapping Code, Property Name, Park Name, Developer, Market, Submarket |
| Fila sintética | "Stand Alone" (`ID_PARK:0`) | Ninguna — pedido explícito del usuario ("aquí no habrá el Stand Alone") |
| Al elegir | Escribe `ID_PARK`/`PARK_NAME` en la ficha | Escribe `MAPPING_CODE` y llama `autofillTransactionFromMappingCode(codeInp)` — la MISMA función que ya disparaba el `oninput` de este campo cuando era editable a mano, así `PROPERTY_NAME` + `SIMPLE_AUTOFILL_MAP` + la cascada Market/Submarket se autocompletan igual que siempre |

**Mecánica común (`openRecordPicker(cfg)`/`renderRecordPickerList()`/`selectRecordFromPicker(i)`)**:

- **Decisión confirmada con el usuario vía AskUserQuestion** (aplicada al 1er caso de uso, reafirmada tal
  cual para el 2do por ser "la misma idea" — palabras del usuario): el/los campo(s) de destino quedan **de
  solo lectura** (`readonly`, siguen siendo `data-key` normales — el texto mostrado y el valor guardado son
  el mismo string, no hace falta hidden+resumen aparte como el patrón de `GEOJSON_POLIGONO`), se llenan
  ÚNICAMENTE eligiendo desde el picker. El botón ("Seleccionar Parque…"/"Seleccionar Propiedad…") vive junto
  al campo readonly, clase `.field-recordpicker` (celda normal de 1 columna del grid de 4, cae justo a la
  derecha del input por simple orden de aparición en el HTML — ver "Botón al lado del campo" más abajo).
- **Listado**: `[...(cfg.extraRows?cfg.extraRows():[]), ...state[cfg.ent]]`. Una fila "extra" sintética
  (ej. "Stand Alone", `__extra:true`) **no vive en `state[ent]`** — se agrega al vuelo en cada render, se
  muestra en cursiva (`.record-picker-row.extra`) para distinguirla a simple vista, pero participa en
  búsqueda/orden como una fila más (decisión confirmada con el usuario: "fila normal", no fija sin importar
  la búsqueda) — solo se ve primera cuando el picker abre sin ningún sort aplicado.
- **Búsqueda global** (`#recordPickerSearch`, `oninput="renderRecordPickerList()"`) — filtra sobre TODAS
  las columnas configuradas en `cfg.columns`, case-insensitive, `String(...).includes(q)` sencillo (sin
  normalizar acentos). **Decisión confirmada con el usuario vía AskUserQuestion**: sin filtro por columna
  tipo Excel (`.col-filter-dropdown`) — se descartó a propósito para no duplicar esa maquinaria dentro de
  un modal que se abre/cierra rápido; alcanza con búsqueda global + orden por columna.
- **Orden por columna**: clic en el encabezado (`setRecordPickerSort(key)`, encabezado generado dinámico
  en `openRecordPicker()` a partir de `cfg.columns`) — asc → desc → asc, con una flechita (▲/▼) junto a la
  columna activa. Estado propio y AISLADO (`recordPickerSort`, variable global del picker, se resetea cada
  vez que se abre) — a propósito NO reutiliza `sortState[ent]` (el estado global de orden de esa tabla en
  su pestaña normal) para que ordenar dentro del picker nunca altere cómo se ve esa pestaña después.
- **Columnas configurables por llamada**: `cfg.columns=[{key,label,width}]` — el `grid-template-columns`
  de `#recordPickerHead`/cada `.record-picker-row` se fija inline vía JS a partir de `width` de cada
  columna (a diferencia del `.park-picker-head` original de v3.42.0, hardcodeado a 5 columnas fijas en
  CSS) — así el mismo modal sirve para 5 columnas (Parque) o 6 (Propiedad) sin tocar CSS.
- Cierre: clic en el fondo (`e.target.id==="recordPickerOverlay"`) o Escape — este último con el mismo
  chequeo de "capa por encima de la ficha" que ya usa `#photoEditOverlay` (Escape cierra el picker primero;
  un segundo Escape cierra la ficha).
- **Botón al lado del campo, no en su propia fila (v3.42.1)** — pedido explícito del usuario tras ver la
  ficha real. `.field-recordpicker` (antes `.field-parkpicker`, renombrada al generalizar) perdió
  `grid-column:1/-1;margin-top:-8px` (lo que lo mandaba a una fila propia debajo); ahora es una celda normal
  de 1 columna, así que cae justo a la derecha del input readonly que lo precede, por simple orden de
  aparición en el HTML. Se le agregó un `<label>&nbsp;</label>` vacío para que el botón quede a la altura
  del input, no de su label.
- **2 ajustes más, pedidos explícitos del usuario (v3.43.1)**:
  1. **`cfg.baseFilter`** — nuevo parámetro opcional de `openRecordPicker(cfg)`: filtro FIJO de la llamada
     (a diferencia de la búsqueda, no depende de lo que el usuario escriba), aplicado en
     `renderRecordPickerList()` ANTES del filtro de búsqueda, para que el conteo/resultado ya refleje el
     universo reducido. `openPropertyPickerForTransaction()` lo usa para excluir propiedades con
     `MARKET_STATUS==="Occupied"` (valor exacto de la lista `MARKET_STATUS`, ver `## Listas condicionales`)
     — no tiene sentido relacionar una transacción nueva con una propiedad ya ocupada.
     `openParkPicker()` no pasa `baseFilter` (sigue mostrando todos los Parques).
  2. **Cierre redundante con Escape directo en el buscador** — el listener global de `document` (ver
     "Cierre" más abajo) ya cerraba el picker con Escape desde v3.42.0, pero se agregó ADEMÁS un
     `onkeydown="if(event.key==='Escape') closeRecordPicker();"` directo en `#recordPickerSearch` (mismo
     patrón de `onkeydown` inline que ya usa `#listsNewValue` para Enter) — refuerzo defensivo pedido por
     el usuario tras reportar que Escape no cerraba el picker.
     - **Bug real introducido por este mismo refuerzo, reportado de inmediato por el usuario (v3.43.2):
       Escape cerraba el picker Y la ficha completa a la vez**, en vez de solo el picker. Causa: el handler
       local en `#recordPickerSearch` cerraba el picker (`closeRecordPicker()`) pero el evento seguía
       burbujeando hacia `document` sin detenerse — para cuando el listener GLOBAL corría (después del
       local, mismo evento), `#recordPickerOverlay` YA no tenía la clase `show` (el handler local ya la
       había quitado), así que el chequeo `if(...classList.contains("show"))` del global daba falso y caía
       al `else` (`closePolygonMap();closeModal();...`), cerrando también la ficha. Fix real: agregar
       `event.stopPropagation()` en el handler local (`onkeydown="if(event.key==='Escape'){
       closeRecordPicker(); event.stopPropagation(); }"`) — el evento nunca llega a `document`, así que el
       listener global ni se entera de que hubo un Escape, y solo se cierra 1 capa por pulsación (mismo
       criterio de "la capa de arriba se cierra primero, un 2do Escape cierra la siguiente" que ya aplica en
       el resto de la app). **Lección para cualquier `onkeydown` local futuro que conviva con el listener
       global de Escape de esta app**: si el handler local actúa y el evento seguiría corriendo por
       `document`, hay que `stopPropagation()` — de otra forma el listener global corre igual y puede
       ejecutar SU propia rama (a veces incorrecta, porque el estado que consulta ya cambió).

- **Título del nombre de propiedad igualado al de "Building Options" (v3.43.3, tamaño; v3.43.4, color)** —
  el usuario pidió confirmar si `.layout-title` (nombre de la propiedad en el header de cada página de
  Layout) usaba el mismo tamaño/color que el título "Building Options" de la tabla comparativa
  (`.deck-header h2`). Ninguno de los 2 coincidía: `.layout-title` estaba en `26px`/`#3f4550`,
  `.deck-header h2` en `32px`/`#666` — ambos ya comparten la misma fuente (`'Dala Moa',Georgia,serif`).
  v3.43.3 igualó el tamaño (`26px→32px`) a pedido explícito; el usuario preguntó por tamaño primero y no se
  tocó el color en ese momento. v3.43.4, pedido explícito de inmediato después ("que sea el mismo color de
  tipografía de la tabla"): igualó también el color (`#3f4550→#666`) — ahora `.layout-title` es idéntico a
  `.deck-header h2` en tamaño y color, solo cambia el `margin`/contexto de layout (no aplica aquí).

## Configuración (menú y submódulos)

El botón **Configuración** del sidebar ya no navega directo — abre un **mini menú flotante** al lado
(`toggleConfigMenu`, mismo patrón que el menú de 3 puntos de las filas de la tabla) con 3 opciones
(`configSection`: `"files" | "fields" | "lists"`). Elegir una opción muestra **exclusivamente** esa sección
ocupando todo el ancho/alto disponible (`renderConfigSections`):

1. **Archivos Excel / Imágenes** (`#configFilesSection`) — vincular Excel y carpeta de imágenes, en dos
   tarjetas lado a lado. Cada chip tiene además una **"nota de ruta"** manual (`pathNotes`, localStorage
   `citius_path_notes_v1`, funciones `renderPathNote`/`editPathNote`): la File System Access API **no
   expone la ruta real del sistema** a la página por seguridad (solo `.name`, nunca la ruta completa) —
   esto es una limitación del navegador, no del código, así que la "ruta" que se ve ahí es un texto que el
   usuario escribe a mano (vía `prompt()`) solo como recordatorio, no se detecta ni se valida sola.
2. **Campos de ficha** (`#fieldConfigPanel`) — el editor de `fieldConfig` descrito arriba (pestañas por
   entidad, encabezado de columnas fijo, tarjetas de grupo arrastrables, filas de campo arrastrables).
   - **Etiqueta visible editable por el usuario** (v3.23.0): botón lápiz junto al nombre del campo en cada
     fila (`editFieldLabel(gi,fi)`, prompt — mismo mecanismo que `editFieldHelp`). Guarda en
     `customFieldLabels` (`citius_custom_field_labels_v1` en localStorage), y `fieldLabel(key)` pasó de
     `FIELD_LABEL_OVERRIDES[key] || key.replace(...)` a `customFieldLabels[key] || FIELD_LABEL_OVERRIDES[key]
     || key.replace(/_/g," ")` — la personalización del usuario le gana incluso a los 3 overrides ya
     hardcodeados por mí (`PROPERTY_TPYE`/`ADRESS`/`TRANSACCION_TYPE`). **GLOBAL por key, no por entidad**
     (decisión confirmada explícitamente con el usuario) — si el mismo key existe en 2 entidades (ej.
     `ADRESS` en Propiedades y Parques), la etiqueta personalizada aplica a ambas, igual que ya hacía
     `FIELD_LABEL_OVERRIDES` desde antes; no se tocaron los ~14 call-sites de `fieldLabel()` para hacerlo
     por-entidad. Dejar vacío el prompt vuelve al default.
   - **Cuidado de grid al agregar botones a `.fc-field-row`**: esa fila usa `display:grid` con
     `grid-template-columns` FIJO (8 columnas desde v3.23.0: `26px 1.6fr 30px 100px 120px 1fr 1fr 30px`) —
     agregar un elemento nuevo al `template literal` de `fcFieldRowHtml` sin agregar también su columna
     correspondiente aquí (y el `<span>` vacío que hace juego en `.fc-col-header`, que comparte el mismo
     template) empuja todo lo que sigue una posición, y el último elemento (el botón de ayuda) se desborda a
     una fila nueva por debajo en vez de quedar alineado — bug real cometido y corregido en la misma v3.23.0
     al agregar el botón de etiqueta. Si se agrega OTRO botón a esta fila en el futuro, sumar su columna en
     AMBOS selectores a la vez.
3. **Listas** (`#configListsSection`) — administrador de `lists`, migrado de un modal a página completa
   (columna de nombres de listas + editor de valores). **"+ Nueva lista"** (`addNewList()`, v3.10.0) crea
   una lista nueva vacía (antes solo se podían editar los valores de listas ya existentes definidas por la
   app) — pide el nombre con `prompt()`, lo normaliza a MAYÚSCULAS_CON_GUION_BAJO (mismo estilo que el
   resto), rechaza duplicados (comparación insensible a mayúsculas). Cero código nuevo necesario aguas
   abajo: el nav (`renderListsNav()`), el selector "Lista" de cada campo en Configuración > Campos, y
   `listOptionsHtml()` (el `<select>` real en la ficha) ya eran 100% genéricos sobre `Object.keys(lists)`/
   `lists[nombre]` — agregar una key nueva a `lists` basta para que aparezca disponible en los 3 lugares.
   - **Reordenar / renombrar / borrar una lista (v3.23.0)** — ninguna de las 3 existía antes de esta
     versión (solo se podía agregar/editar/borrar VALORES dentro de una lista, o crear una lista nueva
     vacía):
     - **Reordenar valores**: drag-and-drop (`listValueDragStart`/`listValueDragOver`/`listValueDrop`,
       v3.23.1 — reemplazó las flechas ↑/↓ de la v3.23.0 original, `moveListValue` ya no existe) por fila de
       valor, mismo patrón exacto que `fcFieldDragStart`/`Over`/`Drop` de Configuración > Campos (variable
       módulo con el índice arrastrado, `.dragover` visual, ajuste `-1` del índice destino cuando el origen
       es anterior). Reordena solo los valores DENTRO de la lista seleccionada; el orden de las listas en el
       nav sigue siendo alfabético fijo (`Object.keys(lists).sort(...)`, sin cambios), reordenar listas
       entre sí no se pidió.
     - **Renombrar** (`renameCurrentList()`, botón lápiz junto a `#listsCurrentName`): además de mover la
       key en `lists` (`lists[nuevo]=lists[viejo]; delete lists[viejo]`), **barre `fieldConfig` de las 4
       entidades y reescribe cualquier `f.list` que apuntara al nombre viejo** — antes de v3.23.0 no existía
       NINGÚN barrido así en toda la app (confirmado por investigación: un rename de lista dejaba campos con
       una referencia huérfana silenciosa — `listOptionsHtml()` regresa `lists[listName]||[]`, así que un
       nombre inexistente simplemente da un `<select>` vacío sin ningún error visible). Si se agrega otro
       lugar en el futuro que guarde un nombre de lista por string (fuera de `fieldConfig[ent][i].list`),
       hay que agregarlo también a este barrido.
     - **Borrar** (`deleteCurrentList()`, botón bote de basura junto a `#listsCurrentName`): **BLOQUEADA
       mientras algún campo (cualquier entidad) todavía use esa lista** — decisión explícita del usuario
       (se prefirió esto sobre "avisar y convertir esos campos a texto libre automáticamente"). Muestra un
       `alert()` con la lista exacta de campos afectados (`"Entidad: Campo"` por línea, usa `fieldLabel(f.key)`
       así que ya refleja cualquier etiqueta personalizada del punto anterior); el usuario debe ir primero a
       Configuración > Campos a quitarle/cambiarle la lista a esos campos. Sin ningún campo usándola, pide
       confirmación (`confirm()`, menciona cuántos valores se perderían) y borra.

No existe ya un botón "Listas" aparte en el sidebar ni el modal `#listsOverlay` — todo vive dentro de este
menú de Configuración.

### Traducción ES de las listas (v4.3.0, pedido explícito del usuario)

Motivo: las presentaciones se generarán en inglés Y en español, pero el usuario fue explícito: **"en el
sistema todos los datos siempre se llenarán y se mostrarán en inglés, solo las traducciones se usarán para
las presentaciones"** — `lists`/`DEFAULT_LISTS` (captura de datos) NO se tocaron para nada; se agregó un
mapa **nuevo y paralelo**, `listTranslationsEs`/`DEFAULT_LIST_TRANSLATIONS_ES`, shape
`{LISTNAME: {"Valor en inglés": "Traducción"}}` — mismo patrón exacto de `localStorage`/merge-de-claves-
faltantes que `lists`/`DEFAULT_LISTS` (`LIST_TRANSLATIONS_KEY = "citius_list_translations_es_v1"`,
`loadListTranslationsEs()`/`saveListTranslationsEs()`), pero en un `localStorage` y una variable
completamente separados — nunca se mezclan.

- **Semilla inicial**: el usuario compartió `LISTAS_CAMPOS.xlsx` (su archivo de referencia, hoja
  "(LD) Lista Desp", columnas `TABLA`/`EN`) y pidió llenarle una 3ra columna `ES` con la traducción de sus
  ~740 valores (99 listas) — se hizo con un script de Python/openpyxl, tradujo todo lo que es terminología
  real (`AVAILABILITY_DELIVERY.Immediate` → "Inmediata", `CONDITION."Grey Shell"` → "Obra Gris", reusando
  el vocabulario ya visto en la referencia de la Presentación de Oficinas donde aplicaba) y **reflejó sin
  traducir** (mismo valor en ES) los valores que son nombres propios/códigos: `BROKERS_CAG` (nombres de
  personas), `ADMIN`/`DEVELOPER`/`OWNER`/`CLASSIFICATION` (A/B/C/D), `ESTATE` (estados de México, ya en
  español), unidades/monedas (`CM_IN`/`FT_M`/`SF_M2`/`CURRENCY`/`CON_COMERCIAL`/etc.), y nombres de marca
  (`NATURAL_GAS_LAND`/`TELECOMMUNICATIONS_LAND`). `ORIGIN_COUNTRY` (193 países) sí se tradujo completo a su
  nombre en español. `DEFAULT_LIST_TRANSLATIONS_ES` (JS) se generó 1:1 desde esa misma columna ES ya
  llenada — una sola fuente de verdad entre el Excel y el código.
- **`loadListsFromSheet()`** ahora también lee una 3ra columna `ES` opcional de la hoja "(LD) Lista Desp"
  vinculada. A diferencia de `lists` (que se REEMPLAZA completo con lo que traiga la hoja, ver arriba), las
  traducciones se **mezclan valor por valor**: si la celda ES de una fila viene vacía, NO borra una
  traducción que ya existiera en la app (editada a mano) — el Excel solo gana para los valores que sí trae
  traducidos. `buildConfigWorkbook()` escribe esa 3ra columna de vuelta al guardar ("Guardar listas" con un
  Archivo de Configuración vinculado).
- **UI, `Configuración > Listas`** (`renderListValues()`): cada fila de valor ahora tiene **2 inputs** en
  vez de 1 — el de siempre (inglés, sigue editando `lists[currentListName][i]`, usado para captura) y uno
  nuevo con tinte azul (`.lists-value-es`, `editListValueTranslation(i, valor)`, guarda en
  `listTranslationsEs[currentListName][valorEnIngles]`) — encabezado de columnas nuevo (`#listsValuesHeader`,
  "English"/"Spanish (presentations only)") explica cuál es cuál. Renombrar un valor en inglés
  (`editListValue`) migra su traducción existente a la nueva key en vez de perderla; borrar un valor
  (`deleteListValue`) borra también su traducción; renombrar/borrar una LISTA completa
  (`renameCurrentList`/`deleteCurrentList`) migra/borra `listTranslationsEs[nombre]` igual que ya hacía con
  `lists[nombre]`.
- **Todavía no se usa en ningún deck** — esto es solo la infraestructura de datos + su UI de mantenimiento;
  conectar `listTranslationsEs` a la generación real de una presentación en español queda pendiente para
  cuando esa función se construya.

## Resumen (dashboard de indicadores)

Primera pestaña del menú lateral (`#resumenView`/`#resumenContent`, mostrada por `showResumenView()`/
`renderResumenView()`). Recrea, calculando en vivo desde `state.PROPIEDADES`/`state.TRANSACCIONES`, los
indicadores de mercado inmobiliario que un Excel de referencia del usuario traía como fórmulas en su
pestaña "Portada" (absorción, disponibilidad, inventario, tasas, desglosados por submercado).

- **Submercados dinámicos**: `resumenSubmarkets()` (unión de `fieldValuesInUse` sobre `SUBMARKET` en
  `PROPIEDADES` y `TRANSACCIONES`, filtrada por `resumenMarket` si hay uno seleccionado) — no hay una lista
  fija de submercados; las tablas por submercado se transponen respecto al Excel original (submercado como
  **fila**, no como columna) justamente porque la cardinalidad es variable.
- **Selector de Mercado** (`resumenMarket`, variable en memoria, no persistida, junto al de Año): filtra TODO
  el dashboard a un solo mercado. `resumenMarkets()` (opciones del selector) es **siempre**
  `fieldValuesInUse("PROPIEDADES","MARKET")` — deliberadamente NO usa `cascadeCatalog` aquí, aunque esté
  cargado con muchos más mercados de los que el usuario ya capturó — el selector debe reflejar solo lo que
  ya existe en la tabla de Propiedades, y crecer solo conforme se agregue información de un mercado nuevo
  (decisión explícita del usuario, ver v2.3.1 en la memoria de versionado — la primera versión sí usaba el
  catálogo completo cuando estaba cargado, y era el comportamiento incorrecto).
  `resumenMarketMatches(row,ent)` resuelve el mercado de una fila — directo vía `row.MARKET` en
  PROPIEDADES/PARKS, o vía `resumenMarketForSubmarket(row.SUBMARKET)` en TRANSACCIONES (que no tiene campo
  `MARKET` propio); esa función SÍ usa el catálogo como fuente autoritativa si está cargado (propósito
  distinto al del selector: resolver a qué mercado pertenece un submercado dado, no listar mercados
  disponibles). Como las 12 tablas ya iteran sobre `resumenSubmarkets()` para armar sus filas, filtrar esa
  función basta para propagar el filtro de mercado a todas ellas sin tocarlas una por una — solo
  `resumenKpis()` necesitó su propio filtrado explícito (agrega sobre
  `state.PROPIEDADES`/`state.TRANSACCIONES` directamente, no por submercado).
- **Selector de año** (`resumenYear`, variable en memoria, no persistida — se resetea al año actual en cada
  carga, igual que `current`/`mapViewActive`): solo filtra las métricas de **actividad transaccional** (KPI
  "Transacciones de Renta", Absorción Bruta/Neta trimestral, panel "Transacciones Completadas"). Las
  métricas de **inventario/estado actual** (los primeros 8 KPIs, el panel "Inventario Disponible") no se
  filtran por año — reflejan el estado presente de los edificios, igual que en las fórmulas del Excel
  original (que tampoco usan `TRANSACTION_DATE` para esas).
- **Fix deliberado sobre el Excel original**: en "Inventario Disponible > Por Estado", la fila "Sublease"
  usa el campo `TRANSACTION_AVAILABLE="Sublease"` en vez de `STATUS="Sublease"` — el Excel original filtraba
  por `STATUS` ahí, pero la lista controlada real de `STATUS` (`Ready/Under Construction/Under
  Development/Planned`, ver `DEFAULT_LISTS`) no incluye "Sublease", así que esa fila del Excel siempre daba
  0 (el KPI principal de Sublease del mismo Excel sí usa `TRANSACTION_AVAILABLE` correctamente — el Resumen
  replica ese campo consistentemente en ambos lugares).
- **Omitido a propósito**: dos datos del Excel que están escritos a mano (no son fórmulas) — un rango de
  precio de terreno, y una lista de 3 transacciones destacadas. No hay forma de derivarlos de las tablas de
  datos existentes; se dejaron fuera de esta primera versión.
- Todas las comparaciones de texto usan `ciEq(a,b)` (insensible a mayúsculas — replica el comportamiento
  nativo de `SUMIFS`/`COUNTIFS` en Excel, que también es insensible a mayúsculas).
- **KPI "Precio de Renta Promedio (SF/Mes)"** (`avgAsking`, dentro de `resumenKpis`, línea ~2186) — promedia
  `ASKING_RATE_VALUE` de Propiedades, pero solo sobre las filas que cumplen LAS 3 a la vez:
  `ASKING_RATE_UNIT==="USD/SF"` (excluye `MXN/m2`/`MXN/SF`/`USD/m2` — no tiene sentido promediar rentas en
  distinta moneda/unidad de área juntas), `MARKET_STATUS==="Available"` (mismo criterio que el resto de los
  KPIs del Resumen, que solo cuentan inventario disponible), y `Number(ASKING_RATE_VALUE)>0` (v3.22.3 —
  antes solo excluía `null`/`""`; un `0` explícito SÍ entraba al promedio como si fuera una renta real de
  $0/SF, arrastrándolo hacia abajo — se cambió a un solo chequeo `>0` que de paso ya cubre `null`/`""`/
  negativos/no-numéricos, sin necesitar el chequeo viejo por separado). El promedio se divide entre la
  CANTIDAD de filas que sí califican (`askingRows.length`), no entre el total de Propiedades — así que
  registros sin dato, en otra unidad, o con estatus distinto a Available simplemente no cuentan ni en la
  suma ni en el divisor, en vez de aportar un 0 que sesgaría el promedio hacia abajo.
- Builders genéricos reutilizados en los 4 bloques: `resumenQuarterTableHtml()` (fila=submercado, columnas
  1T-4T + acumulado año) y `resumenBreakdownTableHtml()` (fila=submercado, columnas=categorías fijas —
  rangos de tamaño, tipo de edificio, tipo de operación, clase, estado, generación). Ambos agregan una fila
  "Total" al final. `RESUMEN_SIZE_BUCKETS`/`inSizeBucket()` son los 5 rangos de tamaño (0-50k, 50-100k,
  100-150k, 150-250k, +250k SF) compartidos entre "Transacciones Completadas" e "Inventario Disponible".
- La tabla del dashboard es una tabla ligera propia (`table.resumen-table`), **no** reutiliza `#tbl` (la
  tabla principal de Propiedades/Parques/Transacciones) a propósito, para no heredar el comportamiento de
  "clic en la fila abre la ficha".
- **Sección "Cumplimiento" (v3.6.0)**, al final del dashboard — originalmente 2 tablas (Propiedades y
  Parques, el usuario eligió explícitamente cubrir ambas entidades ya que `MARKET_STATUS` existe en las 2).
  A diferencia de TODAS las demás tablas del Resumen (que suman un valor agregando filas de `state`), esta
  **promedia** un porcentaje que ya se calculó por registro — reutiliza `completionStats(ent,rec)` (la
  misma función que ya alimenta la columna "% Completado" de la tabla principal), no una fórmula nueva.
  `resumenComplianceTableHtml(title,subtitle,ent)`: filas = `resumenMarkets()` (respeta el filtro de
  Mercado del Resumen si hay uno activo — igual que el resto del dashboard), columnas =
  `lists.MARKET_STATUS` (la lista real configurada, no un array fijo — se adapta sola si se edita desde
  Configuración > Listas). Si la entidad no tiene ningún campo `required` configurado
  (`completionStats(ent,{})` da `null`), muestra un aviso en vez de una tabla vacía. Celdas sin ningún
  registro en esa combinación mercado×estatus muestran "—", no "0%" (son significados distintos: sin datos
  vs. datos capturados incompletos). **Sin columna "Total"** (quitada en v3.7.2 a pedido del usuario, no se
  usaba) — solo tiene la fila "Total" al fondo (promedio por mercado a través de todos los `MARKET_STATUS`).
  - **Parques pasó a "full" sin quiebre por `MARKET_STATUS` (v3.40.0)** — pedido explícito del usuario: "que
    se calcule el % de cumplimiento sobre el full de los datos, no por MARKET_STATUS". Solo se cambió
    Parques; **Propiedades se quedó exactamente igual** (sigue usando `resumenComplianceTableHtml`, con su
    quiebre por `MARKET_STATUS`) — el pedido fue específico de Parques. Nueva función
    `resumenComplianceFullTableHtml(title,subtitle,ent)`: misma filosofía de promedio por registro, pero
    UNA sola columna "% Completado" (sin desglose adicional), filas = `resumenMarkets()` igual que la
    versión con quiebre, + fila "Total" al fondo. También se agregaron 2 tablas nuevas con esta misma
    función "full": **Transacciones** y **Terrenos** (pedido explícito, "una última más para los
    Terrenos"). Las 4 tablas de Cumplimiento (Propiedades con quiebre, Parques/Transacciones/Terrenos sin
    quiebre) se muestran juntas en `resumenRowsOf3([...])` (queda en 3+1, sin código especial por cantidad).
  - **`resumenMarketOfRow(ent,row)`** — nuevo helper que resuelve a qué Mercado pertenece una fila de
    cualquier entidad, para las tablas "full": Propiedades/Parques/Terrenos ya traen `MARKET` directo; para
    Transacciones (que no tiene ese campo, solo `SUBMARKET`) se resuelve vía
    `resumenMarketForSubmarket(row.SUBMARKET)` — mismo criterio exacto que ya usaba `resumenMarketMatches()`
    para el filtro de Mercado del selector superior, generalizado aquí para las tablas de cumplimiento.
  - **Terrenos vive en su propio archivo Excel independiente** (`landFileHandle`, ver `## Modelo de datos`),
    pero su tabla de Cumplimiento consulta `state.LAND` igual que cualquier otra — no hizo falta ningún
    ajuste especial, `completionStats("LAND",rec)` y `resumenMarketOfRow("LAND",row)` funcionan sobre
    `state.LAND` exactamente igual que sobre las demás entidades.
- Verificado contra los datos de ejemplo (`BUNDLE`) comparando cada número a mano con una suma manual en
  Python — coincide exactamente.
- El contenedor `#resumenView` necesita el selector `#resumenView.resumenview` (no `.resumenview` solo) para
  su `overflow-y:auto;display:block` — con un selector de una sola clase pierde la cascada contra
  `.wrap{overflow:hidden;display:flex}` (misma especificidad, `.wrap` está definida más abajo en la hoja de
  estilos y gana por orden). Si el contenido del Resumen alguna vez parece "cortado" sin scroll, revisar
  primero la especificidad de este selector antes que la lógica de datos.
- `table.resumen-table` usa `table-layout:fixed` a propósito (columnas de ancho igual repartidas en el ancho
  total) — es una decisión de diseño específica del Resumen, no aplica a `#tbl` (la tabla principal), que
  tiene su propio sistema de anchos personalizables/drag-reorder.
- **Cuidado con reglas globales sin scope**: `th,td{white-space:nowrap}` (línea ~281, pensada para la tabla
  principal `#tbl`) se aplica a CUALQUIER `th`/`td` del documento, incluyendo `.resumen-table`. CSS resuelve
  la cascada por propiedad, no por regla completa — si una regla más específica (ej. `table.resumen-table
  th`) no declara explícitamente `white-space`, esa propiedad sigue viniendo de la regla global aunque la
  regla específica "gane" en las propiedades que sí declara (mayor especificidad no implica "reset" de
  propiedades no mencionadas). Por eso `table.resumen-table th, td` declara `white-space:normal` explícito.
  Antes de asumir que un override más específico debería ganar y no está funcionando, revisar si hay una
  regla global sin scope pisando esa propiedad puntual.
- **`resumenColgroupHtml(dataColCount)`** — genera el `<colgroup>` de cualquier tabla del Resumen (llamado
  desde `resumenQuarterTableHtml`/`resumenBreakdownTableHtml`). La columna "Submercado" recibe 1.5 "unidades"
  de ancho contra 1 unidad para cada columna de datos, convertidas a porcentaje (`<col style="width:X%">`) —
  o sea, siempre 50% más ancha que las demás, y el resto del espacio se reparte proporcionalmente (igual)
  entre las columnas de datos. Aplica igual en tablas de ancho completo y en `.resumen-row-3`, sin caso
  especial por contexto.
- El encabezado de tabla (`table.resumen-table thead th`) usa `font-size:1.05em` (relativo al `font-size`
  heredado de esa tabla en particular) en vez de un valor fijo — así escala junto con `.resumen-row-3 table
  {font-size:9.5px}` en vez de quedar desproporcionadamente grande ahí y forzar wraps a 2 líneas en palabras
  cortas.
- **`resumenRowsOf3(blocks)`** — agrupa un array de bloques de tabla HTML de a 3 por fila, envolviendo cada
  grupo en su propio `.resumen-row-3`; el sobrante (1 o 2) queda en la última fila. Las 3 secciones del
  Resumen (Absorción, Transacciones Completadas, Inventario Disponible) usan este mismo helper.
  `.resumen-row-3` usa `grid-template-columns:repeat(3,1fr)` (3 columnas de grid SIEMPRE fijas, a propósito
  — no `auto-fit`) para que una fila con 1 o 2 tablas mantenga el mismo ancho de columna que la fila de
  arriba y deje espacio vacío visible, en vez de estirarse a ocupar todo el ancho disponible.
- La sección "Absorción por Submercado" tiene 3 tablas lado a lado (`.resumen-row-3`): Bruta, Negativa y
  Neta — igual que el bloque "Absorption" (GROSS/NEGATIVE/NET) de la pestaña Portada del Excel de
  referencia. Con `table-layout:fixed` y solo ~380-480px disponibles por tabla en un layout de 3 columnas,
  las 6 columnas (Submercado + 1T-4T + Acum. Año) se comprimían por debajo de un ancho legible y el texto se
  veía encimado — por eso esas tablas tienen `min-width:560px` explícito y dependen del scroll horizontal de
  su propio `.resumen-scroll` para las últimas columnas en pantallas medianas. Si se agrega otra tabla de
  varias columnas dentro de `.resumen-row-3` (o un layout de N-en-fila similar), aplicar el mismo patrón
  (`min-width` + scroll propio) en vez de dejar que `table-layout:fixed` comprima libremente.

## Presentación (deck)

Genera una presentación imprimible a partir de los registros marcados con el círculo amarillo en la tabla
de Propiedades. Incluye páginas de Portada y Cierre (toggle on/off, imágenes embebidas en base64 desde la
carpeta `Logo`), un Mapa interactivo con pines numerados (CARTO Voyager como basemap desde v3.7.0 — antes
Esri Gray Canvas, cambiado para usar el mismo estilo "Calles claro" ya disponible en la pestaña Mapa, con
lógica de "decluttering" para pines superpuestos), y las tablas de datos por propiedad con las unidades de medida
alineadas. Tiene zoom/centrado en la vista previa en pantalla, preservando el tamaño real al imprimir.

- **Fix real: el mapa aéreo de Layout se veía ENCIMA de la ficha y de "Datos de portada" (v3.43.6)** — el
  usuario mandó capturas mostrando el mapa aéreo de una página de Layout flotando por encima de la ficha de
  edición (abierta desde el listado lateral de la presentación) y también por encima del popover "Datos de
  portada". Causa: Leaflet fija z-index ALTOS de forma inline en sus panes internas (~200 tiles, ~600
  marcadores, ~700 popups). `.layout-aerial` tenía `position:relative` pero SIN `z-index` — sin un
  `z-index` explícito, un elemento posicionado NO crea su propio "stacking context", así que esas panes
  internas de Leaflet escapaban del contenedor y competían por z-index DIRECTO contra el resto de la
  página: `.overlay{z-index:100}` (la ficha) y `.columns-panel`/`.cover-edit-panel{z-index:300}` (Datos de
  portada) — como 600-700 > 100 y > 300, Leaflet ganaba y se pintaba encima. Fix: `z-index:0` explícito en
  `.layout-photo,.layout-aerial` — mismo criterio que ya tenía `.map-page-canvas` (la página "Mapa" del
  deck, agregado en una ronda anterior, nunca tuvo este bug reportado). Con `z-index:0`, `.layout-aerial` SÍ
  crea su propio stacking context: todo lo de adentro (sin importar qué z-index interno use Leaflet) queda
  contenido como una sola capa en el lugar que le toca en el documento, nunca por encima de un modal/popover
  que esté más arriba en la página. **Lección para cualquier mapa Leaflet nuevo que se agregue a futuro
  dentro de la app**: su contenedor SIEMPRE necesita `position:relative;z-index:<algo>` explícito (no basta
  con `position:relative` solo) — si no, sus panes internas pueden terminar por encima de cualquier
  modal/popover de la página, sin importar qué tan bajo se vea "a simple vista" el mapa en el HTML.
- **Portada/Cierre re-embebidas en mayor resolución (v3.43.5)** — el usuario reportó que la Portada se veía
  "fea"/pixelada al exportar a PDF. Diagnóstico (sesión anterior, sin cambios de código en ese momento): el
  base64 de `COVER_IMAGE_SRC` NO había perdido calidad (esa conversión es sin pérdida) — el problema real
  era que el PNG fuente (`Logo/Portada Banco.png`) medía apenas 1652×1269px, insuficiente para imprimir a
  las ~300dpi típicas de un PDF/impresora una vez que Chrome estira la página al tamaño físico calculado
  por `preparePrintPageSize()`. El usuario consiguió versiones en mayor resolución de `Portada Banco.png` y
  `Cierre.png` (ambas ahora 6250×4800px, ~3.8× más anchas) y las reemplazó en la carpeta `Logo`. Fix: se
  re-generaron `COVER_IMAGE_SRC`/`CLOSING_IMAGE_SRC` (líneas ~4895-4896) a partir de esos 2 archivos nuevos
  — verificado con hash MD5 que el base64 embebido calza byte-por-byte con el PNG fuente (mismo criterio de
  integridad que la primera vez que se armaron estas constantes). `DECK_LOGO_SRC` y el resto de imágenes de
  `Logo` (`EmblemaAmarillo.png`, `Logotipo.png`, `Portada Amarillo.png`) **no se tocaron** — el usuario solo
  actualizó Portada y Cierre; si algún día se quiere mayor resolución en esas otras, es el mismo
  procedimiento (reemplazar el PNG en `Logo/`, re-embebido en base64, verificar MD5). Apareció además 2
  archivos nuevos en la carpeta (`Fondo-01.png`, `Fondo-04.png`, también 6250×4800) que **no están
  referenciados en ningún lado del código** — quedaron sin usar, presumiblemente insumos del diseño que no
  hacía falta embeber.
  - **2da actualización de `Portada Banco.png` (v3.43.7)** — el usuario volvió a reemplazar el archivo en
    `Logo/` (mismo tamaño 6250×4800, contenido/diseño distinto — hash MD5 cambió). Mismo procedimiento:
    re-generado `COVER_IMAGE_SRC` a partir del PNG actualizado, verificado byte-por-byte con MD5.
    `CLOSING_IMAGE_SRC`/`Cierre.png` no cambiaron esta vez.
- **Nueva opción "Layout": 1 página de spec sheet por propiedad (v3.41.0)** — quinto checkbox en
  `#presPageToggles` (Portada/Mapa/Tabla/**Layout**/Cierre), a diferencia de "Tabla" (hasta 6 propiedades
  comparadas lado a lado en 1 sola página) genera **1 página POR CADA** propiedad seleccionada, insertada
  en el deck entre Tabla y Cierre. Cada página (`buildLayoutPageHtml`, clase `.layout-page`) tiene 4
  bloques:
  - **Header**: logo + título (`PROPERTY_NAME`, itálica) + subtítulo (`DEVELOPER`, subrayado) a la
    izquierda; el número de la propiedad dentro del orden de la presentación (`idx+1`, mismo número que su
    pin en el mapa aéreo) en grande arriba a la derecha (`.layout-num`).
  - **Foto Fachada** (`loadDeckPhotoInto`, kind `FACHADA` — mismo helper y mismo archivo que ya usa "Tabla",
    no se duplicó nada).
  - **Tabla de specs en 2 columnas** (`PRESENTATION_LAYOUT_ROWS`): reusa `PRESENTATION_ROWS`/
    `presValueFor`/`presCellHtml` tal cual (misma fuente de verdad que "Tabla", hereda el selector de
    Unidades/Renta del topbar sin trabajo adicional), quitando Building Name/Industrial Park/Developer/
    Submarket/Location Pin (esos 4 ya se muestran en el header de la página o en el mapa aéreo) y
    repartiendo el resto mitad/mitad en el mismo orden de `PRESENTATION_ROWS` (`layoutRowColumns()`). El
    título de la tabla en sí usa `PARK_NAME` (Industrial Park | Location). **Supuesto sin confirmar,
    marcado explícito al usuario**: el agrupamiento mitad/mitad es simple, no replica el agrupamiento por
    tema (comercial vs. specs físicos) de la captura de referencia que compartió el usuario — si se pide
    ese orden exacto, reordenar `PRESENTATION_LAYOUT_ROWS` es el único cambio necesario.
  - **Mapa aéreo en vivo** (`renderPresLayoutMaps`, 1 instancia de Leaflet POR propiedad, a diferencia de
    `presMapInstance` que es una sola instancia compartida): basemap satelital Esri World Imagery (mismas
    tiles que la pestaña Mapa en modo "Satelital"), dibuja el polígono propio de la propiedad si existe
    (`polygonKeyFor("PROPIEDADES")` → campo `GEOJSON_POLIGONO`, `L.geoJSON` con el color de acento
    `#f0b323`) y un pin numerado clickeable a Google Maps en su `LATITUDE`/`LONGITUDE` (mismo patrón que
    `.map-pin` del Mapa de pines). Decisión confirmada con el usuario vía AskUserQuestion: el polígono/pin
    se dibujan en vivo, **no** vienen ya dibujados en la "Foto Aérea" (kind `AEREA`) que se sube por
    separado — esa foto no se usa en este bloque. Sin coordenadas válidas, muestra un placeholder de texto
    en vez de mapa.
  - **Foto Layout** grande abajo (`loadDeckPhotoInto`, kind `LAYOUT`, mismo archivo que ya sube el usuario
    en la ficha — guardado fit-width/fondo blanco, se muestra con `object-fit` natural, sin recortar).
  - **Print**: grupo de página propio (`layoutPage`) en `preparePrintPageSize()`, separado de `tablePage` a
    propósito — el alto de Layout es muy distinto al de la tabla comparativa de "Tabla", así que
    compartir grupo habría estirado la más chica de las 2 con margen en blanco de sobra (mismo criterio que
    ya separaba `coverPage`/`mapPage` del resto).
  - **Sin verificar aún contra una impresión/export real** (mismo caveat que otros ajustes visuales de esta
    sección) — pendiente que el usuario confirme con un PDF real antes de encadenar más ajustes finos de
    tamaño/espaciado, siguiendo su preferencia habitual con temas de impresión.
  - **6 ajustes visuales/de datos, todos pedidos explícitos del usuario tras ver la 1ra versión (v3.41.1)**:
    1. `.layout-title` (Building Name) — se quitó `font-style:italic`, queda en serif normal.
    2. `.layout-subtitle` (Developer, la línea bajo el título) — mismo gris que el título (`#3f4550`, antes
       `#d99a0e` naranja) y sin `text-decoration:underline`.
    3. `.layout-header` — se quitó el `border-bottom` gris que separaba el header del resto de la página.
    4. `.layout-top-row` — Foto Fachada y mapa aéreo ahora miden lo mismo de ancho
       (`grid-template-columns:1fr 2.2fr 1fr`, antes `1fr 2.3fr 1.7fr` con el aéreo más angosto que la
       fachada).
    5. **Fix real de datos**: el polígono del mapa aéreo mostraba el polígono PROPIO de la propiedad
       (`polygonKeyFor("PROPIEDADES")`, si existía) — el usuario aclaró que debe ser el polígono del
       **Parque industrial relacionado**, no uno propio de la propiedad. Fix: `relatedParkFor(rec)` busca
       en `state.PARKS` el registro cuyo `ID_PARK` calce con `rec.ID_PARK` (comparado con `String()` de
       ambos lados, por si un lado llega numérico y el otro texto) y se lee su `GEOJSON_POLIGONO`
       (`polygonKeyFor("PARKS")`) en vez del de la propiedad. El pin numerado sigue en el
       `LATITUDE`/`LONGITUDE` de la propiedad misma (eso no cambió, ver `.map-pin`). **Nota**: esto asume
       que `PROPIEDADES.ID_PARK` es una relación real 1:1 hacia `PARKS.ID_PARK` (mismo campo, mismo tipo de
       ID) — si una propiedad tiene `ID_PARK` capturado pero ningún Parque con ese mismo ID existe en el
       Excel, simplemente no se dibuja polígono (queda solo el pin, sin error visible).
    6. `.layout-plan` (Foto Layout/plano grande de abajo) — el usuario reportó que "se ve todo grande y
       salta de hoja" al imprimir. Causa: `width:100%;height:auto` deja que el alto real de la página
       dependa del aspecto de la foto subida — una Foto Layout con aspecto más alto de lo normal estiraba
       la página entera más allá de una hoja. Fix: `aspect-ratio:1050/430` fijo + `object-fit:contain` (en
       vez de `cover`, para no recortar el plano) — el bloque del plano ahora SIEMPRE mide lo mismo sin
       importar el aspecto real de la foto, con franjas blancas arriba/abajo si no calza exacto en vez de
       estirar la página. **Sin verificar aún contra una impresión/export real** — el valor `1050/430` es
       una estimación a ojo comparando contra la captura de referencia que compartió el usuario, puede
       necesitar otro ajuste fino si no calza del todo.
  - **6 ajustes más, todos pedidos explícitos tras ver la 2da captura de referencia (v3.41.2)**:
    1. **Zoom del mapa aéreo, más alejado** (`renderPresLayoutMaps`) — `fitBounds` maxZoom 18→15 y padding
       [20,20]→[40,40]; el fallback sin polígono (`setView`) 17→14. El usuario reportó "tiene mucho zoom".
    2. **Submarket agregado a la tabla, antes de "Leasable Area (GLA)"** — se quitó `"SUBMARKET"` de
       `PRESENTATION_LAYOUT_EXCLUDED_KEYS` (antes se excluía junto con Building Name/Parque/Developer/
       Location Pin). No hizo falta reordenar nada: en `PRESENTATION_ROWS`, Submarket ya está justo antes
       de Leasable Area (solo Location Pin queda entre los 2 y sigue excluido), así el filtro lo deja
       exactamente ahí. Con este campo de vuelta, la tabla queda en **22 filas exactas → 11 por columna**
       (`layoutRowColumns()`, `Math.ceil(22/2)`), el número que pidió el usuario en el punto 5 de esta
       misma tanda de cambios.
    3. **Formato de fila uniforme, excepto Asking Rate** — `layoutColumnHtml()` ya NO aplica
       `highlight`/`subtotal` (antes "Minimum Divisible" se pintaba de acento amarillo); todas las filas
       alternan gris/blanco por zebra-stripe salvo Asking Rate (`row.dark`), que "se queda como está"
       (fondo gris oscuro) tal cual pidió. Fix real de contraste: se agregó
       `.layout-row.deck-dark,.layout-row.deck-dark .layout-row-label,.layout-row.deck-dark
       .layout-row-unit,.layout-row.deck-dark .layout-row-value{color:#fff}` — antes solo el contenedor
       `.layout-row` tenía `color:#fff`, pero `.layout-row-unit`/`.layout-row-value` traían su propio color
       fijo (gris/negro) que le ganaba en cascada, dejando esa fila casi ilegible sobre el fondo oscuro.
    4. **Pin del mapa aéreo, clic en cualquier parte lo abre (no solo el número)** — antes el número iba
       dentro de un `<a href>` (mismo patrón que `presMap`/`landPresMap`, documentado arriba en
       `### Pin numerado del mapa de Presentación`), pero su área clickeable real era solo el glifo del
       número (~11px, rotado), fácil de fallar en un mapa tan chico. Ahora es un `<span>` sin link +
       `marker.on("click", ()=>window.open(mapsUrl,"_blank","noopener"))` — Leaflet dispara ese evento con
       un clic en cualquier punto del ícono completo (26×26px). CSS: se agregó `.map-pin span` (mismo
       estilo visual que `.map-pin a` de siempre) — los pines de `presMap`/`landPresMap` NO se tocaron,
       siguen usando `<a>` tal cual, solo Layout usa este mecanismo nuevo.
    5. **22 valores en la tabla (11+11)** — ver punto 2 arriba, quedó resuelto solo con agregar Submarket
       de vuelta, sin ningún cambio adicional de `layoutRowColumns()`.
    6. **Unidad en columna dedicada, separada del atributo** (`.layout-row-unit`) — antes era un `<span>`
       pegado inline al final del texto del label (`"Leasable Area (GLA) (SF)"` corrido); ahora es un
       hermano aparte con ancho fijo (`width:34px`) y su propio `padding-left`, mismo criterio que
       `.deck-row-label`/`.deck-row-unit` de la tabla comparativa (2 columnas de grid separadas, no un
       span inline). `layoutColumnHtml()` ahora emite 3 `<div>` hermanos (label/unit/value) en vez de anidar
       la unidad dentro del label.

  - **2 ajustes más tras ver una captura real de la tabla (v3.41.3)**:
    1. **Zoom un poco más cerca** — el usuario pidió "más alejado" en v3.41.2 y resultó demasiado; ajuste
       intermedio (no un regreso completo a los valores originales): `fitBounds` maxZoom 15→16, padding
       [40,40]→[30,30]; fallback sin polígono (`setView`) 14→15.
    2. **Fix real: la columna de unidad de la v3.41.2 NO quedaba alineada** — el usuario mandó una captura
       mostrándolo. Causa real: cada fila era un `.layout-row` flex con `label:flex:1` + `unit:34px` fijo,
       pero `.layout-row-value` no tenía ancho fijo (solo padding) — su ancho variaba según el texto de esa
       fila ("0" vs "US $ 90.36"), y como el label absorbía TODO lo que quedaba después de restar el value,
       el label terminaba con un ancho DISTINTO en cada fila; con él, el punto de inicio de la columna de
       unidad se recorría fila por fila (su propio ancho sí era fijo, pero su posición de arranque no).
       Fix: `.layout-col` (el contenedor de las ~11 filas de cada columna) ahora es el GRID real
       (`grid-template-columns:1fr auto auto`, mismo criterio que `.deck-grid` de la tabla comparativa) y
       `layoutColumnHtml()` emite label/unit/value como 3 `<div>` HERMANOS directos de ese grid — ya no hay
       un `.layout-row` envolviendo cada fila. Como una columna de grid comparte el mismo ancho en TODAS
       sus filas por definición (a diferencia de un ancho "heredado" vía flex-grow), la unidad ahora arranca
       exactamente en el mismo punto en cada fila, sin importar el largo del label/value de esa fila en
       particular — igual que en la tabla comparativa de referencia que compartió el usuario.

  - **Fix real: "delineado" raro en Asking Rate (v3.41.4)** — el usuario mandó una captura mostrando el
    fondo oscuro de esa fila partido en cajitas sueltas con una franja clara entre cada una, en vez de una
    sola barra continua. 2 causas, las 2 heredadas de la v3.41.3 (`.layout-col` como grid real):
    1. `column-gap:6px` deja un hueco REAL entre columnas que ninguna celda cubre con su propio fondo — en
       una fila normal (blanca) es invisible, pero en Asking Rate (fondo oscuro) ese hueco se ve como la
       franja clara que "corta" el fondo en 3 cajitas.
    2. `align-items:baseline` hacía que el fondo de cada celda cubriera solo el alto de su propio contenido,
       no el alto completo de la fila — como la unidad usa `font-size:9px` (más chico que el 11px del
       label/valor), su cajita de fondo quedaba más baja/chica que las otras 2 de la misma fila.
    Fix: mismo criterio EXACTO que `.deck-grid` de la tabla comparativa (que nunca tuvo este problema) —
    `gap:0` (el espacio entre columnas ahora es puro padding horizontal de cada celda, `padding:2.5px 6px`
    las 3, antes 4px/0px/4px asimétrico) + `align-items:stretch` (el default de grid, ya no forzado a
    `baseline`) para que el fondo de cada celda cubra siempre el alto completo de su fila. Las 3 celdas de
    una fila con color quedan pegadas en una sola barra continua, sin fisuras entre ellas.

  - **Unidad centrada dentro de su columna (v3.41.5)** — el usuario reportó que se veían "muy a la derecha"
    en la primera de las 2 columnas de la tabla. La columna de unidad es `auto` (se ajusta al texto MÁS
    ancho de esa columna en las 11 filas — ahí vive "(year/m²)" de Asking Rate, uno de los más largos),
    así que en filas con unidad corta ("(SF)", "(ft)", "(%)") sobra espacio dentro de esa columna; sin
    `text-align` explícito ese espacio de sobra quedaba todo de un lado. Fix de una línea:
    `.layout-row-unit{text-align:center}`.
  - **Fix real: la unidad seguía pegada al valor, no realmente centrada entre label y valor (v3.41.6)** —
    el fix anterior centraba el TEXTO dentro de la columna de unidad, pero esa columna en sí (`auto`)
    quedaba empujada contra el borde derecho: con `grid-template-columns:1fr auto auto`, el label era la
    ÚNICA columna flexible (`1fr`), así que absorbía TODO el espacio libre de la fila, dejando unit+value
    pegados entre sí contra el borde derecho — de ahí que se viera "muy pegado al valor". Fix: las 2
    columnas de los extremos pasan a `1fr` cada una (`grid-template-columns:1fr auto 1fr`) — con el mismo
    peso, se reparten el espacio libre mitad y mitad, y la columna de unidad (en medio, sigue `auto`) queda
    fija aproximadamente a la mitad del ancho de la fila, sin pegar ni al label ni al value. El value sigue
    alineado a la derecha dentro de SU mitad (`text-align:right` no cambió), así que los números no se
    movieron de la orilla derecha — solo la unidad se corrió hacia la izquierda, hacia el centro.
  - **Ajuste fino: "un poco más a la derecha" que el centro exacto (v3.41.7)** — el usuario pidió esto
    justo después de ver el centrado de v3.41.6. `grid-template-columns:1fr auto 1fr` (mismo peso) →
    `1.3fr auto 0.7fr`: el label se queda con más proporción del espacio libre, recorriendo la columna de
    unidad hacia la derecha del punto medio, sin llegar a pegarse al value (que sigue con `text-align:right`
    dentro de su propia columna, ahora solo más angosta — el número no se movió de la orilla derecha).

  - **3 ajustes de estilo, todos pedidos explícitos del usuario (v3.41.8)**:
    1. **Sin borde en la Foto Layout/plano de abajo** — `.layout-plan` perdió `border:1px solid #e3e6eb`.
    2. **Bold quitado de TODOS los valores por default** — `.layout-row-value` pasó de `font-weight:600` a
       `400`.
    3. **Bold agregado SOLO a Industrial Park Location, Leasable Area (GLA), Available Area, Minimum
       Divisible y Asking Rate** — de esas 5, 2 YA estaban en 700 sin tocar nada: Asking Rate (vía
       `row.dark`/`.deck-dark`, que siempre tuvo `font-weight:700`) e Industrial Park Location (`PARK_NAME`,
       que no es una fila de la lista — es el título de la tabla, `.layout-table-title`, ya bold aparte).
       Solo hizo falta agregar las otras 3: `LAYOUT_BOLD_KEYS` (`LEASABLE_AREA_GLA_VALUE`,
       `AVAILABLE_AREA_VALUE`, `MINIMUM_DIVISIVLE_VALUE`) + clase `.layout-bold` aplicada en
       `layoutColumnHtml()` cuando `row.key` está en ese set.

- **Zoom más preciso con la rueda del mouse en el mapa de pines (v3.39.2)** — el usuario reportó que al
  hacer zoom in/out con la ruedita del mouse sobre el mapa de la Portada/Mapa de la presentación, el salto
  era demasiado grande y pedía algo "intermedio con más precisión" (dejó explícito que NO se refería al
  zoom del navegador ni pedía un control en pantalla — el mapa se sigue moviendo/zoomeando solo con la
  rueda del mouse, sin botones visibles, "lo cual está perfecto"). Causa: `L.map(canvas,{...})` en
  `renderPresMap()`/`renderLandPresMap()` no traía opciones de zoom, así que Leaflet usaba sus defaults
  (`zoomSnap:1`, `zoomDelta:1`, `wheelPxPerZoomLevel:60`), que con un mouse físico (deltaY grande por
  "click" de rueda, a diferencia de un trackpad) casi siempre saltan un nivel de zoom completo por click.
  Fix: se agregaron `zoomSnap:0.25, zoomDelta:0.25, wheelPxPerZoomLevel:200` a las DOS instancias de
  `L.map(canvas,...)` (Propiedades y Terrenos comparten el mismo mecanismo de mapa de presentación, así
  que ambas se ajustaron igual). `zoomSnap` más chico permite que el mapa quede "a la mitad" entre dos
  niveles de zoom entero en vez de forzar solo enteros; `wheelPxPerZoomLevel` más alto exige más scroll
  acumulado por nivel, así que cada click de rueda mueve una fracción más pequeña. Los mapas de la ficha
  de edición (`polygonMapEl`, `pinMapEl`, `allPolygonsMapEl`) NO se tocaron — el usuario fue explícito en
  que hablaba solo de la presentación.

- **Texto editable sobre la Portada (v3.29.0)** — la Portada sigue siendo una sola imagen estática
  (`COVER_IMAGE_SRC`, con el logo y "Market Availability" ya *dentro* de la imagen, no seleccionables),
  pero ahora se le puede superponer Mes/Año, "Building Options" + m², y una lista de Mercados. Los 3
  datos se **escriben a mano** (no se calculan de los registros) en un popover ("Datos de portada",
  botón `#presCoverEditBtn`, mismo patrón visual que el panel "Columnas"), persistidos en `localStorage`
  (`citius_pres_cover_v1`, variable `presCoverInfo`). `buildCoverPageHtml()` dibuja ese texto como HTML
  real (`.cover-overlay`, position:absolute sobre la imagen) — por eso sí queda incluido al exportar/
  imprimir el PDF, a diferencia de la imagen de fondo. Si un campo queda vacío no se muestra esa línea.
  Los mercados se escriben separados por coma y se listan como `<li>`.
  - **Botón "Datos de portada" movido junto a "Imprimir / PDF" (v3.32.0)** — antes vivía dentro de
    `.pres-page-toggles`, junto al checkbox "Portada" (que sigue ahí, sin cambios); ahora es el último
    botón antes de "Imprimir / PDF" en `#presTopbarControls`, pedido explícito del usuario. El botón de
    imprimir también se renombró en esta misma versión: "Imprimir / Descargar PDF" → **"Imprimir / PDF"**
    (`#presPrintBtn`, mismo `onclick`, sin cambios de comportamiento — solo el texto).
  - **2 campos más en el popover, que a propósito NUNCA se dibujan en la portada (v3.32.0)** — el usuario
    los pidió para guardarlos junto con el resto de la configuración de portada, explicó que contaría el
    motivo después:
    - **Empresa / Proyecto** (`presCoverInfo.company`, texto libre) — input simple, mismo patrón que
      Mes/Año.
    - **Brokers** (`presCoverInfo.brokers`, array) — lista de checkboxes (`renderCoverEditBrokersList`/
      `toggleCoverEditBroker`, clase `.cover-edit-checklist` con scroll propio) poblada desde
      `lists["BROKERS_CAG"]` (nombre exacto corregido en v3.32.1 — v3.32.0 usaba `"BROKERS CAG"` con
      espacio, no calzaba con el nombre real de la lista ya administrada por el usuario en Configuración >
      Listas; no hardcodeada en `DEFAULT_LISTS` — si esa lista está vacía o no existe, el panel muestra un
      aviso en vez de checkboxes). Los checkboxes usan `data-broker` + `this.dataset.broker` (no un valor
      embebido directo en el `onclick`) — a propósito, para que un nombre de broker con apóstrofe no rompa
      el string de JS armado a mano (mismo riesgo que ya existe en otros checkboxes de la app con valor
      embebido, ej. `onColFilterToggleValue`, pero aquí sí importa porque nombres de persona/empresa
      reales pueden traer apóstrofes).
    - **Fix real (v3.32.2): los checkboxes se veían apilados (uno arriba, la etiqueta abajo) y en gris**,
      en vez de checkbox+texto en la misma línea con el texto normal. Causa: `#coverEditBrokersList`
      (el `<div class="cover-edit-checklist">`) vivía ANIDADO dentro del mismo `.field` que su
      `<label>Brokers (...)</label>` — como cada checkbox real también es un `<label class="col-filter-item">`,
      terminaba siendo descendiente de ese `.field`, y la regla `.field label{display:block;color:var(--muted);
      font-size:11px;...}` (especificidad 0,1,1) le ganaba en cascada a `.col-filter-item{display:flex;...}`
      (especificidad 0,1,0) — mayor especificidad gana sin importar cuál regla se declaró después. Fix: el
      `<div class="cover-edit-checklist">` se sacó del `.field` (ahora es un `<div>` hermano, no hijo, del
      `.field` que solo contiene el `<label>` de la sección) — sus checkboxes ya no son descendientes de
      ningún `.field`, así que `.col-filter-item` aplica sin competencia. Si se agrega otro checklist de
      checkboxes dentro de un panel que use la convención `.field label{...}` en el futuro, mismo cuidado:
      nunca anidar un checklist de valores (con sus propios `<label>` por fila) dentro de un `.field`.
    - Ninguno de los dos dispara `renderPresentationView()` al cambiar (a diferencia de Mes/Año/m²/
      Mercados, que sí) — no tendría efecto visual, así que se omite el re-render completo del deck.
    - `loadPresCoverInfo()` hace merge contra un objeto default (`{monthYear,m2,markets,brokers:[],
      company:""}`) en vez de regresar el JSON parseado tal cual — así alguien que ya tenía guardado el
      `presCoverInfo` viejo (solo 3 campos, de antes de v3.32.0) no truena al leer `.brokers`/`.company`
      (quedan en su default vacío en vez de `undefined`).
    - **El "para qué" prometido: nombre de archivo del PDF auto-generado (v3.33.0)** — Empresa/Proyecto y
      Brokers no se dibujan en la portada, pero sí alimentan el nombre sugerido del PDF exportado.
      `buildPresPdfFilename()`: `[Empresa/Proyecto] [Tamaño] - [Mercado] (Iniciales-de-broker)` — ej.
      `"dasdfsd 10,000 m2 - Monterrey (AG-AKDC)"` con 2 brokers marcados. `brokerInitials(name)` toma la
      primera letra de cada palabra del nombre (`"Adrian Garza"`→`"AG"`); con 2+ brokers seleccionados, sus
      iniciales se unen con `-` y el conjunto va entre paréntesis (`(AG-AKDC)`, no `AG-AKDC` suelto —
      ajustado a pedido explícito del usuario). Cada mitad (izquierda: empresa+tamaño; derecha:
      mercado+iniciales) omite las partes vacías sin dejar espacios/guiones huérfanos; si absolutamente
      todo está vacío, cae a `"Presentacion"`. `sanitizeFilenamePart()` reemplaza `\/:*?"<>|` (inválidos en
      Windows/macOS) por `-` y colapsa espacios de sobra — el resto de caracteres (comas, guiones, acentos,
      paréntesis) se dejan tal cual.
      - **Mecanismo**: Chrome usa `document.title` como nombre sugerido en su diálogo "Guardar como PDF" —
        `presPrintBtn.onclick` cambia `document.title` justo antes de `window.print()` y lo restaura justo
        después (`window.print()` es síncrono/bloqueante en Chrome mientras el diálogo del sistema está
        abierto, así que el cambio de título nunca llega a verse en la pestaña ni afecta nada más). Mismo
        patrón de "cambiar estado → imprimir → restaurar estado", ya usado para `pagesEl.style.zoom` en el
        mismo handler.
      - Verificado con una simulación en Python de la lógica exacta (sin Chrome disponible esta sesión):
        casos con 2 brokers, 1 broker, campos vacíos combinados, y una empresa con "/" en el nombre
        (sanitizado a "-") — todos dieron el resultado esperado.
  - **Posición y tamaño de letra del overlay ajustados (v3.32.0)**, comparando contra una captura real de
    la portada: `left:12.5%→9.5%` y `top:40%→42%` (el bloque ahora empieza casi a la altura de la "M" de
    "Market Availability" — más a la izquierda y un poco más abajo que antes), y cada tamaño de fuente del
    overlay +2px (fecha 14→16, título "Building Options" 17→19, valor m² 15→17, lista de mercados 14→16).
    Sigue siendo una estimación a ojo (mismo caveat que v3.29.0) — **no verificado aún contra una
    impresión/export real de esta ronda**, puede necesitar otro ajuste fino si no calza del todo.
- **El nombre en el listado lateral ("Propiedades en la presentación") es clickeable** (v3.6.1,
  `editFromPresentation(id)`, en `renderPresSelectedList`) — abre la ficha en modo edición sin salir de la
  vista de Presentación (el modal es un overlay independiente de la vista activa, a diferencia de
  `editFromMap` no navega a la tabla primero). Fija `current="PROPIEDADES"` por si se venía de otra
  entidad. `saveBtn.onclick` llama `renderPresentationView()` si `presentationViewActive` es verdadero, así
  el listado y el deck reflejan el fix de inmediato al guardar, sin recargar ni navegar manualmente.
  - **Fix real (v3.38.0): el mismo refresco instantáneo faltaba para Terrenos** — `editFromLandPresentation`
    (equivalente a `editFromPresentation` para Terrenos, ver `## Presentación de Terrenos` más abajo) sí
    fijaba `current="LAND"` y abría la ficha, pero `saveBtn.onclick` solo revisaba `presentationViewActive`,
    nunca `landPresentationViewActive` — el usuario reportó que editar un Terreno desde su presentación y
    guardar NO reflejaba el cambio hasta salir y volver a entrar a esa vista (a diferencia de Propiedades,
    donde sí era instantáneo). Fix de una línea: `if(landPresentationViewActive)
    renderLandPresentationView();` agregado justo al lado de la línea de Propiedades. Mismo cuidado a
    futuro que ya aplica aquí: cualquier bandera de vista nueva que dependa de refrescarse al guardar
    necesita su propio chequeo explícito en este mismo punto, no se generaliza solo.
  - **Reordenar por drag-and-drop (v3.23.1)** — `presItemDragStart`/`presItemDragOver`/`presItemDrop`
    reemplazaron las flechas ↑/↓ y `movePresItem` (borrada, ya no existe) a pedido del usuario, mismo
    patrón que el drag-and-drop de Configuración > Campos. Como `list[i]` (el parámetro de
    `renderPresSelectedList`) y `presentationOrder[i]` están siempre en el mismo orden/índice
    (`getOrderedPresentationList()` ya sincroniza `presentationOrder=valid` antes de regresar la lista),
    el drop hace un `splice` directo sobre `presentationOrder` por índice — sin necesitar mapear IDs.
    Llama `renderPresentationView()` completo al soltar (no solo re-renderiza el sidebar), igual que ya
    hacía `movePresItem`, para que el deck paginado y el mapa numerado reflejen el nuevo orden de inmediato.
  - **El orden se sincroniza con el sort de la tabla de Propiedades (v3.30.0)** — pedido explícito del
    usuario ("que el orden de la presentación sea el mismo que el de Propiedades, con los sorts que se
    hicieron"). `syncPresentationOrderToTableSort()` reordena `presentationOrder` para que calce con
    `applySort(selectedRecs,"PROPIEDADES")` (ver `## Modelo de datos` arriba, orden multi-columna) — sin
    ningún sort activo es un no-op (`applySort` regresa las filas en el mismo orden en que ya venían). Se
    llama en 2 tipos de momento: al **entrar** a Presentación (`showPresentationView`), y cada vez que
    cambia la **selección** (`togglePresentationSelection`/`toggleSelectAllPresentation`) o el **sort de
    Propiedades** (`toggleSort` y los 6 mutadores del panel "Ordenar por" — `addSortLevel`/
    `setSortLevelKey`/`setSortLevelDir`/`removeSortLevel`/`clearSortLevels`/`sortLevelDrop`/
    `clearAllFilters`, todos con guard `if(current==="PROPIEDADES")`). **Es solo el PUNTO DE PARTIDA, no
    reemplaza el drag-and-drop manual de arriba** (decisión confirmada con el usuario vía
    AskUserQuestion, entre "siempre igual a la tabla, sin arrastre" y esta opción): después de
    sincronizar, el usuario sigue pudiendo arrastrar para ajustar el orden a mano — pero la PRÓXIMA vez que
    cambie la selección o el sort de Propiedades, se vuelve a resincronizar y ese ajuste manual se pierde
    (comportamiento esperado, no un bug — si se pide lo contrario en el futuro, es una decisión distinta a
    confirmar con el usuario, no asumir).
- **`PRESENTATION_ROWS`** (array que define las filas de la tabla comparativa, `{label, key, unit?,
  unitClass?, rule?, prefix?, highlight?, dark?, subtotal?, spacer?, ...}`) y **`presValueFor(rec,row)`**
  (formatea el valor de cada celda) son el corazón del deck — totalmente aparte de `fmt()`/
  `pairValueLabel()`/`unitKeyFor()` de la tabla principal, lee `rec[row.key]` directo. `row.rule` controla
  placeholders: `"suit_tbc"` (0→"To Suit", vacío→"TBC") y `"upon_request"` (0→"Upon Request").
  - **2 keys sintéticas** combinan más de un campo real en una sola fila (ninguna lee `rec.__ALGO__`, que
    no existe — `presValueFor` las intercepta por key antes de leer `rec[row.key]`): `"__BAY_SIZE__"`
    combina `BAY_WIDTH_VALUE`/`BAY_DEPTH_VALUE` en `"W x D"` (respeta el selector Métrico/Imperial vía
    `convertCellForView`); `"__LOCATION_PIN__"` (v3.19.0) combina `LATITUDE`/`LONGITUDE` en texto plano
    `"lat, long"` — si falta cualquiera de las 2, muestra "—" completo (un pin no tiene sentido con una
    sola coordenada), sin conversión de unidades (son coordenadas, no dimensiones físicas).
  - **`presCellHtml(rec,row)`** (v3.20.0) — el bucle que arma cada celda de valor del deck pasa por esta
    función en vez de hacer `escapeHtml(presValueFor(...))` directo. Por default regresa exactamente lo
    mismo; para `"__LOCATION_PIN__"` con coordenadas válidas, envuelve el texto en
    `<a href="https://www.google.com/maps?q=lat,lng" target="_blank" rel="noopener">` — abre el pin en
    Google Maps, tanto en la vista previa en pantalla como **dentro del PDF final**: Chrome preserva los
    `<a href>` como hipervínculos reales de PDF al "Imprimir > Guardar como PDF" (el mismo mecanismo que ya
    usa el botón "Imprimir / Descargar PDF"), sin trabajo adicional para que sobreviva la exportación. CSS
    `.deck-cell a{color:inherit;text-decoration:underline}` — el link hereda el color de texto de la fila
    (nunca "azul de link" desentonando si esa fila combinara con `highlight`/`dark`/`subtotal`), solo se ve
    el subrayado como pista de que es clickeable. Si se necesita otro link de este tipo en el futuro
    (otra fila con datos que apunten a una URL externa), extender `presCellHtml` con otro caso por key,
    mismo patrón.
  - **`subtotal:true`** (v3.19.0, clase `.deck-subtotal`: `background:#e2e2e2` gris claro + texto negro en
    negrita) — mismo nivel de prioridad que `highlight`/`dark` en la cadena `if/else if` que decide la
    clase de cada fila (`highlight` > `dark` > `subtotal` > zebra-stripe) — una 3ra opción de "fila de
    sección", distinta del ámbar de `highlight` y del gris oscuro/blanco de `dark`. Usada hoy en "Leasable
    Area (GLA)" y "Available Area". **"Asking Rate" usó `subtotal` brevemente en v3.19.0 pero volvió a
    `dark:true` en v3.19.3** (pedido explícito del usuario, quería su estilo gris oscuro/texto blanco
    original de vuelta) — así que `dark` SÍ sigue en uso real, no es un mecanismo huérfano. Si se agrega un
    4to estilo de "fila especial" en el futuro, seguir el mismo patrón: nuevo flag booleano + su propia
    clase CSS + una entrada más en esa cadena `if/else if` (en los 3 lugares que la repiten: label, unidad,
    y cada celda de valor).
  - **Tamaño de letra**: `.deck-row-label` y `.deck-cell` comparten `font-size:12px` explícito (heredaban
    originalmente los 12px de `.deck-grid`; bajó a 10px en v3.19.3 para ganar espacio vertical, subió a
    11px en v3.22.1 y de nuevo a 12px en v3.22.2 — 2 pedidos seguidos de "un poco más grande" en la misma
    sesión — terminando, coincidentemente, en el mismo valor que el original heredado de `.deck-grid`, solo
    que ahora explícito en vez de heredado). Esto nunca arriesga que el disclaimer se pase a una 2da hoja:
    `preparePrintPageSize()`, ver `## Presentación (deck)` más abajo, calcula el tamaño de `@page` a partir
    de la altura REAL renderizada de `.deck-page`, así que el alto de la página impresa siempre se ajusta
    al contenido, nunca es un tamaño fijo tipo Carta/A4 que el texto pueda desbordar. Existe un mecanismo de
    excepción por-fila para un tamaño distinto al general: la fila "Location Pin" (`__LOCATION_PIN__`) tiene
    `smallValue:true`, que agrega la clase `deck-cell-sm` (especificidad de 2 clases — gana sobre
    `.deck-cell` sin importar el orden en la hoja de estilos) SOLO a su celda de valor — es aditivo, no
    forma parte de la cadena `highlight/dark/subtotal/stripe`, mismo patrón que `bold`/`nameRow`.
    `deck-cell-sm` se dejó fijo en `font-size:10px` desde que se creó (nunca cambió) — desde v3.22.1 ya NO
    coincide con el tamaño general, y la diferencia (hoy 12px vs 10px) se hace más notoria cada vez que el
    tamaño general vuelve a subir: exactamente el escenario para el que se había dejado listo este mecanismo
    desde v3.19.3.
  - **Texto del disclaimer** (`.deck-note`, párrafo fijo al pie del deck, v3.21.0): se quitaron 2 frases a
    pedido del usuario — `"CAM" Common Maintenance Area` y `"TBD" to be determined` — el resto del texto
    (incluyendo `"TBC" to be confirmed`, que sí se queda) no se tocó. Es texto plano hardcodeado en
    `buildDeckPageHtml`, no hay lista/config detrás; si se pide quitar o agregar otra frase a futuro, es
    edición directa de ese mismo string. `.deck-row-unit` (el "(SF)"/"(ft)") sigue en 9.5px, sin cambios
    en ninguna de estas iteraciones.
    - **`margin-top` reducido de 44px a 20px (v3.39.1)**, pedido del usuario ("el margen de abajo se ve
      grande") — es el espacio entre la última fila de la tabla y este párrafo de aviso legal, al pie de
      cada hoja de tabla. **Esta regla es COMPARTIDA por Propiedades y Terrenos** (`buildDeckPageHtml` y
      `buildLandDeckPageHtml` usan la misma clase `.deck-note` para su disclaimer) — el ajuste aplica a los
      2 decks por igual, no se pudo aislar a uno solo sin duplicar la clase. Si el usuario en realidad se
      refería a otro espacio (ej. el padding de impresión, o algo específico de solo uno de los 2 decks),
      es una corrección aparte — se eligió este primero por ser el candidato más directo a "el margen de
      abajo" de una hoja de tabla.
- **Selector Métrico/Imperial (v3.3.0)** — `#presUnitSelect` en `#presTopbarControls`, estado `presUnitPref`
  (`"imperial"` por defecto, persistido en `citius_pres_unit_pref_v1`). **Independiente de `unitViewPref`**
  (el de la tabla principal) — su propio select, su propia clave de localStorage, y solo 2 opciones (sin
  "Como se registró": el deck siempre debe salir en UN sistema consistente entre todas las propiedades
  incluidas, nunca mezclado). Pedido explícito del usuario, que compartió un Office Script de referencia
  ("Tabla comparativa Métrico/Imperial") — se replicó su clasificación de unidades (AREA/LONG/ESP/RATE) y
  criterio de redondeo, pero reutilizando `UNIT_CONVERSIONS`/`convertRateUnit`/`convertCellForView` (ver
  `## Vista de unidades en tabla` abajo) en vez de duplicar factores de conversión.
  - `convertCellForView(...,targetView)` ganó un 6to parámetro opcional (default `unitViewPref` si se
    omite) — los 3 call-sites de la tabla principal no cambiaron (siguen con 5 argumentos); solo la
    Presentación pasa `presUnitPref` como 6to argumento.
  - Filas con `unitClass` ("AREA"|"LONG"|"ESP"|"RATE" — solo en las que sí son dimensión física
    convertible: áreas, Bay Size/Clear Height, Floor Thickness, Asking Rate) resuelven su campo `_UNIT`
    real vía `unitKeyFor` y convierten el valor TAL COMO fue capturado ese registro — dos propiedades en el
    mismo deck, una en SF y otra en m², salen ambas normalizadas al sistema elegido. El resto de las filas
    (Skylights %, Ventilation cph, Substation kVA, Water Source, texto) no tiene `unitClass` y no se
    toca, igual que antes.
  - Redondeo por clase (además del `round2` que ya hace `convertCellForView`): AREA/ESP se redondean a
    entero (`Math.round`, números whole como en el script de referencia); LONG/RATE quedan en 2 decimales.
  - El header de unidad de cada fila (`presUnitLabel(unitClass)`) ahora muestra el sistema ELEGIDO, no un
    string fijo — correcto porque tras convertir, todas las celdas de esa fila comparten el mismo sistema.
  - **Bug preexistente corregido de paso**: el prefijo de Asking Price era el string fijo `"US $ "` sin
    importar la moneda real de `ASKING_RATE_UNIT` (podía ser `MXN/SF`/`MXN/m2`) — `presCurrencyPrefix()`
    ahora deriva "MX $ "/"US $ " de la unidad real, post-conversión.
  - `onchange` del select llama `renderPresentationView()` de inmediato — necesario porque `presPrintBtn`
    solo hace `window.print()` sobre lo que ya esté en el DOM, no vuelve a renderizar al momento de imprimir.
- **`preparePrintPageSize()`** (llamada desde `presPrintBtn.onclick`, justo antes de `window.print()`) —
  el deck no imprime a un tamaño de papel fijo (Carta/A4): mide con `getBoundingClientRect()` el tamaño real
  de las `.deck-page` actuales y escribe un `@page{...; margin:0}` dinámico (`<style
  id="dynamicPrintPageSize">`) para que el papel mida EXACTO lo que el contenido necesita.
  - **Un solo `@page` sin nombre si solo hay 1 tipo de hoja activo; un `@page` con nombre por tipo si hay 2+
    (v3.27.5)** — Portada/Cierre (`aspect-ratio` fijo), Mapa (`aspect-ratio` fijo) y Tabla (alto según
    contenido) casi nunca miden lo mismo de alto. Con 2+ tipos convivientes (el caso normal: las 4 hojas,
    ~85% del uso real), se calcula `@page coverPage{...}`/`@page mapPage{...}`/`@page tablePage{...}` por
    separado (cada `.deck-page` usa `page:<nombre>` para indicar cuál le toca) — así Portada/Mapa ya no
    quedan forzados a la altura de la Tabla (antes, con presentaciones grandes — probado con 15 propiedades
    — el `@page` compartido se volvía casi cuadrado y dejaba una franja de sobra en Portada/Mapa). Con SOLO
    1 tipo de hoja activo (ej. solo Mapa), se usa un `@page` simple sin nombre — usar páginas con nombre
    para este caso agrega una hoja en blanco de más (confirmado con pruebas sistemáticas cubriendo cada tipo
    solo); sin nombre, es tan simple como antes y no tiene ese problema.
  - **`lockSize()` fija ancho/alto en px directo (por JS) sobre las hojas de Portada y Mapa** (no las de
    Tabla) — no basta confiar en que el `aspect-ratio` de CSS las recalcule igual al momento real de
    imprimir: Mapa y Cierre (que "regresan" a un tipo de página ya usado antes, después de pasar por Tabla)
    miden lo mismo que Portada al medir, pero imprimen más chicas, dejando margen de sobra. **Excepción**: la
    PRIMERA y la ÚLTIMA `.deck-page` del documento completo (`pages[0]` / `pages[pages.length-1]`) se
    excluyen de este fijado — fijarle tamaño a cualquiera de las dos agrega una hoja en blanco de más (al
    inicio o al final según cuál sea). Se probaron otras formas de evitar esto sin excluirlas (fijar solo
    ancho, agregar un elemento vacío después de la última para que técnicamente no sea "la última") y
    ninguna funcionó — quedó la exclusión simple.
  - **`#presPages` NO usa `position:absolute` (antes sí)** — con `position:absolute`, el contenedor del deck
    queda fuera del flujo normal del documento, y `page-break-after`/`page-break-inside` (lo que separa cada
    `.deck-page` en su propia hoja) **solo funcionan dentro del flujo normal** — en un elemento fuera de
    flujo, Chrome pinta el contenido como una capa continua y la corta en rebanadas de píxeles exactas cada
    `altura_de_página`, ignorando dónde empieza/termina cada `.deck-page` (causaba que Portada y Mapa
    aparecieran empalmadas dentro de la misma hoja física). Confirmado además que `position:absolute` en el
    contenedor **también** rompe cualquier `@page` con nombre, no solo los cortes de página.
  - **Clase `body.printing-deck`** (agregada/quitada por `presPrintBtn.onclick`, no solo dentro de
    `@media print`) — con `#presPages` de vuelta en flujo normal, lo que antes quedaba "invisible pero
    ocupando espacio real" (`header.topbar`, `.pres-sidebar`) empezaría a desplazar el contenido de forma
    rara, así que esta clase los oculta con `display:none` (no solo `visibility:hidden`), resetea
    `.main{margin-left:0}` y pone `#presPages{position:static}`. Vive en una clase JS y no solo en
    `@media print` porque `preparePrintPageSize()` mide con `getBoundingClientRect()` en pantalla normal,
    ANTES de que el navegador entre en modo impresión real — si estas reglas solo existieran en
    `@media print`, la medición vería el layout viejo y calcularía un tamaño de página más chico que el
    real. `presPrintBtn.onclick` hace: `classList.add("printing-deck")` → mide → `window.print()` →
    `classList.remove("printing-deck")` en la siguiente línea, de forma síncrona (no con un listener
    `afterprint`) — probado en Chrome real, funciona bien así.
  - **`body.printing-deck .pres-main{padding:10px 1px 1px 13px}`** (arriba/derecha/abajo/izquierda) — este
    padding es lo que le da espacio alrededor a Portada/Mapa/Cierre dentro de la hoja impresa. El total
    horizontal (14px: 13+1) NO debe bajar de ~12px sin volver a probar — por debajo de eso, el ancho
    disponible supera `max-width:1180px` de `.deck-page` (que se quita en impresión real pero no durante la
    medición en pantalla), y ese mismatch entre medición e impresión trae de vuelta hojas en blanco. Los 4
    valores concretos (10/1/1/13, no repartidos parejo) son el resultado de varias rondas de ajuste manual
    pedidas directamente por el usuario ("sube el de la izquierda", "baja el de abajo", etc.) hasta que se
    vio aceptable a su ojo — no hay una lógica de diseño detrás de esos números específicos, solo no bajar
    el total horizontal del umbral de seguridad si se vuelven a tocar.
  - **`.deck-page:last-child{page-break-after:auto}`** — `.deck-page{page-break-after:always}` se aplicaba
    también a la ÚLTIMA hoja del documento, forzando un salto de página aunque no hubiera nada después —
    dejaba una hoja en blanco extra al final. Esta regla lo evita para el caso más común (termina en
    Cierre, mismo tipo que Portada, la primera hoja).
  - **Pendiente, NO resuelto**: 1) si el documento tiene 2+ tipos de hoja pero el ÚLTIMO tipo no coincide
    con el PRIMERO (ej. Portada+Mapa+Tabla, sin Cierre — termina en Tabla en vez de volver a Portada),
    aparece una hoja en blanco al final que no se logró eliminar sin arriesgar romper el caso principal. 2)
    Portada sola (sin ningún otro tipo) mostró en una prueba real un margen chico y una hoja en blanco al
    final — las pruebas automatizadas (CDP `Page.printToPDF`) mostraron algo distinto (contenido duplicado,
    no una hoja en blanco), lo que sugiere que CDP no está replicando fielmente el comportamiento real del
    navegador para este caso específico; no se confirmó la causa real. **El caso de uso principal (~85%,
    las 4 hojas completas) está confirmado funcionando por el usuario** — estos son pendientes de
    combinaciones parciales específicas. **Confirmado en todas las pruebas de esta ronda**: el contenido de
    las hojas de Tabla (los datos) nunca ha fallado en ningún escenario — todos los problemas encontrados
    son cosméticos, en las hojas de Portada/Mapa/Cierre.
- **Selector Mensual/Anual del Asking Rate (v3.23.0)** — `#presPeriodSelect` en `#presTopbarControls`
  (junto al de Unidades), estado `presPeriodPref` (`"monthly"` por defecto, persistido en
  `citius_pres_period_pref_v1`). Mismo patrón exacto que el selector Métrico/Imperial (su propio `<select>`,
  su propia key de localStorage, `onchange` llama `renderPresentationView()` de inmediato). **No existía
  ningún campo de periodicidad ligado a Asking Rate antes de esto** — la lista `PERIOD` en `DEFAULT_LISTS`
  (`["Month to Month","Month","Year"]`) estaba huérfana, sin ningún campo real de ninguna entidad
  usándola; este selector es funcionalidad nueva, no un rewiring de algo ya conectado.
  - En `presValueFor`, dentro del branch `row.unitClass==="RATE"`, si `presPeriodPref==="annual"` se
    multiplica el valor YA convertido por Métrico/Imperial por 12 (`round2(v*12)`) — el orden importa: la
    conversión de moneda/área corre primero (`convertCellForView`), el ×12 después, así ambos ajustes
    componen bien sin importar en qué orden los toque el usuario.
  - En `presUnitLabel("RATE")`, el literal `"month/"` pasó a `(presPeriodPref==="annual"?"year/":"month/")`
    — el `unit:"month/SF"` estático que trae el row de `PRESENTATION_ROWS` nunca se llegó a mostrar (queda
    muerto desde que la fila tiene `unitClass`, que le gana en la cadena `row.unitClass?presUnitLabel(...):
    row.unit` de `buildDeckPageHtml`) — se dejó ese string estático tal cual, inofensivo.
- **Asking Rate forzado a 2 decimales (v3.23.0)** — `fmtNum` (`Number(n).toLocaleString("en-US")`, sin
  opciones) recorta ceros a la derecha por default (`fmtNum(1)` da `"1"`, no `"1.00"`). Para
  `row.unitClass==="RATE"` específicamente (hoy solo "Asking Rate" usa esa clase, confirmado — no hay otra
  fila RATE en `PRESENTATION_ROWS`), `presValueFor` ahora usa
  `v.toLocaleString("en-US",{minimumFractionDigits:2,maximumFractionDigits:2})` en el paso final en vez de
  `fmtNum` — se aplica DESPUÉS del ×12 del punto anterior, así que funciona igual sin importar la
  combinación de Mensual/Anual × Métrico/Imperial elegida.
- **Fix: fila "Water Source" mostraba el dato equivocado (v3.23.0)** — el `key` del row apuntaba a
  `WATER_SUPPLY_CAPACITY_LPS` (la capacidad en lps, probablemente un copy-paste de la fila vecina
  "Installed Capacity | Transformer") en vez del campo categórico real `WATER_SOURCE`
  (Municipal/Well Water/Private, grupo `INFRAESTRUCTURA` en `BUNDLE.PROPIEDADES` — no "Instalaciones
  Generales", que es el grupo de otros campos de agua/drenaje). Corregido a
  `{label:"Water Source", key:"WATER_SOURCE"}`, sin `unit` (igual que su fila vecina "Sewer Source", que sí
  estaba bien desde siempre). El dato de capacidad en lps no se agregó como fila aparte — confirmado con el
  usuario que no hacía falta, solo corregir esta fila.

### Presentación de Terrenos — deck SEPARADO (v3.36.0 navegación; v3.37.0 tabla; v3.38.0 portada/mapa/cierre; v3.39.0 imprimir/PDF)

Pedido explícito del usuario: Terrenos necesita su propia presentación imprimible, con **estructura
distinta** a la de Propiedades (campos totalmente diferentes) — se decidió duplicar el mecanismo en vez de
generalizar uno solo para las 2 entidades (mismo criterio ya establecido en toda la app de "duplicar el
patrón por recurso en vez de refactorizar prematuramente", ej. el Excel de Terrenos vs. el compartido).

- **Navegación — mini-menú "Propiedades" / "Terrenos"** (decisión confirmada con el usuario vía
  AskUserQuestion, entre esto y 2 botones separados en el sidebar): el botón "Presentación" del menú
  lateral (`presTabBtn`) ya NO navega directo — `onclick` ahora es `togglePresentationMenu`, que abre un
  popover flotante (mismo patrón exacto que `toggleConfigMenu` del botón Configuración: clase
  `.row-menu-dropdown.config-menu-dropdown`, ningún CSS nuevo) con 2 opciones en este orden fijo:
  **"Propiedades" primero, "Terrenos" segundo** (pedido explícito del usuario). Elegir una llama
  `selectPresentationMenuOption('propiedades'|'terrenos')`, que cierra el popover y navega a
  `showPresentationView()` o `showLandPresentationView()`.
  - **El popover SIEMPRE se muestra al hacer clic** (no recuerda cuál se usó la última vez) — decisión
    explícita del usuario. Esto es un mecanismo TOTALMENTE APARTE de `saveLastView()`/`loadLastView()` (la
    persistencia entre recargas del navegador, `localStorage citius_last_view_v1`): si la última vista
    activa antes de recargar era la Presentación de Terrenos, recargar SIGUE restaurando directo ahí sin
    pasar por el mini-menú (mismo comportamiento de siempre para cualquier vista) — el mini-menú solo
    interviene en clics en vivo sobre el botón del sidebar, nunca en la restauración de sesión. Nuevo valor
    `"landpresentation"` agregado al dispatch de `loadLastView()` al arrancar.
  - `renderTabs()`: el botón `__PRES__` del sidebar se ve "activo" cuando CUALQUIERA de las 2
    (`presentationViewActive || landPresentationViewActive`) está activa — un solo botón representa ambas
    vistas.
  - **Nueva bandera `landPresentationViewActive`**, sumada al set mutuamente excluyente de siempre
    (`mapViewActive`/`presentationViewActive`/`configViewActive`/`resumenViewActive`) — se agregó su reset
    a `false` en las 5 funciones `show*View()` existentes (omitir esto en alguna sería el mismo bug que ya
    se documentó para otras banderas nuevas en el pasado: 2 vistas mostrándose "activas" a la vez).
- **`showLandPresentationView()`/`renderLandPresentationView()`** — mismo shell HTML que la Presentación de
  Propiedades (`.presentationview` con `.pres-sidebar` + `.pres-main`, contenedores `#landPresentationView`/
  `#landPresSelectedList`/`#landPresPages`, topbar `#landPresTopbarControls` con su propio
  `#landPresPageToggles`/`#landPresCount`/`#landPresUnitSelect`). **Sin el selector de Renta** (Mensual/
  Anual) que sí tiene la de Propiedades (`#presPeriodSelect`) — pedido explícito del usuario, Terrenos no
  maneja un Asking Rate con esa temporalidad. `refreshLandFileIfLinked()` es el único refresh que llama (a
  diferencia de `showPresentationView()`, que refresca los 3 archivos — Terrenos vive en su propio Excel,
  no le interesan los otros 2).
  - **Etapa 1 (navegación)**: como quedó en v3.36.0, sin cambios.

**Etapa 2 (v3.37.0) — selección de registros + tabla comparativa real**, a partir de una hoja de referencia
que el usuario compartió (columna A = etiqueta a mostrar, columna B = unidad, columna C = campo real de la
ficha a mapear, columna D = qué mostrar si el valor viene vacío o en 0). Antes de construir, 4 dudas se
resolvieron con el usuario (AskUserQuestion + una pregunta suelta sobre el campo "Location"):

1. **Layout confirmado: comparación LADO A LADO, como Propiedades** (no "una página completa por
   terreno", que era la lectura literal de la captura compartida — la captura en realidad mostraba solo
   una COLUMNA de esa tabla, indistinguible de "una página" cuando solo hay 1 terreno seleccionado).
2. **La fila "Location" SÍ es un campo real** (no un título de sección, como se preguntó) — la hoja de
   referencia repite el mismo nombre en la columna de etiqueta y en la de campo a mapear. **Supuesto sin
   confirmar 100%, marcado explícitamente al usuario**: se mapeó al campo existente más parecido,
   `INDUSTRIAL_PARK_LOCATION` (no hay ningún campo literalmente llamado `LOCATION` en los 37 campos de
   `BUNDLE.LAND`) — si el usuario quería otro campo (o uno nuevo), es un cambio de una sola línea en
   `PRESENTATION_ROWS_LAND`.
3. **"Land Use" era un campo nuevo** — agregado a `BUNDLE.LAND.fields` (grupo "CARACT. GENERALES", después
   de `FENCE`) con key `LAND_USE`, lista `LAND_USE` vía `FIELD_LIST_MAP.LAND` (lista NO creada en
   `DEFAULT_LISTS` — mismo criterio que `BROKERS_CAG`/`TRANSACTION_TYPE_LAND`: si el usuario ya la tiene en
   Configuración > Listas, funciona directo; si no, el `<select>` sale vacío hasta que la cree).
4. **Solo la Foto Satelital va en el deck** (Panorámica/Layout quedan solo en la ficha, sin aparecer en el
   PDF) — igual que Propiedades solo muestra su foto Fachada en el deck aunque tenga 6 tipos disponibles.

**Selección de registros** (`landPresentationOrder`, localStorage `citius_land_pres_order_v1`) — mecanismo
IDÉNTICO al de Propiedades (`presentationOrder`) pero completamente separado, nunca comparten selección:
`toggleLandPresentationSelection`/`toggleSelectAllLandPresentation`/`clearAllLandPresentationSelection`/
`getOrderedLandPresentationList`, mismos nombres con el prefijo `Land`. `renderTable()` se generalizó para
que el círculo/checkbox de selección (`.th-select`/`.td-select`, chip "N en presentación") aparezca también
en la tabla de Terrenos (`showSelect=current==="PROPIEDADES"||current==="LAND"`), leyendo/escribiendo
`landPresentationOrder` en vez de `presentationOrder` cuando `current==="LAND"` (variables `presOrder`/
`toggleOneFn`/`toggleAllFn`/`clearFn` resueltas según `current` en los 3 puntos que ya usaban
`presentationOrder` a secas). **Sin `syncPresentationOrderToTableSort` equivalente para Terrenos** — ese
"resincronizar con el sort de la tabla" fue un pedido aparte, específico de Propiedades (v3.30.0); si se
pide igual para Terrenos más adelante, replicar ese mecanismo también, no asumido aquí.

**Tabla comparativa** (`PRESENTATION_ROWS_LAND`, `presLandValueFor`, `presLandCellHtml`,
`buildLandDeckPageHtml`, `loadLandDeckPhotoInto`) — mismo patrón exacto que
`PRESENTATION_ROWS`/`presValueFor`/`presCellHtml`/`buildDeckPageHtml` de Propiedades (SLOTS=6,
`.deck-highlight`/`.deck-dark`/`.deck-stripe`/`.deck-name-row`), con los 21 campos de la hoja de referencia.
**Simplificación deliberada**: ningún row usa `unitClass` (conversión Metric/Imperial) — las 3 áreas son
hectáreas (`HAS_M2`, un par que esta app nunca convierte bajo ninguna vista) y el resto son campos
categóricos o en unidades sin conversión aplicable (kv/kVA/lps/month per m2); el selector "Unidades" del
topbar de Terrenos (de la etapa 1) queda sin efecto sobre esta tabla por ahora. 2 reglas de valor vacío/0
nuevas, tomadas literalmente de la columna D de la hoja de referencia: `"viability_none"` (Electric/Water
Capacity: 0→"Viability", vacío→"None") y `"tbc_empty"` (Telecommunications: vacío→"TBC", sin caso especial
para 0 — es categórico). `renderLandPresentationView()` (reemplaza el stub de la etapa 1) sigue el mismo
flujo que `renderPresentationView()`: arma la lista ordenada, pinta el sidebar (`renderPresLandSelectedList`,
con drag-and-drop `landPresItemDragStart/Over/Drop`, mismo patrón que `presItemDragStart/Over/Drop`), y
arma las páginas de tabla en chunks de 6.

- **Fila "Location Pin" (v3.38.1)**, pedido explícito del usuario, justo debajo de "Submarket" (mismo lugar
  que ocupa en `PRESENTATION_ROWS` de Propiedades) — key sintética `"__LOCATION_PIN__"` (no lee
  `rec.__ALGO__`, que no existe; `presLandValueFor`/`presLandCellHtml` la interceptan por key antes de leer
  `rec[row.key]`, mismo patrón exacto que Propiedades). Combina `LATITUDE`/`LONGITUDE` de Terrenos en texto
  plano `"lat, long"` (o "—" si falta cualquiera de las 2); `presLandCellHtml` (nueva, mismo patrón que
  `presCellHtml`) envuelve el texto en un link a Google Maps cuando ambas coordenadas son válidas — Chrome
  preserva `<a href>` como hipervínculo real al exportar a PDF, así que funcionará igual el día que se
  conecte el botón de imprimir de Terrenos. `row.smallValue:true` (mismo flag que usa Propiedades para esta
  fila) agrega la clase `deck-cell-sm` solo a su celda de valor. `buildLandDeckPageHtml` pasó de llamar
  `escapeHtml(presLandValueFor(...))` directo a `presLandCellHtml(...)` para que este link (y cualquier otro
  caso especial futuro) se aplique sin tener que tocar el bucle de nuevo.
- **Prefijo de moneda en "Asking Price" (v3.38.2)**, pedido explícito del usuario ("similar a la que
  tenemos en propiedades con su signo de $"): unidad del row cambiada de `"month/m2"` a `"$/m2"`, y nuevo
  flag `row.currency:true` en `presLandValueFor` — antepone `"MX $ "`/`"US $ "` según la moneda real
  capturada en `ASKING_PRICE_UNIT` (lista `CON_COMERCIAL`: "MXN/m2"/"USD/m2"/etc.), y fuerza 2 decimales
  (`v.toLocaleString(...,{minimumFractionDigits:2,maximumFractionDigits:2})`, igual que "Asking Rate" de
  Propiedades — `fmtNum` por sí solo recorta ceros a la derecha, aquí siempre se quiere "1.00", no "1").
  **Reutiliza tal cual, sin duplicar**, 2 funciones que ya existían para Propiedades por ser agnósticas de
  entidad: `presCurrencyPrefix(unitStr)` (el regex `/^mxn/i` no depende de ninguna entidad) y
  `unitKeyFor(ent,key)` (ya recibía el nombre de la entidad como parámetro — aquí se le pasa `"LAND"` en
  vez de `"PROPIEDADES"`, resuelve `ASKING_PRICE_UNIT` automáticamente por el sufijo estándar `_VALUE`/
  `_UNIT`, sin necesitar tocar `IRREGULAR_VALUE_UNIT_PAIRS`). Si se agrega otro campo con moneda a la tabla
  de Terrenos en el futuro, basta con sumarle `currency:true` en su definición dentro de
  `PRESENTATION_ROWS_LAND` — no hace falta ningún cambio en `presLandValueFor` para ese campo nuevo.

**Guard de cross-contaminación** (agregado desde la etapa 2, sigue vigente): tanto `renderPresentationView()`
como `renderLandPresentationView()` limpian el `#...Pages` del OTRO deck antes de dibujar el propio — como
`preparePrintPageSize()`/las reglas CSS de `@media print` buscan `.deck-page` de forma global (por clase,
no por contenedor), páginas viejas de un deck podrían mezclarse con las del otro si no se limpiara.

**Etapa 3 (v3.38.0) — Portada, Mapa y Cierre reales**, mismo patrón exacto que Propiedades en los 3 casos:

- **Portada** (`landPresCoverInfo`, localStorage `citius_land_pres_cover_v1`, `toggleLandCoverEditPanel`/
  `buildLandCoverPageHtml`) — duplicado completo de `presCoverInfo`/`toggleCoverEditPanel`/
  `buildCoverPageHtml`, con sus propios ids de DOM (`landCoverEditPanel`, `landCoverEditMonthYear/M2/
  Markets/Company`, `landCoverEditBrokersList`) para no chocar con el de Propiedades. Reutiliza la MISMA
  imagen de portada (`COVER_IMAGE_SRC`) y la MISMA lista `BROKERS_CAG` (mismo equipo de brokers para ambos
  decks) — pero el texto capturado (mes/año, tamaño, mercados, empresa/proyecto, brokers marcados) es
  independiente entre los 2 decks, nunca se comparte. `landPresCoverEditBtn` ya no es un stub — se conectó
  con `document.getElementById("landPresCoverEditBtn").onclick=toggleLandCoverEditPanel` al final del
  script (sobreescribe el `onclick="landPresStub()"` que traía inline desde la etapa 1 — se quitó ese
  atributo del HTML para no dejar un rastro confuso de qué función realmente corre).
- **Mapa** (`buildLandMapPageHtml`/`renderLandPresMap`, canvas propio `#landPresMapCanvas`, instancia
  propia `landPresMapInstance`) — duplicado de `buildMapPageHtml`/`renderPresMap`, usando `LATITUDE`/
  `LONGITUDE` de Terrenos (agregados en v3.22.0) y `ID__LAND` en vez de `MAPPING_CODE` para el popup.
  `declutterMapPins()` se **reutiliza tal cual, sin duplicar** — ya era agnóstica de entidad (solo recibe
  un mapa Leaflet + coordenadas de pines, nunca lee campos de ningún registro).
- **Cierre** — **no se duplicó, se reutiliza `buildClosingPageHtml()` literal** (la de Propiedades): esa
  función no tiene NINGÚN contenido específico de una entidad (solo `CLOSING_IMAGE_SRC`), así que no había
  nada que separar.
- `landPresShowState` (las 4 banderas, ya guardadas desde la etapa 2 "por si acaso") ahora SÍ tienen efecto
  real las 4; default cambiado de `{cover:false,map:false,table:true,closing:false}` a
  `{cover:true,map:false,table:true,closing:true}` (igual al default de Propiedades) — solo afecta a quien
  abra la Presentación de Terrenos por primera vez, sin nada guardado todavía.
**Etapa 4 (v3.39.0) — Imprimir / PDF real**, pedido explícito del usuario tras confirmar que Portada/Mapa/
Cierre se veían bien. `landPresStub()` (la función stub que usaban "Datos de portada"/"Imprimir" en la
etapa 1) se **eliminó por completo** — ya no queda ningún botón usándola.

- **`landPresPrintBtn.onclick`** — mismo flujo EXACTO que `presPrintBtn.onclick` de Propiedades
  (`document.body.classList.add("printing-deck")` → `preparePrintPageSize()` → cambia `document.title` →
  `window.print()` → restaura título → limpia estilos), apuntando a `#landPresPages`/
  `landPresentationView` en vez de los de Propiedades.
- **`preparePrintPageSize()` NO necesitó ningún cambio** — sigue buscando `.deck-page` de forma global (sin
  parámetro de contenedor), pero como `renderPresentationView()`/`renderLandPresentationView()` YA se
  limpian mutuamente desde la etapa 2 (guard de cross-contaminación, ver arriba), en cualquier momento dado
  solo el deck que se está viendo tiene páginas reales en el DOM — la función funciona igual de bien para
  cualquiera de los 2 sin tocarla. Esto es justo lo que se había dejado preparado a propósito en la etapa 2
  para este momento.
- **2 reglas CSS que sí estaban hardcodeadas a `#presPages` se extendieron** para incluir también
  `#landPresPages`: la visibilidad/`display` dentro de `@media print`, y el
  `position:static;...;zoom:1` de `body.printing-deck`. Inofensivo tener ambos selectores siempre activos
  (mismo razonamiento del guard de cross-contaminación: solo uno de los 2 contenedores tiene contenido real
  en cualquier momento). `.pres-sidebar`/`.pres-main` (clases, no ids) ya eran compartidas por los 2 decks
  desde que se construyó el shell de Terrenos en la etapa 1 — no necesitaron ningún cambio.
- **`buildLandPresPdfFilename()`** — misma lógica EXACTA que `buildPresPdfFilename()` de Propiedades
  (`[Empresa/Proyecto] [Tamaño] - [Mercado] (Iniciales-de-broker)`), pedido explícito del usuario, leyendo
  `landPresCoverInfo` en vez de `presCoverInfo`. `brokerInitials()`/`sanitizeFilenamePart()` se reutilizan
  tal cual — ya eran agnósticas de entidad.
- El listener `beforeprint` (invalida el tamaño del mapa de Leaflet justo antes de imprimir) ahora también
  revisa `landPresMapInstance`, además del `presMapInstance` de siempre.
- **Selector "Unidades" quitado del topbar de Terrenos (v3.39.0)**, pedido explícito del usuario — nunca
  tuvo efecto real sobre la tabla (`PRESENTATION_ROWS_LAND` no usa `unitClass` en ningún row, ver arriba),
  así que se quitó del HTML en vez de dejarlo decorativo/inerte. Ningún JS lo tenía wireado, así que no hizo
  falta tocar ninguna función al quitarlo.
- **Fix menor: "Building Options" → "Land Options"** en el texto fijo de la portada de Terrenos
  (`buildLandCoverPageHtml`, `.cover-overlay-bo-title`) — se había copiado literal de Propiedades sin
  adaptar (mismo texto que el título de la página de tabla, `<h2>Land Options</h2>`, que sí se había puesto
  bien desde la etapa 2).

- Verificado: el resto del `<script>` parsea limpio con esprima (mismo método de siempre), y los ids nuevos
  de DOM (`landCoverEditPanel`, `landPresMapCanvas`, etc.) no chocan con ninguno existente. Sin Chrome real
  disponible — pendiente que el usuario confirme que "Imprimir / PDF" de Terrenos genera un PDF correcto
  (sin páginas de más/de menos, con el nombre de archivo esperado) y que Vista Previa de macOS no muestra
  ningún problema nuevo de renderizado (dado el historial de esa app con PDFs generados por Chrome, ver
  `## Presentación (deck)` arriba).

### Presentación de Oficinas — Etapa 1 (selección en la tabla de Spaces, v4.1.0 en construcción)

Pedido explícito del usuario, tras liberar los módulos de Oficinas/Spaces a PRD (v4.0.0): un deck nuevo,
distinto de Propiedades/Terrenos, pensado para ofrecer pisos (Spaces) a un cliente. Confirmado vía
`AskUserQuestion` con el usuario 2 decisiones de diseño antes de empezar:

1. **El requerimiento del cliente se cumple seleccionando 1+ Spaces**, y el analista decide MANUALMENTE, por
   cada selección, si van agrupados en 1 sola Opción (suma de área — ej. 4 pisos de un mismo edificio
   ofrecidos como un solo paquete de 1,200 m²) o agregados individualmente (1 Opción por Space — ej. 5 pisos
   de 300 m² en el mismo edificio, ofrecidos como 5 opciones separadas, **nunca** sumados automáticamente).
   Esta decisión **nunca es automática por edificio** — puede haber varias Opciones (agrupadas o no) dentro
   del mismo edificio en una misma propuesta.
2. La propuesta armada **no necesita guardarse** (vive una sola sesión) y el flujo **arranca directo en la
   tabla de Spaces** — a diferencia de Propiedades/Terrenos, que tienen un deck/vista completa aparte
   (`presentationView`/`landPresentationView`), la Etapa 1 no agrega ninguna vista nueva.

**Mecanismo (Etapa 1, ya implementado):**
- La tabla de Spaces reusa la MISMA columna de checkbox que ya usan Propiedades/Terrenos (`showSelect` en
  `renderTable()`), pero con su propio arreglo transitorio `officeSpaceStaging` (Space `__id` marcados,
  pendientes de convertirse en Opción) — nunca comparte selección con `presentationOrder`/
  `landPresentationOrder`. Checkbox "Select all", chip de conteo y su "×" (limpiar) funcionan igual que en
  Propiedades/Terrenos, solo apuntando a `officeSpaceStaging` (ver `toggleOfficeSpaceStaging`/
  `toggleSelectAllOfficeSpaceStaging`/`clearAllOfficeSpaceStaging`).
- Con 1+ Spaces marcados aparece una barra de 2 acciones junto al chip: **"Group as one option (N)"**
  (`groupOfficeStagingAsOption()`, junta TODO lo marcado en 1 sola Opción) y **"Add N individually"**
  (`addOfficeStagingIndividually()`, crea 1 Opción por cada Space marcado). Cualquiera de las 2 vacía
  `officeSpaceStaging` al terminar.
- Cada Opción es `{id, spaceIds:[...]}` — 1 `spaceId` = individual, 2+ = agrupada (la etiqueta "Grouped"/
  "Individual" se deriva del largo del arreglo, no se guarda por separado). Viven en `officePresOptions`,
  **sin `localStorage`** (a propósito, pedido explícito "no necesita guardarse" — a diferencia de
  `PRES_ORDER_KEY`/`LAND_PRES_ORDER_KEY`, que sí persisten).
- Botón "Presentation options" (con pill de conteo) siempre visible en la tabla de Spaces, abre un modal
  simple (`officeOptionsOverlay`/`openOfficeOptionsPanel()`) con la lista de Opciones armadas hasta ahora —
  edificio(s), piso(s), # de spaces, área total y badge Grouped/Individual — cada una removible
  (`removeOfficePresOption()`).
- **Área sumada = `NET_LEASABLE_AREA_VALUE` de cada Space, NO `AVAILABLE_AREA_VALUE`.** Trampa real evitada
  a propósito: `AVAILABLE_AREA_VALUE` en Spaces es un campo autofill de solo lectura tomado de la Oficina
  completa (ver `SPACE_AUTOFILL_READONLY_KEYS`) — sumarlo entre varios Spaces del mismo edificio contaría el
  área disponible del edificio varias veces. `NET_LEASABLE_AREA_VALUE` sí es específico de cada piso.
  Normalizado a SF antes de sumar reusando `areaValueInSF()` (mismo criterio que Resumen, v3.51.8) para no
  mezclar SF y m² como si fueran el mismo número.

**Etapa 2, primera parte (Cover/Mapa/Cierre — v4.2.0 en construcción, pedido explícito del usuario: "de
entrada ya vayamos generando la portada, el mapa, y el cierre")**: el deck real de Oficinas, como 3ra opción
del mini-menú "Presentación" (`togglePresentationMenu`/`selectPresentationMenuOption('oficinas')`), deck
SEPARADO de Propiedades/Terrenos (mismo criterio de siempre) — `showOfficePresentationView()`/
`renderOfficePresentationView()`, shell `#officePresentationView` (sidebar + `#officePresPages`) y
`#officePresTopbarControls` (Design/Cover data/Print, mismo patrón que Terrenos, sin checkbox "Table" porque
esa página no existe todavía). Diferencia real con Propiedades/Terrenos: este deck **no tiene su propio
checkbox de selección** — solo LEE `officePresOptions` (la Etapa 1, arriba), nunca la modifica; el sidebar
reusa literalmente `renderOfficeOptionsPanel()` (parametrizada con `targetId`, v4.2.0) apuntando a
`#officePresSelectedList`, una sola fuente de verdad.

- **3 imágenes nuevas** (`OFFICE_COVER_IMAGE_SRC`/`OFFICE_DIVIDER_IMAGE_SRC`/`OFFICE_CLOSING_IMAGE_SRC`),
  tomadas de `Config. del Proyecto/Logo/OFFICINA_PORTADA.jpeg`/`OFFICINA_SEPARADOR.jpeg`/`OFFICINA_CIERRE.jpeg`
  (compartidas por el usuario) y embebidas en base64 exactamente igual que `COVER_IMAGE_SRC`/
  `CLOSING_IMAGE_SRC` de Propiedades/Terrenos — Oficinas necesitaba fondos propios (distintos), así que no
  comparte esas 2 constantes. `OFFICE_DIVIDER_IMAGE_SRC` ("Separador") **todavía no se usa en ninguna
  página** — reservada para las páginas divisoras por submercado de una etapa posterior (ver la referencia de
  37 páginas del usuario, `Arca 4,000 m2 (VM).pdf`), se dejó lista de una vez.
- **Portada** (`buildOfficeCoverPageHtml`): a diferencia de `.cover-overlay` de Propiedades/Terrenos
  (mes/año + m² + lista de mercados), la imagen base de Oficinas YA trae "Análisis de Mercado" quemado en la
  foto — el overlay dinámico (`.office-cover-overlay`, nueva clase CSS) es solo "Para: / [Cliente] /
  [Fecha]", 2 campos (`officePresCoverInfo`, editables desde "Cover data") en vez de los 5 de Propiedades
  (sin Size/Markets/Brokers/Company — no aplican a este formato de portada).
- **Mapa** (`buildOfficeMapPageHtml`/`renderOfficePresMap`, propia instancia `officePresMapInstance` +
  canvas `#officePresMapCanvas`, mismo patrón CARTO Voyager + `declutterMapPins` que los otros 2 mapas):
  los pines NO son los Spaces de `officePresOptions` (varios pisos del mismo edificio serían el mismo pin
  repetido) sino los **edificios (OFICINAS) únicos** referenciados por esas Opciones —
  `officePresBuildingList()` resuelve cada Space a su Oficina vía `MAPPING_CODE` y deduplica.
- **Cierre**: `buildClosingPageHtml()` se parametrizó (`imgSrc` opcional, default `CLOSING_IMAGE_SRC` — sin
  cambios para Propiedades/Terrenos) en vez de duplicarse una 3ra vez, ya que el resto de esa página sigue
  siendo agnóstico de entidad; Oficinas la llama como `buildClosingPageHtml(OFFICE_CLOSING_IMAGE_SRC)`.
- **Guard de cross-contaminación** (mismo criterio que Propiedades↔Terrenos): `renderPresentationView()` y
  `renderLandPresentationView()` ahora TAMBIÉN limpian `#officePresPages`, y
  `renderOfficePresentationView()` limpia `#presPages`/`#landPresPages` — en cualquier momento dado solo el
  deck que se está viendo tiene páginas reales en el DOM (`preparePrintPageSize()`/impresión buscan
  `.deck-page` de forma global, sin distinguir contenedor).
- **Imprimir/PDF** (`officePresPrintBtn`) — mismo flujo exacto que Propiedades/Terrenos,
  `buildOfficePresPdfFilename()` usa el nombre del Cliente. `preparePrintPageSize()` no necesitó ningún
  cambio (ya agrupa por clase `.cover-page`/`.map-page`, agnóstico de qué deck las generó).
- Sin persistir nada de `officePresOptions` (eso sigue siendo la Etapa 1, deliberadamente sin
  `localStorage`) — pero `officePresCoverInfo`/`officePresShowState` (Cliente/Fecha, qué secciones se
  muestran) SÍ persisten en `localStorage`, igual que sus equivalentes de Propiedades/Terrenos.

**Etapa 2, ajustes (v4.2.1, pedidos explícitos del usuario tras la primera prueba)**:
- **Bug real: "hice una prueba de impresión y no me mostró nada... hojas en blanco"** — causa confirmada:
  las 2 reglas CSS de `@media print`/`body.printing-deck` que hacen visible el deck activo
  (`#presPages`/`#landPresPages{visibility:visible}` y su versión `position:static`) nunca se habían
  extendido a `#officePresPages` — heredaba `visibility:hidden` de la regla `body *` de arriba sin importar
  que `preparePrintPageSize()` sí midiera bien sus `.deck-page`. Se agregó `#officePresPages` a ambas reglas
  (ver `@media print` y el bloque `body.printing-deck` cerca de la línea 590 del HTML).
- **Reordenar Opciones (drag&drop)** — mismo mecanismo que `.pres-item` de Propiedades
  (`officeOptionDragStart`/`DragOver`/`Drop` sobre `officePresOptions`, `FC_DRAG_ICON` incluido). Funciona
  en los 2 lugares donde vive `renderOfficeOptionsPanel()` (panel acoplado de Spaces y sidebar del deck),
  porque ambos pintan el mismo `officePresOptions` — reordenar en cualquiera de los 2 refresca ambos (y el
  deck completo, si está activo, porque el orden también cambia la numeración de páginas/pines).
- **Selector "Units" propio del deck** (`officePresUnitSelect`/`officePresUnitPref`, con su propio
  `localStorage`) — controla el área mostrada en el sidebar de Opciones DE ESTE deck, independiente del
  selector "Units" de la tabla de Spaces (`unitViewPref`, que sigue controlando el panel acoplado ahí).
  `officeOptionAreaForDisplay()`/`officeOptionInfo()`/`renderOfficeOptionsPanel()` ahora aceptan un
  `unitPref` opcional (default `unitViewPref` si se omite) para poder pasar cualquiera de los 2 según dónde
  se esté pintando la lista.
- **Rediseño del Mapa** (referencia visual del usuario): reemplaza el badge flotante genérico de
  `.map-page-badge`/`.map-page-attribution` por un banner blanco arriba (`DECK_LOGO_SRC` + nombre del
  Mercado — **corrección v4.2.4**: `DECK_LOGO_SRC` ES el ícono circular solo, sin ningún wordmark; la nota
  original aquí decía lo contrario y estaba mal — se creó una `CITIUS_ICON_SRC` redundante por error en
  v4.2.1/2 y se eliminó al confirmarse que era el mismo PNG) y un banner amarillo a la derecha (cada
  Submercado con la lista numerada de sus edificios, `.office-map-legend`). Los pines del mapa y los
  números de la leyenda usan la MISMA lista reordenada por submercado
  (`officePresBuildingsGroupedBySubmarket()`, calculada una sola vez en `renderOfficePresentationView()` y
  reusada tanto para el HTML de la leyenda como para `renderOfficePresMap()`) — nunca la lista cruda de
  `officePresBuildingList()`, para que un pin y su número en la leyenda siempre calcen. La página conserva
  la clase `.map-page` (además de la nueva `.office-map-page`) a propósito: `preparePrintPageSize()` agrupa
  el tamaño de impresión por esa clase (ver `groupOf()`), perderla la habría hecho caer en el grupo
  "tablePage" por default.
- **v4.2.4, corrección pedida por el usuario**: el logo/texto del banner blanco se había igualado a
  `.map-page-badge` (el mapa CHICO de Propiedades/Terrenos, 26px/18px) — el usuario aclaró que se refería a
  la cabecera de la TABLA COMPARATIVA "Building Options" (`.deck-header`/`.deck-logo`/`.deck-header h2`,
  34px/32px, mucho más grande). Se igualó a esa especificación en su lugar. Además, un 2do incremento de
  letra del banner amarillo (13px/14px → 15px/17px), pedido explícito, tras el primer incremento de v4.2.2
  (11px/12px → 13px/14px) resultar insuficiente.
- **v4.2.5, bug real reportado por el usuario ("empiece un poco más arriba, a la misma altura de como
  empieza el mapa")**: aun con `padding-top:14%` (v4.2.2) el contenido del banner amarillo seguía arrancando
  visiblemente más abajo que el mapa. Causa: `padding-top`/`padding-bottom` de CUALQUIER caja (posicionada o
  no) se resuelve en CSS contra el ANCHO del contenedor, nunca el alto — con la página en horizontal (ancho
  > alto), 14% del ancho daba más píxeles que 14% del alto real del banner. Fix: se movió ese offset de
  `padding-top` (en `.office-map-legend`) a `top` (offset real, que sí se resuelve contra el alto) en un
  nuevo wrapper interno, `.office-map-legend-inner` — el `overflow-y:auto` para scroll también se movió a
  ese wrapper.

**Etapa 2, ajustes (v4.2.2, pedidos explícitos del usuario tras probar el mapa nuevo)**:
- **Bug real: "no está mostrando todos los pins"** — causa confirmada: `officePresBuildingList()` resolvía
  cada Space a su Oficina buscando `sp.MAPPING_CODE` en un `Map` de Oficinas; si ese código no calzaba
  EXACTO con ningún registro (dato mal capturado/editado a mano en alguno de los 2 lados), el edificio se
  descartaba **en silencio** — ni pin, ni aviso, ni contaba como "sin coordenadas". Ahora se cuenta
  (`unresolvedCount`) y se avisa en la página del mapa junto a la nota de coordenadas faltantes. No se
  intentó "adivinar" el match (ej. comparar sin mayúsculas/espacios) — MAPPING_CODE es llave exacta en toda
  la app, así que si el dato real no calza es mejor que el usuario lo vea y lo corrija en Spaces/Oficinas.
- **Banner blanco del Mapa** ahora es EXACTAMENTE la misma especificación que el badge de "Building Options"
  (`.map-page-badge`/`img`/`h2`: fila, gap 10px, ícono 26x26, texto 18px/400/#666/'Dala Moa') — se quitó la
  línea decorativa que tenía antes (esa badge tampoco la tiene).
- **Banner amarillo**: la lista de submercados/edificios ahora arranca a la misma altura donde arranca el
  mapa (antes arrancaba pegada arriba, a la altura del banner blanco) y con letra más grande (11px/12px →
  13px/14px).

**Etapa 3 (v4.4.0 en construcción, pedido explícito del usuario con referencia visual real): página de
detalle por Opción + selector de idioma EN/ES.**

- **Selector de idioma** (`officePresLangSelect`/`officePresLangPref`, propio de este deck, con su propio
  `localStorage`) — solo cambia el TEXTO mostrado: las etiquetas fijas de la página nueva (bilingües,
  `{en,es}` en cada fila de `officeDeckSpaceInfoRows`/`officeDeckBuildingInfoRows`, elegidas con el helper
  local `t(en,es)` dentro de cada `build*Html()`) y los VALORES que vienen de una lista (vía
  `officePresTranslateValue(listName, valor)`/`officePresTranslateMulti` para AMENITIES, que usa
  `listTranslationsEs` — la traducción global de la Etapa de Configuración > Listas, ver arriba). Los datos
  capturados (`lists`) nunca se tocan — si no hay traducción registrada para un valor, se muestra el inglés
  tal cual, nunca vacío.
- **2 páginas por Opción** (`buildOfficeOptionDetailPageHtml`/`buildOfficeOptionPhotosPageHtml`), agregadas
  entre Mapa y Cierre, con su propio toggle "Options" en el panel Design (`officePresShowState.options`,
  default `true`):
  1. Foto del edificio (arriba-izq) + tabla "Información del Espacio" (arriba-der) + Plano (abajo-izq) +
     tabla "Información del Edificio" (abajo-der). La 1ra fila de "Información del Espacio" es "Nivel(es)"
     (pedido explícito del usuario: "esta información esté en la tabla como primer dato antes de área neta
     rentable, justo como ya lo muestra en el selector de espacios" — reusa el mismo `info.floors` que ya
     arma `officeOptionInfo()` para el panel de Opciones).
  2. 4 fotos de interior.
  - **Regla de grupo, confirmada explícitamente por el usuario**: cuando una Opción agrupa 2+ Spaces, la
    foto del edificio/plano/4 fotos de interior SIEMPRE se toman del **primer Space del grupo**
    (`officeOptionPrimarySpace()`, primer elemento de `opt.spaceIds` en el orden en que se marcaron) —
    NUNCA se combinan fotos de varios Spaces en una sola página. Las ÁREAS sí se SUMAN entre todos los
    Spaces del grupo (`sumSpacesAreaSF`, mismo criterio que ya usaba `officeOptionAreaSF` para el sidebar);
    el resto de los campos del Space (condición, disponibilidad, tarifas) usa el primer Space, mismo
    criterio que las fotos — documentado aquí por si el usuario pide un criterio distinto más adelante (ej.
    mostrar un rango cuando los pisos del grupo difieren).
  - **Fallback de las 4 fotos de interior, pedido explícito del usuario**: slots 1-2 son SIEMPRE Interior
    (1)/(2) del Space primario; slots 3-4 son Interior (3)/(4) del MISMO Space **si existen**
    (`hasPhoto()`), y si no, caen a Lobby + la 1ra foto de Amenidades del EDIFICIO ("si no tiene 4 fotos
    del interior sería usar al menos 2... las otras 2 de abajo serían del Lobby y la primera de amenidades
    del edificio").
  - **Mapeo de campos** (`officeDeckSpaceInfoRows`/`officeDeckBuildingInfoRows`) — elegido a falta de un
    campo 1:1 exacto en algunos casos, documentado para que el usuario lo corrija si no es lo que esperaba:
    "Uso del Edificio" usa `CATEGORY_OFFICES` (no hay un campo "Building Use" separado); "Desarrollador" usa
    `OFICINAS.DEVELOPER` (lista de códigos A/B/C/D, igual que en el resto de la app). Tarifas (Precio de
    renta/Mantenimiento/Costo de cajón) reusan el mismo criterio de `presValueFor()` de Propiedades: 2
    decimales siempre, prefijo "MX $"/"US $" real (`presCurrencyPrefix`, agnóstica de entidad),
    convertidas a la unidad de área elegida vía `convertCellForView` (ya sabe convertir la lista
    CON_COMERCIAL, ej. "MXN/SF" ↔ "MXN/m²" — sin necesitar ningún cambio ahí).
  - **`renderOfficePresentationView()` ahora es `async`** — primer deck de Oficinas con fotos reales, por
    lo que `await ensurePhotoSet()` antes de decidir el fallback (necesita saber si Interior 3/4 existen
    ANTES de renderizar) y `revokeAllDeckPhotoUrls()` al inicio (mismo criterio que
    `renderPresentationView()` de Propiedades). `.office-detail-page` es una `.deck-page` normal (padding
    de siempre, sin aspect-ratio forzado) — cae en el bucket "tablePage" por default de
    `preparePrintPageSize()`, igual que "Table"/"List" de Propiedades, sin necesitar ningún cambio ahí.

**Etapa 3, ajustes (v4.4.1, pedidos explícitos del usuario tras la primera prueba)**:
- **Título + ícono de la página de Opción, igualado al del Mapa** ("mismo tamaño de ícono y de letra") —
  se dejó de usar el estilo propio (línea + serif 20px, calcado del building-detail de la referencia Arca)
  y ahora reusa literal `.deck-header`/`.deck-logo`/`<h2>` (34px/32px, la misma especificación de
  "Building Options" que ya usa el Mapa) — ya no existen las clases `.office-detail-header`/
  `.office-detail-title-row`/`.office-detail-title-rule`/`.office-detail-title`/`.office-detail-logo`.
- **Quitadas 2 etiquetas redundantes**: el título "Plano"/"Floor Plan" arriba de la foto del plano, y el
  texto "Niveles X, Y" debajo de él (`.office-detail-plan-caption`, eliminada) — el nivel/piso ya se ve
  como 1ra fila de "Información del Espacio" ("Nivel(es)"), pedido explícito: "no es necesario" repetirlo.
- **Nota de pie de página**: alineada a la izquierda (antes centrada) y letra más grande (9px → 11px).
- **El plano ya NO se recorta**: usaba `object-fit:cover` (igual que la foto del edificio/interiores, que sí
  deben llenar el recuadro) — el plano necesita verse COMPLETO, así que tiene su propia regla
  (`.office-detail-plan img{object-fit:contain}`) que lo deja con franjas del fondo gris si no calza exacto
  al recuadro 4:3, en vez de recortar bordes.

**Etapa 3, ajustes (v4.4.2, pedidos explícitos del usuario tras ver el `contain` de arriba en pantalla)**:
- **Relleno gris arriba/abajo del plano, corregido de raíz** — el recuadro genérico es 4:3 (1.33), pero los
  Layout de Spaces se GUARDAN siempre a 600×405 (`PHOTO_TARGET_W`/`H`, SPACES no tiene entrada en
  `PHOTO_SIZE_OVERRIDES`) — 600/405 ≈ 1.48, esa diferencia de proporción era justo el relleno gris que
  `contain` (v4.4.1) dejaba arriba/abajo. `.office-detail-plan{aspect-ratio:600/405}` iguala el recuadro al
  aspect-ratio REAL del archivo guardado — `contain` sigue ahí solo como red de seguridad, ya no como el
  mecanismo que "resuelve" el ajuste rellenando.
- **Marco del plano quitado** (`.office-detail-plan{border:none}`) — a diferencia de la foto del
  edificio/interiores, que sí lo conservan.
- **Alineación entre "Información del Espacio" e "Información del Edificio"** — son 2 `<table>` DISTINTAS;
  sin ancho fijo, cada una autoajustaba su columna de etiqueta al texto más largo de ESA tabla nada más
  (`Space` tiene etiquetas más largas, ej. "Additional Spot Cost ($/spot/month)"), así que la columna de
  valor de cada una arrancaba en un punto horizontal distinto. Fix: `table-layout:fixed` +
  `td:first-child{width:230px}` igual en las 2 — ahora ambas columnas de valor arrancan exactamente en el
  mismo punto.
- **2do incremento de letra de la nota de pie** (11px → 13px), tras el primero (v4.4.1, 9px → 11px)
  resultar insuficiente.

**Etapa 3, ajustes (v4.4.3, pedidos explícitos del usuario)**:
- Título de tabla ("Space Information"/"Building Information") más grande (13px → 15px) y en el amarillo
  REAL de Citius (`--accent`, no `--accent-ink` — ese es el tono oscuro para texto SOBRE el amarillo).
- Nota de pie reducida un poco (13px → 12px) — el 2do incremento de arriba se sintió grande.
- **Divisor de Submercado** (`buildOfficeDividerPageHtml()`) — usa `OFFICE_DIVIDER_IMAGE_SRC` (embebida
  desde la Etapa 2, reservada para esto desde entonces; el ícono blanco ya viene quemado en la foto, lo
  único dinámico es el nombre del submercado, mismo estilo línea+serif que el resto del deck). Se inserta
  en `renderOfficePresentationView()` cada vez que el submercado de la Opción actual
  (`officeOptionSubmarket()`, vía la Oficina ligada) difiere del de la anterior, **en el orden actual de
  `officePresOptions`** — no se reagrupan/reordenan las Opciones por submercado; si el usuario quiere los
  divisores agrupando un mismo submercado, debe dejar esas Opciones contiguas él mismo (ya puede
  reordenarlas arrastrando en el sidebar). No tiene su propio toggle en Design — vive bajo "Options"
  (dividers solo tienen sentido si las páginas de Opción están visibles).

**Etapa 3, ajustes (v4.4.4, pedidos explícitos del usuario)**:
- **Tipografía real del deck** — `.office-detail-page{font-family:'Unitext',sans-serif}` (nuevo). `.deck-page`
  nunca trae font-family propio (cada hijo lo declara aparte: `.deck-grid`/`.deck-note`/`.layout-table`); la
  tabla de Opción nunca lo había declarado, así que caía en la fuente del sistema de la app en vez de
  'Unitext' (la que el usuario ve en el resto de la presentación, y a la que se refería como "OpenSans").
  Se declara UNA vez en el contenedor y cae en cascada a la tabla/nota/título de sección — `.deck-header h2`
  (los títulos, "Dala Moa" serif) no se ve afectado, tiene su propio font-family más específico.
- **Etiqueta/valor invertidos** ("quiero que el título de la etiqueta... esté en negritas y los datos en
  normal, al revés de como los tenemos") — antes la etiqueta era gris + peso normal y el valor negrita;
  ahora la etiqueta es gris + NEGRITA (mismo gris de siempre, el usuario no pidió cambiar el color) y el
  valor pasa a peso normal.
- Letra de la tabla más grande (12px → 14px).

**Etapa 3, ajustes (v4.4.5, pedidos explícitos del usuario)**:
- **Columna de valor casi se sobreponía con la etiqueta** ("Additional Spot Cost ($/spot/month)" era la más
  larga, y con la etiqueta en negrita de v4.4.4 ocupaba más ancho que antes) — `td:first-child` sube de
  230px+12px a 270px+18px de aire.
- **2do incremento del título de tabla** ("tienen que resaltar más"), 15px → 19px, tras el primero (v4.4.3,
  13px → 15px) no ser suficiente.

**Etapa 3, ajustes (v4.4.6, bugs reales reportados por el usuario)**:
- **Bug grave de raíz: Mapping Code cambiaba de tipo (String↔Number) al guardar** — reportado como "en el
  mapa no me muestra el pin... Mapping Code no coincide" y "esto solo pasa cuando edito algo de las
  Propiedades/Oficinas, no pasa con los Spaces" (con capturas confirmando que Oficina y Space SÍ tenían el
  mismo código capturado en pantalla). Causa raíz real, en `saveBtn.onclick`: el colector genérico de
  campos de texto convertía CUALQUIER valor que "pareciera" numérico a un Number de JS vía `maybeNum()` —
  `isNumericField()` siempre regresa `true`, así que esto aplicaba a TODOS los campos de texto sin
  distinción. Un Mapping Code corto como "1" pasaba de String a Number cada vez que ESE registro se
  guardaba; el otro lado de la relación (un Space con el mismo código, guardado en otro momento) se quedaba
  como String — cualquier comparación estricta (`Map.get()`, `.find(r=>r.MAPPING_CODE===...)`) dejaba de
  encontrar el match, aunque en pantalla ambos se vieran idénticos ("1" y 1 se muestran igual). Esto
  explica también por qué el usuario podía "arreglarlo" resolviendo cada Space a mano (los dejaba con el
  MISMO tipo que la Oficina en ese momento) pero se volvía a romper la próxima vez que editara la Oficina
  (que reconvierte su propio Mapping Code a Number en cada guardado, sin importar qué campo se haya
  editado).
  - **Fix de raíz**: nuevo `ID_LIKE_FIELD_KEYS = new Set(["MAPPING_CODE","ID_PARK","ID__LAND","SPACE_CODE"])`
    — estos 4 campos (los mismos valores de `PHOTO_ID_FIELD`, más "MAPPING_CODE" que además se reusa como
    referencia foránea en Spaces/Transacciones) quedan EXENTOS de `maybeNum()` en el colector — nunca se
    convierten a número sin importar qué tan "numérico" parezca su valor, de aquí en adelante.
  - **Defensa adicional** en el código de Oficinas construido en esta Etapa (`officePresBuildingList()`/
    `officeOptionOfficeRecord()`): sus 2 `Map` de Mapping Code → Oficina ahora comparan con `String(...)` de
    los 2 lados (mismo criterio que ya usaba `relatedParkFor()` con ID_PARK, un bug de la misma familia
    corregido antes) — así los datos que YA quedaron con el tipo equivocado (guardados antes de este fix)
    también encuentran su match, sin que el usuario tenga que volver a guardar nada a mano.
- **Bug real: texto largo se desbordaba de la tabla** (reportado con Amenidades) — `white-space:nowrap`
  (pensado para que un valor corto tipo "$320.00" nunca partiera a media palabra) forzaba TODO el texto a
  una sola línea; con la columna de ancho fijo (`table-layout:fixed`), un valor largo no tenía a dónde ir
  más que desbordarse. Quitado de `td:last-child` — el texto envuelve normal dentro de su columna.
- **Divisor de Submercado, reposicionado** (pedido explícito del usuario) — el ícono Y la línea horizontal
  YA vienen quemados en `OFFICE_DIVIDER_IMAGE_SRC` (parte de la foto); antes se dibujaba una 2da línea
  propia (CSS) redundante con la de la foto, y el texto no estaba alineado con la línea real. Medido
  directo sobre el archivo: la línea real arranca en el borde izquierdo (x=0) y termina ~5% del ancho, a
  ~33% de alto. Se quitó la línea propia y el texto ahora se posiciona exactamente a esa altura
  (`top:33.3%`, `transform:translateY(-50%)` para centrarlo verticalmente sobre ese punto) arrancando justo
  después de donde termina la línea real (`left:6.5%`). Letra más grande (32px → 40px).

**Etapa 3, v4.4.7 — el desborde de texto (Amenidades) seguía pasando tras v4.4.6**: quitar `white-space:
nowrap` no fue suficiente porque la causa de fondo era otra. Un hijo de CSS Grid tiene `min-width:auto` por
default (no `0`) — el TRACK de la columna (o la tabla dentro de él) podía terminar desbordándose sin
encogerse para no bajar del ancho mínimo de su contenido, sin importar que la tabla ya tuviera
`table-layout:fixed` y texto que sí podía envolver (un problema conocido de tablas dentro de Grid/Flexbox).
Fix: `min-width:0` en los hijos directos de `.office-detail-grid` y en `.office-detail-table` misma, más
`overflow-wrap:break-word` en `td:last-child` como red de seguridad para un valor sin espacios donde
envolver.

**Etapa 3, v4.4.8 — el desborde de Amenidades seguía pasando tras v4.4.6/v4.4.7**: el elemento `<table>`
seguía sin respetar el ancho de su columna en este contexto anidado (Grid > div > table), incluso con
`table-layout:fixed` + `white-space:nowrap` quitado + `min-width:0` en la tabla y en los hijos de la
grilla. En vez de seguir peleando contra el algoritmo de layout de `<table>` dentro de Grid/Flexbox, se
reemplazó por 2 `<div>` en fila (flexbox, `.office-detail-row`/`.office-detail-row-label`/
`.office-detail-row-value`, `flex:1;min-width:0` en el valor) — mismo look exacto, pero un patrón mucho más
simple y confiable para que un texto largo (Amenidades) envuelva dentro de su columna. `officeDeckInfoTableHtml()`
ya no genera un `<table>`, genera estos `<div>`.

**Etapa 3, v4.4.9, pedido explícito del usuario**: "Floor(s)"/"Nivel(es)" se movió de 1er lugar en
"Información del Espacio" a justo debajo de "Delivery Condition"/"Condición de Entrega" (`officeDeckSpaceInfoRows()`)
— las menciones más arriba en este documento de que es "la 1ra fila" quedan como historial de la Etapa 3
original, ya no reflejan el orden actual.

**Etapa 3, v4.4.10, 3 ajustes más de la página de detalle, pedidos explícitos del usuario (captura con 4
puntos)**:
1. **Línea final en las tablas de información** — `.office-detail-row:last-child{border-bottom:none}` le
   quitaba a propósito el borde a la ÚLTIMA fila (criterio normal de "no dejar una línea colgando"), pero el
   usuario quiere justo lo contrario ("así como se tiene en cada valor") — se quitó esa excepción, ahora la
   última fila también cierra con su `border-bottom`.
2. **Más espacio entre el Título y las imágenes/tabla** — `.deck-header{margin-bottom:16px}` es GLOBAL (la
   comparten el Mapa y "Building Options"), así que no se tocó esa regla directo; se agregó un override más
   específico `.office-detail-page .deck-header{margin-bottom:32px}` (2 clases le gana a 1 sin importar el
   orden en la hoja) que solo afecta esta página.
3. **Área de cada piso en paréntesis, en la fila "Floor(s)"** — antes esa fila reusaba
   `officeOptionInfo(opt).floors` (un join de los `LEVEL_FLOOR` únicos del grupo, sin ningún dato de área).
   Función nueva `officeDeckFloorsWithArea(recs, unitPref)`: recorre cada Space del grupo SIN deduplicar por
   piso (a propósito — 2 Spaces podrían compartir el mismo `LEVEL_FLOOR` pero cada uno tiene su propia
   área) y arma `"Piso (área unidad)"` por cada uno, reusando `areaValueInSF`/`officeDeckAreaText` (misma
   fuente de conversión/redondeo que ya usa el total sumado de "Net Leasable Area" arriba) para que la
   unidad siempre calce con la vista Métrico/Imperial activa.

**Market Status pasa a ser un campo propio de Spaces, ya no de Oficinas (v4.4.10)** — 4to punto de la misma
captura ("Market Status debe estar en los espacios no en Offices"), confirmado con el usuario vía
`AskUserQuestion` que se trataba del **modelo de datos** (no de agregar una fila nueva a la página de
detalle del deck). Antes de este cambio, `MARKET_STATUS` existía en LAS 2 entidades a la vez con roles
distintos: en `BUNDLE.OFICINAS.fields` era un campo editable de verdad (con su propio `<select>`, vía
`FIELD_LIST_MAP.OFICINAS`); en `BUNDLE.SPACES.fields` (grupo "PROPIEDAD") vivía como copia de **solo
lectura**, autocompletada desde la Oficina ligada por `MAPPING_CODE` — parte de
`SPACE_AUTOFILL_READONLY_KEYS`, así que se llenaba sola al elegir la Oficina desde el picker
(`autofillSpaceFromMappingCode`) y se volvía a sobreescribir cada vez que esa Oficina se guardaba
(`recomputeAllSpaceLinkedFields`), sin que el usuario pudiera nunca editarla directo en el Space. El pedido
invierte esa relación: el estatus de mercado es genuinamente por-piso (un edificio puede tener pisos
Vacant/Available/Occupied distintos a la vez), así que debe capturarse en Spaces, y Oficinas deja de tener
su propio Market Status independiente.
- **`BUNDLE.OFICINAS.fields`**: se quitó la entrada `{"key":"MARKET_STATUS","group":"IDENTIFICATION"}` —
  mismo patrón ya establecido varias veces en este documento ("Quitar un campo de esta app sin tocar el
  Excel del usuario", ver `## Modelo de datos`): `pruneFieldConfigToKnownFields()` limpia sola
  `fieldConfig.OFICINAS` en la próxima carga, y `buildWorkbook()` deja de escribir esa columna pero
  preserva intacta cualquier columna/dato ya capturado en el Excel real (cae a `r.__row[ci]`, el valor
  crudo). También se quitó su entrada de `FIELD_LIST_MAP.OFICINAS`.
- **`FIELD_LIST_MAP.SPACES`** ganó `"MARKET_STATUS":"MARKET_STATUS"` (antes no existía ahí, porque el
  campo nunca necesitó su propio `<select>` — se llenaba solo). `SPACE_AUTOFILL_READONLY_KEYS` perdió
  `"MARKET_STATUS"` — con eso, el branch de `renderStandaloneField()` que renderiza los campos de este Set
  como `<input readonly>` (línea ~5392) deja de interceptarlo, y el campo cae al branch genérico de lista
  (`if(listName)`, ~línea 5444): se renderiza como un `<select>` normal, editable, con la lista
  `MARKET_STATUS` (Available/Vacant/Occupied) — igual que cualquier otro campo de lista de la ficha.
- **`ENTITIES.OFICINAS`**: se quitó `"MARKET_STATUS"` de `listCols`, y `statusField` pasó de
  `"MARKET_STATUS"` a `"STATUS"` (Ready/Under Construction/Under Development/Planned, campo que Oficinas ya
  tenía desde antes) — el chip de color de esa columna en la tabla de Oficinas ahora usa `statusPill()`
  genérico (por substring) en vez de `marketStatusPill()` (comparación exacta contra
  Available/Vacant/Occupied), mismo tratamiento que ya usa Parques para su propio `statusField:"STATUS"`.
  El dispatch en `renderTable()` (`c==="MARKET_STATUS"?marketStatusPill(...):statusPill(...)`) es por
  NOMBRE DE COLUMNA, no por entidad, así que no necesitó ningún cambio — simplemente ya no evalúa a
  verdadero para la tabla de Oficinas.
- **`ENTITIES.SPACES`** no cambió — ya tenía `statusField:"MARKET_STATUS"` desde que se creó esa entidad
  (aunque el campo en sí fuera de solo lectura hasta ahora), así que la tabla de Spaces ya mostraba el chip
  de color correcto; lo único que cambia es que ahora el VALOR se captura ahí mismo en vez de heredarse.
- **Datos ya capturados no se pierden ni se tocan**: los Spaces que ya tenían un `MARKET_STATUS` copiado
  (de cuando aún era autofill) se quedan con ese mismo valor tal cual — simplemente deja de refrescarse
  solo cuando se edite la Oficina ligada; el usuario lo puede corregir a mano desde ahora si hace falta.
  Ningún otro punto del código (Resumen, KPIs, el deck de Oficinas) leía `OFICINAS.MARKET_STATUS`
  directamente — se revisó cada uso real de `MARKET_STATUS` en el archivo antes de hacer el cambio y todos
  los demás (Resumen, `AVAILABLE_BUILDINGS` de Parques) ya operan sobre `state.PROPIEDADES`, nunca sobre
  `state.OFICINAS`.

**Etapa 3, pendiente**: el "Resumen de Espacios Propuestos" final (tabla comparativa de todas las Opciones).

## Convención de versionado

Debajo del botón Configuración se muestra un tag de versión (`<div class="app-version">v3.40.1</div>`,
texto plano hardcodeado, sin build). **Se debe subir a mano en cada cambio futuro** siguiendo esta regla
dada por el usuario:
- Último dígito (patch): cambios pequeños o corrección de bugs.
- Dígito medio (minor): nueva funcionalidad sobre un módulo ya existente.
- Primer dígito (major): módulo(s) nuevo(s) — ej. pasar de 1.x.x a 2.0.0.

Al subir un dígito, los dígitos a su derecha se reinician a 0 (semver estándar). Si no es obvio en qué
categoría cae un cambio, preguntar antes de decidir el número.

## Paleta y convenciones de diseño usadas en todo el archivo

```
--bg:#f4f5f7        fondo de página
--panel:#ffffff     tarjetas/paneles
--panel-2:#f0f1f4   fondo de inputs/hover
--line:#e3e6eb      bordes
--txt:#1f2733       texto principal
--muted:#6b7480     texto secundario/labels
--accent:#f0b323    acento (amarillo Citius)
--accent-2:#d99a0e  acento hover
--accent-ink:#3a2c05 texto sobre acento
--green:#1e9e5a  --red:#e2483d  --amber:#f5b041   (estados)
```
Tipografía: stack de sistema (`-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, ...`). Los paneles
usan `border-radius:12px` (`--radius`), tarjetas con `border:1px solid var(--line)`. Los menús flotantes
(`.row-menu-dropdown` y variantes como `.config-menu-dropdown`) son `position:fixed`, con animación de
entrada corta y se cierran con un listener de `mousedown` fuera del elemento + tecla Escape.

**Íconos**: no se usan emoji en ningún botón de la interfaz — todos son SVG inline en un solo estilo
consistente (`viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
stroke-linejoin="round"`, típicamente 13-15px, alineados junto a texto con `vertical-align:-3px`). Constantes
reutilizables cerca de `MAP_ICON` (~línea 1039): `ICO_EDIT`, `ICO_TRASH`, `ICO_PRINTER`, `ICO_CAMERA`,
`ICO_LOCK`, `ICO_MAP`, `ICO_FOLDER`, más `FC_DRAG_ICON` (grip de arrastre en Configuración > Campos) definido
por separado. Si un botón vive en HTML estático (no en un template de JS), el SVG va escrito literal ahí
mismo (no se puede interpolar una constante JS en markup estático) — ver el botón "Editar" de la ficha o
"Imprimir / Descargar PDF" de Presentación como ejemplos. Símbolos como `✕` (cerrar), `✓` (confirmación
inline) y `↩`/`⚙` en texto descriptivo se dejaron igual a propósito — no son emoji ni íconos de botón per se.

## Cosas explícitamente decididas por el usuario (no revertir sin preguntar)

- Clic en cualquier parte de la fila de la tabla abre la ficha; solo el círculo amarillo selecciona para
  presentación (no el Mapping Code).
- Guardar en la ficha **no cierra el modal** y se queda en modo edición.
- El botón "Listas" del sidebar fue eliminado a propósito; Listas vive dentro del menú de Configuración.
- La protección de concurrencia es deliberadamente ligera (comparar timestamp + confirmar antes de
  escribir), **no** un merge automático ni polling continuo — se evaluaron esas opciones y se descartaron
  por complejidad/costo de rendimiento frente al beneficio.
