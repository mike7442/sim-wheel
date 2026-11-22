# CHECKLIST — одна страница

Распечатать/держать открытым. Галочки ставить по факту, не «на глаз».

## Перед первым включением

- [ ] колесо зажато, рукава/провода вне плоскости вращения
- [ ] БП выставлен, лимит тока выставлен, полярность проверена мультиметром
- [ ] предохранитель 5 А в плюсовом проводе у БП
- [ ] E-stop (NC) в разрыве плюса или выключатель на входе БП
- [ ] предзаряд (резистор/XT90-S) на 42 В
- [ ] земля ST-Link соединена с землёй платы
- [ ] PinFinder — только 24 В, лимит 2 А

## Софт

- [ ] `python -V` → 3.11.x
- [ ] `python -c "import serial, pyvjoy"` → без ошибок
- [ ] `pyvjoy.VJoyDevice(1).ffb_supported()` → `True`
- [ ] `pyocd list` → ST-Link виден
- [ ] `C:\tools\arm-14.2\bin\arm-none-eabi-gcc.exe --version` → 14.2.1
- [ ] `make --version`, `bash --version` работают
- [ ] `pio --version`
- [ ] vJoyConf: устройство 1, ось X, **Enable FFB**
- [ ] `vendor\ALIFE-MM32` склонирован (`fetch_vendor.ps1`)

## Плата

- [ ] `pyocd reset -t mm32spin05pf` проходит
- [ ] `mm32flash.py` пишет и верифицирует
- [ ] `persist.py --show` → `version.protocol 2`, `heartbeat.errors 0`
- [ ] `snapshot.py --diff config\config_snapshot.json` → `0 of 113`
- [ ] `drivemode 10`, `commutation 2`, `control_mode 1`

## Связь

- [ ] loopback 17↔16 пройден, перемычка **снята**
- [ ] `serial_probe.py COM4 --send "ping"` → эхо
- [ ] `--boot-delay 4` используется везде
- [ ] один процесс на vJoy

## Мотор

- [ ] `spin.py --vel 0.5` → ровное вращение, `errors 0`
- [ ] `spin.py --vel -0.5` → ровное вращение назад
- [ ] `torque.py --check` → `kt 0.342`, `commutation 2`
- [ ] `torque.py --torque 0.1 --torque -0.1` → `I_q` ≈ ±0.29 А
- [ ] момент симметричен, `errors 0`

## 42 В

- [ ] `vbus` ≈ 42.0
- [ ] `batfull` 41900, trip ≈ 44.9
- [ ] `errors 0` после включения
- [ ] нагрев платы/мотора через 2–3 минуты — тёплый, не горячий

## Игра

- [ ] `vjoy_wheel.py` → ось X ходит в `joy.cpl`
- [ ] в игре руль назначен на ось X, калибровка пройдена
- [ ] FFB в игре включён
- [ ] `vjoy_ffb.py --dry-run` → `ffb` меняется в заезде
- [ ] `vjoy_ffb.py --max-torque 0.2` → момент есть, знак правильный
- [ ] `--max-torque 0.4` → стабильно, без звона

## После перерыва (5 минут)

```powershell
python scripts\persist.py --boot-delay 4 --show
python scripts\snapshot.py --boot-delay 4 --diff config\config_snapshot.json
python scripts\torque.py --boot-delay 4 --check
python scripts\vjoy_ffb.py --dry-run --boot-delay 4 --timeout 1.0
```

- [ ] `drivemode 10`, `vbus` ~42, `errors 0`
- [ ] `0 of 113`
- [ ] `ffb` реагирует на игру
- [ ] только после этого — момент

## Стоп-сигналы

Немедленно снять питание, если:

- [ ] дым, запах, щелчки
- [ ] разъём/провод горячий
- [ ] колесо дёргается само по себе
- [ ] `errors` не сбрасывается
- [ ] момент идёт «в стену» дольше пары секунд
