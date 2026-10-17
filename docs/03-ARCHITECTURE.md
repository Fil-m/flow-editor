# 03 · Цільова архітектура

## 1. Принципи

1. **Час — єдине джерело кадру.** `renderFrame(doc, t)` — чиста функція від документа і часу.
   Жодних `Math.random()`, `Date.now()`, незалежних CSS-анімацій.
2. **Документ — це дані.** JSON за схемою, валідується на вході, ніколи не інтерполюється в HTML як рядок.
3. **Один движок — три споживачі:** редактор, мініатюри, headless-рендер.
4. **Пресет = плагін.** Новий візуал не потребує змін у движку.
5. **Прогресивна складність:** vanilla JS + Vite, без важкого фреймворку для сцени.

## 2. Стек

| Шар | Вибір | Чому |
|---|---|---|
| Збірка | **Vite** + TypeScript (strict) | швидко, статичний output для Pages |
| UI-оболонка (панелі, модалки, інспектор) | **Preact** + signals *(або vanilla, якщо власник проти фреймворку)* | малий розмір, реактивність без ручних `paint*()` |
| Сцена | власний движок (DOM + WAAPI + Canvas 2D), пізніше опційно WebGL | зберегти перевірене ядро |
| Валідація | **Zod** (схема → типи + runtime) | B1, B11, B12 |
| Сховище | **IndexedDB** (`idb`) — проєкти + медіа-блоби | B20 |
| Тести | **Vitest** (unit), **Playwright** (e2e + візуальна регресія кадрів) | |
| Рендер-сервер | **Node 20 + Playwright Chromium + FFmpeg**, черга **BullMQ/Redis** | відтворюваний бекенд (B2) |
| Сховище файлів | S3-сумісне (Cloudflare R2 / Hetzner Object Storage) | |
| Інфра | Docker Compose → Hetzner VPS; фронт — GitHub Pages / Cloudflare Pages | |
| Моніторинг | Sentry (фронт + сервер), Plausible (аналітика без cookies) | |

## 3. Структура репозиторію (monorepo)

```
flow-editor/
├─ apps/
│  ├─ editor/                # Vite-застосунок (те, що на Pages)
│  │  ├─ src/
│  │  │  ├─ ui/              # панелі, інспектор, таймлайн, модалки
│  │  │  ├─ stage/           # маніпулятор (gizmo) поверх сцени
│  │  │  ├─ store/           # стан, команди, undo/redo
│  │  │  ├─ persistence/     # IndexedDB, імпорт/експорт, міграції
│  │  │  └─ main.ts
│  │  └─ index.html
│  └─ render-server/         # Node: API + воркер
│     ├─ src/api.ts
│     ├─ src/worker.ts
│     └─ Dockerfile
├─ packages/
│  ├─ schema/                # Zod-схеми, типи, migrate()
│  ├─ engine/                # renderFrame, mount, applyTime, keyframes, easing
│  └─ presets/               # кожен візуал — окремий файл
├─ templates/                # готові проєкти-шаблони (*.flow.json)
├─ docs/
└─ .github/workflows/
```

## 4. Модель документа v2

```ts
interface FlowDoc {
  schema: 2;
  id: string;                 // nanoid(12)
  meta: {
    title: string;
    format: "9:16" | "1:1" | "4:5" | "16:9";
    w: number; h: number;     // похідні від format
    fps: 24 | 30 | 60;
    duration: number;         // с, редагується явно
    bg: Color;
  };
  tracks: Track[];            // порядок = z-порядок (0 — найнижчий)
  items: Item[];
  assets: Asset[];            // медіа, на які посилаються items
}

interface Track { id: string; name: string; kind: "visual" | "audio"; muted?: boolean; locked?: boolean; hidden?: boolean; }

interface Item {
  id: string;
  type: string;               // ключ у реєстрі пресетів
  track: string;
  start: number; dur: number; // с
  offset: number;             // колишній anchor: зсув локального часу (для split)
  transform: { x: number; y: number; w: number; h: number; rot: number; scale: number; alpha: number; blur: number };
  params: Record<string, Param>;          // специфічні для пресета, валідуються схемою пресета
  keyframes?: Record<string, Keyframe[]>; // "transform.x", "params.color" ...
  inAnim?: AnimRef; outAnim?: AnimRef;    // анімації появи/зникнення (окремо від loop)
  speed: number;              // множник локального часу — для ВСІХ анімацій
}

interface Keyframe { t: number; v: number | string; ease: EaseName | [number, number, number, number]; }
type Color = `#${string}`;    // валідація /^#[0-9a-f]{6}$/i
```

Ключові відмінності від v1: `transform` відокремлено від `params`; `anim` прибрано;
`anchor` → `offset`; явні `tracks` з z-порядком; `assets`; кейфрейми.

### Міграції

`packages/schema/migrate.ts`: ланцюжок `v1 → v2 → …`, кожен крок — чиста функція
з юніт-тестом на фікстурі. Невідомий `type` → елемент-заглушка «Невідомий візуал» (не втрачається).

## 5. Контракт пресета

```ts
interface Preset<P> {
  id: string; name: I18n; group: string; version: number;
  kind: "dom" | "canvas" | "webgl" | "media";
  params: ZodSchema<P>;                         // діапазони, дефолти, UI-підказки
  defaults: { transform: Partial<Transform>; params: P; dur: number };
  create(ctx: MountCtx, item: Item): Instance;  // будує DOM через createElement / style.setProperty — НЕ innerHTML
}
interface Instance {
  update(params: P): void;        // зміна параметрів без повного remount
  frame(localT: number): void;    // детермінований кадр
  destroy(): void;
}
```

Інспектор генерується з `params`-схеми (min/max/step/control). Це прибирає глобальні
`LABEL`/`RANGE`/`SEL` і баг B4.

## 6. Движок

```
renderFrame(t):
  for track in tracks (z-order):
    for item in track.items where start <= t < start+dur:
      local = (t - start + offset) * speed
      props  = evalKeyframes(item, local)        // transform + params
      inst.update(props.params) if changed
      inst.frame(local)
      applyTransform(el, props.transform, inAnim/outAnim(local))
```

- Відтворення: `requestAnimationFrame`, час з `AudioContext.currentTime`, коли є аудіо (синхрон звуку).
- Індекс `Map<id, Item>` замість `some()` (B14). Видимість — через інтервальне дерево при >200 елементах.
- `ResizeObserver` для fit (B16).

## 7. Стан і команди

- Store: `doc` + `ui` (selection[], playhead, zoom, tool).
- Кожна зміна — **команда** `{do, undo, label, mergeKey}`; слайдер під час drag зливає кроки
  в один (mergeKey) → вирішує B3 без 60 снепшотів по 50 КБ.
- Undo-стек — до 200 команд; persist у IndexedDB для відновлення після перезавантаження.

## 8. Рендер-сервер

### API (v1)

| Метод | Шлях | Тіло / відповідь |
|---|---|---|
| `POST` | `/v1/renders` | `{doc, w, h, fps, quality}` → `202 {id}` |
| `GET` | `/v1/renders/:id` | `{state: queued\|rendering\|encoding\|done\|error, frame, total, eta, url?, error?}` |
| `DELETE` | `/v1/renders/:id` | скасувати |
| `POST` | `/v1/assets` | presigned upload URL для медіа |
| `GET` | `/healthz` | |

Обмеження: тіло ≤ 2 МБ, `duration ≤ 180 с`, `items ≤ 500`, rate-limit per IP / per user,
схема валідується **тією ж** Zod-схемою, що й на клієнті.

### Пайплайн

1. Воркер піднімає Chromium (Playwright), відкриває `editor/render.html` (той самий `engine`).
2. Для кожного кадру: `renderFrame(t)` → `page.screenshot({type:"png", clip})` або
   швидше — `HeadlessExperimental.beginFrame` / CDP `Page.captureScreenshot`.
3. Кадри стримом у `ffmpeg -f image2pipe … -c:v libx264 -pix_fmt yuv420p -crf 18`, аудіо мікшується FFmpeg.
4. Результат у S3, посилання з TTL 7 днів. Тимчасові файли видаляються.
5. Паралелізм: шардинг діапазону кадрів між N вкладками, потім `concat`.

Ціль: 30 с @ 30 fps 1080p (900 кадрів) ≤ 60 с на 4 vCPU.

> [!] **Ціль не обґрунтована.** Нині за README ≈ 0.4 с/кадр → 900 кадрів ≈ 6 хв послідовно. Для 60 с потрібне ≥ 6× прискорення
> (шардинг на 4 вкладки дає ~3–4×, решту — кеш статичних шарів, JPEG-кадри замість PNG, `-preset veryfast`).
> **Перший крок Ф4 — бенчмарк** на реальному залізі; після нього перегляньте ціль (реалістично 90–120 с).

### Альтернатива: клієнтський експорт

Для коротких відео (≤ 30 с) можливий експорт у браузері через **WebCodecs `VideoEncoder` + mp4-muxer**.
Але це працює лише коли кадр можна отримати як bitmap: DOM + CSS-фільтри надійно в canvas не переносяться.
Тому передумова — міграція візуалів на Canvas/WebGL (довга робота). Планується як оптимізація після Фази 4, не як заміна сервера.

## 8a. Відносні одиниці (передумова багатьох форматів)

У v0 усі розміри — абсолютні px під 1080×1920 (аудит B24). Щоб підтримати 1:1 / 4:5 / 16:9, у `engine` слід ввести
сцену-контейнер з `container-type: size` і одиниці `cqw`/`cqh`, або єдиний множник `k = min(w/1080, h/1920)`,
на який множаться px-значення пресетів. Це робиться у Ф1 (разом з перенесенням пресетів), а не у Ф2.

## 9. Безпека

- Жодного `innerHTML` з даними документа; кольори — тільки через `style.setProperty` після регекс-валідації.
- CSP: `default-src 'self'; img-src 'self' blob: data: https://*.r2.dev; connect-src 'self' https://api.flow…`.
- Сервер рендера: Chromium без `--no-sandbox` де можливо, окремий непривілейований користувач/контейнер, **заборона мережі** вкладки
  (крім бакету ассетів), без доступу до метаданих хмари і внутрішніх адрес. Документ валідується на сервері тією ж схемою.
- Імпортований JSON → Zod `safeParse` → міграція → тільки потім у store.

## 10. ADR (архітектурні рішення — фіксувати в `docs/adr/`)

| ADR | Рішення | Статус |
|---|---|---|
| 001 | Зберегти DOM+WAAPI движок, не переходити на Remotion/Lottie | запропоновано |
| 002 | Vite + TS monorepo замість одного HTML | запропоновано |
| 003 | Preact для UI-оболонки | **[?]** потрібне рішення |
| 004 | Серверний рендер — основний, WebCodecs — опційний | запропоновано |
| 005 | IndexedDB як основне локальне сховище | запропоновано |
