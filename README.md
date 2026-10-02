# LRE-CR
Climate risk assessment of Costa Rica’s 41 phytogeographic subunits under IUCN Red List of Ecosystems C2a using Random Forest and 13 CMIP6 GCMs.
# LRE-CR

**Evaluación climática del subcriterio C2a de la Lista Roja de Ecosistemas para las Subunidades Fitogeográficas de Costa Rica**

> **Estado del repositorio: versión 1 (v1).**  
> Esta versión documenta y conserva el flujo desarrollado y validado durante la construcción metodológica del proyecto. La implementación actual todavía contiene dependencias de rutas locales y conserva una rama piloto/diagnóstica. Una versión posterior del repositorio consolidará scripts agnósticos a las rutas y eliminará del flujo publicado los análisis piloto.

---

## Descripción

**LRE-CR** implementa un flujo reproducible en **R** para evaluar el riesgo climático de las **Subunidades Fitogeográficas (SUF)** de Costa Rica bajo el subcriterio **C2a** de la **Lista Roja de Ecosistemas (LRE) de la UICN**.

El enfoque combina:

- la cartografía actual de 41 SUF;
- 19 variables bioclimáticas (`bio1`–`bio19`);
- un clasificador **Random Forest (RF)** multiclase;
- validación espacial **4-fold** mediante bloques de 10 km;
- 13 **Modelos Climáticos Globales (GCM)** de CMIP6;
- escenario **SSP2-4.5**;
- horizonte **2061–2080**;
- cálculo de **Relative Severity (RS; severidad relativa)**;
- clasificación C2a independiente para cada GCM;
- síntesis de incertidumbre entre GCM;
- controles de calidad asociados a validación, evaluabilidad y novedad climática.

El flujo definitivo utiliza directamente las 19 variables BIO. Aunque algunos directorios de insumos conservan nombres históricos que contienen `PCA`, **esta implementación no usa Análisis de Componentes Principales (PCA)**.

---

## Objetivo

El objetivo es estimar el cambio futuro de la idoneidad bioclimática de cada SUF y traducir ese cambio a una evaluación C2a, manteniendo explícita la incertidumbre asociada a:

1. desempeño del modelo de clasificación;
2. disponibilidad de observaciones por SUF;
3. diferencias entre los 13 GCM;
4. proporción del área efectivamente evaluable;
5. extrapolación hacia condiciones bioclimáticas novedosas.

La categoría C2a no se modifica de forma automática por las capas de control de calidad. Estas capas acompañan el resultado para comunicar su robustez y sus limitaciones.

---

## Alcance de esta versión

La **v1** corresponde a la implementación actualmente validada.

El flujo principal es:

```text
00 Configuración
  → 01 Auditoría de insumos
  → 02 Preparación del dominio común
  → 09 Entrenamiento y validación RF definitiva
  → 10 Proyección actual + 13 GCM
  → 11 Severidad relativa + C2a
  → 12 Síntesis final + control de calidad
```

Los scripts `03–08` corresponden a una **rama piloto y diagnóstica** desarrollada inicialmente sobre la SUF 61. No forman parte del pipeline definitivo y se conservan únicamente como trazabilidad del desarrollo metodológico.

Un análisis exploratorio de consenso espacial `9/13` fue realizado dentro del piloto, pero **no se utiliza en la evaluación C2a definitiva**.

### Hoja de ruta posterior a v1

La siguiente versión del repositorio buscará:

- eliminar dependencias de rutas absolutas;
- centralizar la configuración de entradas y salidas;
- prescindir de la rama piloto `03–08`;
- conservar únicamente el pipeline definitivo;
- mejorar la trazabilidad mediante hashes y control explícito de versiones;
- documentar completamente la preparación de los insumos climáticos originales;
- incorporar un entorno reproducible de dependencias.

---

## Área de estudio y unidades de análisis

El análisis comprende **41 Subunidades Fitogeográficas de Costa Rica**.

Características espaciales de la ejecución actual:

| Parámetro | Valor |
|---|---|
| CRS | EPSG:5367 — CR05 / CRTM05 |
| Resolución | 1 km × 1 km |
| Campo identificador | `SUF_int` |
| SUF excluidas | 181 y 182 |
| Celdas SUF dentro de la grilla | 51 012 |
| Celdas comunes utilizables | 50 841 |
| Celdas excluidas dentro de la grilla | 171 |

El soporte común incluye únicamente celdas con información válida para las 19 BIO en clima actual y en los 13 GCM futuros. Por tanto, el dominio analítico no representa necesariamente el 100 % de los polígonos vectoriales originales.

---

## Datos climáticos

### Variables predictoras

El modelo utiliza las 19 variables bioclimáticas:

```text
bio1, bio2, bio3, bio4, bio5, bio6, bio7, bio8, bio9,
bio10, bio11, bio12, bio13, bio14, bio15, bio16, bio17,
bio18, bio19
```

### Escenario futuro

| Parámetro | Valor |
|---|---|
| Escenario | SSP2-4.5 (`ssp245`) |
| Horizonte | 2061–2080 |
| Número de GCM | 13 |

GCM utilizados:

1. `ACCESS-CM2`
2. `BCC-CSM2-MR`
3. `CMCC-ESM2`
4. `EC-Earth3-Veg`
5. `FIO-ESM-2-0`
6. `GISS-E2-1-G`
7. `HadGEM3-GC31-LL`
8. `INM-CM5-0`
9. `IPSL-CM6A-LR`
10. `MIROC6`
11. `MPI-ESM1-2-HR`
12. `MRI-ESM2-0`
13. `UKESM1-0-LL`

Cada GCM se procesa y evalúa **de forma independiente**. No se construye un raster climático de consenso ni se promedian primero las condiciones futuras.

---

## Metodología

### 1. Auditoría y preparación

Los scripts `01` y `02` verifican y preparan:

- códigos SUF;
- geometrías;
- CRS;
- resolución;
- correspondencia BIO1–BIO19;
- cobertura climática;
- geometría común;
- máscara de celdas válidas;
- rasterización de SUF;
- tabla maestra de celdas.

La tabla utilizada para modelación contiene:

```text
cell, x, y, SUF_int, bio1, ..., bio19
```

La auditoría inicial identifica problemas de cobertura y compatibilidad, mientras que el script `02` construye el **soporte común utilizable** sobre el que se ejecuta el análisis.

---

### 2. Random Forest multiclase

Se ajusta un único **Random Forest multiclase** con las 41 SUF.

Parámetros principales:

| Parámetro | Implementación |
|---|---|
| Algoritmo | `randomForest::randomForest` |
| Clases | 41 SUF |
| Predictoras | BIO1–BIO19 |
| Semilla base | `20260930` |
| Muestreo | Estratificado por SUF |
| Máximo por SUF | 500 celdas |
| SUF con <500 celdas | Se utilizan todas |
| Bloques espaciales | 10 km × 10 km |
| Folds espaciales | 4 |
| `mtry` candidatos | 3, 5, 7, 10, 14 |
| Árboles para ajuste | 200 |
| Árboles por modelo CV | 800 |
| Árboles RF final | 1000 |
| Fracción por clase dentro del RF | 70 % |

La ejecución de referencia seleccionó:

```text
mtry = 5
```

---

### 3. Validación espacial

La validación utiliza cuatro grupos espaciales de bloques de 10 km.

La asignación es determinista:

```r
fold = 1 +
       (floor(x / 10000) %% 2) +
       2 * (floor(y / 10000) %% 2)
```

Cada fold actúa una vez como conjunto de evaluación y los otros tres como entrenamiento. Las predicciones resultantes son **OOF (out-of-fold)**.

El ajuste de `mtry` se realiza dentro del entrenamiento de cada fold. Posteriormente se repite el ajuste utilizando toda la muestra para entrenar el RF definitivo.

La ejecución de referencia produjo:

```text
17 249 celdas de entrenamiento/muestreo
Exactitud multiclase espacial OOF = 0.8381
```

Calidad de validación por número de observaciones objetivo OOF:

| Observaciones | Clase QC |
|---:|---|
| ≥100 | `VALIDACION_ROBUSTA` |
| 30–99 | `VALIDACION_LIMITADA` |
| <30 | `VALIDACION_MUY_LIMITADA` |

En la ejecución de referencia:

- 36 SUF: validación robusta;
- SUF 72, 83 y 93: validación limitada;
- SUF 104 y 174: validación muy limitada.

Estas etiquetas describen **cantidad de evidencia**, no una categoría ecológica ni un umbral de desempeño.

---

### 4. Umbral `spec_sens`

Para cada SUF se calcula un umbral independiente utilizando los votos **OOB (out-of-bag)** del RF definitivo.

El procedimiento:

1. construye una evaluación uno-contra-resto;
2. calcula sensibilidad y especificidad para los posibles umbrales;
3. maximiza:

```text
J = sensibilidad + especificidad − 1
```

4. utiliza el umbral que maximiza el índice de Youden;
5. en caso de empate, selecciona el menor umbral finito.

Estos umbrales son posteriormente utilizados como referencia operativa para el cálculo de severidad relativa.

---

### 5. Proyección climática

El mismo RF definitivo se aplica a:

- clima actual;
- cada uno de los 13 GCM futuros.

Cada salida es un GeoTIFF multibanda con 41 capas:

```text
SUF_<codigo>
```

Los valores son votos/proporciones de clase del clasificador RF y **no deben interpretarse como probabilidades ecológicas calibradas**.

---

## Severidad relativa

Para cada SUF `k`, celda `i` y GCM `g`, la implementación calcula:

```text
RS_raw(i,k,g) =
100 × [p_actual(i,k) − p_futuro(i,k,g)]
      ---------------------------------
      [p_actual(i,k) − spec_sens(k)]
```

y luego:

```text
RS = min(100, max(0, RS_raw))
```

donde:

- `p_actual` = voto RF de la SUF bajo clima actual;
- `p_futuro` = voto RF bajo el GCM futuro;
- `spec_sens` = umbral específico de la SUF.

Las mejoras futuras (`RS < 0`) se fijan en 0 y los valores >100 se limitan a 100.

### Dominio espacial de RS

RS se calcula únicamente en celdas que cumplen:

```text
SUF_int == k
y
p_actual(k) > spec_sens(k)
```

Esto asegura un denominador positivo.

Las celdas de la distribución actual que no cumplen la condición se registran como **no evaluables** y no se fuerzan dentro del cálculo.

### Severidad del ecosistema

Para cada combinación SUF × GCM se calcula una media ponderada por el área real de las celdas:

```text
RS_media(k,g) =
Σ [area(i) × RS(i,k,g)]
------------------------
Σ area(i)
```

También se conservan estadísticas espaciales adicionales y porcentajes del área con `RS ≥30`, `RS ≥50` y `RS ≥80`.

---

## Clasificación C2a

### Categoría por GCM

La implementación operativa de este proyecto clasifica la **RS media ponderada** de cada SUF bajo cada GCM:

| RS media | Categoría |
|---:|---|
| ≥80 | CR — En Peligro Crítico |
| ≥50 y <80 | EN — En Peligro |
| ≥30 y <50 | VU — Vulnerable |
| <30 | LC — Preocupación Menor |

Las estadísticas de extensión espacial con RS ≥30/50/80 se conservan como diagnóstico, pero **no intervienen directamente en la regla de clasificación implementada**.

---

### Síntesis de los 13 GCM

Las categorías se codifican ordinalmente:

```text
LC = 0
VU = 1
EN = 2
CR = 3
```

Para cada SUF:

- `C2a_central` = mediana ordinal de los 13 resultados;
- límite inferior = cuantil empírico 0.05;
- límite superior = cuantil empírico 0.95;
- se conservan las frecuencias de GCM en cada categoría.

Con 13 GCM y `quantile(type = 1)`, los límites 5–95 % coinciden en la práctica con los extremos observados. Deben interpretarse como **límites plausibles empíricos**, no como intervalos de confianza inferenciales.

### Datos Insuficientes (DD)

La implementación final utiliza:

```text
si límite inferior = LC
y límite superior = CR
→ C2a_reportada = DD
```

En los demás casos:

```text
C2a_reportada = C2a_central
```

La categoría central se conserva incluso cuando la categoría reportada es DD.

---

## Novedad climática y extrapolación

El script `12` incorpora un diagnóstico de **novedad climática / extrapolación marginal**, inspirado en el concepto de **MESS (Multivariate Environmental Similarity Surface)**.

Para cada BIO se comparan los valores futuros contra el rango observado en el dominio climático actual:

```text
score = 0                              dentro del rango actual
score < 0                              fuera del rango actual
```

El score de una celda corresponde al mínimo entre las 19 BIO.

Por tanto:

```text
score < 0
```

indica que al menos una variable futura está fuera de su rango marginal actual.

Este diagnóstico:

- calcula porcentaje de área novedosa por SUF y GCM;
- identifica la BIO limitante;
- resume la novedad entre los 13 GCM;
- **no modifica la categoría C2a**.

> Nota: este diagnóstico final no es idéntico al MESS empírico más completo utilizado durante la etapa piloto. En la implementación definitiva, los valores dentro del rango actual reciben 0; por ello se recomienda describirlo como **novedad climática / extrapolación marginal**.

---

## Resultados de referencia de la v1

La ejecución de referencia corresponde a:

```text
Escenario: SSP2-4.5
Horizonte: 2061–2080
41 SUF
13 GCM
533 evaluaciones SUF × GCM
```

Distribución final:

| Categoría reportada | Número de SUF |
|---|---:|
| LC | 1 |
| VU | 4 |
| EN | 16 |
| CR | 19 |
| DD | 1 |

En esta ejecución:

- 35 de 41 SUF quedaron clasificadas como EN o CR;
- la SUF 31 fue reportada como DD por abarcar resultados desde LC hasta CR entre GCM;
- la mediana entre SUF del área utilizable para RS fue 99.819 %;
- la mediana entre SUF de la novedad climática mediana entre GCM fue 5.85 %.

Estos resultados corresponden a una ejecución concreta y deben interpretarse junto con las capas de control de calidad.

---

## Productos finales

La ejecución final genera, entre otros:

| Archivo | Contenido |
|---|---|
| `TABLA_MAESTRA_FINAL_C2a_41SUF.csv` | Tabla integrada de resultados y QC |
| `FINAL_C2a_reportada_41SUF.tif` | Categoría C2a final por SUF |
| `FINAL_RS_mediana_13GCM_41SUF.tif` | Mediana entre GCM de la RS media por SUF |
| `QC_amplitud_C2a_90_41SUF.tif` | Amplitud ordinal entre límites plausibles |
| `QC_calidad_validacion_RF_41SUF.tif` | Calidad de validación RF |
| `QC_novedad_climatica_mediana_pct_41SUF.tif` | Novedad climática mediana por SUF |
| `QC_area_no_evaluable_pct_41SUF.tif` | Área no evaluable para RS |
| `novedad_climatica_por_SUF_GCM.csv` | Novedad por SUF × GCM |
| `novedad_climatica_global_por_GCM.csv` | Diagnóstico global por GCM |
| `resumen_novedad_climatica_41SUF.csv` | Síntesis de novedad por SUF |
| `leyenda_FINAL_C2a.csv` | Codificación de categorías |
| `DICCIONARIO_TABLA_MAESTRA.csv` | Diccionario parcial de resultados |
| `RESUMEN_FINAL_C2a.txt` | Resumen de ejecución |

Códigos del raster C2a:

```text
0 = LC
1 = VU
2 = EN
3 = CR
9 = DD
```

---

## Estructura del proyecto v1

La estructura local de desarrollo contiene tanto el pipeline definitivo como productos intermedios y la rama piloto:

```text
modelacion_v3/
├── scripts/
│   ├── 00_config.R
│   ├── 01_auditar_inputs.R
│   ├── 02_preparar_inputs.R
│   ├── 03–08                         # rama piloto / diagnóstica
│   ├── 09_entrenar_modelo_final*.R
│   ├── 10_proyectar_modelo_final_41SUF_13GCM.R
│   ├── 11_severidad_C2a_final_41SUF_13GCM.R
│   └── 12_sintesis_final_C2a_41SUF.R
├── 00_inputs_audit/
├── 01_clima_actual/
├── 01_clima_futuro/
├── 02_entrenamiento/
├── 03_modelo/
├── 04_proyecciones/
├── 05_severidad/
├── 06_c2a/
├── 07_resultados_finales/
├── qc/
└── logs/
```

Para GitHub se recomienda no versionar el árbol completo de resultados derivados. Una organización más limpia para la v1 es:

```text
LRE-CR/
├── README.md
├── scripts/
│   ├── 00_config.R
│   ├── 01_auditar_inputs.R
│   ├── 02_preparar_inputs.R
│   ├── 09_entrenar_modelo_final.R
│   ├── 10_proyectar_modelo_final_41SUF_13GCM.R
│   ├── 11_severidad_C2a_final_41SUF_13GCM.R
│   └── 12_sintesis_final_C2a_41SUF.R
├── archive/
│   └── pilot_SUF61/
│       └── 03–08
├── docs/
│   └── ...
└── results/
    └── productos seleccionados de la ejecución publicada
```

---

## Ejecución

### Dependencias directas

Los scripts utilizan principalmente:

```r
terra
randomForest
```

y funciones de R base, `stats`, `utils`, `graphics` y `grDevices`.

La ejecución de referencia registró:

```text
R 4.6.1
terra 1.9-50
randomForest 4.7-1.2
Ubuntu 26.04.1 LTS
```

Estas son versiones registradas de la ejecución de referencia, no requisitos mínimos formalmente definidos.

### Orden de ejecución

El flujo definitivo es:

```r
source("scripts/01_auditar_inputs.R")
source("scripts/02_preparar_inputs.R")
source("scripts/09_entrenar_modelo_final.R")
source("scripts/10_proyectar_modelo_final_41SUF_13GCM.R")
source("scripts/11_severidad_C2a_final_41SUF_13GCM.R")
source("scripts/12_sintesis_final_C2a_41SUF.R")
```

`00_config.R` es cargado por los scripts operativos.

> **Limitación de la v1:** varios scripts conservan dependencias de rutas absolutas asociadas al entorno local de desarrollo. Para reproducir el flujo en otra computadora puede ser necesario editar las rutas de configuración. Este problema será eliminado en una versión posterior.

---

## Insumos esperados

La implementación actual espera conceptualmente:

```text
SUF cartografiadas
└── GeoPackage con campo SUF_int

Clima actual
└── 19 variables BIO

Clima futuro
└── 13 GCM × 19 variables BIO
```

Los insumos originales no se almacenan dentro del repositorio Git debido a su tamaño y a que su procedencia, licencia y preparación deben documentarse por separado.

La v1 no reconstruye automáticamente desde cero los pasos previos de descarga, recorte, reproyección y remuestreo de los insumos climáticos.

---

## Archivos que no deberían versionarse en Git

Se recomienda excluir del repositorio ordinario:

```gitignore
.RData
.Rhistory
*.aux.xml

00_inputs_audit/
01_clima_actual/
01_clima_futuro/
02_entrenamiento/
03_modelo/
04_proyecciones/
05_severidad/
06_c2a/
07_resultados_finales/
qc/
logs/
```

También deberían excluirse:

- modelos RF `.rds`;
- predicciones OOF;
- punteros `*_actual.rds`;
- GeoTIFF climáticos preparados;
- proyecciones futuras;
- ejecuciones intermedias y parciales;
- archivos temporales o históricos.

Las tablas y figuras finales pequeñas que se deseen publicar deberían copiarse explícitamente a una carpeta `results/` controlada.

---

## Limitaciones

La v1 debe interpretarse considerando las siguientes limitaciones:

1. **Rutas locales.** La implementación todavía no es agnóstica al sistema de archivos.
2. **Preparación de insumos.** Los pasos anteriores a `modelacion_v3` no están completamente automatizados ni documentados dentro del código.
3. **Soporte común.** El análisis se realiza sobre el dominio climático común utilizable y no necesariamente sobre la totalidad de los polígonos SUF originales.
4. **Cobertura.** La auditoría inicial conservada reportó pendientes de cobertura; el script `02` construye posteriormente el soporte válido para modelación.
5. **Autocorrelación espacial.** Los bloques de 10 km mejoran la independencia respecto de un muestreo aleatorio, pero no existe un buffer explícito entre train y test.
6. **Clases pequeñas.** Algunas SUF tienen pocas observaciones disponibles y se reportan con validación limitada o muy limitada.
7. **Umbrales.** Las métricas binarias OOF utilizan umbrales `spec_sens` estimados a partir del RF final.
8. **Votos RF.** Los votos de clase no constituyen probabilidades ecológicas calibradas.
9. **Novedad climática.** El diagnóstico final identifica extrapolación marginal, pero un score no negativo no garantiza analogía multivariada completa.
10. **Incertidumbre climática.** Se representa mediante variación entre 13 GCM; no se propagan explícitamente todas las fuentes de incertidumbre del RF, cartografía, umbral o escenarios alternativos.
11. **Límites plausibles.** Con 13 GCM y cuantiles empíricos `type=1`, los límites 5–95 % corresponden a los extremos observados.
12. **C2a.** Esta es una implementación operativa del indicador climático inspirada en aplicaciones publicadas de la LRE; la equivalencia ecológica entre los votos RF, `spec_sens` y un umbral real de colapso requiere interpretación ecológica independiente.

---

## Base metodológica

El flujo se desarrolló tomando como referencia aplicaciones de modelación bioclimática y evaluación del riesgo de colapso de ecosistemas dentro del marco de la Lista Roja de Ecosistemas.

En particular, la metodología se apoya en los principios de:

- modelar cambios futuros de idoneidad ambiental;
- evaluar el desempeño y la transferencia espacial de modelos;
- utilizar múltiples escenarios/modelos climáticos para representar incertidumbre;
- evaluar extrapolación ambiental;
- expresar degradación mediante severidad relativa;
- mantener explícita la incertidumbre asociada a los resultados.

La implementación específica de LRE-CR debe entenderse como una **adaptación computacional para las SUF de Costa Rica**, no como una reproducción literal de todos los componentes de una evaluación completa de la Lista Roja de Ecosistemas.

---

## Referencias principales

Ferrer-Paris, J. R., Zager, I., Oliveira-Miranda, M. A., Rodríguez, J. P., González-Gil, M., Miller, R. M., Keith, D. A., Josse, C., Zambrana-Torrelio, C., & Barrow, E. (2019). **An ecosystem risk assessment of temperate and tropical forests of the Americas with an outlook on future conservation strategies.** *Conservation Letters, 12*, e12623. https://doi.org/10.1111/conl.12623

Keith, D. A., Elith, J., & Simpson, C. C. (2014). **Predicting distribution changes of a mire ecosystem under future climates.** *Diversity and Distributions, 20*, 440–454. https://doi.org/10.1111/ddi.12173

Murray, N. J., Keith, D. A., Duncan, A., Tizard, R., Ferrer-Paris, J. R., Worthington, T. A., Armstrong, K., Hlaing, N., Htut, W. T., Oo, A. H., Ya, K. Z., & Grantham, H. (2020). **Myanmar's terrestrial ecosystems: Status, threats and conservation opportunities.** *Biological Conservation, 252*, 108834. https://doi.org/10.1016/j.biocon.2020.108834

Tierney, D. A. (2022). **Linking restoration to the IUCN red list for ecosystems: A case study of how we might track the Earth's ecosystems.** *Austral Ecology, 47*, 852–866. https://doi.org/10.1111/aec.13168

---

## Reproducibilidad

La semilla base utilizada en la ejecución de referencia es:

```text
20260930
```

Cada etapa conserva archivos de trazabilidad como:

```text
config_utilizada.R
sessionInfo.txt
consola.txt
estado_*.txt
```

Sin embargo, la v1 todavía no incluye:

- hashes de insumos;
- hashes de scripts;
- bloqueo de versiones de paquetes;
- descarga automatizada de datos;
- reconstrucción completa de los insumos originales.

Estos elementos forman parte de la hoja de ruta para una versión posterior.

---

## Citación

La forma de citación del repositorio será definida cuando se asigne una versión pública estable y, si corresponde, un DOI.

Mientras tanto, los resultados deben citar explícitamente las fuentes originales de datos climáticos, la cartografía SUF y la literatura metodológica utilizada.

---

## Licencia

**Pendiente de definir.**

Antes de distribuir públicamente el código y los productos derivados debe verificarse la compatibilidad entre:

- licencia del código;
- licencia de la cartografía SUF;
- licencia de los datos climáticos;
- condiciones de redistribución de los productos derivados.

---

## Contacto

**Proyecto LRE-CR — Costa Rica**

Repositorio en desarrollo.
