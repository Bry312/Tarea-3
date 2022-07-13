# Tarea-3

---

```{r setup, include=FALSE}
library(flexdashboard)
```


```{r paquetes}

library(dplyr)
library(sf)
library(leaflet)
library(DT)
library(readr)
library(ggplot2)
library(plotly)
library(tidyverse)
library(terra)
library(sf)
library(stringi)
library(readxl)
```

```{r lectura-datos}
cantones <- 
  st_read(dsn = "C:/Users/Bryan/OneDrive/Desktop/Procesamiento de Datos/Tarea3/cantones_simplificados.geojson", quiet = TRUE) %>%
  st_transform(4326)

# Transformación de datos de cantones
estadisticas_policiales <-
 readxl::read_excel ("C:/Users/Bryan/OneDrive/Desktop/Procesamiento de Datos/Tarea3/estadisticaspoliciales2021.xls"
  )

```

### Mapa de cantones 

```{r}
# Lectura
cantones <-
  st_read(
    dsn = "C:/Users/Bryan/OneDrive/Desktop/Procesamiento de Datos/Tarea3/cantones_simplificados.geojson",
    quiet = TRUE
  ) %>%
  st_transform(4326) # transformación a WGS84

# Transformación
cantones <-
  cantones %>%
  st_transform(5367) %>%
  st_simplify(dTolerance = 100) %>% # simplificación de geometrías
  st_transform(4326)
# En el data frame de cantones
cantones <-
  cantones %>%
  mutate(canton_normalizado = tolower(stri_trans_general(canton, id = "Latin-ASCII")))

delitos <-
  read_xls(path = "C:/Users/Bryan/OneDrive/Desktop/Procesamiento de Datos/Tarea3/estadisticaspoliciales2021.xls")

# En el data frame de delitos
delitos <-
  delitos %>%
  mutate(canton_normalizado = tolower(stri_trans_general(Canton, id = "Latin-ASCII")))


delitos %>%
  left_join(
    dplyr::select(st_drop_geometry(cantones),
                  canton_normalizado, cod_canton),
    by = "canton_normalizado",
    copy = FALSE,
    keep = FALSE
  ) %>%
  filter(is.na(cod_canton) & canton_normalizado != "desconocido") %>% # los cod_canton = NA son los que no están en el data frame de cantones
  distinct(canton_normalizado)

# Corrección de nombres de cantones en delitos
delitos <-
  delitos %>%
  mutate(Canton = if_else(Canton == "LEON CORTES", "LEON CORTES CASTRO", Canton)) %>%
  mutate(Canton = if_else(Canton == "VASQUEZ DE CORONADO", "VAZQUEZ DE CORONADO", Canton))

# Se realiza nuevamente esta operación para reflejar los cambios en los nombres de cantones
delitos <-
  delitos %>%
  mutate(canton_normalizado = tolower(stri_trans_general(Canton, id = "Latin-ASCII")))

# Revisión
delitos %>%
  left_join(
    dplyr::select(st_drop_geometry(cantones),
                  canton_normalizado, cod_canton),
    by = "canton_normalizado",
    copy = FALSE,
    keep = FALSE
  ) %>%
  filter(is.na(cod_canton) & canton_normalizado != "desconocido") %>% # los cod_canton = NA son los que no están en el data frame de cantones
  distinct(canton_normalizado)

# Unión del código de cantón a delitos
delitos <-
  delitos %>%
  left_join(
    dplyr::select(
      st_drop_geometry(cantones),
      cod_canton,
      canton_normalizado
    ),
    by = "canton_normalizado",
    copy = FALSE,
    keep = FALSE
  )

# Conteo de registros por código de cantón
delitos_x_canton <-
  delitos %>%
  count(cod_canton, name = "delitos")

# Unión de cantidad de delitos por cantón a cantones
cantones_delitos <-
  cantones %>%
  left_join(
    delitos_x_canton,
    by = "cod_canton",
    copy = FALSE,
    keep = FALSE
  )

# Paleta de colores para los mapas
colores_cantones_delitos <-
  colorNumeric(palette = "Blues",
               domain = cantones_delitos$delitos,
               na.color = "transparent")

# Mapa leaflet de delitos en cantones
leaflet() %>%
  setView(# centro y nivel inicial de acercamiento
    lng = -84.19452,
    lat = 9.572735,
    zoom = 7) %>%
  addTiles(group = "OpenStreetMap") %>% # capa base
  addPolygons(
    # capa de polígonos
    data = cantones_delitos,
    fillColor = ~ colores_cantones_delitos(cantones_delitos$delitos),
    fillOpacity = 0.8,
    color = "black",
    stroke = TRUE,
    weight = 1.0,
    popup = paste(
      # ventana emergente
      paste(
        "<strong>Cantón:</strong>",
        cantones_delitos$canton
      ),
      paste(
        "<strong>Delitos:</strong>",
        cantones_delitos$delitos
      ),
      sep = '<br/>'
    ),
    group = "Delitos en cantones"
  ) %>%
  addLayersControl(
    # control de capas
    baseGroups = c("OpenStreetMap"),
    overlayGroups = c("Delitos en cantones")
  ) %>%
  addLegend(
    # leyenda
    position = "bottomleft",
    pal = colores_cantones_delitos,
    values = cantones_delitos$delitos,
    group = "Delitos",
    title = "Cantidad de delitos"
  )
```


### Tabla

```{r}

# Transformación de datos de cantones
estadisticas_policiales <-
 readxl::read_excel ("C:/Users/Bryan/OneDrive/Desktop/Procesamiento de Datos/Tarea3/estadisticaspoliciales2021.xls"
  )

# Transformación de datos 
estadisticas_policiales <-
  estadisticas_policiales %>%     
  select(Delito= Delito, Fecha, Victima, Edad, Genero = Genero, Provincia, Canton) %>%  
  mutate(Fecha = as.Date(Fecha, format = "%d/%m/%Y"))

# Visualización de datos en formato tabular
estadisticas_policiales %>%  
  datatable( 
    colnames = c("Delito", "Fecha", "Victima", "Edad", "Genero", "Provincia", "Canton"),
    options = list(
    pageLength = 5,
    language = list(url = '//cdn.datatables.net/plug-ins/1.10.11/i18n/Spanish.json')
  ))
```

### Grafico de Delitos

```{r}
ggplot2_barras_proporcion <-
  estadisticas_policiales %>%  
  ggplot(aes(x = Delito, y = stat(count), group = 100)) +
  geom_bar() +
  ggtitle("Tipos de Delitos") +
  xlab("Delito") +
  ylab("Cantidad") +
  theme_minimal()

ggplotly(ggplot2_barras_proporcion) %>% config(locale = 'es')
```

### Grafico por victima
```{r}

# ggplotly - Gráfico de barras simples con valores de conteo
ggplot2_barras_conteo <-
  cantones %>%
  ggplot(aes(x = "Victima", y = "Delito")) +
  geom_bar(stat = "summary", fun.y = "mean") +
  ggtitle("Delitos por victimas") +
  xlab("Delito") +
  ylab("Victima") +
  theme_minimal()

ggplotly(ggplot2_barras_conteo) %>% config(locale = 'es')

```


### Grafico por mes 
```{r}

ggplot2_histograma_estadisticas_policiales <-
  estadisticas_policiales %>%  
  ggplot(aes(x = Fecha)) +
  geom_histogram(binwidth = 80) + 
  ggtitle("Delitos por Mes") +
  xlab("Fecha") +
  ylab("Cantidad de Delitos") +
  theme_minimal()

ggplotly(ggplot2_histograma_estadisticas_policiales) %>% config(locale = 'es')  
```


### Grafico barras apiladas
```{r}

ggplot2_barras_apiladas_cantidad <-
  estadisticas_policiales %>%
  ggplot(aes(x = Delito, fill = Genero)) +
  geom_bar(position = "fill") +
  ggtitle("Proporcion de delito por genero") +
  xlab("Delito") +
  ylab("Cantidad") +
  labs(fill = "Delito") +
  theme_minimal()

ggplotly(ggplot2_barras_apiladas_cantidad) %>% config(locale = 'es')
```
