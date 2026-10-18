# aim_training_v1 — рабочий план сборки в Hammer (CS2 / Source 2)

Документ написан как пошаговая инструкция: открыл Hammer → собрал карту → скомпилировал → протестировал.
Все координаты даны в юнитах Hammer (1 unit ≈ 1.9 см; игрок: рост 72, ширина 32, глаза на высоте 64 от «подошвы»).

**Система координат этой карты:**
- Пол: `Z = 0` (верхняя грань), толщина пола 32 → низ `Z = -32`.
- Потолок: `Z = 1000` (нижняя грань), толщина 32 → верх `Z = 1032`.
- Арена: `X ∈ [-1500, 1500]`, `Y ∈ [-1500, 1500]`. Игровое поле 3000 × 3000, высота 1000.
- Игрок стоит на линии огня `Y = -1200` и стреляет в сторону `+Y`.
- Сетка: **64** (крупные операции — 512).

---

## 0. Пять поправок к ТЗ: что в Source 2 работает НЕ так, как в Source 1

Прочитать ДО начала сборки — это экономит часы.

| № | В ТЗ | Как на самом деле в CS2 (Source 2) | Что делать |
|---|------|------------------------------------|-----------|
| 1 | Детали — `func_detail` | **`func_detail` и hint-брашей в Source 2 не существует.** Понятие BSP-брашей упразднено, геометрия — меши. Аналог — галочка **«Exclude from VIS»** в свойствах объекта. Статичные пропы по умолчанию **не** участвуют в VIS (включается галочкой «Vis Occluder»). | Все мишени, рамки, декор → отметить **Exclude from VIS = Yes**. |
| 2 | `light_environment` освещает закрытый бокс | Свет `light_environment` в Source 2 берётся из **неба**: сущность **`env_sky`** + поле **«Sky IBL Source»** у `light_environment` (указывает имя `env_sky`). В глухом боксе без `env_sky` карта будет тёмной. | Обязательная связка: `env_sky` + `light_environment` (Sky IBL Source = имя env_sky) + потолок/проём с `tools/toolsskybox`. Плюс подстраховка baked-источниками. |
| 3 | `game_player_equip` выдаёт оружие надёжно | В CS2 у `game_player_equip` изменившееся/глючное поведение (осень 2023 — мапперы массово жаловались). Рабочий приём сообщества — **`point_servercommand` + конвары `mp_ct_default_primary`, `mp_t_default_primary`, `mp_free_armor`**. | Делаем **обе** схемы (раздел 6): equip-энтити по ТЗ + дублируем конварами. |
| 4 | Компиляция как в CS:GO | **VRAD3 считает свет на GPU (ray tracing)** — нужна видеокарта с RTX (минимум NVIDIA 2060 / AMD 6600XT). Без запечённого света карта тёмная — это не баг сцены. | Fast Compile (без света) на итерациях, Full Compile с Lightmap 4096 / Quality Final — на релиз. |
| 5 | Навмеш сгенерируется сам | Автогенерация часто падает с `no walkable seeds present`. | Добавить сущность **`point_nav_walkable`** (аналог старой `nav_mark_walkable`), достаточно одной точки в связном пространстве. |

Дополнительные грабли, подтверждённые на реальной опубликованной карте:
- Спавны T/CT должны быть **минимум на 16 units выше пола** (практически ставят 32), иначе движок помечает их `Invalid spawnpoint`. Между спавнами ≥ 32, от стен ≥ 16.
- Кроме T/CT-спавнов **нужен `info_player_start`**, иначе «No player start».
- Коробка должна быть **полностью замкнута** (пол + 4 стены + потолок), иначе `leak` на компиляции.
- Первый `Save As` может незаметно сохранить файл как `untitled_1.vmap` — проверяй заголовок окна Hammer.
- Чёрные руки и оружие в первом лице = нет Light Probe Volume → ставим **`env_combined_light_probe_volume`**.
- Полноценный тест (боты, nav, кастомные конвары) — только в **обычном CS2, не в режиме `-tools`**: `map_workshop <аддон> <карта>`.

---

## 1. Создание проекта

1. Steam → CS2 → Properties → DLC → включить **Counter-Strike 2 Workshop Tools**.
2. Запустить CS2 → выбрать **Counter-Strike 2 (Tools)** → **Workshop Tools → Launch Tools → Asset Browser** → иконка **Hammer**.
3. В Asset Browser: **Create new addon** → имя аддона `aim_training_v1` (латиницей, без пробелов).
4. Hammer: `File → New` (Ctrl+N) → сохранить как
   `...\content\csgo_addons\aim_training_v1\maps\aim_training_v1.vmap`.
5. `Map → Map Properties`: при необходимости задать Skybox / комментарий. Убедись, что в заголовке окна — `aim_training_v1.vmap`, не `untitled_1.vmap`.
6. Настройки сетки: **Grid 64**, `Snap to grid` вкл. Точная подстановка координат — через панель **Transform** (или Object Properties → Transform).

Пути, которые понадобятся:
```
content\csgo_addons\aim_training_v1\maps\aim_training_v1.vmap   <- исходник
game\csgo_addons\aim_training_v1\maps\aim_training_v1.vpk       <- результат компиляции
```

---

## 2. Геометрия арены (блок 1) — минимум брашей, ноль проблем с VIS

Собирай **мировые браши** (не энтити!) — только они герметизируют карту.
Материал назначай только на видимые внутренние грани, на остальные — `tools/toolsnodraw`.

| # | Элемент | Min (X, Y, Z) | Max (X, Y, Z) | Материал (видимая грань) |
|---|---------|---------------|---------------|--------------------------|
| 1 | Пол | -1532, -1532, -32 | 1532, 1532, 0 | `dev/dev_measuregeneric01` (сетка 64 — визуальный ориентир дистанции) |
| 2 | Потолок | -1532, -1532, 1000 | 1532, 1532, 1032 | `dev/dev_measuregeneric01b` (светлый) — **или** `tools/toolsskybox` (вариант B ниже) |
| 3 | Стена задняя (за спиной, -Y) | -1532, -1564, -32 | 1532, -1500, 1032 | `dev/dev_measuregeneric01b` |
| 4 | Стена фоновая (дальняя, +Y) | -1532, 1500, -32 | 1532, 1564, 1032 | `dev/dev_measuregeneric01` (серый — контрастный фон для ярких мишеней) |
| 5 | Стена левая (-X) | -1564, -1564, -32 | -1500, 1564, 1032 | `dev/dev_measuregeneric01b` |
| 6 | Стена правая (+X) | 1500, -1564, -32 | 1564, 1564, 1032 | `dev/dev_measuregeneric01b` |

Углы намеренно перекрыты (±1532 / ±1564) — это исключает щели в стыках и, как следствие, `leak`.

**Правила брашворка для этой карты:**
- Никаких внутренних стен, колонн, цилиндров, лестниц в базовой версии: одна коробка = 1–2 VIS-кластера = мгновенная компиляция VIS и стабильный FPS.
- Никаких `func_detail`: не существует (см. §0).
- Все «неполезные» грани (снаружи, торцы) — `tools/toolsnodraw`; они не рендерятся и не получают лайтмапу.

**Вариант B (если хочешь солнечный свет от `light_environment`):**
нижнюю грань потолка (браш №2) текстурируй `tools/toolsskybox` и обязательно поставь `env_sky` (раздел 5). В Source 2 `tools/toolsskybox` герметизирует карту и блокирует VIS, а картинку неба даёт именно `env_sky`.

---

## 3. Зона игрока, спавны, разметка (блок 2)

### 3.1 Линия огня и спавны

Линия огня: `Y = -1200`. Спавны — на одной линии, с шагом 100 по X.

**CT:** `info_player_counterterrorist`, `Z = 32`, `Y = -1200`, **Angles (Pitch Yaw Roll) = `0 90 0`** (смотрит в +Y, на мишени).
X: `-450, -350, -250, -150, -50, 50, 150, 250, 350, 450` → **10 точек**.

**T:** `info_player_terrorist`, `Z = 32`, `Y = -1260` (смещение на 60, чтобы команды не спавнились друг в друге), те же X, те же Angles.

**`info_player_start`** — 1 шт. в `(0, -1200, 32)` (обязателен движку, даже если геймплей на T/CT-спавнах).

Контроль валидности: `Z = 32` (≥16 над полом), шаг между точками 100 (≥32), до задней стены 300 (≥16).

### 3.2 Разметка пола (визуальная, 1 браш)

Тонкий браш-«полоса»: `X ∈ [-900, 900]`, `Y ∈ [-1210, -1200]`, `Z ∈ [0, 2]`.
Материал — контрастный: `dev/dev_measuregeneric01c` либо свой `materials/aim/aim_line_yellow.vmat` (раздел 4.4).

### 3.3 Зона закупки (опционально)

Два триггерных браша над линией огня: `X ∈ [-950, 950]`, `Y ∈ [-1450, -1000]`, `Z ∈ [0, 256]`, материал `tools/toolstrigger`, `Ctrl+T` → **`func_buyzone`**:
- один с **Team Number = 3** (CT), второй с **Team Number = 2** (T).

Они перекрываются — это нормально. Без них боты в логе будут ругаться `bot spawned outside of a buy zone`; при `mp_buy_anywhere 1` можно не ставить, но для чистоты лога лучше поставить.

---

## 4. Мишени (блок 3)

### 4.1 Расстановка статичных мишеней

Линия огня `Y = -1200`. Дистанция = `Y_мишени − (−1200)`.
Размер мишени: **64 (X) × 8 (Y) × 64 (Z)**, центр по высоте `Z = 64` (уровень головы/глаз игрока: рост 72, глаза 64).

| Ряд | Дистанция | Y | X-позиции (центры) | Угол от центра (≈) | Назначение |
|-----|-----------|----|--------------------|--------------------|------------|
| **R1** | 500 u (9.5 м) | -700 | -350, -175, 0, 175, 350 | 19° / 35° | Широкие флики, рефлекс |
| **R2** | 1000 u (19 м) | -200 | -500, -250, 0, 250, 500 | 14° / 27° | Базовая дистанция AK/M4 |
| **R3** | 1500 u (28.5 м) | 300 | -600, -300, 0, 300, 600 | 11° / 22° | Контроль отдачи на дальней |
| **R4** | 2000 u (38 м) | 800 | -400, -200, 0, 200, 400 | 6° / 11° | AWP / SSG 08, один tap |

Итого: **20 статичных мишеней**. Полная таблица координат:

| ID | X | Y | Z (центр) | | ID | X | Y | Z (центр) |
|----|---|---|-----------|-|----|---|---|-----------|
| tg_r1_1 | -350 | -700 | 64 | | tg_r3_1 | -600 | 300 | 64 |
| tg_r1_2 | -175 | -700 | 64 | | tg_r3_2 | -300 | 300 | 64 |
| tg_r1_3 | 0 | -700 | 64 | | tg_r3_3 | 0 | 300 | 64 |
| tg_r1_4 | 175 | -700 | 64 | | tg_r3_4 | 300 | 300 | 64 |
| tg_r1_5 | 350 | -700 | 64 | | tg_r3_5 | 600 | 300 | 64 |
| tg_r2_1 | -500 | -200 | 64 | | tg_r4_1 | -400 | 800 | 64 |
| tg_r2_2 | -250 | -200 | 64 | | tg_r4_2 | -200 | 800 | 64 |
| tg_r2_3 | 0 | -200 | 64 | | tg_r4_3 | 0 | 800 | 64 |
| tg_r2_4 | 250 | -200 | 64 | | tg_r4_4 | 200 | 800 | 64 |
| tg_r2_5 | 500 | -200 | 64 | | tg_r4_5 | 400 | 800 | 64 |

### 4.2 Мишень = `func_button` (видимая + реагирует на попадание)

Один браш = и мишень, и детектор попадания. Это избавляет от лишних энтити.

1. Создай браш 64 × 8 × 64, поставь в координаты из таблицы.
2. Назначь яркий материал мишени (раздел 4.4).
3. Выдели браш → **Ctrl+T** → класс **`func_button`**.

| Key / Flag | Значение |
|------------|----------|
| Name | `tg_r2_3` (по таблице выше) |
| Health | `1` (если не срабатывает с первого выстрела — поставь `0`, оставив только флаг) |
| Delay Before Reset | `0` |
| **Flags** | ☑ **Don't move**, ☑ **Damage Activates** (остальные — выкл.) |
| Свойства объекта | **Exclude from VIS = Yes** |

**Outputs (вкладка Outputs):**

| My output | Target | Input | Delay | Комментарий |
|-----------|--------|-------|-------|-------------|
| OnDamaged | `!self` | `Disable` | 0.00 | Мишень «гаснет» при попадании |
| OnDamaged | `!self` | `Enable` | 1.00 | Через секунду появляется снова |
| OnDamaged | `hits_counter` | `Add` `1` | 0.00 | Опционально: общий счётчик попаданий |

Резервный вариант (если `func_button` в твоей версии FGD не выдаёт `OnDamaged`): два браша —
видимый `func_brush` `tg_...` + перед ним (на 8 units ближе к игроку) невидимый браш с материалом **`tools/toolsblockbullets`**, привязанный к `func_button` с теми же флагами; его `OnDamaged` → `tg_...` `Disable`/`Enable`.

### 4.3 Движущиеся мишени: `func_movelinear`

| Параметр | M1 (1000 u) | M2 (1500 u) | M3 (треккинг в глубину) |
|----------|-------------|-------------|--------------------------|
| Name | `mt_1000` | `mt_1500` | `mt_depth` |
| Размер браша | 64 × 8 × 64 | 64 × 8 × 64 | 64 × 8 × 64 |
| Старт (центр) | (-600, -200, 64) | (-700, 300, 64) | (700, -450, 64) |
| **Move Direction (Pitch Yaw Roll)** | `0 0 0` (+X) | `0 0 0` (+X) | `0 90 0` (+Y) |
| Move Distance (`move_distance`) | `1200` | `1400` | `900` |
| Speed | `220` (≈ бег с автоматом, 215) | `250` (≈ бег с ножом) | `180` |
| Start Position | `0` | `0` | `0` |
| Свойства объекта | Exclude from VIS = Yes | то же | то же |

**Outputs (у каждой мишени, автопинг-понг без таймеров):**

| My output | Target | Input | Delay |
|-----------|--------|-------|-------|
| OnFullyOpen | `!self` | `Close` | 0.30 |
| OnFullyClosed | `!self` | `Open` | 0.30 |

Запуск: `logic_auto` → `OnMapSpawn` → `mt_1000` / `mt_1500` / `mt_depth` → `Open`, delay `1.0`.

> Скорость 200–250 u/s выбрана не случайно: это реальная скорость бега игрока (250 с ножом, 215 с AK). Тренировка на такой скорости переносится в матч; 400+ u/s тренирует не то, что встречается в игре.

### 4.4 Материалы и визуальный стиль

База из игры (проверено, существует в CS2):
- `materials/dev/dev_measuregeneric01.vmat` — серый с сеткой 64 (пол + фоновая стена);
- `materials/dev/dev_measuregeneric01b.vmat` — светлый (стены, потолок);
- `materials/dev/dev_measuregeneric01c.vmat` — тёмный вариант (разметка).

Яркая мишень — свой материал (создаётся в аддоне, так надёжнее, чем искать подходящий в базе):
1. Asset Browser → в аддоне `aim_training_v1` создай папку `materials/aim/`.
2. Скопируй туда `dev/dev_measuregeneric01.vmat` → переименуй в `target_orange.vmat`.
3. Открой в Material Editor: задай цвет/тинт (насыщенный оранжевый `255 120 20`) и, если доступен, SelfIllum/Emissive — чтобы мишень не «провалилась» в тень.
4. Сохрани, назначь на все мишени через **Active Material**.

Принцип контраста: **светло-серый фон + насыщенная мишень + тёмная разметка**. Никаких пёстрых/детализированных текстур, никакого «шума» за мишенями — глаз должен находить цель за 1 кадр.

Альтернатива: `prop_dynamic` с моделью из браузера ассетов — но это зависимость от конкретных моделей и отдельная возня с коллизией. Для v1 мишени-браши надёжнее и точно совпадают с хитбоксом, который ты хочешь тренировать.

### 4.5 Опциональные модули (ставь после того, как база работает)

**Спрей-стена (контроль отдачи).** Браш `X ∈ [700, 1400]`, `Y = -700`, `Z ∈ [0, 384]`, материал — светлый (`dev/dev_measuregeneric01b`). Игрок встаёт в `X ≈ 1050, Y = -1200` → дистанция 500 u, классическая для отработки спрея. Добавь 2 CT и 2 T спавна в `(1050, -1200, 32)` и `(1050, -1260, 32)`.

**Стена выбора оружия (стреляй, чтобы взять).** На левой стене (`X = -1500`), 4 панели `8 (X) × 192 (Y) × 192 (Z)`, центр по Z = 128, центры по Y: `-1000` (AK), `-750` (M4A1-S), `-500` (AWP), `-250` (Deagle). Каждая — `func_button` с флагами **Don't move + Damage Activates**:
`OnDamaged` → `equip_ak` → **`TriggerForActivatedPlayer`** с параметром `weapon_ak47` (аналогично `equip_m4` → `weapon_m4a1_silencer`, `equip_awp` → `weapon_awp`, `equip_deagle` → `weapon_deagle`). Каждая equip-энтити содержит `weapon_knife 1`, `item_assaultsuit 1` и свой ствол, spawnflags = **3** (Use Only + Strip All Weapons First).

---

## 5. Освещение (блок 4)

Цель: ровный яркий свет, тёмных углов нет, мишени читаются с любой дистанции.

### 5.1 `env_sky` — источник неба для `light_environment`

| Key | Значение |
|-----|----------|
| Name | `sky_aim` |
| Sky Material | Любая светлая из `materials/skies/*.vmat` (Asset Browser, фильтр `sky`); для нейтрального света — светло-серая. При необходимости подкрась полем Sky Color/Tint. |
| yaw | `0` |

Без `env_sky` `light_environment` в CS2 не даёт света — частая причина «почему у меня всё чёрное».

### 5.2 `light_environment` — по ТЗ

| Key | Значение |
|-----|----------|
| Name | `sun_main` |
| **Brightness** | `255 255 255 200` (формат «R G B A», где A — множитель яркости/HDR-scale) |
| **Ambient** | `190 200 210 130` (indirect light из неба — убирает чёрные тени) |
| **Pitch Yaw Roll** | `-45 0 0` (pitch −45 = солнце под 45° к горизонту; −90 = строго сверху) |
| **Sky IBL Source** | `sky_aim` ← имя `env_sky` (ключевое поле CS2) |
| SunSpreadAngle | `2` (мягкие тени) |
| Shadow Texture Size | `2048` |
| Cast Shadows | Yes |
| Misc → Create Client-Only Entity | **No** |

После настройки: ПКМ по `light_environment` → **Estimate lighting from HDR skybox** — редактор сам подберёт brightness/ambient под выбранное небо (потом подправь под «нейтральный яркий»).
Имена полей сверяй с FGD своей версии: они могут отличаться на 1–2 символа.

### 5.3 Подстраховка baked-светом (чтобы точно не было тёмных углов)

4 × **`light_omni2`** под потолком, симметрично:

| Name | Origin | Brightness | Range / Max Distance | Режим |
|------|--------|-----------|----------------------|-------|
| fill_a | (-750, -750, 800) | `170 170 175` | 2500 | Baked |
| fill_b | (750, -750, 800) | `170 170 175` | 2500 | Baked |
| fill_c | (-750, 750, 800) | `170 170 175` | 2500 | Baked |
| fill_d | (750, 750, 800) | `170 170 175` | 2500 | Baked |

Если захочешь «лампы» под потолком визуально — добавь `prop_static` с моделью светильника и `light_rect` под ней. 4 baked-омни для коробки 3000×3000 — копеечная нагрузка и на FPS, и на время запекания.

### 5.4 `env_combined_light_probe_volume` — обязательно

Без него руки и оружие в первом лице **чёрные** (классическая болезнь CS2-карт).

| Key | Значение |
|-----|----------|
| Name | `probe_main` |
| Origin | (0, 0, 300) |
| **Box Mins** | `-1500, -1500, 0` |
| **Box Maxs** | `1500, 1500, 1000` |
| Сфера (радиус, через Transform → Scale) | перекрыть центр игровой зоны (~400–600) |

У этой сущности куб (объём влияния) и сфера (точка съёмки отражения) — **независимые параметры**, их легко перепутать. Тяни за разные хендлы.

### 5.5 `tools/toolslightmapres` — объём приоритета лайтмапы

Браш-коробка чуть внутри стен (например `±1480` по X/Y, `Z ∈ [0, 980]`), все грани — материал **`tools/toolslightmapres.vmat`**. Он подсказывает VRAD3, где тратить разрешение лайтмапы. Без него компилятор ругается: `No lightmap resolution volumes set for this geometry! ... texel resolution of the lightmap will be sub-optimal`.

---

## 6. Выдача оружия и респавн (блок 5)

### 6.1 Схема 1 — по ТЗ (`game_player_equip`)

**Энтити A — автоматическая выдача при спавне:**

| Key | Значение |
|-----|----------|
| Class | `game_player_equip` |
| Name | `equip_auto` |
| **SmartEdit = OFF**, добавь keyvalues: | `weapon_ak47` = `1`<br>`weapon_m4a1_silencer` = `1`<br>`weapon_awp` = `1`<br>`weapon_deagle` = `1`<br>`weapon_knife` = `1`<br>`item_assaultsuit` = `1` |
| Flags | ☑ **Strip All Weapons First** (2). ☐ Use Only — **выключен**, иначе автовыдачи на спавне не будет. |

Справка по флагам `game_player_equip`: `1` = Use Only, `2` = Strip All Weapons First, `4` = Only Strip Same Weapon Type.

**Энтити B — для ручных триггеров:**

| Key | Значение |
|-----|----------|
| Class / Name | `game_player_equip` / `equip_all` |
| Keyvalues | те же 6 (SmartEdit OFF) |
| Flags | ☑ Use Only + ☑ Strip All Weapons First → **spawnflags = 3** |

> Выдача AK + M4A1-S + AWP одновременно = игрок носит три «праймари» в инвентаре и переключает их клавишей 1/колёсиком. Для aim-карты это нормально и ожидаемо; если хочешь «одно оружие за раз» — используй стену выбора из раздела 4.5.

### 6.2 Схема 2 — рекомендуемая для CS2 (конвары через `point_servercommand`)

| Key | Значение |
|-----|----------|
| Class / Name | `point_servercommand` / `server_cmd` |

`logic_auto` → вкладка **Outputs**. Каждую команду — отдельным выходом с шагом **0.05 с** (одновременные команды `point_servercommand` теряет):

| # | Output | Target | Input | Parameter | Delay |
|---|--------|--------|-------|-----------|-------|
| 1 | OnMapSpawn | `server_cmd` | `Command` | `mp_warmup_end` | 0.00 |
| 2 | OnMapSpawn | `server_cmd` | `Command` | `mp_freezetime 0` | 0.05 |
| 3 | OnMapSpawn | `server_cmd` | `Command` | `mp_roundtime 60` | 0.10 |
| 4 | OnMapSpawn | `server_cmd` | `Command` | `mp_timelimit 0` | 0.15 |
| 5 | OnMapSpawn | `server_cmd` | `Command` | `mp_free_armor 2` | 0.20 |
| 6 | OnMapSpawn | `server_cmd` | `Command` | `mp_ct_default_primary weapon_ak47` | 0.25 |
| 7 | OnMapSpawn | `server_cmd` | `Command` | `mp_t_default_primary weapon_ak47` | 0.30 |
| 8 | OnMapSpawn | `server_cmd` | `Command` | `mp_ct_default_secondary weapon_deagle` | 0.35 |
| 9 | OnMapSpawn | `server_cmd` | `Command` | `mp_t_default_secondary weapon_deagle` | 0.40 |
| 10 | OnMapSpawn | `server_cmd` | `Command` | `mp_ct_default_melee weapon_knife` | 0.45 |
| 11 | OnMapSpawn | `server_cmd` | `Command` | `mp_t_default_melee weapon_knife` | 0.50 |
| 12 | OnMapSpawn | `server_cmd` | `Command` | `mp_buy_anywhere 1` | 0.55 |
| 13 | OnMapSpawn | `server_cmd` | `Command` | `mp_startmoney 60000` | 0.60 |
| 14 | OnMapSpawn | `server_cmd` | `Command` | `mp_maxmoney 60000` | 0.65 |
| 15 | OnMapSpawn | `server_cmd` | `Command` | `mp_respawn_on_death_ct 1` | 0.70 |
| 16 | OnMapSpawn | `server_cmd` | `Command` | `mp_respawn_on_death_t 1` | 0.75 |
| 17 | OnMapSpawn | `server_cmd` | `Command` | `mp_limitteams 0` | 0.80 |
| 18 | OnMapSpawn | `server_cmd` | `Command` | `mp_autoteambalance 0` | 0.85 |
| 19 | OnMapSpawn | `server_cmd` | `Command` | `sv_infinite_ammo 2` | 0.90 |
| 20 | OnMapSpawn | `equip_all` | `Use` | *(пусто)* | 1.00 |

`mp_free_armor 2` = кевлар + шлем каждому на респавне.
`mp_respawn_on_death_ct/t 1` = deathmatch-респавн: именно он делает карту «тренировочной», а не раундовой.
Если `sv_infinite_ammo`/другие чит-конвары не проходят — добавь первым выходом `sv_cheats 1` (и при необходимости `sv_allow_point_servercommand 1`).

### 6.3 Схема 3 — выдача при КАЖДОМ возрождении (финальная, ставь обе)

Поверх схемы 2 добавь триггер над зоной спавна:

| Key | Значение |
|-----|----------|
| Class | `trigger_multiple` |
| Name | `trg_respawn_equip` |
| Brush | `X ∈ [-950, 950]`, `Y ∈ [-1450, -1000]`, `Z ∈ [0, 256]`, материал — любой триггерный |
| Delay Before Reset | `0` |
| Filter Name | *(пусто — обе команды)* |

| Output | Target | Input | Param | Delay |
|--------|--------|-------|-------|-------|
| OnStartTouch | `equip_all` | `TriggerForActivatedPlayer` | *(пусто)* | 0.10 |

Задержка 0.10 — не укради её: `game_player_equip` часто «срезается» логикой респавна, если сработать в тот же тик.
Если пустой параметр не выдаёт всё — сделай 4 equip-энтити и передавай параметром конкретный ствол (`weapon_ak47` и т.д.).

**Итоговая цепочка (схема 3):**
```
Spawn игрока → mp_ct/t_default_* (конвары) ┐
                                           ├→ оружие + броня + нож
trg_respawn_equip (OnStartTouch +0.10) ────┘
Смерть → mp_respawn_on_death_ct/t 1 → мгновенный респавн → цепочка повторяется
```

---

## 7. Оптимизация (Source 2)

1. **Мишени, разметка, декор, движущиеся платформы → «Exclude from VIS = Yes»** в свойствах объекта. Это прямой аналог `func_detail` из Source 1 (которого в CS2 нет).
2. **Пропы** (`prop_static`/`prop_dynamic`) по умолчанию не участвуют в VIS — их не надо «исключать». Наоборот, для крупного укрытия можно включить **«Vis Occluder»**, чтобы оно реально резало видимость.
3. **Не дроби VIS лишней геометрией.** Одна коробка 3000×3000 = минимум VIS-кластеров. Каждая внутренняя стена = новые порталы. Спрей-стену и панели выбора оружия держи мешами с «Exclude from VIS».
4. **`func_viscluster` не нужен**: он полезен на больших открытых картах, здесь VIS считается секунды.
5. **Лайтмапа**: разрешение по умолчанию подходит (плоские большие поверхности дёшевы); объём `tools/toolslightmapres` направит разрешение в игровую зону.
6. **Материалы**: на всё невидимое — `tools/toolsnodraw` (не рендерится, не получает лайтмапу).
7. **Герметичность**: пол + 4 стены + потолок, углы внахлёст. Любая щель = `leak` = Visc/rad деградируют.

---

## 8. Чек-лист перед компиляцией

**Геометрия / герметичность**
- [ ] Все 6 брашей коробки на месте, углы перекрыты; VIS-проверка в редакторе не показывает луча в «войд».
- [ ] Ни одна энтити и ни один браш не торчат за пределы коробки.
- [ ] На невидимых гранях — `tools/toolsnodraw`.

**Спавны**
- [ ] 10 × `info_player_counterterrorist` (Z = 32, Y = -1200), 10 × `info_player_terrorist` (Z = 32, Y = -1260), шаг 100.
- [ ] 1 × `info_player_start` в (0, -1200, 32).
- [ ] У всех спавнов Angles = `0 90 0`; минимум 16 над полом, ≥32 между точками.

**Мишени**
- [ ] 20 статичных мишеней по таблице координат, центр Z = 64.
- [ ] У каждой: `func_button` + флаги Don't move / Damage Activates + `Exclude from VIS = Yes`.
- [ ] 3 × `func_movelinear` с парой выходов Open/Close и запуском из `logic_auto`.

**Свет**
- [ ] `env_sky` (`sky_aim`) со sky-материалом.
- [ ] `light_environment`: Brightness `255 255 255 200`, Pitch `-45`, **Sky IBL Source = `sky_aim`**.
- [ ] 4 × `light_omni2` подстраховки.
- [ ] `env_combined_light_probe_volume` перекрывает всю игровую зону (и куб, и сфера).
- [ ] Объём `tools/toolslightmapres` внутри стен.

**Логика**
- [ ] `game_player_equip` ×2 с правильными spawnflags (2 и 3) и 6 ключами.
- [ ] `point_servercommand` + 20 выходов из `logic_auto` с шагом 0.05 с.
- [ ] `trigger_multiple` над спавном → `TriggerForActivatedPlayer`, delay 0.10.
- [ ] Уникальные имена у всех энтити; нет опечаток в именах таргетов.
- [ ] `point_nav_walkable` стоит в центре арены (например, в (0, 0, 32)).

**Файл**
- [ ] Сохранён именно `aim_training_v1.vmap` (проверь заголовок окна, не `untitled_1.vmap`).

---

## 9. Компиляция

**Горячая клавиша: `F9`** (в меню Map этого пункта нет) → окно **Build Map**.

> **Видеокарта без RTX?** Свет запечь не получится, но карту можно собрать, запустить и тренироваться: `F9 → Custom Settings →` выключить **Generate Lightmaps**, материалы перевести на SelfIllum, в игре `mat_fullbright 1`. Полный разбор — [`aim_training_v1_no_rtx.md`](aim_training_v1_no_rtx.md).

| Режим | Когда | Что даёт |
|-------|-------|----------|
| **Fast Compile** | Каждая итерация по геометрии/мишеням | Без света, VIS и звука — быстро. Карта в игре будет тёмной: это ожидаемо. |
| **Full Compile** — Lightmap 1024, Quality Standard | Проверка света | Свет есть, качество черновое |
| **Full Compile** — **Lightmap 4096, Quality Final** | Релиз | Ниже 4096 запечённый свет может вообще не появиться в игре |

Требования и частые сбои:
- Нужен **GPU с аппаратным RT** (NVIDIA 2060+ / AMD 6600XT+). VRAD3 — GPU-рейтрейсер; на Ryzen со встройкой компиляция падает — отключить встройку в BIOS/диспетчере устройств.
- `Unable to run Vrad3 / light mapper didn't return valid triangles` — обнови или откати драйвер GPU; иногда компиляция из командной строки стабильнее, чем из Hammer.
- Компиляция вручную (полезно при сбоях Hammer): взять командную строку из окна Build Map → Settings и запустить `resourcecompiler.exe` в консоли, при желании добавив `-lightmapcpu`:
  ```
  resourcecompiler.exe -i "content\csgo_addons\aim_training_v1\maps\aim_training_v1.vmap" ^
    -world -bakelighting -phys -vis -nav -sareverb -retail -nop4 ^
    -lightmapMaxResolution 4096 -lightmapVRadQuality 2
  ```
- После релизной компиляции один раз перезапусти карту и дай отработать авто-кубмапам (или выполни `buildcubemaps`).

---

## 10. Тестирование: консольные команды

Готовый файл лежит рядом: **`cfg/aim_training_v1_test.cfg`** — скопируй его в
`...\game\csgo\cfg\` и выполни в консоли `exec aim_training_v1_test`.

**Загрузка карты** (обычный CS2, НЕ `-tools`):
```
map_workshop aim_training_v1 aim_training_v1
```

**Базовый запуск:**
```
sv_cheats 1
mp_warmup_end
bot_kick
mp_restartgame 1
```

**Оружие / броня / респавн (проверка логики из раздела 6):**
```
mp_free_armor 2
mp_ct_default_primary weapon_ak47
mp_t_default_primary weapon_ak47
mp_ct_default_secondary weapon_deagle
mp_t_default_secondary weapon_deagle
mp_respawn_on_death_ct 1
mp_respawn_on_death_t 1
mp_buy_anywhere 1
mp_freezetime 0
```

**Стрельба / спрей:**
```
sv_infinite_ammo 2          // 1 — бесконечный БК, 2 — бесконечный магазин
weapon_accuracy_nospread 1  // убрать разброс: чистый паттерн отдачи
sv_showimpacts 1            // видеть точки попаданий
cl_weapon_debug_show_accuracy 2
```

**Nav mesh (для ботов):**
```
sv_cheats 1
nav_edit 1
nav_generate
nav_save
mp_restartgame 1
```
Если `nav_generate` падает с `no walkable seeds present` — в карте должна стоять сущность `point_nav_walkable` (раздел 0/8).

**Боты (опция, вместо статичных мишеней):**
```
bot_kick
bot_add_ct
custom_bot_difficulty 2     // 0..3 — реальный рычаг сложности (bot_difficulty не влияет)
bot_stop 1                  // боты стоят: статичная мишень
bot_kick                    // убрать всех
```

**Диагностика entity I/O:**
```
developer 1                 // в логе видно цепочки outputs
cl_showfps 2
```

---

## 11. Если что-то не работает

| Симптом | Причина | Лечение |
|---------|---------|---------|
| Карта чёрная | Нет запечённого света (Fast Compile) или нет `env_sky` | Full Compile; проверить `env_sky` + `Sky IBL Source` у `light_environment` |
| `leak` в логе | Щель в коробке / энтити снаружи | Проверить углы, потолок; убрать всё, что за пределами |
| «Invalid spawnpoint» / «все укомплектованы» | Спавн вплотную к полу | Поднять origin минимум на 16 (ставь 32) |
| «No player start» | Нет `info_player_start` | Добавить |
| Оружие не выдаётся | Особенность `game_player_equip` в CS2 | Подстраховаться `mp_*_default_*` конварами + триггером с задержкой 0.10 |
| Чёрные руки / оружие | Нет Light Probe Volume | `env_combined_light_probe_volume` на всю зону |
| `no walkable seeds present` | Автоген nav не находит старт | `point_nav_walkable` |
| VRAD3 не запускается | GPU без RT / встройка / драйвер | Отключить встройку, обновить драйвер, компилировать из консоли |
| Боты тупят | `bot_difficulty` не влияет | `custom_bot_difficulty 0..3` |

---

## Приложение: полная сводка энтити

| Класс | Name | Ключевые параметры | Flags / свойства |
|-------|------|--------------------|------------------|
| `info_player_counterterrorist` | — | origin (X, -1200, 32), Angles `0 90 0` ×10 | — |
| `info_player_terrorist` | — | origin (X, -1260, 32), Angles `0 90 0` ×10 | — |
| `info_player_start` | — | (0, -1200, 32) | обязателен |
| `game_player_equip` | `equip_auto` | `weapon_ak47`1, `weapon_m4a1_silencer`1, `weapon_awp`1, `weapon_deagle`1, `weapon_knife`1, `item_assaultsuit`1 | Strip All Weapons First (2), Use Only выкл. |
| `game_player_equip` | `equip_all` | те же 6 ключей | Use Only + Strip All Weapons First (3) |
| `logic_auto` | — | 20 outputs (раздел 6.2) + запуск мишеней | — |
| `point_servercommand` | `server_cmd` | — | — |
| `trigger_multiple` | `trg_respawn_equip` | brush над спавном, Delay Before Reset 0 | OnStartTouch → `equip_all` → `TriggerForActivatedPlayer`, 0.10 |
| `env_sky` | `sky_aim` | Sky Material из `materials/skies/*.vmat` | — |
| `light_environment` | `sun_main` | Brightness `255 255 255 200`, Ambient `190 200 210 130`, Pitch Yaw Roll `-45 0 0`, **Sky IBL Source `sky_aim`**, SunSpreadAngle 2 | Create Client-Only Entity = No |
| `light_omni2` ×4 | `fill_a…fill_d` | (±750, ±750, 800), Brightness `170 170 175`, Range 2500 | Baked |
| `env_combined_light_probe_volume` | `probe_main` | Box Mins (-1500,-1500,0) / Maxs (1500,1500,1000) | сфера отдельно, через Scale |
| `func_button` ×20 | `tg_r*_n` | Health 1, Delay Before Reset 0, outputs OnDamaged | Don't move + Damage Activates; **Exclude from VIS = Yes** |
| `func_movelinear` ×3 | `mt_1000`, `mt_1500`, `mt_depth` | movedir, move_distance, speed | outputs OnFullyOpen/OnFullyClosed; **Exclude from VIS = Yes** |
| `func_buyzone` ×2 | — | Team Number 3 и 2 | триггерный браш над спавном |
| `point_nav_walkable` | — | (0, 0, 32) | для `nav_generate` |
| `math_counter` (опция) | `hits_counter` | max 0 (без лимита) | OnDamaged мишеней → `Add 1` |

---

## Источники (по ним сверены CS2-специфичные утверждения)

- Реальный рабочий проект CS2-карты с задокументированными граблями движка (спавны 16+ units, `info_player_start`, sealed box, `point_nav_walkable`, `custom_bot_difficulty`, `env_combined_light_probe_volume`, F9-компиляция, `map_workshop`): https://github.com/ALEXalesha/MyCS2Map
- `func_detail` отсутствует в Source 2, аналог — «Exclude from VIS»: https://steamcommunity.com/app/211/discussions/0/4353369456413071982/ · https://www.source2.wiki/EngineTools/HammerEditor/visibility
- `env_sky` + «Sky IBL Source» у `light_environment`, «Estimate lighting from HDR skybox»: https://www.reddit.com/r/hammer/comments/17qanzx/has_env_sun_been_removed_in_cs2/
- `game_player_equip` в CS2 и обходной путь через `mp_*_default_*` / `mp_free_armor`: https://www.reddit.com/r/hammer/comments/174qojf/cs2_game_player_equip_entity_not_working_new/ · https://totalcsgo.com/commands/mpctdefaultprimary · https://totalcsgo.com/commands/mpfreearmor
- Флаги `game_player_equip` (1 Use Only, 2 Strip All Weapons First, 4 Only Strip Same Weapon Type): https://www.reddit.com/r/SourceEngine/comments/grl6im/hammer_editor_csgo_game_player_equip/
- Список энтити CS2 (`func_brush`, `func_button`, `func_movelinear`, `func_tracktrain`, `path_track`, `logic_eventlistener`, `math_counter`, `light_omni2`, `light_rect`, `light_barn`…): https://oof.lv/custom/?page=custom&id=csentity
- Инструментальные материалы Source 2 (`tools/toolsnodraw`, `tools/toolsblockbullets`, `tools/toolsskybox`, `tools/toolslightmapres`): https://developer.valvesoftware.com/wiki/Tool_textures_(Source_2)
- Компиляция CS2 (F9, Fast/Full, ресурс-компилятор, lightmap 4096, RTX): https://profilerr.net/how-to-create-your-own-map-in-cs2/ · https://www.mapcore.org/forums/thread/26872-counter-strike-2-workshop-q-a/ · https://steamcommunity.com/app/730/discussions/0/594010527885244109/
- Ошибка «No lightmap resolution volumes set…»: https://www.reddit.com/r/hammer/comments/178ibpb/my_cs2_maps_are_not_compiling/
- `dev/dev_measuregeneric01*.vmat` существуют в CS2: https://www.reddit.com/r/hammer/comments/1cxbybn/cs2_sdk_compiling_issue/
- Nav mesh: `nav_generate` требует `sv_cheats 1` и `nav_edit 1`: https://developer.valvesoftware.com/wiki/Nav_mesh
