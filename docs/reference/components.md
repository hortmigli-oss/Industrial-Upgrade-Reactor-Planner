# Компоненты реакторов Industrial Upgrade

## 1. Назначение документа

Этот документ фиксирует модель компонентов реакторов Industrial Upgrade, которая используется как эталон для реализации симулятора.

Все данные относятся к зафиксированному состоянию Industrial Upgrade:

- Repository: https://github.com/ZelGimi/industrialupgrade
- Commit: `16b7fb9cfb7fbb3171e35a63532ebc0a112f665c`
- Minecraft: `1.12.2`
- Forge: `14.23.5.2860`

Основной принцип проекта:

> Поведение TypeScript-симулятора должно соответствовать поведению Industrial Upgrade на указанном commit.

Источник нельзя заменять документацией, описанием мода или приблизительными формулами.

---

## 2. API компонента

Базовый контракт компонента задаётся `IReactorItem`.

Источник:

`src/main/java/com/denfop/api/reactors/IReactorItem.java`

Контракт:

```java
EnumTypeComponent getType();

default double getRadiation() {
    return 0;
}

int getLevel();

int getAutoRepair(IAdvReactor reactor);

int getRepairOther(IAdvReactor reactor);

int getDamageCFromHeat(int heat, IAdvReactor reactor);

int getHeat(IAdvReactor reactor);

double getHeatRemovePercent(IAdvReactor reactor);

void damageItem(ItemStack stack, int damage);

boolean updatableItem();

boolean caneExtractHeat();

double getEnergyProduction(IAdvReactor reactor);

boolean needClear(ItemStack stack);
```

Таким образом, для симулятора критическими характеристиками компонента являются:

| Свойство | Назначение |
|---|---|
| `type` | Логический тип |
| `radiation` | Радиация |
| `level` | Уровень компонента |
| `autoRepair` | Саморемонт |
| `repairOther` | Ремонт других компонентов |
| `damageFromHeat` | Урон от нагрева |
| `heat` | Выделяемое тепло |
| `heatRemovePercent` | Отвод тепла |
| `energyProduction` | Производство энергии |
| `updatableItem` | Участвует ли в tick |
| `canExtractHeat` | Может ли извлекать тепло |
| `needClear` | Требует ли замены |

---

## 3. Логические типы компонентов

Тип компонента задаётся `EnumTypeComponent`.

```java
ROD,
PLATE,
CAPACITOR,
HEAT_EXCHANGER,
HEAT_SINK,
COOLANT_ROD,
ENERGY_COUPLER,
NEUTRON_PROTECTOR
```

Источник:

`src/main/java/com/denfop/api/reactors/EnumTypeComponent.java`

| Тип | Назначение |
|---|---|
| `ROD` | Топливный стержень |
| `PLATE` | Реакторная пластина |
| `CAPACITOR` | Конденсатор |
| `HEAT_EXCHANGER` | Теплообменник |
| `HEAT_SINK` | Теплоотвод |
| `COOLANT_ROD` | Охлаждающий стержень |
| `ENERGY_COUPLER` | Энергетический соединитель |
| `NEUTRON_PROTECTOR` | Нейтронный протектор |

---

## 4. Соответствие логических типов и Java-классов

| `EnumTypeComponent` | Класс |
|---|---|
| `ROD` | `ItemBaseRod` |
| `PLATE` | `ItemReactorPlate` |
| `CAPACITOR` | `ItemReactorCapacitor` |
| `HEAT_EXCHANGER` | `ItemReactorHeatExchanger` |
| `HEAT_SINK` | `ItemReactorVent` |
| `HEAT_SINK` | `ItemComponentVent` |
| `COOLANT_ROD` | `ItemReactorCoolant` |
| `ENERGY_COUPLER` | `ItemEnergyCoupler` |
| `NEUTRON_PROTECTOR` | `ItemNeutronProtector` |

Важно: `ItemReactorVent` и `ItemComponentVent` имеют один логический тип `HEAT_SINK`, но это разные конкретные компоненты с разным поведением.

---

## 5. Полный набор конкретных компонентов

На основании регистрации компонентов в `IUItem.java`:

| Группа | Количество |
|---|---:|
| Fuel Rod | 45 |
| Reactor Plate | 4 |
| Reactor Capacitor | 4 |
| Heat Exchanger | 4 |
| Reactor Vent | 4 |
| Component Vent | 4 |
| Coolant Rod | 3 |
| Energy Coupler | 4 |
| Neutron Protector | 4 |
| **Итого** | **76** |

---

# 6. Fuel Rod

## 6.1. Java-класс

`ItemBaseRod`

Источник:

`src/main/java/com/denfop/items/reactors/ItemBaseRod.java`

Класс реализует:

- `IRadioactiveItemType`
- `IReactorItem`

## 6.2. Поля Fuel Rod

`ItemBaseRod` хранит:

```text
numberOfCells
heat
power
name
level
radiation
```

Радиация:

```text
radiation = power × level × cells
```

Energy production использует массив:

```text
[5, 20, 60, 200]
```

и зависит от количества ячеек.

Для поддерживаемых вариантов:

| Cells | Множитель |
|---:|---:|
| 1 | 5 |
| 2 | 20 |
| 4 | 60 |

В исходнике множитель получается через логарифмическое выражение:

```java
double temp = Math.log10(this.numberOfCells);
double temp1 = Math.log10(2);
double m = temp / temp1;
return p[(int) m] * this.power * this.level;
```

Для TypeScript допустимо хранить значения `5 / 20 / 60` явно, если итоговые результаты совпадают с IU.

## 6.3. Общее поведение Fuel Rod

Для `ItemBaseRod`:

```text
type               = ROD
heat               = constructor heat
energyProduction   = cellsMultiplier × power × level
radiation          = power × level × cells
heatRemovePercent  = 0
autoRepair         = 0
repairOther        = 0
damageFromHeat     = 1
updatableItem      = true
canExtractHeat     = true
```

---

## 6.4. Uranium

```text
uranium_fuel_rod
dual_uranium_fuel_rod
quad_uranium_fuel_rod
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `uranium_fuel_rod` | 1 | 25 | 1.5 | 1 |
| `dual_uranium_fuel_rod` | 2 | 50 | 1.5 | 1 |
| `quad_uranium_fuel_rod` | 4 | 100 | 1.5 | 1 |

---

## 6.5. MOX

```text
mox_fuel_rod
dual_mox_fuel_rod
quad_mox_fuel_rod
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `mox_fuel_rod` | 1 | 40 | 3.5 | 1 |
| `dual_mox_fuel_rod` | 2 | 80 | 3.5 | 1 |
| `quad_mox_fuel_rod` | 4 | 160 | 3.5 | 1 |

---

## 6.6. Uranium-233

В регистрации используются:

```text
reactoruran233Simple
reactoruran233Dual
reactoruran233Quad
```

Параметры:

```text
cells = 1 / 2 / 4
heat  = 35 / 70 / 140
level = 1
power = Config.uran233Power
```

---

## 6.7. Thorium

```text
reactortoriysimple
reactortoriydual
reactortoriyquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactortoriysimple` | 1 | 50 | `Config.toriyPower` | 2 |
| `reactortoriydual` | 2 | 100 | `Config.toriyPower` | 2 |
| `reactortoriyquad` | 4 | 200 | `Config.toriyPower` | 2 |

---

## 6.8. Americium

```text
reactoramericiumsimple
reactoramericiumdual
reactoramericiumquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactoramericiumsimple` | 1 | 80 | `Config.americiumPower` | 2 |
| `reactoramericiumdual` | 2 | 160 | `Config.americiumPower` | 2 |
| `reactoramericiumquad` | 4 | 320 | `Config.americiumPower` | 2 |

---

## 6.9. Neptunium

```text
reactorneptuniumsimple
reactorneptuniumdual
reactorneptuniumquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactorneptuniumsimple` | 1 | 65 | `Config.neptuniumPower` | 2 |
| `reactorneptuniumdual` | 2 | 130 | `Config.neptuniumPower` | 2 |
| `reactorneptuniumquad` | 4 | 260 | `Config.neptuniumPower` | 2 |

---

## 6.10. Curium

```text
reactorcuriumsimple
reactorcuriumdual
reactorcuriumquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactorcuriumsimple` | 1 | 100 | `Config.curiumPower` | 3 |
| `reactorcuriumdual` | 2 | 200 | `Config.curiumPower` | 3 |
| `reactorcuriumquad` | 4 | 400 | `Config.curiumPower` | 3 |

---

## 6.11. Californium

```text
reactorcaliforniasimple
reactorcaliforniadual
reactorcaliforniaquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactorcaliforniasimple` | 1 | 120 | `Config.californiaPower` | 3 |
| `reactorcaliforniadual` | 2 | 240 | `Config.californiaPower` | 3 |
| `reactorcaliforniaquad` | 4 | 480 | `Config.californiaPower` | 3 |

---

## 6.12. Fermium

```text
reactorfermiumsimple
reactorfermiumdual
reactorfermiumquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactorfermiumsimple` | 1 | 230 | `Config.mendeleviumPower` | 4 |
| `reactorfermiumdual` | 2 | 460 | `Config.mendeleviumPower` | 4 |
| `reactorfermiumquad` | 4 | 920 | `Config.mendeleviumPower` | 4 |

Важно: в исходнике для Fermium используется `Config.mendeleviumPower`. Это особенность baseline commit и не должна исправляться при переносе.

---

## 6.13. Mendelevium

```text
reactormendeleviumsimple
reactormendeleviumdual
reactormendeleviumquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactormendeleviumsimple` | 1 | 260 | 36 | 4 |
| `reactormendeleviumdual` | 2 | 520 | 36 | 4 |
| `reactormendeleviumquad` | 4 | 1050 | 36 | 4 |

---

## 6.14. Nobelium

```text
reactornobeliumsimple
reactornobeliumdual
reactornobeliumquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactornobeliumsimple` | 1 | 285 | 49 | 4 |
| `reactornobeliumdual` | 2 | 590 | 49 | 4 |
| `reactornobeliumquad` | 4 | 1200 | 49 | 4 |

---

## 6.15. Lawrencium

```text
reactorlawrenciumsimple
reactorlawrenciumdual
reactorlawrenciumquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactorlawrenciumsimple` | 1 | 300 | 60 | 4 |
| `reactorlawrenciumdual` | 2 | 620 | 60 | 4 |
| `reactorlawrenciumquad` | 4 | 1300 | 60 | 4 |

---

## 6.16. Berkelium

```text
reactorberkeliumsimple
reactorberkeliumdual
reactorberkeliumquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactorberkeliumsimple` | 1 | 150 | `Config.berkeliumPower` | 4 |
| `reactorberkeliumdual` | 2 | 300 | `Config.berkeliumPower` | 4 |
| `reactorberkeliumquad` | 4 | 600 | `Config.berkeliumPower` | 4 |

---

## 6.17. Einsteinium

```text
reactoreinsteiniumsimple
reactoreinsteiniumdual
reactoreinsteiniumquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactoreinsteiniumsimple` | 1 | 180 | `Config.einsteiniumPower` | 4 |
| `reactoreinsteiniumdual` | 2 | 360 | `Config.einsteiniumPower` | 4 |
| `reactoreinsteiniumquad` | 4 | 720 | `Config.einsteiniumPower` | 4 |

---

## 6.18. Proton

```text
reactorprotonsimple
reactorprotondual
reactorprotonquad
```

| ID | Cells | Heat | Power | Level |
|---|---:|---:|---:|---:|
| `reactorprotonsimple` | 1 | 95 | `Config.ProtonPower` | 3 |
| `reactorprotondual` | 2 | 190 | `Config.ProtonPower` | 3 |
| `reactorprotonquad` | 4 | 380 | `Config.ProtonPower` | 3 |

---

# 7. Reactor Plate

Класс:

`ItemReactorPlate`

Источник:

`src/main/java/com/denfop/items/reactors/ItemReactorPlate.java`

Конкретные компоненты:

| ID | Level | Percent |
|---|---:|---:|
| `reactor_plate` | 1 | 2.00 |
| `adv_reactor_plate` | 2 | 1.50 |
| `imp_reactor_plate` | 3 | 1.25 |
| `per_reactor_plate` | 4 | 1.00 |

Поведение:

```text
type               = PLATE
heat               = 0
energyProduction   = 0
autoRepair         = 0
repairOther        = 0
damageFromHeat     = 1
heatRemovePercent  = percent
updatableItem      = false
canExtractHeat     = false
```

---

# 8. Reactor Vent

Класс:

`ItemReactorVent`

Источник:

`src/main/java/com/denfop/items/reactors/ItemReactorVent.java`

Конкретные варианты:

| ID | Max Damage | Level | Heat→Damage | Heat Damage | Auto Repair |
|---|---:|---:|---:|---:|---:|
| `vent` | 2500 | 1 | 8 | 0.90 | 4 |
| `adv_vent` | 4000 | 2 | 10 | 0.85 | 7 |
| `imp_vent` | 5000 | 3 | 14 | 0.80 | 11 |
| `per_vent` | 6000 | 4 | 20 | 0.75 | 15 |

Поведение:

```text
type              = HEAT_SINK
heat              = 0
energyProduction  = 0
damageFromHeat    = heat_to_damage
heatRemovePercent = heat_damage × reactor.getModuleVent()
autoRepair        = autoRepair × reactor.getModuleComponentVent()
repairOther       = 0
updatableItem     = true
canExtractHeat    = true
```

---

# 9. Component Vent

Класс:

`ItemComponentVent`

Источник:

`src/main/java/com/denfop/items/reactors/ItemComponentVent.java`

Конкретные варианты:

| ID | Level | Base Repair |
|---|---:|---:|
| `component_vent` | 1 | 3 |
| `adv_component_vent` | 2 | 4 |
| `imp_component_vent` | 3 | 5 |
| `per_component_vent` | 4 | 6 |

Поведение:

```text
type              = HEAT_SINK
heat              = 0
energyProduction  = 0
damageFromHeat    = 1
heatRemovePercent = 1.2
autoRepair        = 0
repairOther       = autoRepair × reactor.getModuleComponentVent()
updatableItem     = true
canExtractHeat    = true
```

---

# 10. Heat Exchanger

Класс:

`ItemReactorHeatExchanger`

Источник:

`src/main/java/com/denfop/items/reactors/ItemReactorHeatExchanger.java`

Конкретные варианты:

| ID | Max Damage | Level | Heat→Damage | Heat Damage |
|---|---:|---:|---:|---:|
| `heat_exchange` | 2500 | 1 | 10 | 0.80 |
| `adv_heat_exchange` | 5000 | 2 | 12 | 0.75 |
| `imp_heat_exchange` | 7500 | 3 | 15 | 0.60 |
| `per_heat_exchange` | 10000 | 4 | 20 | 0.45 |

Поведение:

```text
type              = HEAT_EXCHANGER
heat              = 0
energyProduction  = 0
damageFromHeat    = heat_to_damage
heatRemovePercent = heat_damage × reactor.getModuleExchanger()
autoRepair        = 0
repairOther       = 0
updatableItem     = true
canExtractHeat    = true
```

---

# 11. Energy Coupler

Класс:

`ItemEnergyCoupler`

Источник:

`src/main/java/com/denfop/items/reactors/ItemEnergyCoupler.java`

Конкретные варианты:

| ID | Max Damage | Level | `mult` |
|---|---:|---:|---:|
| `proton_energy_coupler` | 9000 | 1 | 0.05 |
| `adv_proton_energy_coupler` | 18000 | 2 | 0.10 |
| `imp_proton_energy_coupler` | 32400 | 3 | 0.15 |
| `per_proton_energy_coupler` | 50400 | 4 | 0.20 |

Значения max damage получаются из регистрации:

```text
3600 × 2.5 = 9000
7200 × 2.5 = 18000
10800 × 3 = 32400
14400 × 3.5 = 50400
```

Поведение:

```text
type              = ENERGY_COUPLER
heat              = 0
heatRemovePercent = 0
energyProduction  = mult
autoRepair        = 0
repairOther       = 0
damageFromHeat    = 1
updatableItem     = true
canExtractHeat    = true
```

Особенность:

`damageItem()` игнорирует переданный damage и применяет `-1` к custom damage.

---

# 12. Reactor Capacitor

Класс:

`ItemReactorCapacitor`

Зарегистрированные компоненты:

| ID | Level | Max Damage | Additional Value |
|---|---:|---:|---:|
| `capacitor` | 1 | 25000 | 4 |
| `adv_capacitor` | 2 | 50000 | 6 |
| `imp_capacitor` | 3 | 100000 | 8 |
| `per_capacitor` | 4 | 200000 | 12 |

Регистрация в `IUItem.java`:

```text
capacitor      = ItemReactorCapacitor("capacitor", 25000, 1, 4)
adv_capacitor  = ItemReactorCapacitor("adv_capacitor", 50000, 2, 6)
imp_capacitor  = ItemReactorCapacitor("imp_capacitor", 100000, 3, 8)
per_capacitor  = ItemReactorCapacitor("per_capacitor", 200000, 4, 12)
```

Важно: `ItemCapacitor` и `ItemReactorCapacitor` — разные Java-классы. Нельзя объединять их только из-за одинакового слова `Capacitor` в названии.

---

# 13. Coolant Rod

Класс:

`ItemReactorCoolant`

Источник:

`src/main/java/com/denfop/items/reactors/ItemReactorCoolant.java`

В baseline существуют только три варианта:

| ID | Level | Max Damage | Heat Damage |
|---|---:|---:|---:|
| `coolant` | 1 | 100000 | 2 |
| `adv_coolant` | 2 | 250000 | 4 |
| `imp_coolant` | 3 | 500000 | 7 |

Варианта `per_coolant` нет.

Поведение:

```text
type              = COOLANT_ROD
heat              = 0
energyProduction  = 0
heatRemovePercent = 0.5
damageFromHeat    = heat parameter
autoRepair        = 0
repairOther       = 0
updatableItem     = true
canExtractHeat    = true
```

Дополнительная механика:

```text
needFill(stack)
fill(stack)
```

Она работает через durability компонента.

---

# 14. Neutron Protector

Класс:

`ItemNeutronProtector`

Зарегистрированные компоненты:

```text
neutron_protector
adv_neutron_protector
imp_neutron_protector
per_neutron_protector
```

Max Damage:

| ID | Level | Max Damage |
|---|---:|---:|
| `neutron_protector` | 1 | 14400 |
| `adv_neutron_protector` | 2 | 43200 |
| `imp_neutron_protector` | 3 | 64800 |
| `per_neutron_protector` | 4 | 86400 |

Исходные выражения:

```text
3600 × 4
7200 × 6
10800 × 6
14400 × 6
```

Полное поведение необходимо дополнительно зафиксировать по `ItemNeutronProtector.java`, `LogicComponent.java` и `LogicReactor.java`.

---

# 15. Параметры, зависящие от реактора

Некоторые свойства компонентов вычисляются не только по собственным полям.

Примеры:

```text
reactor.getModuleVent()
reactor.getModuleExchanger()
reactor.getModuleComponentVent()
```

Поэтому нельзя строить окончательную модель только через статическое поле `heatRemovePercent`.

Правильная архитектура:

```ts
interface ReactorContext {
    getModuleVent(): number;
    getModuleExchanger(): number;
    getModuleComponentVent(): number;
}
```

а компонент получает контекст:

```ts
component.getHeatRemovePercent(context);
component.getAutoRepair(context);
component.getRepairOther(context);
component.getEnergyProduction(context);
```

---

# 16. Предлагаемая TypeScript-модель

## ComponentType

```ts
type ComponentType =
    | 'ROD'
    | 'PLATE'
    | 'CAPACITOR'
    | 'HEAT_EXCHANGER'
    | 'HEAT_SINK'
    | 'COOLANT_ROD'
    | 'ENERGY_COUPLER'
    | 'NEUTRON_PROTECTOR';
```

## ComponentDefinition

```ts
interface ComponentDefinition {
    id: string;
    type: ComponentType;
    level: number;

    maxDamage: number;

    heat: number;
    energyProduction: number;
    radiation: number;

    heatRemovePercent: number;
    damageFromHeat: number;

    autoRepair: number;
    repairOther: number;

    updatable: boolean;
    canExtractHeat: boolean;
}
```

Для динамических параметров окончательные значения вычисляются через `ReactorContext`.

---

# 17. Разделение данных и логики

В симуляторе необходимо разделить:

```text
ComponentDefinition
    ↓
Статические данные компонента

ReactorContext
    ↓
Модификаторы конкретного реактора

ComponentLogic
    ↓
Формулы поведения компонента

ReactorState
    ↓
Текущее состояние компонента

Simulation
    ↓
Порядок выполнения tick
```

---

# 18. Правила переноса

## Не изменять значения

Если baseline содержит:

```text
0.75
```

нельзя без оснований заменять его на другое значение.

## Не округлять

Особенно осторожно обращаться с:

- `double`;
- коэффициентами;
- множителями;
- радиацией.

## Не добавлять отсутствующие компоненты

Например, `per_coolant` не существует в baseline.

## Не объединять разные Java-классы

Особенно:

```text
ItemReactorVent
ItemComponentVent
```

## Не переносить Minecraft-зависимости в simulation core

`ItemStack`, `World`, GUI и регистрацию модели не должны попадать в независимый расчётный движок.

---

# 19. Что ещё необходимо исследовать

Состав компонентов зафиксирован, но физика ещё не полностью исследована.

Необходимо отдельно изучить:

- [ ] все значения `Config.*`, используемые топливными стержнями;
- [ ] `ItemReactorCapacitor.java`;
- [ ] `ItemNeutronProtector.java`;
- [ ] `ItemReactorPlate.java` в контексте `LogicComponent`;
- [ ] `ItemReactorHeatExchanger.java` в контексте `LogicComponent`;
- [ ] `ItemEnergyCoupler.java` в контексте `LogicReactor`;
- [ ] `ItemReactorCoolant.java` в контексте `LogicComponent`;
- [ ] все `getModule*()` у `IAdvReactor`;
- [ ] все множители heat;
- [ ] все множители output;
- [ ] все множители damage;
- [ ] все множители heat rod;
- [ ] полный порядок обработки соседей;
- [ ] полный порядок изменения heat;
- [ ] полный порядок изменения damage;
- [ ] механику durability;
- [ ] механику разрушения;
- [ ] механику ремонта;
- [ ] механику radiation.

---

# 20. Эталонные тесты компонентов

Для каждого логического типа должен существовать минимум один unit test.

## Fuel Rod

Проверить:

- energy production;
- heat;
- radiation;
- level;
- damage from heat;
- cells.

## Plate

Проверить:

- level;
- heat removal;
- damage from heat;
- отсутствие собственного heat;
- отсутствие energy production.

## Reactor Vent

Проверить:

- heat removal;
- auto repair;
- damage from heat;
- durability.

## Component Vent

Проверить:

- heat removal;
- repair other;
- durability.

## Heat Exchanger

Проверить:

- heat removal;
- damage from heat;
- durability.

## Coolant Rod

Проверить:

- heat removal;
- damage from heat;
- durability;
- `needFill`;
- `fill`.

## Energy Coupler

Проверить:

- energy production;
- durability;
- поведение `damageItem`.

## Capacitor

Проверить:

- level;
- durability;
- параметры конкретного уровня.

## Neutron Protector

Проверить:

- level;
- durability;
- радиационное поведение;
- взаимодействие с соседями.

---

# 21. Golden Tests

Для каждого важного компонента необходимо иметь эталонный тест.

Схема:

```text
Industrial Upgrade baseline
          ↓
      expected.json
          ↑
          │
    сравнение результатов
          │
          ↓
TypeScript simulator
          ↓
       actual.json
```

Критерий:

```text
expected === actual
```

Используется только baseline:

```text
16b7fb9cfb7fbb3171e35a63532ebc0a112f665c
```

---

# 22. Критерий завершения исследования компонентов

## Зафиксировано

- [x] определены логические типы;
- [x] определены Java-классы;
- [x] определены конкретные компоненты;
- [x] зафиксировано количество вариантов;
- [x] зафиксированы базовые параметры известных компонентов;
- [x] зафиксированы особые случаи;
- [x] зафиксирован контракт `IReactorItem`;
- [x] определены основные зависимости от `IAdvReactor`.

## Ещё не завершено

- [ ] зафиксированы все значения `Config`;
- [ ] зафиксированы все формулы взаимодействия;
- [ ] созданы эталонные тесты;
- [ ] подтверждено поведение всех компонентов на уровне `LogicComponent`/`LogicReactor`.

Таким образом, на текущем этапе зафиксирован **состав компонентов**, но не закончено полное исследование их физики.

---

# 23. Источники

Industrial Upgrade:

https://github.com/ZelGimi/industrialupgrade

Baseline commit:

https://github.com/ZelGimi/industrialupgrade/commit/16b7fb9cfb7fbb3171e35a63532ebc0a112f665c

Ключевые исходники:

```text
src/main/java/com/denfop/api/reactors/EnumTypeComponent.java
src/main/java/com/denfop/api/reactors/IReactorItem.java
src/main/java/com/denfop/api/reactors/IAdvReactor.java
src/main/java/com/denfop/api/reactors/LogicComponent.java
src/main/java/com/denfop/api/reactors/LogicReactor.java

src/main/java/com/denfop/items/reactors/ItemBaseRod.java
src/main/java/com/denfop/items/reactors/ItemReactorPlate.java
src/main/java/com/denfop/items/reactors/ItemReactorVent.java
src/main/java/com/denfop/items/reactors/ItemComponentVent.java
src/main/java/com/denfop/items/reactors/ItemReactorHeatExchanger.java
src/main/java/com/denfop/items/reactors/ItemReactorCoolant.java
src/main/java/com/denfop/items/reactors/ItemEnergyCoupler.java
src/main/java/com/denfop/items/reactors/ItemReactorCapacitor.java
src/main/java/com/denfop/items/reactors/ItemNeutronProtector.java

src/main/java/com/denfop/IUItem.java
src/main/java/com/denfop/Config.java
```

---

# 24. Следующий этап

После фиксации компонентов следующий этап разработки:

1. Исследовать все параметры `Config`, используемые компонентами.
2. Исследовать полные формулы компонентов.
3. Исследовать `LogicComponent`.
4. Исследовать `LogicReactor`.
5. Создать golden tests.
6. После этого реализовать компоненты в TypeScript.
