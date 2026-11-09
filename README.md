# B03 · Gemelo Digital de Reactor de Mezcla Industrial

Gemelo digital de un **reactor de mezcla industrial por lotes** desarrollado en **Node-RED**. Simula el ciclo completo de un lote (llenado → calentamiento y agitación → descarga) y expone el proceso mediante una **API HTTP/REST** consumida por un **dashboard web** (HTML/CSS/JS) con vista de operación y vista de ingeniería.

> Proyecto B03 · Asignatura Internet de las Cosas · Período 2026-2 · Paralelo B
> Universidad Técnica de Machala (UTMACH)

**Integrantes:**
- Oscar Elias Apolo Cheverria
- Steven Paul Valdiviezo Suquilanda
- Johan Andres Villamar Brito

---

## Tabla de contenido
1. [Arquitectura](#arquitectura)
2. [Modelo dinámico](#modelo-dinámico)
3. [Máquina de estados](#máquina-de-estados)
4. [API HTTP/REST](#api-httprest)
5. [KPI](#kpi)
6. [Alarmas](#alarmas)
7. [Dashboard](#dashboard)
8. [Pruebas de aceptación](#pruebas-de-aceptación)
9. [Instalación y ejecución](#instalación-y-ejecución)
10. [Estructura del repositorio](#estructura-del-repositorio)

---

## Arquitectura

```
Simulador (Node-RED)  ↔  API REST (Node-RED)  ↔  Dashboard HTML/JS (fetch + polling 1 s)
```

El flujo está separado en tres pestañas:

| Pestaña | Función |
|---|---|
| **1. Simulador Físico** | Reloj de simulación (Δt = 1 s) y máquina de estados del lote. Mantiene el estado en el contexto global. |
| **2. Control, KPIs, Alarmas e Históricos** | Cálculo de KPI, detección y registro de alarmas, histórico en disco (`b03_historial.jsonl`). |
| **3. API HTTP/REST** | Endpoints `/api/B03/*`, validaciones, respuestas JSON con CORS y servicio del dashboard en `/dashboard`. |

El dashboard **no lee variables globales**: toda la información llega por HTTP mediante `fetch()` con polling de 1 s.

## Modelo dinámico

Parámetros de referencia: capacidad **500 L**, consigna **65 °C**, agitador hasta **600 rpm**. Paso de simulación Δt = 1 s. Se añade ruido pequeño (±0.06 °C en temperatura, ±1 rpm en agitación) como perturbación.

| Variable | Unidad | Ecuación / regla |
|---|---|---|
| Temperatura `T` | °C | Calentando: `T(k+1) = T + 0.18·(T_set − T)`. Enfriamiento natural: `T − 0.04·(T − 25)`. Embalamiento del calentador: `T + 1.85` por segundo. |
| Nivel `V` | L | Llenado `+20 L/s` (tope 500 L); descarga `−25 L/s`; aborto `−35 L/s`. |
| Agitación `ω` | rpm | `ω(k+1) = ω + 0.35·(ω_set − ω)`; 0 rpm si hay falla del agitador. |
| Concentración `C` | % | `C(k+1) = min(100, C + 2.35·(ω/450)·(T/65))` mientras `ω ≥ 20 rpm`. La mezcla mejora con rpm y tiempo. |
| Potencia calentador | kW | `clamp(1.8·(T_set − T) + 3.2, 1.5, 18)`; 18 kW en embalamiento; 0 si está apagado. |
| Potencia motor | kW | `(ω / 600)·2.4` |

**Supuestos:** valores didácticos de referencia (no corresponden a una instalación real); balance térmico simplificado de primer orden con aporte de calor y pérdidas hacia 25 °C (temperatura ambiente).

## Máquina de estados

```
REPOSO ──INICIAR_LOTE──▶ LLENANDO ──nivel ≥ objetivo──▶ CALENTANDO_MEZCLANDO
   ▲                                                          │
   │                              C ≥ 98 % y |T − T_set| ≤ 3 °C
   │                                                          ▼
   └──────────────nivel = 0────────────────────────────── VACIANDO

(cualquier fase) ──ABORTAR_LOTE──▶ ABORTADO ──nivel = 0──▶ REPOSO
```

En `ABORTADO` se corta el calentador y el agitador, y el tanque se vacía de forma segura.

## API HTTP/REST

Base por defecto: `http://localhost:1880` · Todas las respuestas incluyen `idDispositivo`, unidades y `marcaTiempo` (ISO 8601). CORS habilitado.

| Método | Ruta | Descripción | Códigos |
|---|---|---|---|
| GET | `/api/B03/telemetry` | Temperatura, nivel, agitación y concentración con unidades y consignas | 200 |
| GET | `/api/B03/status` | Fase, lote activo, consignas, actuadores, fallas y alarmas | 200 |
| GET | `/api/B03/kpis` | Los 5 KPI con valor, unidad y fórmula | 200 |
| GET | `/api/B03/history?ventana=1m\|5m\|15m\|todo` | Histórico de muestras con ventana configurable | 200, 400 |
| PUT | `/api/B03/setpoint` | Ajusta consignas de temperatura y agitación | 200, 400, 409 |
| POST | `/api/B03/commands` | Comandos de control (ver abajo) | 200, 400, 409 |
| POST | `/api/B03/batches/:batchId` | Crea e inicia un lote con identificador único | 201, 409 |
| GET | `/api/B03/batches/:batchId` | Consulta un lote por su ID | 200, 404 |
| GET | `/dashboard` | Sirve la interfaz web | 200 |

### Comandos (`POST /api/B03/commands`)

| `comando` | Efecto |
|---|---|
| `INICIAR_LOTE` | Inicia un lote (solo en `REPOSO`, si no responde 409). Acepta `idLote` y `volumenObjetivo_L` (opcionales). |
| `ABORTAR_LOTE` | Aborto seguro y vaciado del tanque. |
| `RECONOCER_ALARMAS` | Marca las alarmas como reconocidas y restablece las fallas. |
| `INYECTAR_FALLA` | Con `tipoFalla`: `EMBALAMIENTO_CALENTADOR`, `CALENTADOR_APAGADO`, `FALLO_AGITADOR` o `SOBRELLENADO`. |
| `RESTABLECER_FALLAS` | Vuelve a operación normal. |

Un comando desconocido responde **400**. También se aceptan los equivalentes en inglés (`START_BATCH`, `ABORT_BATCH`, `ACK_ALARMS`, `SET_FAULT`, `RESET_FAULTS`).

### Ejemplos

**Iniciar un lote**
```bash
curl -X POST http://localhost:1880/api/B03/commands \
  -H "Content-Type: application/json" \
  -d '{"comando":"INICIAR_LOTE","idLote":"LOT-001","volumenObjetivo_L":400}'
```
```json
{ "mensaje": "Lote iniciado en fase LLENANDO", "idLote": "LOT-001", "fase": "LLENANDO", "estado": 200, "marcaTiempo": "2026-11-08T15:00:00.000Z" }
```

**Cambiar consignas** (rangos: 25–90 °C y 0–600 rpm; se envían ambos campos)
```bash
curl -X PUT http://localhost:1880/api/B03/setpoint \
  -H "Content-Type: application/json" \
  -d '{"temperatura_C":70,"agitacion_rpm":520}'
```

**Consigna fuera de rango → 400**
```json
{ "error": "Consigna fuera de rango permitido (Temperatura: 25-90 °C, Agitación: 0-600 rpm)", "estado": 400, "marcaTiempo": "..." }
```

**Crear lote con ID único**
```bash
curl -X POST http://localhost:1881/api/B03/batches/LOT-002 \
  -H "Content-Type: application/json" \
  -d '{"volumenObjetivo_L":350,"temperatura_C":65,"agitacion_rpm":450}'
```
Responde **201** si el reactor está en `REPOSO`, o **409** si está ocupado.

**Telemetría**
```json
{
  "idDispositivo": "REACTOR-B03",
  "marcaTiempo": "2026-11-08T15:00:05.000Z",
  "fase": "CALENTANDO_MEZCLANDO",
  "loteActivoId": "LOT-001",
  "variables": {
    "temperatura":   { "valor": 48.3, "unidad": "°C", "consigna": 65 },
    "nivel":         { "valor": 400,  "unidad": "L",  "objetivo": 400, "maximo": 500 },
    "agitacion":     { "valor": 449.1,"unidad": "rpm","consigna": 450, "maximo": 600 },
    "concentracion": { "valor": 12.4, "unidad": "%" }
  }
}
```

### Códigos de error

| Código | Cuándo ocurre |
|---|---|
| 400 | Consigna fuera de rango, ventana inválida o comando desconocido |
| 404 | Lote inexistente |
| 409 | Comando incompatible con el estado (p. ej. iniciar lote fuera de `REPOSO`, cambiar consignas en `ABORTADO`) |

## KPI

| KPI | Fórmula | Unidad | Actualización | Fuente |
|---|---|---|---|---|
| Lotes por hora | `Lotes_completados / (Tiempo_operación_s / 3600)` | lotes/h | 1 s | Contador de lotes y tiempo de operación |
| Duración del lote | `(t_fin_descarga − t_inicio_llenado) / 60` | min | Al cerrar cada lote | Temporizador de lote |
| Desviación térmica | `\|T_actual − T_consigna\|` | °C | 1 s | Temperatura y consigna |
| Volumen procesado | `Σ Volumen de lotes completados` | L | Al cerrar cada lote | Volumen objetivo del lote |
| Consumo estimado | `Σ (P_calentador + P_motor)·Δt / 3600` | kWh | 1 s | Potencias de actuadores |

## Alarmas

| Alarma | Condición | Severidad |
|---|---|---|
| `SOBRETEMPERATURA` | Temperatura > 75 °C | CRÍTICO |
| `NIVEL_MÁXIMO` | Nivel ≥ 480 L (cierra la válvula de entrada y pasa a mezcla) | CRÍTICO |
| `AGITADOR_DETENIDO` | Agitación < 20 rpm durante `CALENTANDO_MEZCLANDO` | ALTO |

Cada alarma se registra con `id`, código, severidad, umbral, valor, marca de tiempo y estado `reconocida`. Se reconocen con el comando `RECONOCER_ALARMAS`.

## Dashboard

Disponible en `http://localhost:1880/dashboard`. Incluye:

- **Vista de operación:** sinóptico animado del reactor y actuadores (válvulas, calentador, agitador), control de lotes, consignas, reconocimiento de alarmas y aborto seguro.
- **Vista de ingeniería:** gráficas históricas (temperatura y proceso) con ventana de 1, 5 o 15 min, bitácora de solicitudes HTTP (código y latencia), inyección de fallas y botones de las pruebas de aceptación.

El campo de URL base del dashboard permite apuntar a otra dirección de la API.

## Pruebas de aceptación

| # | Prueba | Estímulo | Resultado esperado |
|---|---|---|---|
| 1 | Lote normal | `POST /batches/LOT-xxx` | **201**; recorre llenado → mezcla → descarga → reposo |
| 2A | Cambio de consigna | `PUT /setpoint` a 70 °C y 520 rpm | **200**; el reactor ajusta temperatura y agitación |
| 2B | Consigna fuera de rango | `PUT /setpoint` a 125 °C | **400** |
| 3 | Fallo de calentador | Inyectar `EMBALAMIENTO_CALENTADOR` | Temperatura > 75 °C y alarma `SOBRETEMPERATURA` |
| 4 | Fallo de agitador | Inyectar `FALLO_AGITADOR` | Agitación 0 rpm, la concentración se congela y se activa `AGITADOR_DETENIDO` |
| 5 | Aborto seguro | `ABORTAR_LOTE` y luego `PUT /setpoint` | Vaciado seguro; el cambio de consigna responde **409** |

> Completar el acta con resultado obtenido, evidencia (capturas) y conclusión de cada prueba.

## Instalación y ejecución

**Requisitos:** Node.js LTS y Node-RED. El dashboard carga Tailwind CSS y Chart.js desde CDN, por lo que requiere conexión a internet.

1. Instalar Node-RED:
   ```bash
   npm install -g --unsafe-perm node-red
   ```
2. Iniciar Node-RED en el puerto **1880** (el valor por defecto del dashboard):
   ```bash
   node-red -p 1881
   ```
   Si usas otro puerto, cámbialo en el campo de URL base del dashboard.
3. Abrir el editor en `http://localhost:1880` → menú ☰ → **Import** → seleccionar `flows/B03-Reactor-industrial_V2.json` → **Deploy**.
4. Abrir el dashboard en `http://localhost:1880/dashboard`.
5. El histórico se guarda en `b03_historial.jsonl` (directorio de trabajo de Node-RED).

**Persistencia opcional:** para que el contexto global sobreviva a reinicios, habilita el contexto *file-backed* en `settings.js`:
```js
contextStorage: {
  default: { module: "localfilesystem" }
}
```

**Demostración rápida:** abrir el dashboard → *Iniciar Nuevo Lote* → observar el sinóptico, KPI y gráficas → inyectar una falla desde la vista de ingeniería → reconocer la alarma → ejecutar un aborto seguro.

## Estructura del repositorio

```
.
├── flows/
│   └── B03-Reactor-industrial_V2.json   # Flujos Node-RED (sin credenciales)
├── docs/
│   ├── capturas/                        # Capturas del dashboard y alarmas
│   └── acta-pruebas.md                  # Plan y acta de las 5 pruebas
└── README.md
```

## Licencia
Proyecto académico · UTMACH · Internet de las Cosas 2026-2.