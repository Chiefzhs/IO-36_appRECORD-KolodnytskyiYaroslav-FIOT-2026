## 1. Тема, мета, посилання

### 1.1 Тема
«Безпека та продуктивність серверних додатків. Безпека Node.js-додатків. Оптимізація запитів і кешування. Тестування API».

### 1.2 Мета
Підвищити безпеку та продуктивність серверної частини `EduVault`, реалізувати захист API, додати кешування відповідей, виконати валідацію даних та перевірити роботу сервісу за допомогою автоматизованих тестів.

### 1.3 Посилання
- Репозиторій власного проєкту (GitHub): буде додано після публікації.
- Розгорнутий застосунок: буде додано після розгортання.
- Репозиторій звітного HTML-документа (GitHub): буде додано після публікації.
- Звітний HTML-документ: буде додано після розгортання.

---

## 2. Хід виконання роботи

### 2.1 Загальний напрям удосконалення
П'ята лабораторна робота була присвячена не створенню нових базових маршрутів, а покращенню вже існуючого сервісу `EduVault` з точки зору безпеки, продуктивності та якості супроводу. Після реалізації CRUD, авторизації, логування та завантаження файлів наступним логічним кроком стало:
- захистити API від надмірної активності та небажаних запитів;
- перевіряти коректність вхідних даних на рівні маршрутів;
- зменшити повторне навантаження на сервер і базу даних;
- додати автоматизовану перевірку частини API.

У межах роботи було реалізовано:
- `Helmet` для захисних HTTP-заголовків;
- `express-rate-limit` для обмеження кількості запитів;
- `express-validator` для перевірки даних;
- `compression` для стискання відповідей;
- `Redis` для кешування відповідей;
- `Jest + Supertest` для тестування API.

### 2.2 Захист HTTP-заголовків та обмеження запитів
Першим кроком було додавання `Helmet`, який автоматично виставляє набір безпечних HTTP-заголовків. Це дозволяє зменшити базові ризики, пов'язані з небажаним вбудовуванням контенту, sniffing-атакою на MIME-типи та частиною XSS-сценаріїв.

Разом із цим було додано `rate limiting`, щоб не допустити надмірної кількості звернень до сервера за короткий проміжок часу.

Фрагмент middleware безпеки:

```js
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 40,
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    message: "Too many requests, please try again later"
  }
});
```

У такій конфігурації:
- вікно становить 1 хвилину;
- один клієнт може виконати до 40 запитів;
- у відповідь сервер додає стандартні `RateLimit`-заголовки.

### 2.3 Стиснення відповідей
Для покращення швидкодії було підключено `compression`. Це не змінює логіку маршруту, але зменшує обсяг переданих даних, особливо для JSON-відповідей або текстового контенту.

Фрагмент підключення:

```js
const {
  helmetMiddleware,
  compressionMiddleware,
  limiter
} = require("./middleware/securityMiddleware");

app.use(helmetMiddleware);
app.use(compressionMiddleware);
app.use(limiter);
```

Тобто безпека й оптимізація були винесені в окремий рівень middleware ще до підключення маршрутів.

### 2.4 Валідація вхідних даних
Щоб сервер не працював із некоректними або неповними даними, для ключових маршрутів було додано валідацію через `express-validator`.

Наприклад, для авторизації:

```js
const loginValidator = [
  body("email").isEmail().withMessage("email must be valid"),
  body("password").notEmpty().withMessage("password is required")
];
```

Для створення матеріалів:

```js
const createMaterialValidator = [
  body("title").trim().isLength({ min: 3 }).withMessage("title must be at least 3 characters long"),
  body("format").trim().notEmpty().withMessage("format is required"),
  body("userId").isInt({ min: 1 }).withMessage("userId must be a positive integer"),
  body("categoryId").isInt({ min: 1 }).withMessage("categoryId must be a positive integer")
];
```

Після цього було додано універсальне middleware для обробки результатів валідації:

```js
function validationMiddleware(req, res, next) {
  const errors = validationResult(req);

  if (errors.isEmpty()) {
    return next();
  }

  return res.status(400).json({
    message: "Validation failed",
    errors: errors.array()
  });
}
```

Завдяки цьому сервер повертає зрозумілу помилку `400`, якщо клієнт надсилає некоректний запит.

### 2.5 Реалізація Redis-кешування
Ключовим елементом роботи стало кешування списку матеріалів через `Redis`. Ідея полягає в тому, що повторний запит до одного й того ж ресурсу не повинен кожного разу повторно звертатися до бази даних.

Було реалізовано окремий модуль роботи з кешем:

```js
const { createClient } = require("redis");

client = createClient({
  url: process.env.REDIS_URL || "redis://127.0.0.1:6379"
});
```

Кешування маршруту `GET /api/materials` працює так:
- спочатку сервер перевіряє, чи є відповідь у Redis;
- якщо так, повертає дані з позначкою `source: "cache"`;
- якщо ні, виконує запит до БД, зберігає результат у Redis та повертає `source: "database"`.

Фрагмент логіки:

```js
const cachedMaterials = await getCache(MATERIALS_CACHE_KEY);

if (cachedMaterials) {
  return res.json({
    source: "cache",
    data: cachedMaterials
  });
}
```

Після отримання даних із БД вони зберігаються в кеш:

```js
await setCache(MATERIALS_CACHE_KEY, materials, 120);
res.json({
  source: "database",
  data: materials
});
```

### 2.6 Інвалідизація кешу після змін
Щоб кеш не повертав застарілі дані, його потрібно очищати щоразу після зміни списку матеріалів. Саме тому при створенні, оновленні або видаленні матеріалу, а також після створення категорії виконується видалення ключа кешу.

Фрагмент:

```js
await deleteCache(MATERIALS_CACHE_KEY);
```

Це гарантує, що наступний `GET /api/materials` знову звернеться до БД і збереже вже актуальний результат.

### 2.7 Автоматизоване тестування API
Щоб перевірити базову працездатність сервісу після додавання шару безпеки та кешу, було реалізовано автоматизовані тести через `Jest` і `Supertest`.

Фрагмент тестів:

```js
test("GET /api/health should return 200", async () => {
  const response = await request(app).get("/api/health");

  expect(response.statusCode).toBe(200);
  expect(response.body.status).toBe("ok");
});
```

Також перевіряється маршрут моніторингу:

```js
test("GET /api/status should return performance payload", async () => {
  const response = await request(app).get("/api/status");

  expect(response.statusCode).toBe(200);
  expect(response.body.app).toBe("EduVault");
  expect(response.body.memoryUsage).toBeDefined();
});
```

І тест валідації:

```js
test("POST /api/auth/login should validate email", async () => {
  const response = await request(app)
    .post("/api/auth/login")
    .send({
      email: "wrong-format",
      password: "123456"
    });

  expect(response.statusCode).toBe(400);
  expect(response.body.message).toBe("Validation failed");
});
```

Таким чином, було підтверджено, що базові маршрути працюють стабільно навіть після розширення архітектури.

### 2.8 Практичний результат
Після виконання лабораторної роботи `EduVault` отримав:
- захисні HTTP-заголовки;
- обмеження частоти звернень;
- перевірку вхідних даних;
- стискання відповідей;
- кешування результатів через `Redis`;
- очищення кешу після змін;
- автоматизовані API-тести.

У сукупності це робить застосунок більш придатним до реальної експлуатації, оскільки він не лише виконує функціональні задачі, а й краще захищений та оптимізований.

---

## 3. Скріншоти результату

![Create material after validation and cache reset](/assets/labs/lab-5/screen-10.png)

![Create material validation error](/assets/labs/lab-5/screen-9.png)

![Get materials from cache](/assets/labs/lab-5/screen-8.png)

![Get materials from database](/assets/labs/lab-5/screen-7.png)

![Login validation error](/assets/labs/lab-5/screen-6.png)

![Login admin](/assets/labs/lab-5/screen-5.png)

![Performance status](/assets/labs/lab-5/screen-4.png)

![Health check](/assets/labs/lab-5/screen-3.png)

![Docker compose with MySQL and Redis](/assets/labs/lab-5/screen-2.png)

![Jest and Supertest execution](/assets/labs/lab-5/screen-1.png)

---

## 4. Висновки

У межах п'ятої лабораторної роботи для `EduVault` було реалізовано базовий захист API, валідацію даних, оптимізацію відповідей та кешування через `Redis`. Додатково було впроваджено автоматизоване тестування, що дозволило перевірити стабільність роботи після внесених змін. Отриманий результат демонструє перехід від просто функціонального серверного застосунку до більш безпечного, продуктивного та контрольованого backend-рішення.
