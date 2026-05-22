## 1. Тема, мета, посилання

### 1.1 Тема
«REST API. Реєстрація та авторизація. Валідація даних».

### 1.2 Мета
Реалізувати для сервісу `EduVault` реєстрацію користувачів, авторизацію через `JWT`, валідацію вхідних даних, оновлення профілю, зміну пароля, вихід із системи та захист серверних маршрутів.

### 1.3 Посилання
- Репозиторій власного проєкту (GitHub): буде додано після публікації.
- Розгорнутий застосунок: буде додано після розгортання.
- Репозиторій звітного HTML-документа (GitHub): буде додано після публікації.
- Звітний HTML-документ: буде додано після розгортання.

---

## 2. Хід виконання роботи

### 2.1 Загальна логіка реалізації
У межах третьої лабораторної роботи для `EduVault` було розширено серверну частину, створену на попередніх етапах. Якщо в другій лабораторній роботі основний акцент було зроблено на структурі бази даних та CRUD-операціях, то на цьому етапі головною задачею стало впровадження безпечної взаємодії з користувачами.

Було реалізовано:
- реєстрацію нового користувача;
- вхід до системи за email і паролем;
- перевірку коректності вхідних даних;
- формування `access token` і `refresh token`;
- захист приватних маршрутів;
- оновлення профілю;
- зміну пароля;
- завершення сесії користувача;
- доступ до окремих ресурсів лише для адміністратора.

Для зберігання паролів використано хешування через `bcryptjs`, а для авторизації — бібліотеку `jsonwebtoken`.

### 2.2 Розширення моделі користувача
Щоб підтримати авторизацію, модель `User` була доповнена двома новими полями:
- `passwordHash`
- `refreshToken`

Це дозволило не зберігати пароль у відкритому вигляді та окремо контролювати тривалі сесії користувачів.

Сутність користувача тепер містить:
- базові поля `name`, `email`, `role`;
- хеш пароля `password_hash`;
- токен оновлення `refresh_token`.

Такий підхід є стандартним для серверних застосунків, де важливо забезпечити мінімальний рівень безпеки ще до переходу до більш складних механізмів захисту.

### 2.3 Реєстрація та валідація даних
На етапі реєстрації було додано перевірку повноти та коректності введених даних. Сервер перевіряє:
- чи заповнені всі обов'язкові поля;
- чи має email коректний формат;
- чи достатня довжина пароля;
- чи збігаються `password` і `confirmPassword`;
- чи не існує вже користувач із таким email.

Фрагмент логіки реєстрації:

```js
async function register(req, res) {
  const { name, email, password, confirmPassword, role } = req.body;

  if (!name || !email || !password || !confirmPassword) {
    return res.status(400).json({ message: "All registration fields are required" });
  }

  if (!validateEmail(email)) {
    return res.status(400).json({ message: "Email format is invalid" });
  }

  if (password.length < 6) {
    return res.status(400).json({ message: "Password must be at least 6 characters long" });
  }

  if (password !== confirmPassword) {
    return res.status(400).json({ message: "Password confirmation does not match" });
  }
```

Після проходження перевірок пароль хешується, а користувач зберігається в базі даних:

```js
const passwordHash = await bcrypt.hash(password, 10);

const user = await User.create({
  name,
  email,
  role: role === "admin" ? "admin" : "user",
  passwordHash
});
```

Таким чином, сервер не працює з відкритим паролем далі за межами одного запиту, а в БД потрапляє лише його хеш.

### 2.4 Авторизація та JWT-токени
Після входу в систему користувач отримує два типи токенів:
- `accessToken` — короткочасний токен для доступу до захищених маршрутів;
- `refreshToken` — довший токен для оновлення доступу без повторного введення пароля.

Логіка створення токенів:

```js
function signAccessToken(user) {
  return jwt.sign(
    {
      id: user.id,
      email: user.email,
      role: user.role
    },
    process.env.JWT_SECRET,
    { expiresIn: "1h" }
  );
}

function signRefreshToken(user) {
  return jwt.sign(
    {
      id: user.id,
      email: user.email
    },
    process.env.JWT_SECRET,
    { expiresIn: "7d" }
  );
}
```

Під час логіну система:
- знаходить користувача за email;
- перевіряє пароль через `bcrypt.compare`;
- створює обидва токени;
- зберігає `refreshToken` у записі користувача.

Фрагмент логіки входу:

```js
const passwordMatches = await bcrypt.compare(password, user.passwordHash);
if (!passwordMatches) {
  return res.status(401).json({ message: "Invalid credentials" });
}

const accessToken = signAccessToken(user);
const refreshToken = signRefreshToken(user);

await user.update({ refreshToken });
```

### 2.5 Захист маршрутів
Щоб обмежити доступ до приватних ресурсів, було створено middleware для перевірки `Authorization`-заголовка та валідації токена.

Фрагмент middleware:

```js
function authMiddleware(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return res.status(401).json({ message: "Authorization token is missing" });
  }

  const token = authHeader.split(" ")[1];

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    req.user = payload;
    next();
  } catch (error) {
    return res.status(401).json({ message: "Invalid or expired token" });
  }
}
```

Після успішної перевірки сервер додає у `req.user` дані користувача з токена, і наступні контролери вже можуть працювати з ідентифікатором та роллю без повторного запиту до клієнта.

### 2.6 Рольова модель доступу
Окремо було реалізовано просту перевірку ролей, яка дозволяє обмежити доступ до певних маршрутів лише для адміністратора.

Фрагмент рольового middleware:

```js
function roleMiddleware(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user || !allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ message: "Access denied" });
    }

    next();
  };
}
```

Цей механізм було використано для маршруту перегляду списку всіх користувачів. Таким чином, навіть на базовому рівні система вже підтримує розмежування прав між адміністратором і звичайним користувачем.

### 2.7 Реалізація профілю та зміни пароля
Крім реєстрації та входу, було реалізовано оновлення профілю користувача та зміну його пароля. Це робить систему ближчою до реального сценарію використання.

Оновлення профілю дозволяє:
- змінити ім'я;
- змінити email;
- перевірити, чи новий email не зайнятий іншим користувачем.

Зміна пароля передбачає:
- перевірку поточного пароля;
- перевірку довжини нового пароля;
- перевірку збігу `newPassword` і `confirmPassword`;
- збереження нового хешу.

Фрагмент логіки зміни пароля:

```js
const passwordMatches = await bcrypt.compare(currentPassword, user.passwordHash);
if (!passwordMatches) {
  return res.status(401).json({ message: "Current password is incorrect" });
}

if (newPassword !== confirmPassword) {
  return res.status(400).json({ message: "Password confirmation does not match" });
}

const passwordHash = await bcrypt.hash(newPassword, 10);
await user.update({ passwordHash });
```

### 2.8 Маршрути авторизації
Для роботи із системою авторизації було підготовлено окрему групу маршрутів:
- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`
- `GET /api/auth/me`
- `PUT /api/auth/profile`
- `PUT /api/auth/change-password`
- `DELETE /api/auth/delete-account`

Підключення маршрутів виконано через окремий роутер:

```js
router.post("/register", register);
router.post("/login", login);
router.post("/refresh", refresh);
router.post("/logout", authMiddleware, logout);
router.get("/me", authMiddleware, me);
router.put("/profile", authMiddleware, updateProfile);
router.put("/change-password", authMiddleware, changePassword);
router.delete("/delete-account", authMiddleware, deleteAccount);
```

Такий розподіл робить структуру проєкту більш чистою: логіка авторизації відокремлена від CRUD-маршрутів матеріалів, категорій та користувачів.

### 2.9 Практичний результат
У результаті виконання третьої лабораторної роботи серверна частина `EduVault` отримала повноцінний базовий механізм керування користувачами та доступом. Система вже підтримує:
- створення облікових записів;
- перевірку правильності введених даних;
- авторизацію через токени;
- захист приватних маршрутів;
- розмежування прав доступу;
- оновлення профілю та пароля;
- завершення сесії користувача.

Це створює стабільну основу для подальшого переходу до роботи з файлами, логуванням, безпекою та документуванням API.

---

## 3. Скріншоти результату

![POST logout](/assets/labs/lab-3/screen-12.png)

![PUT change password](/assets/labs/lab-3/screen-11.png)

![PUT update profile](/assets/labs/lab-3/screen-10.png)

![DELETE material](/assets/labs/lab-3/screen-9.png)

![PUT material](/assets/labs/lab-3/screen-8.png)

![POST material](/assets/labs/lab-3/screen-7.png)

![POST category](/assets/labs/lab-3/screen-6.png)

![GET users](/assets/labs/lab-3/screen-5.png)

![POST refresh](/assets/labs/lab-3/screen-4.png)

![GET me](/assets/labs/lab-3/screen-3.png)

![POST login](/assets/labs/lab-3/screen-2.png)

![POST register](/assets/labs/lab-3/screen-1.png)

---

## 4. Висновки

У межах лабораторної роботи для сервісу `EduVault` було реалізовано реєстрацію користувачів, авторизацію через `JWT`, механізм `refresh token`, валідацію даних, захист приватних маршрутів і рольову перевірку доступу. Додатково реалізовано оновлення профілю, зміну пароля та завершення сесії. Отриманий результат переводить проєкт від простого CRUD-рівня до більш реалістичної серверної архітектури, у якій уже враховано базові вимоги до безпеки та контролю доступу.
