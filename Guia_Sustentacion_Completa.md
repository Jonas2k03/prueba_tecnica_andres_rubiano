# Guia de Sustentacion - Prueba Tecnica Uptime Analytics

## Documento de defensa para Andres Rubiano

---

## PARTE 1: VISION GENERAL DE LA ENTREGA

### 1.1 Que se pidio vs. que se entrega

| Requisito de la prueba | Donde se cumple | Estado |
|---|---|---|
| 1. Problema de negocio | Notebook: Seccion 1 (celdas 1-6) | Completo |
| 2. Analisis de datos con linea base y 6 indicadores | Notebook: Secciones 2-7 (celdas 7-83) | Completo |
| 3. Mockup funcional de dashboard | `mockup_uptime_analytics_v5_delta_context.html` | Excede expectativas |
| 4. Video personal de 5 minutos | Pendiente de grabar | Por hacer |

### 1.2 Volumetria del dataset

- 43.200 registros (1 por minuto, junio 2026 completo)
- 21 columnas: produccion, voltaje (3 fases), corriente (3 fases), potencia activa/reactiva/aparente, factor de potencia, frecuencia, THD, estado operacional, modo operativo
- 4 estados operacionales: OPERANDO, PARADO, PARANDO, ARRANQUE
- 2 modos operativos: AUTOMATICO, MANUAL

### 1.3 Fortalezas principales de esta entrega

1. **Rigor metodologico**: no se asumen datos que no existen (tarifa electrica parametrizada, no inventada)
2. **Profundidad analitica**: desviacion segmentada por estado, modo, hora, dia y cruce estado x modo
3. **Dashboard de nivel producto**: no es un grafico estatico sino una simulacion interactiva en tiempo real con 43.200 datos embebidos
4. **Lenguaje prudente**: se habla de "condiciones a investigar", no de "causas raiz", porque el analisis estadistico identifica patrones pero no demuestra causalidad

---

## PARTE 2: ANALISIS DETALLADO DEL NOTEBOOK

### Seccion 1 - Contexto y problema de negocio (Celdas 1-6)

**Que se hizo:**
- Se define el contexto de monitoreo energetico industrial
- Se plantean las preguntas de negocio que guian el analisis
- Se establece el marco: ISO 50001, linea base energetica, indicadores de desempeno

**Como defenderlo:**
> "El analisis no comienza con codigo sino con una pregunta de negocio: la planta esta consumiendo mas o menos de lo esperado? Las siguientes preguntas nos guian por estados operacionales, fuentes de desviacion y priorizacion economica."

**Mejora menor pendiente:** En la celda 3, las preguntas 3 y 4 pueden tener un problema de formato (numeracion). Verificar que cada pregunta tenga su propio numero y parrafo claramente separado.

### Seccion 2 - Carga y exploracion de datos (Celdas 7-20 aprox.)

**Que se hizo:**
- Carga del CSV con pandas
- EDA: `.info()`, `.describe()`, distribucion de variables
- Visualizacion de produccion, potencia activa, factor de potencia
- Deteccion de correlaciones

**Como defenderlo:**
> "Antes de modelar, necesitaba entender la estructura de los datos. El EDA revelo que la potencia activa tiene alta correlacion con la produccion, lo cual valida usar la produccion como unica variable explicativa en la regresion. Tambien identifique que los estados no-operando representan una fraccion menor del tiempo pero concentran la mayor parte de la desviacion."

### Seccion 3 - Linea base energetica (Celdas ~21-35)

**Que se hizo:**
- Regresion lineal simple: `Energia (kWh) = 0.159311 * Produccion (t/h) + 2.002463`
- R² = 0.9692 (96.92% de la varianza explicada)
- Energia real por minuto: `Potencia Activa (kW) / 60`
- Calculo de desviacion: `Energia_real - Energia_base`

**Como defenderlo:**
> "La linea base sigue el enfoque de ISO 50006: un modelo estadistico que predice el consumo esperado en funcion de un driver relevante, en este caso la produccion. Un R² de 0.97 indica un ajuste excelente. La conversion de potencia a energia por minuto (kW/60 = kWh) es fisicamente correcta porque cada registro representa exactamente un minuto."

**Pregunta probable del entrevistador:**
- *"Por que regresion lineal y no un modelo mas complejo?"*
  > "Con R²=0.97, la relacion es claramente lineal. Un modelo mas complejo no aportaria precision significativa y seria mas dificil de interpretar para un operador de planta. En gestion energetica industrial, la interpretabilidad del modelo es clave para la adopcion."

### Seccion 4 - Indicadores de desempeno (6 KPIs del Anexo 3)

**Los 6 indicadores y como se calcularon:**

| # | Indicador | Formula | Valor junio |
|---|---|---|---|
| 1 | Cumplimiento (%) | `(Energia_base / Energia_real) * 100` | ~99.98% |
| 2 | Desempeno energetico (kWh) | `Suma(Energia_real) - Suma(Energia_base)` | +203.49 kWh |
| 3 | Emisiones GEI (kgCO2eq) | `Desempeno_kWh * 0.1000` | ~20.35 kgCO2eq |
| 4 | Desempeno economico ($) | `Desempeno_kWh * tarifa` | **Parametrizado** |
| 5 | Ahorro (%) | `1 - (Energia_real / Energia_base)` | ~-0.02% |
| 6 | Indice de consumo (kWh/t) | `Energia_real / Produccion` | Calculado por minuto |

**Decision clave: Tarifa electrica parametrizada**

Este es uno de los puntos mas fuertes de la entrega. El codigo implementa:

```python
TARIFA_ELECTRICA = None

def calcular_desempeno_economico(tarifa_electrica):
    return desempeno_kwh * tarifa_electrica

if TARIFA_ELECTRICA is not None:
    desempeno_economico = calcular_desempeno_economico(TARIFA_ELECTRICA)
else:
    desempeno_economico = None  # Se muestra como "Parametrizado"
```

**Como defenderlo:**
> "El dataset no incluye informacion de tarifa electrica, y asumir un valor seria metodologicamente incorrecto. Una planta puede tener tarifa regulada, no regulada, con componente de demanda, con penalizacion por factor de potencia, con banda horaria, etc. En lugar de inventar un numero, diseñe una funcion parametrica que el usuario puede invocar con la tarifa real de su contrato. Esto es lo que haria un producto SaaS real: el administrador configura la tarifa en settings, no viene hardcodeada."

**Pregunta probable:**
- *"Pero asi no muestras el impacto economico..."*
  > "Correcto, y eso es intencional. Mostrar un impacto en pesos con una tarifa inventada es peor que no mostrarlo, porque genera una falsa sensacion de precision. En el dashboard si incluyo un campo donde el usuario ingresa la tarifa y la moneda, y el calculo se activa inmediatamente. Eso demuestra la funcionalidad sin comprometer la metodologia."

### Seccion 5 - Diagnostico de desviacion (Celdas ~40-65)

**Que se hizo:**
- Segmentacion de desviacion por estado operacional (OPERANDO, PARADO, PARANDO, ARRANQUE)
- Segmentacion por modo operativo (AUTOMATICO, MANUAL)
- Segmentacion por hora del dia
- Segmentacion por dia de la semana (con promedios por fecha para evitar sesgo por dias desiguales)
- Cruce estado x modo operativo

**Hallazgo central:**
- PARADO x AUTOMATICO concentra el **62.79%** de la desviacion neta total (+127.77 kWh)
- 90.91% de la desviacion ocurre fuera de operacion normal
- OPERANDO solo aporta el 9.09% restante

**Funcion reutilizable para priorizacion:**

```python
def resumen_desviacion(dataframe, columnas_grupo):
    resumen = dataframe.groupby(columnas_grupo, observed=True).agg(
        Minutos=("Desviacion (kWh)", "size"),
        Desviacion_neta_kWh=("Desviacion (kWh)", "sum"),
        Desviacion_media_kWh=("Desviacion (kWh)", "mean")
    ).reset_index()
    resumen["Peso_desviacion_pct"] = (resumen.Desviacion_neta_kWh / resumen.Desviacion_neta_kWh.sum()) * 100
    return resumen
```

**Como defenderlo:**
> "Cree una funcion generica `resumen_desviacion()` que recibe cualquier combinacion de columnas de agrupacion. Esto me permitio segmentar por estado, por modo, por hora, por dia, y por el cruce estado-modo con la misma logica. El hallazgo clave es que 63% de la desviacion se concentra en PARADO x AUTOMATICO: la planta esta parada pero los sistemas automaticos siguen consumiendo energia. Esto sugiere que hay equipos auxiliares que no se apagan correctamente durante las paradas."

**Metodologia de desviacion neta:**
> "Uso desviacion neta (suma algebraica, incluyendo valores negativos) y no solo la suma de desviaciones positivas. Esto es correcto porque en gestion energetica, un minuto donde consumes menos de lo esperado compensa parcialmente un minuto donde consumes mas. La desviacion neta refleja el impacto real acumulado."

**Analisis por dia de la semana:**
> "Para el analisis por dia de la semana, uso el promedio de desviacion por fecha (no la suma directa), porque junio no tiene el mismo numero de lunes, martes, etc. Si sumara directamente, los dias con mas ocurrencias parecerian tener mas desviacion simplemente por tener mas datos."

### Seccion 6 - Grafico de priorizacion (Scatter plot)

**Que se hizo:**
- Grafico de dispersion: impacto total (kWh) vs. intensidad (kWh/minuto) por condicion
- Permite visualizar que condiciones son prioritarias por volumen vs. por severidad

**Como defenderlo:**
> "Este grafico responde a la pregunta: donde deberia enfocar mis recursos? Una condicion puede tener alta desviacion total pero baja intensidad (ocurre mucho tiempo con poca desviacion por minuto), o viceversa. PARADO x AUTOMATICO aparece en el cuadrante superior derecho: alto impacto total Y alta intensidad, lo que lo convierte en la prioridad #1."

### Seccion 7 - Hallazgos y conclusiones (Celdas ~78-84)

**Estructura de cada hallazgo:**
Cada hallazgo incluye una "Implicacion de producto", lo cual demuestra pensamiento orientado a producto, no solo analitico.

**Hallazgos clave y como presentarlos:**

1. **"La planta cumple globalmente (~100%), pero eso esconde problemas localizados."**
   > Implicacion: Un dashboard que solo muestre cumplimiento mensual no seria util. Se necesita drill-down temporal y por estado.

2. **"62.79% de la desviacion se concentra en PARADO x AUTOMATICO."**
   > Implicacion: El producto deberia alertar automaticamente cuando una parada excede cierto umbral de consumo.

3. **"La desviacion durante operacion normal es minima (9.09%)."**
   > Implicacion: El modelo de linea base es preciso para operacion normal. Las alertas deben calibrarse diferenciadamente por estado.

4. **"PARANDO y ARRANQUE, aunque breves, tienen alta intensidad de desviacion."**
   > Implicacion: Merecen dashboards especificos de transicion, no solo acumulados.

5. **"No se puede determinar causa raiz solo con los datos disponibles."**
   > Esto es clave: el analista identifica patrones pero no inventa explicaciones. Las causas requieren inspeccion en campo.

---

## PARTE 3: ANALISIS DETALLADO DEL DASHBOARD

### 3.1 Vision general

El dashboard NO es un grafico estatico. Es una **simulacion interactiva en tiempo real** que:
- Embebece los 43.200 registros del dataset completo
- Simula un flujo de datos en vivo con animaciones
- Permite explorar cualquier minuto del mes con un slider temporal
- Calcula metricas dinamicamente segun la ventana y granularidad seleccionada
- Implementa un sistema de alertas configurable

### 3.2 Componentes del dashboard

**Barra superior:**
- Logo Uptime Analytics
- Titulo: "Monitoreo de desempeno energetico - Planta 01"
- Badge LIVE con punto pulsante y reloj en tiempo real
- Botones: Exportar, Alertas

**Seccion animada de produccion:**
- Icono de engranaje giratorio (CSS animation)
- Produccion en vivo (t/h) con estado de planta
- Cinta transportadora animada que fluye cuando la planta opera
- Estado: "OPERANDO - AUTOMATICO" / "PARADO" etc.

**Telemetria en tiempo real:**
- Consumo por minuto (kWh)
- Linea base (kWh)
- Factor de potencia
- Se actualizan con cada minuto seleccionado

**Panel de cumplimiento:**
- Gauge circular (donut) con porcentaje de cumplimiento (~100%)
- Desviacion neta acumulada del mes (+203.49 kWh)
- Porcentaje dentro de ±5%
- Impacto economico: muestra "Sin tarifa" hasta que el usuario configure una
- Deltas vs. mes anterior: "No disponible" (solo hay datos de junio)

**Panel de desviacion actual:**
- Muestra el minuto seleccionado con su desviacion
- Chips de estado y modo operativo
- Mensaje contextual explicativo
- Codigo de color segun umbrales

**Controles interactivos:**
- Modo: Ahora / Explorar historico
- Slider temporal (0 a 43.199 minutos)
- Granularidad: 1 min, 15 min, 1 h, Turno, Dia, Mes
- Notas contextuales que cambian segun la granularidad seleccionada

**Grafico principal:**
- Canvas HTML5 (no libreria externa)
- Consumo real vs. linea base
- Tooltip interactivo
- Resumen del rango seleccionado

**Tabla de estado operacional:**
- Comportamiento por estado con selector de alcance temporal
- Columnas: Estado, Tiempo, Desviacion, Impacto/min, Desv. energetica, Impacto economico
- Se recalcula dinamicamente segun el periodo seleccionado

**Centro de alertas:**
- Hero banner: estado actual de alertas
- Contadores: Criticas mes, Atencion mes, Activas ahora
- Tabs: Recientes, Criticas, Atencion
- Cada alerta: titulo, descripcion, severidad, tags, periodo
- Alertas basadas en reglas configurables

**Seccion de impacto economico:**
- Campo para ingresar costo por kWh
- Selector de moneda: COP, USD, EUR, MXN, BRL, PEN, CLP
- Boton "Aplicar" que activa el calculo
- Muestra: Impacto junio e Impacto del periodo seleccionado
- Formula explicita visible

**Panel de priorizacion:**
- Top 3 prioridades de investigacion:
  1. PARADO x AUTOMATICO: 62.79% (rojo)
  2. PARANDO: 14.45% (ambar)
  3. ARRANQUE: 13.51% (ambar)
- Cada tarjeta con boton "Ver evento" que navega al periodo correspondiente
- Nota: "son prioridades de investigacion, no causas raiz demostradas"

**Modal de configuracion de alertas:**
- Umbral preventivo (%)
- Umbral critico (%)
- Persistencia de desviacion (min)
- Parada: alertar despues de (min)
- Modo manual: alertar despues de (min)
- Moneda predeterminada

### 3.3 Decisiones de diseño y como defenderlas

**"Por que embeber 43.200 datos en el HTML?"**
> "La prueba pide un mockup funcional. Un HTML estatico con datos falsos no demuestra la capacidad de la plataforma. Al embeber el dataset completo, el dashboard se comporta como el producto real: puedes navegar cualquier minuto de junio, cambiar granularidades, y todas las metricas se recalculan en tiempo real. Esto demuestra como seria la experiencia de usuario de Uptime Analytics."

**"Por que no usar una libreria de graficos como Chart.js?"**
> "Dibuje el grafico directamente con Canvas API para mantener el HTML completamente autocontenido, sin dependencias externas. Esto garantiza que el archivo funciona en cualquier navegador sin conexion a internet."

**"Por que un sistema de alertas?"**
> "Un dashboard de monitoreo energetico sin alertas es un poster. En produccion real, nadie mira un dashboard todo el dia. El valor esta en que el sistema detecte automaticamente condiciones que requieren atencion: paradas prolongadas, desviaciones criticas sostenidas, modo manual extendido. Las reglas son configurables porque cada planta tiene umbrales diferentes."

**"Por que la tarifa es configurable en vez de mostrar un valor?"**
> "Igual que en el notebook: no se asume una tarifa porque seria metodologicamente incorrecto. El dashboard permite al administrador de la planta ingresar su tarifa real y seleccionar su moneda. Eso es lo que haria un producto SaaS que opera en multiples paises (COP, USD, EUR, MXN, BRL, PEN, CLP estan soportados)."

**"Por que las notas contextuales?"**
> "Cada nivel de granularidad tiene una interpretacion diferente. Una desviacion de 5% en un minuto es ruido; en un turno es una señal. Las notas contextuales educan al usuario para no sobrerreaccionar a variaciones puntuales. Esto refleja una comprension profunda de como se usa un producto de monitoreo en la practica."

### 3.4 Detalles tecnicos destacables

- **Responsivo**: `@media(max-width:1080px)` adapta el layout a pantallas menores
- **Sin dependencias externas**: todo CSS inline, canvas nativo, sin CDN
- **Animaciones CSS puras**: engranaje, cinta transportadora, punto pulsante LIVE
- **Datos contextuales dinamicos**: los mensajes cambian segun estado, modo y granularidad
- **Multi-moneda**: preparado para operacion multinacional

---

## PARTE 4: LAS 4 PREGUNTAS DE NEGOCIO

### Pregunta 1: La planta esta consumiendo mas o menos de lo esperado?

**Donde se responde:**
- Notebook: Indicador de cumplimiento (~99.98%), desempeno energetico (+203.49 kWh neto)
- Dashboard: Gauge de cumplimiento, tarjeta de desviacion neta

**Respuesta:**
> "Globalmente, la planta cumple al ~100%. Pero eso esconde una realidad mas matizada: la desviacion neta es positiva (+203.49 kWh), lo que significa que en el acumulado del mes se consumio ligeramente mas de lo esperado. La clave es que esta desviacion NO se distribuye uniformemente."

### Pregunta 2: En que dias, horas, estados o modos ocurre?

**Donde se responde:**
- Notebook: Seccion 5 completa - segmentacion por estado, modo, hora, dia, cruce estado x modo
- Dashboard: Tabla de estado operacional, granularidades, slider temporal

**Respuesta:**
> "La desviacion se concentra en estados no-operativos (90.91% fuera de OPERANDO). Por hora, hay picos en ciertos periodos que coinciden con arranques y paradas. Por dia, el analisis usa promedios por fecha para evitar sesgo. El cruce estado x modo revela que PARADO x AUTOMATICO es la condicion critica."

### Pregunta 3: La desviacion proviene de la operacion normal, de paradas o de arranques?

**Donde se responde:**
- Notebook: Tabla de `resumen_desviacion()` por estado operacional
- Dashboard: Panel de priorizacion, tabla de estado

**Respuesta:**
> "OPERANDO solo aporta 9.09% de la desviacion. PARADO aporta 62.95%, PARANDO 14.45%, ARRANQUE 13.51%. La operacion normal esta bien controlada por la linea base. Los problemas estan en los estados transitorios y de inactividad."

### Pregunta 4: Cuanto cuesta y que accion deberia priorizarse?

**Donde se responde:**
- Notebook: Funcion `calcular_desempeno_economico(tarifa)`, scatter plot de priorizacion
- Dashboard: Seccion de impacto economico configurable, panel de priorizacion

**Respuesta:**
> "El costo depende de la tarifa que aplique cada planta (parametrizado). La priorizacion es clara: (1) investigar el consumo durante PARADO x AUTOMATICO, (2) revisar las secuencias de PARANDO, (3) optimizar los procedimientos de ARRANQUE. Esta priorizacion se basa en impacto total e intensidad, no solo en una de las dos."

---

## PARTE 5: DIFERENCIADORES COMPETITIVOS

### Lo que esta entrega hace mejor que un candidato tipico

1. **No inventa datos**: La tarifa parametrizada demuestra madurez profesional. Un junior asume; un senior parametriza.

2. **Distingue correlacion de causalidad**: Dice "condiciones a investigar" en lugar de "causas". Sabe que un patron estadistico señala donde mirar, no por que ocurre.

3. **Piensa en producto**: Los hallazgos incluyen "implicacion de producto", no solo "dato interesante". Eso conecta el analisis con la hoja de ruta del producto Uptime Analytics.

4. **Dashboard de nivel produccion**: No es un grafico de Matplotlib exportado a HTML. Es un prototipo interactivo que demuestra como podria verse el producto real, con datos reales.

5. **Rigurosidad metodologica**: Promedios por fecha para dias desiguales, desviacion neta (no solo positiva), `observed=True` en groupby, factor de emision documentado.

6. **Coherencia notebook-dashboard**: La priorizacion del notebook (PARADO x AUTOMATICO = 62.79%) aparece identica en el dashboard. Los 6 indicadores del Anexo 3 estan presentes en ambos. La tarifa parametrica es consistente.

---

## PARTE 6: AREAS DE MEJORA MENORES

### 6.1 En el notebook

1. **Formato de preguntas (celda 3)**: Verificar que las preguntas 3 y 4 esten claramente numeradas y separadas. Si hay un salto de linea faltante, corregirlo.

2. **Marco de referencia explicito**: Podria añadirse una mencion breve a ISO 50001 / ISO 50006 como marco normativo de la linea base y los indicadores. No es obligatorio, pero refuerza la credibilidad.

3. **Resumen ejecutivo al inicio**: Considerar agregar una celda despues de la introduccion con un resumen de 3-4 balas con los hallazgos principales, para que quien revise el notebook pueda captar el valor en los primeros 30 segundos.

4. **Numero total de preguntas**: Se redujeron de 10 a 5, lo cual esta bien por ser mas enfocado, pero asegurarse de que todas sean preguntas realmente respondidas en el analisis.

### 6.2 En el dashboard

1. **Datos embebidos**: El archivo pesa ~1.3 MB por tener los 43.200 registros en JSON inline. Esto es aceptable para un mockup, pero mencionarlo proactivamente: "en produccion, estos datos vendrian de un API".

2. **Fuente Inter**: Se referencia la fuente Inter sin cargarla desde un CDN (correcto para autocontencion). Si la fuente no esta instalada en el sistema, usara las fallback del sistema. Funciona correctamente.

3. **Exportar**: El boton "Exportar" es un mockup. Si preguntan, explicar que en produccion exportaria a CSV/PDF con las metricas del rango seleccionado.

### 6.3 Mejoras opcionales (si hay tiempo)

- Agregar una celda en el notebook que genere una tabla-resumen de los 6 indicadores lado a lado para tener una vista compacta
- Incluir un test estadistico de normalidad de residuos en la regresion (Shapiro-Wilk o similar) para reforzar la validez del modelo lineal
- Agregar una nota sobre multicolinealidad para justificar por que solo se usa produccion como variable (y no voltaje, corriente, etc.)

---

## PARTE 7: GUION PARA EL VIDEO DE 5 MINUTOS

### Estructura recomendada

**[0:00 - 0:30] Presentacion y contexto (30 seg)**
> "Hola, soy Andres Rubiano. Les presento mi analisis de desempeno energetico para una planta industrial, usando los datos de junio 2026. El objetivo es responder una pregunta simple pero critica: la planta esta consumiendo lo que deberia?"

**[0:30 - 1:30] Linea base y hallazgo global (60 seg)**
> "Construi una linea base de energia usando regresion lineal contra la produccion, con un R² de 0.97. El cumplimiento global del mes es cercano al 100%, lo que suena bien... pero esconde algo importante."
- Mostrar la regresion y el R² en el notebook
- Mostrar el gauge de cumplimiento en el dashboard

**[1:30 - 3:00] El hallazgo clave: donde esta la desviacion (90 seg)**
> "Al segmentar por estado operacional, descubri que el 91% de la desviacion ocurre fuera de la operacion normal. Y al cruzar estado con modo operativo, PARADO por AUTOMATICO concentra el 63% de toda la desviacion. Esto significa que cuando la planta esta parada, los sistemas automaticos siguen consumiendo energia de forma significativa."
- Mostrar la tabla de resumen_desviacion por estado
- Mostrar el cruce estado x modo
- Mostrar el panel de priorizacion en el dashboard

**[3:00 - 4:00] Dashboard en accion (60 seg)**
> "Para comunicar estos hallazgos, diseñe un dashboard interactivo que simula monitoreo en tiempo real con los 43.200 registros del mes completo."
- Demo en vivo: cambiar granularidad, explorar historico, mostrar alertas
- Mostrar la configuracion de tarifa y como se activa el calculo economico
- Mostrar como las alertas detectan paradas prolongadas

**[4:00 - 4:45] Decisiones de diseño (45 seg)**
> "Dos decisiones importantes: primero, no asumi tarifa electrica porque cada planta tiene un contrato diferente. En su lugar, diseñe un campo parametrico donde el administrador ingresa la tarifa real. Segundo, las conclusiones hablan de condiciones a investigar, no de causas raiz, porque el analisis estadistico identifica donde mirar, pero la causa requiere inspeccion en campo."

**[4:45 - 5:00] Cierre (15 seg)**
> "En resumen: la planta cumple globalmente, pero tiene una oportunidad concreta de ahorro en las paradas automaticas. El dashboard propuesto permite detectar esto en tiempo real y priorizar acciones. Gracias."

### Tips para la grabacion

- Compartir pantalla con el notebook abierto Y el dashboard en otro tab
- No leer un guion: usar las balas como recordatorio pero hablar naturalmente
- Hacer la demo del dashboard en vivo, no como screenshot
- Si es posible, empezar con el dashboard para captar atencion y luego ir al notebook para mostrar el "como"
- Hablar con confianza pero sin arrogancia. "Descubri" no "encontre un insight increible"

---

## PARTE 8: PREGUNTAS FRECUENTES DE ENTREVISTA

### Sobre la metodologia

**P: Por que regresion lineal y no un modelo mas complejo?**
> R: Con R²=0.97, la relacion es lineal y el modelo explica casi toda la varianza. Un modelo mas complejo aportaria complejidad sin precision significativa, y en gestion energetica la interpretabilidad es clave para que los operadores confien en la linea base.

**P: Por que no usaste todas las variables (voltaje, corriente, etc.)?**
> R: La produccion tiene la correlacion mas alta con el consumo y es el driver operativo principal. Las variables electricas (voltaje, corriente) son consecuencia del consumo, no causas. Incluirlas generaria multicolinealidad sin mejorar el modelo.

**P: Que pasa si la linea base no es lineal?**
> R: Verificaria con un analisis de residuos. Si los residuos muestran patron no aleatorio, exploraria modelos polinomiales o segmentados. Pero con R²=0.97, los residuos deben ser aproximadamente aleatorios.

### Sobre los datos

**P: Como manejas los datos de PARADO donde produccion es 0?**
> R: Cuando la produccion es 0, la linea base predice un consumo base de 2.00 kWh (el intercepto del modelo). Si la planta consume mas que eso durante una parada, la desviacion es positiva. Esto es exactamente lo que detectamos: equipos auxiliares que siguen consumiendo.

**P: Los datos tienen outliers?**
> R: El EDA incluye analisis de distribucion. Los "outliers" en este contexto generalmente son eventos operativos reales (arranques, paradas), no errores de medicion. Por eso los analizo en lugar de eliminarlos.

**P: Por que desviacion neta y no solo la positiva?**
> R: La desviacion neta refleja el impacto real acumulado. Si un minuto consumo 0.5 kWh mas y el siguiente 0.5 kWh menos, el impacto neto es cero. Sumar solo las positivas sobreestimaria el problema. Para priorizacion de acciones correctivas, la neta es la metrica correcta.

### Sobre el dashboard

**P: Esto escalaria a multiples plantas?**
> R: Si. La arquitectura del mockup ya usa parametros (planta, tarifa, moneda, umbrales). En produccion, cada planta tendria su propia configuracion y linea base, pero el framework de visualizacion y alertas seria reutilizable.

**P: Como manejarias datos en tiempo real?**
> R: El mockup ya simula streaming con los datos historicos. En produccion, usaria WebSockets o Server-Sent Events para recibir datos minuto a minuto. El Canvas API que uso ya soporta actualizacion incremental.

**P: Las alertas generarian fatiga?**
> R: Por eso implemento persistencia: una alerta solo se dispara si la condicion se mantiene durante N minutos configurables. Un pico de un solo minuto no genera alerta. Los umbrales son ajustables por planta porque cada operacion tiene diferentes tolerancias.

### Sobre decisiones de producto

**P: Que funcionalidad agregarias primero si este fuera el MVP?**
> R: Notificaciones push/email cuando se dispara una alerta critica. Un dashboard que nadie mira no tiene valor. El segundo paso seria comparacion entre periodos (este turno vs. el anterior) para detectar degradacion gradual.

**P: Como validarias que la linea base sigue siendo correcta con el tiempo?**
> R: Monitorearia el R² y los residuos de forma continua. Si el R² cae por debajo de un umbral (ej. 0.90), el sistema deberia alertar para recalibrar la linea base. Esto podria ocurrir si la planta cambia equipos, procesos o materia prima.

---

## PARTE 9: CHECKLIST FINAL

### Antes de enviar

- [ ] Notebook ejecuta de principio a fin sin errores (Kernel > Restart & Run All)
- [ ] Las 6 indicadores del Anexo 3 aparecen claramente identificados
- [ ] El dashboard carga correctamente en Chrome/Edge/Firefox
- [ ] Las preguntas de negocio 1-4 se responden explicitamente
- [ ] El formato de las preguntas en celda 3 es correcto (numeracion clara)
- [ ] No hay celdas con output de error o warnings visibles
- [ ] El video tiene maximo 5 minutos
- [ ] El video muestra tanto el notebook como el dashboard en accion

### Para la entrevista

- [ ] Tener el notebook abierto y ejecutado
- [ ] Tener el dashboard abierto en un tab aparte
- [ ] Poder explicar: por que regresion lineal, por que tarifa parametrizada, por que desviacion neta
- [ ] Poder navegar el dashboard cambiando granularidad y explorando historico
- [ ] Conocer los numeros clave: R²=0.97, Cumplimiento ~100%, PARADO x AUTO = 62.79%

---

*Documento preparado para la sustentacion de la prueba tecnica de Product Analyst - Uptime Analytics.*
*Basado en el notebook de 84 celdas y el dashboard v5 con datos de junio 2026.*
