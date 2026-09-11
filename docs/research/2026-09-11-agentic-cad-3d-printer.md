# Агентный CAD для 3D-принтера: исследование и техпроцесс

Дата: 11 сентября 2026. Литера П (техническое предложение). Полная версия с матрицами, схемами и ~120 источниками опубликована как артефакт (ссылка в чате сессии).

## 0. Шесть выводов

1. **Парадигма: код + B-rep-ядро + цикл измерений.** LLM пишет build123d/CadQuery, ядро измеряет, агент правит. Все пять бенчмарков 2026 (CADGenBench, BenchCAD, CADBench, Text2CAD-Bench, Parametric CAD Bench) оценивают исполняемые программы или STEP. build123d-mcp поднял валидность того же LLM с 88 до 100 % (CADGenBench 0,360 → 0,457).
2. **GenCAD для этой задачи тупик.** Только эскиз + выдавливание, нет текста и размеров, нет сборок, ~10 % невалидных B-rep, репозиторий не обновлялся с июля 2025. Лаборатория авторов перешла на CAD-Coder / GIFT / CADFit (код + геометрическая обратная связь). Сохранить стоит только идею CCIP как поиск похожих деталей.
3. **Готовые детали подставляют, а не генерируют.** step.parts (16 847 деталей, OpenAPI, sha256), вендорские STEP (E3D, Bondtech, Noctua, Mean Well, HIWIN), bd_warehouse (Apache-2.0). CADEngBench: редактировать готовый CAD существенно легче, чем генерировать.
4. **Сборка — открытый фронтир.** У фронтирных моделей ноль строгих прохождений с первой попытки. Работают «интерфейсы раньше геометрии»: типизированные порты + исполняемые сопряжения (ASSEMCAD: 89 % против 0,8–67 % у чистых LLM), коннекторы до геометрии (ArtiCAD), требования как тесты (blueprint.yaml).
5. **Solid Edge: нет агентского API, но COM покрывает всё.** Design Copilot в SE 2026 — чат по справке. COM: импорт STEP, `AddByFilename`, `AddPlanar/AddAxial`, `CheckInterference`, Variable Table, чертежи. Единственный публичный мост — tylerwagler/solidedge-mcp (252 инструмента, февраль 2026, pywin32 извне).
6. **Рекомендация: гибрид, пилот на тулхеде.** Скелет и сборка нативно в SE, кастомные детали как build123d → STEP с лестницей проверок, стандартные детали из библиотеки, чертежи и спецификация через `solid-edge-draft`.

## 1. Исходная позиция

| Компонент | Состояние | Роль |
|---|---|---|
| GenCad (репо) | клон ferdous-alam/GenCAD, обучение не доведено | не генератор; архив или эксперимент с CCIP-поиском |
| скилл `cad` | build123d, STEP-first, `inspect refs/measure/mate/diff`, repair loop | ядро генерации + L0–L2 |
| скилл `step-parts` | клиент api.step.parts, sha256 | уровень 1 библиотеки |
| `render`, `sendcutsend` | viewer + снапшоты; DFM листа | визуальный контроль; панели |
| `solid-edge-draft` (свой) | PowerShell + VSTA + Add-In, ЕСКД, SE 2025 | стадия РКД |
| `snap-window` (свой) | скриншот окна SE | глаза на стороне SE |
| SE 2025 локально | Frames только 40×40/40×80; Design Configurator; ESKD Metric; Reports BOM/EXBOM | добавить профили 2020/1515 |
| не установлено | `dfam-check` из text-to-cad | уровень L3 |

Ограничение из опыта: в SE 2025 часть COM (`AddPartView`) работает только in-process (VSTA/Add-In). solidedge-mcp работает извне — проверить первым делом (фаза 0).

## 2. Ландшафт

| Подход | Представители | Данные | Вердикт |
|---|---|---|---|
| код + ядро + измерения | text-to-cad (15 341★), build123d-mcp, CADSmith, MAC (Tsinghua), Zoo Zookeeper, Autodesk Zero-to-CAD, MCP Anthropic для Fusion/Blender | CADSmith: exec 95→100 %, IoU 0,81→0,96; cadbench.ai v2: Claude Fable 5.1 + Claude Code 84,81 | основа |
| дообученные модели кода | CAD-Coder, CAD-Recode/cadrille, Text-to-CadQuery | лучше только в mesh→CAD; BenchCAD: плохо обобщаются | только реконструкция |
| последовательности команд | DeepCAD, GenCAD, CAD-Llama, CADFusion | Text2CAD-Bench: ухудшают валидность у всех моделей | не генератор |
| нейронный B-rep | HoLa, HiDiGen, SGS-1 | нет дерева и сборок | наблюдать |

Практика: Opus 4.1 → 4.6 на печатных кронштейнах 0 % → 10/10 (Staley, 02.2026); одна операция > 80 %, пять операций 3–30 % (Patrick, 04.2025); AADvark: ножницы 20 итераций, 4,1 ч, $15,85; MAC: JSON между ролями — 116× меньше токенов.

## 3. Solid Edge через COM

Высокая уверенность: подключение (`GetActiveObject`), импорт STEP (`Documents.Open`, `OpenWithTemplate`), переменные (`Variables`, единицы — метры), размещение (`Occurrences.AddByFilename`), сопряжения (`CreateReference` → `Relations3d.AddPlanar/AddAxial/...`), массы и `Range`, `CheckInterference(... IgnoreThreadInterferences)`, чертежи (`ModelLinks.Add` → `AddPartView`), `View.SaveAsImage`.
Средняя: выбор граней на импортированном STEP (нет стабильных имён → плоскости/СК/оси), Standard Parts (нет COM-объекта; библиотека — обычные .par), Excel-параметрика и Design Configurator.

Паттерны: jarvis-onshape-mcp (каждая мутация возвращает `{ok, error, changes, hints}`), SolidPilot (типизированный C# слой + тонкий Python MCP; persistent naming не решён), U-C4N discovery mode (40k → 356 токенов схемы), inventor-mcp (скилл с рецептами DIN-крепежа).

## 4. Библиотека компонентов

Баллы пригодности: step.parts 5, bd_warehouse 5, FreeCAD Fasteners (есть ГОСТ 1144/11860) 4, KiCad packages3D 4, TraceParts 3 (API, но без кэша), SE Standard Parts/Machinery Library (в т.ч. ГОСТ) 3, NopSCADlib 3 (GPL-3, справочник размеров), McMaster 2, MISUMI/igus/GrabCAD/Printables 1–2.

Уровни: 0 вендорские STEP (как есть, с хэшем) · 1 открытые каталоги (с SPDX) · 2 параметрические генераторы (bd_warehouse, свои таблицы для нагревателей/термисторов/ремней) · 3 только рантайм (TraceParts, McMaster).

`metadata.json`: id, aliases, category/family, vendor/mpn, source_url, license_spdx, sha256, units, origin_frame, envelope_bbox, mass_g, mount_patterns[], mate_frames[] (порты с СК), keepouts[], thermal_zones[], parametric_source. Поиск гибридный: лексика по алиасам/стандартам + эмбеддинг по образу над GLB.

Источники по подсистемам: хотэнд — E3D Reference CAD (Printables); вентиляторы — Noctua, SUNON (TraceParts); экструдер — Bondtech STEP, Voron (GPL-3); двигатели — LDO (GPL-3), step.parts; рельсы — HIWIN, TraceParts; профиль — MISUMI, bd_warehouse.open_builds; БП — Mean Well ZIP; платы — BTT GitHub, LDO; дисплей/концевики — Voron-Hardware.

Лицензии: Voron/LDO GPL-3, RatRig CC BY-NC-SA, TraceParts без кэша, вендорские STEP без явной лицензии. Коммерческое изделие → уровни 0, 2, step.parts, KiCad.

## 5. Пакет задания и карточка состояния

Паттерны: интерфейсы раньше геометрии; типизированные порты с СК + детерминированные сопряжения; фабрики компонентов; JSON-артефакты между ролями; воронка проверок (детерминированные → VLM); инструментированные рендеры с метками; локальная правка; правила как цитируемые артефакты.

```yaml
task: toolhead_duct_v1
units: mm
components:                      # только чтение
  - id: hotend
    step: lib/e3d_revo_six.step
    ports:
      - {name: groove_mount, type: FLAT_FACE, frame: [[0,0,0],[0,0,1]]}
      - {name: heater_block, type: KEEPOUT, temp_c: 300, offset: 2.0}
  - id: fan_pc
    step: lib/sunon_5015_blower_24v.step
    ports:
      - {name: outlet, type: FLAT_FACE, frame: [[25,0,0],[1,0,0]], size: [20,15]}
      - {name: bolt_1, type: BOLT_SEAT, thread: M3, pattern: "2×M3 @ 43"}
design_domain:
  keep_in:  {bbox: [[-30,-25,-40],[30,25,10]]}
  keep_out: [hotend.heater_block, belt_path_A, belt_path_B]
parts_to_design:
  - id: duct
    role: "воздуховод от fan_pc.outlet к соплу, два потока"
    mates: [face_to_face(fan_pc.outlet, duct.inlet), coaxial(fan_pc.bolt_1, duct.hole_1)]
    params: {wall: [1.2, 2.0], outlet_area_mm2: ">= 60"}
loads_and_environment: {chamber_temp_c: 60, material: ASA}
manufacturing: {process: FDM, overhang_max_deg: 45, min_wall: 0.8, supports: none, orientation: +Z}
verification:
  - interference_volume(duct, *) == 0
  - min_distance(duct, hotend.heater_block) >= 2.0
  - hole_pattern(duct.holes, fan_pc.bolt_*) within 0.1
  - printable(duct, overhang_max_deg=45, min_wall=0.8)
  - mass(duct) <= 12
rules: [voron_dfam_v2, ercf_dev_notes]
deliverables: [step, assembly_yaml, report.json, renders]
budget: {max_iterations: 8, max_questions: 2}
```

Карточка состояния после каждого круга: таблица «проверка / измерено / требование / статус / действие», рендеры с метками экземпляров. Число вопросов ограничено (ICED 2025: режим «переспрашивать» — худший).

## 6. Лестница проверок

| Уровень | Что | Инструмент | Опора |
|---|---|---|---|
| L0 компиляция | скрипт исполняется | build123d в подпроцессе с таймаутом | CAD-Recode, MUSE |
| L1 валидность | одно тело, объём > 0, watertight | `is_valid()`, trimesh | CADGenBench (жёсткие ворота) |
| L2 интерфейсы | конверт, отверстия, keep-in/out, зазоры | предикаты CADTests; `measure/mate` из `cad`; DistShapeShape; булево | CADTests (F1 0,962 vs 0,549), build123d-mcp, cad-khana |
| L3 печатаемость | свесы, стенки, мосты, вставки | Blender 3D-Print Toolbox / trimesh; истина — OrcaSlicer CLI (`slice_info.config`) в контейнере; лист — `sendcutsend` | AgentsCAD, ERCF-правила |
| L4 клэш сборки | интерференция, соосность, тепловые keep-out | OCC `Common`; финал — SE `CheckInterference` | MUSE, cad-khana |
| L5 физика | запас, прогиб, расход | аналитика → gmsh + CalculiX; OpenFOAM по запросу | Physics-in-the-Loop (81–87 % FEA-pass) |
| L6 человек | подпись | 4 вида + сечения, чертёж, сводный JSON, диф | CADCodeVerify |

Усиления: консенсус из N кандидатов (−1–10 % ошибки), уточнение спецификации до кода (ProCAD). Пробелы: размерные цепи, тепловые keep-out, кабели — кодировать вручную.

## 7. Предметная область

Номенклатура по подсистемам (рама 2020/1515; гантри MGN9/12, NEMA17, GT2, F695; Z; тулхед: хотэнд, нагреватель 24 В, термистор, 4010, 5015, экструдер, CAN-плата, зонд; стол MIC6 + AC-нагреватель + SSR; электроника Octopus/Leviathan, LRS-200-24, DIN; корпус: акрил 3 мм, клипсы; печатные детали ~2 кг ABS/ASA).

Правила печати как предикаты: свесы ≤ 45°, без поддержек, ориентация в STL; отверстия Ø + 0,4, резьба в пластике Ø − 0,1, натяг + 0,2, вставки Ø 4,7 глухое 5 мм с фаской 0,4; фаска 0,4 на кромках к столу; M3×6…40, M5×10…40; именование `[a]_`, `[o]_`, `[c]_`, `_x4`. Рецепт печати: 0,2 мм, 4 стенки, 40 %, 5 верх/низ.

Практика проектов: релиз = CAD + STEP + STL по подсистемам + мануал + BOM/конфигуратор + Klipper + профили + тесты + оснастка; версионирование по деталям (Annex R1.2B1, Voron Superceded_Parts); производственную инженерию делают поставщики китов (LDO); ферма Prusa 27 ч/комплект; NopSCADlib/Mendel90 — единственный законченный генератор (переносим таксономию, не код).

## 8. Техпроцесс (ГОСТ 2.103 × ЕСТД)

| № | Стадия | Выход | Автоматизация |
|---|---|---|---|
| 0 | ТЗ | ТЗ | высокая |
| 1 | Техпредложение (П): матрица вариантов, подсистемы, витамины | ПЗ | высокая, выбор за человеком |
| 2 | Эскизный (Э): скелет в SE, Excel-переменные, Frames 2020, COTS по плоскостям | ВО, ГЧ, variables.xlsx | высокая (inter-part только Ordered) |
| 3 | Подбор компонентов + правила (T-slot, БП, нагреватель/SSR, драйверы, CAN 5/24 В, зонд) | ВП, закупка | высокая |
| 4 | Технический (Т): свои детали по DfAM, панели DXF, ЧПУ STEP | модели, STL, DXF | средне-высокая |
| 5 | Проверка: интерференция, ход осей, кабели, масса, Tg, FEA, усадка | отчёт, ПМ | высокая (геометрия), низкая (физические тесты) |
| 6 | РКД (О→О1): чертежи, СБ, спецификация ГОСТ Р 2.106, ВП, ТУ | КД | высокая |
| 7 | Техпроцесс ЕСТД: МК/ОК по маршрутам печать/профиль/лист/жгуты/сборка | МК, ОК, КЭ, КК, ВО | высокая (нормы из цеха) |
| 8 | Изготовление: платтеры, ОТК, профиль, панели, жгуты | платтеры, листы | средняя |
| 9 | Сборка и испытания: рама→Z→гантри→тулхед→стол→электроника→панели; Klipper, PID, shaper; EVT→DVT→PVT | инструкция, ПМ, протокол | высокая (документы) |
| 10 | Выпуск и изменения | release notes, ИИ | высокая |

Контур: пакет задания → агент (план→код→самопроверка, JSON между ролями) → build123d → STEP → лестница L0–L4 → Solid Edge (скелет, сборка, CheckInterference, чертежи, спецификация) → производство. Библиотека питает агента и SE. Два цикла обратной связи: диагностика к агенту, карточка состояния к инженеру.

## 9. Варианты архитектуры

- **A. Только нативный API** (форк solidedge-mcp): + дерево построения, − хрупкие индексы граней, COM извне, нет API стандартных деталей.
- **B. Только код → STEP**: + быстро и тестируемо, − «немые» тела в SE, правки только регенерацией.
- **C. Гибрид (рекомендуется)**: скелет/переменные/простые детали нативно в SE, сложные и одноразовые — build123d → STEP, стандартные — STEP/.par по плоскостям и СК, чертежи через `solid-edge-draft`, проверка — CheckInterference + массы + скриншот. Правило маршрутизации: сложные каналы/органика/частые правки → build123d; пластины/уголки/массивы отверстий от скелета → SE; стандартные никогда не генерируются.

## 10. Дорожная карта (пилот — тулхед)

0. Спайк 1–2 дня: solidedge-mcp против SE 2025 — что работает извне, что требует Add-In.
1. Библиотека v0: 10–15 компонентов тулхеда/электроники с metadata.json (порты, keep-out, тепловые зоны); индекс step.parts; крепёж bd_warehouse.
2. Пакет задания + лестница L0–L4 на build123d; консенсус из N; `dfam-check`; OrcaSlicer CLI в контейнере.
3. Мост в SE: размещение по плоскостям/СК, сопряжения, CheckInterference, SaveAsImage; карточка состояния.
4. Документация: `solid-edge-draft`, parts list/Reports → спецификация, cut-list, DXF панелей с `sendcutsend`, STL с ориентацией.
5. Пилот целиком: метрики — доля предикатов с первой попытки, итерации, время/токены, правки инженера (ориентир 4–6 итераций на деталь).
Позже: FEA-lite (gmsh + CalculiX), Design Configurator для размерных вариантов, CCIP-поиск по образу, МК/ОК по ЕСТД.

## 11. Риски и решения

Риски: COM извне на SE 2025; имена граней на STEP; лицензионное заражение (GPL-3, NC, TraceParts); стоимость циклов; OrcaSlicer CLI на Windows; «немые» тела; нет исследований по тепловым keep-out/кабелям/размерным цепям.

Решения от владельца проекта: (1) подтвердить гибрид C; (2) судьба репозитория GenCad; (3) пилотная подсистема (рекомендация — тулхед); (4) форма моста в SE после спайка; (5) лицензионная политика изделия; (6) серия и маршруты изготовления, нужен ли ЕСТД полностью.

## 12. Ключевые источники

text-to-cad https://github.com/earthtojake/text-to-cad · CADGenBench https://github.com/huggingface/cadgenbench · build123d-mcp https://github.com/pzfreo/build123d-mcp · CADSmith https://arxiv.org/abs/2603.26512 · MAC https://github.com/Pan-Chera/Multi-Agent-CAD · Physics-in-the-Loop https://arxiv.org/abs/2605.19717 · FEA-агенты https://arxiv.org/abs/2605.17448 · ASSEMCAD https://arxiv.org/abs/2607.05123 · ArtiCAD https://arxiv.org/abs/2604.10992 · Embodied CAD https://arxiv.org/abs/2606.31252 · AADvark https://arxiv.org/abs/2604.15184 · BenchCAD https://arxiv.org/abs/2605.10865 · CADBench https://arxiv.org/abs/2605.10873 · Text2CAD-Bench https://arxiv.org/abs/2605.18430 · Parametric CAD Bench https://cadbench.ai/ · CADEngBench https://arxiv.org/abs/2608.09296 · MUSE https://arxiv.org/abs/2605.28579 · CADTests https://arxiv.org/abs/2605.07807 · CADCodeVerify https://arxiv.org/abs/2410.05340 · cad-khana https://github.com/cyberchitta/cad-khana · консенсус https://arxiv.org/abs/2608.09706 · AgentsCAD https://arxiv.org/abs/2607.02448 · GenCAD https://github.com/ferdous-alam/GenCAD · CAD-Coder https://github.com/anniedoris/CAD-Coder · solidedge-mcp https://github.com/tylerwagler/solidedge-mcp · SE 2026 AI https://blogs.sw.siemens.com/solidedge/designcenter-solid-edge-2026-artificial-intelligence/ · SE CheckInterference https://support.industrysoftware.automation.siemens.com/trainings/se/107/api/SolidEdgeAssembly~AssemblyDocument~CheckInterference.html · jarvis-onshape-mcp https://github.com/ReshefElisha/jarvis-onshape-mcp · SolidPilot https://github.com/eyfel/mcp-server-solidworks · step.parts https://www.step.parts/ · bd_warehouse https://github.com/gumyr/bd_warehouse · FreeCAD Fasteners https://github.com/shaise/FreeCAD_FastenersWB · NopSCADlib https://github.com/nophead/NopSCADlib · KiCad packages3D https://github.com/KiCad/kicad-packages3D · TraceParts API https://developers.traceparts.com/v2/ · Voron 2.4 CAD https://github.com/VoronDesign/Voron-2/tree/Voron2.4/CAD · Voron-Hardware https://github.com/VoronDesign/Voron-Hardware · VoronUsers rules https://github.com/VoronDesign/VoronUsers/wiki/Mod-Submission-Rules · ERCF Dev Notes https://github.com/Enraged-Rabbit-Community/ERCF_v2/blob/master/Assets/Dev_Notes.md · FYSETC BOM https://github.com/FYSETC/FYSETC-Voron-2/blob/main/BOM.md · Prusa farm https://blog.prusa3d.com/a-quick-look-to-our-printing-farm_7474/ · OrcaSlicer CLI https://printago.io/blog/orca-slicer-cli-reference · Blender 3D-Print Toolbox https://github.com/blender/blender-addons/blob/main/object_print3d_utils/operators.py · ГОСТ 2.103-2013 https://allgosts.ru/01/100/gost_2.103-2013 · ГОСТ Р 2.106-2019 https://allgosts.ru/01/100/gost_r_2.106-2019 · ГОСТ 3.1102-2011 https://files.stroyinf.ru/Data2/1/4293800/4293800205.htm · Staley 2026 https://kerrickstaley.com/2026/02/22/can-frontier-llms-solve-cad-tasks · Patrick 2025 https://willpatrick.xyz/technology/2025/04/23/teaching-llms-how-to-solid-model.html
