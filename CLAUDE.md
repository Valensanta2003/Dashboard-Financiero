# Radar de rupturas — contexto del proyecto

Este archivo es el contexto que Claude Code tiene que leer antes de tocar nada. Resume lo que hay, cómo opera el usuario, qué decidimos y hacia dónde vamos.

## 1. Qué es

Un screener/dashboard de acciones de EE.UU. (más ETFs y cripto) que filtra por tendencia y fuerza relativa, ordena de mejor a peor con un score de 0 a 100 y detecta setups de entrada. Está en español (Argentina).

- Repo del usuario (fork): https://github.com/valensanta2003/Dashboard-Financiero (rama `main`). Usuario de GitHub: `valensanta2003`.
- Publicado: https://valensanta2003.github.io/Dashboard-Financiero/
- Original del que se hizo el fork (de otra persona, NO tocar): https://github.com/francocella2001-bit/Dashboard-Financiero
- Todo vive en **`radar.py`**: descarga datos con `yfinance`, calcula todo y genera un HTML estático único (la plantilla está en la variable `PLANTILLA`, con los datos en JSON en `__DATA__`).
- GitHub Actions (`.github/workflows/radar.yml`; acá hay una copia en `workflow-github/radar.yml` porque la carpeta `.github` no se pudo copiar) lo corre cada hora de 11 a 17 h (Argentina) de lunes a viernes, a las 18:30 y los sábados al mediodía, y lo publica en GitHub Pages.
- Los parámetros editables están en el bloque **CONFIGURACIÓN** al inicio de `radar.py`.

**Primer paso:** esta carpeta tiene una copia de los archivos SIN historial de git. Antes de trabajar, conectá la carpeta al repo (clonalo acá o hacé `git init` + `git remote add origin https://github.com/valensanta2003/Dashboard-Financiero.git` + `git fetch` + `git reset origin/main`). No pises los archivos de `referencias/` ni este `CLAUDE.md`. Commitear y pushear solo cuando el usuario lo pida.

## 2. Cómo opera el usuario

- Sigue a **Mark Minervini** (Trend Template de 8 criterios, VCP, pivots, cortar pérdidas rápido) y a **William O'Neil / CAN SLIM**.
- Usa AVWAP ancladas (earnings, episodic pivots, zonas de alto volumen, máximos) y pivots al barrer las EMAs 10/20/50.
- Opera con su propio cálculo de R (riesgo por trade). **No** hay que agregar gestión de riesgo ni sizing.
- **No** quiere Elliott ni Fibonacci en el dashboard.
- **No** quiere el precio del CEDEAR según CCL (aunque Warren lo tenga).
- Quiere algo **conciso, con pocas pestañas**, "al pie" para analizar y operar rápido. Ante la duda, mejor agregar una columna, un badge o un filtro dentro de una sección existente que crear una sección nueva.

## 3. Lo que ya hace `radar.py` (no romperlo)

**Filtro de entrada.** Universo S&P 500 + Nasdaq 100 + MidCap 400 + ETFs + cripto + `TICKERS_EXTRA`. Requisitos:
- Precio de 10 o más.
- ADR20 de 2% o más.
- 7/8 criterios de Minervini con RS de 70 o más obligatorio. El RS es un percentil del universo con pesos 40/20/20/20 sobre 63/126/189/252 ruedas.
- AVWAP activa (`EXIGIR_AVWAP`). Si no la tiene, igual entra si tiene VCP, compresión o toque de EMA diaria.

**Score de 0 a 100.**
| Bloque | Puntos |
|---|---|
| Tendencia (Minervini) | 20 |
| RS | 25 |
| Compresión (ATR, ancho de Bollinger, volumen) | 35 |
| Setup | 20 |

Se restan 5 puntos por cada bandera mala: rechazo en máximos, ruptura sin volumen, extendida sobre la SMA50, estirada en ADRs sobre la EMA10/20.

**Secciones actuales:** Inicio (régimen de mercado SPY/QQQ + amplitud + VIX, lista de foco, top 3, earnings próximos), Sectores ("calor"), AVWAP, VCP, Compresión, Inside, Episodic y Setups (ruptura de AVWAP, ruptura de compresión, 30 Min Pivot). También tiene la ficha de cada activo.

## 4. Cambios decididos (a implementar)

### 4.0 Universo completo del mercado de EE.UU. (PRIMERA TAREA)
Hoy el universo es solo S&P 500 + Nasdaq 100 + MidCap 400 (unas 1.000 acciones, sacadas de Wikipedia) + ETFs + cripto + `TICKERS_EXTRA` (tickers sueltos agregados a mano: NBIS, CLS, HOOD). Así se pierden las small caps, las IPOs recientes y muchas ADRs (TSM, ASML, GGAL, YPF, VIST…), que es justo donde suelen salir los grandes ganadores de Minervini/O'Neil.

El usuario quiere que **se analicen TODAS las acciones que cotizan en EE.UU.** y que los filtros vayan descartando las que no sirven. Cómo hacerlo:
- **Fuente del listado:** los archivos públicos de NASDAQ Trader (`nasdaqlisted.txt` y `otherlisted.txt` en nasdaqtrader.com/dynamic/SymDir/), que cubren NASDAQ, NYSE, NYSE American y Arca. Incluir acciones comunes y ADRs. Excluir ETFs (ya hay una lista propia), warrants, units, rights, preferidas y test issues (`Test Issue = Y`, sufijos y nombres tipo "Warrant", "Unit", "Right", "Preferred"). Convertir los símbolos al formato de Yahoo (ej. `BRK.B` → `BRK-B`).
- **Embudo de filtros**, en este orden y todo configurable en CONFIGURACIÓN:
  1. Precio de al menos `PRECIO_MINIMO`.
  2. Filtros de tamaño y liquidez (decididos por el usuario; promedios de 50 ruedas; cada uno configurable y desactivable con 0):
     - `MARKET_CAP_MIN = 1_000_000_000` (USD 1.000 millones).
     - `VOLUMEN_DOLARES_MIN = 20_000_000` (promedio diario de precio × volumen ≥ USD 20 millones).
     - `VOLUMEN_ACCIONES_MIN = 500_000` (promedio diario ≥ 500 mil acciones). Ojo: con este filtro quedan afuera acciones caras y líquidas (MELI, BKNG, AZO, FICO operan menos de 500 mil acciones por día, pero cientos de millones de dólares). Ver la sección 7.
     - El market cap se obtiene de `yfinance` (`fast_info` / `info`) y va al mismo caché que sector e industria.
  3. ADR.
  4. Minervini.
  5. El resto de los criterios que ya existen.
- Mostrar en el dashboard cuántas acciones quedan en cada paso del embudo (ej.: 5.800 → 2.400 → 900 → 180 → 78).
- **Sector e industria:** Wikipedia ya no alcanza. Obtenerlos de `yfinance` (`info["sector"]`, `info["industry"]`) guardando un **caché** (ej. `sectores_cache.json` con actions/cache, igual que `earnings_cache.json`) para no consultarlos en cada corrida. Pedir solo los que falten.
- **Rendimiento:** van a ser unos 5.000–6.000 tickers. Descargar en lotes, tolerar fallos y rate limits de Yahoo sin romper la corrida, y subir `timeout-minutes` del workflow (hoy 40) si hace falta. Medir el tiempo de una corrida completa e informárselo al usuario.
- Mantener las opciones `INCLUIR_SP500`, etc., o reemplazarlas por una opción `UNIVERSO = "completo"` / `"indices"`, para poder volver atrás.

### 4.0b Criterios Minervini = los del indicador de TradingView del usuario
El usuario mira en TradingView su indicador **"Minervini Trend Compact v2.3"** (código completo en `referencias/minervini_trend_compact_v2.3.pine`, y captura en `referencias/tradingview_TSM_indicadores.png`). El conteo X/8 del radar tiene que **coincidir con lo que ve ahí**. Hoy NO coincide: `radar.py` usa SMAs, umbrales del 30% y RS por percentil. Hay que reemplazarlo por su definición:
- Medias **EMA** (no SMA): EMA 10, 20, 50, 150, 200.
- c1: precio > EMA150 y > EMA200
- c2: EMA150 > EMA200
- c3: EMA200 subiendo (hoy > hace 22 velas, como `ta.rising(v200, 22)`, que exige subida en cada una de las 22 barras; replicar esa semántica)
- c4: EMA50 > EMA150 y > EMA200
- c5: precio > EMA50
- c6: precio ≥ mínimo de 252 velas × 1,25
- c7: precio ≥ máximo de 252 velas × 0,75
- c8: **línea de RS vs SPY sobre su SMA 50** (close/SPY > SMA50(close/SPY)). No es el percentil.
- El **RS percentil** (RS Rating 1–99) se sigue calculando y mostrando, y pesa en el score (bloque "Fuerza RS"), pero aparte del conteo X/8. **Ya NO es obligatorio** (decisión del usuario): `EXIGIR_RS = False`; ninguna acción queda afuera por tener RS < 70.
- Mínimo para ser candidata: **6 de 8** (decidido por el usuario; su amigo usaba 7). Dejarlo configurable en `MIN_CRITERIOS_MINERVINI = 6`.
- Colores del X/8 como en su indicador: 7–8 verde, 5–6 naranja, menos de 5 rojo.

**Otros datos del indicador que el usuario quiere ver en el radar** (en la ficha y como columnas compactas):
- **Pendiente de cada EMA** (E10, E20, E50, E150, E200) con flecha ▲/▼ y dos intensidades: "inmediata" (hoy vs ayer) y "sostenida" (vs hace 3/5/10/20/22 velas respectivamente). ▲ verde fuerte si sube y es sostenida; ▲ verde claro si solo sube hoy; ▼ rojo claro / rojo fuerte igual.
- **Momentum 35-5** = EMA5 − EMA35 del cierre, en diario (M1D, sostenida vs 5 velas) y semanal (M1W, sostenida vs 4 velas), con la misma lógica de flechas. Ejemplo de uso: "5/8 con momentum descendente".
- **ADR%** de 20 con su escala de color: <1,5 gris; 1,5–2,5 verde claro; 2,5–4,5 verde fuerte; 4,5–6 naranja; >6 rojo.
- Del panel de la captura: Market Cap, % desde el máximo de 52 semanas, **volumen promedio en $**, RS Rating, sector/industria y el ETF del sector con su RS y variación.

**Volumen estilo MarketSmith** (indicador aparte que usa el usuario): barras de volumen coloreadas según suba/baje el cierre, media de 50 del volumen y **volumen de hoy vs. su promedio de 50 en %** (ej. "14,92M +52%"). En el radar: mostrar "Vol vs prom50 %" como columna y usarlo en la ficha. Sumar si es posible el **ratio Up/Down volume de 50 días** (volumen de días alcistas / volumen de días bajistas), que es un clásico de MarketSmith/IBD para ver acumulación vs distribución.

### 4.1 Fundamentals CAN SLIM en el score (no excluyentes)
- Suman **hasta 10 puntos** sobre 100:
  - Aceleración del EPS trimestral interanual: hasta 5 puntos.
  - Aceleración de las ventas trimestrales interanuales: hasta 5 puntos.
  - Si crece fuerte (25% o más) pero sin acelerar, suma la mitad.
- **Nunca excluyen**: si no cumple o falta el dato, suma 0 y se muestra "s/d". Ninguna acción deja de aparecer por esto.
- Para que el total siga en 100, la propuesta es reescalar: Tendencia 18, RS 23, Compresión 32, Setup 17, Fundamentales 10. Que quede configurable en CONFIGURACIÓN.
- Datos de `yfinance`:
  - EPS: `get_earnings_dates` (EPS reportado, estimado y sorpresa, con buena historia).
  - Ventas: `quarterly_income_stmt` (suele traer solo 4 o 5 trimestres, así que la aceleración interanual de ventas va a estar limitada).
  - Agregar caché como el de `earnings_cache.json`, porque es lento y Yahoo limita las consultas.

### 4.2 Cierres semanales apretados (tight closes) dentro de VCP
- **No es una sección nueva.** Es un badge o columna en la tabla de VCP (ej. "3 TC") y suma algunos puntos de setup.
- Criterio inicial: 2 o 3 o más cierres semanales consecutivos dentro de un rango de ~1,5% (configurable), idealmente con volumen bajo y cerca del pivot.
- Si hay cierres apretados sin VCP detectada, la acción igual aparece en VCP con la etiqueta "solo TC".

### 4.3 Menos pestañas
Propuesta para validar con el usuario:
- Juntar Compresión + Inside + Setups en una sola vista "Gatillos".
- Episodic, RSI y MACD semanal pasan a ser filtros (chips) dentro de esa vista, no secciones aparte.

## 5. Referencia de diseño: "Warren Dashboard" (@Warren_Bife en X)

Al usuario le atrae más que el actual y quiere acercarse a ese diseño. Hay capturas y un video en `referencias/`: mirarlos. Ojo: los cuatro pilares del score de Warren son los mismos que ya usa `radar.py` (Tendencia 20, Fuerza RS 25, Contracción 35, Setup estructural 20). La base es la misma; lo que cambia es la presentación y los módulos extra.

**Estética**
- Fondo casi negro, tarjetas con borde sutil y una barrita verde a la izquierda del título de cada tarjeta.
- Títulos en tipografía monoespaciada con letra espaciada.
- Números grandes en verde, ámbar o rojo según el valor.
- Sparklines por fila y logos de las empresas.
- Mobile-first.

**Módulos que tiene Warren** (marcados según si sirven para este proyecto):
- **Régimen de mercado**: medidor semicircular de 0 a 100, desglose por capa (Índices SPY/QQQ x/80, Amplitud x/30, VIX x/15), nuevos máximos/mínimos de 52 semanas y gráfico de "% de activos sobre su EMA200, últimos ~2 meses". Tiene también "exposición sugerida": mostrarla solo como lectura del mercado, sin sizing. ✅
- **Retorno Top 10 del Warren Score**: el retorno de cada acción desde que entró al top, con fecha de ingreso, comparado con el SPY. Es el track record del score y requiere guardar historial entre corridas (por ejemplo un JSON en caché o en una rama `data`). ✅
- **Nuevos ingresos al Top 30**: score, RS, distancia a la EMA200 y sparkline. ✅
- **Cartelera del día**: las que más subieron y más bajaron. ✅ (chico)
- **RSI semanal cruzando su SMA14** y **MACD semanal: cruces** (alcista bajo cero / bajista sobre cero, "cruzó hace X"). Van como filtros dentro de Gatillos. ✅
- **Volumen semanal seco**: caída mínima de 40% y mínimo 2 semanas. Se puede integrar con VCP/TC. ✅
- **Cuadrantes de rotación (tipo RRG)**: burbujas con eje X = RS y eje Y = ΔRS contra la semana anterior, en cuadrantes Mejorando / Liderando / Debilitándose / Rezagados. Tiene filtro por ETF, un "Play" con slider de 16 semanas y un texto automático de "Lectura de rotación". También listas de "Recién a Líderes" y "Aceleración inusual" para ETFs. ✅ Reemplazaría o enriquecería la pestaña Sectores.
- **Valuación fundamental**: tabla con score fundamental, lectura ("Barata + sector líder", "Barata, sector débil", "Posible trampa de valor"), subsector con el RS de su ETF, PER futuro contra el del sector, PEG, ventas interanuales, EPS esperado, upside al precio objetivo, consenso de analistas, calidad /20 y Warren Score. Chips de filtro: Todas las baratas / Sector líder / Sector emergente / Sector débil / Trampas / Todas. 🟡 Es opcional: priorizar lo de CAN SLIM (4.1) y sumar esto después si el usuario lo pide.
- **Ficha de acción** (al tocar una fila o usar el buscador):
  - Encabezado: ranking (#13 de 391), precio y variación.
  - Score con barras por pilar y posición en el rango de 52 semanas.
  - "Gates Stage 2" (precio sobre la EMA200 con pendiente positiva).
  - Penalizaciones (ej. "Distribución activa −15").
  - Evolución del RS en las últimas 16 semanas.
  - Métricas: distancia a SMA50/EMA200, EMA200 semanal, RSI semanal, RS hoy / semana pasada / hace un mes, línea de RS sobre su SMA50.
  - "Vs. sus sectores": rendimiento contra cada ETF relacionado a 1 y 3 meses.
  - **Análisis fundamental**: crecimiento trimestral de EPS y ventas, P/E, % institucional, barras de EPS por trimestre contra el mismo trimestre del año anterior (con un punto si superó lo estimado) y EPS anual con crecimiento compuesto (CANSLIM pide 25%). ✅
  - Precio recomendado del CEDEAR (según CCL y ratio): ❌ **NO incluir**, el usuario no lo quiere.
- Buscador de ticker, filtros y pestañas Acciones / Índices. También tiene un tutorial guiado de 9 pasos (opcional).

## 6. Forma de trabajar
- Hablar en español rioplatense, claro y conciso.
- Antes de cambios grandes de estructura o diseño, proponer y esperar el OK. Los ajustes chicos se hacen directo.
- Todo parámetro nuevo va al bloque CONFIGURACIÓN, con nombre en español.
- Probar `radar.py` localmente (con un universo reducido o un modo demo) antes de dar algo por terminado, y revisar el HTML generado.
- Los datos de Yahoo pueden faltar: nunca romper la corrida por un ticker; mostrar "s/d".

## 7. Decisiones tomadas
- Liquidez (15-10-2026): el usuario eligió filtrar solo por volumen en dólares (USD 20M promedio de 50 ruedas). `VOLUMEN_ACCIONES_MIN = 0` (desactivado), así no quedan afuera MELI, BKNG, AZO o FICO.
- `ta.rising(v200, 22)` en Pine significa "EMA200 de hoy mayor que cada uno de los 22 valores anteriores", no "subió en cada una de las 22 velas". Se replica la semántica de Pine.
