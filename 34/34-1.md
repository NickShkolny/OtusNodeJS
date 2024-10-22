# Создание Telegram бота для управления задачами с использованием Node.js, MongoDB и Nginx


## 1. Установка необходимых библиотек

Сначала создаем новую папку для нашего проекта и переходим в неё:

```bash
mkdir task-manager-bot
cd task-manager-bot
```

Инициализируем новый проект Node.js:

```bash
npm init -y
```

Установите необходимые библиотеки:

```bash
npm install express node-telegram-bot-api mongoose node-cron body-parser
```

- **express**: фреймворк для Node.js.
- **node-telegram-bot-api**: библиотека для работы с Telegram API.
- **mongoose**: библиотека для работы с MongoDB.
- **node-cron**: библиотека для периодического запуска кода.
- **body-parser**: библиотека для обработки JSON-данных.

## 2. Создание файлов и папок

Создаем следующие файлы в корне вашего проекта:

- `db.js`
- `User.js`
- `Task.js`
- `botApp.js`
- `notificationApp.js`

## 3. Настройка подключения к MongoDB

### Файл: `db.js`

```javascript
const mongoose = require('mongoose'); // Подключаем библиотеку mongoose

// Функция для подключения к MongoDB
const connectDB = async () => {
  try {
    await mongoose.connect('mongodb://mongo-user-mygeobot:password@195.80.51.100:27017/mygeobot', { // Подключаемся к MongoDB по указанному URL
      useNewUrlParser: true, // Используем новый парсер URL
      useUnifiedTopology: true, // Используем новую топологию подключения
    });
    console.log('MongoDB подключен'); // Выводим сообщение об успешном подключении
  } catch (error) {
    console.error('Ошибка подключения к MongoDB:', error.message); // Обрабатываем ошибки подключения
    process.exit(1); // Завершаем процесс при ошибке
  }
};

module.exports = connectDB; // Экспортируем функцию подключения
```

## 4. Создание модели пользователя

### Файл: `User.js`

```javascript
const mongoose = require('mongoose'); // Подключаем библиотеку mongoose

// Определяем схему пользователя
const userSchema = new mongoose.Schema({
    chatId: {
        type: String,
        required: true,
        unique: true, // Уникальное значение для каждого пользователя
    },
    firstName: {
        type: String,
        required: true, // Обязательное поле
    },
    lastName: {
        type: String,
        required: false, // Необязательное поле
    },
    username: {
        type: String,
        required: false, // Необязательное поле
    },
    createdAt: {
        type: Date,
        default: Date.now, // Дата создания по умолчанию
    },
});

// Создаем модель пользователя
const User = mongoose.model('User', userSchema); // Создаем модель на основе схемы

module.exports = User; // Экспортируем модель пользователя
```

## 5. Создание модели задачи

### Файл: `Task.js`

```javascript
const mongoose = require('mongoose'); // Подключаем библиотеку mongoose

// Определяем схему задачи
const taskSchema = new mongoose.Schema({
    userId: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'User', // Связываем с моделью пользователя
        required: true, // Обязательное поле
    },
    taskId: {
        type: Number,
        required: true, // Обязательное поле
        unique: true, // Уникальное значение для каждой задачи
    },
    text: {
        type: String,
        required: true, // Обязательное поле
    },
    status: {
        type: String,
        enum: ['актуальна', 'выполнена'], // Статусы задачи
        default: 'актуальна', // Статус по умолчанию
    },
    reminderTime: {
        type: Date, // Время напоминания
        required: false, // Необязательное поле
    },
    createdAt: {
        type: Date,
        default: Date.now, // Дата создания по умолчанию
    },
});

// Создаем модель задачи
const Task = mongoose.model('Task', taskSchema); // Создаем модель на основе схемы

module.exports = Task; // Экспортируем модель задачи
```

## 6. Создание Telegram бота

### Файл: `botApp.js`

```javascript
const express = require('express'); // Подключаем библиотеку express
const bodyParser = require('body-parser'); // Подключаем body-parser для обработки JSON
const TelegramBot = require('node-telegram-bot-api'); // Подключаем библиотеку для работы с Telegram API
const connectDB = require('./db'); // Подключаем функцию подключения к MongoDB
const User = require('./User'); // Подключаем модель пользователя
const Task = require('./Task'); // Подключаем модель задачи

const app = express(); // Создаем экземпляр приложения express
const PORT = process.env.PORT || 3030; // Устанавливаем порт для сервера
const TELEGRAM_TOKEN = 'token'; // Заменяем на наш токен
const WEBHOOK_URL = 'https://tc-app.ru/webhook'; // Заменяем на наш URL

// Подключаемся к MongoDB
connectDB(); // Вызываем функцию подключения к базе данных

// Middleware для обработки входящих запросов
app.use(bodyParser.json()); // Используем body-parser для обработки JSON

// Создаем экземпляр бота
const bot = new TelegramBot(TELEGRAM_TOKEN); // Создаем экземпляр бота с токеном

// Функция для установки вебхука
const setWebhook = async () => {
  try {
    await bot.setWebHook(WEBHOOK_URL); // Устанавливаем вебхук
    console.log('Webhook установлен успешно'); // Выводим сообщение об успешной установке
  } catch (error) {
    console.error('Ошибка установки вебхука:', error.message); // Обрабатываем ошибки установки
    process.exit(1); // Завершаем процесс при ошибке
  }
};

// Запускаем установку вебхука
setWebhook();

// Переменная для хранения последнего ID задачи
let lastTaskId = 0;

// Обработка входящих сообщений
app.post('/webhook', async (req, res) => {
    const message = req.body.message; // Получаем сообщение из тела запроса
    const chatId = message.chat.id; // Получаем ID чата

    // Проверяем, есть ли пользователь в базе данных
    let user = await User.findOne({ chatId: chatId.toString() });
    if (!user) {
        user = new User({ chatId: chatId.toString(), firstName: message.from.first_name });
        await user.save();
    }

    // Кнопки для управления задачами
    const options = {
        reply_markup: {
            keyboard: [
                [{ text: 'Создать новую задачу' }],
                [{ text: 'Показать все задачи' }],
                [{ text: 'Изменить статус задачи' }],
                [{ text: 'Удалить задачу' }],
                [{ text: 'Изменить текст задачи' }],
                [{ text: 'Установить время напоминания' }],
                [{ text: 'Изменить дату напоминания' }],
            ],
            resize_keyboard: true,
            one_time_keyboard: true,
        },
    };

    // Приветственное сообщение
    if (message.text === '/start') {
        bot.sendMessage(chatId, 'Добро пожаловать в Task Manager Bot! Выберите действие:', options);
    }

    // Команда для создания новой задачи
    else if (message.text === 'Создать новую задачу') {
        bot.sendMessage(chatId, 'Пожалуйста, введите текст задачи:');
        bot.once('message', async (msg) => {
            const taskText = msg.text; // Получаем текст задачи
            lastTaskId += 1; // Увеличиваем ID задачи
            const newTask = new Task({ userId: user._id, taskId: lastTaskId, text: taskText }); // Создаем новую задачу
            await newTask.save(); // Сохраняем задачу в базе данных
            bot.sendMessage(chatId, `Задача "${taskText}" создана с ID ${lastTaskId}!`, options); // Подтверждаем создание задачи
        });
    }

    // Команда для показа всех задач
    else if (message.text === 'Показать все задачи') {
        const tasks = await Task.find({ userId: user._id }); // Получаем все задачи пользователя
        const response = tasks.map(task => `ID: ${task.taskId}, Задача: ${task.text}, Статус: ${task.status}`).join('\n') || 'Нет задач.'; // Формируем ответ
        bot.sendMessage(chatId, response, options); // Отправляем список задач
    }

    // Команда для изменения статуса задачи
    else if (message.text === 'Изменить статус задачи') {
        bot.sendMessage(chatId, 'Пожалуйста, введите ID задачи для изменения статуса:');
        bot.once('message', async (msg) => {
            const taskId = parseInt(msg.text); // Получаем ID задачи
            const task = await Task.findOne({ taskId: taskId }); // Ищем задачу по ID
            if (task) {
                task.status = task.status === 'актуальна' ? 'выполнена' : 'актуальна'; // Меняем статус на противоположный
                await task.save(); // Сохраняем изменения
                bot.sendMessage(chatId, `Статус задачи с ID ${taskId} изменен на "${task.status}".`, options); // Подтверждаем изменение статуса
            } else {
                bot.sendMessage(chatId, `Задача с ID ${taskId} не найдена.`, options); // Сообщаем, если задача не найдена
            }
        });
    }

    // Команда для удаления задачи
    else if (message.text === 'Удалить задачу') {
        bot.sendMessage(chatId, 'Пожалуйста, введите ID задачи для удаления:');
        bot.once('message', async (msg) => {
            const taskId = parseInt(msg.text); // Получаем ID задачи
            await Task.findOneAndDelete({ taskId: taskId }); // Удаляем задачу
            bot.sendMessage(chatId, `Задача с ID ${taskId} удалена.`, options); // Подтверждаем удаление задачи
        });
    }

    // Команда для изменения текста задачи
    else if (message.text === 'Изменить текст задачи') {
        bot.sendMessage(chatId, 'Пожалуйста, введите ID задачи и новый текст, разделенные пробелом:');
        bot.once('message', async (msg) => {
            const [taskId, ...newText] = msg.text.split(' '); // Получаем ID задачи и новый текст
            await Task.findOneAndUpdate({ taskId: parseInt(taskId) }, { text: newText.join(' ') }); // Обновляем текст задачи
            bot.sendMessage(chatId, `Текст задачи с ID ${taskId} обновлен.`, options); // Подтверждаем изменение текста
        });
    }

    // Команда для установки времени напоминания
    else if (message.text === 'Установить время напоминания') {
        bot.sendMessage(chatId, 'Пожалуйста, введите ID задачи и время (в формате ЧЧ:ММ), разделенные пробелом:');
        bot.once('message', async (msg) => {
            const [taskId, time] = msg.text.split(' '); // Получаем ID задачи и время
            const [hours, minutes] = time.split(':'); // Разделяем часы и минуты
            const reminderTime = new Date(); // Получаем текущее время
            reminderTime.setHours(hours, minutes, 0); // Устанавливаем время напоминания
            await Task.findOneAndUpdate({ taskId: parseInt(taskId) }, { reminderTime: reminderTime }); // Обновляем время напоминания
            bot.sendMessage(chatId, `Время напоминания для задачи с ID ${taskId} установлено на ${time}.`, options); // Подтверждаем установку времени
        });
    }

    // Команда для изменения даты напоминания
    else if (message.text === 'Изменить дату напоминания') {
        bot.sendMessage(chatId, 'Пожалуйста, введите ID задачи и дату (в формате ДД.М.ГГ или Д.М.ГГ), разделенные пробелом:');
        bot.once('message', async (msg) => {
            const [taskId, date] = msg.text.split(' '); // Получаем ID задачи и дату
            const reminderDate = new Date(date); // Формируем дату напоминания
            if (isNaN(reminderDate.getTime())) {
                bot.sendMessage(chatId, 'Неверный формат даты. Пожалуйста, попробуйте снова.', options);
                return;
            }
            await Task.findOneAndUpdate({ taskId: parseInt(taskId) }, { reminderTime: reminderDate }); // Обновляем дату напоминания
            bot.sendMessage(chatId, `Дата напоминания для задачи с ID ${taskId} обновлена.`, options); // Подтверждаем изменение даты
        });
    }

    res.sendStatus(200); // Отправляем статус 200 (OK)
});

// Запускаем приложение
const startApp = async () => {
    await setWebhook(); // Устанавливаем вебхук
    app.listen(PORT, () => {
        console.log(`Бот запущен на http://localhost:${PORT}`); // Выводим сообщение о запуске сервера
    });
};

startApp(); // Запускаем приложение
```

## 7. Создание приложения для уведомлений

### Файл: `notificationApp.js`

```javascript
const TelegramBot = require('node-telegram-bot-api'); // Подключаем библиотеку для работы с Telegram API
const cron = require('node-cron'); // Подключаем библиотеку для планирования задач
const connectDB = require('./db'); // Подключаем функцию подключения к MongoDB
const User = require('./User'); // Подключаем модель пользователя
const Task = require('./Task'); // Подключаем модель задачи

const TELEGRAM_TOKEN = 'token'; // Заменяем на наш токен

// Подключаемся к MongoDB
const startApp = async () => {
    await connectDB(); // Вызываем функцию подключения к базе данных

    // Создаем бота
    const bot = new TelegramBot(TELEGRAM_TOKEN); // Создаем экземпляр бота с токеном

    // Функция для отправки уведомлений о задачах
    const sendTaskReminders = async () => {
        try {
            const tasks = await Task.find({ reminderTime: { $ne: null } }); // Получаем все задачи с установленным временем напоминания
            const currentTime = new Date(); // Получаем текущее время
            for (const task of tasks) { // Перебираем все задачи
                if (task.reminderTime <= currentTime) { // Проверяем, наступило ли время напоминания
                    const user = await User.findById(task.userId); // Получаем пользователя по ID
                    await bot.sendMessage(user.chatId, `Напоминание: ${task.text}`); // Отправляем уведомление
                    console.log(`Напоминание отправлено пользователю: ${user.chatId}`); // Логируем отправку уведомления
                }
            }
        } catch (error) {
            console.error('Ошибка при отправке уведомлений:', error.message); // Обрабатываем ошибки отправки
        }
    };

    // Запланировать выполнение функции каждую минуту
    cron.schedule('* * * * *', () => {
        console.log('Проверка времени напоминаний...'); // Логируем начало проверки
        sendTaskReminders(); // Вызываем функцию отправки уведомлений
    });

    console.log('Приложение уведомлений запущено и будет проверять время напоминаний каждую минуту.'); // Выводим сообщение о запуске приложения
};

// Запускаем приложение
startApp(); // Вызываем функцию для запуска приложения
```

## 8. Установка и настройка Nginx

### Установка Nginx

Если Nginx еще не установлен, устанавливаем его с помощью следующей команды:

```bash
sudo apt update
sudo apt install nginx
```

### Настройка конфигурации Nginx

Создаем новый файл конфигурации для нашего сайта. Например, создайте файл `/etc/nginx/sites-available/task-manager-bot`:

```bash
sudo nano /etc/nginx/sites-available/task-manager-bot
```

Добавляем следующую конфигурацию в файл:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name your-domain.com; # Замените на ваш домен
    return 301 https://$server_name$request_uri;

    server_tokens off;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name your-domain.com; # Замените на ваш домен
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    location /webhook {
          proxy_pass http://127.0.0.1:3030; # Проксируем запросы к вашему приложению
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $remote_addr;
          proxy_set_header HTTPS $https;
          proxy_set_header Scheme $scheme;
        }
}
```

### Включение конфигурации

Создаем символическую ссылку на файл конфигурации в директории `sites-enabled`:

```bash
sudo ln -s /etc/nginx/sites-available/task-manager-bot /etc/nginx/sites-enabled/
```

### Проверка конфигурации Nginx

Проверяем, правильно ли настроен Nginx:

```bash
sudo nginx -t
```

Если конфигурация правильная, мы увидим сообщение `syntax is ok` и `test is successful`.

### Перезапуск Nginx

Перезапускаем Nginx, чтобы применить изменения:

```bash
sudo systemctl restart nginx
```

## 9. Запуск приложений

1. Запускаем первое приложение (Telegram бот):

```bash
node botApp.js
```

2. В другом терминале запускаем второе приложение (уведомления):

```bash
node notificationApp.js
```

## Заключение

Теперь у нас есть Telegram бот, который позволяет пользователям создавать и управлять задачами, устанавливать напоминания и получать уведомления. Также настроен Nginx для проксирования запросов с HTTPS на ваше приложение, запущенное на `http://localhost:3030`.

## Структура REST API

### 1. Основные компоненты

- **Модель пользователя**: Мы создали модель `User`, которая хранит информацию о пользователях, включая их `chatId`, `firstName`, `lastName`, `username`, `createdAt`.
  
- **Модель задачи**: Мы создали модель `Task`, которая хранит информацию о задачах, включая `userId`, `taskId`, `text`, `status`, `reminderTime` и `createdAt`.

- **Вебхук**: Бот использует вебхук для получения обновлений от Telegram. Все входящие сообщения обрабатываются на конечной точке `/webhook`.

### 2. Конечные точки (Endpoints)

#### 2.1. `/webhook` (POST)

- **Описание**: Эта конечная точка обрабатывает входящие сообщения от Telegram.
- **Функциональность**:
  - Команда `/start`: Приветственное сообщение и кнопки для управления задачами.
  - Кнопка **Создать новую задачу**: Позволяет пользователю ввести текст задачи.
  - Кнопка **Показать все задачи**: Отображает все задачи пользователя.
  - Кнопка **Изменить статус задачи**: Позволяет пользователю ввести ID задачи, и статус меняется на противоположный.
  - Кнопка **Удалить задачу**: Позволяет пользователю ввести ID задачи для удаления.
  - Кнопка **Изменить текст задачи**: Позволяет пользователю ввести ID задачи и новый текст.
  - Кнопка **Установить время напоминания**: Позволяет пользователю ввести ID задачи и время (в формате ЧЧ:ММ).
  - Кнопка **Изменить дату напоминания**: Позволяет пользователю ввести ID задачи и дату (в формате ДД.М.ГГ или Д.М.ГГ).

### 3. Уведомления

- **Функция отправки уведомлений**: Каждую минуту приложение проверяет текущее время и сравнивает его с временем напоминания, сохраненным в базе данных. Если время совпадает, пользователю отправляется уведомление.

## Выводы

Таким образом, мы получили простое REST API, которое позволяет взаимодействовать с пользователями Telegram бота. API обрабатывает входящие сообщения, сохраняет данные пользователей и задач в базе данных и отправляет уведомления в соответствии с заданным временем напоминания.
