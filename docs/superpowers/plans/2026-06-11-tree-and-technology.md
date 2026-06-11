# Дерево проекту + команда /technology — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Живе дерево проекту зліва в уроці (росте по мірі друку) + команда `/vibe-typing:technology` (курс по технології: інвентаризація / є в проекті / реальна інтеграція в гілці).

**Architecture:** Дерево — чисто клієнтська фіча template.html: дані з `fragments[].file` + нове опційне поле `tree` (файли попередніх уроків); генератори (how-to, technology) заповнюють `tree` з course.md. Technology — нова навичка, що реюзає механіку how-to та lesson-format.md.

**Tech Stack:** vanilla HTML/CSS/JS (template), markdown-навички Claude-плагіна, перевірка через playwright MCP.

Спеки: `docs/superpowers/specs/2026-06-11-project-tree-design.md`,
`docs/superpowers/specs/2026-06-11-technology-command-design.md`.

---

### Task 1: Дерево проекту в template.html

**Files:**
- Modify: `skills/practicum/template.html`

- [ ] **Step 1.1: CSS панелі дерева** — додати перед блоком `#explanation`:

```css
  /* Дерево проекту: росте по мірі друку; тільки на широких екранах */
  #tree {
    display: none;
    position: fixed;
    top: 132px;
    left: calc((100vw - 880px) / 2 - 300px);
    width: 272px;
    max-height: calc(100vh - 172px);
    overflow-y: auto;
    font-family: var(--mono);
    font-size: 12.5px;
    line-height: 1.9;
    color: var(--subtext);
  }
  @media (min-width: 1480px) { #tree { display: block; } }

  #tree .tree-root {
    font-size: 11px;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--accent);
    opacity: 0.85;
    margin-bottom: 8px;
  }

  #tree .node { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  #tree .node.dir { color: var(--dim); }
  #tree .node.file::before { content: "· "; color: var(--dim); }
  #tree .node.file.done { color: var(--subtext); }
  #tree .node.file.done::before { content: "✓ "; color: var(--ok); }
  #tree .node.file.current { color: var(--text); }
  #tree .node.file.current::before { content: "› "; color: var(--accent); }

  #tree .node.appear { animation: node-appear 0.45s ease-out; }
  @keyframes node-appear {
    from { opacity: 0; transform: translateX(-8px); }
    to   { opacity: 1; transform: none; }
  }

  #tree-action {
    margin-top: 10px;
    color: var(--ok);
    opacity: 0;
  }
  #tree-action.show { animation: action-fade 2.6s ease forwards; }
  @keyframes action-fade {
    0%   { opacity: 0; }
    12%  { opacity: 0.9; }
    70%  { opacity: 0.9; }
    100% { opacity: 0; }
  }
```

- [ ] **Step 1.2: HTML панелі** — одразу після `<div id="lesson-progress"></div>`:

```html
<nav id="tree">
  <div class="tree-root" id="tree-root"></div>
  <div id="tree-nodes"></div>
  <div id="tree-action"></div>
</nav>
```

- [ ] **Step 1.3: JS дерева** — додати після функції `saveProgress`. `state` отримує нове поле `maxFrag: 0` в ініціалізаторі.

```js
// --- Дерево проекту: файли попередніх уроків (LESSON.tree) + файли
// відвіданих фрагментів; раз з'явившись, вузол не зникає ---
const treeShown = new Set(); // ключі вузлів, що вже з'являлись (анімація)

function fragPath(f) { return f.file.split(' (')[0].trim(); }

function treeFiles() {
  const files = [];
  const push = (path, cls) => {
    const ex = files.find(x => x.path === path);
    if (ex) { if (cls === 'current') ex.cls = 'current'; }
    else files.push({ path, cls });
  };
  (LESSON.tree || []).forEach(p => push(p, 'done'));
  const last = Math.min(state.maxFrag, LESSON.fragments.length - 1);
  for (let i = 0; i <= last; i++) {
    const cls = i === state.frag && $('results').hidden ? 'current' : 'done';
    push(fragPath(LESSON.fragments[i]), cls);
  }
  return files;
}

function renderTree() {
  $('tree-root').textContent = LESSON.project + '/';
  const rows = [];
  const dirs = new Set();
  for (const f of treeFiles()) {
    const parts = f.path.split('/');
    for (let d = 1; d < parts.length; d++) {
      const dir = parts.slice(0, d).join('/');
      if (!dirs.has(dir)) {
        dirs.add(dir);
        rows.push({ key: dir, label: parts[d - 1] + '/', cls: 'dir', depth: d - 1 });
      }
    }
    rows.push({ key: f.path, label: parts[parts.length - 1],
                cls: 'file ' + f.cls, depth: parts.length - 1 });
  }
  const box = $('tree-nodes');
  box.innerHTML = '';
  const newDirs = [];
  for (const r of rows) {
    const el = document.createElement('div');
    el.className = 'node ' + r.cls;
    el.style.paddingLeft = (r.depth * 14) + 'px';
    el.textContent = r.label;
    if (!treeShown.has(r.key)) {
      el.classList.add('appear');
      treeShown.add(r.key);
      if (r.cls === 'dir') newDirs.push(r.key);
    }
    box.appendChild(el);
  }
  if (newDirs.length) treeAction('+ mkdir ' + newDirs[newDirs.length - 1]);
}

function treeAction(text) {
  const el = $('tree-action');
  el.textContent = text;
  el.classList.remove('show');
  void el.offsetWidth;
  el.classList.add('show');
}
```

- [ ] **Step 1.4: інтеграція в життєвий цикл**
  - `state`: `frag: 0, pos: 0` → додати `maxFrag: 0`.
  - `renderFragment()`: на початку, після `const f = ...`: `state.maxFrag = Math.max(state.maxFrag, state.frag);`; в кінці (після `updateCursor()`): `renderTree();`
  - `showResults()`: після `$('results').hidden = false;` → `renderTree();` (current → done).
  - `saveProgress()`: додати `maxFrag: state.maxFrag` у JSON.
  - `init()`: при відновленні — `state.maxFrag = saved.maxFrag ?? saved.frag;` перед `renderFragment()`.
  - `restart-btn`: скинути `state.maxFrag = 0;` поряд з `state.frag = 0`.

- [ ] **Step 1.5: зібрати тестовий урок і перевірити playwright**

Зібрати з `sample-lesson.json` (додавши йому `"tree": ["existing/prev.py"]` у тимчасовій копії) за рецептом з README. Відкрити через локальний http-сервер, viewport 1920×1080:
  - `#tree` видимий, корінь = project, вузли tree-файлів done (✓), файл фрагмента 0 — current (›).
  - Перейти «вперед →» — файл наступного фрагмента з'являється з анімацією; нова папка → `#tree-action` показує `+ mkdir ...`.
  - «← назад» — вузли НЕ зникають, current повертається.
  - Завершення уроку → всі вузли done.
  - Viewport 1366×768 → `display: none` у `#tree`, боді без h-скролу.

- [ ] **Step 1.6: Commit**

```bash
git add skills/practicum/template.html
git commit -m "feat: живе дерево проекту зліва в уроці"
```

### Task 2: Поле tree у форматі уроку + генерація в how-to

**Files:**
- Modify: `skills/practicum/lesson-format.md`
- Modify: `skills/how-to/SKILL.md`

- [ ] **Step 2.1: lesson-format.md** — нова секція після «Що не друкується (skips)»:

```markdown
## Дерево проекту (tree)

- Опційне поле верхнього рівня `"tree": ["шлях/до/файла", ...]` —
  файли, збудовані ПОПЕРЕДНІМИ уроками курсу, в порядку появи.
  Двигун показує їх у дереві зліва одразу (✓), а файли поточного
  уроку додає по мірі друку.
- Шлях у `fragments[].file` ДО першої ` (` має бути точним шляхом
  файла від кореня проекту — з нього двигун будує дерево.
- Разовий урок (practicum) — поле не задавай.
```

  У JSON-приклад додати `"tree": ["шлях/до/файла.py"],` перед `"fragments"`. У валідатор (перед циклом фрагментів):

```python
tree = lesson.get('tree', [])
assert len(tree) == len(set(tree)), 'tree: дублікати шляхів'
assert all(isinstance(p, str) and p.strip() and ' (' not in p
           for p in tree), 'tree: некоректний шлях'
```

- [ ] **Step 2.2: how-to SKILL.md**
  - У «Формат course.md» додати після опису рядка уроку: «Рядок уроку МУСИТЬ містити точні шляхи файлів уроку (з них будується дерево проекту в наступних уроках).»
  - У «Генерація уроку» новий пункт (після кроку 2 про інкремент): «`tree` уроку = точні шляхи файлів усіх уроків ВИЩЕ поточного в course.md, без дублів, у порядку уроків. Для уроку 01 — відсутній.»

- [ ] **Step 2.3: Commit**

```bash
git add skills/practicum/lesson-format.md skills/how-to/SKILL.md
git commit -m "feat: поле tree у lesson-JSON; how-to генерує дерево з course.md"
```

### Task 3: Перезбірка існуючих уроків ai-post-generated-bot

**Files:**
- Modify: `/home/jarvis/Programming/ai-post-generated-bot/.practicum/lessons/*.html`

- [ ] **Step 3.1: вшити tree і перезібрати** (скрипт по патерну попередніх перезбірок: витягти `const LESSON = (...);`, модифікувати, вставити в новий template):
  - `2026-06-10-ssrf-safe-fetch.html` — без tree (разовий practicum).
  - `course-01-env-settings.html` — `tree: []` не додавати (перший урок).
  - `course-02-db-logging-base.html` — `"tree": [".env.example", "app/core/config.py"]`.

- [ ] **Step 3.2: playwright-перевірка course-02 на 1920×1080**: дерево показує `.env.example` (✓), `app/core/` → `config.py` (✓), і `db.py` як current; перейти вперед — `logging.py` з'являється; далі — `app/models/` + `base.py` з `+ mkdir app/models` у `#tree-action`. Звичайний друк не зламаний (надрукувати перший символ фрагмента — без помилки). Прибрати тестові сліди localStorage.

- [ ] **Step 3.3: Commit** — у repo плагіна комітити нічого (зміни в чужому проекті), тільки переконатися `git status` чистий від артефактів.

### Task 4: Команда /technology

**Files:**
- Create: `commands/technology.md`
- Create: `skills/technology/SKILL.md`
- Modify: `README.md`

- [ ] **Step 4.1: commands/technology.md**

```markdown
---
description: Курс по технології в проекті — як вона інтегрована; нема в проекті — реальна інтеграція в гілці (за явною згодою)
argument-hint: "[технологія, напр. redis]"
---

Використай навичку technology (Skill tool): курс typing-уроків по
технології в цьому проекті. Аргумент від користувача: $ARGUMENTS
```

- [ ] **Step 4.2: skills/technology/SKILL.md** — повний текст:

```markdown
---
name: technology
description: Курс typing-уроків по одній технології в поточному проекті — як вона інтегрована або (за явною згодою) як її інтегрувати. Використовуй для команди /vibe-typing:technology.
---

# Technology — курс по технології в проекті

Аргумент (опціонально): назва технології. Стан курсу:
`.practicum/course-<tech>.md` — окремий файл на кожну технологію,
формат як у how-to; `<tech>` — латиницею в нижньому регістрі.

## Вибір режиму

1. Без аргументу → «Інвентаризація».
2. З аргументом визнач, чи технологія реально використовується:
   файли залежностей (pyproject.toml, requirements*.txt, package.json,
   go.mod, ...), імпорти в коді, конфіги, docker-compose.
   Використовується → «Технологія є», ні → «Технології нема».
3. Назва неоднозначна (сервіс чи клієнтська бібліотека, кілька
   збігів) — перепитай користувача, не вгадуй.

## Інвентаризація (без аргументу)

- Просканируй залежності, docker-compose, конфіги, ключові імпорти.
- Покажи список у чаті: технологія + один рядок «що це і де воно
  у нас» + позначка, чи вже є курс `.practicum/course-<tech>.md`.
- Зупинись: файлів не пиши, користувач сам викличе
  `/vibe-typing:technology <обрана>`.

## Технологія є

Дій за `../how-to/SKILL.md` (прочитай його), з відмінностями:

- Стан: `.practicum/course-<tech>.md`.
- Скоуп: ТІЛЬКИ точки дотику технології: залежність → конфіг/env →
  ініціалізація (клієнт/двигун/підключення) → перше використання →
  решта патернів (кеш, локи, черги, ... — кожен своїм уроком/етапом).
- План еволюційний: «як би ми додавали цю технологію з нуля».
- Орієнтир 3–10 уроків. Використання тривіальне (1–2 рядки) → чесно
  попередь, що курс буде закоротким, і спитай, чи продовжувати.
- Ім'я уроків: `tech-<tech>-NN-<slug>.html`.

## Технології нема — інтеграційний курс

Вчитись можна лише на коді, що точно працює: інтеграція реальна,
в окремій гілці, з перевіркою.

1. Побудуй план інтеграції: мінімальне робоче ядро технології в ЦЬОМУ
   проекті — залежність → конфіг → ініціалізація → 1–2 реальні точки
   використання. НЕ «обвішуй усе». Список файлів, спосіб перевірки.
2. ЯВНО ЗАСТЕРЕЖИ і ДОЧЕКАЙСЯ згоди. Користувач має чітко розуміти,
   що станеться: це НЕ просто генерація уроку, а повноцінна
   реалізація — буде створено гілку і написано реальний код; це може
   бути тривалим і витратити багато токенів. Без явного «так» — стоп.
3. Робоче дерево брудне → зупинись: попроси закомітити або стешнути.
4. Гілка `vibe-typing/<tech>` від HEAD. Реалізуй: код → самостійне
   рев'ю → тести проекту → smoke-перевірка. У звіті чесно: що
   перевірено, що ні.
5. Закоміть у гілці й повернись на вихідну (`git checkout -`).
   Далі робоче дерево користувача не чіпай.
6. Запиши `.practicum/course-<tech>.md` з позначкою «інтеграційний
   курс, гілка vibe-typing/<tech>, база <вихідна гілка>». Уроки
   генеруй з diff `<вихідна>..vibe-typing/<tech>`; актуальний код
   фрагментів бери З ГІЛКИ (`git show vibe-typing/<tech>:шлях`),
   еволюційними етапами.
7. Після останнього уроку запропонуй вибір: змерджити гілку (код
   готовий) або видалити її і внести зміни руками за уроками
   (максимум навчання).

## Спільне

- Урок — за `../practicum/lesson-format.md` (прочитай його), включно
  з полем `tree` і правилом точних шляхів у рядках course-<tech>.md.
- Відмічай уроки `[x]` у course-<tech>.md, як у how-to.
```

- [ ] **Step 4.3: README** — у список команд додати:

```markdown
- `/vibe-typing:technology [назва]` — курс по технології: без
  аргументу — інвентаризація стеку; технологія є в проекті — курс
  «як вона інтегрована»; нема — реальна інтеграція в окремій гілці
  (за явною згодою) і курс з її diff
```

- [ ] **Step 4.4: смоук інвентаризації** — на ai-post-generated-bot виконати кроки інвентаризації вручну (прочитати pyproject.toml/docker-compose) і переконатися, що інструкція навички дає осмислений список (fastapi, sqlalchemy, redis, celery?, structlog, openai, ...). Це перевірка тексту навички, не повний прогін.

- [ ] **Step 4.5: Commit**

```bash
git add commands/technology.md skills/technology/SKILL.md README.md
git commit -m "feat: команда /technology — курс по технології (інвентаризація / є / інтеграція в гілці)"
```

### Task 5: Версія і кеш

**Files:**
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 5.1:** `version` → `0.2.8`.
- [ ] **Step 5.2:** `claude plugin update vibe-typing@vibe-typing-local` → очікувано «updated from 0.2.7 to 0.2.8».
- [ ] **Step 5.3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: bump 0.2.8"
```

## Верифікація після всього

- Уроки в браузері: дерево живе, друк/скіпи/навігація/resume не зламані.
- `/vibe-typing:technology redis` на ai-post-generated-bot — повний
  прогін користувачем (UAT) у новій сесії після рестарту Claude Code.
