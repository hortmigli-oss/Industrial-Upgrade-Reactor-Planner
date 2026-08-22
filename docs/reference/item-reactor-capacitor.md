# Исследование ItemReactorCapacitor

## 1. Источник

Industrial Upgrade:

- Repository: https://github.com/ZelGimi/industrialupgrade
- Commit: `16b7fb9cfb7fbb3171e35a63532ebc0a112f665c`
- Minecraft: `1.12.2`
- Forge: `14.23.5.2860`

Основной источник:

`src/main/java/com/denfop/items/reactors/ItemReactorCapacitor.java`

Связанная логика:

- `src/main/java/com/denfop/api/reactors/IReactorItem.java`
- `src/main/java/com/denfop/api/reactors/IAdvReactor.java`
- `src/main/java/com/denfop/api/reactors/LogicComponent.java`
- `src/main/java/com/denfop/api/reactors/LogicReactor.java`
- `src/main/java/com/denfop/IUItem.java`

---

## 2. Назначение

`ItemReactorCapacitor` — компонент типа `CAPACITOR`, который:

- получает тепло от соседних компонентов;
- сам тепло не выделяет;
- не передаёт тепло дальше;
- получает damage от накопленного тепла;
- имеет durability;
- не ремонтирует себя;
- не ремонтирует другие компоненты;
- не производит энергию.

---

## 3. Структура класса

Класс:

```java
public class ItemReactorCapacitor extends ItemDamage implements IReactorItem
```

Собственные поля:

```java
private final int level;
private final int heat_to_damage;
```

Максимальная durability передаётся в `ItemDamage`.

Конструктор:

```java
public ItemReactorCapacitor(
    final String name,
    final int maxDamage,
    int level,
    int heat_to_damage
)
```

При создании устанавливается:

```java
setMaxStackSize(1);
```

Компонент нельзя складывать в стак больше одного предмета.

---

## 4. Все варианты capacitor

В `IUItem.java` зарегистрированы четыре варианта:

| ID | Max Damage | Level | Heat-to-Damage |
|---|---:|---:|---:|
| `capacitor` | 25000 | 1 | 4 |
| `adv_capacitor` | 50000 | 2 | 6 |
| `imp_capacitor` | 100000 | 3 | 8 |
| `per_capacitor` | 200000 | 4 | 12 |

Регистрация:

```java
IUItem.capacitor =
    new ItemReactorCapacitor("capacitor", 25000, 1, 4);

IUItem.adv_capacitor =
    new ItemReactorCapacitor("adv_capacitor", 50000, 2, 6);

IUItem.imp_capacitor =
    new ItemReactorCapacitor("imp_capacitor", 100000, 3, 8);

IUItem.per_capacitor =
    new ItemReactorCapacitor("per_capacitor", 200000, 4, 12);
```

---

## 5. Реализация IReactorItem

### 5.1. Тип

```java
getType() = EnumTypeComponent.CAPACITOR
```

---

### 5.2. Уровень

```java
getLevel() = level
```

Значения:

```text
capacitor      = 1
adv_capacitor  = 2
imp_capacitor  = 3
per_capacitor  = 4
```

---

### 5.3. Heat

```java
getHeat() = 0
```

Сам capacitor не создаёт собственного тепла при создании `LogicComponent`.

Но это **не означает**, что поле `LogicComponent.heat` всегда равно нулю после расчёта.

`LogicReactor` может передать capacitor тепло от соседнего компонента:

```java
lg.heat +=
    ((component.heat / col)
    * lg.getItem().getHeatRemovePercent(this.reactor))
    * reactor.getMulHeat(...);
```

Поэтому необходимо различать:

```text
базовое heat компонента = 0
текущее heat состояния    = может быть > 0
```

---

## 5.4. Energy Production

```java
getEnergyProduction() = 0
```

Capacitor не является генератором.

---

## 5.5. Heat Remove Percent

```java
getHeatRemovePercent() = 1
```

Это означает, что если capacitor получает тепло через механизм `LogicReactor`, его коэффициент приёма тепла равен `1`.

Следовательно, до применения reactor-specific `getMulHeat(...)` используется:

```text
receivedHeat = component.heat / col
```

---

## 5.6. Can Extract Heat

```java
caneExtractHeat() = false
```

Capacitor может получать тепло, но не может использовать себя как источник теплопередачи соседям.

Это принципиально отличается, например, от:

```text
Vent
Heat Exchanger
Coolant Rod
```

---

## 5.7. Auto Repair

```java
getAutoRepair() = 0
```

Capacitor не ремонтирует себя автоматически.

---

## 5.8. Repair Other

```java
getRepairOther() = 0
```

Capacitor не ремонтирует соседние компоненты.

---

## 5.9. Damage From Heat

Главная формула класса:

```java
return (int) (heat_to_damage * reactor.getModuleCapacitor());
```

То есть:

```text
damageCoefficient =
    heat_to_damage × reactor.getModuleCapacitor()
```

Здесь уже присутствует reactor-specific modifier:

```text
reactor.getModuleCapacitor()
```

Его конкретные значения исследуются отдельно в разделе reactor-specific modifiers.

---

## 5.10. needClear

```java
return this.getMaxCustomDamage(stack)
    - this.getCustomDamage(stack) == 0;
```

Компонент считается полностью израсходованным, когда текущая durability равна нулю.

---

## 5.11. damageItem

```java
damageItem(stack, damage) {
    applyCustomDamage(stack, damage, null);
}
```

В отличие от `ItemEnergyCoupler`, здесь переданное значение damage используется напрямую.

---

## 5.12. updatableItem

```java
updatableItem() = true
```

Capacitor попадает в список компонентов, обрабатываемых на tick.

---

# 6. Полная модель поведения в LogicReactor

Важно: часть физики capacitor находится не в `ItemReactorCapacitor`, а в `LogicReactor`.

Когда `LogicReactor` обнаруживает соседний компонент, способный передавать тепло, выполняется:

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

Для capacitor:

```text
getHeatRemovePercent() = 1
```

Поэтому:

```text
capacitor.heat +=
    (sourceComponent.heat / numberOfConnections)
    × reactor.getMulHeat(...)
```

---

# 7. Формула damage capacitor

После получения тепла `LogicReactor` вычисляет:

```java
lg.damage =
    (short) (
        (lg.heat
        / lg.getItem().getDamageCFromHeat(
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

Для capacitor:

```text
getDamageCFromHeat(heat)
    =
    floor(heat_to_damage × moduleCapacitor)
```

а:

```text
getAutoRepair() = 0
```

После этого `LogicReactor` выполняет отдельное правило:

```java
if (lg.getItem().getType() == EnumTypeComponent.CAPACITOR) {
    lg.damage *= 3;
}
```

Итоговая модель:

```text
coefficient =
    floor(heat_to_damage × moduleCapacitor)

baseDamage =
    floor(currentHeat / coefficient)
    × reactor.getMulDamage(x, y, stack)

damage =
    baseDamage × 3
```

Важно:

1. `getDamageCFromHeat()` сначала приводит результат к `int`.
2. Затем `currentHeat` приводится к `int` перед передачей в `getDamageCFromHeat()`.
3. Результирующий `damage` хранится в `short`.
4. Для capacitor применяется дополнительный `×3`.

Порядок этих операций необходимо сохранить в TypeScript максимально близко к оригиналу.

---

# 8. Почему capacitor получает heat, несмотря на getHeat() = 0

Есть два разных значения:

```text
ItemReactor.getHeat()
```

и:

```text
LogicComponent.heat
```

При создании:

```java
this.heat = item.getHeat(reactor);
```

Поэтому первоначально:

```text
LogicComponent.heat = 0
```

Но дальше `LogicReactor` непосредственно меняет:

```text
lg.heat
```

через передачу тепла от соседей.

Следовательно:

```text
getHeat() = 0
```

означает только:

> базовое собственное выделение тепла компонента равно нулю.

Это не запрещает накапливать тепло в состоянии компонента.

---

# 9. Capacitor как конечный потребитель тепла

Поскольку:

```text
canExtractHeat() = false
```

capacitor является конечным потребителем тепла в цепочке.

Упрощённо:

```text
Fuel Rod
   ↓
Vent / Exchanger / Plate
   ↓
Capacitor
   ↓
damage
```

Capacitor не может выполнить обратную передачу:

```text
Capacitor
   X
   ↓
сосед
```

---

# 10. Durability

При создании `LogicComponent`:

```java
this.maxDamage =
    damageItem.getMaxCustomDamage(stack);

this.maxDamageItem =
    maxDamage
    - damageItem.getCustomDamage(stack);
```

Для нового capacitor:

```text
maxDamageItem = maxDamage
```

Примеры:

```text
capacitor      = 25000
adv_capacitor  = 50000
imp_capacitor  = 100000
per_capacitor  = 200000
```

---

# 11. Обработка damage в LogicComponent.onTick()

Capacitor не является:

```text
ROD
ENERGY_COUPLER
NEUTRON_PROTECTOR
```

Поэтому используется обычная ветка:

```java
if (damage != 0) {
    if (!componentVent) {
        if (maxDamageItem < maxDamage) {
            item.damageItem(stack, -1 * damage);
        }

        maxDamageItem -= damage;

        if (maxDamageItem > maxDamage) {
            maxDamageItem = maxDamage;
        }
    }
}
```

Для capacitor:

```text
componentVent = false
```

---

# 12. Важная особенность первой потери durability

Для нового компонента:

```text
maxDamageItem == maxDamage
```

Поэтому при первом damage:

```java
if (maxDamageItem < maxDamage)
```

ложно.

Следовательно:

```text
item.damageItem(...) не вызывается
```

но:

```text
maxDamageItem -= damage
```

выполняется.

Таким образом, внутреннее состояние `LogicComponent` может на первом шаге отличаться от durability `ItemStack`.

На следующих damage вызов `damageItem()` происходит.

Это поведение исходного IU необходимо сохранить для точного simulation core.

---

# 13. Уничтожение capacitor

`LogicComponent.onTick()` возвращает:

```java
return maxDamageItem <= 0;
```

Затем `LogicReactor.onTick()` добавляет компонент в список на удаление:

```java
if (tick) {
    logicComponentList.add(component);
}
```

После обработки:

```java
logicComponentList.forEach(
    logicComponent ->
        reactor.setItemAt(
            logicComponent.getX(),
            logicComponent.getY()
        )
);
```

Таким образом:

```text
damage
↓
maxDamageItem <= 0
↓
onTick() = true
↓
LogicReactor помечает компонент
↓
reactor.setItemAt(x, y)
↓
компонент удаляется
```

---

# 14. Параметры четырёх Capacitor

## capacitor

```text
id              = capacitor
type            = CAPACITOR
level           = 1
maxDamage       = 25000
heatToDamage    = 4

getHeat()       = 0
heatRemove      = 1
energy          = 0
autoRepair      = 0
repairOther     = 0
canExtractHeat  = false
updatable       = true
```

## adv_capacitor

```text
id              = adv_capacitor
type            = CAPACITOR
level           = 2
maxDamage       = 50000
heatToDamage    = 6
```

Поведение идентично, кроме level / durability / heatToDamage.

## imp_capacitor

```text
id              = imp_capacitor
type            = CAPACITOR
level           = 3
maxDamage       = 100000
heatToDamage    = 8
```

## per_capacitor

```text
id              = per_capacitor
type            = CAPACITOR
level           = 4
maxDamage       = 200000
heatToDamage    = 12
```

---

# 15. TypeScript-модель

Базовая модель:

```ts
interface ReactorCapacitorDefinition {
    id: string;
    type: 'CAPACITOR';

    level: number;
    maxDamage: number;
    heatToDamage: number;
}
```

Данные:

```ts
const capacitors: ReactorCapacitorDefinition[] = [
    {
        id: 'capacitor',
        type: 'CAPACITOR',
        level: 1,
        maxDamage: 25000,
        heatToDamage: 4,
    },
    {
        id: 'adv_capacitor',
        type: 'CAPACITOR',
        level: 2,
        maxDamage: 50000,
        heatToDamage: 6,
    },
    {
        id: 'imp_capacitor',
        type: 'CAPACITOR',
        level: 3,
        maxDamage: 100000,
        heatToDamage: 8,
    },
    {
        id: 'per_capacitor',
        type: 'CAPACITOR',
        level: 4,
        maxDamage: 200000,
        heatToDamage: 12,
    },
];
```

---

# 16. Поведение TypeScript-компонента

Минимальная модель:

```ts
getHeat() {
    return 0;
}

getHeatRemovePercent() {
    return 1;
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
    return false;
}

isTickable() {
    return true;
}
```

Damage coefficient:

```ts
getDamageFromHeat(context: ReactorContext) {
    return Math.trunc(
        this.definition.heatToDamage *
        context.moduleCapacitor
    );
}
```

---

# 17. Тесты

Минимальный набор unit tests для capacitor:

## Данные

- [ ] Проверить `capacitor`.
- [ ] Проверить `adv_capacitor`.
- [ ] Проверить `imp_capacitor`.
- [ ] Проверить `per_capacitor`.

## Базовое поведение

- [ ] `getType() === CAPACITOR`.
- [ ] `getHeat() === 0`.
- [ ] `getHeatRemovePercent() === 1`.
- [ ] `getEnergyProduction() === 0`.
- [ ] `getAutoRepair() === 0`.
- [ ] `getRepairOther() === 0`.
- [ ] `canExtractHeat() === false`.
- [ ] `updatableItem() === true`.

## Damage

Проверить:

```text
heat_to_damage × moduleCapacitor
```

и:

```text
damage × 3
```

после обработки `LogicReactor`.

## Durability

Проверить:

- [ ] первоначальную durability;
- [ ] первый damage;
- [ ] второй damage;
- [ ] накопление damage;
- [ ] достижение zero durability;
- [ ] удаление компонента.

## Integration

Создать схему:

```text
Rod → Capacitor
```

и проверить:

- [ ] capacitor получает heat;
- [ ] capacitor не передаёт heat дальше;
- [ ] capacitor получает damage;
- [ ] damage соответствует baseline;
- [ ] capacitor удаляется при zero durability.

---

# 18. Что не относится к simulation core

Следующие части `ItemReactorCapacitor` не являются физикой:

```text
getModelLocation()
addInformation()
getItemStackDisplayName()
registerModel()
```

Это Minecraft-specific GUI/rendering.

В TypeScript simulation core они не нужны.

---

# 19. Выявленные зависимости

`ItemReactorCapacitor` напрямую зависит от:

```text
IAdvReactor.getModuleCapacitor()
```

и косвенно от:

```text
IAdvReactor.getMulHeat()
IAdvReactor.getMulDamage()
```

Дополнительный multiplier:

```text
CAPACITOR × 3
```

находится не в `ItemReactorCapacitor`, а в:

```text
LogicReactor.calculateFirstLogic()
```

---

# 20. Итоговая формула поведения

Упрощённо для одного прохода расчёта:

```text
receivedHeat =
    sourceHeat / neighborCount
    × capacitorHeatRemove
    × reactorHeatMultiplier
```

Для capacitor:

```text
capacitorHeatRemove = 1
```

Затем:

```text
damageCoefficient =
    trunc(
        heatToDamage
        × reactor.moduleCapacitor
    )
```

Затем:

```text
baseDamage =
    trunc(currentHeat)
    / damageCoefficient
```

с учётом фактического порядка целочисленных преобразований Java.

После `getMulDamage()`:

```text
damage =
    baseDamage
    × reactorDamageMultiplier
```

После отдельного правила `LogicReactor`:

```text
damage =
    damage × 3
```

После чего damage применяется к durability в `LogicComponent.onTick()`.

---

# 21. Статус исследования

- [x] Полный класс `ItemReactorCapacitor`.
- [x] Все четыре варианта.
- [x] Базовые параметры.
- [x] `getHeat()`.
- [x] `getHeatRemovePercent()`.
- [x] `getDamageCFromHeat()`.
- [x] `getAutoRepair()`.
- [x] `getRepairOther()`.
- [x] `getEnergyProduction()`.
- [x] `damageItem()`.
- [x] `needClear()`.
- [x] `updatableItem()`.
- [x] `caneExtractHeat()`.
- [x] Получение heat в `LogicReactor`.
- [x] Дополнительный multiplier `×3`.
- [x] Поведение durability в `LogicComponent`.
- [x] Удаление компонента после zero durability.
- [x] Список необходимых тестов.

Не входит в это исследование:

- конкретные значения `moduleCapacitor`;
- конкретные `getMulHeat`;
- конкретные `getMulDamage`.

Они относятся к отдельному этапу исследования reactor-specific modifiers.
