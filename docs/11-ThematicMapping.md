# Тематические карты {#mapping}



## Предварительные условия  {#mapping_prerequisites}

Для выполнения кода данной лекции вам понадобятся следующие пакеты:

```r
library(sf)
library(stars)
library(tmap)
library(readxl)
library(raster)
library(mapview)
library(classInt)
library(gapminder)
library(tidyverse)
library(googlesheets4)
library(rnaturalearth)
```

## Введение {#thematic_mapping_intro}

Тематические карты представляют собой важный инструмент географических исследований. Таблицы и графики не дают полного представления о пространственном распределении изучаемого явления. Это знание способна дать исследователю карта.

Разнообразие типов и видов карт достаточно велико. Комплексные картографические произведения, содержащие многослойный набор объектов, создаются, как правило, средствами геоинформационных пакетов. Такие карты требуют тщательной и кропотливой работы с легендой, устранения графических конфликтов между знаками, многократного редактирования входных данных, условий, фильтров и способов изображения в попытке достичь эстетичного и вместе с тем информативного результата.

В то же время, гораздо большее количество создаваемых в повседневной практике карт носят простой аналитический характер. Такие карты показывают одно, максимум два явления, и могут иллюстрировать входные данные, результаты промежуточных или итоговых расчетов. Создание именно таких карт целесообразно автоматизировать средствами программирования. В этом разделе мы познакомимся с созданием тематических карт средствами пакета [__tmap__](https://cran.r-project.org/web/packages/tmap/index.html). В качестве источника открытых данных мы будем использовать [Natural Earth](https://www.naturalearthdata.com/) и [WorldClim](http://www.worldclim.org/).

### Данные Natural Earth {#thematic_mapping_intro_ne}

[Natural Earth](https://www.naturalearthdata.com/) — это открытые мелкомасштабные картографические данные высокого качества. Данные доступны для трех масштабов: 1:10М, 1:50М и 1:110М. Для доступа к этим данным из среды R без загрузки исходных файлов можно использовать пакет [__rnaturalearth__](https://cran.r-project.org/web/packages/rnaturalearth/index.html). Пакет позволяет выгружать данные из внешнего репозитория, а также содержит три предзакачанных слоя:

- `ne_countries()` границы стран
- `ne_states()` границы единиц АТД 1 порядка
- `ne_coastline()` береговая линия

Для загрузки других слоев необходимо использовать функцию `ne_download()`, передав ей масштаб, название слоя и его категорию: 


```r
countries = ne_countries() %>% st_as_sf()

coast = ne_coastline() %>% st_as_sf()

# ocean = ne_download(scale = 110, 
#                     type = 'ocean', 
#                     category = 'physical', 
#                     returnclass = 'sf')
# 
# cities = ne_download(scale = 110, 
#                      type = 'populated_places', 
#                      category = 'cultural', 
#                      returnclass = 'sf')
# 
# rivers = ne_download(scale = 110, 
#                      type = 'rivers_lake_centerlines', 
#                      category = 'physical', 
#                      returnclass = 'sf')

ne = '/Volumes/Data/Spatial/Natural Earth/natural_earth_vector.gpkg'
ocean = st_read(ne, 'ne_110m_ocean')
## Reading layer `ne_110m_ocean' from data source 
##   `/Volumes/Data/Spatial/Natural Earth/natural_earth_vector.gpkg' 
##   using driver `GPKG'
## Simple feature collection with 2 features and 3 fields
## Geometry type: POLYGON
## Dimension:     XY
## Bounding box:  xmin: -180 ymin: -85.60904 xmax: 180 ymax: 90
## Geodetic CRS:  WGS 84
cities = st_read(ne, 'ne_110m_populated_places')
## Reading layer `ne_110m_populated_places' from data source 
##   `/Volumes/Data/Spatial/Natural Earth/natural_earth_vector.gpkg' 
##   using driver `GPKG'
## Simple feature collection with 243 features and 119 fields
## Geometry type: POINT
## Dimension:     XY
## Bounding box:  xmin: -175.2206 ymin: -41.29999 xmax: 179.2166 ymax: 64.15002
## Geodetic CRS:  WGS 84
rivers = st_read(ne, 'ne_110m_rivers_lake_centerlines')
## Reading layer `ne_110m_rivers_lake_centerlines' from data source 
##   `/Volumes/Data/Spatial/Natural Earth/natural_earth_vector.gpkg' 
##   using driver `GPKG'
## Simple feature collection with 13 features and 31 fields
## Geometry type: LINESTRING
## Dimension:     XY
## Bounding box:  xmin: -135.3134 ymin: -33.99358 xmax: 129.956 ymax: 72.90651
## Geodetic CRS:  WGS 84
```

Познакомимся с загруженными данными:

```r
plot(ocean %>% st_geometry(), col = 'lightblue')
plot(countries, col = 'white', border = 'grey', add = TRUE)
plot(coast, add = TRUE, col = 'steelblue')
plot(rivers, add = TRUE, col = 'steelblue')
plot(cities, add = TRUE, col = 'black', pch = 19, cex = 0.2)
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-3-1.png" width="100%" />

Перед построением карт мира данные целесообразно спроецировать. Чтобы не трансформировать каждый слой отдельно, можно объединить слои в список и воспользоваться функционалом `lapply` для множественного трансформирования. Для создания списка воспользуемся функцией `lst()` из пакета `tibble`, которая присваивает компонентам списка имена, соответствующие названиям входных переменных (чтобы не писать `ocean = ocean`):

```r
lyr = tibble::lst(ocean, coast, countries, rivers, cities)
lyrp = lapply(lyr, st_transform, crs = "+proj=eck3") # Псевдоцилиндрическая проекция Эккерта

plot(lyrp$countries %>% st_geometry(), col = 'white', border = 'grey', lwd = 0.5)
plot(lyrp$ocean , col = 'lightblue', lwd = 0.5, border = 'steelblue', add = TRUE)
plot(lyrp$rivers, add = TRUE, lwd = 0.5, col = 'steelblue')
plot(lyrp$cities, add = TRUE, col = 'black', pch = 19, cex = 0.1)
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-4-1.png" width="100%" />

### Данные WorldClim {#thematic_mapping_intro_wc}

[WorldClim](http://www.worldclim.org/) --- это открытые сеточные наборы климатических характеристик с пространственным разрешением от $30''$ (около 1 км) до $10'$ (около 20 км). Данные можно выгрузить в виде файлов GeoTiff, однако эту операцию можно сделать и программным путем через пакет __raster__ --- используя функцию `getData()`. 

Выполним загрузку 10-минутного растра с суммарным количеством осадков за год:

```r
prec = raster::getData("worldclim", var = "prec", res = 10) %>% 
  st_as_stars() # преобразуем в stars для удобства работы
plot(prec) # это 12-канальный растр
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-5-1.png" width="100%" />

> Использовать программную загрузку целесообразно для небольших наборов данных. Если счет пошел на десятки мегабайт и выше, следует все-таки выкачать данные в виде файла и работать с ним.

Выполним трансформирование данных в [__проекцию Миллера__](https://proj4.org/operations/projections/mill.html). Для того чтобы карта не обрезалась по охвату растра (он не включает данные на Антарктиду), необходимо расширить его охват на весь земной шар. Для этого используем функцию `extend()` из пакета __raster__:

```r
precm = prec %>% 
  st_warp(crs = "+proj=mill")
  # extend(extent(-180, 180, -90, 90)) %>% 
  # projectRaster(crs = "+proj=mill")
lyrm = lapply(lyr, st_transform, crs = "+proj=mill") # Цилиндрическая проекция Миллера
```

Визуализируем полученные данные на карте:

```r
ramp = colorRampPalette(c("white", "violetred"))

# Визуализируем данные на январь:
plot(precm[,,,1], 
     col = ramp(10),
     main = 'Количество осадков в январе, мм',
     reset = FALSE) # разрешаем добавлять объекты на карту.
## downsample set to c(3,3,1)
plot(st_geometry(lyrm$ocean), border = 'steelblue', 
     col = 'lightblue', add = TRUE)
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-7-1.png" width="100%" />

## Способы изображения {#thematic_mapping_tmap}

> В этом разделе изложение сосредоточено на параметрах способов изображения. Приведение легенд и компоновки карты в аккуратный вид рассматривается далее в разделе [Компоновка](#thematic_mapping_layout).

Пакет __tmap__ предоставляет простой в использовании и достаточно мощный механизм формирования тематических карт. Шаблон построения карты в этом пакете напоминает _ggplot_ и выглядит следующим образом:

```r
tm_shape(<DATA>) +
  tm_<METHOD>(<PARAMETERS>)
```

где:

- `DATA` - объект пространственного типа (`sf`, `sp`, `stars` или `raster`)
- `METHOD` - метод визуализации этого объекта (способ изображения)
- `PARAMETERS` - параметры метода

### Векторные данные {#thematic_mapping_vectors}

Для реализации качественного и количественного фона, а также картограмм используется метод `tm_polygons()`. Он автоматически определяет тип переменной и строит соответствующую шкалу:

```r
tm_shape(lyrp$countries) +
  tm_polygons('economy') + # качественная переменная
tm_shape(lyrp$ocean)+
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-9-1.png" width="100%" />

__Количественный фон__ или __картограммы__ получаются при картографировании числового показателя применением той же функции `tm_polygons()`:

```r
lifexp = WDI::WDI(indicator = 'SP.DYN.LE00.IN')
gap = read_excel('data/gapminder.xlsx', 2)

lifedf = left_join(gap, 
                   filter(lifexp, year == 2016), 
                   by = c('name' = 'country')) |>
  rename(lifexp = SP.DYN.LE00.IN) |> 
  mutate(geo = stringr::str_to_upper(geo))

# (read_sheet('1H3nzTwbn8z4lJ5gJ_WfDgCeGEXK3PVGcNjQ_U5og8eo') %>% # продолжительность жизни
#   rename(name = 1) %>% 
#   gather(year, lifexp, -name) %>% 
#   dplyr::filter(year == 2016) %>% 
#   left_join(read_excel('data/gapminder.xlsx', 2)) %>% 
#   mutate(geo = stringr::str_to_upper(geo)) -> lifedf) # выгружаем данные по продолжительности и сохраняем в переменную lifedf

coun = lyrp$countries %>% 
  left_join(lifedf, by = c('adm0_a3' = 'geo'))

tm_shape(coun) +
  tm_polygons('lifexp', border.col = 'gray20') + # количественная переменная
tm_shape(lyrp$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue4')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-10-1.png" width="100%" />

Для реализации способа __картодиаграмм__ используется геометрия `tm_bubbles()`. Чтобы оставить отображение границ полигонов, нам необходимо к одной геометрии применить несколько способов изображения:

```r
tm_shape(coun) +
  tm_fill(col = 'white') +
  tm_borders(col = 'grey') +
  tm_bubbles('gdp_md_est', 
             scale = 3,
             col = 'red', 
             alpha = 0.5) + # количественная переменная
tm_shape(lyrp$ocean) +
  tm_fill(col = 'lightblue') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-11-1.png" width="100%" />

Аналогичным образом реализуется __значковый способ__ применительно к объектам, локализованным по точкам. Картографируем численность населения по крупнейшим городам:

```r
tm_shape(lyrp$countries) +
  tm_fill(col = 'white') +
  tm_borders(col = 'grey') +
tm_shape(lyrp$ocean) +
  tm_fill(col = 'lightblue') +
  tm_borders(col = 'steelblue') +
tm_shape(lyrp$cities) +
  tm_bubbles('POP2015', col = 'olivedrab', alpha = 0.8)
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-12-1.png" width="100%" />

__Надписи__ объектов на карте размещаются с помощью функции `tm_text`. Данная функция содержит весьма полезные параметры `remove.overlap` и `auto.placement`, которые позволяют убрать перекрывающиеся подписи и автоматически разместить из вокруг точек так, чтобы уменьшить перекрытия с самими знаками и другими подписями. Дополним предыдущую карту названиями городов:

```r
tm_shape(lyrp$countries) +
  tm_fill(col = 'white') +
  tm_borders(col = 'grey') +
tm_shape(lyrp$ocean) +
  tm_fill(col = 'lightblue') +
  tm_borders(col = 'steelblue') +
tm_shape(lyrp$cities) +
  tm_bubbles('POP2015', col = 'olivedrab', alpha = 0.8) +
  tm_text('name_ru', size = 0.5, remove.overlap = TRUE, auto.placement = TRUE)
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-13-1.png" width="100%" />

### Растровые данные {#thematic_mapping_rasters}

При отображении растровых данных используется способ отображения `tm_raster()`. В случае отображения количественных растров Параметр `breaks` определяет границы интервалов, для которых будут использованы цвета, взятые из параметра `palette`:

```r
box = st_bbox(c(xmin = -180, xmax = 180, ymax = 90, ymin = -90), crs = st_crs(4326))
tm_shape(precm[,,,1],
         bbox = box) +
    tm_raster('prec1',
              breaks = c(10, 50, 100, 200, 500, 1000),
              palette = ramp(5)) +
tm_shape(lyrm$ocean) +
  tm_fill(col = 'lightblue') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-14-1.png" width="100%" />

Растровые данные могут хранить и качественную информацию: например, тип почв или вид землепользования. В качестве примера визуализируем типы земельного покрова (land cover) из растрового стека `land`, который есть в пакете __tmap__. Цвета здесь выбираются автоматически, их настройка рассматривается в следующем параграфе:


```r
data(land, package = 'tmap')

tm_shape(land) +
  tm_raster('cover')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-15-1.png" width="100%" />


## Цветовые шкалы {#thematic_color_scales}

Для изменения цветовой шкалы при определении способа изображения вы можете определить параметр `palette`. Пакет __tmap__ позволяет работать с цветовыми палитрами _Color Brewer_ или задавать цвета вручную. Очень удобным инструментом подбора шкалы является функция `palette_explorer()` из пакета __tmaptools__. При вызове функции открывается интерактивное приложение, позволяющее менять настройки цветовых палитр:

```r
tmaptools::palette_explorer()
```

![Приложение Palette Explorer из пакета __tmaptools__](images/palette_explorer.png)

Данных палитр хватит для решения большинства задач по картографической визуализации. Применим категориальную палитру _Dark2_:

```r
tm_shape(lyrp$countries) +
  tm_polygons('economy', palette = 'Dark2') + # качественная переменная
tm_shape(lyrp$ocean)+
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-17-1.png" width="100%" />

Для количественного показателя (количество осадков) применим палитру _PuBuGn_:

```r
tm_shape(precm[,,,1],
         bbox = box) +
    tm_raster('prec1',
              breaks = c(10, 50, 100, 200, 500, 1000),
              palette = 'PuBuGn') +
tm_shape(lyrm$ocean) +
  tm_fill(col = 'lightblue') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-18-1.png" width="100%" />

Вы всегда можете, конечно, определить цвета вручную. В этом случае их количество должно совпадать с количеством интервалов классификации:

```r
tm_shape(precm[,,,1],
         bbox = box) +
    tm_raster('prec1',
              breaks = c(10, 50, 100, 200, 500, 1000),
              palette = c('white', 'gray80', 'gray60', 'gray40', 'gray20')) +
tm_shape(lyrm$ocean) +
  tm_fill(col = 'lightblue') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-19-1.png" width="100%" />

Для категориальных данных необходимо тщательно подбирать цвета, стандартные шкалы тут могут не подойти (более подробно о шкалах --- далее). Для вышеприведенного примера с растром типов земельного покрова можно подобрать следующие цвета:

```r
pal = c("#003200", "#3C9600", "#006E00", "#556E19", "#00C800", "#8CBE8C",
		   "#467864", "#B4E664", "#9BC832", "#EBFF64", "#F06432", "#9132E6",
		   "#E664E6", "#9B82E6", "#B4FEF0", "#646464", "#C8C8C8", "#FF0000",
		   "#FFFFFF", "#5ADCDC")

tm_shape(land) +
  tm_raster('cover', palette = pal)
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-20-1.png" width="100%" />


## Классификация {#thematic_mapping_class}

### Методы классификации {#thematic_mapping_class_methods}

Классификация данных --- важнейший этап картографирования, который во многом определяет, как данные будут представлены на карте и какие географические выводы читатель сделает на ее основе. Существует множество методов классификации числовых рядов. Классифицировать данные автоматически можно с помощью функции `classIntervals()` из пакета `classInt`. Наберите в консоли `?classInt` чтобы прочитать справку о методах классификации.

Посмотрим несколько методов классификации. Первый параметр функции `classInt` --- это числовой ряд. Число классов следует передать в параметр `n =`, метод классификации указывается в параметре `style =`.

Для начала попробуем метод равных интервалов, который просто делит размах вариации (диапазон от минимума до максимум) на $n$ равных интервалов. Функция `plot()` применительно к созданной классификации рисует замечательный график, на котором показаны границы классов и  эмпирическая функция распределения показателя. В параметр `pal` можно передать цветовую палитру:

```r
# Запишем число классов в переменную
nclasses = 5

intervals = classIntervals(coun$lifexp, n = nclasses, style = "equal")

# извлечь полученные границы можно через $brks
intervals$brks
## [1] 51.59300 58.07138 64.54975 71.02813 77.50650 83.98488

plot(intervals, pal = ramp(nclasses), cex=0.5, main = "Равные интервалы MIN/MAX")
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-21-1.png" width="100%" />

Созданные интервалы хоть и равны, но не аккуратны. Зато метод классификации `"pretty"` создает также равные интервалы, но может слегка расширить диапазон или добавить 1 класс, чтобы получить границы интервалов, округленные до целых чисел:

```r
intervals = classIntervals(coun$lifexp, n = nclasses, style = "pretty")
intervals$brks
## [1] 50 55 60 65 70 75 80 85
plot(intervals, pal = ramp(nclasses), cex=0.5, main = "Округленные равные интервалы")
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-22-1.png" width="100%" />

Квантили --- равноколичественные интервалы. В каждом классе содержится одинаковое число объектов:

```r
intervals = classIntervals(coun$lifexp, n = nclasses, style = "quantile")
intervals$brks
## [1] 51.59300 63.70640 70.89153 75.18710 78.64560 83.98488
plot(intervals, pal = ramp(nclasses), cex=0.5, main = "Квантили (равноколичественные)")
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-23-1.png" width="100%" />

Метод "естественных интервалов", или метод Фишера-Дженкса позволяет найти классы, максимально однородные внутри и при этом максимально отличающиеся друг от друга:

```r
intervals = classIntervals(coun$lifexp, n = nclasses, style = "jenks")
intervals$brks
## [1] 51.59300 58.30900 66.20500 72.64400 78.60700 83.98488
plot(intervals, pal = ramp(nclasses), cex=0.5, main = "Естественные интервалы")
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-24-1.png" width="100%" />

### Применение на картах {#thematic_mapping_class_application}

Чтобы использовать заранее вычисленные интервалы классификации, их необходимо подать в параметр `breaks` при построении карты:

```r
brks = classIntervals(coun$lifexp, n = 4, style = "pretty")$brks

tm_shape(coun) +
  tm_polygons('lifexp', 
              border.col = 'gray20',
              palette = 'YlGn',
              breaks = brks) + # количественная переменная
tm_shape(lyrp$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue4')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-25-1.png" width="100%" />

Аналогичным путем работают шкалы для растровых данных:

```r
tm_shape(precm[,,,1],
         bbox = box) +
    tm_raster('prec1',
              breaks = classIntervals(precm[,,,1][[1]], n = 5, style = "quantile", na.rm = TRUE)$brks,
              palette = 'PuBuGn') +
tm_shape(lyrm$ocean) +
  tm_fill(col = 'lightblue') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-26-1.png" width="100%" />

> Учтите, что метод естественных интервалов --- ресурсоемкий в вычислительном плане. Поэтому если вы хотите с его помощью классифицировать растровые данные, целесообразно сделать выборку не более чем из нескольких тысяч пикселов. Иначе придется долго ждать.

Для классификации естественными интервалами сделаем выборку в 2 000 значений с растра c помощью функции `sampleRandom()` из пакета __raster__:

```r
smpl = sample(precm[,,,1][[1]], 2000) 

tm_shape(precm[,,,1],
         bbox = box) +
    tm_raster('prec1',
              breaks = classIntervals(smpl, n = 5, style = "jenks")$brks,
              palette = 'PuBuGn') +
tm_shape(lyrm$ocean) +
  tm_fill(col = 'lightblue') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-27-1.png" width="100%" />

### Классификация при отображении {#thematic_mapping_class_application}

Пакет __tmap__ позволяет выполнять классификацию данных непосредственно при отображении. Это бывает удобно, когда одну и ту же классификацию не надо использовать несколько раз, и когда нет необходимости делать выборку значений (как в случае метода естественных интервалов). Для этого функции способов изображения предлагают несколько параметров:

- `n` --- количество классов
- `style` --- метод классификации (так же как и в `classIntervals()`)
- `breaks` --- значения границ интервалов (необходимы, если `style == fixed`)
- `interval.closure` --- замыкание интервала (по умолчанию стоит `left`, что означает, что в интервал включается нижняя граница, за исключением последнего интервала, включающего и нижнюю и верхнюю границу) 
- `midpoint` --- нейтральное значение, которое используется для сопоставления с центральным цветом в расходящихся цветовых палитрах

Построим карту продолжительности жизни, используя классификацию при отображении:

```r
tm_shape(coun) +
  tm_polygons('lifexp', 
              palette = 'YlGn',
              n = 5,
              style = 'fisher',
              border.col = 'gray20') + # количественная переменная
tm_shape(lyrp$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue4')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-28-1.png" width="100%" />

Установка средней точки при классификации оказывается очень полезной в тех случаях, когда данные являются биполярными. Покажем это на примере данных WorldClim по температуре:

```r
temp = raster::getData("worldclim", var = "tmean", res = 10) %>% 
  st_as_stars() %>% 
  mutate(tmean1 = tmean1 / 10) %>% 
  st_warp(crs = "+proj=mill")
  # extend(extent(-180, 180, -90, 90)) %>% 
  # projectRaster(crs = "+proj=mill") / 10

 # не забываем поделить результат на 10, 
 # так как данные хранятся в виде целых чисел!
```

Визуализируем данные по температуре, используя классическую красно-бело-синюю палитру _RdBu_ и нейтральную точку 0 градусов по Цельсию. По умолчанию в данной палитре красный цвет соответствует малым значениям. пакет __tmap__ позволяет инвертировать цвета палитры, добавив знак минус перед ее названием. Помимо этого, для размещения положительных значений наверху выполним обратную сортировку элементов легенды, используя параметр `legend.reverse = TRUE`: 

```r
tm_shape(temp[,,,1],
         bbox = box) +
    tm_raster('tmean1',
              n = 11,
              midpoint = 0,
              style = 'pretty',
              legend.reverse = TRUE,
              palette = '-RdBu') +
tm_shape(lyrm$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-30-1.png" width="100%" />

### Пропущенные данные {#thematic_mapping_na}

Весьма важно отметить на карте области, для которых данные отсутствуют. Вы могли обратить внимание, что для способов изображения, применимых к векторным данным, _tmap_ автоматически добавляет класс легенды, который отвечает за пропуски. Для растров, однако, он это не делает. Чтобы принудительно вывести в легенду и на карту символ, отвечающий за пропущенные значения, необходимо определить параметр `colorNA`. Обычно, в зависимости от цветовой палитры легенды, для этого используют серый или белый цвет:

```r
tm_shape(temp[,,,1],
         bbox = box) +
    tm_raster('tmean1',
              colorNA = 'grey', # определяем цвет для пропущенных значений
              n = 11,
              midpoint = 0,
              style = 'pretty',
              legend.reverse = TRUE,
              palette = '-RdBu') +
tm_shape(lyrm$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-31-1.png" width="100%" />


## Компоновка {#thematic_mapping_layouts}

Пакет __tmap__ предоставляет широкий набор настроек компоновки картографического изображения, который включает настройку легенды, заголовка карты и ряда других важных параметров. Большинство настроек компоновки осуществляется через функцию `tm_layout()`, однако часть из них, специфичная для конкретного слоя, определяется непосредственно при настройке способа изображения.

В примере ниже показано, как:

- добавить заголовок карты (`main.title`), 
- разместить легенду в нижнем левом углу (`legend.position = c('left', 'bottom')`)
- поместить ее легенду в полупрозрачный прямоугольник (параметры `legend<...>`), 
- убрать заголовок легенды (`title`),
- заменить стандартный шрифт на _Open Sans_ (`fontfamily`):


```r
tm_shape(lyrp$countries) +
  tm_polygons('economy', title = '') + # убираем заголовок легенды
tm_shape(lyrp$ocean)+
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue') +
tm_layout(legend.position = c('left', 'bottom'),
          fontfamily = 'Open Sans', # шрифт
          main.title.size = 1.2,   # масштаб шрифта в заголовке
          main.title = 'Тип экономики', # заголовок
          legend.frame = TRUE, # рамка вокруг легенды
          legend.frame.lwd = 0.2, # толщина рамки вокруг легенды
          legend.bg.alpha = 0.8, # прозрачность фона в легенде
          legend.bg.color = 'white') # цвет фона легенды
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-32-1.png" width="100%" />

Для того чтобы определить заголовок легенды размера значка или диаграммы, необходимо задать параметр `title.size`. Помимо этого, легенду можно пристыковать непосредственно к рамке карты, если задать значения параметра `legend.position` в верхнем регистре:

```r
tm_shape(coun) +
  tm_fill(col = 'white') +
  tm_borders(col = 'grey') +
  tm_bubbles('gdp_md_est', 
             scale = 2.5,
             col = 'red', 
             alpha = 0.5,
             title.size = '$ млн') + # количественная переменная
tm_shape(lyrp$ocean) +
  tm_fill(col = 'lightblue') +
  tm_borders(col = 'steelblue') +
tm_layout(legend.position = c('LEFT', 'BOTTOM'), # верхний регистр — легенда встык
          fontfamily = 'Open Sans', # шрифт
          main.title.size = 1.2,   # масштаб шрифта в заголовке
          main.title = 'Валовый внутренний продукт стран мира', # заголовок
          frame.lwd = 2,
          legend.frame = TRUE, # рамка вокруг легенды
          legend.frame.lwd = 0.5, # толщина рамки вокруг легенды
          legend.bg.color = 'white') # цвет фона легенды
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-33-1.png" width="100%" />

По умолчанию __tmap__ размещает легенду внутри фрейма картографического изображения. Однако ее можно вынести и наружу, используя параметр `legend.outside` функции `tm_layout()`. В примере ниже показано также, как можно 

- задать текст легенды для отсутствующих данных (`textNA`),
- отформатировать разделитель в легенде с интервалами значений (`legend.format`),
- убрать рамку карты (`frame`),
- сдвинуть заголовок вдоль строки, выровняв его с центром карты (`main.title.position`):

```r
tm_shape(coun) +
  tm_polygons('lifexp', 
              border.col = 'gray20', 
              palette = 'YlGn',
              n = 4,
              style = 'jenks',
              title = 'Лет',
              colorNA = 'lightgray',
              textNA = 'Нет данных',
              legend.format = list(text.separator = '—')) + # количественная переменная
tm_shape(lyrp$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue4') +
tm_layout(frame = FALSE,
          main.title.position = 0.15,
          legend.outside = TRUE,
          legend.outside.position = 'right',
          fontfamily = 'Open Sans',
          main.title.size = 1.2,
          main.title = 'Продолжительность жизни',
          legend.bg.color = 'white')
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-34-1.png" width="100%" />

Для отображения __координатной сетки__ вы можете использовать функцию `tm_grid()`. По умолчанию она строит координатную сетку в единицах измерения проекции. Однако если требуется градусная сетка, то ее можно определить, используя параметр `projection = 4326`:

```r
tm_shape(temp[,,,1],
         bbox = box) +
  tm_raster('tmean1',
            title = '°C',
            colorNA = 'grey', # определяем цвет для пропущенных значений
            textNA = 'Нет данных',
            legend.format = list(text.separator = '—'),
            n = 11,
            midpoint = 0,
            style = 'pretty',
            legend.reverse = TRUE,
            palette = '-RdBu') +
tm_shape(lyrm$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue') +
tm_layout(legend.position = c('left', 'bottom'),
          fontfamily = 'Open Sans',
          main.title.size = 1.2,
          main.title = 'Средняя температура января',
          legend.frame = TRUE,
          legend.frame.lwd = 0.2,
          legend.bg.alpha = 0.5,
          legend.bg.color = 'white') # +
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-35-1.png" width="100%" />

```r
# tm_graticules(x = seq(-180, 180, by = 30), 
#         y = seq(-90, 90, by = 30), 
#         lwd = 0.2,
#         col = "black")
```

Если вам необходимо обеспечить значки градуса, вы можете сделать это, используя параметр `labels.format`, определив в нем анонимную функцию, добавляющую значок градуса в переданный ей вектор подписей. 

Помимо этого, вам может понадобиться увеличить поля вокруг карты, чтобы освободить пространство для размещения меток (на предыдущей карте они не влезли). Это делается через параметр `outer.margins`, который ожидает получить вектор из четырех значений (по умолчанию все они равны 0.02, т.е. 2% от размера окна).


```r
tm_shape(temp[,,,1],
         bbox = box) +
  tm_raster('tmean1',
            title = '°C',
            colorNA = 'grey', # определяем цвет для пропущенных значений
            textNA = 'Нет данных',
            legend.format = list(text.separator = '—'),
            n = 11,
            midpoint = 0,
            style = 'pretty',
            legend.reverse = TRUE,
            palette = '-RdBu') +
tm_shape(lyrm$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue') +
tm_layout(legend.position = c('LEFT', 'BOTTOM'),
          fontfamily = 'Open Sans',
          main.title.size = 1.2,
          main.title = 'Средняя температура января',
          legend.frame = TRUE,
          legend.frame.lwd = 0.2,
          legend.bg.alpha = 0.8,
          legend.bg.color = 'white',
          outer.margins = c(0.05, 0.02, 0.02, 0.02),
          inner.margins = c(0, 0, 0, 0)) # +
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-36-1.png" width="100%" />

```r
# tm_grid(x = seq(-180, 180, by = 30), 
#         y = seq(-90, 90, by = 30), 
#         lwd = 0.2,
#         col = "black", 
#         projection = 4326,
#         labels.inside.frame = FALSE,
#         labels.format = list(fun = function(X) paste0(X, '°')))
```

Подписи сетки координат можно добавить и для более сложных проекций, однако располагаться они будут по-прежнему вдоль осей _X_ и _Y_. В примере ниже также показано как можно увеличить расстояние между заголовком и картой, определив более крупный отступ от верхней стороны в параметре `inner.margins`:

```r
tm_shape(coun) +
  tm_polygons('lifexp', 
              palette = 'YlGn',
              n = 4,
              style = 'jenks',
              border.col = 'gray20', 
              title = 'Лет',
              colorNA = 'lightgray',
              textNA = 'Нет данных',
              legend.reverse = TRUE,
              legend.format = list(text.separator = '—')) + # количественная переменная
tm_shape(lyrp$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue4') +
tm_layout(frame = FALSE,
          main.title.position = 0.22,
          legend.outside = TRUE,
          legend.outside.position = 'right',
          fontfamily = 'Open Sans',
          main.title.size = 1.2,
          main.title = 'Продолжительность жизни',
          legend.bg.color = 'white',
          outer.margins = c(0.02, 0.05, 0.02, 0.02),
          inner.margins = c(0.02, 0.02, 0.07, 0.02)) +
tm_grid(x = seq(-180, 180, by = 60), 
        y = seq(-90, 90, by = 30), 
        lwd = 0.2,
        col = "black", 
        projection = 4326,
        labels.inside.frame = FALSE,
        labels.format = list(fun = function(X) paste0(X, '°')))
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-37-1.png" width="100%" />

## Фасеты и серии карт {#thematic_mapping_facets}

Фасетная компоновка предполагает, упорядочение элементов в матричной форме на одной странице. Как правило, картографические фасеты идентичны по содержанию, но показывают одно и то же явление при различных заданных условиях: за разные года, по разным странам и т.д. Создание фасет с помощью __tmap__ осуществляется с помощью специальной функции `tm_facets()`, которой необходимо передать название переменной, отвечающей за разделение. В свою очередь, это означает, что данные должны быть приведены к «длинной» форме (если информация за разные года содержится в разных столбцах, то нужно год записать в отдельную переменную). Здесь вам пригодится знание пакета __tidyr__.

Рассмотрим создание фасет на примере данных Gapminder по средней продолжительности жизни c 1960 по 2010 г:

```r
lifexp_dec = lifexp |> 
  filter(year %in% c(1960, 1970, 1980, 1990, 2000, 2010))

lifedf_dec = left_join(gap, lifexp_dec, by = c('name' = 'country')) |>
  rename(lifexp = SP.DYN.LE00.IN) |> 
  mutate(geo = stringr::str_to_upper(geo))

# ('1H3nzTwbn8z4lJ5gJ_WfDgCeGEXK3PVGcNjQ_U5og8eo' %>% # продолжительность жизни
#   read_sheet() %>% 
#   rename(name = 1) %>% 
#   gather(year, lifexp, -name) %>% 
#   dplyr::filter(year %in% c(1960, 1970, 1980, 1990, 2000, 2010)) %>% 
#   left_join(read_excel('data/gapminder.xlsx', 2)) %>% 
#   mutate(geo = stringr::str_to_upper(geo)) -> lifedf2) # выгружаем данные по ВВП на душу населения и сохраняем в переменную lifedf

coun_dec = lyrp$countries |>  
  left_join(lifedf_dec, by = c('adm0_a3' = 'geo'))
```

Создадим серию карт за разные года:

```r
tm_shape(coun_dec) +
  tm_polygons('lifexp', 
              palette = 'YlGnBu',
              n = 3,
              style = 'pretty',
              border.col = 'gray20', 
              title = 'Лет',
              colorNA = 'lightgray',
              textNA = 'Нет данных',
              legend.reverse = TRUE,
              legend.format = list(text.separator = '—')) + # количественная переменная
  tm_facets(by = 'year',
            free.coords = FALSE,
            drop.units = TRUE,
            drop.NA.facets = TRUE,
            ncol = 2) +
tm_shape(lyrp$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue4') +
tm_layout(frame = FALSE,
          legend.outside = TRUE,
          legend.outside.position = 'bottom',
          fontfamily = 'Open Sans',
          main.title.size = 1.2,
          main.title = 'Средняя продолжительность жизни',
          legend.bg.color = 'white',
          outer.margins = c(0.02, 0.1, 0.02, 0.02),
          inner.margins = c(0.02, 0.02, 0.07, 0.02))
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-39-1.png" width="100%" />

Фасетные карты по растровым данным в настоящий момент не поддерживаются в пакете __tmap__, но вы можете создать их, используя функцию `tmap_arrange()`, которая принимает на вход список из карт __tmap__ и упорядочивает их в фасетной компоновке. 

В примере ниже показано, как:

- вычислить равноступенную шкалу, единую для всех карт --- используя максимум и минимум по всем растрам из стека, а также функцию `fullseq()` из пакета [__scales__](https://cran.r-project.org/web/packages/scales/index.html), заведомо накрывающую указанный диапазон значений интервалами заданного размера.
- применить функционал `map2()`из пакета [__purrr__](https://purrr.tidyverse.org/) (входит в __tidyverse__) для одновременной итерации по двум спискам: названий растров в стеке (`X`) и названий месяцев (`Y`), которые нужны для формирования заголовков
- упорядочить карты по регулярной сетке с двумя столбцами и полями отступа каждой фасеты (параметр `outer.margins`), используя `tmap_arrange()`


```r
minval = min(temp[[1]], na.rm = TRUE)
maxval = max(temp[[1]], na.rm = TRUE)

brks = scales::fullseq(c(minval, maxval), 10)

months = c('Январь', 'Февраль', 'Март', 'Апрель', 
           'Март', 'Июнь', 'Июль', 'Август', 
           'Сентябрь', 'Октябрь', 'Ноябрь', 'Декабрь')

tm_shape(temp,
         bbox = box) +
    tm_raster('temp1',
              title = '°C',
              colorNA = 'grey', # определяем цвет для пропущенных значений
              textNA = 'Нет данных',
              legend.format = list(text.separator = '—'),
              breaks = brks,
              midpoint = 0,
              style = 'fixed',
              legend.reverse = TRUE,
              palette = '-RdBu') +
  tm_shape(lyrm$ocean) +
    tm_fill(col = 'azure') +
    tm_borders(col = 'steelblue') +
  tm_layout(legend.position = c('LEFT', 'BOTTOM'),
            fontfamily = 'Open Sans',
            main.title.size = 1.2,
            main.title = 'Среднемесячная температура',
            legend.frame = TRUE,
            legend.frame.lwd = 0.2,
            legend.bg.alpha = 0.8,
            legend.bg.color = 'white',
            inner.margins = c(0, 0, 0, 0)) #+
```

<img src="11-ThematicMapping_files/figure-html/unnamed-chunk-40-1.png" width="100%" />

```r
  # tm_grid(x = seq(-180, 180, by = 30), 
  #         y = seq(-90, 90, by = 30), 
  #         lwd = 0.2,
  #         col = "black", 
  #         projection = 4326,
  #         labels.inside.frame = FALSE,
  #         labels.format = list(fun = function(Z) paste0(Z, '°')))

# tmap_arrange(maps, asp = NA, ncol = 2,
#              outer.margins = 0.05)
```

## Картографические анимации {#thematic_mapping_animations}

Картографические анимации вы пакете __tmap__ создаются путем следующей последовательности действий:

1. Добавить в построение карты функцию `tm_facets(along = "name")`, где `name` — название атрибута, значения которого отвечают за каждый кадр анимации.
2. Записать созданную карту в переменную (условно назовем ее `map`).
3. Вызвать для созданной переменной функцию `tmap_animation(map, filename = "filename.gif", delay = 25)`, определив имя файла и задержку в миллисекундах между кадрами.

> __Внимание:__ для того чтобы работало построение анимаций средствами __tmap__, на вашем компьютере должна быть установлена библиотека [__ImageMagick__](https://www.imagemagick.org/).

Для примера построим анимацию по данным изменения средней продолжительности жизни:

```r
map = tm_shape(coun_dec) +
  tm_polygons('lifexp', 
              palette = 'YlGnBu',
              n = 3,
              style = 'pretty',
              border.col = 'gray20', 
              title = 'Лет',
              colorNA = 'lightgray',
              textNA = 'Нет данных',
              legend.reverse = TRUE,
              legend.format = list(text.separator = '—')) + # количественная переменная
  tm_facets(along = 'year',
            free.coords = FALSE,
            drop.units = TRUE) +
tm_shape(lyrp$ocean) +
  tm_fill(col = 'azure') +
  tm_borders(col = 'steelblue4') 

tmap_animation(map, 'images/lifexp.gif', delay = 100)
```

![](images/lifexp.gif)

## Интерактивные карты  {#thematic_mapping_interactive}

Любую карту tmap можно перевести в интерактивный режим с помощью функции `tmap_mode()` с параметром `'view'`. Управлять дополнительными параметрами, специфичными для интерактивного режима, можно используя функцию `tm_view()`. В частности, можно установить координаты центра карты и масштабный уровень в параметре `set.view` и ограничить диапазон масштабных уровней в параметре `set.zoom.limits`. Состав полей, значения которых отображаются во всплывающем окне при щелчке на символе, определяются параметром `popup.vars`:

```r
tmap_mode('view')
tmap_options(check.and.fix = TRUE)

tm_shape(coun) +
  tm_polygons('lifexp', 
              border.col = 'gray20', 
              palette = 'YlGn',
              n = 4,
              style = 'jenks',
              title = 'Лет',
              colorNA = 'lightgray',
              textNA = 'Нет данных',
              legend.format = list(text.separator = '—'),
              popup.vars = c('sovereignt', 'lifexp')) + # поля для всплывающего окна
tm_view(set.view = c(20, 45, 2),    # центр карты и масштабный уровень
        set.zoom.limits = c(1, 4))
```

Чтобы добавить карту-подложку, необходимо предварительно вызвать функцию `tm_basemap()`, передав ей название картографического сервиса. В примере ниже также показано, как можно сделать размер кружка постоянным во всех масштабах (параметр `symbol.size.fixed`):

```r
tmap_mode('view')
tmap_options(check.and.fix = TRUE)

coun = coun %>% mutate(gdp_scaled = round(0.001 * gdp_md_est))

tm_basemap("OpenStreetMap") +
tm_shape(coun) +
  tm_borders(col = 'black', alpha = 0.5, lwd = 0.3) +
tm_shape(st_point_on_surface(coun)) + # делаем точки, чтобы диаграммы были точно внутри
  tm_bubbles('gdp_scaled', 
             scale = 3,
             col = 'violetred', 
             alpha = 0.5,
             popup.vars = c('sovereignt', 'gdp_scaled')) +
  tm_text('gdp_scaled', size = 'gdp_scaled', 
          remove.overlap = TRUE,
          size.lowerbound = 0.2,
          scale = 2) +
tm_view(set.view = c(20, 45, 3),
        set.zoom.limits = c(2, 4),
        symbol.size.fixed = TRUE,
        text.size.variable = TRUE)
```

## Контрольные вопросы и упражнения {#questions_tasks_tmap}

### Вопросы {#questions_tmap}

1. Опишите шаблон построения тематической карты средствами __tmap__. Что из себя представляют его три основные компоненты?
1. Могут ли на одной тематической карте комбинироваться пространственные данные в разных проекциях?
1. Перечислите названия функций, отвечающих за отображение полигонов, линий и окружностей средствами tmap.
1. Чему должно быть равно значение параметра `col` при отображении одноканального растра в случае если классификация и цвета определяются посредством параметров `breaks` и `palette`?
1. Опишите порядок использования функции `classIntervals()` и ее основные параметры.
1. Перечислите методы классификации, доступные в `classIntervals()`, а также принципы и работы. Какой из методов наиболее трудоемок в вычислительном плане?
1. В каком соотношении должно быть количество граничных классов и количество цветов при классификации?
1. График какой функции отображается при вызове функции `plot()` применительно к результату выполнения `classIntervals()`?
1. Какие возможности существуют для применения классификации при построении карт средствами tmap? Обязательно ли заранее определять количество классов? В каком случае это может быть полезно.
1. Как можно изменить порядок размещения элементов легенды в tmap?
1. Опишите возможности управления расположением и внутренним форматированием легенды средствами tmap.
1. С помощью какой функции можно построить координатную сетку на карте tmap?
1. Как добавить значки градусов в подписи выходов сетки координат на карте tmap?
1. Какие параметры позволяют управлять внешними и внутренними полями карты tmap?
1. Опишите последовательность действий, которую необходимо реализовать для построения фасетной карты средствами tmap. Как можно реализовать построение таких карт на основе растровых данных?
1. Опишите последовательность действий, которую необходимо реализовать для построения картографических анимаций средствами tmap. Какая библиотека должна быть установлена для этого на компьютере пользователя? 
1. Каким образом можно перевести отображение карт tmap в интерактивный режим? А обратно в статичный?
1. Расскажите, что вы знаете о данных Natural Earth. На каком сайте они размещены? Сколько существует масштабных уровней? В каких форматах доступны данные? Как получить доступ к ним программным путем непосредственно из среды R?


### Упражнения {#tasks_tmap}

1. Используя возможности пакетов __rnaturalearth__ и __tmap__, создайте карту мира, в которой страны раскрашены в соответствии с континентом (переменная _continent_). Визуализируйте ее в статичном и интерактивном режиме.

2. Скачайте [цифровую модель рельефа GEBCO](https://github.com/tsamsonov/r-geo-course/blob/master/data/world/gebco.tif). Используя слои `ocean` и `land` из масштаба `110` данных Natural Earth, разделите ее на два растра, отвечающих за рельефа суши и моря соответственно. Подберите для них классификации и создайте физическую карту мира, которая будет содержать помимо рельефа также основные объекты гидрографии.

3. Выполните выборку стран из набора данных _Natural Earth_ масштаба $50$ на Европейский континент. Трансформируйте данные о странах в коническую равнопромежуточную проекцию. Визуализируйте численность населения по странам (переменная _pop\_est_) способом картодиаграмм. Добавьте на карту реки, озера и города, используя возможности `ne_download()`.

----
_Самсонов Т.Е._ **Визуализация и анализ географических данных на языке R.** М.: Географический факультет МГУ, `lubridate::year(Sys.Date())`. DOI: [10.5281/zenodo.901911](https://doi.org/10.5281/zenodo.901911)
----
