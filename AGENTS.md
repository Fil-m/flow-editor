# AGENTS.md — правила для розробників і AI-агентів

Читати перед будь-якою зміною коду. Повна документація — у [`docs/`](docs/README.md).

## Інваріанти (не порушувати)

1. **Кадр залежить тільки від `(doc, t)`.** Ніяких `Math.random()`, `Date.now()`, `performance.now()`
   у рендері. Випадковість — лише `seeded(item.id + сіль)`.
2. **Усі зміни документа — через `mut()` / команди store.** Інакше ламається undo і autosave.
3. **Дані документа не потрапляють у `innerHTML` як рядок.** Кольори/числа/enum — через санітизацію;
   текст — через `textContent` або `esc()`.
4. **Анімації WAAPI завжди на паузі**, час ставиться вручну в `applyTime`. Easing — у keyframes, не в опціях (баг Chrome).
5. **Зміна схеми документа = підняти версію + написати міграцію + фікстуру.**
6. Новий візуал = новий запис у реєстрі пресетів. Движок не чіпати.

## Як додати пресет (поточна v0)

```js
reg("myfx", {
  name: "Назва", group: "Група",
  def: { x:50, y:50, w:60, h:60, alpha:1, speed:1, dur:6, anim:6, rot:0, blur:0, color:"#ffffff" },
  fields: ["color", "x", "y", "w", "alpha", "speed"],
  node(it) { /* розмітка; тільки санітизовані значення */ },
  anims(it) { return [{ sel:"div", loop:true, kf:[…], opt:{ duration: 4000, easing:"ease-in-out" } }]; }
});
```

## Локальний запуск

v0: відкрити `index.html` у браузері або `npx serve .`.
Після Фази 1: `pnpm i && pnpm dev`.

## Гілки і коміти

- `main` — завжди деплоїться; робота в `feat/<id>-<slug>`, `fix/<id>-<slug>`.
- Conventional Commits: `feat(timeline): multi-track (F2-1)`.
- Версії SemVer, теги `vX.Y.Z`, `CHANGELOG.md`.
