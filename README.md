# Konsultacijų apskaita — развёртывание на Firebase + GitHub

Что изменилось по сравнению с версией внутри Claude: раньше данные сохранялись
во временном хранилище, которое работает только внутри чата. Теперь приложение
использует настоящую базу данных Firebase (Firestore) и вход по e-mail/паролю,
чтобы данные о консультациях не были доступны посторонним в интернете.

## Что нужно один раз установить на компьютер

1. **Node.js** (если ещё нет) — https://nodejs.org (LTS-версия).
2. **Firebase CLI**:
   ```
   npm install -g firebase-tools
   ```
3. **Git** (если ещё нет) — https://git-scm.com

## Шаг 1. Создать проект в Firebase

1. Откройте https://console.firebase.google.com и нажмите «Add project» / «Добавить проект».
2. Дайте проекту имя, например `konsultaciju-apskaita`.
3. В меню слева откройте **Build → Authentication** → «Get started» →
   включите способ входа **Email/Password**.
4. Там же, во вкладке **Users**, добавьте себя вручную: нажмите «Add user»,
   впишите свой e-mail и пароль. Это и будет ваш логин в приложении.
5. Если хотите ещё и вход через Google: там же, в **Sign-in method**,
   включите провайдер **Google**, укажите «Project support email» и
   сохраните.
6. В меню слева откройте **Build → Firestore Database** → «Create database» →
   выберите режим **Production mode** и ближайший регион (например `eur3`).

## Шаг 2. Взять конфигурацию проекта

1. В Firebase Console нажмите на шестерёнку рядом с «Project Overview» →
   **Project settings**.
2. Внизу раздела «Your apps» нажмите значок `</>` (Web), придумайте имя
   приложения и нажмите «Register app». Регистрировать hosting отдельно не нужно.
3. Скопируйте появившийся объект `firebaseConfig` — он выглядит так:
   ```js
   const firebaseConfig = {
     apiKey: "...",
     authDomain: "...",
     projectId: "...",
     storageBucket: "...",
     messagingSenderId: "...",
     appId: "..."
   };
   ```
4. Откройте файл `public/index.html` в этом проекте, найдите блок
   `// ====== 1. FIREBASE CONFIG ======` и замените значения-заглушки на свои.

## Шаг 2а. Кто имеет доступ (важно при включённом Google-входе)

Пока вход был только по паролю, доступ был закрыт автоматически — только те,
кого вы вручную добавили в Authentication → Users. С включённым Google-входом
**любой** человек с Google-аккаунтом сможет войти и увидеть базу консультаций,
если не ограничить доступ явно.

Список разрешённых e-mail нужно указать в двух местах (они должны совпадать):

1. В файле `firestore.rules`:
   ```
   allow read, write: if request.auth != null &&
     request.auth.token.email in [
       'ваш.email@school.lt',
       'коллега@school.lt'
     ];
   ```
2. В файле `public/index.html`, в переменной `ALLOWED_EMAILS`:
   ```js
   const ALLOWED_EMAILS = ['ваш.email@school.lt', 'коллега@school.lt'];
   ```

Второй список нужен только для того, чтобы человек без доступа сразу видел
понятное сообщение — реальную защиту данных обеспечивает `firestore.rules`.
После изменения `firestore.rules` не забудьте выполнить:
```
firebase deploy --only firestore:rules
```
(или `npx firebase-tools deploy --only firestore:rules`, если firebase-tools
не установлен глобально).

Если хотите пускать вообще любого пользователя без ограничения по e-mail
(не рекомендуется для данных о консультациях), можно оставить
`ALLOWED_EMAILS = []` и в `firestore.rules` — `if request.auth != null;`.

## Шаг 3. Залить проект на GitHub

1. Создайте новый пустой репозиторий на https://github.com/new (например,
   `konsultaciju-apskaita`). Ничего не отмечайте (без README, без .gitignore).
2. В папке проекта на компьютере выполните:
   ```
   git init
   git add .
   git commit -m "Pirmas commit"
   git branch -M main
   git remote add origin https://github.com/ВАШ_ЛОГИН/konsultaciju-apskaita.git
   git push -u origin main
   ```

## Шаг 4. Подключить Firebase Hosting к этому GitHub-репозиторию

Самый простой способ — команда `firebase init`, которая сама создаёт
GitHub Action и добавляет нужный секретный ключ в репозиторий.

1. В папке проекта выполните:
   ```
   firebase login
   firebase init hosting
   ```
   - «Use an existing project» → выберите созданный проект.
   - «What do you want to use as your public directory?» → введите `public`
     (папка уже существует, просто подтвердите).
   - «Configure as a single-page app?» → **No**.
   - «Set up automatic builds and deploys with GitHub?» → **Yes**.
   - Войдите через GitHub, когда попросит, и выберите ваш репозиторий
     `ВАШ_ЛОГИН/konsultaciju-apskaita`.
   - «Set up the workflow to run a build script before every deploy?» → **No**.
   - Согласитесь перезаписать файл `firebase.json`, если спросит (он такой же).

   Это автоматически создаст GitHub Action и положит секретный ключ доступа
   в настройки репозитория — готовый файл `.github/workflows/firebase-hosting-merge.yml`
   из этого проекта тогда не нужен, `firebase init` создаст свой.

2. Также выполните, чтобы включить правила базы данных:
   ```
   firebase deploy --only firestore:rules
   ```

## Шаг 5. Проверить

После каждого `git push` в ветку `main` GitHub Action сам соберёт и
опубликует сайт. Ссылку на сайт (вида `https://ВАШ_ПРОЕКТ.web.app`) можно
увидеть в Firebase Console → Hosting, либо сразу после первого
`firebase deploy` в терминале.

Откройте ссылку, войдите под своим e-mail и паролем — приложение готово.

## Как вносить изменения в будущем

1. Меняете файл `public/index.html` (или просите об этом здесь, в чате).
2. `git add . && git commit -m "описание изменения" && git push`
3. Через минуту-две сайт обновится автоматически — GitHub Action сделает
   деплой сам.

## Если нужно добавить второго психолога

Просто добавьте ещё одного пользователя в Firebase Console → Authentication →
Users. Оба увидят одну и ту же общую базу консультаций.
