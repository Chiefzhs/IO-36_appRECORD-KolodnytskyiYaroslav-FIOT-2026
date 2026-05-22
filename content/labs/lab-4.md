## 1. Тема, мета, посилання

### 1.1 Тема
«Розширені можливості Node.js-додатків: логування, завантаження файлів, моніторинг продуктивності».

### 1.2 Мета
Розширити серверну частину `EduVault` за рахунок логування HTTP-запитів і подій, реалізації завантаження файлів на сервер, а також додати базовий моніторинг продуктивності застосунку.

### 1.3 Посилання
- Репозиторій власного проєкту (GitHub): буде додано після публікації.
- Розгорнутий застосунок: буде додано після розгортання.
- Репозиторій звітного HTML-документа (GitHub): буде додано після публікації.
- Звітний HTML-документ: буде додано після розгортання.

---

## 2. Хід виконання роботи

### 2.1 Загальна ідея розширення застосунку
На цьому етапі `EduVault` було розширено можливостями, які наближають сервер до більш практичного використання. Якщо попередні лабораторні роботи зосереджувались на даних, авторизації та маршрутах, то четверта робота додає інструменти супроводу й експлуатації:
- фіксацію запитів і подій у логах;
- завантаження файлів матеріалів на сервер;
- контроль базових показників роботи процесу.

Для сервісу з каталогом матеріалів це особливо доречно, оскільки користувачі працюють із файлами, а адміністратор повинен мати змогу бачити, що відбувається із сервером і які дії виконуються.

### 2.2 Реалізація логування
Для логування HTTP-запитів було використано `Morgan`, а для файлового логування подій і помилок — `Winston`.

Спочатку створено окремий logger:

```js
const logger = winston.createLogger({
  level: "info",
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({
      filename: path.join(logDirectory, "error.log"),
      level: "error"
    }),
    new winston.transports.File({
      filename: path.join(logDirectory, "combined.log")
    })
  ]
});
```

У такій конфігурації:
- `combined.log` містить усі основні події;
- `error.log` зберігає лише помилки;
- формат логів є структурованим і придатним для подальшого аналізу.

Після цього було підключено `Morgan`, який не просто виводить запити у консоль, а передає їх до `Winston`:

```js
const requestLogger = morgan("combined", {
  stream: {
    write(message) {
      logger.info(message.trim());
    }
  }
});
```

Завдяки цьому всі HTTP-запити автоматично фіксуються, а журнал подій зберігається у файлах у директорії `logs`.

### 2.3 Логування продуктивності запитів
Окремо було реалізовано middleware, яке вимірює час обробки кожного запиту. Це дозволяє бачити, які маршрути відповідають швидко, а які потенційно можуть стати проблемними.

Фрагмент middleware:

```js
function performanceMiddleware(req, res, next) {
  const startTime = process.hrtime.bigint();

  res.on("finish", () => {
    const endTime = process.hrtime.bigint();
    const durationMs = Number(endTime - startTime) / 1_000_000;

    logger.info("request_completed", {
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      durationMs: Number(durationMs.toFixed(2))
    });
  });

  next();
}
```

Після завершення відповіді в лог записується:
- метод запиту;
- шлях;
- статус-код;
- тривалість виконання в мілісекундах.

Це дає простий, але корисний рівень моніторингу без додаткових зовнішніх сервісів.

### 2.4 Завантаження файлів матеріалів
Оскільки `EduVault` працює з навчальними та довідковими файлами, у сервісі було реалізовано окремий маршрут для завантаження матеріалів. Для цього використано `Multer`.

Було створено окрему директорію:
- `uploads/materials`

Фрагмент налаштування `Multer`:

```js
const storage = multer.diskStorage({
  destination(req, file, cb) {
    cb(null, uploadDirectory);
  },
  filename(req, file, cb) {
    const safeName = file.originalname.replace(/[^a-zA-Z0-9._-]/g, "_");
    cb(null, `${Date.now()}-${safeName}`);
  }
});
```

Така конфігурація дозволяє:
- зберігати файли у визначену директорію;
- уникати проблем з іменами файлів;
- автоматично додавати часову мітку до імені.

### 2.5 Валідація типів файлів
Щоб завантаження було контрольованим, було додано перевірку допустимих MIME-типів. Це важливо, оскільки не кожен файл повинен потрапляти до каталогу матеріалів.

Фрагмент перевірки:

```js
const allowedMimeTypes = new Set([
  "application/pdf",
  "application/msword",
  "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  "image/png",
  "image/jpeg"
]);

function fileFilter(req, file, cb) {
  if (!allowedMimeTypes.has(file.mimetype)) {
    return cb(new Error("Unsupported file type"));
  }

  cb(null, true);
}
```

Було також встановлено обмеження на розмір файлу:

```js
limits: {
  fileSize: 5 * 1024 * 1024
}
```

Отже, сервер приймає лише визначені формати та блокує некоректні типи файлів. Це окремо було перевірено під час тестування.

### 2.6 Маршрут завантаження
Для приймання файлів було створено окремий маршрут:

```js
router.post("/materials", authMiddleware, materialUpload.single("file"), uploadMaterialFile);
```

Його особливості:
- доступ до маршруту має лише авторизований користувач;
- використовується middleware `materialUpload.single("file")`;
- після успішного приймання сервіс повертає інформацію про файл.

Фрагмент контролера:

```js
async function uploadMaterialFile(req, res) {
  if (!req.file) {
    return res.status(400).json({ message: "file is required" });
  }

  return res.status(201).json({
    message: "Material file uploaded successfully",
    file: {
      originalName: req.file.originalname,
      fileName: req.file.filename,
      mimeType: req.file.mimetype,
      size: req.file.size,
      path: `/uploads/materials/${path.basename(req.file.filename)}`
    }
  });
}
```

У результаті користувач отримує підтвердження завантаження та базові метадані про файл.

### 2.7 Моніторинг продуктивності процесу
Для спостереження за станом застосунку було додано окремий маршрут `GET /api/status`, який повертає:
- `uptime`;
- використання пам'яті;
- використання CPU;
- часову мітку.

Фрагмент контролера:

```js
async function getStatus(req, res) {
  const memoryUsage = process.memoryUsage();
  const cpuUsage = process.cpuUsage();

  res.json({
    app: "EduVault",
    uptime: Number(process.uptime().toFixed(2)),
    memoryUsage: {
      rss: memoryUsage.rss,
      heapTotal: memoryUsage.heapTotal,
      heapUsed: memoryUsage.heapUsed
    },
    cpuUsage: {
      user: cpuUsage.user,
      system: cpuUsage.system
    },
    timestamp: new Date().toISOString()
  });
}
```

Цей маршрут є корисним для швидкої діагностики стану сервера без необхідності підключати зовнішні інструменти моніторингу.

### 2.8 Практичний результат
У результаті лабораторної роботи серверна частина `EduVault` отримала:
- журналювання HTTP-запитів;
- файлові логи подій і помилок;
- вимірювання часу виконання запитів;
- маршрут моніторингу продуктивності;
- безпечне завантаження файлів;
- перевірку допустимих типів файлів;
- збереження завантажених матеріалів у файловій системі.

Це значно покращує експлуатаційний рівень застосунку й формує базу для наступних робіт, де пріоритетом стане безпека, продуктивність та стабільність.

---

## 3. Скріншоти результату

![Logs and uploaded files](/assets/labs/lab-4/screen-9.png)

![Upload valid file type](/assets/labs/lab-4/screen-8.png)

![Create material](/assets/labs/lab-4/screen-7.png)

![Create category](/assets/labs/lab-4/screen-6.png)

![Login admin](/assets/labs/lab-4/screen-5.png)

![Upload invalid file type](/assets/labs/lab-4/screen-4.png)

![Performance status](/assets/labs/lab-4/screen-3.png)

![Health check](/assets/labs/lab-4/screen-2.png)

![Server start](/assets/labs/lab-4/screen-1.png)

---

## 4. Висновки

У межах четвертої лабораторної роботи для `EduVault` було реалізовано логування HTTP-запитів і подій, файлове логування помилок, вимірювання часу виконання запитів, моніторинг стану процесу та завантаження файлів матеріалів на сервер. Додатково було впроваджено перевірку типів і розміру файлів, що зробило механізм завантаження контрольованим і безпечнішим. Отриманий результат переводить проєкт на новий рівень практичної готовності, де важливими є не лише маршрути й дані, а й спостереження за роботою сервера.
