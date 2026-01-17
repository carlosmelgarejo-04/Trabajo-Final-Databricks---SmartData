# 🎧 Proyecto Final – Spotify Lakehouse Analytics

<!-- TOC -->
- [📌 Resumen Ejecutivo](#-resumen-ejecutivo)
- [🗂 Fuentes de Datos](#-fuentes-de-datos)
- [🏗 Arquitectura General](#-arquitectura-general)
- [🥉 Capa Bronze (Raw Data)](#-capa-bronze-raw-data)
- [🥈 Capa Silver (Curated / Normalized)](#-capa-silver-curated--normalized)
- [🥇 Capa Gold (Analytics / BI)](#-capa-gold-analytics--bi)
- [🔄 Orquestación del Pipeline](#-orquestación-del-pipeline)
- [📊 Dashboards en Power BI](#-dashboards-en-power-bi)
- [🔐 Gobierno de Datos](#-gobierno-de-datos)
- [🚀 Resultados](#-resultados)
- [📌 Posibles Mejoras Futuras](#-posibles-mejoras-futuras)
- [👤 Autor](#-autor)
- [📎 Referencias](#-referencias)
<!-- /TOC -->

## 📌 Resumen Ejecutivo

Este proyecto implementa una arquitectura **Lakehouse con patrón Medallion (Bronze → Silver → Gold)** sobre **Databricks + Unity Catalog**, integrando datos musicales de Spotify para construir productos analíticos listos para **Business Intelligence en Power BI**.

El resultado es un flujo completo de **ingestión, transformación, gobierno de datos, modelado analítico y visualización**, alineado a prácticas reales de entornos empresariales.

---

## 🗂 Fuentes de Datos

### Dataset Principal

Los datos provienen de Kaggle:

> **Spotify Songs: Audio Features, Lyrics & Genres**  
> https://www.kaggle.com/datasets/serkantysz/550k-spotify-songs-audio-lyrics-and-genres

### Contenido del Dataset

#### Songs

Incluye información a nivel de canción:

- Identificadores: `id`, `name`, `album_name`, `year`
- Artistas: `artists`, `artist_ids`
- Géneros: `genre`, `niche_genres`
- Popularidad: `popularity`
- Métricas de audio: `danceability`, `energy`, `valence`, `tempo`, `loudness`, etc.
- Letras: `lyrics`
- Métricas agregadas de artistas: `total_artist_followers`, `avg_artist_popularity`

#### Artists

Incluye información a nivel de artista:

- `id`
- `name`
- `followers`
- `popularity`
- `genres`
- `main_genre`

---

## 🏗 Arquitectura General

La solución sigue el patrón **Medallion Architecture**, separando claramente responsabilidades por capas:

```
Azure Data Lake / Storage
        │
        ▼
Bronze (Raw)
        │
        ▼
Silver (Curated / Normalized)
        │
        ▼
Gold (Analytics / BI)
        │
        ▼
Power BI Dashboards
```

### Tecnologías Utilizadas

- **Databricks** (PySpark + Delta Lake)
- **Unity Catalog** (Gobierno y organización de datos)
- **Azure Data Lake Storage Gen2**
- **Power BI** (Visualización y análisis)
- **Databricks Jobs** (Orquestación)

---

## 🥉 Capa Bronze (Raw Data)

### Objetivo

Preservar los datos **tal como llegan desde la fuente**, sin transformaciones, garantizando trazabilidad y capacidad de reprocesamiento.

### Tablas

- `bronze.songs`
- `bronze.artists`

### Características

- Datos almacenados en formato **Delta**
- Esquema flexible
- Columnas complejas (listas) almacenadas como `string`

### Proceso

- Ingesta mediante notebooks Databricks
- Parametrización por:
  - `storageName`
  - `container`
  - `catalog`
  - `schema`

---

## 🥈 Capa Silver (Curated / Normalized)

### Objetivo

Estandarizar, limpiar y normalizar los datos para que sean **consistentes, confiables y reutilizables** por múltiples productos analíticos.

### Diseño

Se separaron entidades y relaciones en un modelo relacional tipo estrella.

### Tablas Silver Implementadas

### 1️⃣ `silver.track`


Campos clave:

- `track_id`
- `track_name`
- `album_name`
- `release_year`
- `genre_main`
- `track_popularity`
- `duration_ms`

Reglas:

- Normalización de texto
- Validación de rangos (popularidad, duración, año)
- Eliminación de duplicados

---

### 2️⃣ `silver.track_audio_features`



Incluye métricas sonoras:

- `danceability`
- `energy`
- `valence`
- `tempo`
- `loudness`
- `acousticness`
- `instrumentalness`

Reglas:

- Conversión de tipos
- Normalización de valores entre 0 y 1
- Control de outliers

---

### 3️⃣ `silver.artist`


Campos:

- `artist_id`
- `artist_name`
- `followers`
- `artist_popularity`
- `main_genre`
- `genres_arr`

Reglas:

- Limpieza de texto
- Conversión de métricas numéricas
- Eliminación de duplicados

---

### 4️⃣ `silver.bridge_track_artist`



Campos:

- `track_id`
- `artist_id`

Función:

- Relaciona canciones con uno o múltiples artistas
- Permite análisis cruzado por artista

---

### Gobernanza y Calidad

- Estandarización de esquemas
- Llaves primarias lógicas
- Separación de dominios (tracks / artists / relaciones)
- Preparación para control de accesos con Unity Catalog

---

## 🥇 Capa Gold (Analytics / BI)

### Objetivo

Entregar **productos de datos listos para consumo**, optimizados para Power BI y análisis avanzado.

### Tablas Gold Implementadas

### 1️⃣ `gold.fact_track_enriched`



Incluye:

- Atributos de la canción
- Métricas de audio
- Agregados de artistas:
  - `artist_count`
  - `total_followers`
  - `avg_artist_popularity`
  - `artists_concat`

**Uso:** Base principal para análisis generales y dashboards ejecutivos.

---

### 2️⃣ `gold.artist_impact`



Métricas:

- `tracks_count`
- `avg_track_popularity`
- `max_track_popularity`
- `avg_energy`
- `avg_danceability`
- `avg_valence`
- `avg_tempo`

**Uso:** Ranking y análisis de impacto de artistas.

---

### 3️⃣ `gold.genre_artist_summary`



Métricas:

- `tracks_count`
- `avg_track_popularity`
- `avg_energy`
- `avg_danceability`
- `avg_valence`

**Uso:** Identificación de dominancia de artistas por género.

---

## 🔄 Orquestación del Pipeline

Se implementó un **Databricks Workflow (Jobs API)** con dependencias entre tareas:

1. **Ingests_artists** → Carga datos raw de artistas
2. **Ingests_songs** → Carga datos raw de canciones
3. **Transform** → Construcción de todas las tablas Silver
4. **Load** → Construcción de tablas Gold

### Características

- Ejecución en clúster existente
- Parámetros dinámicos
- Dependencias explícitas
- Programación diaria vía Quartz Cron

---

## 📊 Dashboards en Power BI

### 1️⃣ Music Catalog Overview

**Fuente:** `gold.fact_track_enriched`

Visuales:

- KPIs: Total de canciones, popularidad promedio, duración promedio
- Popularidad por género
- Mapa sonoro (Energy vs Valence)
- Tabla de Top Canciones

---

### 2️⃣ Artist Impact & Ranking

**Fuente:** `gold.artist_impact`

Visuales:

- Ranking Top Artistas
- Impacto vs Followers (Scatter)
- Firma sonora del artista (Radar)
---

## 🔐 Gobierno de Datos

El proyecto utiliza **Unity Catalog** para:

- Organización por catálogos y esquemas
- Control de accesos por capa (Bronze / Silver / Gold)
- Preparación para auditoría y trazabilidad

---

## 🚀 Resultados

Este proyecto demuestra:

- Implementación real de arquitectura Lakehouse
- Separación de capas con responsabilidad clara
- Modelado analítico para BI
- Automatización con orquestación
- Buenas prácticas de gobierno de datos

---

## 📌 Posibles Mejoras Futuras

- Implementar SCD Tipo 2 en dimensiones
- Integrar análisis de sentimiento en letras
- Construir motor de recomendación con similitud de audio
- Exponer tablas Gold vía API

---

## 👤 Autor

Proyecto desarrollado como implementación académica/práctica de ingeniería de datos y analítica con enfoque en arquitecturas empresariales modernas.

---

## 📎 Referencias

- Kaggle Spotify Dataset
- Databricks Lakehouse & Unity Catalog Documentation
- Power BI Data Modeling Best Practices
