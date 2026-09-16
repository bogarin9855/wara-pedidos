# Wara Pedidos — Contexto del proyecto

Sistema de pedidos de viandas para **Wara**, un negocio de comida saludable en Salta, Argentina. Lo construyó Lucas (no es desarrollador de profesión; programa con ayuda de IA). Este README explica cómo está armado para poder seguir editándolo desde Claude Code.

> **Antes de editar nada: leé este README completo y el `index.html` completo.** No rompas las funciones existentes (lista abajo). Si una mejora toca la estructura de datos, avisá que hay que tocar el Apps Script y la planilla.

---

## Qué es y para qué sirve

Wara le provee el menú diario a los colaboradores de la empresa **Andreani** (sucursales en Salta, ~39 personas). Antes los pedidos se tomaban con un Excel semanal engorroso. Este sistema lo reemplazó: cada colaborador elige su menú de la semana desde el celular y Wara recibe todo consolidado. Está **en producción con un cliente real** (Andreani), con abono mensual.

Dos vistas dentro de la misma página:
- **"Hago mi pedido"** (colaborador): elige su nombre, su sucursal (automática salvo los rotativos), y va día por día eligiendo menú + postre + una observación opcional.
- **"Panel Wara"** (privado, con clave): pedidos consolidados, totales de cocina por día, lista de entrega por sucursal, planilla imprimible, quiénes faltan pedir, ausencias, pedidos modificados.

---

## Arquitectura

Stack de bajo costo, sin servidor propio:

- **Frontend**: un único `index.html` (HTML + CSS + JavaScript en un archivo, sin frameworks). Se despliega en **Vercel**, conectado a un repo de GitHub (`wara-pedidos`). Al pushear a GitHub, Vercel actualiza solo.
- **Backend / base de datos**: una **planilla de Google** + **Google Apps Script** (`Apps_Script_Wara.gs`), publicado como Web App. Su URL está en la constante `API_URL` del `index.html`.
- **Logo**: `logo.png` en la raíz del repo, referenciado desde `index.html` (constante `LOGO`). (Antes estaba incrustado en base64; se extrajo para alivianar el archivo: de ~135 KB a ~43 KB.)

Flujo de datos:
- La página hace `fetch` GET a `API_URL` para cargar config, menús, colaboradores y pedidos (`arrancar()` / `refrescarPedidos()`).
- Al enviar un pedido o una acción de panel, hace POST con `body` en **texto plano** (sin headers, para evitar problemas de CORS con Apps Script).
- El Apps Script lee/escribe en las pestañas de la planilla.

---

## Estructura de la planilla de Google

**Pestañas que carga Lucas:**
- **Menu**: `CODIGO | CATEGORIA | <Lunes> | <Martes> | <Miércoles> | <Jueves> | <Viernes>`. Filas con códigos `1..7`, `D` (Dieta), `E` (Ensalada), y `P1/P2/P3` (postres). Los encabezados de las columnas de días son las fechas (ej. "Martes 16"). **Nota:** los postres se leen de la primera columna de días; si el primer día está inactivo (feriado), conviene llenar igual los postres en esa columna.
- **Colaboradores**: `NOMBRE | SECTOR`. El SECTOR define, **desde la planilla y sin tocar código**, cómo elige sucursal cada persona:
  - **Sin barra** (`PLANTA`, `JURAMENTO`, `TAVELLA`, …) → **sucursal fija**: no elige, se le asigna sola.
  - **Con barra** (`PLANTA / JURAMENTO`, `PLANTA / JURAMENTO / TAVELLA`, …) → **rotativo semanal**: elige **una** sucursal al inicio, para toda la semana. Soporta cualquier cantidad de sucursales.
  - **Con barra + `(diario)`** (`PLANTA / JURAMENTO (diario)`, `PLANTA / JURAMENTO / TAVELLA (diario)`) → **rotativo por día**: elige la sucursal en **cada día**, junto con el menú y el postre. Sirve para quien retira en distinta sucursal según el día.
  - El marcador `(diario)` se detecta de forma tolerante (con/sin espacio antes del paréntesis, mayúsculas/minúsculas). Se le quita para leer la lista de sucursales. Agregar/quitar `(diario)` en el SECTOR cambia el modo de esa persona sin tocar nada más.
  - Las sucursales de entrega **no están hardcodeadas**: salen de los SECTOR de la planilla (expandiendo los combinados) y de las que ya eligieron quienes pidieron. Si mañana aparece una cuarta sucursal, funciona sola.
- **Config**: `CLAVE | VALOR`. Claves:
  - `semana` (texto, ej. "Semana del 16 al 19 de junio")
  - `abre` y `cierra` (fecha-hora; soporta formato `yyyy-MM-dd HH:mm` y `dd/MM/yyyy HH:mm:ss`)
  - `dias_activos` (ej. `MAR,MIE,JUE,VIE` para saltear feriados; vacío = todos)

**Pestañas que crea/gestiona el Apps Script (no tocar a mano):**
- **Pedidos**: encabezado `ACTUALIZADO | NOMBRE | SECTOR | VECES | LUN_M | LUN_P | LUN_OBS | MAR_M | MAR_P | MAR_OBS | ... | VIE_M | VIE_P | VIE_OBS | SUCS` (**20 columnas**). Un registro por persona (upsert por nombre; gana el último envío). `VECES` cuenta cuántas veces se envió (>1 = modificado). Un día con "No pido" se guarda como `NO_PIDE` en la columna `_M`. Las observaciones por día van en las columnas `_OBS`.
  - **`SUCS`** (última columna): sucursal por día de los rotativos **`(diario)`**. Formato compacto `LUN=PLANTA;MAR=JURAMENTO;...` (solo los días con sucursal elegida). Los fijos y semanales la dejan vacía (su sucursal va en `SECTOR`). El Panel y la planilla impresa agrupan a cada diario en la sucursal que eligió **ese día**.
  - Esta columna se agregó **al final**, así que no mueve las columnas por día: los pedidos ya cargados no se rompen y no hace falta regenerar la pestaña. Se completa sola cuando el primer diario pide (opcional: escribir `SUCS` en la celda de encabezado para dejarlo prolijo).
- **Ausentes**: `NOMBRE` (colaboradores marcados ausentes desde el panel; ausencia de toda la semana).
- **Pedidos dd-MM-yyyy HH.mm**: copias de respaldo que genera el botón "Empezar semana nueva" antes de limpiar.

> ADVERTENCIA al cambiar la estructura de `Pedidos`: la pestaña solo se crea con el encabezado nuevo **si no existe**. Si ya existe con un formato viejo y **cambiás o insertás columnas en el medio**, hay que **borrarla a mano** (o usar "Empezar semana nueva" tras re-publicar el script) para que se regenere; si no, los datos se leen desalineados.
> **Excepción — agregar una columna al final** (como se hizo con `SUCS`): no desalinea nada, así que **no** hace falta regenerar. Las filas viejas quedan con esa celda vacía y se leen igual; la columna se completa sola cuando se escribe el primer dato (opcionalmente se agrega el encabezado a mano para dejarlo prolijo). Preferí siempre esta vía cuando sea posible.

---

## Lógica clave del index.html

**Constantes de configuración (arriba del `<script>`):**
- `API_URL`: URL del Web App de Apps Script.
- `LOGO`: `"logo.png"`.
> Los **valores** de las tres claves de abajo no se escriben en este README a proposito: el repo es publico. Estan en las constantes correspondientes, arriba del `<script>` de `index.html`.
- `CLAVE_WARA`: clave del Panel Wara.
- `CLAVE_PRUEBA`: interruptor de prueba por URL. `?modo=abierto-<CLAVE_PRUEBA>` fuerza abierto; `?modo=cerrado-<CLAVE_PRUEBA>` fuerza cerrado. Vaciar para desactivar en produccion real.
- `CLAVE_ADMIN`: herramientas solo de Lucas. Si la URL trae `?gestion=<CLAVE_ADMIN>`, se setea `ESLUCAS=true` y aparece el boton "Empezar semana nueva" en el panel (Wara, con el link normal, no lo ve).
- `SESION_WARA` (`"wara_panel_sesion"`): clave de `localStorage` que recuerda que el Panel Wara ya se desbloqueo, para no pedir la clave en cada recarga. `waraUnlocked` se inicializa leyendo esta clave (con `try/catch`; solo funciona en Vercel, no como artifact). `tryWara()` la guarda al acertar la clave; `cerrarSesionWara()` la borra y vuelve a pedir la clave.

**Datos en memoria:** `DIAS` (dias con sus platos), `CATS` (categorias de menu), `POSTRES`, `COLABS` (`{n, s}`), `SECTORES`, `PEDIDOS`, `AUSENTES`. En modo real se sobreescriben con lo que devuelve `API_URL`.

**Ventana de pedidos:** `ABRE`/`CIERRA` (fecha-hora desde Config, parseadas con `parseArg`), `ventanaReal()` compara con la hora de Argentina (`ahoraArg()`). `MODO` = `auto` (segun reloj) / `abierto` / `cerrado`. `estaAbierto()` resuelve el estado.

**Dias activos / feriados:** `DIAS_ACTIVOS` filtra `DIAS` para saltear dias no laborables.

**Vistas (`view`):** `welcome`, `dia0..N`, `resumen`, `listo`, `mipedido`, y el panel admin (`tab==='admin'`). `render()` enruta segun `view` y `tab`.
- `renderWelcome`: elegir nombre + sucursal (fija o eleccion si rotativo). Si ya pidio, muestra aviso con resumen y precarga para modificar (last-write-wins). Link "Ver/descargar mi pedido".
- `renderDia`: por dia, elegir menu, postre (o "Sin postre"), opcion "No pido este dia", y campo **Observaciones** opcional.
- `renderResumen`: revision final antes de enviar (con nombres de menu/postre y observaciones).
- `renderListo`: confirmacion + resumen + boton **Descargar mi pedido** + cargar pedido de otro companero.
- `renderMiPedido`: ver/descargar el pedido propio eligiendo el nombre (funciona tambien con la toma cerrada).
- `renderCerrado`: pantalla de "pedidos cerrados" con acceso a "Ver mi pedido".
- `renderAdmin` (Panel Wara): KPIs (ya pidieron / faltan), recordatorio de quienes faltan (copiable a WhatsApp), ausencias, **pedidos modificados** (con hora), totales "A cocinar" por dia (menu + postre con nombres), lista de entrega por sucursal con observaciones, planilla imprimible, boton **🔄 Actualizar** con hora de ultima actualizacion, boton **🔓 Cerrar sesion**, y -solo si `ESLUCAS`- boton "Empezar semana nueva".

**Sesion y auto-refresco del Panel Wara (solo frontend, corre en Vercel):**
- `waraUnlocked` se recuerda en `localStorage` (clave `SESION_WARA`), asi una recarga no vuelve a pedir la clave. `cerrarSesionWara()` borra ese recuerdo; `bloquearPanel()` solo bloquea en la sesion actual (la clave sigue recordada).
- Auto-refresco cada 60 s mientras el panel esta abierto: `iniciarAutoRefresco()` / `detenerAutoRefresco()` manejan un unico `setInterval` (`panelTimer`), reusando `refrescarPedidos()` via `refrescarPanel()` (recarga pedidos, marca la hora en `ultimaActualizacion` con `horaActual()` y redibuja sin recargar la pagina). Se arranca en `renderAdmin()` y se detiene al cambiar a la vista de pedidos, bloquear o cerrar sesion (sin dejar intervalos duplicados).
- `actualizarAhora()`: lo dispara el boton "🔄 Actualizar" para forzar la recarga en el momento. Tanto el auto-refresco como este boton actualizan la hora mostrada ("Actualizado HH:MM").

**Sucursales (rotativos y modo diario):**
- `sucursalesDe(txt)` saca el marcador `(diario)` y separa por `/` (tolera espacios). `esDiario(sec)` detecta el marcador. `esRotativo(nombre)` = tiene >1 sucursal; `esRotativoDiario(nombre)` = rotativo **y** con `(diario)`. `opcionesSucursal(nombre)` = lista para elegir.
- Modelo en memoria: para un diario, `pedido.sector` queda vacio y la sucursal de cada dia vive en `pedido.dias[k].suc`. Fijos/semanales usan `pedido.sector` como siempre.
- `sucursalPedidoDia(p, dk)` es la clave de la agrupacion de entrega: devuelve `p.dias[dk].suc` (diario) o `p.sector` (fijo/semanal). El Panel (`renderAdmin`) y la planilla (`imprimirPlanilla`) agrupan por esta funcion, no por `p.sector`, asi un diario cae en la sucursal correcta de cada dia. `sucursalesEntrega()` arma la lista de sucursales reales (expande combinados, incluye las ya elegidas). `sectorDePedido(p)` -> etiqueta "sucursal por dia" o la fija, para los resumenes.
- En `renderDia`, si es diario y no marco "No pido", aparece el selector "📍 ¿Donde retiras este dia?" (`pickSuc`), con validacion en `siguienteDia`/`confirmar` (no avanza sin sucursal, igual que el postre).

**Helpers de presentacion:** `menuTxt(m)` -> "Menu 3 - Tradicional"; `postreFull(p)` -> "Postre 1 - Flan con dulce de leche"; `postreTxt(p)` (version corta); `platoDe(dk,m)` (nombre del plato); `resumenCorto(p)` y `resumenCard(p)` (resumenes del pedido con observaciones y sucursal por dia).

**Impresion / descarga:** usan `window.print()` sobre `#printArea`. El tamano de hoja lo fija `hojaImpresion(css)`, que reescribe la regla `@page` del `<style id="hojaimp">` justo antes de imprimir: **legal** para la planilla y **A4** para el comprobante.
- `imprimirPlanilla()`: planilla de entrega para Wara, en **hoja legal vertical** con margen de 8mm. Replica el formato que Wara ya usa en papel: encabezado de hoja en texto (`WARA · Comida saludable · Salta` + `Planilla de entrega — dia — semana`, sin logo), una tabla por sucursal con banda verde (`Sucursal: X · dia · Viandas a entregar: N`), filas alternadas en verde claro, grilla completa y 5 columnas: `# | COLABORADOR | MENU | POSTRE | OBSERVACIONES` (la columna "Entregado" se elimino). Al pie de cada sucursal, la linea de "Recibido por / Firma".
  - **Ningun texto se recorta**: las celdas usan `overflow-wrap:anywhere`, asi que un plato largo baja a un segundo renglon en vez de cortarse. Nunca uses `text-overflow:ellipsis` ni `white-space:nowrap` aca — en una planilla de entrega perder media palabra es perder el dato.
  - **Reparto de hojas:** cada dia arranca en hoja nueva y el objetivo es que el **dia entero entre en UNA hoja**. Se suma el alto estimado de todas sus sucursales (`planAltoBloque()`); si el total entra, van todas juntas. Si no entra, la sucursal mas grande se lleva una hoja y las demas se agrupan en otra — nunca se deja una sucursal de una o dos filas sola ocupando una hoja entera. **No hay sucursales hardcodeadas**: si aparece una cuarta, se acomoda sola.
  - **Clave del alto:** el codigo de menu y el nombre del plato van en el **mismo renglon** (`3 · Tradicional — Pastel de carne`). Ponerlos en renglones separados duplicaba el alto de *todas* las filas y era lo que hacia que la planilla ocupara el doble de hojas. Los anchos de columna estan calibrados para que tanto el Menu como el Postre entren sin partirse: la fila mide lo que mide su celda mas alta, asi que si tocas los anchos y algo se parte, toda la fila crece.
  - Medido con los 41 colaboradores pidiendo los 5 dias (peor caso): **7 hojas**, contra 21 del formato original. Tres de los cinco dias entran completos en una hoja (41 filas incluidas); los otros dos usan dos. Si una sucursal no entrara ni sola en una hoja, se parte repitiendo el encabezado de columnas (`thead` en `table-header-group`).
  - Usa clases propias `.plan-*`, separadas de las `.ptable`/`.pblock` del comprobante, justamente para poder cambiar una sin tocar la otra.
- `descargarComprobante(nombre)`: comprobante individual del colaborador (sigue en A4, con `.ptable`/`.pblock`).

**Acciones POST al Apps Script:** `accion:"pedido"` (guardar/modificar), `accion:"ausente"` (marcar/desmarcar), `accion:"reiniciar"` (empezar semana nueva: respalda y limpia).

---

## Como se edita y se sube

1. Editar `index.html` (frontend) y/o `Apps_Script_Wara.gs` (backend).
2. **Frontend:** push a GitHub -> Vercel actualiza solo. (Subir `logo.png` primero si todavia no esta en el repo.)
3. **Backend (si se toco el Apps Script):** pegar el codigo en el editor de Apps Script (Extensiones -> Apps Script) y **re-publicar**: Implementar -> Administrar implementaciones -> editar (lapiz) -> Version: **Nueva version** -> Implementar. La URL **no cambia**. (No usar el boton "Ejecutar".)

---

## Convenciones y trampas conocidas

- **Acentos:** al cargar CSV en Google Sheets, usar **Archivo -> Importar** (no abrir con Excel, rompe los acentos).
- **Zona horaria:** la planilla debe estar en **America/Buenos Aires** (Archivo -> Configuracion), o las horas de apertura/cierre se desfasan 3 hs.
- **Probar siempre desde la URL de Vercel**, no abriendo el archivo local: el `fetch` a Google falla en `file://` por seguridad del navegador (CORS).
- **El link de prueba escribe en la planilla real.** No hay entorno separado: lo que se carga probando queda guardado. Identificar los pedidos de prueba y limpiar con "Empezar semana nueva" antes del uso real.
- **No usar `localStorage`/`sessionStorage` si el codigo corre como artifact de Claude.** En Vercel (produccion) si funcionan.
- **Costo:** para cobrar el servicio corresponde Vercel plan **Pro** (uso comercial). El plan gratis es solo para la prueba.

---

## Estado actual y pendientes

- **En produccion** con Andreani. Cierre de pedidos configurable desde `Config` (ej. domingo 21:00).
- **Mejoras recientes ya implementadas:** observaciones por dia, codigo+nombre de menu/postre en todos los resumenes, senal de "modificado" en el panel, boton "Empezar semana nueva" (oculto tras `?gestion=<CLAVE_ADMIN>`), ver/descargar pedido propio, manejo de feriados y ventana por fecha-hora exacta, sesion del Panel Wara recordada en `localStorage` (con boton "Cerrar sesion"), auto-refresco del panel cada 60 s, boton "🔄 Actualizar" con hora de ultima actualizacion, aviso de "pedido sin enviar" (`beforeunload` + guardia del boton "atras" con `pushState`/`popstate`), tercera sucursal **TAVELLA** y **sucursales genericas por SECTOR** (fija / rotativo semanal / **rotativo diario** con `(diario)`).
- **Ultimo cambio (rotativo diario):** el SECTOR con `(diario)` deja elegir sucursal por dia; se persiste en la columna **`SUCS`** de la pestaña `Pedidos` (Apps Script re-publicado). Al ser una columna al final, **no** rompe pedidos cargados ni requiere regenerar la pestaña.
- **A futuro (otro proyecto):** variante para **clientes particulares** de Wara (pedido suelto, retiro o delivery), que se disena por separado. Y profesionalizacion bajo la marca **Presiia** (dominio propio, propuesta comercial, Wara como caso de estudio).
