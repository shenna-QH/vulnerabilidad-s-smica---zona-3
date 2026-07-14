# Pipeline de filtrado y escalamiento espectral — CISMID

Configuración lista para filtrar el catálogo de registros del CISMID
según criterios sismológicos/geográficos y calcular el escalamiento
espectral respecto a un espectro objetivo (E.030 o CMS).

## Estructura

```
config/filters.yaml        # criterios de filtrado (edítalo aquí, sin tocar código)
src/filter_records.py      # filtra data/raw/*.csv -> data/processed/*.csv
src/spectral_scaling.py    # calcula factor de escala, RMSE y error % espectral
.github/workflows/pipeline.yml   # corre el filtro automáticamente en cada push
data/raw/                  # metadatos crudos del CISMID (CSV)
data/processed/            # salida filtrada/escalada
```

## Uso local

```bash
pip install pandas numpy pyyaml

# 1. Filtrar registros según config/filters.yaml
python src/filter_records.py \
  --input data/raw/cismid_metadata.csv \
  --config config/filters.yaml

# 2. Escalamiento espectral de un registro respecto a un objetivo
python src/spectral_scaling.py \
  --record data/processed/espectro_registro.csv \
  --target data/processed/espectro_objetivo.csv \
  --t_min 0.1 --t_max 2.0
```

## Editar los criterios de filtrado

Todo se controla desde `config/filters.yaml`. No es necesario tocar
`filter_records.py` para cambiar rangos de magnitud, profundidad,
departamentos incluidos, etc.

## Automatización en GitHub

El workflow `.github/workflows/pipeline.yml` corre automáticamente
cada vez que se hace push a `data/raw/`, `config/` o `src/` en la
rama `main`. El CSV filtrado queda disponible como *artifact*
descargable en la pestaña **Actions** del repositorio, y también se
puede disparar manualmente con el botón "Run workflow".

## Subir esto a tu repositorio

```bash
git checkout -b feature/pipeline-filtrado
git add config/ src/ .github/ data/raw/cismid_metadata.csv README.md
git commit -m "feat: pipeline de filtrado y escalamiento espectral CISMID"
git push origin feature/pipeline-filtrado
```

Luego abre un Pull Request hacia `main` para que el equipo lo revise.
