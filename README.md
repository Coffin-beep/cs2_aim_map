# cs2_aim_map

Проект карты для тренировки стрельбы в Counter-Strike 2: **`aim_training_v1`** (Hammer / Source 2).

## Что внутри

- [`docs/aim_training_v1_guide.md`](docs/aim_training_v1_guide.md) — полный план сборки: пошаговая инструкция по Hammer, точные KeyValues и координаты всех энтити, логика выдачи оружия и респавна, чек-лист перед компиляцией, консольные команды для теста, таблица типовых ошибок.
- [`docs/aim_training_v1_run.md`](docs/aim_training_v1_run.md) — полный ранбук запуска: установка Workshop Tools, создание аддона, компиляция, три способа запустить карту, приёмочный обход, цикл правок, публикация в Workshop.
- [`cfg/aim_training_v1_test.cfg`](cfg/aim_training_v1_test.cfg) — готовый тестовый конфиг (положить в `...\game\csgo\cfg\`, запускать `exec aim_training_v1_test`).
- [`docs/aim_training_v1_no_rtx.md`](docs/aim_training_v1_no_rtx.md) — что делать, если видеокарта без аппаратного ray tracing (нет RTX): как всё равно собрать, запустить и оттестировать карту и где потом один раз запечь свет.

## Кратко про карту

| Параметр | Значение |
|----------|----------|
| Имя аддона / файла | `aim_training_v1` / `aim_training_v1.vmap` |
| Размер арены | 3000 × 3000 units (X/Y: −1500…1500), высота 1000 |
| Линия огня | `Y = -1200`, смотрит в `+Y` |
| Спавны | 10 CT (`Y = -1200`) + 10 T (`Y = -1260`), `Z = 32`, шаг 100 по X, + 1 `info_player_start` |
| Мишени | 20 статичных (`func_button`, ряды 500 / 1000 / 1500 / 2000 units) + 3 движущиеся (`func_movelinear`) |
| Свет | `env_sky` + `light_environment` (Brightness `255 255 255 200`, Pitch −45, Sky IBL Source) + 4 `light_omni2` + `env_combined_light_probe_volume` |
| Оружие | `game_player_equip` + конвары `mp_ct/t_default_*`, `mp_free_armor 2`, `mp_respawn_on_death_ct/t 1` |

## Важно про Source 2

В CS2 **нет `func_detail`** (аналог — галочка «Exclude from VIS»), `light_environment` светит только через **`env_sky`**, а свет считается **VRAD3 на GPU с RTX**. Подробности и обходные пути — в разделе 0 руководства.

## Быстрый старт

1. Steam → CS2 → Properties → DLC → **Counter-Strike 2 Workshop Tools**.
2. Аддон `aim_training_v1`, карта `content\csgo_addons\aim_training_v1\maps\aim_training_v1.vmap`.
3. Собрать по [`docs/aim_training_v1_guide.md`](docs/aim_training_v1_guide.md).
4. Компиляция — **F9** → Build Map (итерации: Fast Compile, релиз: Full Compile, Lightmap 4096, Quality Final).
5. Тест в обычном CS2: `map_workshop aim_training_v1 aim_training_v1`.
