# Спецификация: создание Agent Skill `hd-cli`

Дата: 2026-05-13  
Репозиторий: `nimblemo/Human-Design-cli`  
Статус: draft (для реализации)  

Цель: описать, что именно должно быть создано в репозитории, чтобы появился Agent Skill, который:
1) устанавливает `hd-cli` (скачивает latest release для платформы клиента и кэширует локально, с fallback на сборку из исходников),
2) предоставляет агентам интерфейс запуска `hd-cli` (structured и raw),
3) ведёт персистентное хранилище людей/карт/расчётов/диалогов в виде Markdown-каталогов внутри директории скилла,
4) интегрируется с NotebookLM через `notebooklm-cli` и инжектит compact-данные карт в каждый запрос (без NotebookLM sources).

## 1. Область и принципы

### 1.1. Что делаем
- Добавляем скилл внутрь этого репозитория по пути `skills/hd-cli/`.
- Добавляем персистентное состояние в `skills/hd-cli/.state/` по паттерну Context State / Session Cache / Scratchpad File.
- Добавляем скрипты установки `hd-cli` из GitHub Releases (latest) и fallback-установки из исходников.
- Добавляем правила работы с `notebooklm-cli` в строгом соответствии с ограничениями `nlm-cli-skill` (см. ссылку ниже).

### 1.2. Что не делаем
- Не используем NotebookLM sources для карт (ни создание, ни обновление).
- Не храним никакие секреты/токены в репозитории или в `.state/`.
- Не используем интерактивные режимы `notebooklm-cli` (например `nlm chat start`).

## 2. Артефакты, которые должны появиться в репозитории

### 2.1. Файловая структура
```
Human-Design-cli/
  skills/
    hd-cli/
      SKILL.md
      .state/
        index.md
        calculations/
        people/
        dialogs/
      scripts/
        install.py
        platform.py
        github_release.py
        hd_cli_run.py
        hd_compact.py
        state_io.py
        nlm_query.py
      templates/
        person.md
        chart.md
        calculation.md
        dialog.md
        scratchpad.md
        artifact.calc.md
        artifact.notebooklm_query.md
        error.download_release.md
        error.local_install.md
```

### 2.2. Контракт `SKILL.md`
`skills/hd-cli/SKILL.md` должен описывать:
- назначение скилла и ограничения безопасности,
- интерфейсы/интенты (какие действия скилл выполняет для агента),
- расположение state и правила изменения файлов,
- правила взаимодействия с `hd-cli` и `nlm`.

Минимальный каркас (текст для `SKILL.md`, который нужно создать/поддерживать):
```
# hd-cli (Agent Skill)

## Назначение
- Рассчитывать Human Design chart через hd-cli.
- Вести локальное персистентное хранилище людей/карт/расчётов/диалогов.
- Делать запросы в NotebookLM через notebooklm-cli с инжектом JSON карт в каждый запрос.

## Важные ограничения
- Не создавать и не обновлять NotebookLM sources.
- Не использовать интерактивный режим notebooklm-cli (nlm chat start).
- Не писать секреты/токены в файлы и логи.

## Локальное состояние
- State root: skills/hd-cli/.state/
- Все сущности хранятся как Markdown + YAML frontmatter.
- Для каждого диалога есть scratchpad.md (Session Cache), который можно обновлять часто.

## Основные действия (интенты)
1) Установка hd-cli:
   - Попробовать скачать latest release под текущую платформу и закэшировать.
   - Если релиза нет/недоступен/нет ассета — объяснить причину и предложить установку из исходников.

2) Управление людьми:
   - Создать/обновить person сущность (birth_date, birth_time, utc_offset, lang).

3) Расчёт карты:
   - Запустить hd-cli в формате json.
   - Полученный JSON преобразовать в compact chart (без описательных полей) и сохранить как chart + calculation.

4) Управление диалогами:
   - Создать диалог с 1+ участниками.
   - Сразу создать calc-артефакт для каждого участника (или единый артефакт на диалог, см. спецификацию).
   - При вопросе пользователя сформировать request_text с инжектом chart_compact и вызвать:
     nlm notebook query <notebook_id> "<request_text>"
   - Сохранить artifact.notebooklm_query.md и обновить scratchpad.md.
```

## 3. Персистентное хранилище (`.state/`) и схемы сущностей

### 3.1. Общие правила
- Все сущности: Markdown + YAML frontmatter + секции.
- `id` в frontmatter обязателен.
- Время хранить как ISO-8601 (например `2026-05-13T10:15:30Z`).
- JSON данных карты хранить только в fenced-блоке `json` внутри секции `## Data`.

### 3.2. Дерево `.state/`
```
skills/hd-cli/.state/
  index.md
  people/
    <person-id>/
      person.md
      charts/
        <chart-id>.md
  calculations/
    <calc-id>.md
  dialogs/
    <dialog-id>/
      dialog.md
      scratchpad.md
      artifacts/
        <artifact-id>.md
```

### 3.3. Шаблоны Markdown (обязательный минимум)

#### Person: `people/<person-id>/person.md`
```
---
id: <person-id>
type: person
created_at: <iso8601>
updated_at: <iso8601>
display_name: <string>
birth_date: YYYY-MM-DD
birth_time: HH:MM
utc_offset: <number>
lang: ru
current_chart_id: <chart-id-or-empty>
chart_ids: []
---

## Notes
```

#### Chart: `people/<person-id>/charts/<chart-id>.md`
```
---
id: <chart-id>
type: chart
created_at: <iso8601>
updated_at: <iso8601>
person_id: <person-id>
calc_id: <calc-id>
hd_cli_version: <string>
compact: true
---

## Data

```json
{ }
```
```

#### Calculation: `calculations/<calc-id>.md`
```
---
id: <calc-id>
type: calculation
created_at: <iso8601>
updated_at: <iso8601>
person_id: <person-id>
chart_id: <chart-id>
hd_cli_version: <string>
input:
  birth_date: YYYY-MM-DD
  birth_time: HH:MM
  utc_offset: <number>
  lang: ru
---

## Links
- person: ../people/<person-id>/person.md
- chart: ../people/<person-id>/charts/<chart-id>.md
```

#### Dialog: `dialogs/<dialog-id>/dialog.md`
```
---
id: <dialog-id>
type: dialog
created_at: <iso8601>
updated_at: <iso8601>
title: <string>
notebook_id: c5dd30c7-da41-49a5-a0ce-74ed7ad7ce1b
participant_person_ids: []
artifact_ids: []
---

## Notes
```

#### Scratchpad (Session Cache): `dialogs/<dialog-id>/scratchpad.md`
```
---
id: <dialog-id>
type: scratchpad
updated_at: <iso8601>
notebook_id: c5dd30c7-da41-49a5-a0ce-74ed7ad7ce1b
conversation_id: <string-or-empty>
last_error: <string-or-empty>
attached:
  <person-id>:
    chart_id: <chart-id-or-empty>
    calc_artifact_id: <artifact-id-or-empty>
---

## Working Notes
```

#### Artifact: `dialogs/<dialog-id>/artifacts/<artifact-id>.md`
Frontmatter общий:
```
---
id: <artifact-id>
type: artifact
kind: calc | notebooklm_query
created_at: <iso8601>
dialog_id: <dialog-id>
attached_people:
  - person_id: <person-id>
    chart_id: <chart-id>
---
```

Для `kind=calc`:
```
## Calculation
person_id: <person-id>
chart_id: <chart-id>

## Data

```json
{ }
```
```

Для `kind=notebooklm_query`:
```
## User Query
<text>

## Request Text
<full request_text that was sent to nlm>

## Response Text
<nlm response>
```

## 4. Интеграция с `hd-cli`

### 4.1. Базовый вызов CLI
Нативные флаги `hd-cli` берутся из [cli.rs](file:///workspace/src/cli.rs#L37-L75).

Нормализованный вызов для расчёта карты:
- `hd-cli --date YYYY-MM-DD --time HH:MM --utc +3 --format json --lang ru`

### 4.2. Structured и raw режимы внутри скилла
- Structured:
  - скилл принимает параметры (date/time/utc/lang/short/format) и собирает корректный argv для `hd-cli`;
  - для интеграционных сценариев с NotebookLM всегда получать JSON.
- Raw:
  - скилл принимает строку аргументов и проксирует их в `hd-cli` как есть;
  - raw режим не должен записывать сущности в `.state/` автоматически (только если пользователь явно попросил).

### 4.3. Compact chart (строго без описаний)
Источник данных: JSON вывод `hd-cli --format json` (модель [HdChart](file:///workspace/src/models.rs#L55-L99)).

Требование: при использовании в NotebookLM и при сохранении в chart/calculation не включать текстовые описания.

Правило “compact”:
- удалять/не сохранять:
  - `type_description`, `profile_description`, `authority_description`, `strategy_description`, `cross_description`
  - `personality[].gate_name`, `personality[].gate_description`, `personality[].line_description`
  - `design[].gate_name`, `design[].gate_description`, `design[].line_description`
  - `channels[].description`
  - `centers[].behavior_normal`, `centers[].behavior_distorted`
  - `business|motivation|environment|diet|fear|sexuality|love|vision` (если эти блоки содержат описательные поля)
  - `circuit_scores` (содержит много описательного текста)

Хранимый минимум:
- `birth_date`, `birth_time`, `utc_offset`
- `type`/`profile`/`authority`/`strategy`/`incarnation_cross`
- `personality[]`: `planet`, `index`, `longitude`, `degree`, `zodiac_sign`, `gate`, `line`, `color`, `tone`, `base`
- `design[]`: то же
- `channels[]`: `key`, `name`
- `centers[]`: `name`, `defined`

## 5. Интеграция с NotebookLM через `notebooklm-cli`

### 5.1. Жёсткие правила `nlm-cli-skill`
Скилл обязан следовать правилам `nlm-cli-skill`:
- перед запросами проверять `nlm login --check`, иначе просить пользователя выполнить `nlm login`;
- не использовать интерактивные команды;
- использовать только `nlm notebook query`.

Источник правил: https://raw.githubusercontent.com/jacob-bd/notebooklm-cli/main/nlm-cli-skill/SKILL.md

### 5.2. Фиксированный notebook
- `notebook_id = c5dd30c7-da41-49a5-a0ce-74ed7ad7ce1b`
- URL: `https://notebooklm.google.com/notebook/c5dd30c7-da41-49a5-a0ce-74ed7ad7ce1b`

### 5.3. Инжект данных карт в каждый запрос (без sources)
Выбранная стратегия:
- не создавать и не обновлять sources;
- в каждый запрос добавлять JSON-блок(и) `chart_compact`.

Формат `request_text`:
1) `Participants: <person-id-1>, <person-id-2>, ...`
2) Для каждого участника:
   - `Chart(<person-id>):`
   - fenced `json` block с compact chart
3) `User question: <text>`

## 6. Основные сценарии (обязательное поведение скилла)

### 6.1. Создать человека
Результат:
- создаётся `people/<person-id>/person.md`.

### 6.2. Создать диалог с участниками
Если при создании диалога прикреплено 1+ людей, скилл обязан:
1) рассчитать карты для каждого участника через `hd-cli` (JSON),
2) преобразовать каждую карту в compact,
3) сохранить `chart.md` и `calculation.md`,
4) создать calc-артефакт(ы) в `dialogs/<dialog-id>/artifacts/`,
5) обновить `scratchpad.md` ссылками на актуальные calc-артефакты/карты.

Контракт: calc-артефакты должны быть готовы до первого вопроса пользователя, чтобы не делать “скрытый расчёт” в момент запроса к NotebookLM.

### 6.3. Задать вопрос в диалоге (NotebookLM)
Скилл обязан:
1) собрать `request_text` с инжектом актуальных `chart_compact`,
2) выполнить `nlm notebook query`,
3) сохранить `artifact kind=notebooklm_query` с:
   - user_query
   - request_text
   - response_text
   - attached_people (person_id + chart_id)
4) обновить `scratchpad.md` (включая `conversation_id`, если он используется/возвращается).

## 7. Установка `hd-cli` (latest release + fallback)

### 7.1. Платформа
- OS: linux | darwin | windows
- arch: x86_64
- platform key: `{os}-{arch}`

### 7.2. GitHub Releases (latest)
Источник: `GET https://api.github.com/repos/nimblemo/Human-Design-cli/releases/latest`

### 7.3. Контракт имён ассетов
- Linux: `hd-cli-v{version}-linux-x86_64.tar.gz`
- macOS: `hd-cli-v{version}-darwin-x86_64.tar.gz`
- Windows: `hd-cli-v{version}-windows-x86_64.zip`
- Контрольные суммы (если есть): `SHA256SUMS`

### 7.4. Кэш бинарника
- Cache root: `~/.cache/hd-cli/`
- Layout:
  - `~/.cache/hd-cli/{version}/hd-cli` (Linux/macOS)
  - `~/.cache/hd-cli/{version}/hd-cli.exe` (Windows)
- `~/.cache/hd-cli/current/hd-cli[.exe]` как активная версия (симлинк или копия).

### 7.5. Fallback на локальную установку из исходников
Если релизов нет/недоступен GitHub/нет ассета под платформу:
- скилл объясняет причину,
- предлагает шаги:
  - `git clone https://github.com/nimblemo/Human-Design-cli`
  - `cargo build --release` или `cargo install --path .`

## 8. Безопасность
- не логировать токены/credential’ы,
- не сохранять NotebookLM ответы или injected JSON в публичные места вне `.state/`,
- не выполнять сетевые запросы с токенами, кроме тех, что уже настроены у пользователя локально.

## 9. Критерии приемки (чеклист)
- Скилл размещён в `skills/hd-cli/` и содержит `SKILL.md`.
- Скрипты установки поддерживают “latest release” и корректный fallback на “из исходников”.
- В `.state/` создаются/обновляются сущности `person/chart/calculation/dialog/artifact` и `scratchpad.md`.
- При создании диалога с участниками создаются calc-артефакты (до первого вопроса).
- Каждый запрос в NotebookLM выполняется через `nlm notebook query` и содержит injected `chart_compact` (без sources).
