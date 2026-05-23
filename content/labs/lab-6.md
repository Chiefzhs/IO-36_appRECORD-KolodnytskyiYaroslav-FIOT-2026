## 1. Тема, мета, посилання

### 1.1 Тема
«Документування API за допомогою Swagger. Деплой Node.js-додатку. Підсумковий проєкт: REST API з MySQL».

### 1.2 Мета
Інтегрувати в проєкт `EduVault` документацію `Swagger/OpenAPI`, підготувати API до тестування через `Swagger UI`, а також оформити production-орієнтовану конфігурацію запуску та подальшого розгортання Node.js-додатку.

### 1.3 Посилання
- Репозиторій власного проєкту (GitHub): [Chiefzhs/EduVault](https://github.com/Chiefzhs/EduVault)
- Розгорнутий застосунок: локальний запуск із репозиторію проєкту.
- Репозиторій звітного HTML-документа (GitHub): [Chiefzhs/IO-36_appRECORD-KolodnytskyiYaroslav-FIOT-2026](https://github.com/Chiefzhs/IO-36_appRECORD-KolodnytskyiYaroslav-FIOT-2026)
- Звітний HTML-документ: [chiefzhs.github.io/IO-36_appRECORD-KolodnytskyiYaroslav-FIOT-2026](https://chiefzhs.github.io/IO-36_appRECORD-KolodnytskyiYaroslav-FIOT-2026/)

---

## 2. Хід виконання роботи

### 2.1 Загальна логіка завершального етапу
На шостому етапі розвитку сервісу `EduVault` основну увагу було зосереджено на двох практичних завданнях:
- документуванні реалізованого API;
- підготовці застосунку до подальшого production-запуску.

До цього моменту сервіс уже містив:
- маршрути CRUD для роботи з матеріалами та категоріями;
- авторизацію та автентифікацію на основі `JWT`;
- логування;
- завантаження файлів;
- базові засоби безпеки;
- кешування та тестування.

Тому шоста лабораторна робота стала етапом, на якому API було приведено до більш завершеного вигляду: воно стало не лише робочим, а й задокументованим та придатним для подальшого розгортання.

### 2.2 Інтеграція Swagger UI
Для документування REST API у проєкті було використано:
- `swagger-jsdoc`
- `swagger-ui-express`

Ці бібліотеки дозволили:
- описати API у форматі `OpenAPI`;
- автоматично згенерувати специфікацію;
- відобразити її у вигляді інтерактивного інтерфейсу `Swagger UI`.

Після встановлення залежностей було створено конфігураційний файл `config/swagger.js`, у якому визначено базову інформацію про сервіс.

Фрагмент конфігурації:

```js
const definition = {
  openapi: "3.0.0",
  info: {
    title: "EduVault API",
    version: "1.0.0",
    description: "API documentation for EduVault"
  },
  servers: [
    {
      url: "http://localhost:3000",
      description: "Local server"
    }
  ]
};
```

У цьому блоці задаються:
- стандарт `OpenAPI`;
- назва API;
- версія документації;
- короткий опис;
- адреса локального сервера для тестування.

### 2.3 Опис схем даних і механізму доступу
Окремий блок специфікації було присвячено опису структур запитів і механізму захищеного доступу. Для цього в секції `components` були визначені:
- `securitySchemes`;
- схеми вхідних даних;
- моделі для авторизації та окремих сутностей.

Фрагмент конфігурації:

```js
components: {
  securitySchemes: {
    bearerAuth: {
      type: "http",
      scheme: "bearer",
      bearerFormat: "JWT"
    }
  },
  schemas: {
    LoginRequest: {
      type: "object",
      required: ["email", "password"],
      properties: {
        email: { type: "string", example: "olena@eduvault.app" },
        password: { type: "string", example: "password123" }
      }
    }
  }
}
```

Завдяки цьому в `Swagger UI` можна побачити:
- які маршрути вимагають `Bearer`-токен;
- які саме поля очікуються в тілі запиту;
- які типи даних використовуються в API.

### 2.4 Документування основних endpoint-ів
У специфікації було описано ключові маршрути сервісу:
- `GET /api/health`
- `GET /api/status`
- `POST /api/auth/login`
- `GET /api/categories`
- `POST /api/categories`
- `GET /api/materials`
- `POST /api/materials`
- `POST /api/uploads/materials`

Приклад опису маршруту авторизації:

```js
"/api/auth/login": {
  post: {
    summary: "Authenticate user",
    requestBody: {
      required: true,
      content: {
        "application/json": {
          schema: {
            $ref: "#/components/schemas/LoginRequest"
          }
        }
      }
    },
    responses: {
      200: {
        description: "Login successful"
      },
      400: {
        description: "Validation failed"
      }
    }
  }
}
```

Приклад опису маршруту завантаження файлу:

```js
"/api/uploads/materials": {
  post: {
    summary: "Upload material file",
    security: [{ bearerAuth: [] }],
    requestBody: {
      required: true,
      content: {
        "multipart/form-data": {
          schema: {
            type: "object",
            properties: {
              file: {
                type: "string",
                format: "binary"
              }
            }
          }
        }
      }
    }
  }
}
```

Таким чином, документація охоплює як стандартні `JSON`-маршрути, так і сценарій завантаження файлів.

### 2.5 Підключення Swagger UI в Express-застосунку
Після формування специфікації вона була підключена до серверного застосунку через окремий маршрут `/api-docs`.

Фрагмент інтеграції:

```js
const swaggerUi = require("swagger-ui-express");
const swaggerSpec = require("./config/swagger");

app.use("/api-docs", swaggerUi.serve, swaggerUi.setup(swaggerSpec));
```

Після запуску сервера документація стає доступною за адресою:

```text
http://localhost:3000/api-docs/
```

У результаті користувач може:
- переглядати список маршрутів;
- відкривати опис окремих endpoint-ів;
- бачити схеми запитів і відповідей;
- виконувати тестові запити без окремого клієнта.

### 2.6 Підготовка до розгортання
Після документування API було виконано базову підготовку до подальшого deployment-процесу. Для цього в проєкті створено `Dockerfile`, який описує стандартизований запуск сервісу у контейнеризованому середовищі.

Фрагмент `Dockerfile`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile --prod

COPY . .

ENV NODE_ENV=production
EXPOSE 3000

CMD ["pnpm", "start"]
```

Цей файл визначає:
- базовий образ;
- робочу директорію;
- встановлення production-залежностей;
- порт застосунку;
- команду старту.

Також було додано `.dockerignore`, щоб під час складання контейнера не включати зайві локальні артефакти:

```text
node_modules
logs
uploads
.env
.git
.DS_Store
coverage
```

### 2.7 Практичний результат
У підсумку проєкт `EduVault` отримав:
- інтегровану `Swagger/OpenAPI`-документацію;
- окрему сторінку `Swagger UI`;
- опис основних маршрутів і схем запитів;
- підтримку документування upload-сценарію;
- production-орієнтований `Dockerfile`;
- базову готовність до подальшого розгортання.

Отже, сервіс набув завершенішого вигляду як backend-рішення з документацією, стандартизованим запуском і підготовкою до публічного розміщення.

## 3. Результати виконання

### 3.1 Сторінка Swagger UI
![Swagger UI](/assets/labs/lab-6/screen-1.png)

### 3.2 Виконання запиту через Swagger
![Swagger request execution](/assets/labs/lab-6/screen-2.png)

## 4. Висновки
Під час виконання шостої лабораторної роботи в проєкті `EduVault` було реалізовано інтерактивне документування API за допомогою `Swagger`, а також підготовлено серверну частину до подальшого розгортання. У результаті API стало зручнішим для перевірки, підтримки та подальшого використання, а структура запуску сервісу була приведена до більш стандартизованого вигляду.
