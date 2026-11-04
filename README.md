# Node-RED: смена и проверка пароля администратора

**Окружение:** Docker-контейнер `node-red-app-1`, данные на сервере — `/home/node-red/app/data`, настройки — `settings.js` (в контейнере `/data/settings.js`). Команды выполнять по SSH на хосте.

## 1. Проверить файл и сделать копию

```bash
cd /home/node-red/app/data
docker inspect node-red-app-1 --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
docker logs node-red-app-1 2>&1 | grep -i 'Settings file' | tail -1
ls -l settings.js
cp -p settings.js "settings.js.bak.$(date +%F-%H%M%S)"
```

**Важно:** перед продолжением убедиться, что `/home/node-red/app/data` смонтирована в контейнер как `/data`, а Node-RED загружает `/data/settings.js`.

## 2. Сменить пароль

```bash
docker exec -it node-red-app-1 node-red admin hash-pw
nano /home/node-red/app/data/settings.js
```

Скопировать **целиком** полученный bcrypt-хеш (обычно 60 символов с префиксом `$2a$` или `$2b$`) в поле `password` нужного пользователя в `adminAuth.users`. **Не вставлять сам пароль.** Пример фрагмента:

```js
adminAuth: {
    type: "credentials",
    users: [{
        username: "admin",
        password: "$2b$08$...ПОЛНЫЙ_СГЕНЕРИРОВАННЫЙ_ХЕШ...",
        permissions: "*"
    }]
},
```

Сохранить: `Ctrl+O`, `Enter`, `Ctrl+X`.

## 3. Проверить синтаксис и перезапустить только Node-RED

```bash
docker exec node-red-app-1 node --check /data/settings.js
docker restart node-red-app-1
docker ps --filter name=node-red-app-1
docker logs --tail 40 node-red-app-1
```

Если `node --check` выдаёт ошибку — **не перезапускать**, сначала исправить `settings.js`.

## 4. Проверить, что введённый пароль соответствует хешу

Выполнить в **bash** (пароль не показывается на экране и не попадает в историю команд):

```bash
read -rsp 'Пароль для проверки: ' NR_PASS; echo
printf '%s' "$NR_PASS" | docker exec -i node-red-app-1 node -e '
const fs = require("fs");
const bcrypt = require("bcryptjs");
const s = require("/data/settings.js");
const user = s.adminAuth?.users?.find(u => u.username === "admin");
if (!user) { console.error("Пользователь admin не найден"); process.exit(1); }
const pw = fs.readFileSync(0, "utf8");
console.log("Пароль совпадает:", bcrypt.compareSync(pw, user.password));
'
unset NR_PASS
```

Результат `Пароль совпадает: true` — правильный пароль для хеша в файле. `false` — пароль и хеш не совпадают. После этого проверить вход через веб-интерфейс. Если пользователь называется не `admin`, заменить его имя в команде проверки.

## Если что-то пошло не так

Восстановить **нужную** резервную копию (подставить её точное имя):

```bash
cd /home/node-red/app/data
ls -lt settings.js.bak.*
cp -p settings.js.bak.YYYY-MM-DD-HHMMSS settings.js
docker restart node-red-app-1
```

**Безопасность:** не коммитить `settings.js`, резервные копии, реальные хеши, пароли и токены в Git. В репозиторий добавлять только эту инструкцию.

**Источник:** [официальная документация Node-RED — Securing Node-RED](https://nodered.org/docs/user-guide/runtime/securing-node-red).
