# Переход на NestJS для Telegram Bot с поддержкой вебхука

В этом проекте мы создадим приложение Telegram Bot на NestJS с поддержкой вебхука, используя максимально простую структуру файлов. 
Мы перенесем всю логику из нашего файла `botApp.js` и адаптируем ее для работы с NestJS.

## Шаг 1: Установка NestJS

1. **Установим NestJS CLI**:
   ```bash
   npm install -g @nestjs/cli
   ```

2. **Создаем новый проект NestJS**:
   ```bash
   nest new telegram-bot
   ```

3. **Перейддим в директорию проекта**:
   ```bash
   cd telegram-bot
   ```

4. **Устанавливаем необходимые библиотеки**:
   ```bash
   npm install node-telegram-bot-api mongoose body-parser
   npm install --save-dev @types/node @types/express @types/node-telegram-bot-api @types/body-parser
   ```

## Шаг 2: Создание структуры проекта

Создаем следующие файлы:

```
src
├── app.module.ts
├── main.ts
├── bot
│   ├── bot.module.ts
│   ├── bot.service.ts
│   ├── bot.controller.ts
│   ├── user.schema.ts
│   └── task.schema.ts
└── db
    └── db.module.ts
```




### Описание структуры и назначения файлов

1. **`app.module.ts`**:
   - **Описание**: Это основной модуль приложения, который объединяет все другие модули.
   - **Задача**: Импортирует модули базы данных (`DbModule`) и бота (`BotModule`), обеспечивая их доступность в приложении.
   - **Связь**: Связан с `DbModule` и `BotModule`, которые содержат логику работы с базой данных и функциональность бота соответственно.

2. **`main.ts`**:
   - **Описание**: Это точка входа в приложение.
   - **Задача**: Создает экземпляр приложения Nest и запускает его на указанном порту.
   - **Связь**: Связан с `AppModule`, так как именно этот модуль загружается и инициализируется в `main.ts`.

3. **`bot`**: Директория, содержащая все компоненты, связанные с функциональностью Telegram бота.
   - **`bot.module.ts`**:
     - **Описание**: Модуль, который группирует все компоненты, связанные с ботом.
     - **Задача**: Импортирует необходимые модули, такие как Mongoose для работы с базой данных, и предоставляет сервисы и контроллеры бота.
     - **Связь**: Связан с `BotService` и `BotController`, которые реализуют логику обработки сообщений и команд от пользователей.

   - **`bot.service.ts`**:
     - **Описание**: Сервис, который содержит бизнес-логику бота.
     - **Задача**: Обрабатывает входящие обновления от Telegram, управляет задачами и пользователями, а также взаимодействует с API Telegram.
     - **Связь**: Связан с `User` и `Task`, так как он использует модели для работы с данными пользователей и задач.

   - **`bot.controller.ts`**:
     - **Описание**: Контроллер, который обрабатывает HTTP-запросы.
     - **Задача**: Принимает обновления от Telegram через вебхук и передает их в сервис для обработки.
     - **Связь**: Связан с `BotService`, так как он вызывает методы сервиса для обработки обновлений.

   - **`user.schema.ts`**:
     - **Описание**: Схема для модели пользователя.
     - **Задача**: Определяет структуру данных пользователя, включая поля, такие как `chatId`, `firstName`, `last_command` и `last_task_id`.
     - **Связь**: Связан с `BotService`, который использует эту модель для работы с данными пользователей.

   - **`task.schema.ts`**:
     - **Описание**: Схема для модели задачи.
     - **Задача**: Определяет структуру данных задачи, включая поля, такие как `userId`, `text`, `status` и `date`.
     - **Связь**: Связан с `BotService`, который использует эту модель для работы с данными задач.

4. **`db`**: Директория, содержащая компоненты, связанные с базой данных.
   - **`db.module.ts`**:
     - **Описание**: Модуль, который управляет подключением к базе данных.
     - **Задача**: Настраивает подключение к MongoDB с использованием Mongoose и предоставляет доступ к базе данных для других модулей.
     - **Связь**: Связан с `AppModule`, так как он импортируется в основной модуль приложения.



Эта структура проекта позволяет организовать код в логические модули, что упрощает его поддержку и расширение. Каждый файл и модуль имеют четко определенные задачи и связи, что способствует лучшей архитектуре приложения.


## Шаг 3: Настройка базы данных

1. **Создаем файл `db/db.module.ts`**:
```typescript
import { Module, OnModuleInit } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
   imports: [
      MongooseModule.forRoot('mongodb://mongo-user-mygeobot:password@195.80.51.100:27017/mygeobot'),
   ],
})
export class DbModule implements OnModuleInit {
   onModuleInit() {
      console.log('Подключение к базе данных успешно установлено.');
   }
}
```

## Шаг 4: Создание модуля бота

1. **Создаем файл `bot/bot.module.ts`**:
```typescript
import { Module } from '@nestjs/common';
import { BotService } from './bot.service';
import { BotController } from './bot.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { User, UserSchema } from './user.schema';
import { Task, TaskSchema } from './task.schema';

@Module({
   imports: [
      MongooseModule.forFeature([{ name: User.name, schema: UserSchema }]),
      MongooseModule.forFeature([{ name: Task.name, schema: TaskSchema }]),
   ],
   providers: [BotService],
   controllers: [BotController],
})
export class BotModule {}
```

2. **Создаем файл `bot/bot.service.ts`**:
```typescript


import { Injectable } from '@nestjs/common';
import TelegramBot from 'node-telegram-bot-api';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { User, UserDocument } from './user.schema'; //  импортируеm UserDocument
import { Task } from './task.schema';

@Injectable()
export class BotService {
   private bot: TelegramBot;
   private readonly WEBHOOK_URL = 'https://tc-app.ru/webhook'; // Замените на ваш URL
   private readonly TELEGRAM_TOKEN = 'token'; // Замените на ваш токен

   constructor(
           @InjectModel(User.name) private userModel: Model<UserDocument>, // Используйте UserDocument
           @InjectModel(Task.name) private taskModel: Model<Task>,
   ) {
      this.bot = new TelegramBot(this.TELEGRAM_TOKEN);
      this.setWebhook(); // Устанавливаем вебхук при инициализации
   }

   private async setWebhook() {
      try {
         await this.bot.setWebHook(this.WEBHOOK_URL);
         console.log('Webhook установлен успешно');
      } catch (error) {
         console.error('Ошибка установки вебхука:', error.message);
         process.exit(1);
      }
   }

   // Метод для обработки входящих обновлений
   async handleUpdate(update: any) {
      if (update.message) {
         const chatId = update.message.chat.id;
         const text = update.message.text;

         // Получаем пользователя
         const user: UserDocument = await this.userModel.findOne({ chatId: chatId.toString() });

         // Проверяем, ожидает ли бот текст для новой задачи или изменения
         if (user.last_command === '/new_task') {
            await this.createNewTask(chatId, text, user);
            return; // Завершаем обработку, чтобы не обрабатывать это сообщение как обычное
         } else if (user.last_command === '/edit_task') {
            await this.editTaskText(chatId, text, user);
            return;
         } else if (user.last_command === '/change_date') {
            await this.changeTaskDateInput(chatId, text, user);
            return;
         } else if (user.last_command === '/change_time') {
            await this.changeTaskTimeInput(chatId, text, user);
            return;
         }

         // Обработка команд
         switch (text) {
            case '/start':
               await this.handleStartCommand(chatId, update.message.from);
               break;
            case 'Создать новую задачу':
               await this.handleCreateTask(chatId);
               await this.showMainMenu(chatId); // Отправляем нижнюю клавиатуру
               break;
            case 'Все задачи':
               await this.handleAllTasks(chatId);
               await this.showMainMenu(chatId); // Отправляем нижнюю клавиатуру
               break;
            case 'Актуальные задачи':
               await this.handleCurrentTasks(chatId);
               await this.showMainMenu(chatId); // Отправляем нижнюю клавиатуру
               break;
            case 'Задачи в работе':
               await this.handleInProgressTasks(chatId);
               await this.showMainMenu(chatId); // Отправляем нижнюю клавиатуру
               break;
            default:
               this.sendMessage(chatId, 'Команда не распознана. Пожалуйста, выберите действие из меню.');
               await this.showMainMenu(chatId); // Отправляем нижнюю клавиатуру
         }
      }

      if (update.callback_query) {
         await this.handleCallbackQuery(update.callback_query);
      }
   }

   private async handleStartCommand(chatId: number, user: any) {
      const existingUser: UserDocument = await this.userModel.findOne({ chatId: chatId.toString() });
      if (!existingUser) {
         await this.userModel.create({ chatId: chatId.toString(), firstName: user.first_name });
      }
      this.showMainMenu(chatId);
   }

   private async handleCreateTask(chatId: number) {
      const user: UserDocument = await this.userModel.findOne({ chatId: chatId.toString() });
      user.last_command = '/new_task';
      await user.save();
      this.sendMessage(chatId, 'Пожалуйста, введите текст задачи:');
   }

   private async createNewTask(chatId: number, text: string, user: UserDocument) {
      // Создаем новую задачу
      const newTask = new this.taskModel({
         userId: user._id,
         text: text,
         status: 'в работе', // Устанавливаем статус по умолчанию
         date: new Date(Date.now() - 60 * 1000), // Устанавливаем текущую дату минус 1 минута
      });

      await newTask.save(); // Сохраняем новую задачу в базе данных
      user.last_command = null; // Сбрасываем last_command
      await user.save(); // Сохраняем изменения пользователя

      this.sendMessage(chatId, `Задача "${text}" успешно создана!`); // Подтверждение создания задачи
   }

   private async editTaskText(chatId: number, text: string, user: UserDocument) {
      const taskId = user.last_task_id; // Получаем ID задачи
      const task = await this.taskModel.findById(taskId);
      if (task) {
         task.text = text; // Обновляем текст задачи
         await task.save(); // Сохраняем изменения
         user.last_command = null; // Сбрасываем last_command
         await user.save(); // Сохраняем изменения пользователя
         this.sendMessage(chatId, `Текст задачи обновлен на: "${text}"`);
      } else {
         this.sendMessage(chatId, `Задача с ID ${taskId} не найдена.`);
      }
   }

   private async changeTaskDateInput(chatId: number, text: string, user: UserDocument) {
      const taskId = user.last_task_id; // Получаем ID задачи
      const task = await this.taskModel.findById(taskId);
      if (task) {
         const newDate = this.parseDate(text); // Парсим дату из текста

         // Проверка на корректность даты
         if (!newDate || isNaN(newDate.getTime())) {
            this.sendMessage(chatId, `Некорректный формат даты. Пожалуйста, введите дату в формате ДД.ММ.ГГГГ.`);
            return; // Завершаем выполнение метода
         }

         // Устанавливаем время на 10:00
         newDate.setHours(10);
         newDate.setMinutes(0);
         newDate.setSeconds(0); // Устанавливаем секунды на 0

         task.date = newDate; // Устанавливаем новую дату
         task.status = 'актуальна'; // Устанавливаем статус актуальна
         await task.save(); // Сохраняем изменения
         user.last_command = null; // Сбрасываем last_command
         await user.save(); // Сохраняем изменения пользователя
         this.sendMessage(chatId, `Дата задачи обновлена на: ${this.formatDate(newDate)} в 10:00.`);
      } else {
         this.sendMessage(chatId, `Задача с ID ${taskId} не найдена.`);
      }
   }

   private async changeTaskTimeInput(chatId: number, text: string, user: UserDocument) {
      const taskId = user.last_task_id; // Получаем ID задачи
      const task = await this.taskModel.findById(taskId);
      if (task) {
         const [hours, minutes] = text.split(':').map(Number);
         // Проверка на корректность введенного времени
         if (isNaN(hours) || isNaN(minutes) || hours < 0 || hours > 23 || minutes < 0 || minutes > 59) {
            this.sendMessage(chatId, 'Ошибка: некорректный формат времени. Пожалуйста, введите время в формате ЧЧ:ММ (например, 14:30).');
            return; // Завершаем выполнение метода
         }
         const date = new Date(task.date);
         date.setHours(hours);
         date.setMinutes(minutes);
         task.date = date; // Устанавливаем новое время
         task.status = 'актуальна'; // Устанавливаем статус актуальна
         await task.save(); // Сохраняем изменения
         user.last_command = null; // Сбрасываем last_command

         await user.save(); // Сохраняем изменения пользователя
         this.sendMessage(chatId, `Время задачи обновлено на: ${this.formatTime(date)}`);
      } else {
         this.sendMessage(chatId, `Задача с ID ${taskId} не найдена.`);
      }
   }

   private parseDate(input: string): Date | null {
      const parts = input.split('.');
      if (parts.length === 3) {
         const day = parseInt(parts[0], 10);
         const month = parseInt(parts[1], 10) - 1; // Месяцы начинаются с 0
         const year = parseInt(parts[2], 10);
         const date = new Date(year, month, day);
         return date instanceof Date && !isNaN(date.getTime()) ? date : null; // Проверка на корректность даты
      }
      return null; // Если формат некорректный
   }

   private async handleAllTasks(chatId: number) {
      const user: UserDocument = await this.userModel.findOne({ chatId: chatId.toString() });
      const tasks = await this.taskModel.find({ userId: user._id }).sort({ date: 1 }); // Сортируем по дате
      const inlineKeyboard = tasks.map(task => [
         {
            text: `${this.formatDate(task.date)} ${this.formatTime(task.date)} ${task.text}`, // Форматируем строку с датой, временем и текстом задачи
            callback_data: `show_task_${task._id}`,
         },
      ]);
      await this.bot.sendMessage(chatId, 'Вот все Ваши задачи:', {
         reply_markup: {
            inline_keyboard: inlineKeyboard,
         },
      });
   }





   private async handleCurrentTasks(chatId: number) {
      const user: UserDocument = await this.userModel.findOne({ chatId: chatId.toString() });
      const tasks = await this.taskModel.find({ userId: user._id, status: 'актуальна' }).sort({ date: 1 }); // Сортируем по дате
      const inlineKeyboard = tasks.map(task => [
         {
            text: `${this.formatDate(task.date)} ${this.formatTime(task.date)} ${task.text}`, // Форматируем строку с датой, временем и текстом задачи
            callback_data: `show_task_${task._id}`,
         },
      ]);
      await this.bot.sendMessage(chatId, 'Вот ваши актуальные задачи:', {
         reply_markup: {
            inline_keyboard: inlineKeyboard,
         },
      });
   }

   private async handleInProgressTasks(chatId: number) {
      const user: UserDocument = await this.userModel.findOne({ chatId: chatId.toString() });
      const tasks = await this.taskModel.find({ userId: user._id, status: 'в работе' }).sort({ date: 1 }); // Сортируем по дате
      const inlineKeyboard = tasks.map(task => [
         {
            text: `${this.formatDate(task.date)} ${this.formatTime(task.date)} ${task.text}`, // Форматируем строку с датой, временем и текстом задачи
            callback_data: `show_task_${task._id}`,
         },
      ]);
      await this.bot.sendMessage(chatId, 'Вот ваши задачи в работе:', {
         reply_markup: {
            inline_keyboard: inlineKeyboard,
         },
      });
   }

   private async handleCallbackQuery(callbackQuery: any) {
      const chatId = callbackQuery.message.chat.id;
      const data = callbackQuery.data;

      console.log('Получен callback_query:', callbackQuery); // Логируем весь объект callback_query
      console.log('Данные callback_query:', data); // Логируем данные callback_query

      if (!data) {
         console.error('Ошибка: данные callback_query отсутствуют.');
         return;
      }

      let user: UserDocument = await this.userModel.findOne({ chatId: chatId.toString() });

      if (data.startsWith('show_task_')) {
         await this.showTask(chatId, data.split('_')[2], user);
      } else if (data.startsWith('edit_task_')) {
         await this.editTask(chatId, data.split('_')[2], user);
      } else if (data.startsWith('change_date_')) {
         await this.changeTaskDate(chatId, data.split('_')[2], user);
      } else if (data.startsWith('change_time_')) {
         await this.changeTaskTime(chatId, data.split('_')[2], user);
      } else if (data.startsWith('delete_task_')) {
         await this.deleteTask(chatId, data.split('_')[2], user);
      } else if (data.startsWith('edit_status_')) {
         await this.editTaskStatus(chatId, data.split('_')[2]);
      }

      this.bot.answerCallbackQuery(callbackQuery.id);
   }

   private async showTask(chatId: number, taskId: string, user: UserDocument) {
      const task = await this.taskModel.findById(taskId);
      if (task) {
         const response = `Задача: ${task.text}\nСтатус: ${task.status}\nДата: ${this.formatDate(task.date)}\nВремя: ${this.formatTime(task.date)}`;
         const inlineKeyboard = [
            [
               { text: '📆', callback_data: `change_date_${taskId}` },
               { text: '🕑', callback_data: `change_time_${taskId}` },
               { text: '☑️', callback_data: `edit_status_${taskId}` },
               { text: '🗑', callback_data: `delete_task_${taskId}` },
               { text: '📝', callback_data: `edit_task_${taskId}` },
            ],
         ];
         await this.bot.sendMessage(chatId, response, {
            reply_markup: {
               inline_keyboard: inlineKeyboard,
            },
         });
         user.last_task_id = task._id.toString(); // Сохраняем ObjectId задачи
         await user.save(); // Сохраняем изменения пользователя
      } else {
         this.sendMessage(chatId, `Задача с ID ${taskId} не найдена.`);
      }
   }

   private async editTask(chatId: number, taskId: string, user: UserDocument) {
      user.last_command = '/edit_task'; // Сохраняем команду редактирования
      user.last_task_id = taskId; // Сохраняем ObjectId задачи
      await user.save();
      this.sendMessage(chatId, 'Пожалуйста, введите новый текст задачи:');
   }

   private async changeTaskDate(chatId: number, taskId: string, user: UserDocument) {
      user.last_command = '/change_date'; // Сохраняем команду изменения даты
      user.last_task_id = taskId; // Сохраняем ID задачи
      await user.save();
      this.sendMessage(chatId, 'Пожалуйста, введите новую дату задачи в формате ДД.ММ.ГГГГ:');
   }

   private async changeTaskTime(chatId: number, taskId: string, user: UserDocument) {
      user.last_command = '/change_time'; // Сохраняем команду изменения времени
      user.last_task_id = taskId; // Сохраняем ID задачи
      await user.save();
      this.sendMessage(chatId, 'Пожалуйста, введите новое время задачи в формате ЧЧ:ММ:');
   }

   private async deleteTask(chatId: number, taskId: string, user: UserDocument) {
      const task = await this.taskModel.findById(taskId);
      if (task) {
         await this.taskModel.findByIdAndDelete(taskId);
         this.sendMessage(chatId, `Задача "${task.text}" удалена.`);

         // Обновляем инлайн-клавиатуру, чтобы удалить кнопку удаленной задачи
         const tasks = await this.taskModel.find({ userId: user._id }).sort({ date: 1 }); // Сортируем по дате
         const inlineKeyboard = tasks.map(task => [
            {
               text: `${this.formatDate(task.date)} ${this.formatTime(task.date)} ${task.text}`, // Форматируем строку с датой, временем и текстом задачи
               callback_data: `show_task_${task._id}`, // Используем внутренний ID MongoDB
            },
         ]);
         await this.bot.sendMessage(chatId, 'Вот ваши задачи:', {
            reply_markup: {
               inline_keyboard: inlineKeyboard,
            },
         });
      } else {
         this.sendMessage(chatId, `Задача с ID ${taskId} не найдена.`);
      }
   }

   private async editTaskStatus(chatId: number, taskId: string) {
      const task = await this.taskModel.findById(taskId);
      if (task) {
         // Меняем статус по кругу: 'в работе' -> 'актуальна' -> 'выполнена' -> 'в работе'
         if (task.status === 'в работе') {
            task.status = 'актуальна';
         } else if (task.status === 'актуальна') {
            task.status = 'выполнена';
         } else {
            task.status = 'в работе';
         }

         await task.save();
         this.sendMessage(chatId, `Статус задачи изменен на: ${task.status}`);
      } else {
         this.sendMessage(chatId, `Задача с ID ${taskId} не найдена.`);
      }
   }

   private async showMainMenu(chatId: number) {
      const options = {
         reply_markup: {
            keyboard: [
               [{ text: 'Создать новую задачу' }],
               [{ text: 'Все задачи' }],
               [{ text: 'Актуальные задачи' }],
               [{ text: 'Задачи в работе' }],
            ],
            resize_keyboard: true,
            one_time_keyboard: true,
         },
      };
      await this.bot.sendMessage(chatId, 'Выберите действие:', options);
   }

   private async sendMessage(chatId: number, text: string) {
      try {
         await this.bot.sendMessage(chatId, text);
      } catch (error) {
         console.error('Ошибка отправки сообщения:', error.message);
      }
   }

   private formatDate(date: Date) {
      if (!date) return 'Не указана'; // Проверка на наличие даты
      const options: Intl.DateTimeFormatOptions = {
         day: '2-digit',
         month: '2-digit',
         year: 'numeric'
      };
      return date.toLocaleDateString('ru-RU', options);
   }

   private formatTime(date: Date) {
      if (!date) return 'Не указано'; // Проверка на наличие времени
      const options: Intl.DateTimeFormatOptions = { hour: '2-digit', minute: '2-digit', hour12: false };
      const timeString = date.toLocaleTimeString([], options);

      // Заменяем "24" на "0"
      return timeString.replace(/^24/, '0');
   }
}
```

3. **Создаем файл `bot/bot.controller.ts`**:
```typescript
import { Controller, Post, Body, Res } from '@nestjs/common';
import { Response } from 'express';
import { BotService } from './bot.service';

@Controller('webhook')
export class BotController {
   constructor(private botService: BotService) {}

   @Post()
   async handleWebhook(@Body() update: any, @Res() res: Response) {
      console.log('Получено обновление:', update); // Логируем полученное обновление
      await this.botService.handleUpdate(update); // Передаем обновление в сервис
      res.sendStatus(200); // Отправляем статус 200 обратно Telegram
   }
}
```

4. **Создаем файл `bot/user.schema.ts`**:
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type UserDocument = User & Document;

@Schema()
export class User {
   @Prop({ required: true, unique: true })
   chatId: string;

   @Prop({ required: true })
   firstName: string;

   @Prop()
   last_command?: string;

   @Prop({ type: String, ref: 'Task' })
   last_task_id?: string;

   @Prop({ default: false })
   keyboardShown?: boolean; // Флаг для отслеживания, была ли показана клавиатура
}

export const UserSchema = SchemaFactory.createForClass(User);
```

5. **Создаем файл `bot/task.schema.ts`**:
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type TaskDocument = Task & Document;

@Schema()
export class Task {
   @Prop({ required: true })
   userId: string;

   @Prop({ required: true })
   text: string;

   @Prop({ enum: ['актуальна', 'выполнена', 'в работе'], default: 'в работе' })
   status: string;

   @Prop()
   date?: Date;

   @Prop()
   reminderTime?: Date;
}

export const TaskSchema = SchemaFactory.createForClass(Task);
```

## Шаг 5: Настройка основного модуля

1. **Изменяем файл `app.module.ts`**:
```typescript
import { Module } from '@nestjs/common';
import { BotModule } from './bot/bot.module';
import { DbModule } from './db/db.module';

@Module({
   imports: [DbModule, BotModule],
})
export class AppModule {}
```

## Шаг 6: Настройка `main.ts`

1. **Изменяем файл `main.ts`**:
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
   const app = await NestFactory.create(AppModule);
   const port = 3030; // Установите порт, который вы хотите использовать
   await app.listen(port);
   console.log(`Приложение запущено на http://localhost:${port}`); // Логирование порта
}
bootstrap();
```

## Шаг 7: Запуск приложения

1. **Запустите приложение**:
   ```bash
   npm run start
   ```

## Заключение

Теперь у нас есть приложение Telegram Bot на NestJS с поддержкой вебхука. Вебхуки позволяют запускать несколько серверных приложений для одного бота что позволяет в будущем перейти на микросервесную архитектуру, а уже сейчас распределить нагрузку на разные серверы под разные задачи!
