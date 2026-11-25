# 🎡 Поле Чудес — ТВ-шоу на телефоне

Полностью рабочая веб-игра по мотивам российской телепередачи «Поле Чудес» (аналог Wheel of Fortune).

**Один `index.html`, без сборщиков и фреймворков.** Vanilla JS + Canvas + WebAudio.
Соло-режим работает офлайн, мультиплеер — через Firebase (бесплатный тариф Spark).

---

## Что внутри

- **Соло «против ведущего»**: 3 раунда + финальный раунд, саркастичный ведущий с банком реплик 40+.
- **Мультиплеер «комната по коду»**: 2–4 игрока, ходы по кругу, таймер 20 секунд, живой «эфир».
- **Колесо**: ×1, ×2, ×5, ×10, ПРИЗ (конверт), БАНКРОТ, ПРОПУСК. Анимация ~2,8 с, WebAudio-звуки без файлов.
- **Шкаф наград**: 12 наград с условиями и прогрессом + полка фигурок.
- **Профиль**: матчи, победы, лучший счёт, угаданные буквы.
- **PWA**: `manifest.json` + `sw.js` с очисткой старых кэшей.
- **Telegram Mini App**: берёт имя из `initDataUnsafe.user`, иначе «Гость».
- **Тёмный неон**: фиолетово-чёрный градиент, золото, крупная кириллица, мобильный-first (360×640 без скролла).

---

## Структура

```
.
├── index.html              # весь HTML + CSS + JS (однофайловое приложение)
├── manifest.json           # PWA-манифест
├── sw.js                   # service worker (кэш + очистка старых)
├── icon-192.png
├── icon-512.png
├── icon-maskable-512.png
└── README.md
```

Больше ничего не нужно: ни `node_modules`, ни сборки, ни внешних картинок.

---

## Быстрый старт локально

Так как используется service worker и `localStorage`, открывать лучше через локальный сервер:

```bash
cd polechudes
python3 -m http.server 8080
# открыть http://localhost:8080
```

> Можно открыть и просто `index.html` двойным кликом — соло-режим заработает, но PWA-установка будет недоступна.

---

## Деплой на GitHub Pages за 10 минут

1. Создайте репозиторий `polechudes` (public, **без** README и без `.gitignore`).
2. Положите в корень файлы: `index.html`, `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png`, `icon-maskable-512.png`.
3. Запушьте:
   ```bash
   git init
   git add .
   git commit -m "Поле Чудес"
   git branch -M main
   git remote add origin https://github.com/ВАШ_ЛОГИН/polechudes.git
   git push -u origin main
   ```
4. **Settings → Pages → Deploy from a branch → main / (root) → Save.**
5. Через 1–2 минуты приложение доступно по адресу:
   ```
   https://ВАШ_ЛОГИН.github.io/polechudes/
   ```

> Важно: Firebase и Telegram требуют **HTTPS** — GitHub Pages его даёт автоматически.

---

## Telegram Mini App (2 минуты)

1. В Telegram откройте **@BotFather** → `/newbot` → задайте имя и username бота.
2. `/mybots` → выберите бота → **Bot Settings → Menu Button** → установите:
   - Text: `Играть`
   - URL: `https://ВАШ_ЛОГИН.github.io/polechudes/`
3. (Необязательно) **Bot Settings → Menu Button** тоже можно заменить на Web App.
4. Готово: у бота появится кнопка «Играть». Внутри Telegram приложение получит
   `initDataUnsafe.user.first_name` и покажет имя игрока.

Можно также добавить кнопку через `/setmenubutton` или используя `web_app` в `InlineKeyboardButton`.

---

## Firebase (мультиплеер), ~7 минут

Мультиплеер — единственная внешняя зависимость. Соло работает и без него.

### 1. Проект

1. [console.firebase.google.com](https://console.firebase.google.com) → **Add project** → имя `polechudes` → Analytics можно выключить.
2. **Spark (бесплатный)** план — его хватает с запасом.

### 2. Анонимная авторизация

**Build → Authentication → Sign-in method → Anonymous → Enable → Save.**

### 3. Firestore

**Build → Firestore Database → Create database → Production mode → регион (например, `europe-west1`).**

Затем **Rules** → вставьте блок из раздела «Правила Firestore» ниже → **Publish**.

### 4. Веб-приложение и конфиг

**Project settings → General → Your apps → Web app (`</>`) → Register app.**

Скопируйте `apiKey`, `authDomain`, `projectId`, `messagingSenderId`, `appId` и вставьте в `index.html` в начало скрипта:

```js
const FIREBASE_CONFIG = {
  apiKey: "AIza...",
  authDomain: "polechudes-xxxxx.firebaseapp.com",
  projectId: "polechudes-xxxxx",
  messagingSenderId: "1234567890",
  appId: "1:1234567890:web:abcdef"
};
```

> Альтернатива: создайте рядом `config.js` с `window.PZ_FIREBASE_CONFIG = {...}` и подключите его **перед** `index.html` — тогда конфиг не придётся править в основном файле.
> Если `apiKey` пустой — мультиплеер вежливо отключается, соло работает всегда.

### 5. Правила Firestore

Вставьте в **Firestore → Rules** и опубликуйте:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isAuth() {
      return request.auth != null;
    }

    function isPlayer(room) {
      return room.players.any(p => p.uid == request.auth.uid);
    }

    // ---------- комнаты ----------
    match /rooms/{code} {
      allow read: if isAuth();

      allow create: if isAuth()
        && request.resource.data.code == code
        && request.resource.data.hostId == request.auth.uid
        && request.resource.data.status == 'lobby'
        && request.resource.data.players is list
        && request.resource.data.players.size() >= 1
        && request.resource.data.players.size() <= 4;

      allow update: if isAuth()
        && request.resource.data.code == code
        && (
          // лобби: хост и участники могут менять состав, хост может передать права
          (resource.data.status == 'lobby'
            && (request.auth.uid == resource.data.hostId
                || isPlayer(resource.data)
                || request.resource.data.players.any(p => p.uid == request.auth.uid)))
          // игра: пишет только активный игрок; хост — только на переходах раунда/матча
          || (resource.data.status == 'playing'
            && request.resource.data.hostId == resource.data.hostId
            && (request.auth.uid == resource.data.turnUid
                || (request.auth.uid == resource.data.hostId
                    && resource.data.phase in ['roundEnd', 'matchEnd'])))
          // завершённая комната: только хост
          || (resource.data.status == 'finished'
            && request.auth.uid == resource.data.hostId)
        );

      allow delete: if false;
    }

    // ---------- эфир комнаты ----------
    match /feed/{id} {
      allow read: if isAuth();
      allow create: if isAuth()
        && request.resource.data.room is string
        && request.resource.data.room.size() <= 32
        && request.resource.data.text is string
        && request.resource.data.text.size() <= 200
        && request.resource.data.ts is number;
      allow update, delete: if false;
    }

    // ---------- профили ----------
    match /profiles/{uid} {
      allow read: if isAuth() && request.auth.uid == uid;
      allow create, update: if isAuth() && request.auth.uid == uid;
      allow delete: if false;
    }
  }
}
```

Что важно:

- **delete = false** везде: комнаты и профили не удаляются, только помечаются `status: 'finished'`.
- **Белые списки**: писать в комнату может только её активный игрок (или хост на переходах).
- **Размер**: 1–4 игрока, код = имя документа, текст эфира ≤ 200 символов.
- **Анонимность**: `request.auth.uid` — единственная идентификация, email/пароль не нужны.

---

## Как играть

### Соло

1. «Играть соло» → 3 раунда по слову из 12 категорий.
2. «КРУТИТЬ» → сектор → назовите букву на клавиатуре.
3. Буква есть — открываются все вхождения, очки = `сектор × число букв × 10`.
   Дальше «Крутить снова» или «Открыть слово» (бонус +300).
4. БАНКРОТ — сгорают очки **раунда**; ПРОПУСК — ход ведущего (он открывает одну букву).
5. После 3 раундов — **ФИНАЛ**: 3 согласные + 1 гласная, затем слово целиком.
   Очки финала ×2, угадали слово — **джекпот ×5**. Три ошибки — финал проигран.

### Мультиплеер

1. «Комната друзей» → «Создать комнату» → код из 4 символов.
2. Друзья вводят код → «Войти». Хост жмёт «Начать игру» (нужно ≥ 2).
3. Ход по кругу, на ход 20 секунд (потом автопередача).
4. После 3 раундов лидер играет финал. Победитель — максимальный счёт.

### Награды

12 наград: 🥉 Первое выступление, 🎬 Знаток кино, 🔠 Буквоед, 💥 Банкрот-тусовка,
🚗 Мерседес мечты, 🌍 Кругосветка, ⚡ Молния, 👑 Мистер Поле чудес, 🎁 Счастливчик Приза,
🧩 Слово года, 🚀 Несгораемый, 💎 Бриллиантовый зал.

Плюс **полка фигурок** — выпадают в секторе ПРИЗ (6 штук).

---

## Чеклист самопроверки

- [ ] `index.html` открывается локально, соло-матч проходится до финала.
- [ ] После матча профиль обновился: матчи +1, победа/поражение, рекорд, буквы.
- [ ] В шкафу наград открылась «Первое выступление»; карточки с 🔒 показывают условие и прогресс.
- [ ] Звук включается/выключается, состояние сохраняется после перезагрузки.
- [ ] PWA: в DevTools → Application → Manifest есть имя, иконки, `display: standalone`.
- [ ] Офлайн: после первой загрузки страница открывается без сети (соло).
- [ ] Два окна инкогнито: комната по коду, оба видят ход, буквы и итог.
- [ ] В Telegram: имя из профиля Telegram, кнопка «Играть» открывает приложение.
- [ ] Ошибки Firebase не ломают соло: при пустом конфиге кнопка мультиплеера показывает подсказку.

---

## Намеренно не сделано

| Что | Почему |
|---|---|
| **Античит в мультиплеере** | Слово хранится в документе комнаты и теоретически читается всеми. Осознанно: «честная игра среди друзей», как и просили в ТЗ. |
| **Удаление комнат/профилей** | `delete = false` в правилах — история сохраняется, комнате присваивается `status: 'finished'`. |
| **Регистрация и email** | Только анонимный вход Firebase — без паролей и ПДн. |
| **Реальные деньги/ставки** | Только виртуальные очки, титулы и фигурки. |
| **Серверная валидация слов** | Словарь встроен, судья — клиент. Для дружеской игры этого достаточно. |
| **Переподключение при обрыве** | Комната живёт в Firestore, но если у активного игрока пропала сеть, ход может зависнуть до таймаута. |
| **Банкрот/пропуск в финале** | Финал идёт без колеса — по правилам ТЗ. |

---

## Где что менять

| Что | Где |
|---|---|
| Словарь (12 категорий × 14+) | `const WORDS = {...}` в `index.html` |
| Реплики ведущего | `const HOST = {...}` |
| Награды | `const AWARDS = [...]` |
| Фигурки | `const FIGURES = [...]` |
| Сектора колеса | `const SECTORS = [...]` |
| Цвета/темы | `:root` в `<style>` |
| Firebase | `const FIREBASE_CONFIG` в начале скрипта |

---

## Лицензия

Учебный/развлекательный проект. «Поле Чудес» — традиция телепередачи; все совпадения с реальными людьми и барабанами случайны. 🎡
