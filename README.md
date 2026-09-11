# Agente-Workflow_CAIS

Workflow de **n8n** para el manejo dinámico de tarifas de transporte: extrae datos crudos tipo "Big Data", los limpia y valida con un agente de IA, analiza el desempeño de las tarifas (Completion Rate) con otro agente de IA, pasa por un proceso de **Human in the Loop (HITL)** cuando detecta anomalías, calcula la tarifa final con una fórmula de negocio y publica el resultado en un dashboard sobre PostgreSQL.

Archivo del workflow: [`Manejo de Tarifas - Limpieza, Analisis, HITL y Dashboard (2).json`](<Manejo de Tarifas - Limpieza, Analisis, HITL y Dashboard (2).json>)

## Tabla de contenido

- [Visión general](#visión-general)
- [Disparadores](#disparadores)
- [Control de límite diario](#control-de-límite-diario)
- [1. Extracción y limpieza de datos](#1-extracción-y-limpieza-de-datos)
- [2. Guardado y consulta](#2-guardado-y-consulta)
- [3. Análisis de tarifas](#3-análisis-de-tarifas)
- [4. Umbral de auto-aprobación y HITL](#4-umbral-de-auto-aprobación-y-hitl)
- [5. Cálculo de tarifa final (Fórmula Magellan)](#5-cálculo-de-tarifa-final-fórmula-magellan)
- [6. Publicación en el dashboard](#6-publicación-en-el-dashboard)
- [Tablas de PostgreSQL utilizadas](#tablas-de-postgresql-utilizadas)
- [Modelos de IA y resiliencia (fallback)](#modelos-de-ia-y-resiliencia-fallback)
- [Notificaciones](#notificaciones)
- [Configuración del workflow](#configuración-del-workflow)
- [Requisitos previos](#requisitos-previos)
- [Cómo importar y ejecutar](#cómo-importar-y-ejecutar)

## Visión general

El workflow se llama **"Manejo de Tarifas - Limpieza, Analisis, HITL y Dashboard"** y automatiza el ciclo completo de revisión de tarifas dinámicas de transporte (estilo ride-hailing) por ciudad, producto y franja horaria:

```
Disparador ──▶ Límite diario ──▶ Extraer Big Data ──▶ Agente Limpieza (IA)
   ──▶ Guardar en BD ──▶ Consultar parámetros vigentes + datos del día
   ──▶ Agente Análisis (IA) ──▶ ¿Hay anomalías?
         │no                         │sí
         ▼                           ▼
   Calcular tarifa            Notificar Slack ──▶ HITL (formulario) ──▶ ¿Aprobado?
         │                                              │sí          │no
         ▼                                              ▼            ▼
   Guardar en Dashboard ◀── Calcular tarifa       Fin - Rechazado
         │
         ▼
   Fin - Publicado
```

## Disparadores

El workflow puede iniciarse de dos formas:

- **Inicio Manual** (`manualTrigger`): para pruebas o ejecuciones ad-hoc.
- **Disparador Diario** (`scheduleTrigger`): ejecución automática programada.

Ambos convergen en el nodo **Verificar Límite Diario**.

## Control de límite diario

Antes de gastar llamadas a los modelos de IA, el workflow protege el costo:

- **Verificar Limite Diario** (Postgres, `executeQuery`): hace un `UPSERT` sobre `control_limites`, incrementando el contador de ejecuciones (`ejecuciones`) y de llamadas LLM estimadas (`llamadas_llm_estimadas`) para el día actual.
- **¿Excede Limite Diario?** (`if`): si `ejecuciones > 20` **o** `llamadas_llm_estimadas > 50`, el workflow se detiene.
  - **Detenido - Limite Alcanzado** (`code`): lanza un error explicando el límite alcanzado (ajustable directamente en el nodo).
  - Caso contrario, continúa hacia la extracción de datos.

## 1. Extracción y limpieza de datos

- **Extraer Big Data (origen)** (Postgres `select`, tabla `bigdata_tarifas_raw`): trae todas las filas crudas que simulan el origen de Big Data.
- **Agrupar Filas Crudas** (`code`): agrupa todos los items en un solo objeto `{ filas: [...] }` para enviarlo al agente en un solo prompt.
- **Agente Limpieza de Datos** (`@n8n/n8n-nodes-langchain.agent`): agente de IA (Google Gemini) que valida cada fila:
  - Duplicados por `codigo_ciudad + codigo_producto + fecha + hora`.
  - Horas fuera de rango (0-23).
  - Métricas negativas o nulas en campos obligatorios (`eyeballs`, `calls`, `trips`).
  - Proporciones (`cr`, `ecr`, `etr`, `ar`, `dar`, `oar`) fuera del rango 0-1.
  - Si algo no se puede validar con certeza, la fila se marca como error (no se adivina).
  - Salida forzada por **Parser Salida Limpieza** (`outputParserStructured`, con `autoFix`) al formato `{"datos_limpios": [...], "errores": [...]}`.
  - Ante error del agente principal (`onError: continueErrorOutput`), se reintenta con **Agente Limpieza de Datos (Fallback)**.
- **Separar Tarifas Limpias** (`splitOut` sobre `output.datos_limpios`): convierte el arreglo de filas limpias en items individuales.
- **Preparar Registro Errores** / **Registrar Errores en BD**: registra en `log_errores_limpieza` los errores detectados por el agente en cada ejecución.

## 2. Guardado y consulta

- **Guardar en Database** (Postgres `insert`, tabla `tarifas_datos`): inserta cada fila limpia.
- **Consultar Parámetros Vigentes** (Postgres `select`, tabla `parametros_tarifa_vigentes`, `executeOnce`): trae los parámetros de tarifa actuales (base_fare, distance_fare, time_fare, longdistance_fare, fuel_surcharge, tarifa_minima) por ciudad/producto.
- **Consultar Datos** (Postgres `select`, tabla `tarifas_datos`, filtrando por `fecha = hoy`, `executeOnce`): trae las métricas ya guardadas del día para analizarlas.
- **Combinar Métricas y Parámetros** (`code`): junta ambos resultados en un solo objeto `{ metricas: [...], parametros_vigentes: [...] }` para pasarlo al agente analista.

## 3. Análisis de tarifas

- **Agente Análisis Tarifas** (`@n8n/n8n-nodes-langchain.agent`): agente de IA experto en pricing dinámico. Evalúa el **CR (Completion Rate = trips / calls)** por ciudad+producto+franja horaria con esta regla de negocio:
  - CR > 80% → posible sub-oferta → recomienda **bajar** `base_fare` o `time_fare` entre 5% y 15%.
  - CR < 60% → posible sobre-oferta / precio ahuyenta demanda → recomienda **subir** ese mismo componente entre 5% y 15%.
  - CR entre 60% y 80% → rango normal, no genera ajuste.
  - Nunca modifica impuestos ni parámetros fijos externos.
  - Salida forzada por **Parser Salida Analisis** al formato:
    ```json
    {
      "resumen": "...",
      "riesgo_principal": "...",
      "anomalias": ["..."],
      "recomendaciones": ["..."],
      "ajustes_propuestos": [
        {
          "codigo_ciudad": "...",
          "nombre_ciudad": "...",
          "codigo_producto": "...",
          "hora_inicio": 0,
          "hora_fin": 0,
          "cr_observado": 0.0,
          "componente_ajustado": "base_fare|distance_fare|time_fare|longdistance_fare|fuel_surcharge",
          "ajuste_pct": 0
        }
      ]
    }
    ```
  - Ante error del agente principal, se reintenta con **Agente Analisis Tarifas (Fallback)**.

## 4. Umbral de auto-aprobación y HITL

- **¿Hay Anomalías?** (`if`): evalúa `output.ajustes_propuestos.length > 0`.
  - **Sin anomalías** → auto-aprobación directa → va a **Calcular Tarifa (Formula Magellan)**.
  - **Con anomalías** → requiere revisión humana:
    1. **Notificar HITL (Slack)**: publica en el canal `all-cais` un mensaje con el link de reanudación (`$execution.resumeUrl`).
    2. **Revision Humana (HITL)** (`n8n-nodes-base.wait`, `resume: form`): formulario "Revisión de Análisis de Tarifas" con:
       - Resumen del análisis (prellenado automáticamente con resumen, riesgo principal, anomalías y recomendaciones).
       - Decisión: `aprobado` / `rechazado` (obligatorio).
       - Comentarios (opcional).
    3. **Aprobado?** (`if`): si `Decisión == aprobado` → continúa a **Calcular Tarifa (Formula Magellan)**; si no → **Fin - Rechazado** (`noOp`).

## 5. Cálculo de tarifa final (Fórmula Magellan)

**Calcular Tarifa (Formula Magellan)** (`code`) combina los parámetros de tarifa vigentes con el ajuste propuesto por el agente (si aplica) para cada combinación ciudad+producto:

1. Toma `base_fare`, `distance_fare`, `time_fare`, `longdistance_fare`, `fuel_surcharge` y `tarifa_minima` vigentes.
2. Si hay un ajuste propuesto para esa ciudad/producto, aplica el porcentaje (`ajuste_pct`) únicamente al `componente_ajustado` indicado.
3. Calcula la tarifa final con la fórmula:

   ```
   tarifa_calculada = max(tarifa_minima, base_fare + distance_fare + time_fare + longdistance_fare + fuel_surcharge)
   ```

4. Si no hubo anomalías, el registro queda marcado como `"Auto-aprobado: sin anomalias detectadas por el agente"`; si pasó por HITL, se guarda el comentario del revisor.

## 6. Publicación en el dashboard

**Guardar en Dashboard** (Postgres `insert`, tabla `dashboard_tarifas`): inserta una fila nueva por cada combinación ciudad/producto con el detalle completo (parámetros, ajuste aplicado, resumen del análisis, anomalías, recomendaciones y tarifa calculada). No se leen ni reescriben archivos, evitando condiciones de carrera.

- **Fin - Publicado** (`noOp`): fin exitoso del flujo.
- **Fin - Rechazado** (`noOp`): fin cuando el revisor humano rechaza el análisis.

## Tablas de PostgreSQL utilizadas

| Tabla | Uso |
|---|---|
| `bigdata_tarifas_raw` | Origen de datos crudos que simula el ingreso de Big Data. |
| `tarifas_datos` | Datos ya limpios y validados por el agente de limpieza. |
| `parametros_tarifa_vigentes` | Parámetros de tarifa actuales por ciudad/producto (componentes de la fórmula). |
| `log_errores_limpieza` | Registro de errores detectados en cada ejecución de limpieza. |
| `control_limites` | Contador diario de ejecuciones y llamadas LLM estimadas, para controlar costo. |
| `dashboard_tarifas` | Tabla final que alimenta el dashboard, con la tarifa calculada y el contexto del análisis. |

## Modelos de IA y resiliencia (fallback)

Todos los agentes usan **Google Gemini** (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) como modelo de lenguaje, cada uno con su propia credencial y `retryOnFail` configurado. Para tolerancia a fallos, cada agente principal tiene un agente y modelo de respaldo (fallback) con el mismo prompt/lógica, que se activa automáticamente si el agente principal falla:

- **Agente Limpieza de Datos** → **Agente Limpieza de Datos (Fallback)** (modelo `models/gemini-3.1-flash-lite`).
- **Agente Analisis Tarifas** → **Agente Analisis Tarifas (Fallback)** (modelo `models/gemini-3.1-flash-lite`).

## Notificaciones

- **Notificar HITL (Slack)**: envía un mensaje al canal `all-cais` cuando un análisis con anomalías queda pendiente de revisión humana, incluyendo el link directo para aprobar/rechazar.

## Configuración del workflow

- `executionOrder`: v1
- `callerPolicy`: `workflowsFromSameOwner`
- `availableInMCP`: true
- `saveExecutionProgress`: true
- `executionTimeout`: 600 segundos

## Requisitos previos

- Instancia de **n8n** con soporte para nodos de LangChain (`@n8n/n8n-nodes-langchain`).
- Credenciales configuradas en n8n:
  - **Postgres** (una o varias cuentas, según se use en cada nodo).
  - **Google Gemini (PaLM) API** (una credencial por agente/fallback).
  - **Slack API** (para las notificaciones de HITL).
- Base de datos PostgreSQL con las tablas listadas en [Tablas de PostgreSQL utilizadas](#tablas-de-postgresql-utilizadas).

## Cómo importar y ejecutar

1. En n8n, ir a **Workflows → Import from File** y seleccionar el archivo [`Manejo de Tarifas - Limpieza, Analisis, HITL y Dashboard (2).json`](<Manejo de Tarifas - Limpieza, Analisis, HITL y Dashboard (2).json>).
2. Configurar/actualizar las credenciales de Postgres, Google Gemini y Slack en cada nodo correspondiente.
3. Crear las tablas de PostgreSQL necesarias (ver sección de tablas).
4. Activar el workflow o ejecutarlo manualmente desde el nodo **Inicio Manual**.
5. Si el análisis detecta anomalías, revisar el formulario de HITL desde el link enviado a Slack para aprobar o rechazar los ajustes propuestos antes de que se publiquen en el dashboard.
