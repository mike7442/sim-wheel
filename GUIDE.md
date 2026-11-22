# GUIDE — от чистого ноутбука до FFB в игре

Это пошаговая инструкция для **повторения** уже собранного проекта. Железо
собрано; задача — заново поднять софт и убедиться, что всё работает.

Читать по порядку. Каждый шаг заканчивается **признаком успеха**. Если признака
нет — не идти дальше, а открыть [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

Все команды — из **корня репозитория** (`C:\wheel\sim-wheel`), в **PowerShell**,
если не написано иное. Сборка прошивки — только в **Git Bash**.

Обозначения:

- `COM4` — номер порта твоего ESP32. Везде замени на свой.
- `…>` — приглашение PowerShell, не вводить.

---

## 0. Что вообще происходит

Четыре независимых слоя, и их надо поднимать именно в этом порядке:

| Слой | Что проверяем | Чем |
|---|---|---|
| 1. Плата | чип виден, прошивка льётся | ST-LINK + `pyocd` + `mm32flash.py` |
| 2. Связь | PC → ESP32 → плата, ответы есть | `pf.py`, `persist.py` |
| 3. Мотор | крутится, держит момент | `spin.py`, `torque.py` |
| 4. Игра | ось и FFB ходят в обе стороны | `vjoy_wheel.py`, `vjoy_ffb.py` |

Правило: **не переходить на следующий слой, пока текущий не даёт стабильный
результат.** 90 % времени теряется на попытках «а может, дело в игре», когда
на самом деле не работает UART.

---

## 1. Что должно быть в руках

### 1.1 Железо

| Узел | Требование | Проверка |
|---|---|---|
| Мотор-колесо | BLDC с Холлами, 15 пар полюсов (30 обмоток) | 3 фазы + 5 проводов Холлов |
| Плата | гироскутер Gen2.x на **MM32SPIN05PF**, с шунтами фаз | надпись на чипе, 2 шунта рядом с фазами |
| ESP32 | **ESP32-S3** с native USB (не CH340/CP2102) | на плате USB идёт прямо в чип |
| ST-LINK | V2 (клон подходит) | 4 провода: 3.3V, SWDIO, SWCLK, GND |
| БП | 24 В с лимитом тока (стенд), 42 В ≥ 5 А (рабочий) | мультиметр |
| Предохранитель | 5 А T, керамика 5×20, 125 В DC | — |
| E-stop | грибок NC, или выключатель на вход БП | прозвонка |
| Провода | 14–16 AWG на силовую часть, витая пара на UART | — |

### 1.2 Что должно быть уже сделано руками

- мотор закреплён, вал соединён с колонкой/ободом;
- Холлы и фазы подключены к плате;
- UART платы (TX PB6 / RX PB4) выведен на ESP32;
- ESP32 воткнут в USB ноутбука;
- силовая часть собрана по схеме из [SAFETY.md](SAFETY.md) §2.2.

Если плата новая или её flash стёрт — понадобится PinFinder (шаг 6). Если плата
та же, что работала, конфиг в ней уже есть, и шаг 6 можно пропустить.

---

## 2. Софт на чистом ноутбуке

Ставим один раз. Версии важны: на других версиях мы уже ловили отказы.

### 2.1 Python 3.11

1. https://www.python.org/downloads/windows/ → Python 3.11.x (64-bit).
2. При установке **обязательно** галочка **Add python.exe to PATH**.
3. Проверка:

```powershell
python -V
```

Ожидается `Python 3.11.x`.

### 2.2 Пакеты Python

```powershell
python -m pip install --upgrade pip
python -m pip install pyserial pyvjoyffb platformio pyocd
```

Проверка:

```powershell
python -c "import serial, pyvjoy; print(serial.__version__); print(pyvjoy.VJoyDevice(1).ffb_supported())"
```

- `pyserial` — COM-порт.
- **`pyvjoyffb`** — это форк `pyvjoy` **с поддержкой FFB** (модуль называется
  `pyvjoy`). Обычный `pyvjoy` из PyPI не умеет callback'и силы — не путать.
- `platformio` — прошивка ESP32.
- `pyocd` — только для `reset` по SWD; **прошивать им MM32 нельзя** (нет
  алгоритма flash).

### 2.3 Git и make

```powershell
winget install --id Git.Git -e
winget install --id ezwinports.make -e
```

После установки **закрыть и открыть терминал**. Проверка:

```powershell
git --version
make --version
bash --version
```

`bash` нужен из Git for Windows (Git Bash): Makefile прошивки использует
`sh`, `tr`, `cmp`.

### 2.4 Arm GNU Toolchain 14.2

Скачать **Arm GNU Toolchain 14.2.Rel1**, Windows (mingw-w64-i686), target
`arm-none-eabi`:

https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads

Распаковать, например в `C:\tools\arm-14.2`. Проверка:

```powershell
C:\tools\arm-14.2\bin\arm-none-eabi-gcc.exe --version
```

Ожидается `14.2.1 20241119`.

> **Почему не 12.2.1:** прошивка не влезает в 32 КБ, переполнение +580 байт.
> Это проверено, не гипотеза.

### 2.5 ST-LINK

Поставить драйвер ST-LINK (STSW-LINK009) с сайта ST. Проверка:

```powershell
pyocd list
```

Должен появиться `ST-Link/V2` с UID.

Если `pyocd` не видит программатор, а ST-Link Utility видит — поставить
WinUSB-драйвер через **Zadig**: Options → List All Devices → ST-Link → WinUSB.

### 2.6 vJoy (форк с FFB, обязательно)

Нужен именно **vJoy 2.2.2.0 (Fully Signed Fork)** — обычный vJoy на Windows 11
не проходит проверку подписи:

https://vjoy.pro/download/#win11-signed-track

После установки:

1. Открыть **vJoyConf**.
2. Устройство **1**: включить ось **X** (остальные по желанию), **Enable FFB**.
3. Кнопки можно оставить 0.
4. Apply.

Проверка (уже делали в 2.2):

```powershell
python -c "import pyvjoy; j=pyvjoy.VJoyDevice(1); print(j.ffb_supported())"
```

Ожидается `True`.

> Если `False` — FFB не включён в vJoyConf для этого устройства.

### 2.7 PlatformIO

Ставится вместе с пакетами (`pio` в PATH). Проверка:

```powershell
pio --version
```

Первый `pio run` сам скачает платформу ESP32 и фреймворк Arduino (несколько
минут, нужен интернет).

---

## 3. Забрать проект

```powershell
cd C:\
git clone https://github.com/mike7442/sim-wheel.git
cd sim-wheel
```

Сторонняя прошивка платы в репозиторий не входит (она большая и чужая).
Подтянуть её на закреплённом коммите:

```powershell
powershell -ExecutionPolicy Bypass -File scripts\fetch_vendor.ps1
```

Ожидается:

```
vendor at 68f880d47947a9cfaf3fc4fe6ca1fca5e824e271
```

Проверка структуры:

```powershell
Test-Path vendor\ALIFE-MM32\HoverBoardMindMotion\Makefile
Test-Path vendor\ALIFE-MM32\tools\cansimple_uart.py
```

Оба — `True`.

### 3.1 Какой у меня COM-порт

1. Воткнуть ESP32 в USB.
2. Диспетчер устройств → **Порты (COM и LPT)**.
3. Найти `USB Serial Device (COMx)` / `USB JTAG/serial debug unit (COMx)`.

Или:

```powershell
Get-CimInstance Win32_PnPEntity | Where-Object { $_.Name -match 'COM\d+' } | Select-Object Name
```

Дальше в командах подставляй свой номер вместо `COM4`.

---

## 4. Стенд 24 В: плата жива

> Перед этим: колесо зажато, БП 24 В, **ограничение тока 2 А**, БП выключен.

### 4.1 Подать питание

1. Выставить на БП 24.0 В, лимит 2 А.
2. Подключить плату, включить её штатно (кнопка/защёлка гироскутера).
3. Признак жизни: зуммер пищит один раз, потребление в покое десятки мА.

### 4.2 SWD

```powershell
pyocd list
pyocd list --targets | findstr /i mm32
pyocd reset -t mm32spin05pf
```

Признак успеха: `reset` проходит без ошибки, плата перезагружается (писк).

Если `mm32spin05pf` не найден в списке targets — обновить pyocd:

```powershell
python -m pip install --upgrade pyocd
```

### 4.3 Прошить PinFinder

```powershell
python vendor\ALIFE-MM32\tools\mm32flash.py firmware\pinfinder-2026-11-22.bin --backup my-original.bin
```

Ожидается `programmed and verified 31744 bytes`, затем `soft start`.

- `--backup my-original.bin` сохранит текущее содержимое flash (32 КБ) — на
  всякий случай. В репозитории уже лежит `firmware/before-original.bin` —
  дамп этой платы до всех изменений.
- **`pyocd flash` для MM32 не работает.** Только `mm32flash.py`: он стирает
  через регистры FLASH и пишет RAM-рутиной.

Признак успеха: верификация прошла, плата отвечает на UART (шаг 5).

---

## 5. ESP32-мост

Мост — тупой: байт из USB → байт в UART1 и обратно. Никакого протокола.

### 5.1 Прошить мост

```powershell
cd esp32\bridge
pio run -t upload
cd ..\..
```

- Плата: `esp32-s3-devkitc-1`, Arduino, native USB CDC.
- Если PlatformIO не находит порт — зажать **BOOT**, воткнуть USB, отпустить.
- Порт для загрузки — тот же COM, что потом использует Python.

### 5.2 Loopback (проверка моста без платы MM32)

1. Соединить проводом **GPIO17 ↔ GPIO16** на ESP32.
2. Выполнить:

```powershell
python scripts\serial_probe.py COM4 --send "ping" --seconds 3
```

Признак успеха: в выводе появится `ping` (байты вернулись через UART1).

3. **Снять перемычку.** Если её забыть, плата MM32 не получит команды.

### 5.3 Почему везде `--boot-delay`

Открытие COM-порта дёргает DTR/RTS, и ESP32-S3 **перезагружается**. Первые
байты теряются. Поэтому во всех скриптах есть `--boot-delay`:

- `2` секунды — обычный старт;
- `4` секунды — native USB после прошивки/перезагрузки.

Если скрипт пишет «no answer» — первым делом увеличить `--boot-delay`.

---

## 6. PinFinder: определить пины (только для новой/стёртой платы)

> Если плата уже работала и в ней лежит рабочий конфиг — **пропустить шаг 6**,
> перейти к 7. Проверить, что конфиг есть, можно так:
>
> ```powershell
> python scripts\pf.py --boot-delay 4 info
> ```
>
> Если в ответе есть `tx PB6 / rx PB4`, `hall 2`, `battery ...` — конфиг на месте.

PinFinder — прошивка, которая сама находит, к каким пинам подключены Холлы,
шунты и делитель батареи. Она **крутит мотор open-loop**, поэтому:

- только БП 24 В с лимитом **2 А**;
- колесо зажато;
- никогда не батарея.

### 6.1 Связь

```powershell
python scripts\pf.py --boot-delay 4 info
```

Ожидается баннер PinFinder 2.0, `MM32SPIN05`, напряжение батареи, `tx/rx`.

### 6.2 Холлы

```powershell
python scripts\pf.py --boot-delay 4 "detect hall"
```

Ожидается (наш результат):

```
A=PC14  B=PC15  C=PC13  invert=1  timer=TIM2  lead90=6
```

Записать. Если получилось иначе — это не ошибка, у платы может быть другой
layout; важно, чтобы PinFinder сам проверил вращение.

### 6.3 Батарея

```powershell
python scripts\pf.py --boot-delay 4 "detect vbat"
```

PinFinder попросит две точки напряжения. Выставить на БП **24.0 В**, подтвердить,
затем **20.0 В**, подтвердить. Результат у нас: `vbat=PB1`, `vbatgain=29985`.

### 6.4 Ток

```powershell
python scripts\pf.py --boot-delay 4 "detect current"
```

Мотор коротко дёргается. Результат у нас: фазы `PA2` (B) / `PA6` (C),
шина `PA7`, `iphasecfg=1`, шунт ~3 мОм.

> Если `detect current` говорит, что датчиков нет — **FOC и момент на этой
> плате невозможны**. Дальше идти нет смысла, нужна другая плата.

### 6.5 Сохранить и забрать конфиг

```powershell
python scripts\pf.py --boot-delay 4 save
python scripts\pf.py --boot-delay 4 --export config\board.json
```

`save` пишет конфиг в отдельную страницу flash (0x08007C00) — она не стирается
при прошивке основной программы.

Признак успеха: после снятия/подачи питания `info` показывает те же пины.

---

## 7. Основная прошивка

Это ALIFE `Hoverboard-Firmware-Hack-Gen2.x-MM32`: FOC, torque/position,
CANSimple поверх UART, конфиг в flash.

### 7.1 Собрать

Только в **Git Bash** (нужны `sh`, `tr`, `cmp`):

```bash
cd /c/wheel/sim-wheel/vendor/ALIFE-MM32/HoverBoardMindMotion
make MCU=SPIN05 CROSS="/c/tools/arm-14.2/bin/arm-none-eabi-" clean both
```

Путь к тулчейну подставить свой. Ожидаемый результат:

```
build/spin05/hoverboard.bin   31648 bytes
build/spin05/pinfinder.bin    31744 bytes
```

Размеры — эталон. Если `hoverboard.bin` заметно больше 31648 — почти наверняка
взят GCC 12, а не 14.2.

### 7.2 Прошить

```powershell
python vendor\ALIFE-MM32\tools\mm32flash.py vendor\ALIFE-MM32\HoverBoardMindMotion\build\spin05\hoverboard.bin
```

Или, если сборки нет, готовым проверенным бинарником:

```powershell
python vendor\ALIFE-MM32\tools\mm32flash.py firmware\hoverboard-2026-11-22.bin
```

Прошивка занимает 31648 байт и **не задевает** страницу конфига.

Признак успеха: после включения плата отвечает на UART:

```powershell
python scripts\persist.py --boot-delay 4 --show
```

Ожидается `version {'protocol': 2, 'hw': '0.0.0', 'fw': '2.0.0'}` и
`heartbeat {'errors': 0, 'state': 1, ...}`.

---

## 8. Первое вращение по UART

Колесо зажато или свободно, но рядом ничего лишнего. Момент ещё не включаем.

```powershell
python scripts\spin.py --boot-delay 4 --vel 0.5 --seconds 4
```

Ожидается: колесо плавно едет вперёд ~0.5 об/с (28 об/мин) 4 секунды, потом
останавливается; в выводе `errors=0`, в конце `state=1` (IDLE).

```powershell
python scripts\spin.py --boot-delay 4 --vel -0.5 --seconds 4
```

Назад. Если крутится рывками или гудит «не в фазе» — вернуться к шагу 6
(порядок Холлов/инверсия).

> **Почему скрипт шлёт скорость каждые 250 мс:** в конфиге `timeout` (сейчас
> 400 мс). Без keep-alive прошивка обнуляет задание. Это не баг, а защита.

---

## 9. Восстановить рабочий конфиг

Одна команда ставит все 113 слов конфига в проверенное состояние: FOC,
torque, `batfull` под 42 В, `timeout` 400 мс, лимиты, пины.

```powershell
python scripts\snapshot.py --boot-delay 4 --restore config\config_snapshot.json --flash
```

Ожидается `restored 113 words`, затем `written to flash (no reset)`.

**Обязательно** снять и подать питание (запись в flash без ресета — power latch
остаётся включённым), затем:

```powershell
python scripts\snapshot.py --boot-delay 4 --diff config\config_snapshot.json
```

Признак успеха: `changed 0 of 113`.

Если не 0 — повторить restore, проверить питание, посмотреть, какие именно
слова не легли (`--diff` их печатает).

> Снапшот рассчитан на 42 В. На 24 В он тоже безопасен: `batfull=41900`
> просто разрешает рекуперацию, а `batempty=18000` не мешает.

Проверить ключевое:

```powershell
python scripts\persist.py --boot-delay 4 --show
```

Должно быть: `drivemode 10`, `commutation 2`, `control_mode 1`, `timeout 400`,
`batfull 41900`, `vel_limit 5.0`, `errors 0`.

---

## 10. Переход на 42 В

> **Сначала [SAFETY.md](SAFETY.md) §2.2.** Предохранитель и кнопка — до первого
> включения на 42 В, а не «потом».

### 10.1 Железо

```
БП+ → предохранитель 5 А → E-stop (NC) → предзаряд → плата+
БП− ──────────────────────────────────────────────── плата−
```

- Выставить 42.0 В **до** подключения, проверить мультиметром и полярность.
- Ограничение тока на БП — выставить по возможности (5–9 А).
- Предзаряд: резистор 10–50 Ом на 1–2 с, потом основной контакт.

### 10.2 Проверка

```powershell
python scripts\persist.py --boot-delay 4 --show
```

Ожидается:

- `vbus` ≈ 42.0 (у нас 41.98–42.02);
- `batfull` 41900;
- `dc_bus_overvoltage_trip_level` ≈ 44.9;
- `errors 0`.

Если `vbus` показывает ~24 В — плата не переключилась на новый БП.
Если `errors` содержит `DC_BUS_OVER_VOLTAGE` — `batfull` слишком низкий для
этого напряжения (см. [TROUBLESHOOTING.md](TROUBLESHOOTING.md)).

---

## 11. Момент (FOC torque)

С этого момента колесо **толкает**. Держать рукой, не отпускать.

### 11.1 Только чтение

```powershell
python scripts\torque.py --boot-delay 4 --check
```

Показывает `kt` (0.342 Нм/А), `commutation 2`, `control_mode 1`, лимит тока,
текущий `I_q`. Ничего не армит.

### 11.2 Шаги момента

```powershell
python scripts\torque.py --boot-delay 4 --torque 0.1 --torque -0.1 --seconds 2
```

Ожидается: `I_q` измеренный ≈ заданный. Арифметика: 0.1 Нм → 0.29 А,
0.2 Нм → 0.59 А. У нас сходилось в пределах ±15 %.

```powershell
python scripts\torque.py --boot-delay 4 --torque 0.2 --torque -0.2 --seconds 2
```

Признак успеха: момент в обе стороны, симметрично, `errors=0`, шина почти не
потребляет (колесо зажато — мощность не тратится).

### 11.3 Убедиться, что режим в flash

```powershell
python scripts\persist.py --boot-delay 4 --show
```

`drivemode 10`. Снять/подать питание, снова `--show`: должно остаться `10`.
Если сбросилось — конфиг не записался, повторить шаг 9.

---

## 12. Ось в игру (vJoy)

Пока без момента: игра должна увидеть руль как ось.

### 12.1 Запустить

```powershell
python scripts\vjoy_wheel.py --boot-delay 4 --timeout 1.0 --range-deg 900
```

Ожидается поток строк:

```
pos +0.0000  axis 16384  ...  61 Hz
```

`axis` = 16384 в центре, 0…32767 по краям.

### 12.2 Проверить в Windows

```powershell
joy.cpl
```

Свойства vJoy Device → крутить колесо → ползунок **X** должен ходить.

Признак успеха: ось ходит плавно, в центре ~50 %.

Если ось ходит в другую сторону — добавить `--invert`.

### 12.3 Диапазон

`--range-deg` — сколько градусов соответствует полному ходу оси. Пока
механики нет, оставить 900. Когда появится упор — померить реальный ход
(например, 540) и поставить его, иначе игра будет видеть «недокрут».

---

## 13. FFB из Dirt Rally 2.0

### 13.1 Как это работает

DR2 не требует телеметрии. Игра отправляет **DirectInput force feedback** на
устройство vJoy:

- один эффект, тип 1 (**constant force**);
- magnitude `−10000…+10000`;
- обновление каждые **10 мс (100 Гц)**;
- gain 255.

`vjoy_ffb.py` ловит callback pyvjoy, берёт magnitude, нормирует в −1…+1,
умножает на `--max-torque` и шлёт момент на плату. Одновременно он же кормит
игру позицией колеса.

### 13.2 Сначала dry-run (без момента)

```powershell
python scripts\vjoy_ffb.py --dry-run --boot-delay 4 --timeout 1.0
```

Затем:

1. Запустить DR2.
2. Настройки → Управление → назначить руль на **ось X** устройства vJoy.
3. Пройти калибровку руля.
4. Включить **Force Feedback** в настройках игры.
5. Зайти в **заезд** (в меню сила = 0, это нормально).

Признак успеха: в строке `ffb` появляются ненулевые значения, когда крутишь
руль; `pos` и `axis` меняются. У нас `ffb` доходил до −0.992…+0.959.

> Если `ffb` всегда 0.000 — игра не отдаёт эффект. Проверить: FFB включён,
> устройство выбрано, заезд начат, скрипт запущен **до** игры.

### 13.3 Реальный момент

Остановить dry-run (`Ctrl+C`), затем:

```powershell
python scripts\vjoy_ffb.py --max-torque 0.2 --max-iq-a 2.0 --max-bus-a 2.0 --boot-delay 4 --timeout 1.0
```

Порядок: **сначала скрипт, потом игра.** Если перезапустить скрипт — перезапустить
и заезд, иначе игра не пересоздаст эффект.

Ожидается:

| Строка | Значение |
|---|---|
| `ffb` | меняется как в dry-run |
| `tq` | до ±0.2 Нм |
| `iq` | 0.2–0.7 А |
| `bus` | 0.0–0.5 А (шум шунта) |
| колесо | сопротивляется, тянет к центру в повороте |

Если тянет **от** поворота — `--invert-torque`. Если игра крутит руль не в ту
сторону — `--invert-axis`.

### 13.4 Поднять до 0.4

```powershell
python scripts\vjoy_ffb.py --max-torque 0.4 --max-iq-a 3.0 --max-bus-a 3.0 --boot-delay 4 --timeout 1.0
```

У нас на 0.4 Нм `I_q` доходил до ~1.1 А, шина ~0.05 А, `errors=0`.

Выше 0.4 Нм — **только** с установленными предохранителем и кнопкой и после
проверки механики.

---

## 14. Настройка ощущений

| Параметр | По умолчанию | Что делает | Куда крутить |
|---|---|---|---|
| `--max-torque` | 0.3 | Нм на полной величине | главная «сила» |
| `--ffb-deadzone` | 150 | игнор слабых сигналов (из 10000) | больше = меньше дрожи в центре |
| `--ffb-smoothing` | 0.3 | 0 = плавно/медленно, 1 = резко | меньше при звоне |
| `--range-deg` | 900 | градусов на полный ход оси | под реальную механику |
| `--invert-axis` | — | направление оси | если игра крутит наоборот |
| `--invert-torque` | — | направление момента | если тянет от поворота |

### 14.1 Частота и задержка

Цикл сейчас ~61–64 Гц, игра шлёт 100 Гц. Значит:

- задержка ~15 мс;
- часть кадров FFB теряется, применяется последний.

Для первых заездов это нормально. Если ощущается «резина»:

1. поднять baud до 460800 (в конфиге платы `baud=4608` и в мосте);
2. или перенести цикл на ESP32 (он станет мастером UART, PC только шлёт FFB).

### 14.2 Если колесо звенит/раскачивается

Это петля FFB попала в резонанс с задержкой. По порядку:

1. `Ctrl+C`;
2. `--ffb-smoothing 0.15`;
3. `--max-torque` меньше;
4. `--ffb-deadzone` больше;
5. только потом — архитектура (частота).

---

## 15. Если всё сломалось

### 15.1 Плата отвечает, но конфиг не тот

```powershell
python scripts\snapshot.py --boot-delay 4 --restore config\config_snapshot.json --flash
```

Снять/подать питание, `--diff` → `0 of 113`.

### 15.2 Плата не отвечает

1. `python scripts\serial_probe.py COM4 --seconds 5` — есть ли вообще байты.
2. Проверить loopback 17↔16 (шаг 5.2), снять перемычку.
3. `--boot-delay 4`.
4. Проверить, что мост прошит (шаг 5.1), а не просто «COM-порт есть».

### 15.3 Прошивка сломалась

```powershell
python vendor\ALIFE-MM32\tools\mm32flash.py firmware\hoverboard-2026-11-22.bin
python scripts\snapshot.py --boot-delay 4 --restore config\config_snapshot.json --flash
```

### 15.4 Хочу вернуть заводское

```powershell
python vendor\ALIFE-MM32\tools\mm32flash.py firmware\before-original.bin
```

Это оригинальный дамп платы до всех изменений. Конфиг при этом тоже вернётся
к заводскому — дальше шаг 6 (PinFinder) заново.

---

## 16. Что дальше (ещё не сделано)

- механика: рама, колонка, подшипники, механический упор;
- калибровки FOC: `hallang1..6`, `motorr`, `phasel`, `sinekv`;
- точный `torqueconst` вместо умолчания 0.342 Нм/А;
- снижение задержки (baud / ESP32-мастер);
- HID PID на ESP32-S3: ось и FFB без vJoy, как у «взрослых» рулей;
- диагностика редких обрывов связи (`lost the board`) на большом моменте.

---

## 17. Все команды одной страницей

```powershell
# --- один раз ---
python -m pip install pyserial pyvjoyffb platformio pyocd
winget install --id Git.Git -e
winget install --id ezwinports.make -e
# Arm GNU 14.2 -> C:\tools\arm-14.2
# vJoy 2.2.2.0 signed fork + vJoyConf: X, FFB

# --- проект ---
git clone https://github.com/mike7442/sim-wheel.git
cd sim-wheel
powershell -ExecutionPolicy Bypass -File scripts\fetch_vendor.ps1

# --- плата ---
pyocd list
pyocd reset -t mm32spin05pf
python vendor\ALIFE-MM32\tools\mm32flash.py firmware\pinfinder-2026-11-22.bin --backup my-original.bin
python scripts\pf.py --boot-delay 4 info
python scripts\pf.py --boot-delay 4 "detect hall"
python scripts\pf.py --boot-delay 4 "detect vbat"
python scripts\pf.py --boot-delay 4 "detect current"
python scripts\pf.py --boot-delay 4 save
python scripts\pf.py --boot-delay 4 --export config\board.json

# --- сборка и прошивка (Git Bash) ---
# cd vendor/ALIFE-MM32/HoverBoardMindMotion
# make MCU=SPIN05 CROSS="/c/tools/arm-14.2/bin/arm-none-eabi-" clean both

python vendor\ALIFE-MM32\tools\mm32flash.py vendor\ALIFE-MM32\HoverBoardMindMotion\build\spin05\hoverboard.bin

# --- ESP32 мост ---
cd esp32\bridge; pio run -t upload; cd ..\..
python scripts\serial_probe.py COM4 --send "ping" --seconds 3

# --- проверки ---
python scripts\persist.py --boot-delay 4 --show
python scripts\snapshot.py --boot-delay 4 --restore config\config_snapshot.json --flash
python scripts\snapshot.py --boot-delay 4 --diff config\config_snapshot.json
python scripts\spin.py --boot-delay 4 --vel 0.5 --seconds 4
python scripts\torque.py --boot-delay 4 --check
python scripts\torque.py --boot-delay 4 --torque 0.1 --torque -0.1 --seconds 2

# --- игра ---
python scripts\vjoy_wheel.py --boot-delay 4 --timeout 1.0
python scripts\vjoy_ffb.py --dry-run --boot-delay 4 --timeout 1.0
python scripts\vjoy_ffb.py --max-torque 0.2 --max-iq-a 2.0 --max-bus-a 2.0 --boot-delay 4 --timeout 1.0
```
