# Lógica de Trading, Señales y Gestión de Riesgo

Este documento detalla la implementación matemática y de dominio de la estrategia algorítmica del **Trading Terminal**. El bot no es un simple ejecutor de órdenes; actúa como una máquina de estados compleja que gestiona una estrategia direccional de promediación de costos (DCA) combinada con una matriz de riesgo multinivel.

---

## 1. Modelo de Dominio: Posiciones Lógicas vs Físicas

Para lograr una gestión de riesgo granular (como tener múltiples Stop Loss para distintas entradas en el mismo activo), el bot implementa un patrón de **Desacoplamiento de Posición**.

*   **Posición Física (`PhysicalPosition`):** Es la realidad del Exchange. Bybit, al estar en modo Hedge, solo permite tener UNA posición `LONG` y UNA posición `SHORT` abiertas por activo. Esta entidad agrupa el tamaño total y el precio de entrada promedio.
*   **Posición Lógica (`LogicalPosition`):** Es la abstracción interna del bot. El capital total operativo se divide en "lotes" representados por objetos `LogicalPosition`. 

**Mecánica:**
Al configurar una estrategia, el bot pre-asigna capital a $N$ `LogicalPositions` en estado `PENDIENTE`. Cuando se emite una señal de compra, el `PositionExecutor` envía una orden de mercado y transiciona la primera posición lógica a estado `ABIERTA`, guardando su *entry price* individual. 
Esto permite que, aunque Bybit promedie tu entrada, el bot recuerde el precio exacto al que compraste el Lote 1, Lote 2, etc., pudiendo calcular y ejecutar Trailing Stops y Stop Loss de manera completamente independiente para cada "lote".

```python
# Referencia: core/strategy/entities/__init__.py
@dataclass
class LogicalPosition:
    id: str
    capital_asignado: float
    estado: str = 'PENDIENTE'
    entry_price: Optional[float] = None
    size_contracts: Optional[float] = None
    # Riesgo individual mapeado a este lote específico
    stop_loss_price: Optional[float] = None
    ts_is_active: bool = False
    ts_peak_price: Optional[float] = None
```

---

## 2. Motor de Análisis Técnico y Generación de Señales

La toma de decisiones de entrada al mercado recae sobre los módulos `TAManager` y `SignalGenerator`. Se ejecuta una evaluación completa en cada *tick* válido recibido del websocket/REST.

### 2.1. Momentum Ponderado Personalizado (WMA)
En lugar de depender exclusivamente del RSI o MACD, el bot utiliza un cálculo de **Incremento/Decremento Ponderado**. Mide no solo si el precio subió o bajó, sino la "densidad" de ese movimiento en una ventana de $N$ ticks.

Se calcula aplicando una Media Móvil Ponderada (WMA) sobre un array booleano de ticks de subida (`increment`) o bajada (`decrement`). A los ticks más recientes se les otorga mayor peso matemático mediante un producto punto (`np.dot`):

```python
# Referencia: core/strategy/ta/_calculator.py
def _calculate_weighted_moving_average(series: np.ndarray, window_size: int) -> float:
    weights = np.arange(1, window_size + 1)
    # ... (sanitización de NaNs) ...
    wma = np.dot(series_valid, weights_valid) / np.sum(weights_valid)
    return wma
```

### 2.2. Reglas de Ejecución (`_rules.py`)
Para que se dispare una señal direccional, deben confluir tres validaciones vectoriales: Filtro de Tendencia (EMA), Magnitud del Movimiento (Porcentaje de caída/subida) y Consistencia del Movimiento (WMA de Momentum).

Ejemplo de la condición para **LONG (BUY)**:
1.  **Filtro de Tendencia:** `price < EMA` (Buscamos rebotes o *mean reversion* en tendencias bajistas locales).
2.  **Magnitud:** `dec_pct <= PRICE_CHANGE_BUY_PERCENTAGE` (La caída porcentual debe ser igual o más profunda que el umbral configurado, ej. `-0.05%`).
3.  **Consistencia:** `w_dec >= WEIGHTED_DECREMENT_THRESHOLD` (Debe haber un peso consolidado de caídas recientes).

---

## 3. Estrategia de Promediación (DCA)

Una vez que la primera `LogicalPosition` está `ABIERTA`, las subsecuentes entradas no solo requieren una señal técnica, sino que deben respetar la validación de distanciamiento espacial definida por la estrategia.

El `PositionManager` (`_private_logic.py`) intercepta la señal y valida la `averaging_distance_pct` (distancia de promediación). 

Para posiciones `LONG`, la nueva posición solo se abre si:
$Precio_{actual} \le Precio_{ultima\_entrada} \times \left(1 - \frac{Distancia}{100}\right)$

Esto garantiza la formación de una "Grid" asimétrica dinámica que absorbe caídas en contra de la posición sin agotar el capital de manera prematura.

---

## 4. Matriz de Gestión de Riesgo Multinivel

Este es el núcleo de supervivencia del bot. El riesgo se evalúa en el `EventProcessor` en tiempo real (cada tick), con un orden estricto de prioridad de ejecución para prevenir colisiones.

### Nivel 1: Prevención de Liquidación (Prioridad Máxima)
El bot simula y recalcula el Precio de Liquidación Agregado con cada orden. Si el precio de mercado cruza la liquidación estimada calculada (antes de que el exchange lo haga), el `OperationManager` emite un evento `handle_liquidation_event` y dispara el modo `DETENIENDO` (cierre forzoso a mercado).

### Nivel 2: Riesgo de Nivel de Operación (Global)
Evalúa la salud de toda la "cesta" de `LogicalPositions` de manera conjunta.
*   **Stop Loss / Take Profit por ROI:** Liquidación basada en el Retorno sobre Inversión (Total Equity). Si la operación entera (todos los lotes) cae un `X%` o gana un `Y%`, se cierra todo.
*   **TSL por ROI (Trailing Stop sobre PNL):** Persigue el pico máximo de ROI obtenido por la cuenta. Muy útil para capturar mega-tendencias.
*   **SL Dinámico (Dynamic ROI SL):** Ancla el Stop Loss a una distancia fija del **ROI Realizado** (ganancias ya cerradas y transferidas a Profit). Si la cuenta de ganancias crece, el límite de pérdida se mueve hacia arriba (Break-Even garantizado).
*   **SL/TP por Break-Even:** Calcula dinámicamente el precio de entrada promedio real, deduce las comisiones estimadas, y coloca los umbrales de cierre a una distancia fija desde ese punto de equilibrio.

### Nivel 3: Riesgo de Posición Individual (Local)
Delegado al `PositionManager`. Si la operación general está "sana", se revisa el estado individual de cada lote.
*   **SL Individual:** Cierra solo la `LogicalPosition` perdedora.
*   **TSL Individual:** `ts_peak_price` registra el máximo beneficio que alcanzó un lote particular. Si el precio retrocede la `tsl_distancia_pct` desde ese pico, se cierra ese lote específico capturando la ganancia, mientras que el resto de posiciones (si las hay) continúan abiertas.

```python
# Prioridad de ejecución orquestada en core/strategy/_event_processor.py:
# 1. Heartbeat API Sync (Prevención de desincronización)
# 2. Comprobar Liquidación Estimada Agregada
# 3. Comprobar TSL de Operación (ROI-based)
# 4. Comprobar SL Global (Break-Even, Dynamic ROI)
# 5. Comprobar Riesgos Individuales (PositionManager)
```
