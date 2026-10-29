# Modelo predictivo de priorización para intervenciones en infraestructura peatonal — Bogotá

Cuadernos de preprocesamiento, análisis exploratorio (EDA), unión espacial y modelado de la siniestralidad peatonal en Bogotá D.C. (2015–2020). El flujo produce un conjunto de datos **a nivel de vía** y estima sobre él una *función de desempeño de seguridad* (FDS) con modelos de conteo (Poisson / Binomial Negativa / ZINB).

**Autores:** Jhoan Santiago Toro Chaparro y Brayan Sebastian Hernández Nieto
**Directora:** Yessica Liceth Velásquez Castiblanco
**Universidad Externado de Colombia — Departamento de Matemáticas**

---

## 1. Flujo general

```
Data/raw ──► 3 EDA ──► Data/processed ──► Union_segmentos ──► segmentos_unificado.csv ──► modelo_FDS_peatonal_v4
```

| Orden | Cuaderno | Qué hace | Salidas en `Data/processed/` |
|---|---|---|---|
| 1 | `EDA_Siniestros_Viales_Bogota_peatones_indices_ajustados.ipynb` | Une el histórico georreferenciado (CSV) con las hojas del Excel de la SDM por `CODIGO_ACCIDENTE`, filtra a `CONDICION = PEATON`, limpia la edad, hace el EDA (temporal, espacial, gravedad, pruebas de asociación) y georreferencia. | `siniestros_procesados.gpkg` |
| 2 | `EDA_Malla_Vial_Puentes_zona_urbana_adecuado.ipynb` | EDA de la Malla Vial Integral y de los puentes. Reproyecta a EPSG:3116, imputa nulos (media/moda; KNN para `MVI_VELREG` y `MVINUMC`) y calcula longitudes reales en metros. | `malla_vial_procesada.gpkg`, `puentes_procesado.gpkg`, `puentes_peatonales_supuesto.gpkg` |
| 3 | `EDA_Malla_peatonal_semaforos_adecuado.ipynb` | EDA de la red peatonal y los semáforos. Reproyecta, elimina columnas vacías o constantes (análisis de entropía) e imputa faltantes en semáforos. | `peatonal_procesada.gpkg`, `semaforos_procesados.gpkg` |
| 4 | `Union_segmentos.ipynb` | Recorta al perímetro urbano (POT), descarta geometrías < 4 m y construye el dataset final **a nivel de vía** (vía completa, sin subdividir): distancia a puente y a semáforo, cobertura peatonal, conteo de siniestros (con `sjoin_nearest`) y población por localidad. Calcula también la tasa por 100.000 habitantes. | `segmentos_unificado.gpkg`, `segmentos_unificado.csv`, `siniestros_tasa_100k_localidad_anio.csv` |
| 5 | `modelo_FDS_peatonal_v4.ipynb` | **Modelo vigente.** Ajusta Nulo, Poisson, BN y ZINB; verifica la BN y la dispersión; aplica una regla de decisión declarada; evalúa con validación cruzada aleatoria y por bloques espaciales; calcula el I de Moran de los residuales y errores estándar robustos. | — |

**Orden de ejecución: 1 → 2 → 3 → 4 → 5.** Los cuadernos 1, 2 y 3 son independientes entre sí; el 4 necesita las salidas de los tres, y el 5 lee solo `segmentos_unificado.csv`.

### Otros cuadernos del repositorio

| Archivo | Estado |
|---|---|
| `modelo_FDS_peatonal.ipynb` | Versión anterior del modelo (solo validación cruzada aleatoria). Concluía que la BN colapsaba a α ≈ 0 y dejaba Poisson como modelo final; la v4 corrige esa verificación (ver sección 6). Se conserva como referencia; **no usar para resultados**. |
| `BackUp/modelo_FDS_peatonal_organizado*.ipynb` (3 versiones) | Respaldos de iteraciones previas del modelo. |

---

## 2. Estructura de carpetas 

```
.
├── Data/
│   ├── raw/          # fuentes originales (ver sección 3)
│   └── processed/    # salidas de los cuadernos
├── BackUp/           # versiones antiguas del modelo
├── *.ipynb
├── requirements.txt
└── README.md
```

Todas las rutas están escritas en relativo y con `Data/` en mayúscula inicial; se ejecutan desde la raíz del proyecto.

## 3. Datos de entrada (`Data/raw/`)

| Archivo | Fuente | Usado en |
|---|---|---|
| `DATO_ABIERTOS_siniestros_viales_consolidados_bogota_dc (3).xlsx` | SDM (Datos Abiertos) | Cuaderno 1 |
| `historico_siniestros_bogota_d_c_-.csv` | SDM | Cuaderno 1 |
| `Loca.shp` (+ `.shx`, etc.) | Localidades de Bogotá (mapas del EDA) | Cuaderno 1 |
| `Malla_Vial_Integral_Bogota_D_C (3).geojson` | IDECA / Catastro Bogotá | Cuaderno 2 |
| `Puentes bogota.gpkg` | IDU / IDECA | Cuaderno 2 |
| `red_peatonal_conexion (1).gpkg` | Datos Abiertos Bogotá | Cuaderno 3 |
| `semaforos_bogota.geojson` | SDM | Cuaderno 3 |
| `perimetrourbano (1).gpkg` | SDP (POT, Decreto 555 de 2021) | Cuaderno 4 |
| `localidades_bogota.gpkg` | SDP — polígonos de localidades (columna `LocNombre`) | Cuaderno 4 |
| `osb_demografia-poblacion-localidad.csv` | SDS / SaluData (convenio DANE-SDP), población por localidad 2005–2035 | Cuaderno 4 |



Los archivos de datos no se versionan en el repositorio (ver sección 8). Descárgalos de las fuentes oficiales y colócalos con estos nombres.

## 4. Convenciones del proyecto

- **CRS:** todas las capas se trabajan en **EPSG:3116** (MAGNA-SIRGAS / Bogotá, metros). Los cuadernos fuerzan el CRS en vez de asumirlo y fallan si una capa no lo trae definido.
- **Periodo de estudio:** 2015–2020. La hoja `SINIESTROS` del Excel llega hasta 2020 y el CSV hasta 2021, por lo que los años finales del CSV están incompletos en las variables que vienen del Excel.
- **Unidad de observación:** la **vía original completa** de la Malla Vial Integral (una fila = una vía), con su longitud real en `long_seg_m`. Ya no se subdivide en tramos de longitud fija.
- **Parámetros de diseño (`Union_segmentos.ipynb`):**

| Parámetro | Valor | Significado |
|---|---|---|
| `UMBRAL_RUIDO` | 4 m | Se descartan geometrías más cortas |
| `UMBRAL_SINIESTROS_M` | 30 m | Distancia máxima para asignar un siniestro a su vía más cercana |
| `UMBRAL_PEATONAL_M` | 15 m | Alcance para considerar un punto de la vía "cubierto" por malla peatonal |
| `PASO_COBERTURA_M` | 5 m | Separación aproximada de los puntos de muestreo a lo largo de cada vía |

- **Puentes y semáforos:** se resumen como distancia al más cercano (`sjoin_nearest`), sin umbral.
- **Empates:** un siniestro equidistante de varias vías (típico en intersecciones) se asigna a la de menor `seg_id`; es un criterio explícito y reproducible, sin fundamento sustantivo.
- **Variable objetivo:** `n_siniestros`, conteo de siniestros peatonales por vía (también se exportan `n_herido` y `n_muerto`).

## 5. Dataset final: `segmentos_unificado.csv`

131.704 vías, 14 columnas. Solo el 10,12 % (13.322 vías) registra al menos un siniestro peatonal; el 89,88 % tiene cero.

| Columna | Descripción |
|---|---|
| `long_seg_m` | Longitud de la vía (m); entra al modelo como offset `log(longitud)` |
| `MVI_VELREG` | Velocidad reglamentaria (imputada por KNN); en el modelo, variable categórica |
| `MVINUMC` | Número de calzadas (imputada por KNN) |
| `dist_puente_peatonal_m`, `dist_puente_vehicular_m` | Distancia al puente peatonal / vehicular más cercano |
| `dist_semaforo_m` | Distancia al semáforo más cercano |
| `ratio_cobertura_peatonal` | Proporción ∈ [0, 1] de la vía con malla peatonal a ≤ 15 m. Reemplaza a `dist_peatonal_m`, que era ≈ 0 m para casi todas las vías y codificaba igual "sin malla" y "pegada a la malla" |
| `n_siniestros`, `n_herido`, `n_muerto` | Conteos asignados a la vía |
| `poblacion_localidad`, `log_poblacion` | Población representativa de la localidad de la vía (promedio 2015–2020 sobre el centroide de la vía) |
| `x_m`, `y_m` | Centroide de la vía en EPSG:3116; se usan para el I de Moran y la validación por bloques |

La población de años no censales (todos salvo 2005 y 2018) es una **proyección o retroproyección**, no una medición directa; debe declararse así en el documento. `poblacion_localidad` está en el dataset pero **no es covariable del modelo v4**.

`siniestros_tasa_100k_localidad_anio.csv` contiene la tasa descriptiva de siniestros peatonales por 100.000 habitantes por localidad y año (cada siniestro se cruza con la población de su localidad en su año exacto). Es un indicador aparte, independiente del dataset por vía.

## 6. Modelo (`modelo_FDS_peatonal.ipynb`)

**Especificación.** Conteo `n_siniestros` con offset `log(long_seg_m)`. Covariables: `log1p` de las tres distancias (estandarizadas por pliegue), `ratio_cobertura_peatonal`, `MVINUMC` y dummies de `MVI_VELREG`. En el ZINB la inflación de ceros depende de la longitud estandarizada.

**Verificación de la BN.** El `alpha` se estima por perfil de verosimilitud (no depende del optimizador de `statsmodels`, que en la versión anterior devolvía α ≈ 0). Resultado: α = 3,168; razón de verosimilitud Poisson vs BN = 11.160 (p < 0,001 con corrección de frontera). Hay sobredispersión y la BN es el modelo base.

**Regla de decisión (declarada de antemano).** El ZINB reemplaza a la BN solo si la base es BN, mejora el BIC, mejora log-score y devianza fuera de muestra, y todos los ajustes convergen. Las cuatro condiciones se cumplen, por lo que el **modelo final es el ZINB**.

| Modelo | Parámetros | AIC | BIC |
|---|---|---|---|
| Nulo | 1 | 127.950,7 | 127.960,5 |
| Poisson | 7 | 111.923,0 | 111.991,5 |
| BN | 8 | 100.765,0 | 100.843,3 |
| **ZINB** | 10 | **100.564,0** | **100.661,9** |

**Validación (K = 5).** Dos esquemas: aleatoria y por bloques espaciales (celdas de 2.000 m, 132 celdas). Devianza explicada del ZINB: 0,172 (aleatoria) y 0,170 (bloques); ningún ajuste dejó de converger. La caída entre esquemas es pequeña.

**Autocorrelación espacial.** I de Moran de los residuales (8 vecinos, 199 permutaciones): 0,023 para el ZINB (frente a 0,055 del modelo nulo). Sigue siendo significativo (p = 0,005) por el tamaño de muestra, pero el correlograma cae a ≈ 0 antes de 1.000 m, menos que el lado del bloque.

**Coeficientes (BN de referencia, EE agrupados por celda).**

| Variable | IRR | IC 95 % |
|---|---|---|
| `dist_puente_peatonal_m` | 1,012 | 0,964 – 1,063 (no significativo) |
| `dist_puente_vehicular_m` | 1,119 | 1,067 – 1,174 |
| `dist_semaforo_m` | 0,797 | 0,755 – 0,842 |
| `ratio_cobertura_peatonal` | 2,176 | 2,069 – 2,289 |
| `MVINUMC` | 1,231 | 1,166 – 1,298 |
| `vel_60` | 1,398 | 1,251 – 1,562 |

Son asociaciones, no efectos causales.

**Limitaciones señaladas por los diagnósticos.**
- El modelo subestima los conteos bajos-medios (1 siniestro: observado/esperado = 1,11) y **sobreestima los altos** (≥ 5 siniestros: 0,52–0,68); en el último decil de predicción el esperado es 0,757 frente a 0,553 observado.
- **Sensibilidad a la longitud:** con `log(longitud)` como covariable libre la elasticidad es 0,50 (IC 95 % 0,46–0,55), no 1 como supone el offset (LR = 1.278, p ≈ 7·10⁻²⁸⁰). La devianza explicada por bloques sube de 0,156 a 0,186 en la BN.
- Los siniestros que no caen a ≤ 30 m de ninguna vía quedan fuera del modelo (cobertura reportada en la sección 4.6 de `Union_segmentos.ipynb`).

## 7. Requisitos y reproducibilidad

- Python 3.12 (entorno conda `proyecto`). Versiones usadas en la última ejecución del modelo: `statsmodels` 0.14.6, `scikit-learn` 1.9.0, `pandas` 3.0.5, `numpy` 2.5.2, `geopandas` 1.1.4.
- Paquetes principales: `geopandas`, `pandas`, `numpy`, `scipy`, `statsmodels`, `scikit-learn`, `matplotlib`, `matplotlib-scalebar`, `seaborn`, `openpyxl`, `libpysal`, `pyogrio`, `shapely`, `pyproj`.
- **Atención con `requirements.txt`:** es un `pip freeze` de un entorno conda en Windows; varias líneas (`@ file:///C:/...`) apuntan a rutas locales y fallarán al instalar en otra máquina. Para reproducir el entorno, instala los paquetes principales y fija las versiones de arriba.
- Algunos cuadernos guardados muestran avisos de incompatibilidad entre NumPy 2.x y módulos compilados con NumPy 1.x. Si aparecen errores, usa `numpy<2` o actualiza los paquetes afectados.
- Ejecuta los cuadernos en el orden de la sección 1, con **Restart & Run All**. Semilla del modelo: `1949`.
- El cuaderno 4 verifica el CRS de cada capa antes de las uniones espaciales y detiene la ejecución si faltan `MVI_VELREG` o `MVINUMC`.

## 8. Control de versiones

- Ramas: `main` y `v2` (rama de trabajo actual).
- `.gitattributes` registra `*.gpkg`, `*.geojson` y `*.shp` en Git LFS, y `.gitignore` excluye `Data/raw/`, `Data/processed/` y `*.gpkg`. Los datos se regeneran con los cuadernos a partir de las fuentes de la sección 3.

## 9. Uso de inteligencia artificial

## Declaración de Uso de Inteligencia Artificial

En concordancia con los *Lineamientos para el uso de Inteligencia Artificial* de la Universidad Externado de Colombia, se declara el uso de herramientas de Inteligencia Artificial Generativa durante el desarrollo del presente proyecto, estructurado según los siguientes apartados:

* **Corregido con IA (Numeral 2.1.2.c):** Aplicado a la totalidad del documento (Capítulos 1 a 6) para revisión de estilo, ortotipografía, corrección sintáctica y ajuste del formato de citación APA.
* **Redactado con IA (Numeral 2.1.2.b):** Aplicado a la Sección 3 (*Metodología*) y al Capítulo 4 (*Resultados*), como apoyo en la estructuración del texto y la fluidez conceptual de la descripción técnica.
* **Revisado con IA (Numeral 2.1.2.a):** Aplicado a la Sección 6 (*Conclusiones y Recomendaciones*), para la verificación sintética, coherencia narrativa y pulido del estilo académico.

**Herramienta y Periodo de Uso:**  
Se empleó **Claude AI (versione Claude 5.5 Sonnet, Anthropic)** durante el periodo comprendido entre agosto de 2026 y noviembre de 2026.

**Responsabilidad Autoral e Integridad Técnica:**  
El contenido sustantivo, el procesamiento de datos geoespaciales, la formulación y calibración del modelo estadístico (ZIBN), el análisis de resultados y las conclusiones son autoría exclusiva de los investigadores. El código generado o modificado con apoyo de la herramienta fue revisado, adaptado y validado técnicamente por los autores en el entorno de desarrollo. La inteligencia artificial actuó de manera complementaria como herramienta de apoyo editorial y no como sustituto del criterio técnico y metodológico de la investigación.
