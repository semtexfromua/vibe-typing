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

Установка копіює плагін у кеш — правки навичок у репо НЕ підхоплюються
живо. Після змін:

```bash
claude plugin update vibe-typing
```
