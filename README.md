# SPY 0DTE Tracker

Sistema personal de registro y análisis de trading de opciones 0DTE sobre SPY, operado en paper money vía Schwab/thinkorswim (ejecución) y TradingView (análisis técnico). Incluye una app web standalone y un indicador de TradingView como acompañante.

## Componentes

| Archivo | Qué es |
|---|---|
| `index.html` | App web standalone (sin backend ni base de datos) |
| `script1_OCO_SPY_OVERLAY.pine` | Indicador de TradingView (Pine Script v5), overlay 15 min con checklist de entrada |

## App web (`index.html`)

### Arquitectura

- **Sin backend, sin base de datos.** Todo corre en el navegador.
- **Almacenamiento:** `trades.json` en un repo de GitHub privado, vía GitHub Contents API directo desde el navegador. Si no hay configuración de GitHub, cae automáticamente a `localStorage`.
- **Capturas de TradingView:** se comprimen en el navegador y se guardan como archivos aparte en el repo (`capturas/{tradeId}_{tf}.jpg`); solo la referencia vive en `trades.json`.

### Configuración inicial

1. Abre la app → sección de configuración de GitHub.
2. Genera un Personal Access Token: GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token, con permiso **`repo`** únicamente.
3. Ingresa **usuario (owner)**, **nombre del repo** y **token**.
4. Sin esta configuración, la app sigue funcionando guardando solo en `localStorage` del navegador (no persiste entre dispositivos ni se respalda).

### Secciones

- **Dashboard** — vista general
- **Nuevo Trade** — registro manual o importado de un trade
- **Historial** — lista de trades registrados
- **Checklist** — checklist de entrada 0DTE (5 condiciones, score 0–5)
- **Calculadora OCO** — niveles teóricos de SL/TP y break-even sobre la prima
- **Reportes** — análisis: resumen general, por franja horaria, por mes, por ticker, y **MFE / SL-TP**

### Importar desde TOS

Botón "Importar CSV de TOS (Account Statement)" en Nuevo Trade. Lee la sección **Account Trade History** del export de TOS y prellena ticker, strike, tipo, prima entrada/salida, contratos, hora y P&L. **RSI, BB Width, Score y Delta no vienen en ese archivo** — se completan a mano desde el Data Window de TradingView.

### Checklist de entrada (5 condiciones, score 0–5)

1. EMA 9 > EMA 21 + precio sobre VWAP (15 min) — tendencia + sesgo direccional
2. RSI 35–70 — momentum sin extremos
3. BB Width < 3.5% y sin expansión — IV controlada
4. Entrada antes de 11:30 AM ET
5. No es día de cautela (Lun/Mié/Vie)

### Parámetros de estrategia actuales

| Parámetro | Valor |
|---|---|
| Capital base | $2,800 |
| Riesgo por trade | 5% ($140) |
| Stop Loss / Take Profit | 30% / 30% (R:R 1:1) |
| Delta objetivo | 0.45–0.50 |
| Rango de prima | $0.90–$1.79 |
| Contratos por trade | 1 |
| Trades/día máx | 1 |
| Ventana de entrada | 9:45–11:30 AM ET |
| Días de cautela | Lunes, Miércoles (FOMC), Viernes (gamma) |

### MFE (Maximum Favorable Excursion)

Para calibrar si el SL/TP está bien ajustado, cada trade cerrado (sobre todo pérdidas) puede documentar su MFE: el mejor precio que alcanzó a favor antes de revertir.

**Cómo llenarlo** (modal Editar → sección "MFE — Maximum Favorable Excursion"):
1. **Precio SPY al entrar** — lee el Data Window de TradingView (1 min) con el cursor en la vela de entrada.
2. **Precio SPY en MFE** — ubica visualmente el punto más favorable entre la entrada y el cierre, y lee su Data Window.
3. **Delta en punto MFE** — el delta estimado en ese mismo punto (columna "Delta CALL/PUT estimado" del Data Window).

La app calcula automáticamente:
- **Prima MFE estimada** = `primaEntrada + (movimiento SPY a favor) × delta` (aproximación lineal — no considera gamma ni theta)
- **% de camino a TP** = qué tan cerca estuvo esa prima estimada del TP objetivo (30% por defecto)

Estos datos alimentan la pestaña **Reportes → MFE / SL-TP**, que muestra el % promedio de camino a TP en pérdidas y una tabla ordenable para identificar qué tan ajustado (o no) está el SL/TP actual.

> **Limitación conocida:** este cálculo asume delta constante entre la entrada y el punto de MFE. En la práctica el delta cambia con el movimiento del subyacente (gamma) y la prima decae con el tiempo incluso sin movimiento (theta), especialmente relevante en 0DTE. Es una referencia de calibración, no un valor exacto de ejecución.

### Movimiento de SPY necesario (Calculadora OCO)

Tarjeta nueva en la Calculadora: dado el SL y el TP en % de la prima, estima cuántos puntos debe moverse SPY para tocarlos. La prima no se mueve 1:1 con SPY, así que usa la aproximación de segundo orden `cambio de prima ≈ delta·m + ½·gamma·m² − decaimiento` y la resuelve al revés.

- **Obligatorio:** delta (sincronizado con el slider de "Filtro delta"). **Opcionales:** gamma y theta (como los muestra la cadena de TOS), minutos a mantener el trade, costos de ida y vuelta (% de la prima), tipo CALL/PUT, precio de SPY y ATR de 15 min.
- **Tres lecturas** según los datos que des: solo delta, delta + gamma, y delta + gamma + theta. Cada una muestra los puntos para el TP y para el SL, y la relación TP ÷ SL. Con SPY actual muestra los niveles de precio; con ATR, los múltiplos de ATR.
- **Theta:** TOS lo da por día; se reparte linealmente entre los minutos que faltan al cierre (se precarga con la hora de Nueva York). En la práctica el decaimiento se acelera hacia el cierre.
- **Win rate de equilibrio** con y sin costos, comparado con tu win rate real de la vista activa (Todos/Paper/Real) y la expectativa teórica por contrato.
- **Límites:** vale para movimientos chicos (~1.5 pts o menos) y asume delta, gamma y theta constantes. Es un orden de magnitud, no un precio de ejecución. Si cambias el SL/TP, tu win rate también cambia.

### Cuenta: Paper vs Real

Cada trade guarda `modo` (`paper` o `real`). Los trades anteriores sin ese campo se tratan como `paper`.

- **Nuevo Trade:** selector PAPER MONEY / DINERO REAL arriba del formulario (recuerda la última elección). En modo real el formulario cambia de borde y el botón dice "Guardar trade REAL". Elígelo **antes** de importar el CSV de TOS: el archivo no indica de qué cuenta viene.
- **Filtro global Todos / Paper / Real** en la barra superior: afecta dashboard, capital del sidebar, historial y reportes. El capital de cada cuenta parte de `BUDGET`.
- **Historial y dashboard:** badge PAPER / REAL por trade. La cuenta se puede corregir desde Editar.
- **Reportes → Paper vs Real:** compara ambas cuentas lado a lado (ignora el filtro global): win rate, P&L, ganancia/pérdida promedio, expectativa, profit factor, tipo de cierre, % real de ganancia en TP y pérdida en SL (objetivo ±30%), adherencia (score ≥4), trades en días de cautela, días con más de 1 trade y slippage de entrada. Incluye curva de capital por número de trade.
- **Prima de referencia (opcional):** la prima (mid/ask) que veías al decidir. Con ella se mide el slippage de entrada: `prima ejecutada − referencia` (positivo = pagaste más).
- El export CSV incluye las columnas `Cuenta` y `Prima referencia`.

### Estrategia (panel de referencia)

Botón **📘 Estrategia vN** en la barra superior: abre un panel lateral desde cualquier pantalla, sin perder el formulario que tengas abierto.

- **Reglas:** parámetros, checklist de entrada, reglas de lectura, disciplina, patrones de falla y pendientes por validar. Cada regla tiene un **estado** (vigente, en validación o descartada) y un campo **dónde se aplica** (script, app, script + app o solo manual). El panel cuenta cuántas reglas activas dependen solo de ti.
- **Historial de cambios:** cada edición o alta pide un **motivo** (obligatorio) y, opcionalmente, la evidencia. Se registra fecha, regla y el antes → después. Es de solo agregar: las reglas no se borran, se marcan como descartadas.
- **Versión:** cada cambio sube la versión de la estrategia. Cada trade nuevo guarda `versionEstrategia`, y **Reportes → Por versión** compara el rendimiento por versión junto al cambio que la originó. Los trades anteriores aparecen como "Sin versión".
- **Almacenamiento:** `estrategia.json` en el mismo repo de GitHub (mismo token), con respaldo en `localStorage`. Al iniciar se adopta la versión más nueva entre la local y la remota.
- **Es documentación:** editar una regla en el panel no cambia el script Pine ni el checklist de la app; esos se editan por separado. El campo "dónde se aplica" sirve para no perder de vista esa diferencia.

### Otros campos por trade

- `esPrueba` — marca un trade como prueba consciente (fuera de estrategia), para excluirlo de las métricas sin perder el registro en Historial.
- `checklistDetalle` — array estructurado de qué condiciones pasaron/fallaron al guardar, generado automáticamente.
- Capturas multi-timeframe (15 min para contexto de setup, 1 min para precisión de entrada/salida), adjuntables solo desde el modal de Editar (post-trade).

## Indicador de TradingView (`script1_OCO_SPY_OVERLAY.pine`)

Overlay para el gráfico de 15 min de SPY. Calcula EMA 9/21, VWAP, RSI 14, BB Width, ventana horaria y día de cautela, y los combina en una tabla de score 0–5.

### Uso correcto — importante

- **La tabla de checklist en pantalla muestra el estado en vivo del momento de la captura**, no el histórico de la vela donde esté el cursor.
- **Para revisión histórica de cualquier trade pasado, usar siempre el Data Window (Alt+D)** con el cursor parado en la vela exacta — nunca la tabla visual.
- Series ocultas disponibles solo vía Data Window: RSI 14, BB Width %, Score checklist, Delta CALL/PUT estimado, y las medidas de régimen de abajo.

### Medidas de régimen (chop vs tendencia)

Seis series ocultas, pensadas para el gráfico de 15 min. **No clasifican nada**: son datos crudos para que los umbrales salgan de tus trades reales, no de reglas ajenas. Usan solo velas de hoy (en la ventana 9:45–11:30 hay muy pocas), salvo el ATR diario.

| Serie en el Data Window | Qué mide | Cuándo aparece | Cómo leerla |
|---|---|---|---|
| `ATR pts` | Rango típico de la vela del gráfico, en puntos de SPY | Siempre | Se teclea en el campo ATR de la calculadora de movimiento necesario |
| `ATR diario pts` | ATR de 14 días hasta ayer | Siempre | Base para normalizar las dos medidas siguientes |
| `Gap apertura / ATR diario` | Tamaño del gap de hoy frente al rango diario normal | Todo el día | Mayor = apertura con gap grande |
| `Rango apertura / ATR diario` | Rango de los primeros 30 min frente al rango diario normal | Desde la vela de las 10:00 | Mayor = apertura amplia |
| `Cruces VWAP hoy` | Cuántas veces cruzó el precio el VWAP desde la apertura | Desde la 2.ª vela | Más cruces = más chop |
| `Eficiencia precio 0-1` | Movimiento neto ÷ recorrido total de las últimas 4 velas de hoy | Desde la vela de las 10:30 | Cerca de 1 = direccional; cerca de 0 = va y viene |

**Límites:** a las 9:45 y 10:00 todavía no hay eficiencia (necesita 4 velas de hoy), así que la clasificación más temprana es poco fiable. Con datos de horario extendido (ETH) activado, el gap puede incluir información de la preapertura. Como siempre, leer del Data Window con el cursor en la vela exacta, no de la tabla en vivo.

**Nota de verificación:** la lógica de las medidas se probó portada a Python con velas sintéticas, pero el script no se ha compilado en TradingView desde este entorno. Si el editor marca algún error al pegarlo, avísame con el mensaje exacto.

### Señales en el chart

- Flechas de cruce EMA9/21 (`EMA Bull` / `EMA Bear`) — se disparan solo con el cruce, independientes del score compuesto del checklist.
- Labels "CALL"/"PUT" — señal compuesta (cruce + VWAP + RSI + IV + ventana horaria).
- Triángulos pequeños "CALL cont"/"PUT cont" — continuación de tendencia ya establecida.

## Patrones de falla identificados (análisis de julio 2026)

De la revisión de trades reales con Data Window:

- **Patrón A** — conflicto direccional entre 1 min y 15 min (decisión tomada en el timeframe equivocado).
- **Patrón B** — RSI cerca de límite y/o BB Width expandiendo en 15 min, no siempre capturado si se lee el score del 1 min en vez del 15 min.
- **Patrón C** — EMA9/21 y VWAP en conflicto entre sí dentro del mismo timeframe de 15 min.

Regla de decisión recomendada: **verificar primero el score real de 15 min (vía Data Window), usar el 1 min solo para confirmar** — si no hay coincidencia entre ambos, no operar y esperar otra ventana.

## Estado del proyecto

La infraestructura de registro y tracking está funcionalmente completa. El foco actual es refinamiento de estrategia con datos reales, no desarrollo de la app.
