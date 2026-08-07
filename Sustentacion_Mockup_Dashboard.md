# Sustentacion del Mockup: Dashboard de Monitoreo Energetico

## Documento de defensa tecnica del archivo `mockup_uptime_analytics_v5_delta_context.html`

---

## 1. PROPOSITO DEL MOCKUP

### 1.1 Que pide la prueba

La prueba tecnica solicita un **mockup funcional de un dashboard** que comunique los resultados del analisis de desempeno energetico. No pide un grafico estatico ni una captura de pantalla: pide una pieza que demuestre como se veria el producto Uptime Analytics en produccion.

### 1.2 Que se entrega

Un archivo HTML autocontenido de ~1.4 MB que:

- Embebece los **43.200 registros** del dataset completo de junio 2026
- Simula **monitoreo en tiempo real** con animaciones y reloj en vivo
- Permite **explorar cualquier minuto** del mes con un slider temporal
- Calcula **todas las metricas dinamicamente** segun la ventana y granularidad seleccionada
- Implementa un **sistema de alertas configurable** basado en reglas
- Muestra los **6 indicadores del Anexo 3** de la prueba
- Funciona en **cualquier navegador** sin conexion a internet ni dependencias externas

### 1.3 Decision de diseño fundamental

Se opto por construir una simulacion interactiva en lugar de un mockup estatico. La razon es que un dashboard de monitoreo energetico tiene valor en su capacidad de responder preguntas en tiempo real, no en mostrar un numero fijo. Un mockup estatico no puede demostrar eso. Esta simulacion si.

---

## 2. ARQUITECTURA DEL DASHBOARD

### 2.1 Stack tecnologico

| Componente | Tecnologia | Justificacion |
|---|---|---|
| Estructura | HTML5 semantico | Accesibilidad y SEO |
| Estilos | CSS3 inline (variables, grid, flexbox) | Autocontenido, sin dependencias |
| Graficos | SVG nativo con manipulacion JS | Crisp rendering, accesibilidad (`aria-label`), zoom sin pixelado |
| Interactividad | JavaScript vanilla | Sin framework = sin dependencias, sin build step |
| Datos | JSON embebido en `<script>` | Archivo unico, funciona offline |
| Animaciones | CSS `@keyframes` | GPU-accelerated, sin JS overhead |

### 2.2 Por que no se uso una libreria de graficos

Chart.js, D3, Plotly o Recharts habrian simplificado el desarrollo, pero:

1. **Dependencia externa**: El archivo dejaria de ser autocontenido. Si el CDN no responde, el dashboard no funciona.
2. **Peso innecesario**: Chart.js minificado son ~200 KB adicionales. El SVG nativo logra lo mismo con menos codigo.
3. **Control total**: El tooltip, los hover indicators, las bandas de tolerancia y el state strip requieren personalizacion que en librerias genericas implica workarounds.
4. **Mensaje implicito**: Demuestra dominio tecnico real, no solo capacidad de conectar una API de graficos.

### 2.3 Estructura de datos embebidos

```javascript
const DATA = {
  start: "2026-06-01T00:00:00",  // Timestamp de inicio
  n: 43200,                       // Total de registros (30 dias × 24h × 60min)
  prod: [...],                    // Produccion (t/h) por minuto
  real: [...],                    // Energia real (kWh) por minuto
  base: [...],                    // Energia esperada segun linea base (kWh)
  fp: [...],                      // Factor de potencia por minuto
  state: "OOOO...PPPP...AAAA",   // Estado operacional codificado (O/P/A/T)
  mode: "AAAA...MMMM...AAAA",    // Modo operativo codificado (A/M)
  summary: {                      // Metricas precalculadas del mes
    month_compliance: 99.98,
    net_dev: 203.49,
    within5: 98.80
  }
};
```

**Por que codificar estado y modo como strings de un caracter?**

Un array de strings `["OPERANDO","OPERANDO",...]` con 43.200 elementos ocuparia ~500 KB. Un string de caracteres unicos `"OOOOPPPP..."` ocupa 43 KB. Es 10x mas eficiente y el acceso `STATE[i]` sigue siendo O(1).

### 2.4 Responsividad

El dashboard tiene tres breakpoints:

| Breakpoint | Adaptacion |
|---|---|
| > 1080px | Layout completo: workspace 2 columnas, bottom 2 columnas |
| 720px - 1080px | Workspace y bottom pasan a 1 columna, alertas sin sticky |
| < 720px | Topbar simplificada, oculta liveBadge y gauge, conveyor oculto, grids apilados |

---

## 3. COMPONENTES DEL DASHBOARD - DETALLE Y JUSTIFICACION

### 3.1 Barra superior (Header)

```
[Logo Uptime Analytics] [Titulo: Monitoreo de desempeno energetico · Planta 01]  [● LIVE 30 jun · 23:59:14] [Exportar] [Alertas]
```

**Elementos:**
- **Logo**: Imagen base64 embebida (no requiere fetch externo)
- **Subtitulo**: "Simulacion de operacion en tiempo real usando el historico de junio de 2026" — transparencia total sobre lo que es el mockup
- **Badge LIVE**: Punto verde con animacion CSS `pulse`. Reloj que se actualiza cada segundo via `setInterval`. En modo historico, cambia a mostrar el minuto seleccionado.
- **Boton Exportar**: Genera un CSV real con las metricas del periodo seleccionado. No es un boton muerto: descarga un archivo `uptime_monitoreo_energetico.csv` con headers, datos por estado y configuracion de tarifa.
- **Boton Alertas**: Abre el modal de configuracion de umbrales.

**Como defenderlo:**
> "La barra superior establece el contexto operativo inmediato: que planta, que momento, si hay alertas activas. El badge LIVE con el punto pulsante es un patron estandar en dashboards de monitoreo (Datadog, Grafana, New Relic). El boton Exportar no es decorativo: genera un CSV real."

### 3.2 Seccion de produccion en vivo

```
[⚙ Engranaje giratorio]  126,8 t/h    PLANTA PRODUCIENDO    [═══ Flujo de produccion ═══>]
                                                               OPERANDO · AUTOMATICO
```

**Elementos:**
- **Engranaje animado**: CSS `@keyframes spin`, gira a 3s por revolucion. Se **pausa** cuando la planta esta en PARADO.
- **Cinta transportadora**: Gradiente lineal animado con `@keyframes flow`. Opacidad reducida a 0.25 en PARADO.
- **Estado dinamico**: Cambia color y texto segun estado. Verde = PRODUCIENDO, Rojo = DETENIDA, Ambar = PARANDO/ARRANQUE.

**Por que animaciones?**

> "En un centro de control real, el operador necesita saber de un vistazo si la planta esta activa. Las animaciones no son decorativas: transmiten estado operacional sin requerir lectura de texto. El engranaje parado y la cinta detenida son señales visuales inmediatas."

### 3.3 Telemetria en tiempo real

```
[Consumo minuto: 22,15 kWh]  [Linea base: 22,20 kWh]  [Factor de potencia: 0,922]
```

Tres cajas que se actualizan con cada minuto seleccionado. Muestran las variables operativas clave del instante.

**Por que factor de potencia y no voltaje/corriente?**

> "El factor de potencia es el indicador electrico mas relevante para gestion energetica: un FP bajo implica penalizaciones tarifarias y perdida de eficiencia. Voltaje y corriente son datos de ingenieria, no de gestion. El dashboard esta diseñado para un Product Analyst, no para un electricista."

### 3.4 Panel de cumplimiento acumulado

```
┌──────────────────────────────────────────────────────────────┐
│ CUMPLIMIENTO ACUMULADO · JUNIO 2026                          │
│ 99,98 %                                    [Gauge ≈100%]     │
│ ● OPERACION GLOBAL ESTABLE                                   │
│ vs. mayo 2026: No disponible                                 │
│                                                              │
│ [Desviacion neta]  [Dentro de ±5%]  [Impacto economico]      │
│  +203,49 kWh        98,80 %          Sin tarifa              │
│                                                              │
│ [GEI]              [Ahorro]          [Indice consumo]        │
│  20,35 kgCO₂eq     -0,022 %          10,5452 kWh/t          │
└──────────────────────────────────────────────────────────────┘
```

**Los 6 indicadores del Anexo 3:**

| # | Indicador | Valor | Calculo |
|---|---|---|---|
| 1 | Cumplimiento (%) | 99,98 % | `(E_base / E_real) × 100` |
| 2 | Desempeno energetico (kWh) | +203,49 kWh | `Σ(E_real) - Σ(E_base)` |
| 3 | GEI (kgCO₂eq) | 20,35 | `Desviacion × 0,1000 kgCO₂eq/kWh` |
| 4 | Desempeno economico ($) | Parametrizado | `Desviacion × tarifa` (configurable) |
| 5 | Ahorro (%) | -0,022 % | `(1 - E_real/E_base) × 100` |
| 6 | Indice de consumo (kWh/t) | 10,5452 | `Σ(E_real) / (Σ(Produccion) / 60)` |

**Decisiones clave:**

**Gauge circular:** Representa visualmente que el cumplimiento esta cerca del 100%. El `conic-gradient` CSS rellena 359 de 360 grados en verde, comunicando "casi perfecto" de un vistazo.

**"vs. mayo 2026: No disponible":** Se muestra explicitamente que no hay datos del mes anterior para comparar. No se inventa un delta. El texto aclara: "Se requiere al menos un mes historico adicional para calcular el delta mensual real."

**"Sin tarifa" en impacto economico:** Consistente con la decision del notebook de no asumir tarifa electrica. El dashboard tiene un campo donde el usuario ingresa su tarifa real (ver seccion 3.9).

**Indice de consumo 10,5452 kWh/t:** Se calcula como el total de energia real dividido entre el total de produccion en toneladas. La produccion se convierte de t/h a toneladas reales dividiendo por 60 (cada registro representa 1 minuto = 1/60 de hora).

**GEI 20,35 kgCO₂eq:** Factor de emision de 0,1000 kgCO₂eq/kWh aplicado a la desviacion neta. Este factor es un placeholder y deberia ajustarse al factor de emision del sistema interconectado del pais de la planta.

**Como defenderlo:**
> "Los 6 indicadores del Anexo 3 estan visibles en el panel principal sin necesidad de scroll. El ahorro es negativo (-0,022%) porque la planta consumio marginalmente mas de lo esperado. El indice de consumo de 10,55 kWh/t es el ratio agregado del mes. Todos son coherentes con los valores del notebook."

### 3.5 Panel de desviacion actual

```
┌─────────────────────────────────────────────┐
│ ▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬ (barra verde)      │
│ Ahora · ultimo dato recibido                │
│ 30 jun · 23:59    [OPERANDO] [AUTOMATICO]   │
│                                  ● NORMAL   │
│ Desviacion vs. linea base                   │
│ -0,20 %               1 minuto              │
│                        Cumplimiento 100,20% │
│                                             │
│ La planta esta produciendo y el ultimo      │
│ minuto se encuentra dentro de los umbrales. │
│                                             │
│ [1 minuto]     [Minuto anterior]   [Δ Delta]│
│  -0,20 %        -1,07 %           +0,86 pp │
│  Periodo actual  Periodo anterior   Menor   │
│                                   desviacion│
│                                             │
│ Esta desviacion esta calculada sobre el     │
│ ultimo minuto. Es la vista mas sensible...  │
└─────────────────────────────────────────────┘
```

**Elementos:**
- **Barra superior coloreada**: Verde/ambar/rojo segun severidad de la desviacion
- **Chips de estado**: Muestran OPERANDO/PARADO/etc. y AUTOMATICO/MANUAL
- **Desviacion porcentual**: Tamaño grande (42px), color segun severidad
- **Comparacion temporal**: 3 cajas que comparan el periodo actual vs. el anterior equivalente, con delta en puntos porcentuales
- **Nota contextual**: Explica que significa la granularidad seleccionada

**La comparacion temporal es clave:**

El delta muestra si la desviacion esta mejorando o empeorando respecto al periodo anterior equivalente:
- En granularidad "1 min": compara con el minuto inmediatamente anterior
- En "15 min": compara con los 15 minutos previos
- En "1 h": compara con la hora anterior
- En "Turno": compara con el turno anterior (480 minutos antes)
- En "Dia": compara con el dia anterior (1440 minutos antes)
- En "Mes": no hay comparacion (solo hay datos de junio)

Cuando no existe periodo anterior (inicio del mes, primer turno), muestra "No disponible" con explicacion.

**Como defenderlo:**
> "Este panel responde a la pregunta mas inmediata del operador: como esta la planta AHORA. La comparacion temporal le dice si estamos mejor o peor que antes. El color cambia automaticamente: verde si esta dentro de umbrales, ambar si supera el preventivo, rojo si supera el critico. La nota contextual educa al operador para que no sobrerreaccione a una fluctuacion de un solo minuto."

### 3.6 Controles interactivos

```
[● Ahora] [Explorar historico]    [01 jun 00:00 ════════════ 30 jun 23:59]    [1m] [15m] [1h] [Turno] [Dia] [Mes]
```

**Modos de visualizacion:**
- **Ahora (Live)**: Fija la vista en el ultimo minuto (43199). El slider se deshabilita. La etiqueta del reloj muestra segundos.
- **Explorar historico**: Habilita el slider. El usuario puede navegar a cualquier minuto del mes. La etiqueta cambia a "Historico · [fecha/hora]".

**Granularidades:**
| Granularidad | Bucket | Span visible | Uso |
|---|---|---|---|
| 1 min | 1 min | 3 horas (180 pts) | Deteccion de eventos puntuales |
| 15 min | 15 min | 12 horas (720 pts) | Medio turno, reduccion de ruido |
| 1 h | 1 hora | 3 dias (4320 pts) | Confirmacion de tendencia |
| Turno | 1 hora | 7 dias (10080 pts) | Evaluacion operativa |
| Dia | 4 horas | 14 dias (20160 pts) | Tendencia semanal |
| Mes | 1 dia | 30 dias (43200 pts) | Vision global |

Cada cambio de granularidad recalcula: el grafico, la tabla de estados, los range summary boxes, la comparacion temporal, y la nota contextual. Todo instantaneo porque los datos estan en memoria.

**Como defenderlo:**
> "La granularidad es fundamental para evitar la paradoja del monitoreo: si miras muy de cerca, todo parece alarma; si miras muy lejos, todo parece bien. Las 6 granularidades permiten al operador ajustar su nivel de detalle segun la pregunta que quiere responder. La nota contextual cambia para explicar que significa cada nivel."

### 3.7 Grafico SVG: Consumo real vs. linea base

**Elementos visuales:**
- **Polyline azul oscuro (navy)**: Energia real por bucket
- **Polyline cyan**: Linea base esperada
- **Banda verde clara (±3%)**: Zona de operacion normal
- **Banda azul clara (±5%)**: Zona de atencion
- **State strip**: Barra horizontal coloreada por estado operacional (verde=OPERANDO, rojo=PARADO, azul=ARRANQUE, ambar=PARANDO)
- **Cursor rojo punteado**: Marca el punto seleccionado/actual
- **Labels de eje Y**: Se regeneran con cada cambio de granularidad para mostrar la escala correcta

**Interactividad al pasar el cursor:**

Al hacer hover sobre el grafico:

1. **Linea vertical de hover** (punteada navy): Sigue el mouse sobre el eje X
2. **Circulo navy**: Se posiciona sobre la curva real en el punto mas cercano
3. **Circulo cyan**: Se posiciona sobre la curva base en el punto mas cercano
4. **Tooltip enriquecido** que muestra:

```
┌──────────────────────────────┐
│ 30 de jun, 22:26             │
│ Real           22,22 kWh     │
│ Linea base     22,35 kWh     │
│ Desviacion     -0,61 %       │
│ Produccion     127,7 t/h     │
│ Factor pot.    0,910         │
│ [OPERANDO] [AUTOMATICO]      │
└──────────────────────────────┘
```

El tooltip muestra:
- **Fecha y hora exacta** del punto (responde "cuando")
- **Energia real y base** para cuantificar la desviacion
- **Desviacion porcentual** con color (verde/ambar/rojo)
- **Produccion** para contextualizar el consumo
- **Factor de potencia** como indicador de calidad electrica
- **Estado y modo** para saber en que condicion operaba la planta

Cuando la granularidad no es 1 minuto, se muestra el rango completo del bucket: "30 de jun, 22:00 → 22:59".

**Range summary boxes:**

Debajo del grafico, 5 cajas muestran la desviacion del punto actual en todas las granularidades simultaneamente, con delta vs. periodo anterior:

```
[1 min: -0,20%]  [15 min: -0,46%]  [1 hora: +0,36%]  [Turno: +0,11%]  [Dia: -0,01%]
 vs +0,86 pp      vs -0,83 pp       vs +0,37 pp        vs +0,39 pp      vs +0,10 pp
```

Esto permite al operador ver si la desviacion es puntual o sostenida sin cambiar de granularidad.

**Como defenderlo:**
> "El grafico no es un chart estatico: es la herramienta central de investigacion. Las bandas de ±3% y ±5% dan contexto visual inmediato de si un punto es normal o anormal. El state strip en la parte inferior conecta los picos del grafico con los estados operacionales. Y el tooltip muestra TODA la informacion relevante del punto: cuando fue, que producia la planta, en que estado y modo estaba, y cuanto se desvio. Eso es lo que un operador necesita para decidir si investiga o no."

### 3.8 Tabla de comportamiento por estado operacional

```
Analizar respecto a: [Junio completo ▼]
Periodo: Junio completo · 43.200 minutos

| Estado    | Tiempo    | Desviacion | Impacto/min | Desv. energetica | Impacto economico |
|-----------|-----------|------------|-------------|------------------|-------------------|
| ARRANQUE  | 0,25%     | +19,47%    | Alto/min    | +27,48 kWh       | —                 |
| PARANDO   | 0,33%     | +17,97%    | Alto/min    | +29,38 kWh       | —                 |
| OPERANDO  | 96,51%    | -0,02%     | Bajo/min    | +18,46 kWh       | —                 |
| PARADO    | 2,91%     | +14,39%    | Alto/min    | +128,16 kWh      | —                 |
```

**Funcionalidades:**
- **Selector de alcance**: Permite analizar la misma tabla sobre distintos periodos (misma granularidad, ultima hora, turno, dia, 7 dias, junio completo)
- **Chips de impacto**: Clasificacion visual (Bajo/Medio/Alto) basada en la intensidad de desviacion por minuto vs. los umbrales configurados
- **Impacto economico**: Se activa cuando el usuario configura una tarifa

**Como defenderlo:**
> "Esta tabla responde directamente a la pregunta 3 de negocio: de donde viene la desviacion? OPERANDO ocupa el 96,5% del tiempo pero solo aporta una desviacion de -0,02%. PARADO ocupa el 2,9% pero aporta +128 kWh. El selector de alcance permite ver si este patron se repite en un turno especifico o si es un fenomeno mensual."

### 3.9 Seccion de impacto economico configurable

```
┌─────────────────────────────────────────────────────────────┐
│ IMPACTO ECONOMICO CONFIGURABLE                              │
│ Convierte kWh desviados a la moneda del cliente             │
│ No se asume una tarifa. El usuario define el costo.         │
│                                                             │
│ Costo por kWh: [________]  Moneda: [COP · $ ▼]  [Aplicar]  │
│                                                             │
│ [Impacto junio: —]         [Impacto periodo tabla: —]       │
│                                                             │
│ Impacto economico = Desviacion × Costo por kWh.             │
│ Configure una tarifa para activar el calculo.               │
└─────────────────────────────────────────────────────────────┘
```

**Monedas soportadas:** COP, USD, EUR, MXN, BRL, PEN, CLP — cada una con su simbolo y formato de localizacion (`toLocaleString` con locale especifico).

**Flujo de uso:**
1. El usuario ingresa la tarifa (ej. 850)
2. Selecciona la moneda (ej. COP)
3. Presiona "Aplicar"
4. Inmediatamente se actualizan: los dos cuadros de impacto, la columna de impacto economico en la tabla de estados, la formula visible, y el KPI de impacto economico en el panel de cumplimiento

**Persistencia:** La configuracion se guarda en `localStorage` bajo la clave `uptimeMockupCfgV3`. Al recargar la pagina, se restaura la tarifa y moneda configuradas.

**Como defenderlo:**
> "Esta seccion es la materializacion de una decision metodologica: no asumir datos que no existen. En produccion, cada planta de cada pais tiene una estructura tarifaria diferente. COP puede tener tarifa regulada o no regulada, con componente de demanda, con penalizacion por factor de potencia. La unica forma responsable de mostrar impacto economico es dejar que el administrador configure la tarifa real. El dashboard soporta 7 monedas porque Uptime Analytics opera en Latinoamerica."

### 3.10 Centro de alertas

```
┌─────────────────────────────┐
│ CENTRO DE ALERTAS           │
│ Lo que requiere atencion    │
│                             │
│ ┌─ ✓ Sin alertas activas ─┐│
│ │  La planta esta operando ││
│ └──────────────────────────┘│
│                             │
│ [Criticas: 3] [Atencion: 8] │
│ [Activas ahora: 0]          │
│                             │
│ [Recientes] [Criticas] [At] │
│                             │
│ ■ Parada prolongada    CRIT │
│   22 jun 10:47 · 216 min   │
│   AUTO · consumo inactivo  │
│   [+61,11 kWh] [+14,10 %]  │
│                             │
│ M Modo manual         ATEN  │
│   21 jun 00:08 · 61 min    │
│   ...                       │
│                             │
│ Parada ≥5m · crit ±5% /10m │
│                [Configurar] │
└─────────────────────────────┘
```

**Tipos de alerta detectados automaticamente:**

| Tipo | Icono | Condicion | Severidad |
|---|---|---|---|
| Parada prolongada | ■ | Estado PARADO por ≥ N minutos | Critica si ≥30 min, sino Atencion |
| Desviacion critica sostenida | Δ | Desviacion ≥ ±crit% por ≥ persist minutos | Critica |
| Desviacion preventiva sostenida | Δ | Desviacion entre ±warn% y ±crit% por ≥ persist minutos | Atencion |
| Modo manual prolongado | M | Modo MANUAL por ≥ N minutos | Atencion |

**Logica anti-fatiga:**

La persistencia es clave. Una desviacion de 6% durante 1 minuto NO genera alerta. Solo se dispara si la condicion se mantiene durante N minutos configurables. Esto evita la fatiga de alertas que invalida cualquier sistema de monitoreo.

Las paradas que ya generan alerta propia no se cuentan como desviacion critica (anti-duplicacion via funcion `overlaps()`).

**Interaccion:** Cada alerta es clickeable. Al hacer clic, el dashboard navega automaticamente al momento del evento, cambia a modo historico y granularidad 1 minuto, y muestra un toast "Evento historico abierto".

**Hero banner dinamico:** El recuadro superior cambia segun haya alertas activas en el minuto seleccionado:
- Verde: "Sin alertas activas"
- Ambar/Rojo: Muestra la alerta mas severa con su detalle

**Como defenderlo:**
> "Un dashboard sin alertas es un poster. Nadie mira un dashboard todo el dia. El valor de un sistema de monitoreo esta en detectar automaticamente condiciones que requieren atencion. La persistencia evita falsos positivos: un pico de un minuto es ruido, una parada de 216 minutos consumiendo energia es un problema real que cuesta +61 kWh. Y las alertas son clickeables: un operador ve la alerta, hace clic, y el dashboard lo lleva al momento exacto para investigar."

### 3.11 Panel de priorizacion

```
┌──────────────────────────────────────────────────────────────┐
│ PRIORIDAD DE INVESTIGACION · JUNIO                           │
│                                                              │
│ [1] PARADO × AUTO    [2] PARANDO         [3] ARRANQUE       │
│     62,79 %               14,45 %             13,51 %       │
│     +127,77 kWh           Transiciones...     110 minutos... │
│     [Ver evento →]        [Ver evento →]      [Ver evento →] │
│                                                              │
│ Nota: son prioridades de investigacion, no causas raiz.      │
└──────────────────────────────────────────────────────────────┘
```

**Datos hardcodeados del analisis del notebook:**
- #1 PARADO × AUTOMATICO: 62,79% de la desviacion neta (+127,77 kWh)
- #2 PARANDO: 14,45% (transiciones operativas)
- #3 ARRANQUE: 13,51% (alta intensidad en pocos minutos)

**Botones "Ver evento":** Cada uno navega al episodio mas representativo:
- PARADO × AUTO: busca el episodio mas largo de PARADO en modo AUTOMATICO (`findLongest("P","A")`)
- PARANDO: busca el minuto de maxima desviacion en estado PARANDO (`findMax("T")`)
- ARRANQUE: busca el minuto de maxima desviacion en arranque (`findMax("A")`)

**La nota al pie es intencional:**
> "Son prioridades de investigacion, no causas raiz demostradas."

Esto refleja rigurosidad metodologica: el analisis estadistico identifica DONDE esta el problema, no POR QUE ocurre. La causa requiere inspeccion en campo.

**Como defenderlo:**
> "Este panel conecta el analisis del notebook con la accion. El notebook identifica que 63% de la desviacion esta en PARADO × AUTOMATICO. El dashboard materializa eso como la prioridad #1 y permite al operador navegar directamente al evento para investigarlo. Eso es cerrar el ciclo: dato → insight → accion."

### 3.12 Modal de configuracion de alertas

Parametros configurables:
- Umbral preventivo (%)
- Umbral critico (%)
- Persistencia de desviacion (minutos)
- Parada: alertar despues de (minutos)
- Modo manual: alertar despues de (minutos)
- Moneda predeterminada

**Validacion:** El umbral preventivo debe ser menor al critico. Si no, muestra un toast de error.

**Persistencia:** Se guarda en `localStorage`. Al guardar, se recalculan todas las alertas inmediatamente.

**Como defenderlo:**
> "Cada planta tiene tolerancias diferentes. Una planta de cemento puede aceptar ±10% de desviacion; una farmaceutica puede requerir ±2%. Los umbrales configurables permiten que el producto se adapte al cliente sin cambiar codigo."

### 3.13 Funcion de exportacion

El boton "Exportar" genera un CSV con:
- Resumen general (periodo, cumplimiento, desviacion, tarifa, moneda)
- Tabla de estados con metricas completas
- BOM UTF-8 para compatibilidad con Excel

**Como defenderlo:**
> "Un operador necesita llevar datos a una reunion, a un informe, a un correo. El CSV con BOM se abre correctamente en Excel sin problemas de caracteres especiales. Es la forma mas simple de sacar datos del dashboard sin requerir una API."

---

## 4. COHERENCIA NOTEBOOK - DASHBOARD

| Concepto | Valor en notebook | Valor en dashboard | Coherente? |
|---|---|---|---|
| Linea base | E = 0,159311 × Prod + 2,002463 | Precalculada en `DATA.base` | Si |
| R² | 0,9692 | No mostrado (dato del modelo, no del monitoreo) | Correcto |
| Cumplimiento | ~99,98% | 99,98% | Si |
| Desviacion neta | +203,49 kWh | +203,49 kWh | Si |
| PARADO × AUTO | 62,79% | 62,79% | Si |
| Tarifa | `None` (parametrizada) | "Sin tarifa" (configurable) | Si |
| Factor emision | 0,1000 kgCO₂eq/kWh | 0,1000 (en label del KPI) | Si |
| GEI | ~20,35 kgCO₂eq | 20,35 kgCO₂eq | Si |
| Metodologia desviacion | Neta (suma algebraica) | Neta (`real-base`, incluye negativos) | Si |

---

## 5. LAS 4 PREGUNTAS DE NEGOCIO EN EL DASHBOARD

### Pregunta 1: La planta consume mas o menos de lo esperado?

**Donde se responde:** Panel de cumplimiento (99,98%, gauge), desviacion neta (+203,49 kWh), KPI de ahorro (-0,022%)

**Respuesta visual inmediata:** El gauge casi completo + texto "OPERACION GLOBAL ESTABLE" comunica "bien en general". La desviacion neta positiva matiza: "ligeramente por encima".

### Pregunta 2: En que dias, horas, estados o modos ocurre?

**Donde se responde:** Grafico SVG con state strip (muestra cuando ocurren paradas/arranques), tooltip con fecha/hora/estado/modo exactos, slider temporal para navegar a cualquier momento, tabla de estados con selector de alcance

**Respuesta interactiva:** El usuario puede recorrer el mes entero con el slider y ver en tiempo real como cambian las metricas.

### Pregunta 3: La desviacion proviene de operacion normal, paradas o arranques?

**Donde se responde:** Tabla de estados (OPERANDO +18 kWh vs PARADO +128 kWh), panel de priorizacion (PARADO×AUTO = 62,79%)

**Respuesta directa:** La tabla muestra inequivocamente que PARADO concentra la mayor desviacion absoluta a pesar de ocupar solo 2,91% del tiempo.

### Pregunta 4: Cuanto cuesta y que accion priorizar?

**Donde se responde:** Seccion de impacto economico (configurable), panel de priorizacion con las 3 acciones ordenadas, botones "Ver evento" para investigar cada una

**Respuesta accionable:** La priorizacion es: (1) investigar PARADO×AUTOMATICO, (2) revisar transiciones PARANDO, (3) optimizar ARRANQUE.

---

## 6. PREGUNTAS PROBABLES SOBRE EL DASHBOARD

**P: Esto escalaria a multiples plantas?**
> R: Si. La arquitectura ya usa parametros: nombre de planta, tarifa, moneda, umbrales. En produccion, cada planta tendria su propia configuracion y linea base. El framework de visualizacion y alertas es reutilizable.

**P: Por que pesa 1.4 MB?**
> R: Los 43.200 registros embebidos ocupan ~1.3 MB. En produccion, estos datos vendrian de un API REST y el HTML pesaria ~50 KB. Se embebieron para que el mockup funcione como el producto real sin necesidad de backend.

**P: Como manejarias datos en tiempo real?**
> R: El mockup ya simula streaming. En produccion, usaria WebSockets o Server-Sent Events para recibir datos minuto a minuto. El renderizado es incremental: solo se agrega un punto nuevo al grafico, no se redibuja todo.

**P: Las alertas generarian fatiga?**
> R: Por eso implemento persistencia configurable. Una desviacion de 6% durante 1 minuto NO genera alerta. Solo se dispara si se mantiene durante N minutos. Los umbrales son ajustables porque cada planta tiene tolerancias diferentes.

**P: Por que SVG y no Canvas?**
> R: SVG escala sin pixelado en cualquier resolucion, es accesible para screen readers (tiene `aria-label`), y permite manipular elementos individuales (la linea de hover, los circulos, el cursor) sin redibujar todo el grafico. Canvas requiere redibujar en cada frame.

**P: Por que no tiene dark mode?**
> R: Un dashboard industrial se usa en centros de control con iluminacion controlada. El modo claro con alto contraste es la opcion mas legible para un entorno industrial. En una iteracion futura, dark mode seria un toggle de preferencia del usuario, no un requisito del MVP.

**P: El boton Exportar realmente funciona?**
> R: Si. Genera un CSV con BOM UTF-8 que incluye el resumen del periodo, cumplimiento, desviacion, y la tabla de estados completa. Se descarga automaticamente como `uptime_monitoreo_energetico.csv`.

---

## 7. DECISIONES QUE DEMUESTRAN PENSAMIENTO DE PRODUCTO

1. **No asumir tarifa** → Flexibilidad multinacional
2. **Persistencia en alertas** → Anti-fatiga operativa
3. **Notas contextuales por granularidad** → Educacion del usuario
4. **"Prioridades de investigacion, no causas raiz"** → Honestidad metodologica
5. **Botones "Ver evento"** → Ciclo dato → insight → accion
6. **localStorage para configuracion** → Persistencia entre sesiones sin backend
7. **Multi-moneda con formato local** → Internacionalizacion
8. **Exportar CSV con BOM** → Compatibilidad Excel
9. **Comparacion temporal** → Tendencia, no solo snapshot
10. **Badge LIVE con animacion** → Confianza de que el sistema esta activo

---

*Documento de sustentacion tecnica para la prueba tecnica de Product Analyst — Uptime Analytics.*
*Basado en el archivo `mockup_uptime_analytics_v5_delta_context.html` (v5, ~800 lineas, 43.200 registros embebidos).*
