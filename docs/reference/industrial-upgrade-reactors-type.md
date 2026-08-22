# Типы реакторов Industrial Upgrade

## Источник данных

Все значения в этом документе зафиксированы по исходникам Industrial Upgrade на commit:

`16b7fb9cfb7fbb3171e35a63532ebc0a112f665c`

Minecraft:

`1.12.2`

Forge:

`14.23.5.2860`

Основные источники:

* `src/main/java/com/denfop/api/reactors/ITypeRector.java`
* `src/main/java/com/denfop/api/reactors/EnumReactors.java`

## Базовые типы реакторов

Industrial Upgrade определяет четыре базовых типа реакторов:

| Тип IU             | Назначение                          |
| ------------------ | ----------------------------------- |
| `FLUID`            | Жидкостный реактор                  |
| `HIGH_SOLID`       | Твердотопливный / тепловой реактор  |
| `GRAPHITE_FLUID`   | Графитовый жидкостный реактор       |
| `GAS_COOLING_FAST` | Газовый реактор быстрого охлаждения |

## Полный список реакторов

### 1. Fluid Reactor

Базовый тип IU: `FLUID`

| ID   | Название               | Размер | Stable Heat | Max Heat | Radiation |
| ---- | ---------------------- | -----: | ----------: | -------: | --------: |
| `FS` | Water Reactor          |    3×3 |         500 |      750 |    100000 |
| `FA` | Advanced Water Reactor |    4×4 |         950 |     1400 |    150000 |
| `FI` | Improved Water Reactor |    5×5 |        1500 |     2300 |    300000 |
| `FP` | Perfect Water Reactor  |    6×6 |        2300 |     3500 |    600000 |

Исходные имена multiblock:

* `WaterReactorMultiBlock`
* `AdvWaterReactorMultiBlock`
* `ImpWaterReactorMultiBlock`
* `PerWaterReactorMultiBlock`

---

### 2. High Solid Reactor

Базовый тип IU: `HIGH_SOLID`

| ID   | Название              | Размер | Stable Heat | Max Heat | Radiation |
| ---- | --------------------- | -----: | ----------: | -------: | --------: |
| `HS` | Heat Reactor          |    4×4 |         700 |     1250 |    100000 |
| `HA` | Advanced Heat Reactor |    5×5 |        1500 |     1900 |    150000 |
| `HI` | Improved Heat Reactor |    6×6 |        2100 |     2450 |    300000 |
| `HP` | Perfect Heat Reactor  |    7×7 |        2800 |     3500 |    600000 |

Исходные имена multiblock:

* `HeatReactorMultiBlock`
* `advHeatReactorMultiBlock`
* `impHeatReactorMultiBlock`
* `perHeatReactorMultiBlock`

---

### 3. Graphite Fluid Reactor

Базовый тип IU: `GRAPHITE_FLUID`

| ID    | Название                  | Размер | Stable Heat | Max Heat | Radiation |
| ----- | ------------------------- | -----: | ----------: | -------: | --------: |
| `GRS` | Graphite Reactor          |    4×3 |         500 |      750 |    100000 |
| `GRA` | Advanced Graphite Reactor |    6×4 |         950 |     1400 |    150000 |
| `GRI` | Improved Graphite Reactor |    7×5 |        1500 |     2300 |    300000 |
| `GRP` | Perfect Graphite Reactor  |    9×5 |        2500 |     3500 |    600000 |

Исходные имена multiblock:

* `GraphiteReactorMultiBlock`
* `advGraphiteReactorMultiBlock`
* `impGraphiteReactorMultiBlock`
* `perGraphiteReactorMultiBlock`

---

### 4. Gas Cooling Fast Reactor

Базовый тип IU: `GAS_COOLING_FAST`

| ID   | Название             | Размер | Stable Heat | Max Heat | Radiation |
| ---- | -------------------- | -----: | ----------: | -------: | --------: |
| `GS` | Gas Reactor          |    3×4 |         500 |      750 |    100000 |
| `GA` | Advanced Gas Reactor |    4×5 |         950 |     1400 |    150000 |
| `GI` | Improved Gas Reactor |    5×6 |        1500 |     2300 |    300000 |
| `GP` | Perfect Gas Reactor  |    7×6 |        2500 |     3500 |    600000 |

Исходные имена multiblock:

* `GasReactorMultiBlock`
* `AdvGasReactorMultiBlock`
* `ImpGasReactorMultiBlock`
* `PerGasReactorMultiBlock`

## Размеры сетки

Размер, указанный в `EnumReactors`, является рабочей сеткой компонентов реактора:

| ID    | Ширина | Высота |
| ----- | -----: | -----: |
| `FS`  |      3 |      3 |
| `FA`  |      4 |      4 |
| `FI`  |      5 |      5 |
| `FP`  |      6 |      6 |
| `GS`  |      3 |      4 |
| `GA`  |      4 |      5 |
| `GI`  |      5 |      6 |
| `GP`  |      7 |      6 |
| `GRS` |      4 |      3 |
| `GRA` |      6 |      4 |
| `GRI` |      7 |      5 |
| `GRP` |      9 |      5 |
| `HS`  |      4 |      4 |
| `HA`  |      5 |      5 |
| `HI`  |      6 |      6 |
| `HP`  |      7 |      7 |

## Примечания

`Stable Heat` и `Max Heat` являются характеристиками, заданными непосредственно в `EnumReactors`.

`Radiation` также берётся из `EnumReactors` и используется как базовое значение соответствующего реактора.

На первом этапе планера необходимо поддержать все 16 реакторов.

Тип реактора (`ITypeRector`) и конкретная модель (`EnumReactors`) должны быть разделены в архитектуре приложения:

```text
ReactorType
    ├── FLUID
    ├── HIGH_SOLID
    ├── GRAPHITE_FLUID
    └── GAS_COOLING_FAST

ReactorDefinition
    ├── FS
    ├── FA
    ├── ...
    └── HP
```

Это позволит использовать общую логику для семейства реакторов и отдельно хранить параметры конкретной модели.

## Источник

Industrial Upgrade:

https://github.com/ZelGimi/industrialupgrade

Зафиксированный commit:

https://github.com/ZelGimi/industrialupgrade/commit/16b7fb9cfb7fbb3171e35a63532ebc0a112f665c
