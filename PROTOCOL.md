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
- v31.36: همه‌چیز با AES-256-GCM رمز می‌شه. کلید ۲۵۶ بیتی فقط از راه بلوتوث (`pair`) به برنامه داده می‌شه. کد اشتراک برای گوشی دوم: `EBX1-<tid>-<کلید>`
- `<tid>` = ۸ بایت اول SHA-256("eyebox-topic" + کلید). هر پیام: nonce(12) | متن رمزشده | tag(16). دستورها: `<زمان ms>|<دستور>` (قدیمی‌تر از ۲ دقیقه یا تکراری رد می‌شه)
- تاپیک‌ها زیر `eyebox/<tid>/` (همه رمزشده، جز `status`):

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
سرور عمومیه، ولی هر پیام با کلید خود دستگاه رمز و امضا شده؛ بدون کلید نه می‌شه خوند نه دستور داد. کد اشتراک (EBX1-…) رو مثل رمز نگه دار.

## تغییرهای v32.2 / v32.3 (اپ 0.4)
- `key up|down|ok|back|home [n]` (v32.2): کنترل دستگاه از گوشی، روی بلوتوث، وای‌فای و اینترنت یکی است. جواب `ok key …` بی‌صداست. اپ با نگه داشتن ▲/▼ هر ۱۶۰ms یکی می‌فرستد و بیشتر از ۳ دستور در راه نگه نمی‌دارد.
- `page` در liveJson: اسم صفحه‌ی فعلی (مثل `RPM`، `ENGINE OVERVIEW`، `MAIN`، اسم پوشه). برای «آینه‌ی صفحه» و بالای صفحه‌ی کنترل استفاده می‌شود.
- `set fuel 0..8` (v32.3): بنزین باک، مثل صفحه‌ی SET FUEL LEVEL. `cfg.fuel8` = مرحله‌ی فعلی (۱- یعنی تنظیم نشده).
- liveJson (v32.3): `knock` (تعداد ناک در این سفر)، `knockNow` (۲ ثانیه‌ی اخیر)، `fuelL` (لیتر تخمینی)، `fuel8`.
- مسیر (قالب `EYEBOXNAV1`، همان صفحه‌ی /map دستگاه):
  - بلوتوث: `navb`، بعد هر خط با `navl `، بعد `nave`. اپ متن را تیکه‌های ۱۸۰ بایتی می‌فرستد. جواب یک خط ساده است: `nav ok` یا `nav err …`.
  - وای‌فای: `POST /nav`.
  - اینترنت: `<base>/nav` = AES-GCM(`"<ms>|EYEBOXNAV1…"`)، فقط از v32.3 به بعد (قبلاً متن ساده بود و قبول نمی‌شود). جواب `{"reply":"nav ok"}` روی `/x` می‌آید.
- ترتیب اتصال در اپ: بلوتوث (اگر قبلاً اجازه داده شده، خودش بی‌صدا وصل می‌شود)، بعد وای‌فای (فقط وقتی اپ از آدرس http باز شده؛ مرورگر از صفحه‌ی https به http دستگاه اجازه نمی‌دهد)، بعد اینترنت.
