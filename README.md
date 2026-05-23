# 🔍 Incidencia Delictiva en México — Predicción y Análisis Geoespacial

> Modelo de machine learning para predecir la incidencia de robos por estado y mes en México, con visualización interactiva en Power BI.

---

## El problema

México registra cientos de miles de robos al año distribuidos de forma muy desigual entre sus 32 estados. La incidencia no es aleatoria: tiene patrones estacionales, concentración geográfica y tendencias temporales que un modelo bien construido puede anticipar.

La pregunta que guía este proyecto es concreta: **¿es posible predecir cuántos robos ocurrirán en un estado el próximo mes, y detectar cuándo un estado está viviendo un pico anómalo?** Una respuesta confiable a esa pregunta tiene valor directo para la planeación de seguridad pública, la gestión de riesgo operativo en banca y el diseño de productos de seguros.

---

## Objetivo

Construir un sistema de predicción de tres capas:

1. **Regresión** — estimar el volumen de robos por entidad y mes.
2. **Clasificación binaria** — identificar si una entidad estará en estado de *alto crimen* (por encima de su mediana histórica).
3. **Detección de anomalías** — señalar cuándo la incidencia supera significativamente el comportamiento normal de ese estado.

---

## Dataset

- **Fuente:** Secretariado Ejecutivo del Sistema Nacional de Seguridad Pública (SESNSP).
- **Período:** 2022–2023.
- **Filtro:** delitos tipo *Robo* (todas las modalidades).
- **Granularidad:** entidad federativa × mes.
- **Variable objetivo (regresión):** `incidencia_delictiva` — suma mensual de robos por estado.

---

## Metodología

### 1. Limpieza y agregación

Los datos originales están a nivel de subtipo de delito. Se agruparon por `entidad × año × mes` sumando la incidencia, reduciendo el dataset a una serie de tiempo multivariada de 32 entidades.

### 2. Feature engineering

El desafío central es que el modelo no puede ver el futuro, por lo que todas las variables se construyen a partir de información disponible *antes* del mes a predecir:


| Feature               | Descripción                                     | Por qué importa                               |
| --------------------- | ------------------------------------------------ | ---------------------------------------------- |
| `lag_1`               | Incidencia del mes anterior                      | Dependencia temporal directa                   |
| `rolling_3`           | Promedio móvil de 3 meses                       | Memoria histórica reciente                    |
| `pct_change`          | Cambio porcentual respecto al mes previo         | Detecta aceleración o caída                  |
| `diff_1`              | Diferencia absoluta (mes actual − mes anterior) | Señal de cambio brusco                        |
| `tendencia`           | Contador temporal por entidad                    | Captura tendencia secular                      |
| `sen_mes` / `cos_mes` | Codificación cíclica del mes                   | Estacionalidad sin ruptura en diciembre→enero |

La variable `entidad` se transformó con **Target Encoding** para evitar el alto cardinal de 32 categorías con One-Hot Encoding.

Los targets de clasificación (`alto_crimen`, `anomalia`) se derivaron de estadísticos internos de cada estado — mediana e intervalo media ± 1σ — para que el umbral sea relativo a la historia de ese estado, no a un valor absoluto nacional.

### 3. Modelos entrenados

Se entrenaron tres modelos con `GridSearchCV` (5-fold CV):


| Modelo                      | Tarea                      | Métrica       | Resultado                        |
| --------------------------- | -------------------------- | -------------- | -------------------------------- |
| Logistic Regression         | Clasificación alto crimen | F1             | **0.77**                         |
| Logistic Regression + SMOTE | Detección de anomalías   | F1             | **0.62** (clase minoritaria 15%) |
| Ridge Regression            | Predicción de volumen     | R² / RMSE     | 0.9739 / 358                     |
| **Random Forest Regressor** | Predicción de volumen     | **R² / RMSE** | **0.9965 / 131**                 |

El modelo ganador para predicción fue **Random Forest** con `max_depth=None`, `min_samples_split=5`, `n_estimators=200`.

Para la detección de anomalías se aplicó **SMOTE** para balancear la clase minoritaria (15.4% de casos), logrando un recall de 92% en anomalías reales — priorizando no dejar pasar picos sin detectar.

---

## Resultados clave

### El crimen en México está altamente concentrado

Las tres entidades más afectadas concentran más del **45% de todos los robos** del período analizado:


| Entidad           | Promedio mensual (real) | Predicción | Error |
| ----------------- | ----------------------- | ----------- | ----- |
| Estado de México | 10,994                  | 10,599      | −395 |
| Ciudad de México | 6,252                   | 6,194       | −58  |
| Jalisco           | 3,334                   | 3,485       | +151  |

Esta concentración no es solo un dato estadístico: implica que cualquier intervención de política pública o estrategia de cobertura de riesgo que no diferencie a Estado de México y CDMX del resto del país estará mal calibrada desde el inicio.

### El mes anterior predice el siguiente con precisión

La variable `lag_1` (incidencia del mes previo) tiene una importancia del **14.6%** en el modelo, siendo la segunda característica más relevante después del estado mismo. Esto confirma que la incidencia delictiva tiene una fuerte inercia temporal — los estados con alto crimen en un mes tienden a mantenerlo el siguiente.

### El modelo es honesto con sus errores

El modelo tiende a **subestimar** en los estados con mayor volumen (Estado de México: −395, Veracruz: −100) y a **sobreestimar** ligeramente en estados medianos (Jalisco: +151, Guanajuato: +111). Esto es un patrón conocido en regresión con valores extremos — los outliers "jalan" las predicciones hacia la media. Con más datos históricos o un modelo de boosting (XGBoost, LightGBM) este sesgo se puede reducir.

### La estacionalidad existe pero es secundaria

La codificación cíclica del mes (`sen_mes`, `cos_mes`) y el número de mes tienen una importancia combinada menor al 0.1% en el modelo de Random Forest. Esto no significa que no haya estacionalidad, sino que **la entidad y su historial reciente dominan tanto la señal que el mes añade poco**. La estacionalidad sería más relevante en modelos nacionales agregados.

---

## Dashboard interactivo

El dashboard en Power BI permite explorar los resultados desde tres ángulos:

- **Mapa coroplético** — distribución geográfica de la incidencia predicha por estado.
- **Tendencia mensual** — comparación de predicción vs. tendencia a lo largo del año.
- **Ranking por entidad** — promedio de predicción, ordenado de mayor a menor.

![1779500767496](images/README/1779500767496.png)

## Estructura del repositorio

```
incidencia-delictiva/
│
├── data/
│   └── incidencia_delictiva.csv    # Dataset fuente (SESNSP)
│   └── resultados_modelo.csv
│
├── notebook/
│   └── modelo-delictiva.ipynb      # EDA, feature engineering, modelos
│
├── powerbi/
│   └── dashboard.pbix              # Dashboard interactivo
│
├── images/
│   └── dashboard.png               # Imagen del dashboard
│
├── README                          # Documentación del proyecto
└── requirements.txt                # Dependencias del proyecto
```

---

## Cómo reproducir

```bash
    # Clonar el repositorio
git clone https://github.com/Jane-BA/incidencia-delictiva.git 
cd incidencia-delictiva
pip install -r requirements.txt     # Instalar dependencias
jupyter notebook notebook/modelo-delictiva.ipynb    # Ejecutar el notebook
```

**Dependencias principales:** `pandas`, `numpy`, `scikit-learn`, `imbalanced-learn`, `category_encoders`, `matplotlib`, `seaborn`.

---

## Lo que aprendí (y lo que haría diferente)

Este proyecto me dejó tres aprendizajes concretos:

Primero, que la **identidad del estado es la variable más poderosa** (81.9% de importancia). Eso tiene una lectura de negocio clara: el riesgo delictivo en México es estructural, no coyuntural — no sube y baja por el mes, sino que hay estados que históricamente concentran el problema.

Segundo, que manejar **clases desbalanceadas** (15% de anomalías) requiere no solo SMOTE, sino elegir bien la métrica de evaluación. Optimizar accuracy hubiera dado un modelo que simplemente predice "no anomalía" el 85% del tiempo y parece bueno. Optimizar recall en la clase positiva es lo que tiene sentido en un problema de seguridad.

Tercero, que un R² de 0.9965 es un número para poner en el CV, pero el **análisis del error por entidad** es lo que realmente dice si el modelo es útil — y en este caso sí lo es para la mayoría de los estados, con los sesgos esperables en los extremos.

---

## Contacto

**Janeth Brizuela Alba** — [LinkedIn](https://www.linkedin.com/in/janeth-brizuela-alba-6b2771218/) · [GitHub](https://github.com/Jane-BA)

> *Proyecto parte de un portafolio de Data Science orientado a banca y fintech.*
