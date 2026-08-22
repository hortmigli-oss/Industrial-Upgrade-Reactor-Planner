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

Итог:

```text
receivedHeat =
    sourceHeat / col
    × heat_damage
    × moduleExchanger
    × mulHeat
```

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

Heat Exchanger зависит от reactor-specific modifier:

```java
getHeatRemovePercent() {
    return heat_damage * reactor.getModuleExchanger();
}
```

Следовательно:

```text
effectiveRemoval =
    heat_damage × moduleExchanger
```

Конкретные значения `moduleExchanger` для каждого типа реактора исследуются отдельно в разделе `reactor-specific modifiers`.

---

## 27. Роль `col`

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

## 28. TypeScript-модель

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

## 29. Логика компонента в TypeScript

Концептуальная модель:

```ts
getHeat() {
    return 0;
}

getHeatRemovePercent(context: ReactorContext) {
    return this.definition.heatDamage *
        context.moduleExchanger;
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

## 30. Отличие от Plate

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

## 31. Главная последовательность расчёта

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

## 32. Пример расчёта

Вход:

```text
sourceHeat = 100
col = 1
moduleExchanger = 1
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

## 33. Unit Tests

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

## 34. Golden Test

Минимальная схема:

```text
ROD → HEAT_EXCHANGER → VENT
```

Вход:

```text
sourceHeat = 100
col = 1
moduleExchanger = 1
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

## 35. Статус исследования

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
- [ ] Конкретные значения `reactor.getModuleExchanger()`.
- [ ] Конкретные значения `reactor.getMulHeat()`.
- [ ] Конкретные значения `reactor.getMulDamage()`.

Внешние зависимости относятся к отдельному этапу исследования `reactor-specific modifiers`.
