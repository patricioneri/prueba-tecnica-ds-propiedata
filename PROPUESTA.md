# PROPUESTA.md — Plan técnico de mejora para Propiedata

## 4.1 Matching de inmuebles únicos

### Problema

Al ser un producto basado en scraping de distintos portales, es esperable publicaciones duplicadas. Esto puede generar sesgos, inflar ciertas zonas del dataset e la correcta evalución del modelo. 

### Propuesta

Construir un dataset de candidatos usando variables como `neighborhood`, `lat`, `lon`, `total_area_m2`, `covered_area_m2`, `rooms`, `bedrooms`, `bathrooms` y `property_type`. Para evitar comparar todos contra todos y ahorrar tiempo y poder de computo, acotaria por barrio, tipo de propiedad y cercanía geográfica. 

Luego calcularía un score de similitud entre pares de publicaciones. Algunas features útiles serían:

- distancia geográfica entre coordenadas;
- diferencia relativa de superficie;
- diferencia de ambientes, dormitorios y baños;
- similitud textual entre títulos o descripciones;
- diferencia de precio;
- proximidad temporal de publicación.

Como primera versión, definiría reglas simples: por ejemplo, considerar potencial duplicado si las propiedades están a menos de cierta distancia, tienen el mismo tipo, superficies similares y cantidad de ambientes compatible. En una segunda etapa, si se cuenta con datos etiquetados, entrenaría un clasificador binario para predecir si dos publicaciones corresponden al mismo inmueble.

### Métricas

Setearia un umbral a partir del cual dos publicaciones diferentes corresponden a la misma vivienda.

### Esfuerzo estimado

Una primera versión basada en reglas podría implementarse en 1 semana. Una versión supervisada con etiquetado y evaluación robusta requeriría entre 3 y 5 semanas.

### Riesgos

El principal riesgo es fusionar incorrectamente propiedades similares dentro de un mismo edificio o zona. También puede haber problemas por coordenadas imprecisas, datos faltantes o diferencias deliberadas entre publicaciones de la misma unidad.


## 4.2 Arquitectura de tablas del pipeline

### Propuesta

Propondría una arquitectura por capas:

1. **Raw scraping layer**  
   Tabla con el output original de cada scraping, sin modificaciones. Con variables que permitan trazabilidad como `source`, `scrape_run_id`, timestamp de extracción, URL, identificador original de la publicación y payload original.

2. **Staging / normalized layer**  
   Tabla con nombres de columnas unificados, tipos normalizados y limpieza básica. Acá se parsean precios, superficies, coordenadas, fechas y categorías. Todavía no se eliminan duplicados agresivamente.

3. **Entity resolution layer**  
   Tabla donde se agrupan publicaciones que probablemente correspondan al mismo inmueble físico. 

4. **Feature layer**  
   Tabla con variables enriquecidas para modelado: features geoespaciales, precio por m² de la zona, densidad de oferta cercana, antigüedad normalizada, features temporales y agregados por barrio o radio.

5. **Training dataset layer**  
   Dataset versionado utilizado para entrenar modelos. Debe guardar fecha de generación, versión del código, filtros aplicados, features incluidas y target.

6. **Serving / UI layer**  
   Tabla o vista optimizada para la aplicación, con predicciones, intervalos de confianza, metadata del modelo y explicaciones relevantes para el usuario.

### Versionado

Versionaría datasets por fecha de corte, versión de código y versión de features. Idealmente, cada modelo entrenado debería poder reconstruirse exactamente a partir de una versión específica del dataset.

### Esfuerzo estimado

Una versión mínima viable podría implementarse en 2 a 3 semanas. Una arquitectura más robusta con versionado completo y tests automáticos podría requerir 6 semanas o más.

### Riesgos

El principal riesgo es sobrediseñar la arquitectura antes de estabilizar el flujo de scraping. La prioridad inicial debería ser trazabilidad y reproducibilidad, no complejidad innecesaria.


## 4.3 Modelado avanzado para alcanzar R² ≥ 0.8

### Propuesta

Priorizaría el trabajo en tres frentes: feature engineering, modelos no lineales y segmentación.

Primero, incorporaría features geoespaciales más ricas, para enriquecer el dataset. Algunas podrían ser:

- distancia a puntos de interés (estaciones de subte, parques, etc.)
- precio promedio por m² en un radio cercano 
- cantidad de propiedades similares en la zona
- clusters geográficos a partir de latitud y longitud
- variables por barrio normalizado y zona ampliada.

Segundo, probaría modelos de boosting como XGBoost, LightGBM que son los que mejor desempeño considero que presentan para datos tabulares. Estos modelos suelen funcionar muy bien con datos tabulares heterogéneos y capturan interacciones no lineales entre superficie, ubicación, tipo de propiedad y características del inmueble. Adicionalmente, estos modelos son bastante eficientes a la hora de modelar con outliers.

Finalmente, relizaria experimentos con diferentes modelos, ajustando hiperparámetros, recortando outliers, haciendo transformaciones para reducir dispersión de variables, y pruebas que puedan surgir para comprobar de manera empírica el mejor approach para el modelado final.

## Esfuerzo estimado

Una primera ronda de experimentación avanzada podría realizarse en 2 semanas. Una versión más sólida con validación temporal y espacial requeriría entre 4 y 6 semanas.


### Riesgos

El principal riesgo es obtener un R² alto por leakage, especialmente en la generación de nuevas variables a partir de otras. Habría que tener en cuenta la generación de estas variables en su split correspondiente (train/test/validation)


## 4.4 MLOps / CI-CD

### Diagnóstico

Para escalar el modelo de alquileres no alcanza con mejorar métricas en notebooks. Se necesita trazabilidad de experimentos, control de versiones de datasets, validación automática y un mecanismo claro para promover modelos a producción. Sin esto, es difícil saber qué cambió, por qué un modelo mejoró o empeoró, y si el modelo sigue siendo confiable después de nuevas ingestas de scraping.

### Propuesta

Como stack mínimo viable implementaría:

- **MLflow** para tracking de experimentos, parámetros, métricas, artefactos y modelos.
- **GitHub Actions** para correr tests básicos y validaciones del pipeline.
- **Airflow** para orquestar scraping, limpieza, feature engineering, entrenamiento y batch prediction.
- **Model registry** para registrar modelos candidatos, staging y producción.
- **Monitoreo de drift** sobre features, target y performance estimada.

Cada entrenamiento debería guardar:

- versión del dataset;
- commit del código;
- parámetros del modelo;
- métricas de validación;
- artefactos del modelo;
- importancia de variables;
- fecha de entrenamiento.

Para promoción de modelos, definiría reglas simples: un modelo solo puede pasar a producción si mejora R²/RMSE respecto al modelo actual, no degrada segmentos clave y supera validaciones de datos. También agregaría monitoreo post-deploy: distribución de precios, superficies, barrios, tasa de missing values y drift de predicciones.

### Stack ideal

En una versión más madura, incorporaría un feature store, validación temporal automatizada, dashboards de monitoreo, alertas ante drift y despliegue automatizado de modelos vía API. También agregaría tests de inferencia para asegurar que el modelo responde correctamente ante casos típicos y extremos.

### Esfuerzo estimado

Un MVP de MLOps podría implementarse en 2 semanas. Una solución más completa e integrada al producto podría requerir 6 semanas o más.

### Riesgos

El riesgo principal es implementar tooling antes de estabilizar métricas y datos. Por eso empezaría con un stack mínimo, priorizando trazabilidad, reproducibilidad y monitoreo básico.


## Priorización general para 1 mes

Si tuviera un mes, priorizaría los frentes en este orden:

1. **Arquitectura de tablas y calidad de datos**  
   Antes de mejorar modelos, es necesario asegurar trazabilidad, limpieza y reproducibilidad. Sin una base de datos confiable, cualquier mejora de performance puede ser frágil.

2. **Modelado avanzado y feature engineering geoespacial**  
   Es el frente con mayor impacto directo sobre el objetivo de R² ≥ 0.8. Priorizaría features espaciales, target log-transformado y modelos de boosting.

3. **Matching de inmuebles únicos**  
   Una vez ordenado el pipeline, atacaría duplicados para evitar sesgos en entrenamiento y evaluación. Empezaría con reglas simples y luego avanzaría hacia modelos supervisados si hay datos etiquetados.

4. **MLOps / CI-CD**  
   Implementaría un MVP desde el inicio con MLflow y validaciones básicas, pero dejaría el stack ideal para una segunda etapa. La prioridad inicial sería no perder trazabilidad de experimentos mientras se mejora el modelo.

En resumen, atacaría primero la confiabilidad del dato, luego la capacidad predictiva, después la deduplicación avanzada y finalmente la automatización completa del ciclo de vida del modelo.