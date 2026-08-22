# Исследование ItemNeutronProtector

## 1. Источник

Industrial Upgrade:

- Repository: https://github.com/ZelGimi/industrialupgrade
- Commit: `16b7fb9cfb7fbb3171e35a63532ebc0a112f665c`
- Minecraft: `1.12.2`
- Forge: `14.23.5.2860`

Основной источник:

`src/main/java/com/denfop/items/reactors/ItemNeutronProtector.java`

Связанная логика:

- `src/main/java/com/denfop/api/reactors/IReactorItem.java`
- `src/main/java/com/denfop/api/reactors/IAdvReactor.java`
- `src/main/java/com/denfop/api/reactors/LogicComponent.java`
- `src/main/java/com/denfop/api/reactors/LogicReactor.java`
- `src/main/java/com/denfop/IUItem.java`

---

## 2. Назначение документа

Документ фиксирует фактическое поведение `ItemNeutronProtector` в baseline commit.

При реализации TypeScript необходимо воспроизводить именно программное поведение Industrial Upgrade, даже если оно отличается от названия или предполагаемого назначения предмета.

---

## 3. Структура класса

Класс:

```java
public class ItemNeutronProtector extends ItemDamage implements IReactorItem
```

Собственное поле:

```java
private final int level;
```

Конструктор:

```java
public ItemNeutronProtector(
    final String name,
    final int maxDamage,
    int level
)
```

При создании:

```java
setMaxStackSize(1);
```

Компонент имеет максимальный размер стака `1`.

---

## 4. Четыре варианта

В `IUItem.java` зарегистрированы четыре варианта:

| ID | Max Damage | Level |
|---|---:|---:|
| `neutron_protector` | 14400 | 1 |
| `adv_neutron_protector` | 43200 | 2 |
| `imp_neutron_protector` | 64800 | 3 |
| `per_neutron_protector` | 86400 | 4 |

Значения max damage задаются выражениями:

```text
3600 × 4 = 14400
7200 × 6 = 43200
10800 × 6 = 64800
14400 × 6 = 86400
```

В baseline нет параметров `Config.*`, управляющих этими значениями.

---

## 5. Критическое несоответствие типа

Название класса и предмета:

```text
Neutron Protector
```

Ожидаемый логический тип:

```text
NEUTRON_PROTECTOR
```

Фактическая реализация:

```java
@Override
public EnumTypeComponent getType() {
    return EnumTypeComponent.ENERGY_COUPLER;
}
```

То есть фактический тип в Industrial Upgrade:

```text
ENERGY_COUPLER
```

а не:

```text
NEUTRON_PROTECTOR
```

Это критично, потому что `LogicComponent` и `LogicReactor` используют `getType()` для выбора ветки обработки.

При переносе поведения это **нельзя исправлять**.

Для точного симулятора нужно сохранить:

```text
logicalName = NEUTRON_PROTECTOR
type         = ENERGY_COUPLER
```

---

## 6. Реализация IReactorItem

Фактическое поведение:

| Метод | Значение |
|---|---|
| `getType()` | `ENERGY_COUPLER` |
| `getLevel()` | `level` |
| `getAutoRepair()` | `0` |
| `getRepairOther()` | `0` |
| `getDamageCFromHeat()` | `1` |
| `getHeat()` | `0` |
| `getHeatRemovePercent()` | `1` |
| `damageItem()` | `applyCustomDamage(stack, -1, null)` |
| `updatableItem()` | `true` |
| `caneExtractHeat()` | `true` |
| `getEnergyProduction()` | `0` |
| `getRadiation()` | `0` по default реализации интерфейса |
| `needClear()` | `remainingDamage == 0` |

---

## 7. Radiation

`ItemNeutronProtector` не переопределяет:

```java
getRadiation()
```

В `IReactorItem` используется default:

```java
default double getRadiation() {
    return 0;
}
```

Поэтому:

```text
radiation = 0
```

сам по себе `Neutron Protector` радиацию не производит.

---

## 8. Базовое heat

Класс возвращает:

```java
getHeat() = 0
```

При создании `LogicComponent`:

```java
this.heat = item.getHeat(reactor);
```

Поэтому начальное состояние:

```text
heat = 0
```

Это означает только отсутствие собственного источника тепла.

В дальнейшем поле `LogicComponent.heat` может стать ненулевым из-за передачи тепла от соседей.

---

## 9. Получение heat

Поскольку:

```java
getHeatRemovePercent() = 1
```

и:

```java
caneExtractHeat() = true
```

`Neutron Protector` участвует в обычной теплопередаче `LogicReactor`.

Фрагмент общей формулы:

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

Для протектора:

```text
heatRemovePercent = 1
```

Следовательно:

```text
receivedHeat =
    sourceHeat / neighborCount
    × reactor.getMulHeat(...)
```

---

## 10. Может ли протектор передавать heat дальше

Да.

Причина:

```java
caneExtractHeat() = true
```

Если `LogicReactor` рассматривает сам протектор как источник передачи тепла, его поле:

```text
LogicComponent.heat
```

может участвовать в дальнейшей цепочке.

Пример:

```text
ROD
 ↓
NEUTRON PROTECTOR
 ↓
VENT
```

может быть частью тепловой цепочки.

---

## 11. Damage from Heat

Класс возвращает:

```java
getDamageCFromHeat(...) = 1
```

Поэтому до применения других модификаторов:

```text
damageBase =
    currentHeat / 1
```

В `LogicReactor` используется:

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

Для протектора:

```text
damageBase =
    currentHeat
    × reactor.getMulDamage(...)
```

поскольку:

```text
getDamageCFromHeat() = 1
getAutoRepair()      = 0
```

---

## 12. Damage от heat не определяет реальный расход durability

Это ключевая особенность.

Хотя `LogicReactor` рассчитывает:

```text
LogicComponent.damage
```

`LogicComponent.onTick()` обрабатывает компоненты типа:

```text
ENERGY_COUPLER
NEUTRON_PROTECTOR
```

отдельной веткой:

```java
if (
    this.getItem().getType() == EnumTypeComponent.ENERGY_COUPLER
    || this.getItem().getType() == EnumTypeComponent.NEUTRON_PROTECTOR
) {
    this.item.damageItem(this.stack, 1);
    this.maxDamageItem -= 1;
}
```

Так как `ItemNeutronProtector.getType()` фактически возвращает:

```text
ENERGY_COUPLER
```

протектор попадает в эту ветку.

Таким образом реальный расход durability:

```text
1 durability / tick
```

и не зависит от рассчитанного `damage`.

---

## 13. Реальный порядок одного tick

Для `Neutron Protector`:

```text
1. Получение heat от соседей.
2. Расчёт LogicComponent.damage.
3. Переход в LogicComponent.onTick().
4. Проверка типа ENERGY_COUPLER.
5. Вызов damageItem(stack, 1).
6. Уменьшение maxDamageItem на 1.
7. Проверка maxDamageItem <= 0.
8. При достижении нуля компонент добавляется на удаление.
9. LogicReactor удаляет компонент из реактора.
```

Следовательно:

```text
heat
  ↓
damage вычисляется
  ↓
damage не используется для durability
  ↓
durability -= 1
```

---

## 14. damageItem()

Реализация:

```java
@Override
public void damageItem(final ItemStack stack, final int damage) {
    applyCustomDamage(stack, -1, null);
}
```

Переданный аргумент игнорируется.

Фактический вызов всегда:

```text
applyCustomDamage(stack, -1, null)
```

Поэтому независимо от того, какой аргумент пришёл:

```text
damageItem(stack, 1)
damageItem(stack, 100)
damageItem(stack, 1000)
```

поведение внутри класса одинаковое.

---

## 15. Durability в LogicComponent

При создании:

```java
this.maxDamage =
    damageItem.getMaxCustomDamage(stack);

this.maxDamageItem =
    maxDamage
    - damageItem.getCustomDamage(stack);
```

Для нового компонента:

```text
maxDamageItem = maxDamage
```

Далее каждый tick:

```text
maxDamageItem -= 1
```

Примеры:

| Компонент | Max Damage | Расход |
|---|---:|---:|
| `neutron_protector` | 14400 | 1/tick |
| `adv_neutron_protector` | 43200 | 1/tick |
| `imp_neutron_protector` | 64800 | 1/tick |
| `per_neutron_protector` | 86400 | 1/tick |

---

## 16. Теоретический срок службы

При стандартном предположении:

```text
20 ticks = 1 секунда
```

получаем:

| ID | Тики | Секунды | Минуты |
|---|---:|---:|---:|
| `neutron_protector` | 14400 | 720 | 12 |
| `adv_neutron_protector` | 43200 | 2160 | 36 |
| `imp_neutron_protector` | 64800 | 3240 | 54 |
| `per_neutron_protector` | 86400 | 4320 | 72 |

Это не отдельная игровая характеристика IU, а следствие `onTick()`.

---

## 17. needClear()

Реализация:

```java
return this.getMaxCustomDamage(stack)
    - this.getCustomDamage(stack) == 0;
```

Компонент считается пустым при нулевой оставшейся durability.

---

## 18. Energy Coupler branch

Поскольку тип протектора:

```text
ENERGY_COUPLER
```

он попадает в специальную ветку `LogicReactor`:

```java
if (component.getItem().getType() == EnumTypeComponent.ENERGY_COUPLER) {
    ...
}
```

В этой ветке ищутся соседние `ROD`.

Для каждого стержня рассчитывается дополнительная генерация:

```text
rodEnergyProduction
×
couplerEnergyProduction
```

и аналогично радиация:

```text
rodRadiation
×
couplerEnergyProduction
```

Но для `ItemNeutronProtector`:

```text
getEnergyProduction() = 0
```

поэтому:

```text
additionalGeneration = 0
additionalRadiation  = 0
```

Фактически протектор формально участвует в ветке `ENERGY_COUPLER`, но ничего не добавляет.

---

## 19. Почему этот компонент нельзя моделировать как обычный NEUTRON_PROTECTOR

Если в TypeScript сделать:

```ts
type = 'NEUTRON_PROTECTOR'
```

и переписать стандартную логику для такого типа, поведение изменится.

Оригинальная IU семантика:

```text
Название:
Neutron Protector

Фактический EnumTypeComponent:
ENERGY_COUPLER

Energy production:
0

Durability loss:
1/tick

Radiation:
0

Heat:
получает и может передавать
```

Это является частью baseline.

---

## 20. Рекомендуемая TypeScript-модель

Для хранения данных:

```ts
interface NeutronProtectorDefinition {
    id: string;

    // Фактический тип Industrial Upgrade.
    type: 'ENERGY_COUPLER';

    // Человеко-читаемое назначение.
    logicalName: 'NEUTRON_PROTECTOR';

    level: number;
    maxDamage: number;

    heat: 0;
    heatRemovePercent: 1;
    damageFromHeat: 1;

    energyProduction: 0;
    radiation: 0;

    autoRepair: 0;
    repairOther: 0;

    updatable: true;
    canExtractHeat: true;
}
```

Данные:

```ts
const neutronProtectors: NeutronProtectorDefinition[] = [
    {
        id: 'neutron_protector',
        type: 'ENERGY_COUPLER',
        logicalName: 'NEUTRON_PROTECTOR',
        level: 1,
        maxDamage: 14400,
        heat: 0,
        heatRemovePercent: 1,
        damageFromHeat: 1,
        energyProduction: 0,
        radiation: 0,
        autoRepair: 0,
        repairOther: 0,
        updatable: true,
        canExtractHeat: true,
    },
    {
        id: 'adv_neutron_protector',
        type: 'ENERGY_COUPLER',
        logicalName: 'NEUTRON_PROTECTOR',
        level: 2,
        maxDamage: 43200,
        heat: 0,
        heatRemovePercent: 1,
        damageFromHeat: 1,
        energyProduction: 0,
        radiation: 0,
        autoRepair: 0,
        repairOther: 0,
        updatable: true,
        canExtractHeat: true,
    },
    {
        id: 'imp_neutron_protector',
        type: 'ENERGY_COUPLER',
        logicalName: 'NEUTRON_PROTECTOR',
        level: 3,
        maxDamage: 64800,
        heat: 0,
        heatRemovePercent: 1,
        damageFromHeat: 1,
        energyProduction: 0,
        radiation: 0,
        autoRepair: 0,
        repairOther: 0,
        updatable: true,
        canExtractHeat: true,
    },
    {
        id: 'per_neutron_protector',
        type: 'ENERGY_COUPLER',
        logicalName: 'NEUTRON_PROTECTOR',
        level: 4,
        maxDamage: 86400,
        heat: 0,
        heatRemovePercent: 1,
        damageFromHeat: 1,
        energyProduction: 0,
        radiation: 0,
        autoRepair: 0,
        repairOther: 0,
        updatable: true,
        canExtractHeat: true,
    },
];
```

---

## 21. Unit Tests

### Данные

- [ ] Проверить `neutron_protector`.
- [ ] Проверить `adv_neutron_protector`.
- [ ] Проверить `imp_neutron_protector`.
- [ ] Проверить `per_neutron_protector`.

### API

- [ ] `getType() === ENERGY_COUPLER`.
- [ ] Проверить `level`.
- [ ] `getHeat() === 0`.
- [ ] `getHeatRemovePercent() === 1`.
- [ ] `getDamageCFromHeat() === 1`.
- [ ] `getEnergyProduction() === 0`.
- [ ] `getRadiation() === 0`.
- [ ] `getAutoRepair() === 0`.
- [ ] `getRepairOther() === 0`.
- [ ] `canExtractHeat() === true`.
- [ ] `updatableItem() === true`.

### Heat

- [ ] Протектор получает heat.
- [ ] Протектор способен передавать heat дальше.
- [ ] `getHeat()` остаётся базовым значением `0`.

### Durability

- [ ] Новый компонент получает полную durability.
- [ ] За tick расходуется 1 durability.
- [ ] Heat damage не ускоряет этот расход.
- [ ] Переданный аргумент `damageItem()` игнорируется.
- [ ] На нулевой durability компонент удаляется.

### Energy Coupler branch

- [ ] Тип участвует в ветке `ENERGY_COUPLER`.
- [ ] Не добавляется дополнительная генерация.
- [ ] Не добавляется дополнительная радиация.

---

## 22. Golden Tests

Для baseline необходимо создать минимальную тестовую схему:

```text
ROD → NEUTRON_PROTECTOR → VENT
```

Проверить после нескольких tick:

```text
heat протектора
damage протектора
durability протектора
generation
radiation
состояние соседей
```

Особенно важно проверить, что:

```text
LogicComponent.damage
```

может изменяться от heat, но:

```text
durability -= 1/tick
```

остаётся неизменным.

---

## 23. Внешние зависимости

Поведение компонента зависит от:

```text
reactor.getMulHeat(...)
reactor.getMulDamage(...)
```

Конкретные реализации этих modifiers относятся к отдельному исследованию:

**Reactor-specific modifiers**

Они не являются частью `ItemNeutronProtector`.

---

## 24. Статус исследования

- [x] Полный класс `ItemNeutronProtector`.
- [x] Все четыре варианта.
- [x] Max damage.
- [x] Level.
- [x] Фактический `EnumTypeComponent`.
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
- [x] Поведение `LogicComponent.onTick()`.
- [x] Поведение durability.
- [x] Удаление после zero durability.
- [x] Ветка `ENERGY_COUPLER` в `LogicReactor`.
- [x] Выявлено несоответствие имени `Neutron Protector` и фактического типа `ENERGY_COUPLER`.

Осталось исследовать отдельно:

- [ ] `reactor.getMulHeat(...)`.
- [ ] `reactor.getMulDamage(...)`.
- [ ] Конкретные значения reactor-specific modifiers.
