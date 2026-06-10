# Practicum MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Слеш-команда `/practicum` у Claude Code, що з diff чекпоінта генерує самодостатній HTML typing-тренажер з поясненнями українською та active-recall дірками.

**Architecture:** Три сорс-файли в репо `~/Programming/practicum` (command, SKILL.md, template.html), симлінки в `~/.claude/`. Агент генерує lesson-JSON і вшиває його в копію template.html; UI — vanilla JS без залежностей. Спека: `docs/superpowers/specs/2026-06-10-practicum-typing-design.md`.

**Tech Stack:** Claude Code slash command + skill, vanilla HTML/CSS/JS, python3 лише для перевірок/збирання тестового HTML.

**Тестова стратегія:** За спекою автотестів на MVP немає (свідоме рішення). Замість них кожна задача має крок перевірки: python-перевірка офсетів (фактично «тест» формату уроку) і ручні браузерні перевірки на golden-уроці з конкретними очікуваннями. Не пропускай кроки перевірки.

---

### Task 1: Golden-урок `sample-lesson.json` з перевіркою офсетів

**Files:**
- Create: `sample-lesson.json`

- [ ] **Step 1: Створити `sample-lesson.json`**

Зміст файлу (повністю):

```json
{
  "title": "Кешування статистики користувача",
  "project": "sample",
  "date": "2026-06-10",
  "fragments": [
    {
      "file": "app/stats.py",
      "explanation": "Ця функція рахує статистику користувача, але обчислення дороге, тому результат кешується. Спершу дивимось у кеш: якщо значення там є — повертаємо одразу. Якщо нема — рахуємо, кладемо в кеш на 300 секунд (ttl) і повертаємо.",
      "code": "def get_stats(user_id):\n    cached = cache.get(f\"stats:{user_id}\")\n    if cached is not None:\n        return cached\n    stats = compute_stats(user_id)\n    cache.set(f\"stats:{user_id}\", stats, ttl=300)\n    return stats",
      "blanks": [
        {
          "start": 74,
          "end": 92,
          "answer": "cached is not None",
          "hint": "що повертає cache.get, коли ключа нема? І чому перевірка просто if cached: була б помилкою для значення 0?"
        },
        {
          "start": 196,
          "end": 199,
          "answer": "300",
          "hint": "на скільки секунд кешуємо? Дивись пояснення зверху."
        }
      ]
    },
    {
      "file": "app/stats.py",
      "explanation": "Коли дані користувача змінюються, стара статистика в кеші стає брехливою. Тому є друга функція: вона видаляє запис з кешу за тим самим ключем, що й get_stats. Наступний виклик get_stats порахує все заново.",
      "code": "def invalidate_stats(user_id):\n    cache.delete(f\"stats:{user_id}\")",
      "blanks": [
        {
          "start": 48,
          "end": 66,
          "answer": "f\"stats:{user_id}\"",
          "hint": "ключ має бути ТОЙ САМИЙ, що використовує get_stats — інакше видалимо не те."
        }
      ]
    }
  ]
}
```

- [ ] **Step 2: Запустити перевірку офсетів (це наш «тест» формату)**

Run (з кореня репо):

```bash
python3 - <<'EOF'
import json
lesson = json.load(open('sample-lesson.json'))
for f in lesson['fragments']:
    for b in f['blanks']:
        got = f['code'][b['start']:b['end']]
        assert got == b['answer'], f"{f['file']}: '{got}' != '{b['answer']}'"
        assert '\n' not in b['answer'], 'дірка не може перетинати рядок'
print('offsets ok')
EOF
```

Expected: `offsets ok`. Якщо assert падає — виправ `start`/`end` у JSON (не `answer`!) і повтори до проходження.

- [ ] **Step 3: Commit**

```bash
git add sample-lesson.json
git commit -m "feat: add golden sample lesson with verified blank offsets"
```

---

### Task 2: `template.html` — каркас і рендеринг уроку (без друку)

**Files:**
- Create: `skills/practicum/template.html`

- [ ] **Step 1: Створити `skills/practicum/template.html`**

Зміст файлу (повністю). Маркери `// === TYPING ===`, `// === BLANKS ===`, `// === RESULTS ===` — місця, які заповнять Task 3–5; у цій задачі вони лишаються коментарями:

```html
<!DOCTYPE html>
<html lang="uk">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Practicum</title>
<style>
  :root {
    --bg: #1e1e2e; --panel: #27273a; --text: #cdd6f4; --dim: #6c7086;
    --green: #a6e3a1; --red: #f38ba8; --yellow: #f9e2af; --accent: #89b4fa;
  }
  * { box-sizing: border-box; }
  body { margin: 0; background: var(--bg); color: var(--text);
         font-family: system-ui, sans-serif; min-height: 100vh; }
  header { display: flex; justify-content: space-between; align-items: baseline;
           padding: 16px 24px; color: var(--dim); }
  header h1 { font-size: 18px; color: var(--text); margin: 0; }
  main { max-width: 900px; margin: 0 auto; padding: 0 24px 48px; }
  #explanation { background: var(--panel); border-left: 3px solid var(--accent);
                 padding: 14px 18px; border-radius: 6px; line-height: 1.55; }
  #file { color: var(--dim); font-size: 13px; margin: 18px 0 6px;
          font-family: monospace; }
  #code { background: var(--panel); padding: 18px; border-radius: 6px;
          font-family: "JetBrains Mono", "Fira Code", monospace; font-size: 16px;
          line-height: 1.7; overflow-x: auto; white-space: pre; }
  .char { color: var(--dim); }
  .char.correct { color: var(--green); }
  .char.auto { color: var(--dim); }
  .char.cursor { background: var(--accent); color: var(--bg); border-radius: 2px; }
  .char.blank-pending { border-bottom: 2px dashed var(--yellow); color: transparent; }
  .char.blank-done { color: var(--yellow); }
  body.flash #code { outline: 2px solid var(--red); }
  #blank-box { margin-top: 14px; background: var(--panel); padding: 14px 18px;
               border-radius: 6px; }
  #blank-box input { width: 100%; font-family: monospace; font-size: 16px;
                     background: var(--bg); color: var(--text);
                     border: 1px solid var(--dim); border-radius: 4px;
                     padding: 8px 10px; }
  #blank-box .controls { margin-top: 8px; display: flex; gap: 12px;
                         align-items: center; }
  button { background: none; border: 1px solid var(--dim); color: var(--dim);
           border-radius: 4px; padding: 4px 10px; cursor: pointer; font: inherit; }
  button:hover { color: var(--text); border-color: var(--text); }
  #hint-text { color: var(--yellow); margin-top: 8px; display: none; }
  #blank-status { color: var(--red); }
  #results { max-width: 900px; margin: 48px auto; padding: 0 24px; }
  #results table { border-collapse: collapse; margin-top: 16px; }
  #results td { padding: 6px 16px 6px 0; text-align: left; }
  .big { font-size: 32px; color: var(--green); }
  [hidden] { display: none !important; }
</style>
</head>
<body>
<header>
  <h1 id="title"></h1>
  <div id="progress"></div>
</header>
<main id="main">
  <div id="explanation"></div>
  <div id="file"></div>
  <pre id="code"></pre>
  <div id="blank-box" hidden>
    <div>Заповни пропуск і натисни Enter:</div>
    <input id="blank-input" autocomplete="off" spellcheck="false">
    <div class="controls">
      <button id="hint-btn">підказка</button>
      <button id="reveal-btn" hidden>показати відповідь</button>
      <span id="blank-status"></span>
    </div>
    <div id="hint-text"></div>
  </div>
</main>
<section id="results" hidden>
  <h2>Урок завершено 🎉</h2>
  <div class="big" id="r-accuracy"></div>
  <table>
    <tr><td>Час</td><td id="r-time"></td></tr>
    <tr><td>Помилки друку</td><td id="r-errors"></td></tr>
    <tr><td>Дірки з першої спроби</td><td id="r-blank-first"></td></tr>
    <tr><td>Дірки з підказкою/не з першої</td><td id="r-blank-hint"></td></tr>
    <tr><td>Відкриті відповіді</td><td id="r-blank-revealed"></td></tr>
  </table>
</section>
<script>
const LESSON = __LESSON_JSON__;

const $ = id => document.getElementById(id);
const state = {
  frag: 0, pos: 0,
  keystrokes: 0, errors: 0,
  startedAt: null,
  blanks: { first: 0, hint: 0, revealed: 0 },
};
let chars = [];   // span-елементи символів активного фрагмента
let blank = null; // активна дірка {start,end,answer,hint,attempts,usedHint,revealed}

function renderFragment() {
  const f = LESSON.fragments[state.frag];
  $('title').textContent = LESSON.title;
  $('progress').textContent = (state.frag + 1) + ' / ' + LESSON.fragments.length;
  $('explanation').textContent = f.explanation;
  $('file').textContent = f.file;
  const codeEl = $('code');
  codeEl.innerHTML = '';
  chars = [];
  const inBlank = i => f.blanks.some(b => i >= b.start && i < b.end);
  [...f.code].forEach((ch, i) => {
    const span = document.createElement('span');
    span.className = 'char';
    if (inBlank(i)) span.classList.add('blank-pending');
    span.textContent = ch === '\n' ? '⏎\n' : ch;
    codeEl.appendChild(span);
    chars.push(span);
  });
  state.pos = 0;
  blank = null;
  advanceAuto();
  updateCursor();
}

// Автоввід відступів: пробіли на початку рядка вводити не треба
function advanceAuto() {
  const code = LESSON.fragments[state.frag].code;
  while (state.pos < code.length) {
    const atLineStart = state.pos === 0 || code[state.pos - 1] === '\n';
    const prevAuto = state.pos > 0 && chars[state.pos - 1].classList.contains('auto');
    if (code[state.pos] === ' ' && (atLineStart || prevAuto)) {
      chars[state.pos].classList.add('auto');
      state.pos++;
    } else break;
  }
  maybeOpenBlank();
}

function updateCursor() {
  chars.forEach(c => c.classList.remove('cursor'));
  if (state.pos < chars.length) {
    chars[state.pos].classList.add('cursor');
    chars[state.pos].scrollIntoView({ block: 'nearest' });
  }
}

// === TYPING ===  (Task 3)
function maybeOpenBlank() {} // заглушка, реалізація в Task 4
// === BLANKS ===  (Task 4)
// === RESULTS === (Task 5)

renderFragment();
</script>
</body>
</html>
```

- [ ] **Step 2: Зібрати тестовий HTML і перевірити рендеринг**

Run (з кореня репо):

```bash
python3 - <<'EOF'
from pathlib import Path
import json
tpl = Path('skills/practicum/template.html').read_text()
lesson = Path('sample-lesson.json').read_text()
json.loads(lesson)  # валідність JSON
Path('/tmp/practicum-test.html').write_text(tpl.replace('__LESSON_JSON__', lesson))
print('built /tmp/practicum-test.html')
EOF
xdg-open /tmp/practicum-test.html
```

Expected у браузері: заголовок «Кешування статистики користувача», прогрес «1 / 2», пояснення в рамці, код фрагмента 1 моноширинним шрифтом; місця дірок — невидимий текст з жовтим пунктирним підкресленням; курсор (синій фон) стоїть на `d` першого рядка (відступів у першому рядку нема). Дірок-інпута і результатів не видно. Помилок у консолі браузера нема.

- [ ] **Step 3: Commit**

```bash
git add skills/practicum/template.html
git commit -m "feat: add template.html skeleton with lesson rendering"
```

---

### Task 3: Двигун друку

**Files:**
- Modify: `skills/practicum/template.html` (секція `// === TYPING ===`)

- [ ] **Step 1: Замінити рядок `// === TYPING ===  (Task 3)` на код двигуна**

```js
// === TYPING ===
document.addEventListener('keydown', e => {
  if (!$('results').hidden) return;
  if (blank) return; // поки відкрита дірка, друк заблоковано
  if (e.ctrlKey || e.metaKey || e.altKey) return;
  const code = LESSON.fragments[state.frag].code;
  if (state.pos >= code.length) return;
  let key = e.key;
  if (key === 'Enter') key = '\n';
  if (key.length !== 1 && key !== '\n') return; // Shift, F5, стрілки — ігноруємо
  e.preventDefault();
  state.startedAt ??= Date.now();
  state.keystrokes++;
  if (key === code[state.pos]) {
    chars[state.pos].classList.add('correct');
    state.pos++;
    advanceAuto();
    updateCursor();
    checkFragmentDone();
  } else {
    state.errors++;
    document.body.classList.add('flash');
    setTimeout(() => document.body.classList.remove('flash'), 150);
  }
});

function checkFragmentDone() {
  const code = LESSON.fragments[state.frag].code;
  if (state.pos < code.length) return;
  if (state.frag + 1 < LESSON.fragments.length) {
    state.frag++;
    renderFragment();
  } else {
    showResults();
  }
}
function showResults() {} // заглушка, реалізація в Task 5
```

Увага: заглушку `function showResults() {}` Task 5 замінить разом з маркером `// === RESULTS ===`.

- [ ] **Step 2: Перезібрати тестовий HTML і перевірити друк**

Run: та сама python-команда збирання з Task 2 Step 2, потім `xdg-open /tmp/practicum-test.html`.

Expected: друкуєш `def get_stats(user_id):` — символи зеленіють, курсор їде; натискаєш Enter — перенос; відступ наступного рядка проскакується сам (курсор одразу на `c` у `cached`). Неправильний символ — червоний спалах рамки коду, курсор НЕ рухається. Дійшовши до дірки (після `if `) — друк далі неможливий (дірка ще не реалізована — це очікувано, перевіримо в Task 4).

- [ ] **Step 3: Commit**

```bash
git add skills/practicum/template.html
git commit -m "feat: add char-by-char typing engine with auto-indent"
```

---

### Task 4: Дірки (active recall)

**Files:**
- Modify: `skills/practicum/template.html` (заглушка `maybeOpenBlank` і секція `// === BLANKS ===`)

- [ ] **Step 1: Видалити рядок `function maybeOpenBlank() {} // заглушка, реалізація в Task 4` і замінити маркер `// === BLANKS ===  (Task 4)` на:**

```js
// === BLANKS ===
function maybeOpenBlank() {
  const f = LESSON.fragments[state.frag];
  const b = f.blanks.find(b => b.start === state.pos);
  if (b) openBlank(b);
}

function openBlank(b) {
  blank = Object.assign({ attempts: 0, usedHint: false, revealed: false }, b);
  $('blank-box').hidden = false;
  $('hint-text').style.display = 'none';
  $('hint-text').textContent = b.hint || '';
  $('hint-btn').hidden = !b.hint;
  $('reveal-btn').hidden = true;
  $('blank-status').textContent = '';
  $('blank-input').value = '';
  $('blank-input').focus();
}

const norm = s => s.replace(/\s+/g, ' ').trim();

function submitBlank() {
  state.startedAt ??= Date.now();
  if (norm($('blank-input').value) === norm(blank.answer)) {
    for (let i = blank.start; i < blank.end; i++) {
      chars[i].classList.remove('blank-pending');
      chars[i].classList.add('blank-done');
    }
    let bucket = 'first';
    if (blank.revealed) bucket = 'revealed';
    else if (blank.usedHint || blank.attempts > 0) bucket = 'hint';
    state.blanks[bucket]++;
    state.pos = blank.end;
    blank = null;
    $('blank-box').hidden = true;
    advanceAuto();
    updateCursor();
    checkFragmentDone();
  } else {
    blank.attempts++;
    $('blank-status').textContent = 'не те, спробуй ще';
    if (blank.attempts >= 2) $('reveal-btn').hidden = false;
  }
}

$('hint-btn').onclick = () => {
  blank.usedHint = true;
  $('hint-text').style.display = 'block';
  $('blank-input').focus();
};
$('reveal-btn').onclick = () => {
  blank.revealed = true;
  $('blank-input').value = blank.answer;
  $('blank-input').focus();
};
$('blank-input').addEventListener('keydown', e => {
  if (e.key === 'Enter') submitBlank();
  e.stopPropagation(); // щоб введення в інпут не йшло у двигун друку
});
```

- [ ] **Step 2: Перезібрати тестовий HTML і перевірити повний цикл дірки**

Run: команда збирання з Task 2 Step 2, `xdg-open /tmp/practicum-test.html`.

Expected: додрукувавши до `if ` — з'являється блок дірки, фокус в інпуті. Перевірити всі гілки: (а) ввести `cached is not None` + Enter → дірка жовтіє в коді, блок ховається, курсор після неї, друк продовжується; (б) на другій дірці ввести двічі дурницю → статус «не те, спробуй ще», після 2-ї спроби з'являється «показати відповідь», клік підставляє відповідь, Enter приймає; (в) кнопка «підказка» показує жовтий текст підказки. Друк з клавіатури поза інпутом при відкритій дірці нічого не робить.

- [ ] **Step 3: Commit**

```bash
git add skills/practicum/template.html
git commit -m "feat: add active-recall blanks with hints and reveal"
```

---

### Task 5: Екран результатів + localStorage

**Files:**
- Modify: `skills/practicum/template.html` (заглушка `showResults` і маркер `// === RESULTS ===`)

- [ ] **Step 1: Видалити рядок `function showResults() {} // заглушка, реалізація в Task 5` і замінити маркер `// === RESULTS === (Task 5)` на:**

```js
// === RESULTS ===
function showResults() {
  $('main').hidden = true;
  const secs = Math.round((Date.now() - state.startedAt) / 1000);
  const acc = state.keystrokes
    ? Math.round(100 * (state.keystrokes - state.errors) / state.keystrokes)
    : 100;
  $('r-accuracy').textContent = acc + '% точність';
  $('r-time').textContent = Math.floor(secs / 60) + ' хв ' + (secs % 60) + ' с';
  $('r-errors').textContent = state.errors;
  $('r-blank-first').textContent = state.blanks.first;
  $('r-blank-hint').textContent = state.blanks.hint;
  $('r-blank-revealed').textContent = state.blanks.revealed;
  $('results').hidden = false;
  const history = JSON.parse(localStorage.getItem('practicum-history') || '[]');
  history.push({
    title: LESSON.title, project: LESSON.project, date: LESSON.date,
    accuracy: acc, seconds: secs, errors: state.errors,
    blanks: state.blanks, finishedAt: new Date().toISOString(),
  });
  localStorage.setItem('practicum-history', JSON.stringify(history));
}
```

- [ ] **Step 2: Перезібрати і пройти golden-урок до кінця**

Run: команда збирання з Task 2 Step 2, `xdg-open /tmp/practicum-test.html`.

Expected: після останнього символу фрагмента 2 — екран «Урок завершено 🎉» з відсотком точності, часом, помилками і розбивкою дірок (first/hint/revealed відповідає тому, як ти їх проходив). У DevTools: `JSON.parse(localStorage.getItem('practicum-history'))` містить запис із щойно пройденим уроком.

- [ ] **Step 3: Commit**

```bash
git add skills/practicum/template.html
git commit -m "feat: add results screen with metrics and localStorage history"
```

---

### Task 6: Слеш-команда і SKILL.md (генерація уроку агентом)

**Files:**
- Create: `commands/practicum.md`
- Create: `skills/practicum/SKILL.md`

- [ ] **Step 1: Створити `commands/practicum.md`**

```markdown
---
description: Згенерувати typing-практикум з diff поточного чекпоінта
argument-hint: "[діапазон комітів, напр. HEAD~3..]"
---

Використай навичку practicum (Skill tool), щоб згенерувати урок-практикум
з diff. Аргумент діапазону від користувача: $ARGUMENTS
```

- [ ] **Step 2: Створити `skills/practicum/SKILL.md`**

````markdown
---
name: practicum
description: Згенерувати з diff чекпоінта самодостатній HTML typing-практикум — передрук коду з простими поясненнями українською та active-recall дірками. Використовуй для команди /practicum.
---

# Practicum — генерація уроку

Результат: один HTML-файл `.practicum/lessons/YYYY-MM-DD-<slug>.html`
у поточному проєкті, відкритий у браузері. Жодних інших артефактів.

## Крок 1: зібрати diff

- З аргументом (діапазон): `git diff <діапазон>`.
- Без аргументу: `git diff HEAD` (незакомічені зміни); якщо порожньо —
  `git show HEAD --patch` (останній коміт).
- Не git-репо або diff порожній → повідом користувачу й зупинись.
  Урок НЕ генеруй.

## Крок 2: відібрати фрагменти

- 3–7 фрагментів, сумарно 2500–4000 символів коду (~10–20 хв друку).
- Пріоритет: нова бізнес-логіка > нові патерни/ідіоми > решта.
- Пропускай: імпорти, конфіги, boilerplate, згенерований код, механічні
  повтори, чисті перейменування.
- Фрагмент — цілісний шматок (функція/метод/клас/зв'язний блок), що
  читається без решти файлу. Бери ФІНАЛЬНИЙ стан коду з файлу, а не
  рядки diff з +/-. Якщо фрагмент довший за потрібне — скороти до ядра.
- Таби заміни на 4 пробіли. Трейлінг-пробілів бути не повинно.

## Крок 3: пояснення

- Українською, 2–4 речення на фрагмент, рівень початківця.
- Відповідай на два питання: «що це» і «навіщо саме тут у проєкті».
- Терміни без розшифровки не вживай: назвав термін — одразу поясни його
  простими словами.

## Крок 4: дірки

- 1–3 на фрагмент, на ключових місцях: умова, виклик, тип, значуща
  константа. НЕ на синтаксичному шумі (дужки, двокрапки).
- Відповідь має виводитись із пояснення або логіки фрагмента,
  а не вгадуватись.
- `hint` — навідне запитання, НЕ відповідь.
- Дірка в межах одного рядка, дірки не перетинаються, відсортовані
  за `start`.

## Крок 5: lesson-JSON

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
      "blanks": [
        { "start": 0, "end": 0, "answer": "точний текст з code",
          "hint": "навідне запитання" }
      ]
    }
  ]
}
```

`start`/`end` — офсети в `code`: `code[start:end] == answer`.

## Крок 6: перевірити офсети (обов'язково)

Збережи JSON у тимчасовий файл і виконай:

```bash
python3 - <<'EOF'
import json
lesson = json.load(open('/tmp/practicum-lesson.json'))
for f in lesson['fragments']:
    for b in f['blanks']:
        got = f['code'][b['start']:b['end']]
        assert got == b['answer'], f"{f['file']}: '{got}' != '{b['answer']}'"
        assert '\n' not in b['answer'], 'дірка не може перетинати рядок'
print('offsets ok')
EOF
```

Якщо `offsets ok` нема — виправ офсети й повтори. Без цієї перевірки
HTML не збирай.

## Крок 7: зібрати HTML і відкрити

1. Прочитай `template.html` з директорії цієї навички.
2. Заміни рядок `__LESSON_JSON__` на вміст lesson-JSON.
3. Збережи: `.practicum/lessons/YYYY-MM-DD-<slug>.html`
   (директорію створи; slug — латиницею з дефісами).
4. Якщо `.practicum/` нема в `.gitignore` проєкту — запропонуй
   користувачу додати (не додавай сам).
5. Відкрий: `xdg-open <файл>` і скажи користувачу, що урок готовий.
````

- [ ] **Step 3: Перевірити, що формат уроку в SKILL.md збігається з golden-уроком**

Run:

```bash
python3 -c "
import json
lesson = json.load(open('sample-lesson.json'))
frag = lesson['fragments'][0]
assert set(lesson) == {'title','project','date','fragments'}
assert set(frag) == {'file','explanation','code','blanks'}
assert set(frag['blanks'][0]) == {'start','end','answer','hint'}
print('schema ok')
"
```

Expected: `schema ok`.

- [ ] **Step 4: Commit**

```bash
git add commands/practicum.md skills/practicum/SKILL.md
git commit -m "feat: add /practicum command and lesson-generation skill"
```

---

### Task 7: README, встановлення, наскрізна перевірка

**Files:**
- Create: `README.md`

- [ ] **Step 1: Створити `README.md`**

```markdown
# Practicum

Typing-практикум з diff кодового агента: `/practicum` у Claude Code
генерує самодостатній HTML-тренажер — передрук коду з простими
поясненнями українською та active-recall дірками.

Дизайн: `docs/superpowers/specs/2026-06-10-practicum-typing-design.md`

## Встановлення

```bash
mkdir -p ~/.claude/commands ~/.claude/skills
ln -sfn ~/Programming/practicum/commands/practicum.md ~/.claude/commands/practicum.md
ln -sfn ~/Programming/practicum/skills/practicum ~/.claude/skills/practicum
```

## Використання

У будь-якому проєкті в Claude Code: `/practicum` (незакомічені зміни
або останній коміт) чи `/practicum HEAD~3..` (діапазон). Урок
відкриється в браузері: `.practicum/lessons/<дата>-<тема>.html`.

## Розробка

Зібрати тестовий урок з golden-файла:

```bash
python3 - <<'EOF'
from pathlib import Path
tpl = Path('skills/practicum/template.html').read_text()
lesson = Path('sample-lesson.json').read_text()
Path('/tmp/practicum-test.html').write_text(tpl.replace('__LESSON_JSON__', lesson))
EOF
xdg-open /tmp/practicum-test.html
```
```

- [ ] **Step 2: Встановити симлінки**

Run:

```bash
mkdir -p ~/.claude/commands ~/.claude/skills
ln -sfn ~/Programming/practicum/commands/practicum.md ~/.claude/commands/practicum.md
ln -sfn ~/Programming/practicum/skills/practicum ~/.claude/skills/practicum
ls -la ~/.claude/commands/practicum.md ~/.claude/skills/practicum
```

Expected: обидва симлінки вказують у `~/Programming/practicum/...`.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add README with install and usage"
```

- [ ] **Step 4: Наскрізна перевірка (критерій успіху MVP зі спеки)**

У НОВІЙ сесії Claude Code в будь-якому реальному проєкті з нещодавнім
чекпоінтом виконати `/practicum` і пройти урок до екрану метрик.

Expected: агент зібрав diff → відібрав фрагменти → згенерував і відкрив
HTML → урок проходиться без багів (друк, дірки, підказки, результати).
Це ручний крок користувача; виконавець плану лише готує все до нього
і повідомляє, що можна пробувати.
