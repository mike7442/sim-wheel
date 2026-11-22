# REFERENCE — пины, конфиг, протокол, арифметика

Справочник. Не читать целиком; искать нужное.

---

## 1. Плата и распиновка

Плата: гироскутер Gen2.x, **MM32SPIN05PF** (Cortex-M0, 32 КБ flash, 4 КБ RAM).

| Что | Пин | Примечание |
|---|---|---|
| Холл A | **PC14** | `hallinvert=1` |
| Холл B | **PC15** | порядок именно такой; PC13/14/15 без инверсии = «не в фазе» |
| Холл C | **PC13** | |
| Фаза B (шунт) | **PA2** | `iphasecfg=1` |
| Фаза C (шунт) | **PA6** | |
| Шунт шины | **PA7** | `itotaldivider=332` (~3 мОм), `itotalzero=389` |
| Делитель батареи | **PB1** | `vbatgain=29985` |
| UART TX | **PB6** | → RX ESP32 (GPIO16) |
| UART RX | **PB4** | ← TX ESP32 (GPIO17) |
| Зуммер | **PB9** | писк при старте |
| Кнопка | **PB5** | |
| Latch (power hold) | **PB2** | удерживает питание после кнопки |

Идентификация: UID `505FF201F757750000324D4D`, idcode `CC4460B1`, роль master,
node id 0.

> На плате с **другим layout** эти пины могут не совпасть. Тогда — PinFinder
> (`detect hall` / `vbat` / `current`), см. GUIDE §6.

---

## 2. Конфиг: рабочая точка

Полный дамп — `config/config_snapshot.json` (113 rw-слов из 116).
Восстановление: `snapshot.py --restore ... --flash`, затем power cycle.

### 2.1 Ключевые слова

| Слово | Значение | Смысл |
|---|---|---|
| `drivemode` | **10** | FOC + torque |
| `commutation` | 2 | 0 = block, 1 = sine, 2 = FOC |
| `control_mode` | 1 | 0 voltage, 1 torque, 2 velocity, 3 position |
| `timeout` | **400** | мс без команды → задание 0 |
| `batfull` | **41900** | мВ; на 100 мВ ниже шины 42 В |
| `batempty` | 18000 | мВ; для БП. На реальной 10S — 30000 |
| `posrpm` | 65535 → default 300 | `vel_limit` 5 об/с |
| `focimax` | 0 → default 100 | 10 А фазного тока |
| `overspeederr` | 65535 | выключен |
| `vbatgain` | 29985 | делитель батареи |
| `itotaldivider` | 332 | шунт шины |
| `itotalzero` | 389 | ноль шунта |
| `windings` | 30 | 15 пар полюсов |
| `baud` | 1152 | 115200 бод |
| `hallinvert` | 1 | инверсия Холлов |
| `slaveid` | 0 | node id |

### 2.2 ODrive-представления (то, что печатает `persist.py --show`)

| Имя | Значение | Откуда |
|---|---|---|
| `dc_bus_overvoltage_trip_level` | 44.9 В | `batfull + 3000` |
| `dc_bus_undervoltage_trip_level` | 18.0 В | `batempty` |
| `axis0.controller.config.vel_limit` | 5.0 об/с | `posrpm` |
| `axis0.config.motor.current_soft_max` | 10 А | `focimax` |
| `hoverboard.torque_constant_in_use` | 0.3417 Нм/А | `82.7 / sinekv` |

### 2.3 `drivemode` = `commutation * 4 + control`

| | voltage | speed | torque | position |
|---|---|---|---|---|
| block | 0 | 1 | 2 | 3 |
| sine | 4 | 5 | 6 | 7 |
| FOC | 8 | 9 | **10** | 11 |

`set_controller_mode()` сам пересчитывает `drivemode`.

### 2.4 Значения 65535

`65535` (`0xFFFF`) = «не задано», прошивка берёт умолчание. Важные умолчания:

| Слово | Умолчание | Смысл |
|---|---|---|
| `sinekv` | 242 | → `torqueconst = 82.7/242 ≈ 0.342` |
| `focimax` | 100 | 10 А |
| `curkp` / `curki` | 400 мВ/А / 300 В/(А·с) | токовый ПИ |
| `iphasegain` | 515 | ~51.5 мА на отсчёт |
| `hallang1..6` | ±30° | таблица углов Холлов |

---

## 3. Протокол: CANSimple поверх UART

Кадр:

```
0xA5 | node | cmd (bit7 = request) | len (0..8) | payload | CRC-16/XMODEM (LE)
```

CRC считается по `node..payload`. Приёмник ищет `0xA5`, проверяет длину и CRC.

Команды (используемые):

| cmd | Имя | Payload |
|---|---|---|
| 0x00 | GET_VERSION | — |
| 0x01 | HEARTBEAT | errors u32, state u8, ... |
| 0x03 | GET_ERROR | — |
| 0x04 / 0x05 | RXSDO / TXSDO | endpoint id u16 + данные |
| 0x07 | SET_AXIS_STATE | state u32 (1 IDLE, 8 CLOSED_LOOP) |
| 0x09 | GET_ENCODER_ESTIMATES | pos f32, vel f32 (обороты, об/с) |
| 0x0B | SET_CONTROLLER_MODE | control u32, input u32 |
| 0x0D | SET_INPUT_VEL | vel f32 (об/с) |
| 0x0E | SET_INPUT_TORQUE | torque f32 (Нм) |
| 0x14 | GET_IQ | setpoint f32, measured f32 (А) |
| 0x16 | REBOOT | action u32 (1 = save config) |
| 0x17 | GET_BUS_VOLTAGE_CURRENT | V f32, A f32 |

Состояния: `IDLE=1`, `CLOSED_LOOP_CONTROL=8`.

### 3.1 SDO-эндпоинты

Чтение/запись любого слова конфига идёт через SDO по id из
`vendor/ALIFE-MM32/tools/endpoints.json`. Примеры имён:

- `hoverboard.config.drivemode`
- `hoverboard.config.batfull`
- `hoverboard.config.timeout`
- `hoverboard.commutation`
- `axis0.controller.config.control_mode`
- `vbus_voltage`, `ibus`
- `config.dc_bus_overvoltage_trip_level`

`Bus.read(name)` / `Bus.write(name, value)` сами находят id и тип (int/float).

### 3.2 Особенности

- `request()` **сбрасывает входной буфер** перед отправкой. Это важно: иначе
  ответ на предыдущий запрос читается как ответ на текущий.
- `save_configuration()` — это `REBOOT action=1`: прошивка пишет страницу
  конфига, **но не ресетает**. Питание надо снять руками.
- `baud` и `slaveid` при восстановлении пишутся **последними**: они меняют
  то, как плата отвечает.

---

## 4. Мотор и момент

### 4.1 Геометрия

- 30 обмоток → **15 пар полюсов**;
- 3 Холла → 6 состояний на электрический оборот;
- `StepsPerTurn() = 90` шагов на механический оборот;
- разрешение по Холлам ~4°, между шагами — интерполяция (PLL) на скорости.

### 4.2 Момент

```
I_q [A] = torque [Нм] / kt
kt = 0.3417 Нм/А   (умолчание: 82.7 / 242)
```

| Момент | I_q |
|---|---|
| 0.1 Нм | 0.29 А |
| 0.2 Нм | 0.59 А |
| 0.4 Нм | 1.17 А |
| 1.0 Нм | 2.93 А |
| 1.5 Нм | 4.39 А |

`set_input_torque` внутри: `f / TorqueConstant() * 100` (единицы 0.01 А).
В FOC это токовый контур по фазному току; в block — по току шины.

### 4.3 Шунт шины

1 отсчёт АЦП ≈ **0.33 А**. Поэтому `ibus` 0.1–0.55 А на холостом ходу — шум,
а не потребление. Для реальных измерений нужен ток > 1 А.

### 4.4 Мощность

При моменте 1 Нм и 3 об/с механическая мощность ≈ 2π·3·1 ≈ 19 Вт, с потерями
30–50 Вт. БП 42 В × 9 А = 378 Вт — запас большой. Ограничение — не БП, а
нагрев мотора и платы.

### 4.5 Рекуперация

- `vbat > batfull` → рекуперация в шину запрещена, тормозная энергия жжётся
  в обмотках (d-ток);
- `vbat > batfull + 3000` → **coast** + `DC_BUS_OVER_VOLTAGE`;
- БП не принимает ток → `batfull` ставится чуть ниже напряжения БП.

---

## 5. Ошибки (биты heartbeat)

| Бит | Имя | Что значит |
|---|---|---|
| 0x8 | MISSING_ESTIMATE | нет оценки угла/скорости |
| 0x20 | DRV_FAULT | драйвер (OCP/UVLO) |
| 0x100 | DC_BUS_OVER_VOLTAGE | шина выше `batfull+3000` |
| 0x200 | DC_BUS_UNDER_VOLTAGE | шина ниже `batempty` |
| 0x400 | DC_BUS_OVER_CURRENT | ток шины выше лимита |
| 0x800 | DC_BUS_OVER_REGEN_CURRENT | рекуперация выше лимита |
| 0x1000 | CURRENT_LIMIT_VIOLATION | фазный ток выше `focimax` |
| 0x4000 | INVERTER_OVER_TEMP | перегрев |
| 0x8000 | VELOCITY_LIMIT_VIOLATION | скорость выше лимита |
| 0x1000000 | WATCHDOG_TIMER_EXPIRED | таймаут связи |
| 0x2000000 | ESTOP_REQUESTED | ESTOP |
| 0x4000000 | SPINOUT_DETECTED | потеря контроля |

`persist.py` / `torque.py` печатают расшифровку (`error_names`).

---

## 6. FFB: что реально шлют игры

Логи сняты на Arduino-прототипе (`attempt_2/.../22.08/ffb_log_*.txt`),
разобраны по типам отчётов.

### 6.1 Dirt Rally 2.0

| Параметр | Значение |
|---|---|
| Эффект | один, тип **1 (constant force)** |
| Magnitude | −10000…+10000 |
| Период обновления | **10 мс (100 Гц)**, p90 = 11 мс |
| Device gain | 255 |
| Effect gain | 255 |
| Condition/periodic | нет |

То есть весь FFB DR2 — одно число 100 раз в секунду.

### 6.2 ACC

Другой путь: `condX` (condition/spring, тип 9). Поэтому `vjoy_ffb.py` для ACC
не подходит; там нужна модель или отдельная обработка condition-эффектов.

### 6.3 Отчёты pyvjoy, которые надо обрабатывать

| reptype | Что | Действие |
|---|---|---|
| 5 | set constant force | запомнить magnitude для блока |
| 10 | effect operation | `EffectOp=3` (stop) → удалить блок |
| 11 | block free | удалить блок |
| 12 | device control | 2 disable, 3 stop all, 4 reset, 5 pause → обнулить всё |
| 13 | device gain | пересчитать масштаб |

**Важно:** если обрабатывать только тип 5 (как делал прототип), после остановки
эффекта на моторе остаётся последний момент. В `vjoy_ffb.py` это закрыто.

### 6.4 Нормировка

```
raw  = sum(magnitude по активным блокам) * gain / 255     # clamp ±10000
norm = sign(raw) * (|raw| - deadzone) / (10000 - deadzone)  # 0 если |raw| <= deadzone
torque = norm * max_torque
```

---

## 7. Ось vJoy

```
frac  = (pos_turns - centre_turns) / (range_deg / 720)
axis  = 16384 + clamp(frac, -1, 1) * 16384        # 0..32767
```

`range_deg` — полный ход (от упора до упора). `720` = 2 × 360: `range_deg/720`
даёт половину хода в оборотах.

Пример: `range_deg=900` → половина хода 1.25 оборота.

Центр берётся при старте скрипта (`pos_estimate` в этот момент). Поэтому скрипт
запускается, когда руль стоит **в нейтрали**.

---

## 8. Версии и размеры

| Что | Значение |
|---|---|
| Прошивка | ALIFE `Hoverboard-Firmware-Hack-Gen2.x-MM32`, `fw 2.0.0`, protocol 2 |
| Коммит | `68f880d47947a9cfaf3fc4fe6ca1fca5e824e271` (2026-11-18) |
| `hoverboard.bin` | 31648 байт |
| `pinfinder.bin` | 31744 байт |
| Страница конфига | `0x08007C00`, 116 слов (113 rw) |
| GCC | Arm GNU 14.2.Rel1 (14.2.1) |
| vJoy | 2.2.2.0 signed fork |
| pyvjoyffb | 0.4 |
