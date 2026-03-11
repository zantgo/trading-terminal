# Guía de Configuración y Variables de Entorno

El **Trading Terminal** utiliza un enfoque de configuración jerárquico y estructurado mediante diccionarios. Esto permite una clara separación entre credenciales sensibles, parámetros estáticos de la aplicación, y variables estratégicas que pueden ser modificadas en tiempo real (*Hot Reloading*).

---

## 1. Gestión de Credenciales (`.env`)

Por motivos de seguridad, ninguna clave API se hardcodea en el repositorio. El bot exige un archivo `.env` en la raíz del proyecto.

Dada la arquitectura de **Aislamiento de Cuentas**, el sistema requiere credenciales independientes para cada subcuenta. Al arrancar, el módulo `config.py` ejecuta la función `_load_and_validate_uids_and_keys()` que valida la integridad de este archivo; si falta alguna variable, el proceso realiza un `sys.exit(1)` (Fail-Fast) antes de iniciar la aplicación.

### Estructura Requerida:
*   **Cuenta Principal (`MAIN`):** Requiere permisos estrictamente limitados a **Transferencias** (Asset -> Transfer). El bot usa esto para el enrutamiento interno de ganancias.
*   **Subcuentas (`LONGS`, `SHORTS`, `PROFIT`):** Requieren permisos de **Trading Unificado (Contratos)**.
*   **UIDs:** Identificadores únicos de las subcuentas utilizados por la API de transferencia universal de Bybit.

```env
# Ejemplo de .env
BYBIT_MAIN_API_KEY="xxx"
BYBIT_MAIN_API_SECRET="yyy"

BYBIT_LONGS_UID=1234567
BYBIT_LONGS_API_KEY="xxx"
BYBIT_LONGS_API_SECRET="yyy"
# ... (Misma estructura para SHORTS y PROFIT)
```

---

## 2. Jerarquía de Configuración (`config.py`)

El archivo `config.py` ha sido refactorizado (v6.0) para agrupar los parámetros en diccionarios lógicos. Cada diccionario tiene un ciclo de vida diferente dentro de la memoria de la aplicación.

### A. `BOT_CONFIG` (Estado Global de la Aplicación)
Contiene la configuración de alto nivel. Estos valores son inyectados al arrancar y definen el modo de ejecución del bot.
*   **`PAPER_TRADING_MODE`**: Si es `True`, la capa de red (`PositionExecutor` y `TransferExecutor`) omite la llamada HTTP REST al exchange y simula una ejecución exitosa. Ideal para pruebas de lógica de la TUI.
*   **`UNIVERSAL_TESTNET_MODE`**: Cambia los endpoints base de Pybit hacia las URLs de Testnet de Bybit.
*   **`TICKER`**: Define el símbolo a operar (ej. `BTCUSDT`). La TUI permite modificar esto, invocando al `BotController.validate_and_update_ticker_symbol()` para asegurar que el activo existe en el exchange antes de aplicarlo.

### B. `SESSION_CONFIG` (Parámetros Estratégicos Globales)
Define cómo el bot "percibe" e interactúa con el mercado en la sesión actual. **Estos valores pueden modificarse "en caliente" a través de la TUI.**
*   **`TA` (Análisis Técnico):** Ventanas lógicas para la Media Móvil (`EMA_WINDOW`) y el cálculo de Momentum (`WEIGHTED_INC_WINDOW` / `WEIGHTED_DEC_WINDOW`).
*   **`SIGNAL`:** Umbrales porcentuales estocásticos para disparar señales de compra/venta (`PRICE_CHANGE_BUY_PERCENTAGE`, `WEIGHTED_DECREMENT_THRESHOLD`).
*   **`PROFIT`:** Constantes para el cálculo matemático de comisiones y *slippage* simulado, garantizando que el cálculo de ROI interno refleje con alta precisión el PNL real de la cuenta.

### C. `OPERATION_DEFAULTS` (Plantillas de Entidades)
Este diccionario actúa estrictamente como un **Factory Template**.
Cuando el usuario crea una nueva estrategia (ej. `[1] Configurar e Iniciar Nueva Operación` en la TUI), el `OperationManager` lee estos valores predeterminados para inicializar el objeto `Operacion` y crear el array de `LogicalPositions`. Una vez instanciada la Operación, modificar este diccionario no afecta a la estrategia en curso.

---

## 3. Mecanismo de Hot-Reloading (Recarga en Caliente)

El bot permite ajustar la agresividad de la estrategia técnica sin necesidad de detener el bucle de trading, apagar el hilo del websocket o liquidar posiciones abiertas.

Cuando se guardan cambios en el `Editor de Configuración de Sesión` (TUI), el diccionario modificado se pasa a `sm_api.update_session_parameters()`.

**Implementación en `SessionManager`:**
El gestor mantiene un `Set` (`STRATEGY_AFFECTING_KEYS`) con las claves críticas.
```python
# Referencia: core/strategy/sm/_manager.py
if strategy_needs_reset or 'TICKER_INTERVAL_SECONDS' in changed_keys:
    if strategy_needs_reset:
        self._memory_logger.log("SM: Cambios en estrategia detectados. Reconstruyendo componentes...", "WARN")
        self._build_strategy_components() # Reinstancia TAManager y SignalGenerator
    
    # Reinicia el hilo del Ticker limpiamente
    self.stop()
    self.start()
```
Si se modifica la ventana de la EMA, el `SessionManager` destruye la instancia anterior del `TAManager`, vacía el DataFrame histórico en memoria y reinicia el Ticker para comenzar a construir el nuevo contexto matemático, todo de forma asíncrona.

---

## 4. Fallbacks de Precisión del Instrumento (`PRECISION_FALLBACKS`)

Para enviar órdenes válidas a Bybit, las cantidades deben respetar el `qtyStep` (paso de cantidad) y la precisión de precios específicos del activo.

La capa de Exchange (`BybitAdapter.get_instrument_info()`) descarga y almacena en caché estos datos automáticamente. Sin embargo, para hacer el bot altamente resiliente contra timeouts de la API en el arranque, el sistema cuenta con el diccionario `PRECISION_FALLBACKS`.

```python
PRECISION_FALLBACKS = {
    "QTY_PRECISION": 3,
    "MIN_ORDER_QTY": 0.001,
    "PRICE_PRECISION": 4,
    "MAINTENANCE_MARGIN_RATE": 0.005 # MMR Base (ej. 0.5% para BTC)
}
```
*   **Manejo Interno:** El módulo `_helpers.py` (`calculate_and_round_quantity`) utiliza la clase `Decimal` nativa de Python para aplicar un redondeo estricto `ROUND_DOWN` basado en la precisión del instrumento, evitando errores de API `130021 (Invalid qty)`. Si el adaptador no logra obtener los datos reales, inyectará temporalmente estos Fallbacks para que el sistema no colapse.
