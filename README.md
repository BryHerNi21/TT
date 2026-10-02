# Modelo predictivo de priorización para intervenciones en infraestructura peatonal — Bogotá

Cuadernos de preprocesamiento, análisis exploratorio (EDA) y unión espacial de las fuentes usadas para caracterizar el riesgo vial peatonal en Bogotá (2015–2020). Producen el conjunto de datos a nivel de vía que alimenta el modelo de conteo.

**Autores:** Jhoan Santiago Toro Chaparro y Brayan Sebastian Hernández Nieto
**Directora:** Yessica Liceth Velásquez Castiblanco
**Universidad Externado de Colombia — Departamento de Matemáticas**

---

## 1. Flujo general

```
Data/raw ──► EDA (3 cuadernos) ──► Data/processed ──► Uniones espaciales ──► dataset por vía
```

| Orden | Cuaderno | Qué hace | Salidas en `Data/processed/` |
|---|---|---|---|
| 1 | `EDA2_Siniestros_Viales_Bogota_peatones_indices_ajustados.ipynb` | Une el histórico georreferenciado (CSV) con las 5 hojas del Excel de la SDM por `CODIGO_ACCIDENTE`, filtra a `CONDICION = PEATON`, limpia la edad, hace el EDA (temporal, espacial, severidad, pruebas de asociación) y georreferencia. | `siniestros_procesados.gpkg` |
| 2 | `EDA_Malla_Vial_Puentes_zona_urbana_adecuado.ipynb` | EDA de la Malla Vial Integral y de los puentes. Reproyecta a EPSG:3116, imputa nulos (media/moda; KNN para `MVI_VELREG` y `MVINUMC`) y calcula longitudes reales en metros. | `malla_vial_procesada.gpkg`, `puentes_procesado.gpkg`, `puentes_peatonales_supuesto.gpkg` |
| 3 | `EDA_Malla_peatonal_semaforos_adecuado.ipynb` | EDA de la red peatonal y los semáforos. Reproyecta, elimina columnas vacías o constantes (análisis de entropía) e imputa faltantes en semáforos. | `peatonal_procesada.gpkg`, `semaforos_procesados.gpkg` |
| 4 | `Union_bases_malla_vial_adecuado.ipynb` | Recorta las capas al perímetro urbano (POT), excluye ruido (<4 m) y une puentes, semáforos y red peatonal a la malla vial con `sjoin_nearest`. | `malla_urbana_unificada.gpkg` (capas `malla_vial`, `puentes`, `semaforos`, `peatonal`) |
| 5 | `Union_malla_siniestralidad_adecuado.ipynb` | Toma la capa `malla_vial` del cuaderno 4 y asigna cada siniestro a su vía más cercana (umbral de 30 m). No exporta archivos. | — |
| 6 | `Union_segmentos.ipynb` | Construye el dataset final **a nivel de vía** (vía completa, sin subdividir): distancias a puente y semáforo, red peatonal asignada, conteo de siniestros y población por localidad. Calcula la tasa por 100.000 habitantes. | `segmentos_unificado.gpkg`, `segmentos_unificado.csv`, `siniestros_tasa_100k_localidad_anio.csv` |


Orden de ejecución: **1 → 2 → 3 → 4 → 5**. El cuaderno **6** es una ruta alternativa e independiente del 4 y el 5: necesita los archivos de los cuadernos 1, 2 y 3, y lee directamente el perímetro urbano y el anexo de población.

---

## 2. Estructura de carpetas esperada

```
.
├── Data/
│   ├── raw/          # fuentes originales (ver sección 3)
│   └── processed/    # salidas de los cuadernos
└── *.ipynb
```

Las rutas están escritas en relativo. En `Union_segmentos.ipynb` la ruta del anexo de población usa `data/raw/` en minúscula, lo que falla en sistemas sensibles a mayúsculas (Linux, macOS). Conviene unificarla con `Data/raw/`.

## 3. Datos de entrada (`Data/raw/`)

| Archivo | Fuente | Usado en |
|---|---|---|
| `DATO_ABIERTOS_siniestros_viales_consolidados_bogota_dc (3).xlsx` | SDM (Datos Abiertos) | Cuaderno 1 |
| `historico_siniestros_bogota_d_c_-.csv` | SDM | Cuaderno 1 |
| `Malla_Vial_Integral_Bogota_D_C (3).geojson` | IDECA / Catastro Bogotá | Cuaderno 2 |
| `Puentes bogota.gpkg` | IDU / IDECA | Cuaderno 2 |
| `red_peatonal_conexion (1).gpkg` | Datos Abiertos Bogotá | Cuaderno 3 |
| `semaforos_bogota.geojson` | SDM | Cuaderno 3 |
| `perimetrourbano (1).gpkg` | SDP (POT, Decreto 555 de 2021) | Cuadernos 4 y 6 |
| `anexo-proyecciones-poblacion-bogota-desagreacion-loc-2018-2035-UPZ-2018-2024.xlsx` | DANE / SDP | Cuaderno 6 |

Los archivos no se incluyen en el repositorio. Descárgalos de las fuentes oficiales y colócalos con estos nombres.

## 4. Convenciones del proyecto

- **CRS:** todas las capas se trabajan en **EPSG:3116** (MAGNA-SIRGAS / Bogotá, metros).
- **Unidad de observación:** los cuadernos 1–5 y el 6 difieren en la agregación (ver sección 6). La unidad vigente en `Union_segmentos.ipynb` es la **vía original completa**, con su longitud en `long_seg_m`.
- **Umbrales de asignación:** siniestros ≤ 30 m, red peatonal ≤ 15 m. Puentes y semáforos se resumen como distancia al más cercano, sin umbral.
- **Ruido de digitalización:** se descartan geometrías de menos de 4 m.
- **Variable objetivo:** conteo de peatones afectados por vía (`n_peatones_afectados`); el conteo de eventos está en `n_siniestros`.

## 5. Requisitos

- Python 3.12 (entorno conda `proyecto`).
- Paquetes: `geopandas`, `pandas`, `numpy`, `matplotlib`, `seaborn`, `scipy`, `scikit-learn`, `matplotlib-scalebar`, `openpyxl` y, de forma opcional, `statsmodels` (VIF).
- Los cuadernos guardados muestran avisos de incompatibilidad entre NumPy 2.x y módulos compilados con NumPy 1.x. Si aparecen errores, usa `numpy<2` o actualiza los paquetes afectados.

## 6. Reproducibilidad

- Ejecuta los cuadernos en el orden de la sección 1, con **Restart & Run All**.
- Los cuadernos 4–6 verifican el CRS de cada capa antes de las uniones espaciales.
- Documenta las versiones de las librerías (`pip freeze` o `conda env export`) junto con los resultados finales.

## 7. Uso de inteligencia artificial

Se utilizó Claude AI como apoyo en redacción, revisión de estilo y corrección, conforme a los lineamientos de la Universidad Externado de Colombia (apartados 2.1.2.b y 2.1.2.c). El contenido, el análisis y las conclusiones son de los autores.
