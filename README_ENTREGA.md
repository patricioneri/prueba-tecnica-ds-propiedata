# README_ENTREGA.md

## Cómo ejecutar el código

1. Crear y activar un entorno virtual:

```bash
uv venv
.venv\Scripts\activate
```

2. Instalar dependencias:

```bash
uv pip install -r requirements_entrega.txt
```

3. Ejecutar los notebooks en este orden:

```text
notebooks/01_limpieza.ipynb
notebooks/02_modelado.ipynb
```

El primer notebook genera el dataset limpio en:

```text
output/dataset_unificado.parquet
```

El segundo notebook carga ese dataset, entrena los modelos y muestra las métricas de validación.

## Decisiones tomadas

Para la limpieza se definió un schema común entre plataformas, priorizando variables útiles para estimar precios: superficie, ambientes, baños, habitaciones, tipo de propiedad, barrio y coordenadas. Se descartó la descripción porque requería procesamiento de texto y quedaba fuera del alcance inicial. También se eliminaron registros sin precio, ya que `price` es la variable objetivo del modelo. Para valores faltantes se usaron imputaciones simples y se agregaron flags de missing en algunas variables. En modelado se comparó una regresión lineal con Random Forest usando validación cruzada, priorizando métricas reproducibles como R², RMSE y MAE.