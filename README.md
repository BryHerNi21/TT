# Modelo predictivo de priorización para intervenciones en infraestructura peatonal — Bogotá

Cuadernos de preprocesamiento, análisis exploratorio (EDA), unión espacial, modelado y priorización de la siniestralidad peatonal en Bogotá D.C. (2015–2020). El flujo produce un conjunto de datos **a nivel de vía**, estima sobre él una *función de desempeño de seguridad* (FDS) con modelos de conteo (Poisson / Binomial Negativa / ZINB) y usa el modelo final para construir tres escenarios de priorización de segmentos viales (A, B y C).

**Autores:** Jhoan Santiago Toro Chaparro y Brayan Sebastian Hernández Nieto
**Directora:** Yessica Liceth Velásquez Castiblanco
**Universidad Externado de Colombia — Departamento de Matemáticas**

---

## 1. Flujo general

```
Data/raw ──► 3 EDA ──► Data/processed ──► Union_segmentos ──► segmentos_unificado.csv ──► modelo_FDS_peatonal ──► Data/processed/modelo_fds ──► priorizacion ──► Data/processed/obj3
```

| Orden | Cuaderno | Qué hace | Salidas en `Data/processed/` |
|---|---|---|---|
| 1 | `EDA_Siniestros_Viales_Bogota_peatones_indices_ajustados.ipynb` | Une el histórico georreferenciado (CSV) con las hojas del Excel de la SDM por `CODIGO_ACCIDENTE`, filtra a `CONDICION = PEATON`, limpia la edad, hace el EDA (temporal, espacial, gravedad, pruebas de asociación) y georreferencia. | `siniestros_procesados.gpkg` |
| 2 | `EDA_Malla_Vial_Puentes_zona_urbana_adecuado.ipynb` | EDA de la Malla Vial Integral y de los puentes. Reproyecta a EPSG:3116, imputa nulos (media/moda; KNN para `MVI_VELREG` y `MVINUMC`) y calcula longitudes reales en metros. | `malla_vial_procesada.gpkg`, `puentes_procesado.gpkg`, `puentes_peatonales_supuesto.gpkg` |
| 3 | `EDA_Malla_peatonal_semaforos_adecuado.ipynb` | EDA de la red peatonal y los semáforos. Reproyecta, elimina columnas vacías o constantes (análisis de entropía) e imputa faltantes en semáforos. | `peatonal_procesada.gpkg`, `semaforos_procesados.gpkg` |
| 4 | `Union_segmentos.ipynb` | Recorta al perímetro urbano (POT), descarta geometrías < 4 m y construye el dataset final **a nivel de vía** (vía completa, sin subdividir): distancia a puente y a semáforo, cobertura peatonal, conteo de siniestros (con `sjoin_nearest`) y población por localidad. Calcula también la tasa por 100.000 habitantes. | `segmentos_unificado.gpkg`, `segmentos_unificado.csv`, `siniestros_tasa_100k_localidad_anio.csv` |
| 5 | `modelo_FDS_peatonal.ipynb` | **Modelo vigente (v5).** Ajusta Nulo, Poisson, BN y ZINB con `log(longitud)` como covariable libre; verifica la BN y la dispersión; aplica una regla de decisión declarada; evalúa con validación cruzada aleatoria y por bloques espaciales; calcula el I de Moran de los residuales y errores estándar robustos/agrupados; contrasta la longitud libre con el offset. Guarda el modelo final para el cuaderno 6. | `modelo_fds/` (ver sección 6) |
| 6 | `priorizacion.ipynb` | Carga el modelo guardado por el cuaderno 5 y construye tres escenarios de priorización: **A** (frecuencia predicha), **B** (frecuencia esperada con Bayes empírico) y **C** (exceso esperado). Compara escenarios, describe severidad y cobertura peatonal de los priorizados, verifica conteos extremos y genera mapas. | `obj3/` (ver sección 7) |

**Orden de ejecución: 1 → 2 → 3 → 4 → 5 → 6.** Los cuadernos 1, 2 y 3 son independientes entre sí; el 4 necesita las salidas de los tres; el 5 lee solo `segmentos_unificado.csv`; el 6 necesita `modelo_fds/` (cuaderno 5), `segmentos_unificado.csv`/`.gpkg` y `siniestros_procesados.gpkg`, y para los mapas también `Data/raw/perimetrourbano (1).gpkg` y `Data/raw/Loca.shp`.

### Otras carpetas del repositorio

| Ruta | Estado |
|---|---|
| `BackUp/` | Respaldos locales de iteraciones previas del modelo. Está en `.gitignore`, por lo que **no se versiona** ni viaja en el zip del proyecto. No usar para resultados. |

---

## 2. Estructura de carpetas

```
.
├── Data/
│   ├── raw/                  # fuentes originales (ver sección 3)
│   └── processed/            # salidas de los cuadernos 1–4
│       ├── modelo_fds/       # salidas del cuaderno 5 (modelo y tablas)
│       └── obj3/             # salidas del cuaderno 6 (priorización)
├── BackUp/                   # versiones antiguas del modelo (solo local)
├── EDA_*.ipynb               # cuadernos 1–3
├── Union_segmentos.ipynb     # cuaderno 4
├── modelo_FDS_peatonal.ipynb # cuaderno 5
├── priorizacion.ipynb        # cuaderno 6
├── requirements.txt
└── README.md
```

Todas las rutas están escritas en relativo y con `Data/` en mayúscula inicial; se ejecutan desde la raíz del proyecto.

## 3. Datos de entrada (`Data/raw/`)

| Archivo | Fuente | Usado en |
|---|---|---|
| `DATO_ABIERTOS_siniestros_viales_consolidados_bogota_dc (3).xlsx` | SDM (Datos Abiertos) | Cuaderno 1 |
| `historico_siniestros_bogota_d_c_-.csv` | SDM | Cuaderno 1 |
| `Loca.shp` (+ `.shx`, etc.) | Localidades de Bogotá (mapas) | Cuadernos 1 y 6 |
| `Malla_Vial_Integral_Bogota_D_C (3).geojson` | IDECA / Catastro Bogotá | Cuaderno 2 |
| `Puentes bogota.gpkg` | IDU / IDECA | Cuaderno 2 |
| `red_peatonal_conexion (1).gpkg` | Datos Abiertos Bogotá | Cuaderno 3 |
| `semaforos_bogota.geojson` | SDM | Cuaderno 3 |
| `perimetrourbano (1).gpkg` | SDP (POT, Decreto 555 de 2021) | Cuadernos 4 y 6 |
| `localidades_bogota.gpkg` | SDP — polígonos de localidades (columna `LocNombre`) | Cuaderno 4 |
| `osb_demografia-poblacion-localidad.csv` | SDS / SaluData (convenio DANE-SDP), población por localidad 2005–2035 | Cuaderno 4 |

Cuatro insumos están en el repositorio, en `Data/raw/`: el Excel de siniestros de la SDM y, mediante Git LFS, `Puentes bogota.gpkg`, `red_peatonal_conexion (1).gpkg` y `semaforos_bogota.geojson` (después de clonar, ejecuta `git lfs pull` para descargar su contenido completo). Los otros seis no se versionan por su tamaño; descárgalos de estas fuentes y colócalos en `Data/raw/` con el nombre indicado:

| Archivo | Descarga |
|---|---|
| `historico_siniestros_bogota_d_c_-.csv` | https://datosabiertos.bogota.gov.co/dataset/historico-siniestros-bogota-d-c |
| `Malla_Vial_Integral_Bogota_D_C (3).geojson` | https://datosabiertos.bogota.gov.co/dataset/malla-vial-integral-bogota-d-c1 |
| `perimetrourbano (1).gpkg` | https://datosabiertos.bogota.gov.co/dataset/perimetro-urbano-bogota-d-c |
| `Loca.shp` (+ `.shx`, `.dbf`, `.prj`) | https://datosabiertos.bogota.gov.co/dataset/localidad-bogota-d-c (formato shapefile) |
| `localidades_bogota.gpkg` | https://datosabiertos.bogota.gov.co/dataset/localidad-bogota-d-c (formato GeoPackage) |
| `osb_demografia-poblacion-localidad.csv` | https://saludata.saludcapital.gov.co/osb/indicadores/poblacion-de-bogota-d-c-2005-2035/ |

## 4. Convenciones del proyecto

- **CRS:** todas las capas se trabajan en **EPSG:3116** (MAGNA-SIRGAS / Bogotá, metros). Los cuadernos fuerzan el CRS en vez de asumirlo y fallan si una capa no lo trae definido.
- **Periodo de estudio:** 2015–2020. La hoja `SINIESTROS` del Excel llega hasta 2020 y el CSV hasta 2021, por lo que los años finales del CSV están incompletos en las variables que vienen del Excel.
- **Unidad de observación:** la **vía original completa** de la Malla Vial Integral (una fila = una vía), con su longitud real en `long_seg_m`. No se subdivide en tramos de longitud fija.
- **Parámetros de diseño (`Union_segmentos.ipynb`):**

| Parámetro | Valor | Significado |
|---|---|---|
| `UMBRAL_RUIDO` | 4 m | Se descartan geometrías más cortas |
| `UMBRAL_SINIESTROS_M` | 30 m | Distancia máxima para asignar un siniestro a su vía más cercana |
| `UMBRAL_PEATONAL_M` | 15 m | Alcance para considerar un punto de la vía "cubierto" por malla peatonal |
| `PASO_COBERTURA_M` | 5 m | Separación aproximada de los puntos de muestreo a lo largo de cada vía |

- **Parámetros del modelo (`modelo_FDS_peatonal.ipynb`):** semilla `1949`, `K_PLIEGUES = 5`, `LONGITUD_LIBRE = True`, bloques espaciales de `TAM_BLOQUE_M = 2000`, `K_VECINOS = 8` para el I de Moran, `N_PERM = 199`, `N_PERM_CORR = 999`, `N_SIM = 5000`.
- **Parámetros de la priorización (`priorizacion.ipynb`):** `TOP_PCT = [1, 5, 10]` (fijados antes de ver resultados), `P_REF = 5` (top de referencia para tablas descriptivas y mapas), `CALIBRAR = True`.
- **Puentes y semáforos:** se resumen como distancia al más cercano (`sjoin_nearest`), sin umbral.
- **Empates:** un siniestro equidistante de varias vías (típico en intersecciones) se asigna a la de menor `seg_id`; es un criterio explícito y reproducible, sin fundamento sustantivo.
- **Variable objetivo:** `n_siniestros`, conteo de **siniestros distintos** (`CODIGO_ACCIDENTE` únicos) por vía. El conjunto de datos también guarda `n_peatones_afectados`, y se exportan `n_herido` y `n_muerto`.

## 5. Dataset final: `segmentos_unificado.csv`

131.704 vías. Solo el 10,12 % (13.322 vías) registra al menos un siniestro peatonal; el 89,88 % tiene cero. La asignación a ≤ 30 m captura el 99,93 % de los siniestros peatonales (21.701 peatones afectados, que corresponden a 20.037 siniestros distintos en `n_siniestros`).

El `.gpkg` (capa `segmentos`) conserva la geometría y todas las columnas de trabajo, entre ellas `seg_id` y `vial_id` (28 columnas). El `.csv` es la versión de modelado y tiene 14 columnas:

| Columna | Descripción |
|---|---|
| `long_seg_m` | Longitud de la vía (m); entra al modelo como `log(longitud)` |
| `MVI_VELREG` | Velocidad reglamentaria (imputada por KNN); en el modelo, variable categórica (dummy `vel_60`) |
| `MVINUMC` | Número de calzadas (imputada por KNN) |
| `dist_puente_peatonal_m`, `dist_puente_vehicular_m` | Distancia al puente peatonal / vehicular más cercano |
| `dist_semaforo_m` | Distancia al semáforo más cercano |
| `ratio_cobertura_peatonal` | Proporción ∈ [0, 1] de la vía con malla peatonal a ≤ 15 m. Reemplaza a `dist_peatonal_m`, que era ≈ 0 m para casi todas las vías y codificaba igual "sin malla" y "pegada a la malla" |
| `n_siniestros`, `n_herido`, `n_muerto` | Conteos asignados a la vía |
| `poblacion_localidad`, `log_poblacion` | Población representativa de la localidad de la vía (promedio 2015–2020 sobre el centroide de la vía) |
| `x_m`, `y_m` | Centroide de la vía en EPSG:3116; se usan para el I de Moran, la validación por bloques y los mapas |

El CSV **no** incluye `seg_id`, `localidad` ni `tipo_via`; el cuaderno 6 trabaja por posición de fila (mismo orden que el CSV) y omite con aviso las tablas descriptivas por `localidad` y `tipo_via`.

La población de años no censales (todos salvo 2005 y 2018) es una **proyección o retroproyección**, no una medición directa; debe declararse así en el documento. `poblacion_localidad` está en el dataset pero **no es covariable del modelo**.

`siniestros_tasa_100k_localidad_anio.csv` contiene la tasa descriptiva de siniestros peatonales por 100.000 habitantes por localidad y año (cada siniestro se cruza con la población de su localidad en su año exacto). Es un indicador aparte, independiente del dataset por vía.

## 6. Modelo (`modelo_FDS_peatonal.ipynb`, v5)

**Especificación.** Conteo `n_siniestros` con `log(long_seg_m)` como **covariable libre** (`LONGITUD_LIBRE = True`), no como offset. Covariables: `log1p` de las tres distancias (estandarizadas por pliegue), `ratio_cobertura_peatonal`, `MVINUMC` y dummies de `MVI_VELREG` (en la práctica `vel_60`). En el ZINB la inflación de ceros depende de la longitud estandarizada. El modelo Nulo usa intercepto y `log(longitud)`.

**Verificación de la BN.** El `alpha` se estima por perfil de verosimilitud (no depende del optimizador de `statsmodels`). Resultado: α ≈ 2,83; razón de verosimilitud Poisson vs BN ≈ 9.951 (p < 0,001 con corrección de frontera); Pearson/gl pasa de 1,665 (Poisson) a 1,165 (BN). Hay sobredispersión y la BN es el modelo base.

**Regla de decisión (declarada de antemano).** El ZINB reemplaza a la BN solo si la base es BN, mejora el BIC, mejora log-score y devianza fuera de muestra, y todos los ajustes convergen. El ZINB **no** mejora el BIC (99.594 frente a 99.576) ni el log-score ni la devianza, por lo que el **modelo final es la BN**. La priorización (cuaderno 6) exige la BN y falla si el modelo final es otro.

| Modelo | Parámetros | AIC | BIC |
|---|---|---|---|
| Nulo | 2 | 126.366,1 | 126.385,7 |
| Poisson | 8 | 109.437,6 | 109.515,9 |
| **BN** | 9 | **99.488,3** | **99.576,4** |
| ZINB | 11 | 99.486,8 | 99.594,5 |

**Longitud: libre frente a offset (sección 11 del cuaderno).** Con `log(longitud)` como covariable libre la elasticidad es 0,504 (IC 95 % agrupado 0,462–0,547), no 1 como supone el offset (LR = 1.278 en la BN, p ≈ 7·10⁻²⁸⁰). La especificación libre mejora BIC, log-score y devianza por bloques en BN y ZINB, y es la que se adopta. **La decisión se tomó después de ver esta sensibilidad y el texto de la tesis debe declararlo.**

**Validación (K = 5).** Dos esquemas: aleatoria y por bloques espaciales (celdas de 2.000 m, 132 celdas). Devianza explicada de la BN ≈ 0,17 en ambos esquemas (la caída entre esquemas es pequeña); ningún ajuste dejó de converger.

**Autocorrelación espacial.** I de Moran de los residuales (8 vecinos, 199 permutaciones): 0,023 para la BN (frente a 0,055 del modelo nulo). Sigue siendo significativo (p = 0,005) por el tamaño de muestra, pero el correlograma cae a ≈ 0 desde los 250 m, muy por debajo del lado del bloque.

**Coeficientes (BN final, errores estándar agrupados por celda).**

| Variable | IRR | IC 95 % |
|---|---|---|
| `dist_puente_peatonal_m` | 0,980 | 0,938 – 1,025 (no significativo) |
| `dist_puente_vehicular_m` | 1,082 | 1,036 – 1,131 |
| `dist_semaforo_m` | 0,781 | 0,744 – 0,819 |
| `ratio_cobertura_peatonal` | 2,021 | 1,925 – 2,123 |
| `MVINUMC` | 1,241 | 1,179 – 1,306 |
| `log_long` (elasticidad) | 1,656 (coef. 0,504) | 1,587 – 1,728 |
| `vel_60` | 1,409 | 1,279 – 1,553 |

Las distancias están estandarizadas, de modo que sus IRR son por desviación estándar de `log1p(distancia)`. Son asociaciones, no efectos causales; sin exposición peatonal real, la cobertura actúa como proxy de actividad.

**Limitaciones señaladas por los diagnósticos.**
- Calibración en forma de S por deciles de predicción: el modelo sobreestima los deciles bajos y subestima los intermedios (7 y 8); conserva el orden, no los niveles absolutos.
- Los residuos cuantílicos tienen media ≈ 0 y desviación ≈ 1; VIF máximo 1,50.
- Los siniestros que no caen a ≤ 30 m de ninguna vía (0,07 %) quedan fuera del modelo.

**Salidas en `Data/processed/modelo_fds/`:** `config.json` (parámetros, versiones, `alpha_bn`), `modelo_bn.joblib` (paquete del modelo), `predicciones_bn.csv`, `tabla_coeficientes.csv`, `validacion_cruzada.csv`, `regla_longitud.csv`, `sensibilidad_longitud_ajuste.csv` y `sensibilidad_longitud_validacion.csv`.

> Las cifras de esta sección y de `modelo_fds/` corresponden a una ejecución completa de los cuadernos 5 y 6, desde el inicio y con el núcleo reiniciado, en el commit indicado en la sección 3.3.3 del documento.

## 7. Priorización (`priorizacion.ipynb`)

Usa la BN final (α ≈ 2,83) sobre toda la red y aplica antes un factor de calibración (observados / predichos = 0,9856). Con `w = 1 / (1 + α·μ)`:

| Escenario | Criterio de orden | Qué mide |
|---|---|---|
| **A** `A_predicha` | `μ` (frecuencia predicha) | Dónde se espera mayor frecuencia, con o sin historial |
| **B** `B_esperada_EB` | `w·μ + (1 − w)·y` (Bayes empírico) | Frecuencia esperada combinando modelo y conteo observado |
| **C** `C_exceso` | `(1 − w)(y − μ)` (potencial de mejora) | Exceso esperado frente a la predicción; solo segmentos con exceso positivo |

Se evalúan tops de 1 %, 5 % y 10 % de la red (el de C se limita a los segmentos con exceso positivo: 13.147, el 9,98 % de la red).

**Resultados principales (top 5 %, `P_REF`).**
- **Segmentos sin siniestros registrados:** 66,5 % en A; 0 % en B y C (red completa: 89,9 %). En B y C esto se cumple por construcción del Bayes empírico y no es un hallazgo empírico; solo el contraste de A con C es informativo. La probabilidad de cero que asigna la propia BN en el top de A (67,2 %) coincide con lo observado (66,5 %).
- **Coincidencia (Jaccard):** B y C se parecen mucho (0,64 en el top 5 %); A se separa de ambos (0,20 con B y 0,08 con C). En segmentos con siniestros, Spearman A–C = 0,547, A–B = 0,894 y B–C = 0,770; en la red completa A–C es negativa por construcción (−0,653).
- **Longitud:** los priorizados por C tienen mediana de 67,2 m frente a 49,6 m en la red; la sensibilidad por kilómetro cuantifica el sesgo hacia tramos largos.
- **Severidad (descriptiva, no entra al orden):** los fallecidos son el 7,43 % de los siniestros del top 5 % de C frente al 6,91 % en la red.
- **Cobertura peatonal (descriptiva):** es cobertura, no estado del andén ni tipo de infraestructura. Como la cobertura ya está en `μ`, no se suma al puntaje. El top de A concentra cobertura del 100 % (99 %), frente al 35,4 % de la red.
- **Conteos extremos:** el segmento con más siniestros (52, en `KR 80 – CL 2 51`) concentra 41 registros en una misma coordenada; la sección 12 del cuaderno evalúa dos tratamientos (una coordenada por segmento y año; tope de 10) y mide cuánto cambia el orden de C.
- **Sensibilidad de α:** el α por decil de longitud varía entre 2,2 y 5,8; usar α por decil mantiene Jaccard de 0,78–1,00 con los conjuntos de B y C.

**Limitaciones heredadas.** FDS estimada y aplicada sobre la misma red (predicción dentro de muestra), α constante frente a la longitud, regla de asignación de siniestros a segmentos (empates), subregistro, cobertura que mezcla tipos de infraestructura peatonal, infraestructura vigente frente a siniestros 2015–2020, y tamaño del top-N definido por convención y no por presupuesto. La lista indica dónde se espera mayor frecuencia de siniestros peatonales, no dónde la infraestructura es peor ni qué intervención corresponde.

**Salidas en `Data/processed/obj3/`:** `priorizacion_segmentos.csv` (ranking completo, ~34 MB), `resumen_escenarios.csv`, `pct_sin_siniestros.csv`, `coincidencia.csv`, `spearman_*.csv`, `severidad_top.csv`, `sensibilidad_por_km.csv`, `composicion_decil_mu.csv`, `cobertura_bandas.csv`, `cobertura_deficit.csv`, `cobertura_x_velocidad.csv`, `A_sin_siniestros_lista.csv` y `A_sin_siniestros_resumen.csv`, `p0_esperado_A.csv`, `conteos_extremos_top20.csv`, `conteos_extremos_sensibilidad.csv` y los mapas `mapa_mu_predicha.png`, `mapa_priorizados_escenarios.png`, `mapa_A_con_y_sin_siniestros.png` y `mapa_A_vs_C.png`. Los mapas dibujan centroides (`RUTA_MALLA = None`); para dibujar líneas hay que indicar la capa de la MVI de la que salió el conjunto.

## 8. Requisitos y reproducibilidad

- Python 3.12.4. Versiones de referencia: `statsmodels` 0.14.6, `scikit-learn` 1.9.0, `pandas` 3.0.5, `numpy` 2.5.2, `geopandas` 1.1.4 (lista completa en `requirements.txt`). El cuaderno 6 avisa si la versión de `pandas` difiere de la registrada en `config.json`.
- Paquetes principales: `geopandas`, `pandas`, `numpy`, `scipy`, `statsmodels`, `scikit-learn`, `joblib`, `matplotlib`, `matplotlib-scalebar`, `seaborn`, `openpyxl`, `libpysal`, `pyogrio`, `shapely`, `pyproj`.
- **Atención con `requirements.txt`:** es un `pip freeze` de un entorno conda en Windows. La última línea (`zipp @ file:///C:/...`) apunta a una ruta local y fallará al instalar en otra máquina; además repite `zict` y `zipp`, y arrastra paquetes que ningún cuaderno usa (p. ej. `PyQt5`, `cvxpy`, `dask`, `bokeh`, `momepy`, `nose`). Para reproducir el entorno, instala los paquetes principales y fija las versiones de arriba.
- Algunos cuadernos guardados muestran avisos de incompatibilidad entre NumPy 2.x y módulos compilados con NumPy 1.x (visibles en las salidas de `priorizacion.ipynb`). Si aparecen errores, usa `numpy<2` o actualiza los paquetes afectados.
- Ejecuta los cuadernos en el orden de la sección 1, con **Restart & Run All**. Semilla del modelo: `1949`.
- El cuaderno 4 verifica el CRS de cada capa antes de las uniones espaciales y detiene la ejecución si faltan `MVI_VELREG` o `MVINUMC`. El cuaderno 6 verifica con `assert` que el CSV tenga el mismo tamaño, total de siniestros, orden y coordenadas que cuando se ajustó el modelo.

## 9. Control de versiones

- Ramas: `main` y `v2` (rama de trabajo actual).
- `.gitattributes` registra `*.gpkg`, `*.geojson` y `*.shp` en Git LFS, y `.gitignore` excluye `Data/raw/`, `Data/processed/`, `BackUp/` y `*.gpkg` (más una línea de ejemplo, `ruta/al/archivo-pesado.ext`, que puede borrarse). Los cuatro insumos de `Data/raw/` que sí están en el repositorio se agregaron de forma explícita (sección 3); los demás datos se regeneran con los cuadernos a partir de las fuentes de esa sección.

## 10. Uso de inteligencia artificial

En concordancia con los Lineamientos para el uso de Inteligencia Artificial de la Universidad Externado de Colombia, se declara el uso de herramientas de Inteligencia Artificial Generativa durante el desarrollo del presente trabajo, detallado por categorías y capítulos de la siguiente manera:

Corregido con IA (Numeral 2.1.2.c): aplicado a la totalidad del documento (Capítulos 1 a 5) para la revisión ortotipográfica, la corrección sintáctica y el ajuste de puntuación y del formato de citación APA.

Redactado con IA (Numeral 2.1.2.b): aplicado a los Capítulos 3 y 4, como apoyo para la estructuración del texto, la fluidez conceptual de la descripción metodológica y la presentación formal de los hallazgos cuantitativos.

Revisado con IA (Numeral 2.1.2.a): aplicado al Capítulo 5 (Conclusiones y Recomendaciones), para la verificación de estilo, la síntesis académica y la coherencia narrativa final.

Herramienta y periodo de uso: se empleó la herramienta Claude AI (versión Claude Sonnet 5.5, Anthropic) durante el periodo comprendido entre agosto de 2026 y noviembre de 2026.

Responsabilidad autoral e integridad técnica: el contenido sustantivo, el análisis geoespacial y estadístico, las formulaciones de los distintos modelos, la interpretación y las conclusiones fueron desarrollados de manera autónoma por los autores, quienes realizaron la revisión y validación crítica del contenido final. Asimismo, el código generado o modificado con apoyo de estas herramientas fue revisado, adaptado y validado en el entorno de desarrollo local por los autores de acuerdo con las necesidades metodológicas del estudio. La selección de los métodos, variables, parámetros, procedimientos de análisis y resultados obtenidos corresponde exclusivamente a los autores. La inteligencia artificial se utilizó estrictamente como herramienta de apoyo y no como sustituto del criterio técnico, científico y metodológico empleado en el trabajo.

**Responsabilidad autoral e integridad técnica:**
El contenido sustantivo, el procesamiento de datos geoespaciales, la formulación y calibración del modelo estadístico (binomial negativa), el análisis de resultados y las conclusiones son autoría exclusiva de los investigadores. El código generado o modificado con apoyo de la herramienta fue revisado, adaptado y validado técnicamente por los autores en el entorno de desarrollo. La inteligencia artificial actuó de manera complementaria como herramienta de apoyo editorial y no como sustituto del criterio técnico y metodológico de la investigación.
