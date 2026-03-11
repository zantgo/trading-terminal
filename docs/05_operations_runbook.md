# Manual de Operaciones y Respuesta a Incidentes (Runbook)

Este documento proporciona las directrices operativas para desplegar, monitorear y mantener el **Trading Terminal** en un entorno de producción (Live Trading). Además, detalla los procedimientos de mitigación y respuesta ante incidentes críticos.

---

## 1. Entorno de Ejecución y Despliegue

A diferencia de los daemons o servicios en segundo plano tradicionales, este bot está acoplado a una **Interfaz de Usuario en Terminal (TUI)** altamente interactiva construida con `simple-term-menu`. 

### 1.1. Requisitos de Despliegue (VPS)
Para ejecutar el bot 24/7 en un servidor remoto (VPS como AWS EC2, DigitalOcean o Linode), **es obligatorio el uso de un multiplexor de terminales** para evitar que el proceso muera al desconectar el cliente SSH.

**Comandos recomendados (`tmux`):**
```bash
# Iniciar una nueva sesión separada
tmux new -s trading_bot

# Activar el entorno virtual e iniciar el bot
source venv/bin/activate
python main.py

# Para salir de la vista sin apagar el bot: Presionar Ctrl+B, luego la tecla 'D' (Detach)

# Para volver a entrar y ver la TUI en el futuro:
tmux attach -t trading_bot
```

---

## 2. Monitoreo y Auditoría (Logging Asíncrono)

Para garantizar que el guardado de datos en disco no introduzca latencia (I/O blocking) en el hilo principal de trading, el sistema utiliza un modelo de **Logging Asíncrono basado en colas (`FileLogManager`)**. 

Los logs se encolan en memoria y un hilo trabajador en segundo plano (Worker Thread) los vuelca al disco en lotes (`batch_size=10`) o por intervalos de tiempo (`flush_interval=30s`).

### 2.1. Archivos Generados
Los datos se almacenan en la carpeta `/logs` usando el formato **JSONL (JSON Lines)**, ideal para ser ingerido por sistemas de análisis de datos (como ELK Stack o scripts de Python/Pandas).

*   `signals_log.jsonl`: Registra cada tick que genera una señal de análisis técnico.
*   `closed_positions.jsonl`: Registra la autopsia financiera (PNL, comisiones, precios) de cada `LogicalPosition` al cerrarse.
*   `open_positions_snapshot.jsonl`: Se sobrescribe continuamente (o al apagar) reflejando el estado de la memoria para posibles recuperaciones.

### 2.2. Resúmenes de Sesión
Ubicados en la carpeta `/results`. Al ordenar un apagado limpio, el `runner/_shutdown.py` genera un archivo de texto humano-legible (ej. `session_summary_20260311_143000.txt`) con la rentabilidad (ROI), duración de la sesión y capital final.

---

## 3. Procedimientos de Emergencia (Incident Response)

El bot está diseñado para ser resiliente, pero en mercados altamente volátiles, la intervención manual puede ser necesaria.

### 3.1. Escenario A: Volatilidad Extrema o Cisne Negro (Black Swan)
Si el mercado colapsa y la estrategia de DCA está absorbiendo demasiadas pérdidas, el operador debe intervenir.

**Acción: Cierre de Pánico (Panic Close)**
1. En la TUI, navegar a **Gestionar Operación (LONG/SHORT)** -> **Gestionar Posiciones Manualmente**.
2. Seleccionar la opción `[*] CIERRE DE PÁNICO`.
3. **¿Qué hace el código?** El `OperationManager` iterará sobre todas las `LogicalPositions` abiertas, solicitará al `PositionExecutor` que envíe órdenes Market del tipo `ReduceOnly` inmediatas, e inyectará la razón `"PANIC_CLOSE_ALL"`. Finalmente, detendrá la operación para evitar reentradas.

### 3.2. Escenario B: Caída de la API o Pérdida de Sincronización
Si la API de Bybit devuelve timeouts (Error 10006) o devuelve listas vacías de posiciones físicas cuando el bot espera que haya órdenes abiertas.

**Mecanismo de Defensa (Heartbeat):**
El `PositionManager` tiene un método `sync_physical_positions(side)`. Si la API falla repetidamente, el bot incrementará un contador interno `_sync_failure_counters`.
Si este contador alcanza el límite configurado en `SESSION_CONFIG["RISK"]["MAX_SYNC_FAILURES"]` (por defecto 10,000 en el archivo `config.py`), el bot asume que **la posición ha sido liquidada físicamente en el exchange** y dispara un `handle_liquidation_event`, cerrando las posiciones lógicas en memoria y transicionando el estado a `DETENIENDO`.

### 3.3. Precaución Operativa: Rate Limiting (Baneo de IP)
El hilo del Ticker (`connection/_ticker.py`) utiliza peticiones REST dentro de un bucle `while`. 
*   **Peligro:** Configurar `TICKER_INTERVAL_SECONDS` en `config.py` con valores inferiores a `1` segundo de manera sostenida puede resultar en un baneo temporal de la IP por parte del WAF (Web Application Firewall) de Bybit por exceso de peticiones HTTP.
*   **Mitigación actual:** Mantener el intervalo en `1.0` o superior para operativas REST. *(Nota para desarrollo futuro: Migrar el Ticker a un Websocket stream para evitar este límite).*

---

## 4. Troubleshooting: Códigos de Error Comunes de la API

El módulo `core/api/_helpers.py` interpreta los errores comunes de Bybit v5. Si ves estos errores en el "Visor de Logs" de la TUI, esta es su causa:

| Código API                  | Mensaje Interno               | Diagnóstico y Solución                                                                                                                                                         |
| :-------------------------- | :---------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`110007`** o **`180024`** | `ab not enough for new order` | **Fondos insuficientes.** El tamaño de la posición o el apalancamiento requieren más margen del que hay en la subcuenta de trading. Transfiere fondos desde `main`.            |
| **`110043`**                | `Set leverage not modified`   | **Informativo.** El bot intentó fijar el apalancamiento (ej. 10x), pero la cuenta ya estaba en 10x. El sistema ignora el error de forma segura.                                |
| **`110001`**                | `Order/Position not found`    | Ocurre durante cierres manuales si el exchange ya había liquidado o cerrado la orden (por TP del exchange). El bot asume el cierre lógico y continúa.                          |
| **`130021`**                | `Invalid qty`                 | Ocurre por problemas de redondeo o si la cantidad de la orden es inferior al `MIN_ORDER_QTY` permitido por las reglas del instrumento en Bybit. Revisar `PRECISION_FALLBACKS`. |
|                             |                               |                                                                                                                                                                                |
