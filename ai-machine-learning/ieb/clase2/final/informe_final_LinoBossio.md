# Proyecto Final — Clase 2: Predicción del estado fetal a partir de CTG

**Autor:** Lino Bossio
**Dataset:** `ASI_casoPractico.csv` — 2126 cardiotocogramas (CTG) fetales, 22 variables explicativas y 1 variable objetivo (`Target`)
**Notebook adjunto:** `sprint_final_LinoBossio.ipynb`

---

## 1. Análisis descriptivo de las variables explicativas y el target

El dataset recoge características diagnósticas extraídas de registros de cardiotocografía
(frecuencia cardíaca fetal, movimientos fetales, contracciones uterinas y estadísticos del
histograma de FCF), clasificadas por consenso de tres obstetras. Se eliminan `ID`, `b`, `e`
(identificador e instantes de inicio/fin del segmento, sin valor predictivo) y `DR`
(deceleraciones repetitivas), que toma un único valor (0) en las 2126 observaciones.

**Target.** `Target` = 0 (normal) / 1 (anormal) está **desbalanceado**: 1655 registros
normales (77.85 %) frente a 471 anormales (22.15 %). Este desbalance obliga a mirar más allá
de la precisión global al validar el modelo (una precisión alta se puede lograr prediciendo
siempre la clase mayoritaria), de ahí la importancia de la curva ROC y la matriz de confusión.

**Variables explicativas más relevantes.** Según la correlación (en valor absoluto) de cada
variable con el target, las más influyentes son `ASTV` (0.493, % de tiempo con variabilidad
de corto plazo anormal), `ALTV` (0.489, % de tiempo con variabilidad de largo plazo anormal),
`AC` (0.369, número de aceleraciones) y `DP` (0.341, deceleraciones prolongadas). El resto de
variables (frecuencia base, estadísticos del histograma de FCF, contracciones uterinas)
aportan señal moderada o baja.

**Variable nueva creada.** Se construyó `Abn_Variability = ASTV + ALTV`, que combina en un
único indicador el porcentaje de tiempo con variabilidad anormal a corto y largo plazo. Su
correlación con el target (0.575) supera a la de sus dos componentes por separado, por lo que
se incorpora como variable explicativa adicional. Se probó también `Decel_Total = DL + DS +
DP`, pero se descartó: su correlación (0.033) es muy inferior a la de `DP` aislada (0.341),
porque `DL` y `DS` son casi siempre cero y diluyen la señal de `DP` al sumarlas.

---

## 2. División del conjunto de datos en entrenamiento y test

Se reutiliza el mismo muestreo definido en los sprints 5 y 6: **60 % entrenamiento / 40 %
test**, con `random_state=0` para que el resultado sea reproducible y comparable entre
algoritmos.

| Conjunto | Observaciones | % |
|---|---|---|
| Entrenamiento | 1275 | 60 % |
| Test | 851 | 40 % |

---

## 3. Algoritmo seleccionado e hiperparámetros

Se ajustaron y compararon tres modelos sobre el mismo *split*:

1. **Naive Bayes** (`GaussianNB`, sin hiperparámetros a ajustar).
2. **SVM con valores por defecto** (`SVC`, kernel `rbf`, `C=1.0`, `gamma='scale'`).
3. **SVM con hiperparámetros optimizados** mediante `GridSearchCV` (`scoring='roc_auc'`,
   `cv=3`) sobre la rejilla: kernel ∈ {`rbf`, `linear`, `poly`}, `C` ∈ {0.1, 1, 10},
   `gamma` ∈ {1e-3, 1e-4} (solo `rbf`), `degree` ∈ {2, 3} (solo `poly`).

La búsqueda de hiperparámetros identificó como mejor combinación **kernel lineal, `C=1`**
(AUC medio en validación cruzada: 0.961). Este es el **algoritmo final seleccionado**: un
SVM de kernel lineal con `C=1` (resto de hiperparámetros por defecto de `scikit-learn`).

---

## 4. Validación: curva ROC, AUC, matriz de confusión y precisión

| Modelo | AUC train | AUC test | Precisión train | Precisión test |
|---|---|---|---|---|
| Naive Bayes | 0.9499 | 0.9381 | 0.8886 | 0.8872 |
| SVM (por defecto) | 0.9378 | 0.9217 | 0.8784 | 0.8719 |
| **SVM ajustado (Grid Search)** | **0.9710** | **0.9610** | **0.9184** | **0.9048** |

**Matrices de confusión del modelo final (SVM ajustado, kernel lineal, `C=1`):**

Entrenamiento (1275 obs.):

|  | Predicho Normal | Predicho Anormal |
|---|---|---|
| **Real Normal** | 949 | 50 |
| **Real Anormal** | 54 | 222 |

Test (851 obs.):

|  | Predicho Normal | Predicho Anormal |
|---|---|---|
| **Real Normal** | 629 | 27 |
| **Real Anormal** | 54 | 141 |

**Lectura de resultados.** El SVM ajustado supera claramente tanto a Naive Bayes como al SVM
con valores por defecto, tanto en AUC como en precisión, en entrenamiento y en test. La
diferencia entre AUC de entrenamiento (0.971) y de test (0.961) es pequeña (≈ 0.01), al igual
que la diferencia de precisión (≈ 1.4 puntos porcentuales): **no hay indicios de
sobreajuste** relevante, pese a que el modelo ajustado es más complejo (kernel lineal
calibrado vía validación cruzada) que el SVM con valores por defecto. En la matriz de
confusión de test, el modelo comete 54 falsos negativos (anormales clasificados como
normales) y 27 falsos positivos — en un contexto clínico, los falsos negativos son el error
más costoso, y siguen siendo la principal limitación del modelo pese a la mejora obtenida.

**Conclusión.** El algoritmo final entregado es un **SVM con kernel lineal y `C=1`**,
seleccionado por `GridSearchCV` optimizando AUC en validación cruzada, con un AUC de test de
**0.961** y una precisión de test de **90.48 %**, sin sobreajuste apreciable respecto al
conjunto de entrenamiento.
