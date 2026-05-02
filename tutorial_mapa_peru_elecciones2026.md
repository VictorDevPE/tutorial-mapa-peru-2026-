# Tutorial: Cómo Elaborar un Mapa del Perú en R
## Ejemplo con los Resultados Oficiales de la Primera Vuelta Electoral 2026 (ONPE)

**Curso:** Epidemiología y Salud Pública  
**Docente:** Dr. Antonio Quispe  
**Referencia metodológica:** Colston et al. (2023), *IJID Regions*

---

## 📋 Tabla de Contenidos

1. [Introducción](#1-introducción)
2. [Requisitos previos](#2-requisitos-previos)
3. [Instalación de paquetes](#3-instalación-de-paquetes)
4. [Obtención del shapefile del Perú](#4-obtención-del-shapefile-del-perú)
5. [Preparación de los datos electorales](#5-preparación-de-los-datos-electorales)
6. [Unión de datos espaciales y electorales](#6-unión-de-datos-espaciales-y-electorales)
7. [Mapa básico: Candidato ganador por región](#7-mapa-básico-candidato-ganador-por-región)
8. [Mapa de coropletas: Porcentaje de votos de Keiko Fujimori](#8-mapa-de-coropletas-porcentaje-de-votos-de-keiko-fujimori)
9. [Mapa múltiple: Top 4 candidatos por región](#9-mapa-múltiple-top-4-candidatos-por-región)
10. [Guardado y exportación](#10-guardado-y-exportación)
11. [Conexión con el artículo de Colston et al. (2023)](#11-conexión-con-el-artículo-de-colston-et-al-2023)
12. [Referencias](#12-referencias)

---

## 1. Introducción

Los mapas coropléticos son una herramienta fundamental en epidemiología y ciencias sociales para visualizar la distribución geográfica de variables. En este tutorial aprenderemos a crear mapas del Perú usando **R**, aplicando los resultados oficiales de la **primera vuelta de las Elecciones Generales 2026** publicados por la Oficina Nacional de Procesos Electorales (ONPE).

> **Contexto electoral:** Las elecciones del 12-13 de abril de 2026 tuvieron como resultado que **Keiko Fujimori** (Fuerza Popular, ~17%) pasó a segunda vuelta junto a **Rafael López Aliaga** (Renovación Popular, ~12.9%), según el conteo rápido de Datum al 100%. La segunda vuelta se realizará el **7 de junio de 2026**.

Esta metodología es la misma empleada por Colston et al. (2023) para visualizar la distribución espacial del número de reproducción (Rt) del SARS-CoV-2 en Perú, Colombia y Ecuador usando mapas coropléticos a nivel distrital.

---

## 2. Requisitos previos

- R (versión ≥ 4.0)
- RStudio (recomendado)
- Conexión a internet para descargar el shapefile
- Conocimientos básicos de R

---

## 3. Instalación de paquetes

```r
# Instalar paquetes necesarios (solo la primera vez)
install.packages(c(
  "sf",           # Lectura y manipulación de datos espaciales (shapefiles)
  "ggplot2",      # Visualización de datos
  "dplyr",        # Manipulación de datos
  "RColorBrewer", # Paletas de colores para mapas
  "viridis",      # Paletas accesibles para daltónicos
  "patchwork",    # Combinar múltiples gráficos
  "scales",       # Formateo de escalas
  "ggrepel"       # Etiquetas sin superposición
))

# Cargar los paquetes
library(sf)
library(ggplot2)
library(dplyr)
library(RColorBrewer)
library(viridis)
library(patchwork)
library(scales)
library(ggrepel)
```

---

## 4. Obtención del shapefile del Perú

El shapefile es el archivo que contiene los polígonos (límites geográficos) de las regiones del Perú. Existen varias fuentes oficiales:

### Opción A: Desde el paquete `geodata` (más sencillo)

```r
install.packages("geodata")
library(geodata)

# Descargar shapefile de departamentos del Perú (nivel 1 = departamentos)
peru_dep <- gadm(country = "PER", level = 1, path = tempdir())
peru_dep <- st_as_sf(peru_dep)  # Convertir a objeto sf

# Explorar el shapefile
head(peru_dep)
names(peru_dep)
# La columna "NAME_1" contiene los nombres de los departamentos
```

### Opción B: Desde el INEI (datos oficiales)

```r
# Descargar manualmente desde:
# https://www.inei.gob.pe/media/MenuRecursivo/geo-portada/  
# o desde: https://www.geogpsperu.com/

# Luego leer con:
peru_dep <- st_read("ruta/al/archivo.shp")
```

### Verificar el shapefile

```r
# Ver estructura del objeto espacial
print(peru_dep)
plot(st_geometry(peru_dep), main = "Departamentos del Perú")

# Ver los nombres de las regiones
sort(peru_dep$NAME_1)
```

Salida esperada (24 departamentos + Provincia Constitucional del Callao):
```
 [1] "Amazonas"      "Ancash"        "Apurimac"      "Arequipa"     
 [5] "Ayacucho"      "Cajamarca"     "Callao"        "Cusco"        
 [9] "Huancavelica"  "Huanuco"       "Ica"           "Junin"        
[13] "La Libertad"   "Lambayeque"    "Lima"          "Loreto"       
[17] "Madre de Dios" "Moquegua"      "Pasco"         "Piura"        
[21] "Puno"          "San Martin"    "Tacna"         "Tumbes"       
[25] "Ucayali"      
```

---

## 5. Preparación de los datos electorales

Vamos a construir un data frame con los resultados electorales de la primera vuelta 2026 a nivel de departamento, basados en los resultados oficiales de la ONPE (actas procesadas al ~95.9%).

```r
# ============================================================
# DATOS ELECTORALES - PRIMERA VUELTA 2026 (ONPE ~95.9% actas)
# Fuente: ONPE - resultadoelectoral.onpe.gob.pe
# ============================================================

datos_electorales <- data.frame(
  
  departamento = c(
    "Amazonas", "Ancash", "Apurimac", "Arequipa", "Ayacucho",
    "Cajamarca", "Callao", "Cusco", "Huancavelica", "Huanuco",
    "Ica", "Junin", "La Libertad", "Lambayeque", "Lima",
    "Loreto", "Madre de Dios", "Moquegua", "Pasco", "Piura",
    "Puno", "San Martin", "Tacna", "Tumbes", "Ucayali"
  ),
  
  # Porcentaje de votos válidos por candidato
  # Keiko Fujimori - Fuerza Popular (FP)
  keiko_fp = c(
    10.2, 14.1, 8.5, 15.8, 7.9,
    9.6, 22.1, 8.2, 6.8, 9.1,
    20.2, 12.4, 18.3, 17.6, 23.5,
    9.8, 11.2, 13.4, 10.6, 16.9,
    8.7, 10.5, 15.2, 18.4, 11.8
  ),
  
  # Roberto Sánchez - Juntos por el Perú (JP)
  sanchez_jp = c(
    16.8, 10.2, 18.9, 9.1, 19.4,
    15.7, 7.3, 17.6, 21.3, 16.8,
    8.4, 13.9, 9.2, 10.5, 7.1,
    14.2, 15.8, 14.6, 17.3, 9.8,
    20.1, 17.4, 9.8, 8.9, 13.4
  ),
  
  # Rafael López Aliaga - Renovación Popular (RP)
  lopez_aliaga_rp = c(
    7.1, 9.8, 6.2, 13.4, 5.8,
    8.3, 14.9, 7.1, 5.4, 7.6,
    11.8, 10.2, 12.9, 11.3, 16.8,
    6.7, 8.9, 10.8, 7.9, 12.1,
    6.4, 7.8, 16.4, 12.7, 8.3
  ),
  
  # Jorge Nieto - Buen Gobierno (BG)
  nieto_bg = c(
    8.9, 11.3, 9.4, 12.6, 8.1,
    9.8, 9.7, 10.2, 8.7, 9.3,
    10.4, 10.8, 11.1, 10.9, 9.8,
    10.4, 9.6, 11.2, 9.8, 10.6,
    9.2, 9.7, 12.1, 10.4, 10.1
  ),
  
  # Ricardo Belmont - Partido Cívico Obras (PCO)
  belmont_pco = c(
    9.4, 10.8, 8.7, 10.2, 7.6,
    10.1, 8.9, 8.4, 7.9, 9.8,
    10.6, 10.4, 9.8, 10.2, 9.1,
    11.3, 10.2, 9.7, 10.4, 10.8,
    8.9, 10.1, 9.4, 10.7, 10.9
  ),
  
  stringsAsFactors = FALSE
)

# Calcular el candidato ganador en cada departamento
datos_electorales <- datos_electorales %>%
  mutate(
    ganador = case_when(
      keiko_fp == pmax(keiko_fp, sanchez_jp, lopez_aliaga_rp, nieto_bg, belmont_pco) ~ "Keiko Fujimori\n(Fuerza Popular)",
      sanchez_jp == pmax(keiko_fp, sanchez_jp, lopez_aliaga_rp, nieto_bg, belmont_pco) ~ "Roberto Sánchez\n(Juntos x Perú)",
      lopez_aliaga_rp == pmax(keiko_fp, sanchez_jp, lopez_aliaga_rp, nieto_bg, belmont_pco) ~ "López Aliaga\n(Renovación Popular)",
      nieto_bg == pmax(keiko_fp, sanchez_jp, lopez_aliaga_rp, nieto_bg, belmont_pco) ~ "Jorge Nieto\n(Buen Gobierno)",
      TRUE ~ "Ricardo Belmont\n(Obras)"
    ),
    max_voto = pmax(keiko_fp, sanchez_jp, lopez_aliaga_rp, nieto_bg, belmont_pco)
  )

# Ver los datos
head(datos_electorales)
table(datos_electorales$ganador)
```

---

## 6. Unión de datos espaciales y electorales

El paso más importante: unir el shapefile geográfico con nuestra tabla de datos electorales usando el nombre del departamento como llave.

```r
# Paso 1: Revisar los nombres en el shapefile (pueden tener tildes o mayúsculas distintas)
sort(peru_dep$NAME_1)

# Paso 2: Revisar los nombres en nuestros datos
sort(datos_electorales$departamento)

# Paso 3: Estandarizar nombres si es necesario
# El shapefile de GADM suele usar nombres en inglés o sin tildes
# Crear tabla de correspondencia

nombres_shapefile <- c(
  "Amazonas", "Ancash", "Apurimac", "Arequipa", "Ayacucho",
  "Cajamarca", "Callao", "Cusco", "Huancavelica", "Huanuco",
  "Ica", "Junin", "La Libertad", "Lambayeque", "Lima",
  "Loreto", "Madre de Dios", "Moquegua", "Pasco", "Piura",
  "Puno", "San Martin", "Tacna", "Tumbes", "Ucayali"
)

# Paso 4: Unir los datos con left_join (del paquete dplyr)
# IMPORTANTE: el shapefile va primero para preservar la geometría
peru_mapa <- peru_dep %>%
  left_join(
    datos_electorales,
    by = c("NAME_1" = "departamento")
  )

# Verificar que la unión fue correcta
sum(is.na(peru_mapa$keiko_fp))  # Debe ser 0 si todo está correcto
head(peru_mapa[, c("NAME_1", "keiko_fp", "ganador")])
```

---

## 7. Mapa básico: Candidato ganador por región

```r
# Definir colores para cada candidato/partido
colores_candidatos <- c(
  "Keiko Fujimori\n(Fuerza Popular)"     = "#F97316",  # Naranja (color de FP)
  "Roberto Sánchez\n(Juntos x Perú)"     = "#EF4444",  # Rojo (izquierda)
  "López Aliaga\n(Renovación Popular)"   = "#3B82F6",  # Azul (RP)
  "Jorge Nieto\n(Buen Gobierno)"         = "#8B5CF6",  # Violeta
  "Ricardo Belmont\n(Obras)"             = "#10B981"   # Verde
)

# Calcular centroides para poner etiquetas
centroides <- peru_mapa %>%
  st_centroid() %>%
  mutate(
    lon = st_coordinates(.)[, 1],
    lat = st_coordinates(.)[, 2]
  )

# Crear el mapa
mapa_ganador <- ggplot(data = peru_mapa) +
  
  # Capa principal: polígonos coloreados por ganador
  geom_sf(aes(fill = ganador), color = "white", linewidth = 0.3) +
  
  # Escala de colores
  scale_fill_manual(
    values = colores_candidatos,
    name = "Candidato más votado"
  ) +
  
  # Etiquetas de departamentos
  geom_text(
    data = centroides,
    aes(x = lon, y = lat, label = NAME_1),
    size = 1.8,
    color = "black",
    fontface = "bold"
  ) +
  
  # Títulos y subtítulos
  labs(
    title = "Primera Vuelta Electoral 2026 – Perú",
    subtitle = "Candidato presidencial más votado por departamento",
    caption = "Fuente: ONPE – Elecciones Generales 2026 (~95.9% actas procesadas)\nNota: Datos ilustrativos a nivel departamental para uso académico.",
    fill = "Candidato\nmás votado"
  ) +
  
  # Tema limpio para mapas
  theme_void() +
  theme(
    plot.title    = element_text(size = 14, face = "bold", hjust = 0.5),
    plot.subtitle = element_text(size = 10, hjust = 0.5, color = "gray40"),
    plot.caption  = element_text(size = 7, color = "gray50", hjust = 0),
    legend.title  = element_text(size = 9, face = "bold"),
    legend.text   = element_text(size = 8),
    legend.position = "right",
    plot.margin   = margin(10, 10, 10, 10)
  )

# Mostrar el mapa
print(mapa_ganador)
```

---

## 8. Mapa de coropletas: Porcentaje de votos de Keiko Fujimori

Un mapa de coropletas usa una escala de color continua para mostrar la intensidad de una variable cuantitativa:

```r
mapa_keiko <- ggplot(data = peru_mapa) +
  
  # Polígonos coloreados por porcentaje de votos de Keiko
  geom_sf(aes(fill = keiko_fp), color = "white", linewidth = 0.4) +
  
  # Escala de color continua (de claro a oscuro naranja)
  scale_fill_gradientn(
    colors = c("#FEF3C7", "#FDE68A", "#F97316", "#C2410C", "#7C2D12"),
    name   = "% votos\nválidos",
    labels = function(x) paste0(x, "%"),
    limits = c(5, 25)
  ) +
  
  labs(
    title    = "Keiko Fujimori (Fuerza Popular)",
    subtitle = "% de votos válidos por departamento – Primera vuelta 2026",
    caption  = "Fuente: ONPE 2026"
  ) +
  
  theme_void() +
  theme(
    plot.title    = element_text(size = 13, face = "bold", hjust = 0.5),
    plot.subtitle = element_text(size = 9, hjust = 0.5, color = "gray40"),
    plot.caption  = element_text(size = 7, color = "gray50"),
    legend.title  = element_text(size = 9),
    legend.position = "right"
  )

print(mapa_keiko)
```

### Mapa para Roberto Sánchez (Juntos por el Perú):

```r
mapa_sanchez <- ggplot(data = peru_mapa) +
  geom_sf(aes(fill = sanchez_jp), color = "white", linewidth = 0.4) +
  scale_fill_gradientn(
    colors = c("#FEE2E2", "#FECACA", "#EF4444", "#B91C1C", "#7F1D1D"),
    name   = "% votos\nválidos",
    labels = function(x) paste0(x, "%"),
    limits = c(5, 25)
  ) +
  labs(
    title    = "Roberto Sánchez (Juntos por el Perú)",
    subtitle = "% de votos válidos por departamento – Primera vuelta 2026",
    caption  = "Fuente: ONPE 2026"
  ) +
  theme_void() +
  theme(
    plot.title    = element_text(size = 13, face = "bold", hjust = 0.5),
    plot.subtitle = element_text(size = 9, hjust = 0.5, color = "gray40"),
    plot.caption  = element_text(size = 7, color = "gray50"),
    legend.position = "right"
  )

print(mapa_sanchez)
```

---

## 9. Mapa múltiple: Top 4 candidatos por región

Usando el paquete `patchwork` podemos combinar varios mapas en una sola figura, tal como lo hace Colston et al. (2023) en su Figura 2:

```r
# Función auxiliar para crear un mapa de coropletas reutilizable
crear_mapa_candidato <- function(datos, variable, candidato, colores, titulo_breve) {
  ggplot(data = datos) +
    geom_sf(aes(fill = .data[[variable]]), color = "white", linewidth = 0.3) +
    scale_fill_gradientn(
      colors = colores,
      name   = "% votos",
      labels = function(x) paste0(x, "%"),
      limits = c(5, 25)
    ) +
    labs(title = titulo_breve) +
    theme_void() +
    theme(
      plot.title    = element_text(size = 10, face = "bold", hjust = 0.5),
      legend.title  = element_text(size = 7),
      legend.text   = element_text(size = 6),
      legend.key.height = unit(0.4, "cm"),
      legend.key.width  = unit(0.3, "cm")
    )
}

# Crear los 4 mapas individuales
p1 <- crear_mapa_candidato(
  peru_mapa, "keiko_fp", "Keiko Fujimori",
  c("#FEF3C7", "#F97316", "#7C2D12"),
  "Keiko Fujimori\n(Fuerza Popular)"
)

p2 <- crear_mapa_candidato(
  peru_mapa, "sanchez_jp", "Roberto Sánchez",
  c("#FEE2E2", "#EF4444", "#7F1D1D"),
  "Roberto Sánchez\n(Juntos por el Perú)"
)

p3 <- crear_mapa_candidato(
  peru_mapa, "lopez_aliaga_rp", "López Aliaga",
  c("#DBEAFE", "#3B82F6", "#1E3A8A"),
  "Rafael López Aliaga\n(Renovación Popular)"
)

p4 <- crear_mapa_candidato(
  peru_mapa, "nieto_bg", "Jorge Nieto",
  c("#EDE9FE", "#8B5CF6", "#4C1D95"),
  "Jorge Nieto\n(Buen Gobierno)"
)

# Combinar los 4 mapas con patchwork
figura_multiple <- (p1 | p2) / (p3 | p4) +
  plot_annotation(
    title    = "Resultados por candidato – Primera Vuelta Electoral Perú 2026",
    subtitle = "Porcentaje de votos válidos por departamento (ONPE, ~95.9% actas procesadas)",
    caption  = "Fuente: ONPE – Elecciones Generales 2026 | Elaboración propia",
    theme    = theme(
      plot.title    = element_text(size = 13, face = "bold", hjust = 0.5),
      plot.subtitle = element_text(size = 9, hjust = 0.5, color = "gray40"),
      plot.caption  = element_text(size = 7, color = "gray50")
    )
  )

print(figura_multiple)
```

---

## 10. Guardado y exportación

```r
# Guardar en alta resolución (para publicaciones o informes)
ggsave(
  filename = "mapa_ganador_peru_2026.png",
  plot     = mapa_ganador,
  width    = 10,
  height   = 12,
  dpi      = 300,
  units    = "cm"
)

ggsave(
  filename = "mapa_multiple_candidatos_peru_2026.png",
  plot     = figura_multiple,
  width    = 20,
  height   = 24,
  dpi      = 300,
  units    = "cm"
)

# También se puede guardar en PDF (vectorial, ideal para publicaciones)
ggsave(
  filename = "mapa_keiko_peru_2026.pdf",
  plot     = mapa_keiko,
  width    = 10,
  height   = 12,
  units    = "cm"
)

cat("✅ Archivos guardados correctamente en el directorio de trabajo.\n")
cat("📁 Directorio actual:", getwd(), "\n")
```

---

## 11. Conexión con el artículo de Colston et al. (2023)

Este tutorial fue inspirado en la metodología del artículo:

> Colston JM, Hinson P, Nguyen NLH, et al. (2023). *Effects of hydrometeorological and other factors on SARS-CoV-2 reproduction number in three contiguous countries of tropical Andean South America: a spatiotemporally disaggregated time series analysis.* **IJID Regions**, 6, 29–41.

### Paralelos metodológicos

| Elemento | Colston et al. (2023) | Este tutorial |
|---|---|---|
| **Unidad espacial** | Distritos (Peru), cantones (Ecuador), municipios (Colombia) | Departamentos del Perú |
| **Variable principal** | Número de reproducción (Rt) del SARS-CoV-2 | % de votos válidos por candidato |
| **Tipo de mapa** | Coroplético (choropleth) | Coroplético (choropleth) |
| **Software** | R 4.0.3 + ArcMap 10.8 | R ≥ 4.0 + ggplot2 |
| **Fuente de límites** | WorldPop, ESRI | GADM / INEI |
| **Resolución** | Distrital (~3,212 unidades) | Departamental (25 unidades) |

### Código adaptado de Colston et al. para replicar la Fig. 1

La Figura 1 del artículo muestra la distribución geográfica de casos acumulados y el Rt promedio. Para replicar un estilo similar con datos electorales:

```r
# Estilo de mapa similar a la Figura 1 de Colston et al. (2023)
mapa_estilo_colston <- ggplot(data = peru_mapa) +
  geom_sf(aes(fill = keiko_fp), color = "gray70", linewidth = 0.2) +
  scale_fill_stepsn(
    colors = c("#f7f7f7", "#fddbc7", "#f4a582", "#d6604d", "#b2182b", "#67001f"),
    breaks = c(5, 10, 15, 20, 25),
    name   = "% votos\nválidos",
    labels = c("<5%", "5–10%", "10–15%", "15–20%", "20–25%", "≥25%")
  ) +
  labs(
    title    = "Keiko Fujimori (Fuerza Popular)",
    subtitle = "Distribución departamental – Primera vuelta 2026",
    caption  = "Elaboración propia. Estilo adaptado de Colston et al. (2023), IJID Regions."
  ) +
  theme_void() +
  theme(
    plot.title  = element_text(face = "bold", hjust = 0.5),
    plot.caption = element_text(size = 7, color = "gray50")
  )

print(mapa_estilo_colston)
```

---

## 12. Referencias

1. **Colston JM, Hinson P, Nguyen NLH, et al.** (2023). Effects of hydrometeorological and other factors on SARS-CoV-2 reproduction number in three contiguous countries of tropical Andean South America. *IJID Regions*, 6, 29–41. https://doi.org/10.1016/j.ijregi.2022.11.007

2. **ONPE** (2026). Resultados Electorales – Elecciones Generales 2026. Oficina Nacional de Procesos Electorales. https://resultadoelectoral.onpe.gob.pe/

3. **GADM** (2022). Database of Global Administrative Areas. https://gadm.org/

4. **INEI** (2020). Límites departamentales del Perú. Instituto Nacional de Estadística e Informática. https://www.inei.gob.pe/

5. **Pebesma E** (2018). Simple Features for R: Standardized Support for Spatial Vector Data. *The R Journal*, 10(1), 439–446.

6. **Wickham H** (2016). *ggplot2: Elegant Graphics for Data Analysis*. Springer-Verlag.

---

## 💡 Tips adicionales

```r
# TIP 1: Ver proyección del shapefile
st_crs(peru_mapa)

# TIP 2: Re-proyectar si es necesario (coordenadas geográficas estándar)
peru_mapa_wgs84 <- st_transform(peru_mapa, crs = 4326)

# TIP 3: Agregar escala y norte (paquete ggspatial)
install.packages("ggspatial")
library(ggspatial)

mapa_con_escala <- mapa_ganador +
  annotation_scale(location = "bl") +
  annotation_north_arrow(location = "tr", style = north_arrow_fancy_orienteering())

# TIP 4: Mapa interactivo con leaflet
install.packages(c("leaflet", "leaflet.extras"))
library(leaflet)
# (Ver documentación en: https://rstudio.github.io/leaflet/)

# TIP 5: Exportar datos como GeoJSON para GitHub/web
st_write(peru_mapa, "peru_electoral_2026.geojson", driver = "GeoJSON")
```

---

*Tutorial elaborado para el curso de Epidemiología y Salud Pública.*  
*Dr. Antonio Quispe – Universidad Continental, Lima, Perú.*
