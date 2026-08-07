# Правильные горячие клавиши Ctrl для Tkinter

«Программное обеспечение предоставляется "как есть" (AS IS), без каких-либо гарантий».

Ниже — универсальный рабочий шаблон для Python/Tkinter. Он решает основные проблемы:

- поддерживает русскую и английскую раскладку;
- обрабатывает `Ctrl+C`, `Ctrl+V`, `Ctrl+X`, `Ctrl+A`, `Ctrl+Z`, `Ctrl+Y`;
- поддерживает команды уровня окна: `Ctrl+N`, `Ctrl+F`;
- поддерживает сохранение диалогов через `Ctrl+S`;
- привязывается один раз и не плодит дубли.

---

## 1. Модуль с горячими клавишами

Можно сохранить, например, как `shortcuts.py`.

```python
import tkinter as tk
import tkinter.ttk as ttk


def _copy(event):
    try:
        if isinstance(event.widget, tk.Text):
            text = event.widget.get("sel.first", "sel.last")
        elif isinstance(event.widget, (tk.Entry, ttk.Entry)):
            if event.widget.selection_present():
                i1 = event.widget.index("sel.first")
                i2 = event.widget.index("sel.last")
                text = event.widget.get()[i1:i2]
            else:
                text = ""
        else:
            text = ""

        if text:
            event.widget.clipboard_clear()
            event.widget.clipboard_append(text)
            event.widget.update()
    except tk.TclError:
        pass

    return "break"


def _cut(event):
    try:
        if isinstance(event.widget, tk.Text):
            text = event.widget.get("sel.first", "sel.last")
            event.widget.delete("sel.first", "sel.last")
        elif isinstance(event.widget, (tk.Entry, ttk.Entry)):
            if event.widget.selection_present():
                i1 = event.widget.index("sel.first")
                i2 = event.widget.index("sel.last")
                text = event.widget.get()[i1:i2]
                event.widget.delete(i1, i2)
            else:
                text = ""
        else:
            text = ""

        if text:
            event.widget.clipboard_clear()
            event.widget.clipboard_append(text)
            event.widget.update()
    except tk.TclError:
        pass

    return "break"


def _paste(event):
    try:
        try:
            clip = event.widget.clipboard_get()
        except tk.TclError:
            clip = ""

        if clip:
            if isinstance(event.widget, tk.Text):
                try:
                    event.widget.delete("sel.first", "sel.last")
                except tk.TclError:
                    pass
                event.widget.insert(tk.INSERT, clip)
            elif isinstance(event.widget, (tk.Entry, ttk.Entry)):
                try:
                    if event.widget.selection_present():
                        i1 = event.widget.index("sel.first")
                        i2 = event.widget.index("sel.last")
                        event.widget.delete(i1, i2)
                except tk.TclError:
                    pass
                event.widget.insert(tk.INSERT, clip)
    except tk.TclError:
        pass

    return "break"


def _select_all_text(event):
    event.widget.tag_add(tk.SEL, "1.0", tk.END)
    event.widget.mark_set(tk.INSERT, "1.0")
    event.widget.see(tk.INSERT)
    return "break"


def _select_all_entry(event):
    event.widget.select_range(0, tk.END)
    event.widget.icursor(tk.END)
    return "break"


def _undo(event):
    try:
        event.widget.edit_undo()
    except tk.TclError:
        pass
    return "break"


def _redo(event):
    try:
        event.widget.edit_redo()
    except tk.TclError:
        pass
    return "break"


_active_app = None


def set_active_app(app):
    global _active_app
    _active_app = app


def handle_control_key(event):
    widget = event.widget.focus_get() or event.widget

    keysym = getattr(event, "keysym", "").lower()
    char = getattr(event, "char", "").lower()
    keycode = getattr(event, "keycode", 0)

    is_c = keysym in ("c", "с") or char in ("c", "с") or keycode == 67
    is_v = keysym in ("v", "м") or char in ("v", "м") or keycode == 86
    is_x = keysym in ("x", "ч") or char in ("x", "ч") or keycode == 88
    is_a = keysym in ("a", "ф") or char in ("a", "ф") or keycode == 65
    is_z = keysym in ("z", "я") or char in ("z", "я") or keycode == 90
    is_y = keysym in ("y", "н") or char in ("y", "н") or keycode == 89
    is_n = keysym in ("n", "т") or char in ("n", "т") or keycode == 78
    is_f = keysym in ("f", "а") or char in ("f", "а") or keycode == 70
    is_s = keysym in ("s", "ы") or char in ("s", "ы") or keycode == 83

    if is_c:
        _copy(event)
        return "break"

    elif is_v:
        _paste(event)
        return "break"

    elif is_x:
        _cut(event)
        return "break"

    elif is_a:
        if isinstance(widget, tk.Text):
            _select_all_text(event)
        elif isinstance(widget, (tk.Entry, ttk.Entry)):
            _select_all_entry(event)
        return "break"

    elif is_z:
        _undo(event)
        return "break"

    elif is_y:
        _redo(event)
        return "break"

    elif is_n:
        if _active_app and hasattr(_active_app, "new_task"):
            _active_app.new_task()
        return "break"

    elif is_f:
        if _active_app and hasattr(_active_app, "search_entry"):
            _active_app.search_entry.focus_set()
        return "break"

    elif is_s:
        try:
            top = widget.winfo_toplevel()
            if hasattr(top, "_dialog_controller") and hasattr(top._dialog_controller, "_save"):
                top._dialog_controller._save()
                return "break"
        except Exception:
            pass


def bind_shortcuts(widget):
    """Глобально привязывает горячие клавиши с Ctrl."""
    try:
        root = widget._root()
        if not getattr(root, "_ctrl_shortcuts_bound", False):
            root.bind_all("<Control-KeyPress>", handle_control_key)
            root._ctrl_shortcuts_bound = True
    except Exception:
        pass
```

---

## 2. Подключение в главном окне

После того как интерфейс уже создан:

```python
from shortcuts import bind_shortcuts, set_active_app

for w in root.winfo_children():
    bind_shortcuts(w)

set_active_app(app)
```

Если есть команды окна, можно дополнительно привязать их напрямую:

```python
root.bind("<Control-n>", lambda e: app.new_task())
root.bind("<Control-N>", lambda e: app.new_task())
root.bind("<Control-f>", lambda e: app.search_entry.focus_set())
root.bind("<Control-F>", lambda e: app.search_entry.focus_set())
```

`set_active_app(app)` нужен, чтобы обработчик знал, куда вызывать `new_task()` и `search_entry.focus_set()`.

Если таких методов нет, ничего страшного: внутри стоят проверки `hasattr`.

---

## 3. Подключение в диалогах

Для модального окна делается так:

```python
top._dialog_controller = self

top.bind("<Escape>", lambda e: top.destroy())
top.bind("<Return>", lambda e: self._save())
top.bind("<Control-s>", lambda e: self._save())
top.bind("<Control-S>", lambda e: self._save())

bind_shortcuts(top)

top.grab_set()
top.focus_force()
```

После этого:

- `Esc` закрывает диалог;
- `Enter` сохраняет;
- `Ctrl+S` сохраняет;
- `Ctrl+C`, `Ctrl+V`, `Ctrl+X`, `Ctrl+A` работают внутри полей ввода.

---

## 4. Какие клавиши обрабатываются

| Действие | EN | RU | keycode |
|---|---:|---:|---:|
| Копировать | `C` | `С` | `67` |
| Вставить | `V` | `М` | `86` |
| Вырезать | `X` | `Ч` | `88` |
| Выделить всё | `A` | `Ф` | `65` |
| Отменить | `Z` | `Я` | `90` |
| Повторить | `Y` | `Н` | `89` |
| Новая задача | `N` | `Т` | `78` |
| Фокус на поиск | `F` | `А` | `70` |
| Сохранить диалог | `S` | `Ы` | `83` |

---

## 5. Почему это работает нормально

Главных причин пять.

### 1. Проверка идёт не только по символу

Используется сразу:

```python
keysym
char
keycode
```

Поэтому хоткеи не ломаются при переключении раскладки.

---

### 2. Один общий обработчик

Вместо кучи привязок к каждому полю используется:

```python
root.bind_all("<Control-KeyPress>", handle_control_key)
```

---

### 3. Защита от повторной привязки

Есть флаг:

```python
root._ctrl_shortcuts_bound
```

Он не даёт привязать обработчик несколько раз.

---

### 4. Диалоги сохраняют контроллер

```python
top._dialog_controller = self
```

Благодаря этому `Ctrl+S` понимает, какой именно диалог нужно сохранить.

---

### 5. Используется `return "break"`

Это останавливает дальнейшую обработку события, чтобы действие не выполнялось дважды.

---

## Краткая схема использования

```text
1. Скопировать модуль с хоткеями.
2. Вызвать bind_shortcuts() после построения интерфейса.
3. Вызвать set_active_app(app), если есть команды главного окна.
4. В диалогах делать top._dialog_controller = self.
5. В диалогах привязать Escape / Return / Ctrl+S.
```
