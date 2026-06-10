# Vibe Typing Plugin + /how-to Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Перетворити репо practicum на Claude Code-плагін «vibe-typing» з командами `/vibe-typing:practicum` (як було) і `/vibe-typing:how-to` (курс по існуючому проекту).

**Architecture:** Репо перейменовується на `~/Programming/vibe-typing`, отримує `.claude-plugin/plugin.json` + `marketplace.json` (локальна установка). Спільні правила генерації уроку виносяться з practicum/SKILL.md у `skills/practicum/lesson-format.md`; нова навичка `skills/how-to/SKILL.md` (stateful курс через `.practicum/course.md`) посилається на те саме ядро. template.html не змінюється. Спека: `docs/superpowers/specs/2026-06-10-vibe-typing-plugin-howto-design.md`.

**Tech Stack:** Claude Code plugin (plugin.json, commands, skills), markdown-інструкції, python3 для перевірок.

**Тестова стратегія:** Автотестів немає (рішення спеки). Кожна задача має крок перевірки: golden build template+sample (UI незмінний — має працювати як до міграції), перевірка повноти перенесених правил, верифікація установки плагіна. Не пропускай кроки перевірки.

---

### Task 1: Перейменування репо + маніфести плагіна

**Files:**
- Rename: `/home/jarvis/Programming/practicum` → `/home/jarvis/Programming/vibe-typing`
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Перейменувати директорію репо**

```bash
mv /home/jarvis/Programming/practicum /home/jarvis/Programming/vibe-typing
cd /home/jarvis/Programming/vibe-typing && git status --short && git log --oneline | head -2
```

Expected: git працює, дерево чисте. Старі симлінки в `~/.claude/` від цього
повисли (dangling) — це очікувано, Task 4 їх прибере.

- [ ] **Step 2: Створити `.claude-plugin/plugin.json`**

```json
{
  "name": "vibe-typing",
  "description": "Навчальний typing-тренажер: уроки з diff чекпоінта (practicum) та курси по існуючому проекту (how-to)",
  "version": "0.2.0",
  "author": { "name": "Maxim Rabchun", "email": "semtexfromua001@gmail.com" }
}
```

- [ ] **Step 3: Створити `.claude-plugin/marketplace.json`**

```json
{
  "name": "vibe-typing-local",
  "owner": { "name": "Maxim Rabchun" },
  "plugins": [
    {
      "name": "vibe-typing",
      "source": "./",
      "description": "Vibe Typing — навчальний typing-тренажер (practicum, how-to)"
    }
  ]
}
```

- [ ] **Step 4: Перевірити валідність JSON**

```bash
python3 -c "import json; json.load(open('.claude-plugin/plugin.json')); json.load(open('.claude-plugin/marketplace.json')); print('json ok')"
```

Expected: `json ok`.

- [ ] **Step 5: Commit**

```bash
git add .claude-plugin
git commit -m "feat: add vibe-typing plugin manifest and local marketplace"
```

---

### Task 2: Спільне ядро lesson-format.md + скорочення practicum/SKILL.md

**Files:**
- Create: `skills/practicum/lesson-format.md`
- Rewrite: `skills/practicum/SKILL.md`

Працювати з `/home/jarvis/Programming/vibe-typing`.

- [ ] **Step 1: Створити `skills/practicum/lesson-format.md`** — точний зміст:

````markdown
# Формат уроку Vibe Typing — спільне ядро

Спільні правила генерації уроку для навичок practicum і how-to.
Викликаюча навичка дає: відібрані фрагменти коду та ІМ'Я файла уроку.

## Обмеження коду фрагментів

- Таби заміни на 4 пробіли. Трейлінг-пробілів бути не повинно.
- Без emoji та символів поза BMP (> U+FFFF) — двигун друку їх не підтримує.
- Сумарно на урок: 2500–4000 символів коду (~10–20 хв друку).

## Пояснення фрагмента (explanation)

- Українською, 2–4 речення на фрагмент, рівень початківця.
- Відповідай на два питання: «що це» і «навіщо саме тут у проєкті».
- Терміни без розшифровки не вживай: назвав термін — одразу поясни його
  простими словами.

## Покрокові пояснення (steps)

- Розбий код фрагмента на логічні блоки по 1–5 рядків: оголошення,
  ініціалізація, умова, цикл, повернення тощо. Блоки йдуть підряд
  і разом покривають увесь фрагмент.
- Для кожного блоку — `note`: 1–2 речення «як дитині»: що ми зараз
  пишемо і навіщо воно тут. Без жаргону; якщо термін потрібен —
  одразу поясни його простими словами. Нотатка з'являється поруч із
  кодом, поки користувач друкує саме цей блок.
- `start`/`end` — офсети блоку в `code`, вирівняні по межах рядків:
  `start` — початок рядка, `end` — початок наступного блоку
  (або кінець коду для останнього).

## lesson-JSON

```json
{
  "title": "коротка тема уроку",
  "project": "назва проєкту",
  "date": "YYYY-MM-DD",
  "fragments": [
    {
      "file": "шлях/до/файла.py",
      "explanation": "2–4 речення українською",
      "code": "повний текст фрагмента з \\n",
      "steps": [
        { "start": 0, "end": 0,
          "note": "1–2 речення: що пишемо в цьому блоці і навіщо" }
      ]
    }
  ]
}
```

`start`/`end` — офсети в `code`: блок — це `code[start:end]`.

## Перевірка офсетів (обов'язково)

Збережи JSON у тимчасовий файл і виконай:

```bash
python3 - <<'EOF'
import json
lesson = json.load(open('/tmp/practicum-lesson.json'))
for f in lesson['fragments']:
    code = f['code']
    assert '\t' not in code, 'таби мають бути замінені пробілами'
    assert all(ord(c) <= 0xFFFF for c in code), 'символи поза BMP не підтримуються'
    prev_end = 0
    for s in f['steps']:
        assert 0 <= s['start'] < s['end'] <= len(code), 'офсети поза межами'
        assert s['start'] == prev_end, 'кроки мають іти підряд без дірок і перетинів'
        assert s['start'] == 0 or code[s['start'] - 1] == '\n', \
            'крок має починатись з початку рядка'
        assert s['note'].strip(), 'порожня нотатка'
        prev_end = s['end']
    assert prev_end == len(code), 'кроки мають покривати весь фрагмент'
print('offsets ok')
EOF
```

Якщо `offsets ok` нема — виправ офсети й повтори. Без цієї перевірки
HTML не збирай.

## Збирання HTML

1. Прочитай `template.html` — він лежить ПОРЯД із цим файлом
   (`skills/practicum/`).
2. Заміни рядок `__LESSON_JSON__` на вміст lesson-JSON, попередньо
   замінивши в ньому `</` на `<\/` (валідний JSON-escape; без цього
   `</script>` у коді фрагмента обірве скрипт-блок сторінки).
3. Збережи у `.practicum/lessons/<ім'я від викликаючої навички>.html`
   (директорію створи).
4. Якщо `.practicum/` нема в `.gitignore` проєкту — запропонуй
   користувачу додати (не додавай сам).
5. Відкрий: `xdg-open <файл>` і скажи користувачу, що урок готовий.
````

- [ ] **Step 2: Переписати `skills/practicum/SKILL.md`** — точний повний зміст:

````markdown
---
name: practicum
description: Згенерувати з diff чекпоінта самодостатній HTML typing-практикум — передрук коду з простими поясненнями українською та покроковим гайдом по блоках. Використовуй для команди /vibe-typing:practicum.
---

# Practicum — урок з diff чекпоінта

Результат: один HTML-файл `.practicum/lessons/YYYY-MM-DD-<slug>.html`
у поточному проєкті, відкритий у браузері. Жодних інших артефактів.

## Крок 1: зібрати diff

- З аргументом (діапазон): `git diff <діапазон>`.
- Без аргументу: `git diff HEAD` (незакомічені зміни); якщо порожньо —
  `git show HEAD --patch` (останній коміт).
- Нові untracked-файли в `git diff HEAD` не видно — перевір
  `git status --porcelain` і додай їхній вміст до розгляду.
- Не git-репо або diff порожній → повідом користувачу й зупинись.
  Урок НЕ генеруй.

## Крок 2: відібрати фрагменти

- 3–7 фрагментів (ліміт обсягу — у lesson-format.md).
- Пріоритет: нова бізнес-логіка > нові патерни/ідіоми > решта.
- Пропускай: імпорти, конфіги, boilerplate, згенерований код, механічні
  повтори, чисті перейменування.
- Фрагмент — цілісний шматок (функція/метод/клас/зв'язний блок), що
  читається без решти файлу. Бери ФІНАЛЬНИЙ стан коду з файлу, а не
  рядки diff з +/-. Якщо фрагмент довший за потрібне — скороти до ядра.

## Кроки 3–7: згенерувати урок

Прочитай `lesson-format.md` (лежить поряд із цим файлом) і виконай усе
звідти: обмеження коду, пояснення, покрокові нотатки, lesson-JSON,
перевірку офсетів, збирання HTML.

Ім'я файла уроку: `YYYY-MM-DD-<slug>.html` (slug — латиницею з дефісами).
````

- [ ] **Step 3: Перевірити, що жодне правило не загубилось**

```bash
for phrase in "4 пробіли" "BMP" "2500–4000" "як дитині" "покривають увесь фрагмент" "offsets ok" "<\\\\/" "xdg-open" ".gitignore" "git status --porcelain" "ФІНАЛЬНИЙ стан"; do
  grep -ql "$phrase" skills/practicum/SKILL.md skills/practicum/lesson-format.md \
    && echo "OK: $phrase" || echo "MISSING: $phrase"
done
```

Expected: усі рядки `OK:`. Якщо `MISSING:` — знайди правило в git-історії
(`git show HEAD:skills/practicum/SKILL.md`) і додай у відповідний файл.

- [ ] **Step 4: Golden build — UI не зламався**

```bash
python3 - <<'EOF'
from pathlib import Path
import json
tpl = Path('skills/practicum/template.html').read_text()
lesson = Path('sample-lesson.json').read_text()
json.loads(lesson)
Path('/tmp/practicum-test.html').write_text(tpl.replace('__LESSON_JSON__', lesson.replace('</', '<\\/')))
print('built')
EOF
```

Expected: `built` (template не чіпали — це регресійна перевірка).

- [ ] **Step 5: Commit**

```bash
git add skills/practicum/SKILL.md skills/practicum/lesson-format.md
git commit -m "refactor: extract shared lesson-format.md core from practicum skill"
```

---

### Task 3: Команда /how-to + навичка

**Files:**
- Create: `commands/how-to.md`
- Create: `skills/how-to/SKILL.md`

Працювати з `/home/jarvis/Programming/vibe-typing`.

- [ ] **Step 1: Створити `commands/how-to.md`** — точний зміст:

```markdown
---
description: Курс typing-уроків по поточному проекту — навчитись писати його з нуля
argument-hint: "[фокус-шлях, напр. src/api]"
---

Використай навичку how-to (Skill tool), щоб створити або продовжити курс
typing-уроків по цьому проекту. Фокус від користувача: $ARGUMENTS
```

- [ ] **Step 2: Створити `skills/how-to/SKILL.md`** — точний зміст:

````markdown
---
name: how-to
description: Курс typing-уроків по існуючому проекту — «як написати його з нуля»: аналіз → план у .practicum/course.md → покрокова генерація уроків. Використовуй для команди /vibe-typing:how-to.
---

# How-to — курс по існуючому проекту

Стан курсу: `.practicum/course.md` у корені проекту. Один виклик — один
крок: створення плану + урок 01, або наступний невідмічений урок.
Аргумент (опціонально): фокус-шлях — курс лише по цій підсистемі.

## Якщо `.practicum/course.md` НЕМА — побудуй курс

### 1. Аналіз проекту

- Зрозумій стек і структуру: точки входу, модулі, залежності між ними.
- З фокус-аргументом аналізуй лише вказану підсистему.
- Порожній проект або нема читних файлів коду → повідом і зупинись.

### 2. План уроків

- Топологічний порядок: фундамент → залежності → верхівка. Модуль вчимо
  ПІСЛЯ того, від чого він залежить. Приклад FastAPI: .env+settings →
  db → main/lifespan → моделі → схеми → роутери.
- Відбір змісту:
  - однотипне (N схожих ендпоінтів/CRUD-ів) → 1–2 представники типу;
  - патерн (generic repository тощо) → сам патерн + приклад використання;
  - моделі та схеми — повністю (основа розуміння проекту);
  - великий файл → кілька уроків поспіль («db.py: двигун», «db.py: сесії»).
- Кожен урок у межах ліміту з lesson-format.md (2500–4000 символів).
  Орієнтир курсу: 5–25 уроків.

### 3. Знахідки

- Погані або застарілі практики у коді: «тут X, сучасно — Y».
  Чесно, конкретно, з файлами.

### 4. Підтвердження користувача

- Покажи план + знахідки в чаті. ДОЧЕКАЙСЯ рішення: «вчимо як є» чи
  «спершу виправити» (виправлення — окрема робота поза цією навичкою;
  після нього починай аналіз заново).
- Після підтвердження: запиши `.practicum/course.md` (формат нижче)
  і одразу згенеруй урок 01 (див. «Генерація уроку»).

## Якщо course.md Є — наступний урок

1. Розпарси чекбокси секції «Уроки». Нема секції чи жодного чекбокса →
   повідом, що файл зіпсований, запропонуй видалити його і зупинись.
2. Перший `- [ ]` — наступний урок. Усі `[x]` → «курс завершено 🎉»;
   підкажи: новий курс = видалити `.practicum/course.md`.
3. Файл редагований руками — вір йому: порядок і склад уроків
   визначає користувач.

## Генерація уроку

1. Прочитай АКТУАЛЬНИЙ код файлів уроку з диска (проект міг змінитись
   після плану). Файл зник → відміть урок `[x]` з приміткою
   `(файл зник)` і перейди до наступного `- [ ]`.
2. Згенеруй урок за `../practicum/lesson-format.md` (прочитай його):
   пояснення, покрокові нотатки, lesson-JSON, перевірка офсетів,
   збирання HTML.
3. Ім'я файла: `course-NN-<slug>.html` (NN — номер уроку).
4. Якщо у «Знахідках» вирішено «вчимо як є» — нотатки уроку чесно
   згадують кращу практику там, де код її порушує.
5. Відміть урок `[x]` у course.md.

## Формат course.md

```markdown
# Курс: <назва проекту> — <одне речення, про що він>

Фокус: <шлях або «весь проект»>. Створено: YYYY-MM-DD.

## Знахідки (рішення: вчимо як є)

- ⚠ settings.py: pydantic v1 BaseSettings — сучасно pydantic-settings v2.

## Уроки

- [x] 01 · .env + settings.py — конфіг через pydantic-settings
- [ ] 02 · db.py — async engine і фабрика сесій
- [ ] 03 · main.py — FastAPI app + lifespan
```

Рядок уроку: `- [ ] NN · <файли/тема> — <що вчимо, один рядок>`.
````

- [ ] **Step 3: Перевірити посилання на ядро**

```bash
grep -q "lesson-format.md" skills/how-to/SKILL.md && ls skills/practicum/lesson-format.md && echo "core link ok"
```

Expected: `core link ok` (і шлях існує).

- [ ] **Step 4: Commit**

```bash
git add commands/how-to.md skills/how-to/SKILL.md
git commit -m "feat: add /how-to command — project course of typing lessons"
```

---

### Task 4: README, видалення симлінків, установка плагіна

**Files:**
- Rewrite: `README.md`
- Delete: симлінки `~/.claude/commands/practicum.md`, `~/.claude/skills/practicum`

Працювати з `/home/jarvis/Programming/vibe-typing`.

- [ ] **Step 1: Переписати `README.md`** — точний зміст:

````markdown
# Vibe Typing

Плагін Claude Code: навчальний typing-тренажер на базі твого власного
коду. Самодостатні HTML-уроки — передрук коду з простими поясненнями
українською та покроковим гайдом по блоках.

Команди:

- `/vibe-typing:practicum` — урок з diff поточного чекпоінта
- `/vibe-typing:how-to` — курс по існуючому проекту: «як написати його
  з нуля», крок за кроком

Спеки: `docs/superpowers/specs/`

## Встановлення

```bash
claude plugin marketplace add ~/Programming/vibe-typing
claude plugin install vibe-typing@vibe-typing-local
```

## Використання

- `/vibe-typing:practicum` — незакомічені зміни або останній коміт;
  `/vibe-typing:practicum HEAD~3..` — діапазон комітів.
- `/vibe-typing:how-to` — перший виклик аналізує проект, показує план
  курсу і знахідки, після підтвердження пише `.practicum/course.md`
  і генерує урок 01; кожен наступний виклик — наступний урок.
  `/vibe-typing:how-to src/api` — курс лише по підсистемі.

Уроки: `.practicum/lessons/*.html` у поточному проекті.

## Розробка

Зібрати тестовий урок з golden-файла:

```bash
python3 - <<'EOF'
from pathlib import Path
tpl = Path('skills/practicum/template.html').read_text()
lesson = Path('sample-lesson.json').read_text()
Path('/tmp/practicum-test.html').write_text(tpl.replace('__LESSON_JSON__', lesson.replace('</', '<\\/')))
EOF
xdg-open /tmp/practicum-test.html
```
````

- [ ] **Step 2: Видалити старі симлінки (тільки якщо це симлінки)**

```bash
[ -L ~/.claude/commands/practicum.md ] && rm ~/.claude/commands/practicum.md && echo "command symlink removed"
[ -L ~/.claude/skills/practicum ] && rm ~/.claude/skills/practicum && echo "skill symlink removed"
```

Expected: обидва `removed`. Якщо щось із цього НЕ симлінк — зупинись
і повідом (BLOCKED), нічого не видаляй.

- [ ] **Step 3: Встановити плагін**

```bash
claude plugin marketplace add /home/jarvis/Programming/vibe-typing
claude plugin install vibe-typing@vibe-typing-local
claude plugin list
```

Expected: `vibe-typing` у списку встановлених/увімкнених. Якщо таких
підкоманд CLI нема або помилка формату маркетплейсу — НЕ вигадуй
обхідних шляхів: зупинись і повідом BLOCKED з точним виводом команд
(контролер вирішить).

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: README for vibe-typing plugin install and commands"
```

---

### Task 5: Наскрізна перевірка

Без нових файлів. Працювати з `/home/jarvis/Programming/vibe-typing`.

- [ ] **Step 1: Golden build + сanity**

```bash
python3 - <<'EOF'
from pathlib import Path
import json
tpl = Path('skills/practicum/template.html').read_text()
lesson = Path('sample-lesson.json').read_text()
json.loads(lesson)
out = tpl.replace('__LESSON_JSON__', lesson.replace('</', '<\\/'))
assert '__LESSON_JSON__' not in out
Path('/tmp/practicum-test.html').write_text(out)
print('golden build ok')
EOF
```

Expected: `golden build ok`.

- [ ] **Step 2: Структура плагіна відповідає докам**

```bash
ls .claude-plugin/plugin.json .claude-plugin/marketplace.json \
   commands/practicum.md commands/how-to.md \
   skills/practicum/SKILL.md skills/practicum/lesson-format.md \
   skills/practicum/template.html skills/how-to/SKILL.md && echo "layout ok"
```

Expected: `layout ok`.

- [ ] **Step 3: Перевірка доступності команд (ручна, користувач/контролер)**

У НОВІЙ сесії Claude Code: автокомпліт `/vibe-typing:` показує обидві
команди; `/vibe-typing:practicum` на проекті з diff генерує урок
(критерій спеки); `/vibe-typing:how-to` на FastAPI-проекті будує план.
Виконавець плану лише повідомляє, що все готово до цієї перевірки.
