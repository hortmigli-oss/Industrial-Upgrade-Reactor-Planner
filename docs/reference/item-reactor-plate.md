# Исследование ItemReactorPlate в контексте LogicComponent

## 1. Источник

Industrial Upgrade:

- Repository: https://github.com/ZelGimi/industrialupgrade
- Commit: `16b7fb9cfb7fbb3171e35a63532ebc0a112f665c`
- Minecraft: `1.12.2`
- Forge: `14.23.5.2860`

Основной источник:

`src/main/java/com/denfop/items/reactors/ItemReactorPlate.java`

Связанная логика:

- `src/main/java/com/denfop/api/reactors/IReactorItem.java`
- `src/main/java/com/denfop/api/reactors/IAdvReactor.java`
- `src/main/java/com/denfop/api/reactors/LogicComponent.java`
- `src/main/java/com/denfop/api/reactors/LogicReactor.java`
- `src/main/java/com/denfop/IUItem.java`

---

## 2. Назначение

`ItemReactorPlate` — компонент типа `PLATE`.

Компонент:

- не генерирует тепло;
- не генерирует энергию;
- принимает тепло от соседних компонентов;
- не передаёт тепло дальше;
- не ремонтирует себя;
- не ремонтирует другие компоненты;
- не имеет рабочей durability;
- не участвует в `onTick()`;
- при передаче тепла непосредственно от `ROD` получает дополнительный множитель `1.5`.

---

## 3. Структура класса

Класс:

```java
public class ItemReactorPlate extends ItemDamage implements IReactorItem
```

Собственные поля:

```java
private final int level;
private final double percent;
```

Конструктор:

```java
public ItemReactorPlate(
    final String name,
    int level,
    double percent
)
```

Важно:

```java
super(name, 0);
```

То есть максимальная custom durability пластины равна `0`.

При создании:

```java
setMaxStackSize(1);
```

Пластина не складывается в стак.

---

## 4. Все варианты

В baseline зарегистрированы четыре варианта:

| ID | Level | Percent | Max Damage |
|---|---:|---:|---:|
| `reactor_plate` | 1 | 2.00 | 0 |
| `adv_reactor_plate` | 2 | 1.50 | 0 |
| `imp_reactor_plate` | 3 | 1.25 | 0 |
| `per_reactor_plate` | 4 | 1.00 | 0 |

`percent` является главным физическим параметром пластины.

---

## 5. Реализация IReactorItem

| Метод | Результат |
|---|---|
| `getType()` | `PLATE` |
| `getLevel()` | `level` |
| `getAutoRepair()` | `0` |
| `getRepairOther()` | `0` |
| `getDamageCFromHeat()` | `1` |
| `getHeat()` | `0` |
| `getHeatRemovePercent()` | `percent` |
| `damageItem()` | ничего не делает |
| `updatableItem()` | `false` |
| `caneExtractHeat()` | `false` |
| `getEnergyProduction()` | `0` |
| `getRadiation()` | `0` по default реализации интерфейса |
| `needClear()` | `remainingDamage == 0` |

---

## 6. Heat

Базовое выделение тепла:

```java
getHeat() = 0;
```

Поэтому при создании `LogicComponent`:

```java
this.heat = item.getHeat(reactor);
```

получаем:

```text
initial plate.heat = 0
```

Это не означает, что `LogicComponent.heat` навсегда остаётся нулевым.

`LogicReactor` может передавать тепло в пластину и непосредственно изменять:

```text
LogicComponent.heat
```

---

## 7. Получение тепла

Передача тепла в `LogicReactor` выполняется:

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

Для пластины:

```text
getHeatRemovePercent() = percent
```

Следовательно:

```text
plateHeat +=
    sourceHeat / neighborCount
    × plate.percent
    × reactor.getMulHeat(...)
```

---

## 8. Может ли пластина передавать heat дальше

Нет.

```java
caneExtractHeat() = false;
```

Следовательно, пластина может быть получателем тепла, но не является источником дальнейшей теплопередачи через стандартный механизм:

```text
ROD
 ↓
PLATE
 ↓
X
```

Пластина является конечной точкой цепочки теплопередачи.

---

## 9. Plate может получать heat не только от Rod

Условие передачи тепла в `LogicReactor` основано на:

```java
component.canExtractHeat()
```

а не на типе источника.

Поэтому пластина может получать heat от любого соседнего компонента, который способен его передавать.

Например:

```text
Heat Exchanger
      ↓
    Plate
```

будет работать по формуле:

```text
plateHeat +=
    sourceHeat / neighborCount
    × plate.percent
    × reactor.getMulHeat(...)
```

Но дополнительный множитель `1.5` будет применён только при источнике `ROD`.

---

## 10. Специальное правило Rod → Plate

В `LogicReactor` существует отдельное правило:

```java
if (component.getItem().getType() == EnumTypeComponent.ROD) {
    if (lg.getItem().getType() == EnumTypeComponent.PLATE) {
        lg.heat *= 1.5;
    }
}
```

Следовательно:

```text
source = ROD
target = PLATE
```

даёт:

```text
plateHeat *= 1.5
```

Это правило не применяется для:

```text
Vent → Plate
Heat Exchanger → Plate
Coolant → Plate
```

если источник не имеет типа `ROD`.

---

## 11. Критический порядок операций

В `LogicReactor.calculateFirstLogic()` сначала выполняется передача тепла:

```text
1. plate.heat += transmittedHeat
```

Затем рассчитывается damage:

```text
2. plate.damage = ...
```

И только после этого проверяется:

```text
3. source == ROD && target == PLATE
```

и выполняется:

```text
4. plate.heat *= 1.5
```

Следовательно:

```text
Heat transfer
→ Damage calculation
→ Rod-to-Plate heat multiplier ×1.5
```

а не:

```text
Heat transfer
→ ×1.5
→ Damage calculation
```

Это принципиально важно для точного TypeScript-переноса.

---

## 12. Пример Rod → Plate

Допустим:

```text
rod.heat = 100
neighborCount = 1
plate.percent = 2
reactor.getMulHeat() = 1
reactor.getMulDamage() = 1
```

### Шаг 1. Heat transfer

```text
plate.heat =
    100 / 1
    × 2
    × 1

plate.heat = 200
```

### Шаг 2. Damage

`getDamageCFromHeat()` возвращает `1`.

Поэтому:

```text
plate.damage =
    200 / 1
    × 1

plate.damage = 200
```

### Шаг 3. Rod → Plate multiplier

```text
plate.heat =
    200 × 1.5

plate.heat = 300
```

Итог:

```text
plate.heat   = 300
plate.damage = 200
```

`plate.damage` не учитывает финальные `300` heat этого же прохода.

---

## 13. Формула итогового heat при Rod → Plate

До специального множителя:

```text
plateHeatBeforeRodModifier =
    sourceHeat / neighborCount
    × plate.percent
    × reactor.getMulHeat(...)
```

После:

```text
plateHeatFinal =
    plateHeatBeforeRodModifier
    × 1.5
```

Итого:

```text
plateHeatFinal =
    (
        sourceHeat / neighborCount
        × plate.percent
        × reactor.getMulHeat(...)
    )
    × 1.5
```

---

## 14. Формула damage

`ItemReactorPlate`:

```java
getDamageCFromHeat(...) = 1;
```

`getAutoRepair()`:

```java
0
```

Поэтому в момент расчёта damage:

```text
plateDamage =
    plateHeatBeforeRodModifier
    × reactor.getMulDamage(...)
```

Важно:

```text
plateDamage
```

использует heat до применения отдельного `×1.5`.

---

## 15. Energy production

```java
getEnergyProduction() = 0;
```

Поэтому пластина не увеличивает generation.

В `LogicReactor`:

```text
generation +=
    component.getEnergyProduction(...)
    × reactor.getMulOutput(...)
```

Для пластины:

```text
+0
```

---

## 16. Radiation

`ItemReactorPlate` не переопределяет `getRadiation()`.

В `IReactorItem` default:

```java
default double getRadiation() {
    return 0;
}
```

Поэтому:

```text
plate radiation = 0
```

---

## 17. Repair

Пластина не имеет repair-механики:

```text
getAutoRepair()  = 0
getRepairOther() = 0
```

---

## 18. Damage и durability

У пластины:

```java
super(name, 0);
```

следовательно:

```text
maxDamage = 0
```

Кроме того:

```java
damageItem(...) {
}
```

метод полностью пустой.

Поэтому фактического изменения durability у пластины нет.

---

## 19. updatableItem()

```java
updatableItem() = false;
```

`LogicReactor` добавляет в `list1` только компоненты:

```java
if (component.getItem().updatableItem()
    && !list1.contains(component)) {
    list1.add(component);
}
```

Следовательно, пластина не попадает в список компонентов, которые потом обрабатываются через:

```text
LogicComponent.onTick()
```

---

## 20. Последствия updatableItem = false

Поскольку Plate не входит в обычный `onTick()` список:

- её damage не применяется к ItemStack;
- её durability не уменьшается;
- она не удаляется через `maxDamageItem <= 0`;
- `damageItem()` фактически не вызывается в обычном tick;
- пластина существует в реакторе независимо от рассчитанного `damage`.

Таким образом, вычисляемое:

```text
LogicComponent.damage
```

для Plate является расчётной величиной, но не механизмом разрушения предмета.

---

## 21. needClear()

Метод:

```java
return this.getMaxCustomDamage(stack)
    - this.getCustomDamage(stack) == 0;
```

При:

```text
maxDamage = 0
customDamage = 0
```

результат формально:

```text
needClear() = true
```

Но это не приводит к автоматическому удалению Plate.

Причина:

```text
updatableItem() = false
```

и пластина не попадает в список `LogicReactor.listComponent`, который обрабатывает `onTick()`.

---

## 22. Архитектурная модель Plate

Логически:

```text
               Plate
                 │
       ┌─────────┼─────────┐
       │         │         │
      Heat     Damage    Output
       │         │         │
       ▼         ▼         ▼
    получает   считается     0
     heat     формально
       │
       X
  дальше не передаёт
```

---

## 23. TypeScript-модель

```ts
interface ReactorPlateDefinition {
    id: string;
    type: 'PLATE';

    level: number;
    percent: number;

    maxDamage: 0;
}
```

Данные:

```ts
const plates: ReactorPlateDefinition[] = [
    {
        id: 'reactor_plate',
        type: 'PLATE',
        level: 1,
        percent: 2,
        maxDamage: 0,
    },
    {
        id: 'adv_reactor_plate',
        type: 'PLATE',
        level: 2,
        percent: 1.5,
        maxDamage: 0,
    },
    {
        id: 'imp_reactor_plate',
        type: 'PLATE',
        level: 3,
        percent: 1.25,
        maxDamage: 0,
    },
    {
        id: 'per_reactor_plate',
        type: 'PLATE',
        level: 4,
        percent: 1,
        maxDamage: 0,
    },
];
```

---

## 24. Unit Tests

### Базовые свойства

- [ ] `type === PLATE`.
- [ ] Проверить level.
- [ ] `getHeat() === 0`.
- [ ] `getEnergyProduction() === 0`.
- [ ] `getAutoRepair() === 0`.
- [ ] `getRepairOther() === 0`.
- [ ] `getDamageCFromHeat() === 1`.
- [ ] `caneExtractHeat() === false`.
- [ ] `updatableItem() === false`.
- [ ] `getRadiation() === 0`.

### Heat transfer

- [ ] Rod → Plate.
- [ ] Heat Exchanger → Plate.
- [ ] Vent → Plate.
- [ ] Проверить `percent`.
- [ ] Проверить `reactor.getMulHeat()`.
- [ ] Проверить, что Plate не передаёт heat дальше.

### Специальное правило

- [ ] Rod → Plate даёт `×1.5`.
- [ ] non-Rod → Plate не даёт `×1.5`.

### Порядок операций

- [ ] Damage рассчитывается до `×1.5`.
- [ ] Финальный `plate.heat` учитывает `×1.5`.
- [ ] `plate.damage` не учитывает этот `×1.5`.

### Durability

- [ ] Max damage равен `0`.
- [ ] `damageItem()` ничего не делает.
- [ ] Plate не удаляется из-за рассчитанного damage.
- [ ] Plate не участвует в обычном `onTick()`.

---

## 25. Golden Test

Минимальная схема:

```text
ROD → PLATE
```

Например:

```text
rod.heat = 100
plate.percent = 2
mulHeat = 1
mulDamage = 1
```

Ожидаемый результат одного расчёта:

```text
plate.heat = 300
plate.damage = 200
```

Этот тест особенно важен, потому что он проверяет порядок:

```text
transfer
→ damage
→ ×1.5
```

---

## 26. Выявленные особенности baseline

1. `ItemReactorPlate` имеет maxDamage `0`.
2. `damageItem()` пустой.
3. `updatableItem()` равен `false`.
4. Plate может получать тепло, но не может передавать его дальше.
5. Plate имеет разные коэффициенты `percent`: `2 / 1.5 / 1.25 / 1`.
6. При передаче именно от `ROD` heat дополнительно умножается на `1.5`.
7. Этот `×1.5` применяется после расчёта damage.
8. Из-за `updatableItem = false` рассчитанный damage не приводит к разрушению Plate.
9. Plate не генерирует энергию.
10. Plate не производит radiation.

---

## 27. Статус исследования

- [x] Полный класс `ItemReactorPlate`.
- [x] Все четыре варианта.
- [x] Level.
- [x] Percent.
- [x] Max damage.
- [x] Heat.
- [x] Heat transfer.
- [x] `getHeatRemovePercent()`.
- [x] `getDamageCFromHeat()`.
- [x] `getAutoRepair()`.
- [x] `getRepairOther()`.
- [x] `getEnergyProduction()`.
- [x] Radiation.
- [x] `damageItem()`.
- [x] `needClear()`.
- [x] `updatableItem()`.
- [x] `caneExtractHeat()`.
- [x] Взаимодействие с `LogicComponent`.
- [x] Взаимодействие с `LogicReactor`.
- [x] Правило Rod → Plate.
- [x] Точный порядок расчёта heat и damage.
- [x] Durability и отсутствие фактического разрушения.

Внешние зависимости:

- [ ] `reactor.getMulHeat(...)`.
- [ ] `reactor.getMulDamage(...)`.

Эти методы относятся к общему исследованию `reactor-specific modifiers`.
