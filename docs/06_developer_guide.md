# Guía del Desarrollador y Extensibilidad

El **Trading Terminal** fue diseñado pensando en la escalabilidad a largo plazo. Gracias a la aplicación de *Clean Architecture*, es posible extender las capacidades del bot (añadir nuevos exchanges, nuevos indicadores técnicos o nuevas pantallas en la TUI) sin necesidad de reescribir ni tocar la lógica de negocio central (Core).

Este documento establece los lineamientos y "contratos" que deben seguirse para modificar o extender el código fuente.

---

## 1. Reglas y Convenciones del Código

Para mantener la base de código mantenible y libre de dependencias circulares, todo desarrollo nuevo debe adherirse a las siguientes directrices:

### 1.1. Uso Estricto de Fachadas (`_api.py`)
Si el módulo A necesita comunicarse con el módulo B, **jamás debe importar la clase gestora directamente**. Toda la comunicación entre sub-paquetes debe realizarse a través de sus respectivos archivos `_api.py` (Patrón Facade).
*   **Correcto:** `from core.strategy.om import api as om_api; om_api.pausar_operacion(...)`
*   **Incorrecto:** `from core.strategy.om._manager import OperationManager; OperationManager.pausar_operacion(...)`

### 1.2. Módulos y Métodos Privados
Cualquier archivo, variable o método que comience con un guion bajo `_` (ej. `_private_logic.py`, `_calculate_weighted_moving_average`) está estrictamente encapsulado. Su uso fuera de su propio paquete o clase está prohibido. Esto garantiza que la interfaz pública de cada componente se mantenga reducida y controlada.

### 1.3. Inyección de Dependencias
No uses `import` para traer módulos funcionales globales (como configuraciones o utilidades) dentro del `__init__` de las clases *Core*. Utiliza el diccionario `dependencies` que es inyectado desde el `runner/_initializer.py`. Esto es vital para poder inyectar *Mocks* durante la creación de Tests Unitarios en el futuro.

---

## 2. Extensibilidad: Cómo añadir un nuevo Exchange (Ej. Binance)

El bot está desacoplado de Bybit mediante un Patrón Adaptador (`Adapter`). Para hacer que el bot opere en Binance, OKX o Bitget, **NO debes modificar nada en la carpeta `core/strategy`**. Solo debes crear un nuevo adaptador.

### Pasos para crear `BinanceAdapter`:

1.  **Crear el archivo:** Crea `core/exchange/_binance_adapter.py`.
2.  **Heredar la Interfaz:** La nueva clase debe heredar de `AbstractExchange` (ubicada en `core/exchange/_interface.py`).
3.  **Implementar los Contratos (Métodos Abstractos):** Debes escribir la lógica específica de la API de Binance para cada método (ej. `place_order`, `get_positions`, `transfer_funds`).
4.  **Devolver Modelos Estándar:** Esta es la clave del patrón. Tus métodos deben mapear el JSON nativo de Binance a las `dataclasses` universales definidas en `core/exchange/_models.py` (`StandardOrder`, `StandardPosition`, `StandardTicker`, etc.). De esta forma, el `EventProcessor` no notará la diferencia.
5.  **Registrar el Adaptador:** Ve a `runner/_initializer.py` y cambia la inyección de dependencias:
    ```python
    # En runner/_initializer.py
    from core.exchange._binance_adapter import BinanceAdapter
    dependencies["ExchangeAdapter"] = BinanceAdapter
    ```

---

## 3. Extensibilidad: Cómo añadir un nuevo Indicador Técnico (TA)

El motor de Análisis Técnico es puro y no mantiene estado, operando sobre DataFrames de Pandas. Para agregar un nuevo indicador (ej. RSI o MACD):

1.  **Añadir el Cálculo Matemático:**
    Abre `core/strategy/ta/_calculator.py`. Añade la función matemática pura (ej. `_calculate_rsi(series, window)`).
2.  **Actualizar la Salida:**
    En el mismo archivo, dentro de la función pública `calculate_all_indicators(raw_df)`, añade el cálculo de tu nuevo indicador al diccionario `latest_indicators` que se devuelve.
3.  **Actualizar el Data Handler:**
    Abre `core/strategy/signal/_data_handler.py`. Actualiza `extract_indicator_values` y `build_signal_dict` para que el nuevo indicador pueda ser procesado y formateado para los logs de señales.

---

## 4. Extensibilidad: Modificar Reglas de Generación de Señales

Si deseas cambiar la estrategia de entrada (actualmente basada en EMA y WMA):

1.  **Modificar las Condiciones Puras:**
    Abre `core/strategy/signal/_rules.py`. Aquí se encuentran las funciones booleanas `check_buy_condition` y `check_sell_condition`.
2.  **Inyectar los nuevos parámetros:**
    Modifica estas funciones para que acepten tu nuevo indicador (ej. RSI) e implementa la lógica de evaluación (ej. `rsi < 30`).
3.  **Añadir Parámetros de Configuración:**
    Si tu nueva regla requiere umbrales configurables por el usuario, añádelos al diccionario `SESSION_CONFIG["SIGNAL"]` en `config.py` y expónlos en el editor de la TUI (`core/menu/screens/_session_config_editor.py`).

---

## 5. Modificación de la Interfaz de Usuario (TUI)

La TUI está basada en `simple-term-menu`. Cada "Pantalla" se comporta como un bucle infinito `while True` que captura el control de la terminal hasta que el usuario selecciona una opción de "Volver" o "Salir" (`break`).

*   **Ruta:** `core/menu/screens/`.
*   **Regla de Oro de la TUI:** Las funciones de la pantalla **solo deben leer y mostrar (renderizar) datos**. Cualquier mutación del estado del sistema o ejecución de acciones críticas debe delegarse a los `_api.py` de los Managers (`bc_api`, `om_api`, `pm_api`). La TUI jamás debe importar la librería de Bybit ni ejecutar peticiones HTTP directamente.
