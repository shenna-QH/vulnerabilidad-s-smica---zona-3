
# Sistema de Adquisición, Depuración y Acondicionamiento Automático de Registros Sísmicos del CISMID

**Aplicaciones de IA en Estructuras — UPC**
Trabajo 2 — Grupo 5

Sistema desarrollado en Python para automatizar la adquisición, validación, acondicionamiento y visualización de registros acelerográficos del repositorio del **CISMID**, correspondientes a la **Zona Sísmica 3** del Perú, con fines de selección de registros para **análisis dinámico no lineal (NLTHA)**.

---

## Tabla de contenidos

- [Descripción](#descripción)
- [Metodología](#metodología)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Instalación](#instalación)
- [Uso](#uso)
- [Plan de algoritmos](#plan-de-algoritmos)
- [Indicadores de evaluación](#indicadores-de-evaluación)
- [Referencias](#referencias)
- [Integrantes](#integrantes)
- [Licencia](#licencia)

---

## Descripción

El proyecto aborda el problema de acondicionar registros sísmicos reales para su uso en análisis estructural dinámico, automatizando etapas que tradicionalmente se realizan de forma manual con software independiente (SeismoSignal, SeismoMatch). El sistema cubre todo el flujo: desde la descarga de registros y metadatos del CISMID hasta la generación de espectros escalados listos para análisis no lineal.

No se emplean modelos de aprendizaje automático como núcleo del procesamiento de señal —ese procesamiento se basa en métodos deterministas ya validados en ingeniería sísmica (corrección de línea base, filtrado Butterworth, integración de Newmark-β)—, aunque sí se usan técnicas de ML (clustering, correlación) en la etapa exploratoria de selección de registros.

## Metodología

El proyecto se enmarca en **Veridical Data Science (VDS)** (Yu & Barter, 2024), priorizando:

- **Trazabilidad**: cada registro procesado conserva su origen y los parámetros aplicados en cada etapa.
- **Estabilidad**: los resultados se validan comparándolos contra herramientas de referencia (SeismoSignal, SeismoMatch).
- **Reproducibilidad**: pipeline versionado y automatizado vía GitHub Actions, configuración desacoplada del código (`config/filters.yaml`).

## Estructura del repositorio

```
├── config/
│   └── filters.yaml          # criterios de filtrado de registros CISMID
├── src/
│   ├── filter_records.py     # filtrado de metadatos según config/filters.yaml
│   └── spectral_scaling.py   # escalamiento espectral y métricas de ajuste
├── data/
│   ├── raw/                  # registros y metadatos crudos del CISMID
│   └── processed/            # registros filtrados / espectros escalados
├── notebooks/
│   └── EDA_T2.ipynb          # análisis exploratorio de datos
├── docs/
│   ├── T1_Propuesta.docx
│   └── T2_Informe.docx       # informe técnico completo
├── .github/workflows/
│   └── pipeline.yml          # CI: corre el filtrado automáticamente en cada push
└── README.md
```

## Instalación

```bash
git clone https://github.com/<usuario-u-org>/<nombre-repo>.git
cd <nombre-repo>
pip install -r requirements.txt
```

Dependencias principales: `pandas`, `numpy`, `matplotlib`, `plotly`, `folium`, `scikit-learn`, `scipy`, `pyyaml`.

## Uso

**1. Filtrar registros del CISMID** según los criterios definidos en `config/filters.yaml`:

```bash
python src/filter_records.py \
  --input data/raw/cismid_metadata.csv \
  --config config/filters.yaml
```

**2. Calcular el escalamiento espectral** de un registro respecto a un espectro objetivo (E.030 o CMS):

```bash
python src/spectral_scaling.py \
  --record data/processed/espectro_registro.csv \
  --target data/processed/espectro_objetivo.csv \
  --t_min 0.1 --t_max 2.0
```

El pipeline también corre automáticamente vía **GitHub Actions** cada vez que se sube un nuevo archivo a `data/raw/` o se modifica `config/filters.yaml`; el CSV filtrado queda disponible como artifact en la pestaña **Actions**.

## Plan de algoritmos

| Etapa | Algoritmo | Propósito |
|---|---|---|
| Adquisición | Consulta/extracción automática (Pandas) | Descargar registros y metadatos del CISMID con trazabilidad |
| Limpieza | Validación de nulos, duplicados, formato | Garantizar solo registros válidos |
| Corrección de línea base | Detrending / ajuste polinomial | Eliminar desplazamientos residuales espurios |
| Filtrado digital | Butterworth fase cero (`filtfilt`) | Eliminar ruido sin desfasar la señal |
| Integración numérica | Regla trapezoidal acumulativa | Obtener velocidad y desplazamiento |
| Espectro de respuesta | Newmark-β (SDOF) | Calcular Sa, Sv, Sd |
| Escalamiento espectral | Factor de escala vs. espectro objetivo (E.030 / CMS) | Acondicionar el registro para NLTHA |
| Visualización | Plotly / Dash | Interpretar resultados interactivamente |

## Indicadores de evaluación

| Indicador | Qué mide | Criterio de éxito |
|---|---|---|
| Error residual de línea base | Desplazamiento residual final tras integración | Cercano a cero |
| Conservación de características físicas | PGA, PGV, PGD, duración significativa, intensidad de Arias | Variaciones dentro de rangos aceptables |
| Compatibilidad espectral | RMSE, error % medio, diferencia máxima espectral | Minimizar diferencia registro vs. objetivo |
| Eficiencia de procesamiento | Tiempo promedio/total por lote | Reducción significativa vs. proceso manual |
| Calidad de selección | % de registros válidos / descartados | Base de datos depurada |
| Exactitud espectral | Comparación contra SeismoSignal / SeismoMatch | Resultados equivalentes a software de referencia |

## Referencias

- Baker, J. W. (2011). *Conditional Mean Spectrum: Tool for Ground-Motion Selection*. Journal of Structural Engineering, 137(3), 322–331.
- Bond et al. (2024). *An unsupervised machine learning approach for ground-motion spectra clustering and selection*. Earthquake Engineering & Structural Dynamics.
- Yu, B., & Barter, R. (2024). *Veridical Data Science*.
- Norma Técnica E.030 — Diseño Sismorresistente (RNE, Perú).
- Manual de Puentes — MTC.

## Integrantes — Grupo 5

| Nombre |
|---|
| Palacios Rojas, Alexander Ítalo |
| Cristobal Villanueva, Jhon Franklin |
| Segura Sanchez, José Gustavo |
| *(completar)* |

**Docente:** Kurt Walter Soncco Sinchi

## Licencia

Proyecto académico — UPC, curso *Aplicaciones de IA en Estructuras*. Uso educativo.
