# HTF - High Frequency Trading

Sistema de trading de alta frecuencia desarrollado como proyecto de diplomatura. Incluye un dashboard en tiempo real, ingesta de datos de mercado desde Binance y un motor de trading automatizado que utiliza estrategias basadas en VWAP y momentum.

## Tabla de Contenidos

- [Características](#-características)
- [Arquitectura](#-arquitectura)
- [Requisitos](#-requisitos)
- [Instalación](#-instalación)
- [Configuración](#-configuración)
- [Uso](#-uso)
- [Estructura del Proyecto](#-estructura-del-proyecto)
- [Estrategia de Trading](#-estrategia-de-trading)
- [Dashboard](#-dashboard)
- [Notas Importantes](#-notas-importantes)

## Características

- **Streaming en tiempo real**: Consumo del order book de Binance mediante WebSocket
- **Almacenamiento en MotherDuck**: Persistencia de datos de mercado en DuckDB cloud
- **Trading automatizado**: Motor de trading con estrategias basadas en indicadores técnicos
- **Dashboard interactivo**: Visualización en tiempo real con gráficos y métricas
- **Modo simulado**: Permite probar estrategias sin riesgo antes de operar en real
- **Gestión de posiciones**: Sistema completo de Take Profit y Stop Loss

## Arquitectura

El sistema está compuesto por tres módulos principales que funcionan de forma independiente:

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│  stream.py  │─────▶│  MotherDuck  │◀─────│  trader.py  │
│  (WebSocket)│      │   (DuckDB)   │      │  (Estrategia)│
└─────────────┘      └──────────────┘      └─────────────┘
                              ▲
                              │
                       ┌──────┴──────┐
                       │   app.py    │
                       │  (Flask)    │
                       └─────────────┘
```

### Componentes

1. **`stream.py`**: Ingesta de datos
   - Se conecta al WebSocket de Binance
   - Captura actualizaciones del order book (depth)
   - Persiste datos en MotherDuck cada vez que recibe una actualización

2. **`trader.py`**: Motor de trading
   - Lee datos históricos de MotherDuck
   - Calcula indicadores técnicos (VWAP, Momentum)
   - Evalúa señales de entrada y salida
   - Ejecuta órdenes (simuladas o reales)
   - Registra todas las operaciones

3. **`app.py`**: Dashboard web
   - Servidor Flask que expone endpoints REST
   - Consulta datos de MotherDuck
   - Calcula métricas en tiempo real
   - Sirve el frontend HTML/JavaScript

## Requisitos

- **Python 3.10+**
- **Token de MotherDuck**: Necesario para almacenar y consultar datos
- **Claves de API de Binance** (opcional): Solo si deseas operar en modo real
- **Conexión a Internet**: Para WebSocket de Binance y MotherDuck

## Instalación

### 1. Clonar o descargar el proyecto

```bash
cd htf-binance
```

### 2. Crear entorno virtual

```bash
python -m venv .venv
```

### 3. Activar entorno virtual

**Linux/Mac:**
```bash
source .venv/bin/activate
```

**Windows:**
```bash
.venv\Scripts\activate
```

### 4. Instalar dependencias

```bash
pip install -r requirements.txt
```

### 5. Configurar variables de entorno

```bash
cp env.example .env
```

Edita el archivo `.env` y completa los valores necesarios (ver sección [Configuración](#-configuración)).

## Configuración

Copia `env.example` a `.env` y configura las siguientes variables:

### Variables Requeridas

```env
# MotherDuck (REQUERIDO)
MOTHERDUCK_TOKEN=tu_token_aqui
```

### Variables Opcionales

```env
# Binance API (solo para trading real)
BINANCE_API_KEY=tu_api_key
BINANCE_API_SECRET=tu_api_secret
USE_BINANCE=false  # true para operar en real, false para simulación

# Configuración de símbolo
SYMBOL=BTCUSDT  # Símbolo a operar (nota: stream.py usa USDTARS)

# Parámetros de estrategia
MOMENTUM_THRESHOLD=0.01      # Umbral de momentum para señales (1%)
TAKE_PROFIT_PCT=0.003        # Take Profit (0.3%)
STOP_LOSS_PCT=0.001          # Stop Loss (0.1%)
SLEEP_TIME=5                 # Segundos entre iteraciones del trader
```

### Obtener Token de MotherDuck

1. Visita [MotherDuck](https://motherduck.com/)
2. Crea una cuenta o inicia sesión
3. Genera un token desde el panel de control
4. Copia el token en tu archivo `.env`

## Uso

El sistema requiere ejecutar tres procesos simultáneamente (en terminales separadas):

### Terminal 1: Ingesta de datos

```bash
python stream.py
```

Este proceso:
- Se conecta al WebSocket de Binance
- Captura actualizaciones del order book
- Las persiste en MotherDuck
- Al detener (Ctrl+C), limpia los datos de la sesión

### Terminal 2: Motor de trading

```bash
python trader.py
```

Este proceso:
- Lee datos de MotherDuck cada `SLEEP_TIME` segundos
- Calcula indicadores (VWAP, Momentum)
- Evalúa señales de entrada/salida
- Ejecuta órdenes (simuladas o reales según configuración)
- Al detener (Ctrl+C), limpia los logs de trading

### Terminal 3: Dashboard web

```bash
python app.py
```

Este proceso:
- Inicia el servidor Flask en `http://localhost:5000`
- Expone endpoints REST para el dashboard
- Actualiza datos cada 2 segundos en el frontend

Abre tu navegador en `http://localhost:5000` para ver el dashboard.

## Estructura del Proyecto

```
htf-binance/
├── app.py                 # Servidor Flask y endpoints API
├── stream.py              # Ingesta de datos desde Binance WebSocket
├── trader.py              # Motor de trading automatizado
├── requirements.txt       # Dependencias de Python
├── env.example            # Plantilla de variables de entorno
├── .env                   # Variables de entorno (crear desde env.example)
└── templates/
    └── index.html         # Dashboard frontend (Plotly + jQuery)
```

## Estrategia de Trading

### Señales de Entrada

El sistema evalúa señales basadas en dos indicadores:

1. **VWAP (Volume Weighted Average Price)**: Precio promedio ponderado por volumen
2. **Momentum (Rate of Change)**: Cambio porcentual de precio en los últimos 10 períodos

**Señal BUY (LONG):**
- Precio actual < VWAP
- Momentum > `MOMENTUM_THRESHOLD` (por defecto 1%)

**Señal SELL (SHORT):**
- Precio actual > VWAP
- Momentum < -`MOMENTUM_THRESHOLD` (por defecto -1%)

### Gestión de Salida

Cada posición abierta tiene:

- **Take Profit**: Se activa cuando la ganancia alcanza `TAKE_PROFIT_PCT` (0.3% por defecto)
- **Stop Loss**: Se activa cuando la pérdida alcanza `STOP_LOSS_PCT` (0.1% por defecto)

### Cálculo de R (Risk/Reward)

El sistema calcula:
- **R Esperado**: Ratio entre ganancia potencial y pérdida potencial
- **R Final**: Ratio real obtenido al cerrar la posición

## Dashboard

El dashboard muestra:

### Gráfico Principal (Order Book)
- Líneas de Bid y Ask en tiempo real
- Líneas de Soporte y Resistencia (calculadas dinámicamente)
- Marcadores de operaciones (BUY, SELL, TP, SL)
- Líneas de posición actual (Entry Price, Take Profit, Stop Loss)

### Gráficos Secundarios
- **VWAP**: Volume Weighted Average Price
- **Momentum**: Rate of Change (ROC)
- **Histograma de Spread**: Distribución del spread bid-ask
- **Distribución de Profundidad**: Histograma de precios bid y ask

### Tabla de Resumen
- Historial de operaciones completadas
- Métricas: Entry Price, Exit Price, PnL, R Esperado, R Final

### Modo Simulado vs Real

Por defecto, el sistema opera en **modo simulado** (`USE_BINANCE=false`). Esto significa:
- Las órdenes se registran pero no se ejecutan en Binance
- Puedes probar estrategias sin riesgo
- Los datos de mercado son reales, pero las operaciones son simuladas

Para operar en real:
1. Configura `USE_BINANCE=true` en `.env`
2. Proporciona `BINANCE_API_KEY` y `BINANCE_API_SECRET` válidas
3. **ADVERTENCIA**: Operarás con dinero real. Prueba primero en modo simulado.

### Limpieza de Datos

Al detener los procesos:
- `stream.py` elimina todos los datos de `htf.depth_updates`
- `trader.py` elimina todos los logs de `htf.trading_log`, `htf.position_state` y `htf.trade_summary`

Esto es intencional para mantener las tablas limpias entre sesiones.

### Rendimiento

- El dashboard actualiza cada 2 segundos
- El trader evalúa señales cada `SLEEP_TIME` segundos (por defecto 5)
- El stream procesa cada actualización del WebSocket en tiempo real

## Dependencias

- `Flask`: Framework web para el dashboard
- `python-dotenv`: Gestión de variables de entorno
- `duckdb`: Cliente para MotherDuck
- `python-binance`: Cliente para API de Binance

## Contribuciones

Este es un proyecto académico. Las contribuciones son bienvenidas, pero ten en cuenta el contexto educativo del proyecto.
