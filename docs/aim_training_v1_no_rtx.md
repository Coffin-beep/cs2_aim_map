# aim_training_v1 без RTX: как собрать и запустить карту

Короткий ответ: **собирать и играть можно уже сегодня**, но без запечённого света. Для публичной версии свет придётся запечь один раз на любой машине с RT-железом (друг / аренда / апгрейд) — это занимает одну компиляцию, а не весь цикл разработки.

---

## 1. Сначала проверь: точно ли RT нет

| Проверка | Как | Что делать при положительном результате |
|----------|-----|------------------------------------------|
| Какая видеокарта | Диспетчер устройств → Видеоадаптеры | **NVIDIA GTX 10xx/16xx — RT нет.** RTX 20xx/30xx/40xx/50xx — RT есть. **AMD RX 6000/7000 — RT есть** (Valve: минимум 6600XT), RX 500 и старше — нет. Intel Arc — RT есть |
| Есть встройка (Ryzen с «Graphics» / Intel HD) | Диспетчер устройств → два видеоадаптера | Hammer часто хватается за **встройку** и падает на VRAD3. Отключи её: BIOS → Internal Graphics = Disabled (или в диспетчере устройств). Это чинит компиляцию даже на RTX-системах |
| Драйвер | Версия драйвера GPU | На RTX-картах «Unable to run Vrad3 / light mapper didn't return valid triangles» лечится откатом драйвера (для NVIDIA часто помогает ветка от 11.09.2024) |

Если RT действительно нет — дальше по плану A.

---

## 2. Что работает без RTX, а что нет

| Что | Статус без RT | Комментарий |
|-----|---------------|-------------|
| Геометрия, браши, `func_*` | ✅ работает | |
| Энтити, I/O-логика, `game_player_equip`, триггеры | ✅ работает | |
| Мишени `func_button` с Damage Activates | ✅ работает | |
| Движущиеся мишени `func_movelinear` | ✅ работает | |
| Компиляция VIS, физики, звука, nav | ✅ работает | |
| `map_workshop` — запуск и игра | ✅ работает | |
| **Запечённый свет (VRAD3, lightmaps)** | ❌ не работает | Требуется GPU с аппаратным RT |
| **Кубмапы / `env_combined_light_probe_volume`** | ❌ не запекаются | Отсюда чёрные руки и оружие в первом лице |
| **Предпросмотр света в Hammer (GPU Path Tracing / Preview Baked Lighting)** | ❌ | Ругается `Must run with -gpuraytracing` |
| Эмиссивные материалы (светящиеся поверхности) | ⚠ частично | Самосвечение видно, но свет на окружение не льётся |

---

## 3. План A (основной) — карта без запечённого света

### 3.1 Компиляция: отключить стадию света

`F9` → **Custom Settings** → в настройках снять галочку **Generate Lightmaps** (Generate Lighting). Всё остальное (world, phys, vis, nav) оставить включённым → Build.

Или проще: **Fast Compile** (он по определению не считает свет, VIS и звук).

Результат тот же, что и раньше: `game\csgo_addons\aim_training_v1\maps\aim_training_v1.vpk`. Запускается одинаково:
```
map_workshop aim_training_v1 aim_training_v1
```

### 3.2 Сделать материалы самосветящимися — это ключевой шаг

Без лайтмапы поверхности рендерятся тёмными. Лечится тем, что материал **сам себе источник яркости** — ему не нужна лайтмапа.

В аддоне создай три материала (Asset Browser → папка `materials/aim/` → Create Material, шейдер по умолчанию **Csgo Simple**):

| Материал | Назначение | Цвет |
|----------|-----------|------|
| `aim/wall_white.vmat` | стены, потолок | светло-серый `230 230 232` |
| `aim/floor_grey.vmat` | пол | серый `150 150 155` (сетку можно нарисовать в текстуре) |
| `aim/target_orange.vmat` | мишени | насыщенный `255 120 20` |

Что включить в каждом материале (Material Editor):
- **SelfIllum / Emissive** — включить; маска `_selfillum` белая = 100 % самосвечения, чёрная = 0 %. Это даёт яркую поверхность **без единого источника света** на карте.
- Опционально: на лице меша в Hammer есть свойство **Emissive** — включи его у мишеней.
- Не надейся, что самосвечение осветит окружение: в CS2 это работает только с запеканием (baked GI). Нам нужно, чтобы **сам материал** был ярким.

Имена полей в Material Editor могут отличаться на 1–2 символа в зависимости от версии тулзов — ищи по смыслу: *SelfIllum*, *Emissive*, *Texture Color*.

### 3.3 `mat_fullbright` — быстрый тест

Уже в игре, в консоли:
```
mat_fullbright 1
```
Если движок сам не включил fullbright на карте без лайтмап, эта команда даст ровную яркую картинку. Для личной тренировки этого достаточно — геометрия, мишени и контраст читаются.

> `mat_fullbright` — клиентская команда. Локально она решает проблему, но другим игрокам (если опубликуешь карту как есть) всё будет темно. Для публичной версии нужен baked-свет по плану C.

### 3.4 Световые энтити — всё равно расставить

Не удаляй из карты `env_sky`, `light_environment`, `light_omni2`, `env_combined_light_probe_volume`, объём `tools/toolslightmapres`. Они «молчат» до первого запекания, но как только ты запустишь Full Compile на машине с RTX — карта сразу станет нормальной, без переделки. Стоимость: ноль минут сейчас, ноль переделок потом.

### 3.5 Что можно тестировать уже сейчас

Всё, что не связано со светом:
- геометрия и герметичность (нет `leak` в логе → VIS посчитан);
- спавны, `info_player_start`, углы обзора;
- 20 мишеней + попадания (декали и звук работают без света);
- движущиеся мишени;
- выдача оружия, броня, DM-респавн;
- nav mesh и боты (`nav_generate`, `custom_bot_difficulty`);
- FPS и общая «геометрия боя».

---

## 4. План B — попытка CPU-компиляции (`-lightmapcpu`)

Шанс низкий: Valve убрала legacy CPU-компилятор в обновлении ~октября 2024, и проект-патчер `CS2MapCompilerCPU` это подтверждает. Но попробовать стоит 15 минут — если прокатит, получишь полноценный свет.

1. `F9` → **Custom Settings** → скопируй содержимое поля **Command Line**.
2. Вставь в **cmd.exe** и добавь `-lightmapcpu` сразу после `resourcecompiler.exe`.
3. Жди (CPU-компиляция — часы, не минуты).
4. Готовый `.vpk` забери из `%LOCALAPPDATA%\Temp\valve\hammermapbuild\game\csgo_addons\aim_training_v1\maps\` и положи в `game\csgo_addons\aim_training_v1\maps\`.

Готовая команда-шаблон (подставь свой путь к CS2):
```
"...\game\bin\win64\resourcecompiler.exe" -threads 5 -quiet -unbufferedio -i
"...\content\csgo_addons\aim_training_v1\maps\aim_training_v1.vmap"
-noassert -world -bakelighting -lightmapMaxResolution 1024
-lightmapDoWeld -lightmapVRadQuality 1 -lightmapLocalCompile
-phys -vis -nav -retail -breakpad -nop4
-outroot "%LOCALAPPDATA%\Temp\valve\hammermapbuild\game" -lightmapcpu
```

Что будет при провале: `Unable to run Vrad3`, `Light mapper didn't return valid triangles`, или молчаливый билд без лайтмап. → идёшь по плану A или C.

---

## 5. План C (рекомендую для релиза) — один раз запечь на машине с RTX

Bake делается **один раз**, дальше ты играешь у себя. Варианты: друг с RTX-картой, аренда Windows-машины с GPU на час, ноутбук, будущий апгрейд.

**Что передать туда:**
```
content\csgo_addons\aim_training_v1\        <- весь аддон целиком (vmap + исходники материалов)
```

**Что сделать там:**
1. Установить CS2 + Workshop Tools, создать аддон с **тем же именем** `aim_training_v1`.
2. Залить туда содержимое папки аддона.
3. Открыть `...\maps\aim_training_v1.vmap` → `F9` → **Full Compile**, Lightmap 4096, Quality Final.

**Что забрать обратно:**
```
game\csgo_addons\aim_training_v1\maps\aim_training_v1.vpk
game\csgo_addons\aim_training_v1\materials\...     <- если использовал свои материалы
```
Положить в свои соответствующие папки → `map_workshop aim_training_v1 aim_training_v1` → карта со светом.

Дальше можно спокойно править геометрию и мишеней у себя (Fast Compile), а bake переделывать только когда меняешь свет или перед публикацией.

---

## 6. План D — апгрейд

Если планируешь делать карты всерьёз, RT-железо всё равно понадобится. Минимум по официальным требованиям Valve:
- **NVIDIA RTX 2060 (6 ГБ)** и выше; рекомендуется 2080 Ti+;
- **AMD RX 6600 XT** и выше; рекомендуется 6800 XT+.

То есть RTX — это не только «зелёные» карты. Б/у RTX 2060/3050 или RX 6600XT закрывают вопрос целиком.

---

## 7. Чего не делать

- **Не откатывай CS2 на старую версию** ради CPU-компилятора: старый Hammer не откроет `.vmap`, сохранённый новой версией, — потеряешь карту.
- **Не ставь левые патчеры** (`hammer_patcher.py` и аналоги): они опирались на файлы, которых в актуальном билде уже нет.
- **Не публикуй карту без baked-света** как финальную: у подписчиков она будет тёмной (у тебя ярко только из-за `mat_fullbright`).

---

## 8. Порядок действий

**Сегодня (без RTX):**
1. Проверить видеоадаптеры и отключить встройку (§1).
2. Собрать карту по основному гайду, материалы сделать **selfillum** (§3.2).
3. `F9` → Custom Settings → **выключить Generate Lightmaps** → Build.
4. Обычный CS2 → `map_workshop aim_training_v1 aim_training_v1` → `mat_fullbright 1` → `exec aim_training_v1_test`.
5. Тестировать геймплей: мишени, оружие, респавн, nav, боты.

**Перед публикацией:**
6. Попробовать `-lightmapcpu` (10–15 минут на проверку).
7. Не сработало — запечь свет по плану C на машине с RTX, забрать `.vpk` себе.
8. Опубликовать уже освещённую версию.

---

## Источники

- Официальные требования Hammer: GPU с ray tracing обязателен (NVIDIA 2060 / AMD 6600XT минимум): https://www.mapcore.org/forums/thread/26872-counter-strike-2-workshop-q-a/
- Без RT доступен только Fast Compile; полноценная компиляция недоступна: https://www.reddit.com/r/hammer/comments/16y3afu/can_i_compile_in_cs2_without_rtx/ · https://steamcommunity.com/sharedfiles/filedetails/?l=brazilian&id=3032828211
- `F9 → Custom Settings` → выключить «Generate Lightmaps»: https://www.reddit.com/r/csmapmakers/comments/142v6g8/rtx_gpu_required_to_map_for_cs2/
- Пользователь на RX 570 собирает без baked-света: «карта чёрная по умолчанию, но с достаточным количеством env-света почти как запечённая»: https://www.reddit.com/r/csmapmakers/comments/142v6g8/rtx_gpu_required_to_map_for_cs2/
- `-lightmapcpu` и инструкция (устарело): https://www.reddit.com/r/hammer/comments/1433dkw/i_dont_have_gpu_with_ray_tracing/ · шаблон команды: https://gist.github.com/NotHosomi/9bcd840b269d08254f644801d6597bfb
- Valve убрала CPU-компилятор (обновление ~октябрь 2024): https://www.reddit.com/r/hammer/comments/1fw6aob/cant_compile_unable_to_run_vrad3/ · https://steamcommunity.com/app/730/discussions/0/3881597253354894430/ · https://github.com/miifanboy/CS2MapCompilerCPU
- Hammer цепляется за встройку Ryzen → отключить iGPU в BIOS: https://www.reddit.com/r/hammer/comments/178ibpb/my_cs2_maps_are_not_compiling/
- Откат драйвера NVIDIA лечит `Unable to run Vrad3` на RTX-картах: https://www.reddit.com/r/hammer/comments/1fw6aob/cant_compile_unable_to_run_vrad3/
- SelfIllum в материалах CS2 (белая маска = 100 % самосвечения без источников света): https://taxidriverhk.com/csmaps/tutorials/textures-for-custom-materials
- Эмиссивные материалы в CS2 реально светят на окружение только при запекании: https://www.mapcore.org/topic/26822-cs2-hammer-raytracing/
- `mat_fullbright` (0 — обычный свет, 1 — полная яркость): https://developer.valvesoftware.com/wiki/Lighting
