# Исследование ItemReactorHeatExchanger в контексте LogicComponent

## 1. Источник

Industrial Upgrade:

- Repository: https://github.com/ZelGimi/industrialupgrade
- Commit: `16b7fb9cfb7fbb3171e35a63532ebc0a112f665c`
- Minecraft: `1.12.2`
- Forge: `14.23.5.2860`

Основной источник:

`src/main/java/com/denfop/items/reactors/ItemReactorHeatExchanger.java`

Связанная логика:

- `src/main/java/com/denfop/api/reactors/IReactorItem.java`
- `src/main/java/com/denfop/api/reactors/IAdvReactor.java`
- `src/main/java/com/denfop/api/reactors/LogicComponent.java`
- `src/main/java/com/denfop/api/reactors/LogicReactor.java`
- `src/main/java/com/denfop/IUItem.java`

---

## 2. Назначение

`ItemReactorHeatExchanger` — компонент типа `HEAT_EXCHANGER`.

Компонент:

- не генерирует собственное тепло;
- принимает тепло от соседних компонентов;
- может передавать накопленное тепло дальше;
- не производит энергию;
- не производит radiation;
- не ремонтирует себя;
- не ремонтирует другие компоненты;
- имеет durability;
- участвует в `LogicComponent.onTick()`;
- получает damage от накопленного heat.

---

## 3. Структура класса

Класс:

```java
public class ItemReactorHeatExchanger extends ItemDamage implements IReactorItem
```

Собственные поля:

```java
private final int level;
private final int heat_to_damage;
private final double heat_damage;
```

Конструктор:

```java
public ItemReactorHeatExchanger(
    final String name,
    final int maxDamage,
    int level,
    int heat_to_damage,
    double heat_damage
)
```

При создании:

```java
setMaxStackSize(1);
```

Максимальная durability передаётся в `ItemDamage`.

---

## 4. Все варианты

В baseline зарегистрированы четыре варианта:

| ID | Max Damage | Level | Heat → Damage | Heat Removal |
|---|---:|---:|---:|---:|
| `heat_exchange` | 2500 | 1 | 10 | 0.80 |
| `adv_heat_exchange` | 5000 | 2 | 12 | 0.75 |
| `imp_heat_exchange` | 7500 | 3 | 15 | 0.60 |
| `per_heat_exchange` | 10000 | 4 | 20 | 0.45 |

`heat_damage` — базовый коэффициент приема тепла.

`heat_to_damage` — делитель при расчёте damage от накопленного heat.

---

## 5. Реализация IReactorItem

| Метод | Результат |
|---|---|
| `getType()` | `HEAT_EXCHANGER` |
| `getLevel()` | `level` |
| `getAutoRepair()` | `0` |
| `getRepairOther()` | `0` |
| `getDamageCFromHeat()` | `heat_to_damage` |
| `getHeat()` | `0` |
| `getHeatRemovePercent()` | `heat_damage × reactor.getModuleExchanger()` |
| `damageItem()` | `applyCustomDamage(stack, damage, null)` |
| `updatableItem()` | `true` |
| `caneExtractHeat()` | `true` |
| `getEnergyProduction()` | `0` |
| `getRadiation()` | `0` по default реализации `IReactorItem` |
| `needClear()` | remaining durability == 0 |

---

## 6. Начальное heat

`ItemReactorHeatExchanger` возвращает:

```java
getHeat() = 0;
```

При создании `LogicComponent`:

```java
this.heat = item.getHeat(reactor);
```

Поэтому первоначально:

```text
LogicComponent.heat = 0
```

Это означает отсутствие собственного выделения тепла.

После построения графа `LogicReactor` может увеличивать поле `LogicComponent.heat` за счёт соседних компонентов.

---

## 7. Роль Heat Exchanger в тепловом графе

В отличие от `ItemReactorPlate`, Heat Exchanger:

```java
caneExtractHeat() = true;
```

Поэтому является промежуточным узлом тепловой сети.

Например:

```text
ROD
 ↓
HEAT_EXCHANGER
 ↓
VENT
```

или:

```text
ROD
 ↓
HEAT_EXCHANGER
 ↓
HEAT_EXCHANGER
```

может быть валидной цепочкой.

---

## 8. Формула получения heat

`LogicReactor` использует:

```java
lg.heat +=
    ((component.heat / col)
    * lg.getItem().getHeatRemovePercent(this.reactor))
    * reactor.getMulHeat(
        lg.getX(),
        lg.getY(),
        lg.getStack()
    );
```

Для Heat Exchanger:

```java
getHeatRemovePercent(reactor) =
    heat_damage × reactor.getModuleExchanger();
```

Итоговая формула `LogicReactor`:

```text
receivedHeat =
    sourceHeat / col
    × heat_damage
    × moduleExchanger
    × mulHeat
```

где:

```text
moduleExchanger =
    модификатор Reactor Modules

mulHeat =
    reactor.getMulHeat(x, y, stack)
```

Для Fluid, Gas и Heat реакторов `mulHeat = 1.0`.

Для Graphite Reactor `mulHeat` зависит от exchanger-блоков, назначенных на колонку `x`.

где `heat_damage` зависит от уровня:

```text
1 → 0.80
2 → 0.75
3 → 0.60
4 → 0.45
```

---

## 9. Разница между heat_damage и heat_to_damage

Это независимые параметры.

### `heat_damage`

Используется для приема heat:

```text
effectiveHeatRemoval =
    heat_damage × moduleExchanger
```

### `heat_to_damage`

Используется для расчёта damage:

```text
damageCoefficient =
    heat_to_damage
```

Например, уровень 1:

```text
heat_damage   = 0.80
heat_to_damage = 10
```

Поэтому компонент принимает:

```text
80% × moduleExchanger × mulHeat
```

от доступного передаваемого heat и получает примерно:

```text
1 damage / 10 heat
```

до применения остальных модификаторов.

---

## 10. Damage от heat

`ItemReactorHeatExchanger`:

```java
getDamageCFromHeat(...) {
    return heat_to_damage;
}
```

В `LogicReactor`:

```java
lg.damage =
    (short) (
        (lg.heat / lg.getItem().getDamageCFromHeat(
            (int) lg.heat,
            this.reactor
        ))
        * reactor.getMulDamage(
            lg.getX(),
            lg.getY(),
            lg.getStack()
        )
        - lg.getItem().getAutoRepair(this.reactor)
    );
```

Для Heat Exchanger:

```text
autoRepair = 0
```

поэтому:

```text
damage =
    (intHeat / heat_to_damage)
    × mulDamage
```

Перед вызовом `getDamageCFromHeat()` `lg.heat` приводится к `int`.

Результат присваивается в `short`.

---

## 11. Специальных множителей для Heat Exchanger нет

В `LogicReactor` есть специальные правила:

```text
COOLANT_ROD → damage × 10
ROD → PLATE → heat × 1.5
CAPACITOR → damage × 3
```

Для `HEAT_EXCHANGER` отдельного множителя нет.

Поэтому его damage не получает дополнительного коэффициента внутри этих правил.

---

## 12. Передача накопленного heat дальше

Поскольку:

```text
caneExtractHeat() = true
```

накопленное:

```text
LogicComponent.heat
```

может стать исходным heat для последующих соседей.

Например:

```text
ROD
 ↓
HEAT_EXCHANGER
 ↓
VENT
```

Сначала Heat Exchanger получает:

```text
heat_exchanger.heat +=
    rodHeat / col
    × exchangerHeatRemoval
    × mulHeat
```

Затем его текущее поле `heat` используется при обработке следующего узла.

---

## 13. Взаимодействие Rod → Heat Exchanger

Для:

```text
ROD → HEAT_EXCHANGER
```

используется только общая формула:

```text
exchangerHeat +=
    rodHeat / col
    × heat_damage
    × moduleExchanger
    × mulHeat
```

Специального `×1.5` нет.

Правило:

```java
if (source == ROD && target == PLATE)
```

касается только `PLATE`.

---

## 14. Взаимодействие Heat Exchanger → Heat Exchanger

Если первый обменник передает heat второму:

```text
HE1 → HE2
```

то второй получает:

```text
HE2.heat +=
    HE1.heat / col
    × HE2.heat_damage
    × moduleExchanger
    × mulHeat
```

То есть коэффициент `heat_damage` берётся именно у **получателя**, а не источника.

В архитектуре это означает:

```text
source:
    предоставляет heat

target:
    определяет heatRemovePercent
```

---

## 15. Взаимодействие Heat Exchanger → Plate

При:

```text
HEAT_EXCHANGER → PLATE
```

источник предоставляет:

```text
sourceHeat / col
```

пластина применяет:

```text
plate.percent
× reactor.getMulHeat(...)
```

Поскольку источник не является `ROD`, специальный множитель `×1.5` для Plate не применяется.

---

## 16. Взаимодействие Heat Exchanger → Capacitor

При:

```text
HEAT_EXCHANGER → CAPACITOR
```

capacitor получает heat через:

```text
heatRemovePercent = 1
```

после чего его собственная логика рассчитывает damage.

Для capacitor дополнительное правило:

```text
damage × 3
```

применяется уже в `LogicReactor`.

Это множитель получателя, а не Heat Exchanger.

---

## 17. Взаимодействие Heat Exchanger → Coolant Rod

Получатель `COOLANT_ROD` использует свой `getHeatRemovePercent()`.

После расчёта damage в `LogicReactor` применяется:

```java
if (lg.getItem().getType() == EnumTypeComponent.COOLANT_ROD) {
    lg.damage *= 10;
}
```

Следовательно `×10` относится к damage Coolant Rod, а не к Heat Exchanger.

---

## 18. Durability

При создании `LogicComponent`:

```java
maxDamage =
    damageItem.getMaxCustomDamage(stack);

maxDamageItem =
    maxDamage
    - damageItem.getCustomDamage(stack);
```

Для нового компонента:

| ID | Max Damage |
|---|---:|
| `heat_exchange` | 2500 |
| `adv_heat_exchange` | 5000 |
| `imp_heat_exchange` | 7500 |
| `per_heat_exchange` | 10000 |

---

## 19. Изменение durability

Heat Exchanger попадает в стандартную ветку `LogicComponent.onTick()`:

```java
if (damage != 0) {
    if (!componentVent) {
        if (this.maxDamageItem < this.maxDamage) {
            this.item.damageItem(this.stack, -1 * damage);
        }

        this.maxDamageItem -= damage;

        if (this.maxDamageItem > this.maxDamage) {
            this.maxDamageItem = this.maxDamage;
        }
    }
}
```

Для Heat Exchanger:

```text
componentVent = false
```

Поэтому концептуально:

```text
durability -= damage
```

---

## 20. Особенность первого damage

Для нового компонента:

```text
maxDamageItem == maxDamage
```

Поэтому первый раз условие:

```java
maxDamageItem < maxDamage
```

ложно.

В результате:

```text
item.damageItem(...)
```

при первом damage не вызывается.

Но:

```text
maxDamageItem -= damage
```

выполняется.

Это особенность общей реализации `LogicComponent`.

---

## 21. Удаление

`LogicComponent.onTick()` возвращает:

```java
return maxDamageItem <= 0;
```

`LogicReactor` после этого удаляет компонент:

```java
reactor.setItemAt(x, y);
```

Последовательность:

```text
heat
 ↓
damage
 ↓
durability -= damage
 ↓
durability <= 0
 ↓
onTick() = true
 ↓
компонент удаляется
```

---

## 22. Auto Repair

```text
getAutoRepair() = 0
```

Поэтому damage не уменьшается автоматически:

```text
damage =
    heat / heat_to_damage
    × mulDamage
```

без вычитания repair.

---

## 23. Repair Other

```text
getRepairOther() = 0
```

Heat Exchanger не ремонтирует соседние компоненты.

---

## 24. Energy

```text
getEnergyProduction() = 0
```

Heat Exchanger не увеличивает generation.

---

## 25. Radiation

`ItemReactorHeatExchanger` не переопределяет `getRadiation()`.

В `IReactorItem` используется default:

```java
default double getRadiation() {
    return 0;
}
```

Следовательно:

```text
Heat Exchanger radiation = 0
```

---

## 26. `reactor.getModuleExchanger()`

`getModuleExchanger()` не является собственным параметром конкретного типа реактора.

В `IAdvReactor` определён метод:

```java
double getModuleExchanger();
```

Его значение формируется системой `InventoryReactorModules`.

`InventoryReactorModules` содержит четыре слота модулей и при загрузке/изменении содержимого начинает расчёт с:

```text
stableHeat   = 1
radiation    = 1
generation   = 1
vent         = 1
componentVent = 1
exchanger    = 1
capacitor    = 1
```

Для каждого установленного `IReactorModule` значение `exchanger` последовательно умножается:

```java
this.exchanger *= module.getExchanger(stack);
```

Итоговая формула:

```text
moduleExchanger =
    Π(exchangerMultiplier каждого установленного модуля)
```

Для пустых слотов:

```text
moduleExchanger = 1.0
```

Источник:

- `src/main/java/com/denfop/api/reactors/IAdvReactor.java`
- `src/main/java/com/denfop/api/reactors/InventoryReactorModules.java`
- `src/main/java/com/denfop/api/reactors/IReactorModule.java`
- `src/main/java/com/denfop/items/modules/ItemReactorModules.java`

### Модули exchanger

В `ItemReactorModules.CraftingTypes` зарегистрированы четыре модуля, изменяющих `moduleExchanger`:

| Модуль | `getExchanger()` | Эффект |
|---|---:|---:|
| `exchanger0` | 1.05 | +5% |
| `exchanger1` | 1.10 | +10% |
| `exchanger2` | 1.20 | +20% |
| `exchanger3` | 1.30 | +30% |

Другие значения этих же модулей:

| Модуль | Stable Heat | Radiation | Generation | Component Vent | Vent | Exchanger | Capacitor |
|---|---:|---:|---:|---:|---:|---:|---:|
| `exchanger0` | 0.98 | 1.00 | 1.00 | 1.10 | 0.90 | 1.05 | 1.00 |
| `exchanger1` | 0.95 | 1.00 | 1.00 | 1.25 | 0.875 | 1.10 | 1.00 |
| `exchanger2` | 0.925 | 1.00 | 1.00 | 1.25 | 0.85 | 1.20 | 1.00 |
| `exchanger3` | 0.90 | 1.00 | 1.00 | 1.25 | 0.825 | 1.30 | 1.00 |

### Перемножение модулей

Модули не складываются и не выбирается только лучший.

Они перемножаются.

Например:

```text
exchanger0 + exchanger0
=
1.05 × 1.05
=
1.1025
```

И:

```text
exchanger1 + exchanger3
=
1.10 × 1.30
=
1.43
```

При четырёх `exchanger3`:

```text
1.30⁴ = 2.8561
```

если все четыре слота заняты этими модулями.

### Влияние на Heat Exchanger

Сам `ItemReactorHeatExchanger` возвращает:

```java
getHeatRemovePercent(reactor) =
    heat_damage * reactor.getModuleExchanger();
```

Поэтому:

```text
effectiveHeatRemoval =
    heat_damage × moduleExchanger
```

Например, для `heat_exchange`:

```text
heat_damage = 0.80
```

Без модулей:

```text
0.80 × 1.00 = 0.80
```

С `exchanger0`:

```text
0.80 × 1.05 = 0.84
```

С `exchanger1`:

```text
0.80 × 1.10 = 0.88
```

С `exchanger2`:

```text
0.80 × 1.20 = 0.96
```

С `exchanger3`:

```text
0.80 × 1.30 = 1.04
```

Таким образом, effective heat removal может быть больше `1.0`.

### Значение без модулей

Для всех 16 поддерживаемых реакторов при отсутствии Reactor Modules:

```text
moduleExchanger = 1.0
```

Сам тип и уровень реактора не меняют это значение.

Поэтому `moduleExchanger` не следует хранить в `ReactorDefinition`.

Его следует рассчитывать из установленного набора модулей.

Рекомендуемая модель:

```ts
interface ReactorModules {
    exchanger: number;
}

function calculateExchangerModifier(
    modules: ReactorModule[],
): number {
    return modules.reduce(
        (result, module) => result * module.exchanger,
        1,
    );
}


Для simulation context необходимо хранить `mulHeat` отдельно:

```ts
interface ReactorContext {
    modules: ReactorModules;
    mulHeat: number;
}
```

Для Fluid, Gas и Heat:

```ts
context.mulHeat = 1;
```

Для Graphite:

```ts
context.mulHeat =
    exchangersForColumn.reduce(
        (result, exchanger) => result * exchanger.columnMultiplier,
        1,
    );
```

Итоговый коэффициент:

```text
effective heat coefficient =
    heatDamage
    × modules.exchanger
    × context.mulHeat
```
```

---


## 27. Конкретные значения `reactor.getMulHeat()`

### 27.1. Базовая реализация

В `IAdvReactor` `getMulHeat(...)` имеет значение по умолчанию:

```java
default double getMulHeat(
    final int x,
    final int y,
    ItemStack stack
) {
    return 1;
}
```

Поэтому для реакторов, которые не переопределяют этот метод:

```text
mulHeat = 1.0
```

В baseline это:

| Семейство | Реакторы | `getMulHeat()` |
|---|---|---:|
| Fluid | FS, FA, FI, FP | 1.0 |
| Gas | GS, GA, GI, GP | 1.0 |
| Heat | HS, HA, HI, HP | 1.0 |
| Graphite | GRS, GRA, GRI, GRP | зависит от exchanger-блоков |

---

### 27.2. Graphite Reactor

Графитовый реактор переопределяет:

```java
@Override
public double getMulHeat(
    final int x,
    final int y,
    final ItemStack stack
) {
    double coef = 1;

    for (IExchanger exchanger : this.listExchanger) {
        coef *= exchanger.getPercent(x);
    }

    return coef;
}
```

Следовательно:

```text
mulHeat(x) =
    Π(exchanger.getPercent(x))
```

Параметры:

```text
x = колонка реактора
y = не используется
stack = не используется
```

---

### 27.3. Что такое `listExchanger`

Графитовый контроллер собирает все multiblock-элементы, реализующие:

```text
IExchanger
```

После этого они попадают в:

```text
listExchanger
```

Каждый exchanger может быть назначен на определённую колонку `x`.

---

### 27.4. Как рассчитывается `TileEntityExchanger.percent`

У `TileEntityExchanger`:

```text
percent = 1
```

при пустом слоте.

При установке `IExchangerItem`:

```java
percent =
    1 - item.getPercent();
```

Метод:

```java
getPercent(int x)
```

возвращает:

```text
1
```

если:

- exchanger не подключён к главному контроллеру;
- передана другая колонка;
- слот exchanger пуст.

Иначе возвращается рассчитанный `percent`.

То есть конкретный exchanger влияет только на назначенную ему колонку.

---

### 27.5. Item Exchanger

В baseline доступны четыре Item Exchanger:

| Item | `item.getPercent()` | `TileEntityExchanger.percent` | Влияние на `mulHeat` |
|---|---:|---:|---:|
| `simple_exchanger` | 0.05 | 0.95 | ×0.95 |
| `adv_exchanger` | 0.10 | 0.90 | ×0.90 |
| `imp_exchanger` | 0.15 | 0.85 | ×0.85 |
| `per_exchanger` | 0.20 | 0.80 | ×0.80 |

Ключевой момент:

```text
item.getPercent()
```

и:

```text
TileEntityExchanger.percent
```

— разные величины.

Например:

```text
simple_exchanger:
    item percent = 0.05
    block percent = 1 - 0.05 = 0.95
```

Именно `0.95` используется в `getMulHeat()`.

---

### 27.6. Один exchanger

Для одной колонки:

| Установленный exchanger | `mulHeat` |
|---|---:|
| Нет | 1.0000 |
| `simple_exchanger` | 0.9500 |
| `adv_exchanger` | 0.9000 |
| `imp_exchanger` | 0.8500 |
| `per_exchanger` | 0.8000 |

---

### 27.7. Несколько exchanger на одной колонке

Модификаторы перемножаются.

Например:

```text
simple + simple
=
0.95 × 0.95
=
0.9025
```

```text
simple + advanced
=
0.95 × 0.90
=
0.855
```

```text
advanced + perfect
=
0.90 × 0.80
=
0.72
```

Четыре `per_exchanger`:

```text
0.80⁴ = 0.4096
```

---

### 27.8. Влияние на Heat Exchanger

Полная формула для Heat Exchanger:

```text
receivedHeat =
    sourceHeat / col
    × heat_damage
    × moduleExchanger
    × mulHeat
```

Для Fluid/Gas/Heat:

```text
mulHeat = 1.0
```

Для Graphite:

```text
mulHeat =
    Π(exchanger.percent текущей колонки)
```

Поэтому, например, для `heat_exchange` первого уровня:

```text
heat_damage = 0.80
moduleExchanger = 1.00
mulHeat = 0.95
```

получаем:

```text
effective heat coefficient =
    0.80 × 1.00 × 0.95
    =
    0.76
```

При:

```text
sourceHeat = 100
col = 1
```

получим:

```text
receivedHeat = 100 × 0.76 = 76
```

---

### 27.9. Важное разделение модификаторов

Нельзя объединять:

```text
moduleExchanger
```

и:

```text
mulHeat
```

Это два независимых механизма.

```text
Reactor Modules
    ↓
moduleExchanger
    ↓
ItemReactorHeatExchanger.getHeatRemovePercent()
```

и:

```text
Graphite Reactor Exchanger blocks
    ↓
getMulHeat(x, y, stack)
    ↓
LogicReactor
```

Итог:

```text
effectiveHeatRemoval =
    heat_damage
    × moduleExchanger
    × mulHeat
```

---

## 36. Роль `col`

`LogicReactor` считает:

```java
int col = 0;

for (LogicComponent lg : component.getLogicComponents()) {
    if (!list.contains(lg)) {
        col++;
    }
}
```

и затем:

```text
component.heat / col
```

Поэтому Heat Exchanger получает не фиксированную долю по числу физических соседей.

Доля зависит от текущего состояния обхода графа `LogicReactor`.

Это нужно сохранить в simulation engine.

---

## 36. TypeScript-модель

```ts
interface ReactorHeatExchangerDefinition {
    id: string;
    type: 'HEAT_EXCHANGER';

    level: number;
    maxDamage: number;

    heatToDamage: number;
    heatDamage: number;
}
```

Данные:

```ts
const heatExchangers = [
    {
        id: 'heat_exchange',
        type: 'HEAT_EXCHANGER',
        level: 1,
        maxDamage: 2500,
        heatToDamage: 10,
        heatDamage: 0.80,
    },
    {
        id: 'adv_heat_exchange',
        type: 'HEAT_EXCHANGER',
        level: 2,
        maxDamage: 5000,
        heatToDamage: 12,
        heatDamage: 0.75,
    },
    {
        id: 'imp_heat_exchange',
        type: 'HEAT_EXCHANGER',
        level: 3,
        maxDamage: 7500,
        heatToDamage: 15,
        heatDamage: 0.60,
    },
    {
        id: 'per_heat_exchange',
        type: 'HEAT_EXCHANGER',
        level: 4,
        maxDamage: 10000,
        heatToDamage: 20,
        heatDamage: 0.45,
    },
];
```

---

## 36. Логика компонента в TypeScript

Концептуальная модель:

```ts
getHeat() {
    return 0;
}

getHeatRemovePercent(context: ReactorContext) {
    return this.definition.heatDamage *
        context.modules.exchanger;
}

getDamageCFromHeat() {
    return this.definition.heatToDamage;
}

getEnergyProduction() {
    return 0;
}

getAutoRepair() {
    return 0;
}

getRepairOther() {
    return 0;
}

canExtractHeat() {
    return true;
}

isTickable() {
    return true;
}
```

---

## 36. Отличие от Plate

| Свойство | Plate | Heat Exchanger |
|---|---:|---:|
| Принимает heat | Да | Да |
| Передаёт heat | Нет | Да |
| Собственный heat | 0 | 0 |
| Heat removal | `percent` | `heat_damage × moduleExchanger` |
| Rod → target `×1.5` | Да | Нет |
| Имеет durability | Нет | Да |
| `updatableItem()` | `false` | `true` |
| Energy | 0 | 0 |
| Radiation | 0 | 0 |

---

## 36. Главная последовательность расчёта

Для Heat Exchanger:

```text
создание
    ↓
heat = 0
    ↓
LogicReactor передаёт heat
    ↓
heat += sourceHeat / col
       × heat_damage
       × moduleExchanger
       × mulHeat
    ↓
damage =
    trunc(intHeat / heat_to_damage)
    × mulDamage
    - autoRepair
    ↓
специального множителя нет
    ↓
LogicComponent.onTick()
    ↓
durability -= damage
    ↓
при zero durability компонент удаляется
```

---

## 36. Пример расчёта

Вход:

```text
sourceHeat = 100
col = 1
modules = []
moduleExchanger = 1.0
mulHeat = 1
mulDamage = 1

heat_damage = 0.80
heat_to_damage = 10
```

Получение:

```text
heat =
    100 / 1
    × 0.80
    × 1
    × 1

heat = 80
```

Damage:

```text
damage =
    80 / 10
    × 1

damage = 8
```

После tick:

```text
durability -= 8
```

Для нового компонента на первом damage внутренняя `maxDamageItem` уменьшится на `8`, но `damageItem()` не вызывается из-за условия `maxDamageItem < maxDamage`.

---

## 36. Unit Tests

### Данные

- [ ] `heat_exchange`.
- [ ] `adv_heat_exchange`.
- [ ] `imp_heat_exchange`.
- [ ] `per_heat_exchange`.

### Базовые свойства

- [ ] `type === HEAT_EXCHANGER`.
- [ ] Проверить level.
- [ ] Проверить maxDamage.
- [ ] `getHeat() === 0`.
- [ ] `getEnergyProduction() === 0`.
- [ ] `getRadiation() === 0`.
- [ ] `getAutoRepair() === 0`.
- [ ] `getRepairOther() === 0`.
- [ ] `updatableItem() === true`.
- [ ] `caneExtractHeat() === true`.

### Heat

- [ ] Проверить `heatDamage`.
- [ ] Проверить `moduleExchanger`.
- [ ] Проверить `mulHeat`.
- [ ] Проверить `col`.
- [ ] Проверить передачу heat дальше.

### Damage

- [ ] Проверить `heatToDamage`.
- [ ] Проверить `mulDamage`.
- [ ] Проверить `autoRepair = 0`.
- [ ] Проверить отсутствие специальных множителей.

### Durability

- [ ] Проверить initial durability.
- [ ] Проверить первый damage.
- [ ] Проверить следующие damage.
- [ ] Проверить уменьшение `maxDamageItem`.
- [ ] Проверить удаление при zero durability.

---

## 36. Golden Test

Минимальная схема:

```text
ROD → HEAT_EXCHANGER → VENT
```

Вход:

```text
sourceHeat = 100
col = 1
modules = []
moduleExchanger = 1.0
mulHeat = 1
mulDamage = 1
```

Для `heat_exchange`:

```text
heatDamage = 0.80
heatToDamage = 10
```

Ожидается:

```text
heat = 80
damage = 8
```

После `onTick()`:

```text
durability уменьшается на 8
```

Но для первого damage необходимо отдельно проверить особенность `LogicComponent`:

```text
maxDamageItem -= 8
item.damageItem(...) не вызывается
```

---

## 36. Тесты `moduleExchanger`

Проверить расчёт итогового `moduleExchanger`:

- [ ] без модулей → `1.0`;
- [ ] один `exchanger0` → `1.05`;
- [ ] один `exchanger1` → `1.10`;
- [ ] один `exchanger2` → `1.20`;
- [ ] один `exchanger3` → `1.30`;
- [ ] два `exchanger0` → `1.1025`;
- [ ] `exchanger1 + exchanger3` → `1.43`;
- [ ] четыре `exchanger3` → `2.8561`.

Проверить влияние на первый Heat Exchanger:

```text
heat_damage = 0.80
```

Ожидается:

```text
без модулей:
0.80

exchanger0:
0.84

exchanger1:
0.88

exchanger2:
0.96

exchanger3:
1.04
```

---


## 36. Тесты `getMulHeat()`

### Базовые реакторы

- [ ] Fluid → `mulHeat = 1.0`.
- [ ] Gas → `mulHeat = 1.0`.
- [ ] Heat → `mulHeat = 1.0`.

### Graphite Reactor

- [ ] Без exchanger-блоков → `1.0`.
- [ ] `simple_exchanger` → `0.95`.
- [ ] `adv_exchanger` → `0.90`.
- [ ] `imp_exchanger` → `0.85`.
- [ ] `per_exchanger` → `0.80`.
- [ ] `simple + simple` → `0.9025`.
- [ ] `simple + advanced` → `0.855`.
- [ ] `advanced + perfect` → `0.72`.
- [ ] четыре `per_exchanger` → `0.4096`.

### Привязка к колонке

- [ ] Exchanger влияет только на назначенную колонку `x`.
- [ ] Для другой колонки `getPercent(x)` возвращает `1`.
- [ ] Пустой слот возвращает `1`.
- [ ] Exchanger без главного контроллера возвращает `1`.

### Интеграция с Heat Exchanger

Для:

```text
heat_damage = 0.80
moduleExchanger = 1.0
mulHeat = 0.95
sourceHeat = 100
col = 1
```

ожидается:

```text
receivedHeat = 76
```

Проверить отдельно:

```text
moduleExchanger
```

и:

```text
mulHeat
```

чтобы они не смешивались в один параметр.

---



## 36. Конкретные значения `reactor.getMulDamage()`

### 36.1. Базовая реализация

В `IAdvReactor` метод имеет default-реализацию:

```java
default double getMulDamage(
    final int x,
    final int y,
    ItemStack stack
) {
    return 1;
}
```

В baseline commit отдельного override `getMulDamage()` у реакторов не обнаружено.

Поэтому для всех поддерживаемых реакторов:

```text
getMulDamage(...) = 1.0
```

Параметры:

```text
x
y
stack
```

в default-реализации не используются.

---

### 36.2. Все 16 реакторов

| Семейство | Реакторы | `getMulDamage()` |
|---|---|---:|
| Fluid | FS, FA, FI, FP | 1.0 |
| Gas | GS, GA, GI, GP | 1.0 |
| Graphite | GRS, GRA, GRI, GRP | 1.0 |
| Heat | HS, HA, HI, HP | 1.0 |

Таким образом, `getMulDamage()` не зависит от:

- типа реактора;
- уровня реактора;
- координат `x/y`;
- конкретного компонента;
- установленных Reactor Modules;
- Graphite Reactor Exchanger.

---

### 36.3. Использование в `LogicReactor`

`LogicReactor` рассчитывает damage:

```java
lg.damage =
    (short) (
        (lg.heat / lg.getItem().getDamageCFromHeat(
            (int) lg.heat,
            this.reactor
        ))
        * reactor.getMulDamage(
            lg.getX(),
            lg.getY(),
            lg.getStack()
        )
        - lg.getItem().getAutoRepair(this.reactor)
    );
```

Так как:

```text
getMulDamage(...) = 1.0
```

в baseline:

```text
damage =
    intHeat / damageCoefficient
    - autoRepair
```

до применения специальных правил `LogicReactor`.

---

### 36.4. Специальные damage-множители

Следующие множители существуют в `LogicReactor`, но не являются частью `getMulDamage()`:

```text
COOLANT_ROD → damage × 10
CAPACITOR   → damage × 3
```

Для `HEAT_EXCHANGER` отдельного множителя нет.

Поэтому для Heat Exchanger:

```text
damage =
    intHeat / heat_to_damage
```

поскольку:

```text
getMulDamage = 1.0
autoRepair   = 0
```

---

### 36.5. Важное разделение

Нельзя считать:

```text
getMulDamage() = 1
```

эквивалентом:

```text
отсутствуют все damage-модификаторы
```

Правильная последовательность:

```text
1. intHeat
        ↓
2. / getDamageCFromHeat()
        ↓
3. × getMulDamage()
        ↓
4. - getAutoRepair()
        ↓
5. специальные правила LogicReactor
   ├── COOLANT_ROD ×10
   └── CAPACITOR ×3
```

Для Heat Exchanger после шага 4 дополнительных модификаторов нет.

---

### 36.6. Влияние на Heat Exchanger

Для всех четырёх Heat Exchanger:

| Компонент | `heat_to_damage` | `getMulDamage()` | `autoRepair` |
|---|---:|---:|---:|
| `heat_exchange` | 10 | 1.0 | 0 |
| `adv_heat_exchange` | 12 | 1.0 | 0 |
| `imp_heat_exchange` | 15 | 1.0 | 0 |
| `per_heat_exchange` | 20 | 1.0 | 0 |

Следовательно:

```text
heat_exchange:
    damage = intHeat / 10

adv_heat_exchange:
    damage = intHeat / 12

imp_heat_exchange:
    damage = intHeat / 15

per_heat_exchange:
    damage = intHeat / 20
```

с учётом фактического приведения `lg.heat` к `int` перед вызовом `getDamageCFromHeat()` и приведения итогового damage к `short`.

---

### 36.7. Архитектура TypeScript

Метод стоит сохранить в интерфейсе движка, даже если baseline всегда возвращает `1`:

```ts
interface ReactorContext {
    getMulDamage(
        x: number,
        y: number,
        component: ComponentState,
    ): number;
}
```

Baseline:

```ts
getMulDamage(): number {
    return 1;
}
```

Это сохраняет соответствие API Industrial Upgrade и позволяет без изменения архитектуры поддержать будущий override в другой версии мода.

---

### 36.8. Тесты

- [ ] Fluid → `getMulDamage() === 1.0`.
- [ ] Gas → `getMulDamage() === 1.0`.
- [ ] Graphite → `getMulDamage() === 1.0`.
- [ ] Heat → `getMulDamage() === 1.0`.
- [ ] Проверить разные `x`.
- [ ] Проверить разные `y`.
- [ ] Проверить разные компоненты.
- [ ] Проверить, что Reactor Modules не изменяют `getMulDamage()`.
- [ ] Проверить, что Graphite Exchanger не изменяет `getMulDamage()`.
- [ ] Проверить `COOLANT_ROD ×10` как отдельное правило.
- [ ] Проверить `CAPACITOR ×3` как отдельное правило.
- [ ] Проверить отсутствие дополнительного множителя для Heat Exchanger.

---

## 37. Статус исследования

- [x] Полный `ItemReactorHeatExchanger`.
- [x] Все четыре варианта.
- [x] Level.
- [x] Max damage.
- [x] `heat_to_damage`.
- [x] `heat_damage`.
- [x] `getHeat()`.
- [x] `getHeatRemovePercent()`.
- [x] `getDamageCFromHeat()`.
- [x] `getAutoRepair()`.
- [x] `getRepairOther()`.
- [x] `getEnergyProduction()`.
- [x] `damageItem()`.
- [x] `updatableItem()`.
- [x] `caneExtractHeat()`.
- [x] Получение heat через `LogicReactor`.
- [x] Передача heat дальше.
- [x] Формула damage.
- [x] Durability через `LogicComponent`.
- [x] Удаление компонента.
- [x] Rod → Heat Exchanger.
- [x] Heat Exchanger → Heat Exchanger.
- [x] Heat Exchanger → Plate.
- [x] Heat Exchanger → Capacitor.
- [x] Heat Exchanger → Coolant Rod.
- [x] Особенность `col`.
- [x] Система `InventoryReactorModules` для `moduleExchanger`.
- [x] Конкретные значения `reactor.getMulHeat()`.
- [x] Конкретные значения `reactor.getMulDamage()`.

`moduleExchanger` исследован полностью на уровне системы Reactor Modules.

Все reactor-specific modifiers, исследованные на текущем этапе:

- `moduleExchanger` — исследован;
- `getMulHeat(...)` — исследован;
- `getMulDamage(...)` — исследован.

Следующий этап: исследование остальных взаимодействий компонентов и полного порядка изменения heat/durability в `LogicReactor` и `LogicComponent`.
