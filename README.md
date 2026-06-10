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
Path('/tmp/practicum-test.html').write_text(tpl.replace('__LESSON_JSON__', lesson.replace('</', '<\\/')))
EOF
xdg-open /tmp/practicum-test.html
```
