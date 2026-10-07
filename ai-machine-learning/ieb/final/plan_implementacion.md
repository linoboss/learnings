# Plan de implementación — Agente Inteligente de Clasificación de Incidencias

Este plan aterriza la propuesta del proyecto global en fases concretas. Reutiliza lo trabajado en las clases
anteriores para que cada decisión técnica tenga respaldo en algo ya practicado:

| Clase / sprint | Lo que se reutiliza aquí |
|---|---|
| `clase1/sprint1–2` | Carga de datos (local / Kaggle / Drive), EDA y estructura de notebook por fases |
| `clase1/sprint3` | Comparación ordenada de escenarios de modelado contra un *baseline* |
| `clase1/final` (fase 2 y 3) | Diseño de IA generativa con LLM acotado, supervisión humana, mitigación de alucinaciones y arquitectura en Azure |
| `clase2/sprint4–6` y `final` | Clasificación: Naive Bayes, SVM (lineal, parámetro C, *grid search*), ROC/AUC, matriz de confusión |
| `clase3/sprint2–3` | Árboles, *bagging*, Random Forest, *boosting*, desbalanceo, validación cruzada, importancia de variables y aviso sobre registros duplicados |

La novedad respecto a las clases es que **la entrada es mayoritariamente texto** (ticket, mensaje y logs), así
que hay que añadir una etapa de vectorización (TF-IDF) antes de aplicar los mismos clasificadores.

---

## 0. Alcance y decisiones de partida

- **Problema de ML**: clasificación supervisada **multiclase**. La entrada es una incidencia y la salida es el
  equipo responsable.
- **Problema de IA generativa**: a partir de la incidencia y del equipo ya predicho, generar un **descriptivo
  estructurado** (equipo, descripción, sistemas afectados, evidencias, cronología, severidad sugerida).
- **Separación estricta**: el LLM **no clasifica**, solo redacta. Así cada componente se evalúa por separado,
  como pide la propuesta.
- **Sin acciones autónomas**: el sistema propone y el técnico valida (*human-in-the-loop*, igual que en
  `clase1/final/fase2`).
- **Catálogo de equipos (propuesta, 7 clases)**: Integraciones, Base de Datos, Infraestructura y Redes,
  Seguridad, Desarrollo de Aplicaciones, Identidad y Accesos, Service Desk (puesto de trabajo). Se cierra en la
  Fase 1 porque todo el dataset depende de él.

### Entregables

Todos van en `final/`:

1. `fase1_generacion_dataset.ipynb`: generación sintética con LLM.
2. `fase2_calidad_dataset.ipynb`: validación, limpieza y control de calidad.
3. `fase3_modelo_clasificacion.ipynb`: EDA, modelos y selección.
4. `fase4_descriptivo_generativo.ipynb`: generación y evaluación del descriptivo.
5. `fase5_pipeline_end_to_end.ipynb`: demo del flujo completo.
6. `arquitectura_y_gobernanza.md`: arquitectura cloud, seguridad y supervisión humana.
7. `informe_final.md` (+ PDF): memoria del proyecto.

> Nota: `**/data/` está en `.gitignore`. Si el dataset sintético debe entregarse en el repo, hay que guardarlo
> directamente en `final/` (p. ej. `final/incidencias_sinteticas.csv`) y no en `final/data/`.

---

## Fase 1 — Diseño y generación del dataset sintético

**Objetivo:** construir un dataset realista sin datos confidenciales, controlando desde el principio los sesgos
típicos del texto generado por LLM.

### 1.1 Esquema de cada incidencia

| Campo | Tipo | Ejemplo |
|---|---|---|
| `ticket_id` | str | `INC-004213` |
| `fecha_hora` | datetime | `2026-03-14 10:32` |
| `canal` | cat | email / portal / chat / monitorización |
| `titulo` | texto corto | "No se puede facturar" |
| `descripcion_usuario` | texto | redactado por un usuario no técnico |
| `mensajes_adicionales` | texto (opcional) | seguimiento del usuario o de soporte N1 |
| `log_extracto` | texto (opcional) | 3–15 líneas de log |
| `codigo_error` | str (opcional) | `HTTP 504`, `ORA-12541`, `0x80070005` |
| `sistema_afectado` | cat | Facturación, ERP, VPN, Correo, CRM… |
| `entorno` | cat | producción / preproducción |
| `severidad_reportada` | cat | baja / media / alta / crítica |
| **`equipo`** | **cat (target)** | Integraciones |
| `descriptivo_referencia` | JSON (opcional) | descriptivo "ideal" para evaluar la Fase 4 |

Los campos opcionales deben faltar en una parte de los registros. En la realidad muchos tickets llegan sin
logs, y el modelo tiene que funcionar con ellos.

### 1.2 Estrategia de generación (para evitar un dataset "demasiado fácil")

- **Catálogo de escenarios por equipo**: unas 10–15 causas raíz por equipo (p. ej. Integraciones: *timeout*
  de API externa, certificado caducado del partner, cambio de contrato de API…). El LLM genera variaciones
  de cada escenario, no textos libres.
- **Diversidad controlada** mediante parámetros que se pasan en cada llamada:
  - perfil del usuario (técnico / no técnico)
  - tono (urgente, neutro, confuso)
  - longitud
  - idioma (castellano con anglicismos técnicos)
  - erratas
  - presencia o ausencia de logs y código de error
- **Casos ambiguos y frontera, alrededor del 15 %**: incidencias cuyo síntoma apunta a un equipo pero cuya
  causa es de otro (p. ej. "la app va lenta" que en realidad es Base de Datos). Son las que dan sentido a usar
  logs y evidencias técnicas.
- **Prohibición explícita en el *prompt*** de mencionar el nombre del equipo en el texto, para evitar fuga del
  target.
- **Salida estructurada**: el LLM devuelve JSON validado contra un esquema (pydantic). Las respuestas que no
  validan se regeneran.
- **Reproducibilidad**: se fijan las semillas de muestreo de parámetros y se guardan el *prompt*, el modelo y la
  temperatura de cada lote en un fichero de metadatos.
- **Tamaño objetivo**: unas 3.000–5.000 incidencias, con un desbalanceo deliberado y realista (Service Desk y
  Aplicaciones con más volumen; Seguridad con menos). Así se puede practicar el tratamiento del desbalanceo de
  `clase3`.

### 1.3 Proveedor LLM

Por coherencia con `clase1/final`, se usa **Azure OpenAI** en el diseño de arquitectura. Para la generación
en el notebook vale cualquier API con salida estructurada. Lo importante es aislarla tras una función
`generar_lote(params) -> list[Incidencia]` para poder cambiar de proveedor sin tocar el resto.

**Dependencias que habrá que instalar** (no se instalan automáticamente): SDK del proveedor LLM elegido
(`openai` o `anthropic`) y `pydantic`. Opcionalmente, `tqdm` para mostrar el progreso.

---

## Fase 2 — Validación, limpieza y control de calidad

**Objetivo:** que el dataset sea apto para entrenar y, sobre todo, que las métricas posteriores no estén
infladas. Es la lección de los registros duplicados de `clase3/sprint2`, que aquí es todavía más crítica.

| Control | Cómo | Acción |
|---|---|---|
| Esquema y tipos | pydantic / asserts en pandas | descartar o regenerar |
| Duplicados exactos | `df.duplicated()` sobre los campos de texto | eliminar |
| **Casi-duplicados** | similitud coseno TF-IDF > 0,9 entre pares | eliminar o agrupar |
| **Fuga del target** | buscar nombres y sinónimos del equipo en el texto | reescribir o descartar |
| Coherencia técnica | reglas: código de error ↔ sistema ↔ equipo (p. ej. `ORA-*` → BD) | revisar los incoherentes |
| Valores nulos | % por campo opcional, comparado con el diseño | ajustar la generación |
| Distribución de clases | conteo y proporción por equipo | regenerar las clases escasas |
| Calidad semántica del label | segunda pasada con un LLM "auditor" + revisión manual de una muestra estratificada (~200) | corregir o eliminar; medir el % de acuerdo |
| Diversidad léxica | longitud media, vocabulario, n-gramas más repetidos por clase | detectar plantillas repetidas |

**Partición train/test** (como en `clase2/sprint4`), estratificada por equipo y agrupada por escenario de
origen (`GroupShuffleSplit` / `StratifiedGroupKFold`). Así las variaciones de un mismo escenario no caen a la
vez en train y test. Sin esta precaución el modelo "memoriza escenarios" y el test no mide generalización.

**Test adicional "fuera de distribución"**: unas 50–100 incidencias redactadas a mano o con otro *prompt* u
otro modelo, para comprobar que el clasificador no ha aprendido solo el estilo del generador.

---

## Fase 3 — Modelo de Machine Learning (clasificación del equipo)

Se sigue la misma estructura de escenarios comparados que en `clase1/sprint3` y `clase3/sprint3`.

### 3.1 EDA

- Proporción de clases (desbalanceo).
- Longitud de los textos por clase.
- Términos más característicos por equipo (TF-IDF medio o χ²).
- Relación entre `codigo_error` / `sistema_afectado` y equipo, con una tabla de contingencia.

### 3.2 Ingeniería de características

- **Texto**: concatenar `titulo + descripcion + mensajes + log` y aplicar `TfidfVectorizer` (palabras 1–2
  gramas + caracteres 3–5 gramas, útil para códigos de error y erratas).
- **Campos estructurados**: *one-hot* de `sistema_afectado`, `canal`, `entorno` y `severidad`; familia del
  código de error extraída con regex (`HTTP 5xx`, `ORA-`, `0x8007…`); indicadores `tiene_log` y
  `tiene_codigo`.
- Todo dentro de un `ColumnTransformer` + `Pipeline` para que el preprocesado se ajuste solo con train (sin
  fuga en la validación cruzada).

### 3.3 Escenarios de modelado

| # | Modelo | Origen en el curso | Por qué |
|---|---|---|---|
| 0 | `DummyClassifier` (clase mayoritaria) | — | suelo de referencia (paradoja de la *accuracy*) |
| 1 | Reglas por código de error / sistema | — | *baseline* "sin ML" que justifica el proyecto |
| 2 | `MultinomialNB` / `ComplementNB` sobre TF-IDF | clase2 sprint5 | *baseline* clásico de texto, rápido |
| 3 | Regresión logística multinomial (`class_weight='balanced'`) | clase3 sprint3 (opcional) | probabilidades calibradas e interpretable |
| 4 | `LinearSVC` con *grid search* de C | clase2 sprint6 | suele ser el mejor en texto disperso |
| 5 | Random Forest / Gradient Boosting (HistGB, sobre TF-IDF reducido con SVD + estructurados) | clase3 sprint3 | contraste no lineal |
| 6 *(opcional)* | *Embeddings* de frases + logística | — | captura sinónimos que TF-IDF no capta |

### 3.4 Evaluación

- **Métrica principal: F1 macro**, porque con clases desbalanceadas la *accuracy* engaña, como se vio en
  `clase3`.
- Métricas secundarias: *accuracy*, recall por equipo y **top-2 accuracy**. Esta última es útil
  operativamente porque el técnico puede elegir entre dos sugerencias.
- **Matriz de confusión normalizada**: qué equipos se confunden entre sí y si tiene sentido de negocio (p. ej.
  Integraciones ↔ Infraestructura).
- **Validación cruzada estratificada y agrupada (5 *folds*)** para comprobar si las diferencias entre modelos
  son reales, como en `clase3/sprint3`.
- Evaluación separada en el **test fuera de distribución** y en el subconjunto de **casos ambiguos**.
- **Ablación**: rendimiento con solo el texto del usuario frente a texto + logs + código de error. Cuantifica
  la tesis de la propuesta de que las evidencias técnicas mejoran la asignación.

### 3.5 Interpretabilidad y umbral de confianza

- Términos con más peso por clase (coeficientes de la logística o la SVM) e importancia por permutación.
- **Umbral de confianza**: si la probabilidad máxima es menor que τ, la incidencia se marca como "revisión
  manual" y no se asigna automáticamente. Se elige τ con la curva cobertura vs. precisión (qué % se automatiza
  frente a qué % de acierto).
- Si el modelo elegido no da probabilidades fiables (SVM), se calibra con `CalibratedClassifierCV`.

### 3.6 Selección del modelo

Se elige el modelo con un criterio explícito (F1 macro en CV, robustez fuera de distribución, coste de
entrenamiento e interpretabilidad), como en la "Elección del mejor ensemble" de `clase3/sprint3`. Después se
serializa el `Pipeline` completo con `joblib`.

---

## Fase 4 — Componente de IA Generativa (descriptivo estructurado)

### 4.1 Diseño

- **Entrada al LLM**: los campos de la incidencia + el equipo predicho + su confianza.
- **Salida**: JSON validado contra un esquema, que luego se renderiza como texto:

```json
{
  "equipo_responsable": "Integraciones",
  "resumen": "Errores en el proceso de facturación desde las 10:30.",
  "sistemas_afectados": ["Aplicación de facturación", "Servicio externo de pagos"],
  "evidencias": ["Connection timeout durante las llamadas al servicio externo"],
  "cronologia": ["10:30 primer error reportado", "..."],
  "impacto": "...",
  "informacion_faltante": ["¿Afecta a todos los clientes o solo a algunos?"],
  "confianza_clasificacion": 0.87
}
```

- ***System prompt*** acotado: rol, tono técnico y la **regla de anclaje** "solo afirmar lo que aparece en la
  entrada; si falta un dato, indicarlo en `informacion_faltante`". Retoma los mecanismos anti-alucinación de
  `clase1/final/fase2`.
- **Pocas muestras (*few-shot*)**: 2–3 ejemplos de descriptivos de referencia.
- El campo `equipo_responsable` se **copia por código** desde la predicción del ML y no lo decide el LLM.
  Así se mantiene la separación de responsabilidades.

### 4.2 Evaluación

Se evalúa sobre unas 100 incidencias del test:

| Dimensión | Método |
|---|---|
| Validez de formato | % de salidas que validan el esquema JSON |
| **Fidelidad (sin alucinaciones)** | cada evidencia y código citado debe aparecer literalmente o casi en la entrada (comprobación automática por coincidencia de cadenas y códigos) |
| Cobertura | % de evidencias clave del `descriptivo_referencia` recogidas |
| Calidad percibida | rúbrica 1–5 (claridad, completitud, utilidad) con un LLM como juez + revisión humana de una muestra para validar al juez |
| Coste / latencia | tokens y segundos por incidencia |

Se compara **zero-shot frente a few-shot** y, opcionalmente, dos modelos de distinto tamaño (calidad frente a
coste).

---

## Fase 5 — Pipeline end-to-end y demo

`Incidencia → preprocesado → clasificador → (confianza ≥ τ ? equipo : "revisión manual") → LLM → descriptivo`

- Una función `procesar_incidencia(dict) -> dict` que encadena las fases 3 y 4.
- Demo sobre 5–10 incidencias variadas: casos claros, ambiguos, sin logs y con baja confianza.
- Pequeño **análisis de BI**: distribución de incidencias por equipo, sistema y severidad sobre el dataset
  procesado. Es la base de los indicadores que menciona la propuesta.
- *(Opcional)* Interfaz mínima con Gradio o Streamlit (otra dependencia que habría que instalar).

---

## Fase 6 — Arquitectura, gobernanza y supervisión humana (`arquitectura_y_gobernanza.md`)

Se reutiliza el enfoque de `clase1/final/fase3_arquitectura_cloud.md`, adaptado al nuevo caso:

| Capa | Servicio Azure | Función en este proyecto |
|---|---|---|
| Ingesta | Logic Apps / Event Grid + conectores ITSM (ServiceNow, Jira) | recibir tickets y logs |
| Logs | Azure Monitor / Log Analytics | fuente de evidencias técnicas |
| Modelo ML | Azure Machine Learning | registro, *endpoint* y monitorización de *drift* |
| IA generativa | Azure OpenAI Service | generación del descriptivo |
| Datos | Data Lake / Blob Storage | dataset, artefactos y predicciones |
| Secretos e identidad | Key Vault, Entra ID | credenciales y RBAC |
| Gobernanza | Purview, Azure Policy | linaje y cumplimiento |
| BI | Power BI | indicadores de incidencias |

Contenido mínimo del documento:

- **Human-in-the-loop**: el técnico acepta o reasigna. Las reasignaciones se registran como **nuevas
  etiquetas reales** para reentrenar, lo que va sustituyendo gradualmente los datos sintéticos.
- **Privacidad**: los logs pueden contener datos personales, así que se aplica enmascaramiento (emails, IPs,
  DNI) antes de enviarlos al LLM.
- **Monitorización**: tasa de reasignación manual (*proxy* de error en producción), *drift* del vocabulario y
  coste del LLM.
- **Riesgos y limitaciones**: el principal es la **brecha sintético → real**. Las métricas obtenidas son una
  cota optimista y hay que validarlas con un piloto.

---

## Fase 7 — Informe final

Sigue la estructura de `clase2/final/informe_final_LinoBossio.md`:

1. Problema y motivación (datos de xMatters de la propuesta).
2. Generación y calidad del dataset (incluido el % de etiquetas corregidas en la auditoría).
3. Modelo de clasificación: escenarios, métricas, matriz de confusión, ablación y umbral.
4. Componente generativo: diseño y resultados de fidelidad y calidad.
5. Pipeline y arquitectura.
6. Limitaciones, riesgos y siguientes pasos.

---

## Calendario orientativo

| Semana | Trabajo | Hito |
|---|---|---|
| 1 | Catálogo de equipos y escenarios, esquema y *prompts*; generación piloto de ~200 | revisión manual del piloto |
| 2 | Generación completa y Fase 2 (calidad, partición) | dataset congelado (versión v1) |
| 3 | Fase 3: EDA y escenarios 0–4 | *baseline* sólido |
| 4 | Fase 3: ensembles, CV, ablación, umbral y selección | modelo final serializado |
| 5 | Fase 4: descriptivo y su evaluación | métricas de fidelidad y calidad |
| 6 | Fases 5–7: pipeline, arquitectura e informe | entrega |

## Riesgos principales y mitigación

| Riesgo | Mitigación |
|---|---|
| Dataset sintético demasiado fácil (métricas ~100 %) | escenarios con casos frontera, partición agrupada por escenario, test fuera de distribución |
| Fuga del target en el texto | prohibirlo en el *prompt* + detector automático en la Fase 2 |
| Etiquetas incorrectas generadas por el LLM | auditoría con un segundo LLM + revisión humana muestral |
| Alucinaciones en el descriptivo | anclaje, salida JSON, comprobación automática de evidencias, campo `informacion_faltante` |
| Coste de las llamadas al LLM | generación por lotes, caché de respuestas en disco y modelo pequeño para el piloto |
