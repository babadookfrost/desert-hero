# Герой Пустыни — заметки для агента

## Структура проекта
- Вся игра — один файл `index.html` (~5900 строк): CSS в `<style>`, JS в `<script type="module">`.
- Three.js подключается через importmap из CDN (unpkg three@0.160.0).
- `serve.js` — статический сервер на чистом Node, порт из `PORT` (по умолчанию 3000), хост `0.0.0.0`.
- `package.json` без зависимостей.

## Архитектура кода (index.html)
- JS — один модуль верхнего уровня; ~361 top-level объявления (функции/классы/let/const).
- State-машина: `'menu' | 'playing' | 'paused' | 'dead' | 'won' | 'story' | 'intro'`.
- DOM-доступ через `const dom = {...}` (все ключи используются, без мусора).
- localStorage: префикс `gd_` (статистика), `gd_ach_` (достижения), `gd_upgrades`, `gd_wheel` и т.д.

## Проверки кода (аудит 2026-08)
- `node --check` — синтаксис валиден.
- AST-анализ (acorn): повторных объявлений в одной области видимости нет; необъявленных/неявно-глобальных переменных нет.
- Все `id`, на которые ссылается JS (`$()`/`getElementById`), есть в HTML (кроме `fatal-banner` — создаётся динамически при ошибке).
- CSS: скобки сбалансированы, конфликтующих селекторов нет.
- В headless-браузере не создаётся WebGL-контекст — это ограничение среды, не баг кода.

### Известная мелкая несогласованность (не критично)
- В `saveStats()` свойство `bestScore` сохраняется дважды: как `gd_bestScore` (через цикл `for k in stats`) и отдельно как `gd_best`. Загружается же только из `gd_best`. Ключ `gd_bestScore` фактически «мёртвый» (записывается, но не читается). Работает корректно, но избыточно.

## Графика (ААА-апгрейд 2026-08)
### Волна 1 — материалы/тени/туман
- Рендерер: ACES tone mapping, PCFSoft shadows + bias/normalBias/radius (чистые тени), powerPreference high-perf.
- Анизотропия: `maxAniso = renderer.capabilities.getMaxAnisotropy()` применяется во всех загрузчиках (`loadTex`, `loadSkinTex`, `gunMat`, `makeNoiseTexture`) — текстуры чёткие под углом и вдаль.
- Карта уровня (rock/wall/pillar) и земля переведены с `MeshLambertMaterial` на `MeshStandardMaterial` + normal maps (`applyTexNormal` + `loadNormTex`/`normCache`). Рельеф скал/стен/песка.
- Туман: `FogExp2` (плотность `2.0/fogFar`) — естественное экспоненциальное затухание.
- Постобработка: UnrealBloom + OutputPass (уже было). Оружие: PBR + PMREM RoomEnvironment envMap (уже было).
- Poly Haven normal URL: `PHN(id)` = `.../{id}/{id}_nor_gl_1k.jpg` (все 200 OK).

### Волна 2 — пост-обработка ААА + атмосфера
- Пайплайн composer: Render → SSAO → Bloom → Vignette → SMAA → Output.
- SSAO (`SSAOPass`): затенение контактов/щелей, только Ультра (`ssaoPass.enabled = gfxQuality>=3`). kernelRadius 6.
- Виньетка (`ShaderPass(VignetteShader)`): offset 1.05, darkness 0.85 — киношное затемнение краёв.
- SMAA (`SMAAPass`): сглаживание краёв в пост-обработке (важно: renderTarget composer'а без MSAA). Включён с качества ≥1.
- Bloom перенастроен под PBR: strength [0,0.30,0.48,0.62], threshold [1,0.88,0.78,0.70] — светятся только вспышки/солнце/элитное свечение.
- Звёздное поле (`THREE.Points`, 900 звёзд на верхней полусфере r=750, `fog:false`): включается на тёмных секциях (яркость верхнего стопа неба < 0.18) без skybox.
- Resize-обработчик вызывает `ssaoPass.setSize`/`smaaPass.setSize` (pass'ы с собств. буферами не обновляются автоматически).

### Верификация
- `node --check` — синтаксис валиден.
- Импорты SSAO/SMAA/ShaderPass/VignetteShader с CDN — 200 OK, меню отрисовывается (модули грузятся без ошибок).
- WebGL-рендер недоступен в headless-окружении (нет GPU/swiftshader) — визуальная проверка требует браузера с GPU у пользователя.

## Запуск
- `PORT=12000 node serve.js` → http://localhost:12000 (или через work-hosts).
