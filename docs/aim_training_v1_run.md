# aim_training_v1 — полная инструкция: как запустить карту

Ранбук «от чистой системы до карты, в которую ты стреляешь». Выполняй по порядку.
Геометрию, энтити и логику собирай по [`aim_training_v1_guide.md`](aim_training_v1_guide.md) — здесь только *что нажимать и куда смотреть, чтобы это запустилось*.

Пути ниже даны относительно папки установки CS2:
```
...\SteamLibrary\steamapps\common\Counter-Strike Global Offensive\
```
(да, папка до сих пор называется `Counter-Strike Global Offensive`).

---

## Шаг 0. Что нужно до старта

| Требование | Зачем |
|-----------|-------|
| Видеокарта с **RTX** (NVIDIA 2060 / AMD 6600XT и выше) | VRAD3 запекает свет на GPU. Без RT компиляция падает на этапе света |
| ~40 ГБ свободного места | CS2 + Workshop Tools |
| Консоль разработчика | Без неё карту не загрузить и не протестировать |

**Включить консоль (один раз):** CS2 → Настройки → Игра → **«Включить консоль разработчика (~)» = Да**. Открывается клавишей `~` (Ё).

---

## Шаг 1. Установить Workshop Tools

1. Steam → Библиотека → **Counter-Strike 2** → ПКМ → **Свойства** → вкладка **DLC**.
2. Включить **Counter-Strike 2 Workshop Tools**. Дождаться скачивания.
3. Запустить CS2: Steam теперь каждый раз предлагает выбрать — **«Играть»** или **«Counter-Strike 2 (Tools)»**. Тебе нужен второй вариант.

---

## Шаг 2. Создать аддон

1. Запусти **Counter-Strike 2 (Tools)** → **Workshop Tools → Launch Tools → Asset Browser**.
2. **Create New Addon** → имя: **`aim_training_v1`** (латиница, без пробелов — это имя потом фигурирует в команде запуска).
3. В Asset Browser нажми иконку **Hammer** (редактор откроется с примонтированным аддоном).

Запомни две папки:
```
content\csgo_addons\aim_training_v1\maps\     <- сюда сохраняешь aim_training_v1.vmap (исходник)
game\csgo_addons\aim_training_v1\maps\        <- сюда компилятор положит aim_training_v1.vpk (готовый билд)
```

> **Одна карта = один аддон.** Не складывай несколько карт в один аддон и не создавай подпапок внутри `maps\` — иначе загрузка и публикация ломаются.

---

## Шаг 3. Собрать карту в Hammer

Открой `File → New` → **сразу `Save As`** в
`content\csgo_addons\aim_training_v1\maps\aim_training_v1.vmap`
(проверь заголовок окна: первый Save As любит сохранять как `untitled_1.vmap`).

Дальше — по разделам основного гайда. Порядок и чекпоинты:

| Чекпоинт | Что сделать | Раздел гайда | Как понять, что готово |
|----------|-------------|--------------|------------------------|
| 1 | Коробка арены: пол, потолок, 4 стены | §2 | Вьюпорт — замкнутый белый бокс 3000×3000×1000 |
| 2 | 10 CT + 10 T спавнов, `info_player_start`, разметка, 2 `func_buyzone` | §3 | У всех спавнов `Z = 32`, Angles `0 90 0` |
| 3 | 20 мишеней `func_button` + 3 `func_movelinear` | §4 | Таблицы координат в §4.1–4.3 |
| 4 | `env_sky` + `light_environment` + 4 `light_omni2` + `env_combined_light_probe_volume` + объём `tools/toolslightmapres` | §5 | Вьюпорт в режиме All Lighting светлый, без чёрных пятен |
| 5 | `game_player_equip` ×2 + `logic_auto` + `point_servercommand` + `trigger_multiple` | §6 | В Outputs у `logic_auto` 20+ строк |
| 6 | Всем мишеням и декору — **Exclude from VIS = Yes** | §7 | — |

**Минимум, чтобы просто запустить и пострелять (MVP на 30 минут):** коробка (6 брашей) + 10 CT-спавнов + 1 `info_player_start` + один ряд из 5 мишеней + `env_sky` + `light_environment` + `env_combined_light_probe_volume` + `logic_auto` с двумя командами `mp_free_armor 2` и `mp_ct_default_primary weapon_ak47`. Остальное добирается потом.

Сохраняйся каждые 5 минут (`Ctrl+S`): Hammer под CS2 умеет падать.

---

## Шаг 4. Скомпилировать

**Клавиша `F9`** (в меню Map этого пункта нет!) → окно **Build Map**.

| Режим | Когда | Время |
|-------|-------|-------|
| **Fast Compile** | Первая проверка геометрии и мишеней. Света не будет — карта тёмная, это нормально | минуты |
| **Full Compile**, Lightmap **1024**, Quality **Standard** | Проверить свет и тени | десятки минут |
| **Full Compile**, Lightmap **4096**, Quality **Final** | Релиз/публикация | час+ |

**Нет видеокарты с RTX:** стадия света (`-bakelighting` / Generate Lightmaps) не запустится — VRAD3 требует аппаратного RT. Карта всё равно собирается и играется: `F9 → Custom Settings` → выключить **Generate Lightmaps** → Build → в игре `mat_fullbright 1`. Подробно, плюс как один раз запечь свет на чужой машине с RTX — [`aim_training_v1_no_rtx.md`](aim_training_v1_no_rtx.md).

Что делать, если компиляция падает — см. §9 основного гайда (VRAD3, драйвер GPU, встройка Ryzen, ручной запуск `resourcecompiler.exe`).

**Проверь результат:** после успешной компиляции должен появиться файл
```
game\csgo_addons\aim_training_v1\maps\aim_training_v1.vpk
```
Нет `.vpk` — карта не соберется ни в одном из способов ниже.

---

## Шаг 5. Запустить карту

### Способ 1 (быстрый, из редактора) — только «посмотреть»

В Hammer есть кнопка загрузки карты в движок (**Load Map In Engine** в ленте/меню). Удобно на итерациях: правил браш → F9 Fast Compile → загрузил → посмотрел.

Ограничение: в этом режиме **часть команд недоступна** (`nav_generate`, часть кастомных конваров). Для финальной проверки нужен способ 2.

### Способ 2 (основной) — обычный CS2 + консоль

1. Закрой Workshop Tools. Запусти **обычный CS2** («Играть», не `-tools`).
2. Открой консоль (`~`) и выполни:

```
map_workshop aim_training_v1 aim_training_v1
```

- Первый аргумент — **имя аддона**, второй — **имя файла карты** (без `.vmap`). Оба подставляются по `Tab`.
- Карта по умолчанию стартует в Competitive. Чтобы сменить режим — задай `game_type 0` (классика) и нужный `game_mode` **до** `map_workshop`, затем повтори команду загрузки.
- Если CS2 не видит карту — проверь, что `.vpk` лежит ровно в `game\csgo_addons\aim_training_v1\maps\` (без подпапок), и что в аддоне только одна карта.

**Удобно:** добавь в `autoexec.cfg` алиас, чтобы не печатать каждый раз:
```
alias aim "map_workshop aim_training_v1 aim_training_v1"
```
Потом в консоли достаточно `aim`.

### Способ 3 (после публикации) — как обычная карта из Workshop

После публикации (шаг 8) карту можно запустить двумя путями:
- Play → **Workshop Maps** → выбрать свою карту;
- консоль: `map_workshop <цифровой Workshop ID> <имя карты>` (ID — число из URL страницы карты).

Для игре с друзьями понадобится сообщество/сервер: `map_workshop` — клиентская команда, на выделенном сервере используется `host_workshop_map <Workshop ID>`.

---

## Шаг 6. Что ввести в консоли сразу после загрузки

Скопируй [`cfg/aim_training_v1_test.cfg`](../../cfg/aim_training_v1_test.cfg) в
```
...\game\csgo\cfg\aim_training_v1_test.cfg
```
и выполни в консоли:

```
exec aim_training_v1_test
```

Или вручную — минимальный набор:

```
sv_cheats 1
mp_warmup_end
bot_kick
mp_restartgame 1
```

Дальше — по задаче:

| Задача | Команды |
|--------|---------|
| Проверить выдачу оружия и брони | `mp_free_armor 2` `mp_ct_default_primary weapon_ak47` `mp_t_default_primary weapon_ak47` `mp_restartgame 1` |
| Включить deathmatch-респавн | `mp_respawn_on_death_ct 1` `mp_respawn_on_death_t 1` |
| Отработка спрея | `sv_infinite_ammo 2` `weapon_accuracy_nospread 1` `sv_showimpacts 1` |
| Навмеш для ботов | `sv_cheats 1` `nav_edit 1` `nav_generate` `nav_save` `mp_restartgame 1` |
| Боты вместо мишеней | `bot_add_ct` `custom_bot_difficulty 2` `bot_stop 1` |
| Диагностика entity-логики | `developer 1` (в логе видны цепочки outputs) |

---

## Шаг 7. Контрольный обход карты (5 минут)

Отметь каждое — это и есть приёмка:

- [ ] **Свет.** Стены и пол светлые, тёмных углов нет, мишени читаются с 2000 units. (Чёрно — значит Fast Compile либо нет `env_sky`.)
- [ ] **Руки и оружие не чёрные.** Чёрные → не перекрывает `env_combined_light_probe_volume`.
- [ ] **Спавн.** Ты появился на линии огня лицом к мишеням, а не в наблюдателях. («Все укомплектованы» в логе → спавны ниже 16 units над полом.)
- [ ] **Оружие.** В руках AK-47 + Deagle, есть броня со шлемом.
- [ ] **Респавн.** Убей себя (`kill` в консоли) — должен мгновенно возродиться с оружием.
- [ ] **Мишени.** Выстрел по `func_button` → мишень гаснет и через секунду появляется; на стене/мишени видны декали попаданий.
- [ ] **Движущиеся мишени.** `mt_1000` / `mt_1500` / `mt_depth` ездят туда-обратно.
- [ ] **Нет утечек/ warining'ов** в логе компиляции (см. §8/§9 гайда).
- [ ] **FPS стабильный** (`cl_showfps 2`): на пустой коробке 3000×3000 просадок быть не должно.
- [ ] **Боты ходят** (если генерил nav): `bot_add_ct` — бот дошёл до линии огня.

---

## Шаг 8. Цикл правок (как работать дальше)

```
Правишь .vmap в Hammer
  → Ctrl+S
  → F9 (Fast Compile — геометрия/мишени; Full Compile — если менял свет)
  → в CS2: map_workshop aim_training_v1 aim_training_v1   (карта НЕ перезагружается сама!)
  → exec aim_training_v1_test
```

Важно: после перекомпиляции игру **не надо** перезапускать, но карту — обязательно (`map_workshop ...` повторно). Движок кеширует загруженный `.vpk`.

---

## Шаг 9. Публикация в Steam Workshop (когда карта готова)

1. Убедись, что в `game\csgo_addons\aim_training_v1\maps\` лежит **ровно один** `.vpk` (старые билды удали — из-за них публикуется не та версия).
2. Запусти **Counter-Strike 2 (Tools)** → выбери аддон `aim_training_v1` → **Launch Tools**.
3. В Asset Browser (или в окне инструментов) открой меню **Tools → Counter-Strike 2 Workshop Manager**.
4. **New / «+»** → заполни название, описание, превью → **Publish**.
5. Видимость выставь **Public** — со скрытой/приватной записью загрузка ломается.
6. После публикации: подпишись на карту → CS2 → Play → **Workshop Maps**. Либо в консоли: `map_workshop <Workshop ID> aim_training_v1`.

Если используешь кастомные материалы (свою яркую текстуру мишеней) и в обычном CS2 они фиолетовые/чёрные — проверь, что скомпилированные материалы лежат в `game\csgo_addons\aim_training_v1\materials\` и попали в публикацию.

---

## Шаг 10. Если что-то не запускается

| Симптом | Причина | Что делать |
|---------|---------|------------|
| `map_workshop` — «Unknown command» / карта не находится | Нет `.vpk`, или он в подпапке, или в аддоне несколько карт | Проверить `game\csgo_addons\aim_training_v1\maps\aim_training_v1.vpk`; перекомпилировать F9 |
| Загружается чёрная карта | Fast Compile (нет запечённого света) или нет `env_sky` | Full Compile; проверить `env_sky` + поле `Sky IBL Source` у `light_environment` |
| Сразу попадаешь в наблюдатели / «все укомплектованы» | Спавны вплотную к полу | Поднять origin спавнов минимум на 16 (ставь 32) |
| «No player start» | Нет `info_player_start` | Добавить в (0, -1200, 32) |
| Нет оружия | Особенность `game_player_equip` в CS2 | Проверить конвары `mp_ct_default_primary` / `mp_free_armor`; триггеру над спавном нужна задержка 0.10 |
| Чёрные руки/ствол | Нет Light Probe Volume | `env_combined_light_probe_volume` на всю игровую зону (и куб, и сфера) |
| `nav_generate` — `no walkable seeds present` | Нет стартовой точки | Добавить сущность `point_nav_walkable` в (0, 0, 32) |
| Команды не работают, конвары «заблокированы» | Запущен режим `-tools` или не включены читы | Обычный CS2 + `sv_cheats 1` |
| VRAD3 падает / `light mapper didn't return valid triangles` | GPU без RT, встройка, драйвер | Отключить встройку в BIOS, обновить/откатить драйвер, компилировать из командной строки (§9 гайда) |
| Боты тупые | `bot_difficulty` не влияет | `custom_bot_difficulty 0..3` |
| Публикуется не та версия карты | Несколько `.vpk` в папке maps | Удалить старые, перекомпилировать, опубликовать заново |

---

## TL;DR — всё действие на одной карточке

```
1. Steam → CS2 → Свойства → DLC → Counter-Strike 2 Workshop Tools
2. Запустить "Counter-Strike 2 (Tools)" → Asset Browser → Create New Addon: aim_training_v1 → Hammer
3. File → Save As: content\csgo_addons\aim_training_v1\maps\aim_training_v1.vmap
4. Собрать: коробка → спавны → мишени → свет → логика   (по aim_training_v1_guide.md)
5. F9 → Fast Compile  →  проверить, что появился game\csgo_addons\aim_training_v1\maps\aim_training_v1.vpk
6. F9 → Full Compile (Lightmap 4096, Quality Final)
7. Обычный CS2 → консоль → map_workshop aim_training_v1 aim_training_v1
8. exec aim_training_v1_test
9. Правки → Ctrl+S → F9 → снова map_workshop aim_training_v1 aim_training_v1
10. Готово → Tools → CS2 Workshop Manager → Publish
```

---

## Источники

- Запуск своей аддон-карты в обычном CS2 через `map_workshop <аддон> <карта>`: https://taxidriverhk.com/csmaps/tutorials/play-your-own-addon-map
- Одна карта на аддон, файлы строго в `maps\` без подпапок: https://github.com/ValveSoftware/csgo-osx-linux/issues/3526
- Публикация: Tools → **Counter-Strike 2 Workshop Manager** → New: https://www.reddit.com/r/csmapmakers/comments/17nrv4h/uploading_a_cs2_map/ · https://steamcommunity.com/app/730/discussions/0/3939021373342747518/ · https://www.reddit.com/r/GlobalOffensive/comments/17n3heh/cs2_workshop_maps_are_now_available/
- Удалить старые `.vpk` перед публикацией: https://www.reddit.com/r/hammer/comments/17muy6c/cant_publish_to_cs2_workshop/ ; публикация должна быть Public: https://www.reddit.com/r/csworkshop/comments/1nr87ep/help_creating_a_custom_workshop_addon/
- `map_workshop` — клиентская команда, на сервере — `host_workshop_map <Workshop ID>`: https://www.gameserverkings.com/knowledge-base/counter-strike-2/how-to-add-workshop-maps/
- Полноценный тест (боты, nav, кастомные конвары) — только в обычном CS2, не в `-tools`: https://github.com/ALEXalesha/MyCS2Map
