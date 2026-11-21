# پروتکل EyeBox ↔ برنامه (فریمور v31.35 به بعد)

دو راه، با یه زبان مشترک:

| راه | کی | چطور |
|---|---|---|
| بلوتوث | نزدیک ماشین | سرور BLE روی دستگاه، اسم `EyeBox-XXXX`، سرویس Nordic UART |
| اینترنت | از هر جا | دستگاه با هر وای‌فایی (مودم خونه، هات‌اسپات، مودم ماشین) به سرور MQTT وصل می‌شه |

## بلوتوث
- سرویس `6e400001-b5a3-f393-e0a9-e50e24dcca9e`
  - `6e400003-…` notify: دستگاه ← گوشی (هر پیام یه خط JSON، آخرش `\n`)
  - `6e400002-…` write: گوشی ← دستگاه (هر خط یه دستور)

## اینترنت (MQTT)
- سرور فعلی (عمومی، برای تست): `broker.emqx.io` — دستگاه TCP `1883`، برنامه `wss://broker.emqx.io:8084/mqtt`
- کد اتصال = `EyeBox-XXXX-KKKKKK` (روی دستگاه: Settings ← Phone App؛ برنامه با بلوتوث خودش می‌گیره)
- تاپیک‌ها زیر `eyebox/<کد اتصال>/`:

| تاپیک | جهت | محتوا |
|---|---|---|
| `live` | دستگاه ← | داده‌ی زنده (`liveJson`) هر ۲ ثانیه |
| `x` | دستگاه ← | `cfg` / `data` / `dtc` / `reply` |
| `status` | دستگاه ← | `online` / `offline` (ماندگار) |
| `alert` | دستگاه ← | هشدارها `{"lvl","en","fa","t"}` |
| `cmd` | ← برنامه | دستورها |

## داده‌ی زنده (`liveJson`)
`rpm` `kmh` `cool` `batt` `load` `lph` `gear` `trip` `dtc`(تعداد) `pitch` `roll` `gps` `demo` `ecu`
`p`: {batt, cool, ign, load, tps, map, iat, stft, ltft, o2, fuel, baro, maf, boost(bar), amb, oil, ecuv, cat}
`dev`: باتری داخلی {bat, pct, ext, chg} · `loc`: GPS {fix, lat, lon, kmh, course, alt, sats} · `imu` · `mic`
`guard` `guardHits` `sim` `wifi`(IP) `cloud` `ble` `page` `time` `up` `fw` `id`

## دستورها (گوشی ← دستگاه)
| دستور | کار |
|---|---|
| `cfg` | همه‌ی تنظیمات ← `{"cfg":{…}}` |
| `data` | سفرها، رکوردها، ECU، کدها، وای‌فای‌ها، قطعه‌ها ← `{"data":{…}}` |
| `set <key> <value>` | یه تنظیم (پایین) |
| `pair` | کد اتصال اینترنتی (فقط بلوتوث) |
| `dtc` / `dtcclear` | خوندن / پاک کردن کدها |
| `perf 0\|1\|2` | شتاب / ترمز ۱۰۰ تا ۰ / ۴۰۲ متر |
| `screen on\|off` · `find` · `beep` · `level` | نمایشگر، پیدا کردن ماشین، بوق، تراز شیب‌سنج |
| `go <page>` | صفحه‌ی دستگاه: home, rpm, speed, coolant, battery, codes, guard, compass, map, incline, gmeter, eyes, about … |
| `guard on\|off` · `demo on\|off` · `bright <n>` · `vol <n>` · `sound on\|off` | میان‌برها |
| `clock <unix>` | ساعت دستگاه |
| `wifi add <ssid>\t<pass>` (فقط بلوتوث) · `wifi forget <ssid>` · `portal` | وای‌فای |
| `gearlearn` · `reconnect` · `elmforget` · `restart` · `power off` · `update [install]` | سیستم |

### کلیدهای `set`
`lang`(fa/en) `theme`(0-6) `gstyle`(0-2) `istyle`(0-1) `rotate`(0/5/10/20) `saver`(0-4) `welcome` `demo`
`bday`(5-100) `bnight`(2-100) `bauto` `sound` `vol`(20-100) `spifast` `guard` `cloud` `sms`(شماره) `wname`(متن)
`weight` `tank` `disp` `fcal` `svo` `sva` `svp` `svb` `spdal`

## نکته‌ی امنیتی
سرور عمومی برای تسته: هر کی کد اتصال رو داشته باشه می‌بینه و دستور می‌ده. رمز وای‌فای و کد اتصال فقط از راه بلوتوث داده می‌شن. برای استفاده‌ی واقعی یه سرور شخصی با رمز و TLS لازمه.
