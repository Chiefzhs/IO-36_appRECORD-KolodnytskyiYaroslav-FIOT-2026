## 1. Тема, мета, посилання

### 1.1 Тема
«Створення бази даних у MySQL. Підключення Node.js до MySQL. Робота з ORM Sequelize».

### 1.2 Мета
Розробити базу даних для сервісу `EduVault`, налаштувати підключення `Node.js` до `MySQL`, реалізувати структуру таблиць і зв'язки між ними, а також показати роботу як із сирими SQL-запитами, так і через ORM `Sequelize`.

### 1.3 Посилання
- Репозиторій власного проєкту (GitHub): буде додано після публікації.
- Розгорнутий застосунок: буде додано після розгортання.
- Репозиторій звітного HTML-документа (GitHub): буде додано після публікації.
- Звітний HTML-документ: буде додано після розгортання.

---

## 2. Хід виконання роботи

### 2.1 Загальна структура даних
Для проєкту `EduVault` було створено окрему базу даних `eduvault_db`, у якій реалізовано три базові таблиці:
- `users`
- `categories`
- `materials`

Побудована структура відображає логіку сервісу зберігання матеріалів. Кожен матеріал має автора та належить до певної категорії. Завдяки цьому можна організувати записи не як хаотичний набір файлів, а як впорядкований каталог із можливістю фільтрації, пошуку та подальшого розширення.

Сутність `users` відповідає за авторів і власників матеріалів. Сутність `categories` використовується для тематичного поділу даних. Центральною таблицею є `materials`, оскільки саме вона містить назву матеріалу, опис, формат, посилання на файл та зовнішні ключі.

### 2.2 Налаштування підключення до MySQL
Підключення до бази даних було реалізовано через `Sequelize`, а параметри підключення винесено в змінні середовища. Це дозволяє змінювати порт, користувача, пароль і назву бази без редагування серверного коду.

Фрагмент конфігурації підключення:

```js
const { Sequelize } = require("sequelize");
require("dotenv").config();

const sequelize = new Sequelize(
  process.env.DB_NAME,
  process.env.DB_USER,
  process.env.DB_PASSWORD,
  {
    host: process.env.DB_HOST,
    port: Number(process.env.DB_PORT || 3306),
    dialect: "mysql",
    logging: false
  }
);
```

У наведеному фрагменті:
- `dotenv` завантажує конфігурацію із `.env`;
- `Sequelize` створює екземпляр підключення;
- параметр `logging: false` прибирає зайвий технічний шум у консолі;
- окреме винесення параметрів у змінні середовища робить запуск більш зручним.

### 2.3 Опис моделей і зв'язків
У межах лабораторної роботи було створено моделі:
- `User`
- `Category`
- `Material`

Після опису моделей між ними налаштовано зв'язки `One-to-Many`. Один користувач може мати багато матеріалів, а одна категорія може містити багато матеріалів.

Фрагмент налаштування зв'язків:

```js
User.hasMany(Material, {
  foreignKey: "user_id",
  as: "materials"
});
Material.belongsTo(User, {
  foreignKey: "user_id",
  as: "author"
});

Category.hasMany(Material, {
  foreignKey: "category_id",
  as: "materials"
});
Material.belongsTo(Category, {
  foreignKey: "category_id",
  as: "category"
});
```

Цей блок є важливим, оскільки саме він визначає логіку зв'язків між таблицями. Після такого налаштування сервер може повертати не тільки сам матеріал, а й інформацію про автора та категорію в одному запиті.

### 2.4 Реалізація CRUD-операцій
У роботі було реалізовано повний набір базових дій над даними:
- `SELECT`
- `INSERT`
- `UPDATE`
- `DELETE`

Частина функціоналу була показана через `mysql2` у демонстраційному SQL-скрипті, а частина через API-маршрути на базі `Sequelize`.

Нижче наведено фрагмент контролера для отримання та створення матеріалів:

```js
async function getMaterials(req, res) {
  const materials = await Material.findAll({
    include: [
      { model: User, as: "author", attributes: ["id", "name", "email", "role"] },
      { model: Category, as: "category", attributes: ["id", "name", "description"] }
    ],
    order: [["id", "ASC"]]
  });

  res.json(materials);
}

async function createMaterial(req, res) {
  const { title, description, format, fileUrl, userId, categoryId } = req.body;

  if (!title || !format || !userId || !categoryId) {
    return res.status(400).json({
      message: "title, format, userId and categoryId are required"
    });
  }

  const material = await Material.create({
    title,
    description,
    format,
    fileUrl,
    user_id: userId,
    category_id: categoryId
  });
```

У цьому коді:
- `findAll` використовується для вибірки всіх матеріалів;
- `include` додає пов'язані сутності автора та категорії;
- перед створенням запису виконується перевірка обов'язкових полів;
- зовнішні ключі зберігаються в таблиці `materials`.

### 2.5 Демонстрація API-маршрутів
Після реалізації моделей і контролерів було підготовлено серверні маршрути для роботи з даними. Це дозволило протестувати базу даних не лише напряму, а й через HTTP-запити.

У застосунку використовувались такі маршрути:
- `GET /api/health` — перевірка доступності сервера;
- `GET /api/users`, `POST /api/users` — робота з користувачами;
- `GET /api/categories`, `POST /api/categories` — робота з категоріями;
- `GET /api/materials`, `POST /api/materials`, `PUT /api/materials/:id`, `DELETE /api/materials/:id` — операції над матеріалами.

Приклад логіки оновлення матеріалу:

```js
async function updateMaterial(req, res) {
  const material = await Material.findByPk(req.params.id);

  if (!material) {
    return res.status(404).json({ message: "material not found" });
  }

  const { title, description, format, fileUrl, userId, categoryId } = req.body;

  await material.update({
    title: title ?? material.title,
    description: description ?? material.description,
    format: format ?? material.format,
    fileUrl: fileUrl ?? material.fileUrl,
    user_id: userId ?? material.user_id,
    category_id: categoryId ?? material.category_id
  });
}
```

Цей фрагмент показує, що під час оновлення запису змінюються лише ті поля, які були передані в запиті. Такий підхід дозволяє робити часткове редагування без повторного надсилання всього об'єкта.

### 2.6 Підсумок виконання
У результаті виконання лабораторної роботи для `EduVault` було отримано серверну основу з окремою базою даних, пов'язаними сутностями, демонстрацією сирих SQL-запитів і набором CRUD-маршрутів. Створена структура є достатньо гнучкою для подальшого розширення, зокрема для додавання авторизації, керування правами доступу, завантаження файлів і документування API.

---

## 3. Скріншоти результату

![DELETE material](/assets/labs/lab-2/screen-12.png)

![PUT material](/assets/labs/lab-2/screen-11.png)

![POST material](/assets/labs/lab-2/screen-10.png)

![POST category](/assets/labs/lab-2/screen-9.png)

![POST user](/assets/labs/lab-2/screen-8.png)

![GET materials](/assets/labs/lab-2/screen-7.png)

![GET categories](/assets/labs/lab-2/screen-6.png)

![GET users](/assets/labs/lab-2/screen-5.png)

![API health](/assets/labs/lab-2/screen-4.png)

![Raw SQL demo](/assets/labs/lab-2/screen-3.png)

![Seed successful](/assets/labs/lab-2/screen-2.png)

![MySQL container running](/assets/labs/lab-2/screen-1.png)

---

## 4. Висновки

У межах лабораторної роботи було створено базу даних `MySQL` для сервісу `EduVault`, налаштовано підключення з `Node.js`, реалізовано таблиці `users`, `categories` і `materials`, а також зв'язки між ними через `Sequelize`. Додатково було продемонстровано роботу з даними через SQL-операції та HTTP-маршрути. Отриманий результат формує стійку основу для наступних етапів розвитку сервісу, де вже можна впроваджувати авторизацію, контроль доступу та додаткові можливості роботи з файлами.
