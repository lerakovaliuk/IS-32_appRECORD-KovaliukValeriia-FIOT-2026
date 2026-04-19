## Тема, мета та посилання

**Тема:** РОЗРОБКА ФУНКЦІОНАЛЬНОГО REST API. РЕЄСТРАЦІЯ ТА АВТОРИЗАЦІЯ КОРИСТУВАЧІВ. ВАЛІДАЦІЯ ДАНИХ І ОБРОБКА ПОМИЛОК.

**Мета:** Вивчити принципи побудови REST API. Набути практичних навичок розробки серверного застосунку з використанням платформи Node.js і фреймворку Express. Реалізувати механізми реєстрації та авторизації користувачів, забезпечити валідацію вхідних даних і обробку помилок. Організувати захищений доступ до ресурсів із використанням JWT-токенів, системи ролей користувачів, а також інтегрувати зовнішню авторизацію OAuth (Google Login).

**Посилання на виконані завдання:**
* **Репозиторій власного веб-застосунку (GitHub):** [https://github.com/lerakovaliuk/ecoplant-shop]
* **Власний веб-застосунок (Жива сторінка):** [https://lerakovaliuk.github.io/ecoplant-shop/]
* **Репозиторій звітного документа (GitHub):** [https://github.com/lerakovaliuk/IS-32_appRECORD-KovaliukValeriia-FIOT-2026]
* **Звітний документ (Жива сторінка):** [https://lerakovaliuk.github.io/IS-32_appRECORD-KovaliukValeriia-FIOT-2026/]

---

## 1. Теоретичний опис та архітектура рішення

Під час виконання роботи було спроєктовано та розроблено повноцінний REST API сервер на базі Node.js та фреймворку Express. Основна логіка взаємодії клієнта та сервера побудована на принципі "Stateless", де кожен запит є незалежним, а для доступу до захищених ресурсів використовується JWT (JSON Web Token).

В рамках проєкту були успішно реалізовані **усі 20 завдань** підвищеної складності, результати яких наведено у наступному розділі. База даних автоматично оновлена за допомогою ORM Sequelize. Серверна логіка (бекенд) реалізована у файлі `server.js`.

---

## 2. Реалізація завдань з програмним кодом

Нижче наведено виконання кожного окремого завдання з відповідними фрагментами коду.

### Завдання 1. Встановити необхідні бібліотеки
У проєкті встановлено та підключено необхідні модулі (`express`, `bcryptjs`, `jsonwebtoken`, `cors`, `express-rate-limit`, `google-auth-library`).
```javascript
const express = require("express");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const cors = require("cors");
const rateLimit = require("express-rate-limit");
const fs = require("fs");
const { OAuth2Client } = require('google-auth-library');
const sequelize = require('./config/database');
const User = require('./models/User');

const app = express();
app.use(express.json());
app.use(cors());

const SECRET_KEY = "ecoplant_secret_key_123";
const REFRESH_SECRET = "ecoplant_refresh_secret";
```

### Завдання 8, 11, 12, 19. Зберігати користувачів у базі (Додавання ролей, Refresh токена, підтвердження email)
Збереження даних реалізовано через ORM Sequelize у файлі моделі `models/User.js`.
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const User = sequelize.define('User', {
    name: { type: DataTypes.STRING, allowNull: false },
    email: { type: DataTypes.STRING, allowNull: false, unique: true },
    password: { type: DataTypes.STRING, allowNull: true }, 
    role: { type: DataTypes.STRING, defaultValue: 'user' }, // Завдання 8
    isEmailConfirmed: { type: DataTypes.BOOLEAN, defaultValue: false }, // Завдання 19
    confirmationToken: { type: DataTypes.STRING, allowNull: true },
    resetPasswordToken: { type: DataTypes.STRING, allowNull: true },
    refreshToken: { type: DataTypes.STRING, allowNull: true } // Завдання 12
});

module.exports = User;
```

### Завдання 13. Додати логування помилок
Усі помилки записуються у файл `error.log`.
```javascript
const logError = (error) => {
    const logMessage = `${new Date().toISOString()} - ${error.message}\n`;
    fs.appendFileSync("error.log", logMessage);
    console.error(error);
};
```

### Завдання 15. Додати middleware для перевірки токена
Функція для захисту маршрутів від неавторизованих запитів.
```javascript
const authenticateToken = (req, res, next) => {
    const token = req.header("Authorization");
    if (!token) return res.status(401).json({ message: "Немає токена" });

    try {
        const verified = jwt.verify(token.replace("Bearer ", ""), SECRET_KEY);
        req.user = verified;
        next();
    } catch (err) {
        res.status(401).json({ message: "Недійсний токен" });
    }
};
```

### Завдання 14. Обмежити кількість спроб входу
Захист від Brute-force атак (максимум 5 спроб на 15 хвилин).
```javascript
const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, 
    max: 5, 
    message: { message: "Забагато спроб входу. Спробуйте через 15 хвилин." }
});
```

### Завдання 20. Реалізувати OAuth (Google login)
Маршрут для входу через Google із генерацією JWT.
```javascript
const GOOGLE_CLIENT_ID = "YOUR_GOOGLE_CLIENT_ID.apps.googleusercontent.com";
const googleClient = new OAuth2Client(GOOGLE_CLIENT_ID);

app.post("/auth/google", async (req, res) => {
    try {
        const { token } = req.body; 
        const ticket = await googleClient.verifyIdToken({
            idToken: token,
            audience: GOOGLE_CLIENT_ID,
        });
        
        const payload = ticket.getPayload(); 
        const { email, name } = payload;
        let user = await User.findOne({ where: { email } });

        if (!user) {
            user = await User.create({
                name: name, email: email, password: null, 
                role: 'user', isEmailConfirmed: true 
            });
        }

        const accessToken = jwt.sign({ id: user.id, role: user.role }, SECRET_KEY, { expiresIn: "15m" });
        const refreshToken = jwt.sign({ id: user.id }, REFRESH_SECRET, { expiresIn: "7d" });

        user.refreshToken = refreshToken;
        await user.save();
        res.json({ accessToken, refreshToken, user: { name: user.name, role: user.role } });
    } catch (error) {
        logError(error);
        res.status(401).json({ message: "Помилка авторизації через Google" });
    }
});
```

### Завдання 2, 3, 7. Реєстрація, валідація даних та підтвердження пароля
Обробка реєстрації з хешуванням пароля та валідацією полів.
```javascript
app.post("/register", async (req, res) => {
    try {
        const { name, email, password, confirmPassword } = req.body;

        if (!name || !email || !password || !confirmPassword) return res.status(400).json({ message: "Всі поля обов'язкові" });
        if (password !== confirmPassword) return res.status(400).json({ message: "Паролі не співпадають" });
        
        const existingUser = await User.findOne({ where: { email } });
        if (existingUser) return res.status(400).json({ message: "Email вже існує" });

        const hashedPassword = await bcrypt.hash(password, 10);
        const confirmToken = jwt.sign({ email }, SECRET_KEY, { expiresIn: '1d' });

        const newUser = await User.create({
            name, email, password: hashedPassword,
            role: await User.count() === 0 ? 'admin' : 'user',
            confirmationToken: confirmToken
        });

        console.log(`[EMAIL SEND]: Для підтвердження пошти: http://localhost:3000/confirm/${confirmToken}`);
        res.status(201).json({ message: "Користувача створено. Перевірте email." });
    } catch (error) {
        logError(error); res.status(500).json({ message: "Помилка реєстрації" });
    }
});
```

### Завдання 19. Підтвердження email
```javascript
app.get("/confirm/:token", async (req, res) => {
    try {
        const user = await User.findOne({ where: { confirmationToken: req.params.token } });
        if (!user) return res.status(400).send("Недійсний токен");
        
        user.isEmailConfirmed = true;
        user.confirmationToken = null;
        await user.save();
        res.send("Email успішно підтверджено!");
    } catch (error) {
        logError(error); res.status(500).send("Помилка");
    }
});
```

### Завдання 2, 12. Авторизація та реалізація refresh token
Перевірка пароля, генерація Access та Refresh токенів.
```javascript
app.post("/login", loginLimiter, async (req, res) => {
    try {
        const { email, password } = req.body;
        const user = await User.findOne({ where: { email } });

        if (!user || !user.password || !(await bcrypt.compare(password, user.password))) {
            return res.status(400).json({ message: "Невірний логін або пароль" });
        }
        if (!user.isEmailConfirmed) return res.status(400).json({ message: "Підтвердіть email" });

        const accessToken = jwt.sign({ id: user.id, role: user.role }, SECRET_KEY, { expiresIn: "15m" });
        const refreshToken = jwt.sign({ id: user.id }, REFRESH_SECRET, { expiresIn: "7d" });

        user.refreshToken = refreshToken;
        await user.save();

        res.json({ accessToken, refreshToken, user: { name: user.name, role: user.role } });
    } catch (error) {
        logError(error); res.status(500).json({ message: "Помилка входу" });
    }
});
```

### Завдання 4. Реалізувати захищений маршрут
```javascript
app.get("/profile", authenticateToken, async (req, res) => {
    const user = await User.findByPk(req.user.id, { attributes: ['name', 'email', 'role'] });
    res.json(user);
});
```

### Завдання 10. Додати оновлення профілю
```javascript
app.put("/profile", authenticateToken, async (req, res) => {
    try {
        const user = await User.findByPk(req.user.id);
        if (req.body.name) user.name = req.body.name;
        await user.save();
        res.json({ message: "Профіль оновлено", user });
    } catch (error) {
        logError(error); res.status(500).json({ message: "Помилка оновлення" });
    }
});
```

### Завдання 16. Реалізувати зміну пароля
```javascript
app.post("/change-password", authenticateToken, async (req, res) => {
    try {
        const { oldPassword, newPassword } = req.body;
        const user = await User.findByPk(req.user.id);
        
        if (!user.password || !(await bcrypt.compare(oldPassword, user.password))) {
            return res.status(400).json({ message: "Старий пароль невірний" });
        }
        
        user.password = await bcrypt.hash(newPassword, 10);
        await user.save();
        res.json({ message: "Пароль змінено" });
    } catch (error) {
        logError(error); res.status(500).json({ message: "Помилка" });
    }
});
```

### Завдання 17. Реалізувати видалення користувача
```javascript
app.delete("/profile", authenticateToken, async (req, res) => {
    await User.destroy({ where: { id: req.user.id } });
    res.json({ message: "Користувача видалено" });
});
```

### Завдання 18. Реалізувати відновлення пароля
```javascript
app.post("/forgot-password", async (req, res) => {
    const user = await User.findOne({ where: { email: req.body.email } });
    if (user) {
        const resetToken = jwt.sign({ id: user.id }, SECRET_KEY, { expiresIn: '1h' });
        user.resetPasswordToken = resetToken;
        await user.save();
        console.log(`[EMAIL SEND]: Відновлення пароля: http://localhost:3000/reset/${resetToken}`);
    }
    res.json({ message: "Якщо email існує, ми надіслали інструкції." });
});
```

### Завдання 9. Реалізувати logout
```javascript
app.post("/logout", authenticateToken, async (req, res) => {
    const user = await User.findByPk(req.user.id);
    user.refreshToken = null;
    await user.save();
    res.json({ message: "Ви успішно вийшли" });
});
```

---

## 3. Тестування API (Скріншоти виконання)

Для тестування розроблених маршрутів (Завдання 5) використовувався інструмент Thunder Client. Логіка OAuth (Google Login) тестується безпосередньо при інтеграції з клієнтським застосунком.

### 1. Синхронізація БД та запуск сервера
*(Демонстрація того, як ORM Sequelize автоматично додала необхідні колонки в таблицю MySQL)*
![Запуск сервера](/assets/labs/lab-3/terminal-start.png)

### 2. Реєстрація користувача (POST /register)
*(Успішне створення користувача після проходження валідації. Статус: 201 Created)*
![Реєстрація](/assets/labs/lab-3/api-register.png)

### 3. Підтвердження Email (GET /confirm)
*(Імітація переходу за посиланням з електронного листа. Статус: 200 OK)*
![Підтвердження пошти](/assets/labs/lab-3/api-confirm.png)

### 4. Авторизація користувача (POST /login)
*(Успішна перевірка пароля та генерація JWT Access та Refresh токенів)*
![Вхід](/assets/labs/lab-3/api-login.png)

### 5. Доступ до захищеного профілю (GET /profile)
*(Успішне отримання даних за допомогою JWT-токена у заголовку Authorization)*
![Профіль](/assets/labs/lab-3/api-profile.png)

### 6. Вихід із системи (POST /logout)
*(Знищення Refresh токена на сервері для завершення сесії користувача)*
![Вихід](/assets/labs/lab-3/api-logout.png)

---

## Висновки

Під час виконання лабораторної роботи було детально вивчено принципи побудови REST API та клієнт-серверної архітектури (на прикладі середовища Node.js з використанням фреймворку Express). 

**Набуті навички:** На практиці було опановано механізми забезпечення безпеки інформаційної системи. Реалізовано безпечне хешування паролів користувачів за допомогою бібліотеки `bcryptjs`. Освоєно процес створення та верифікації JSON Web Tokens (JWT) для авторизації клієнтів за принципом Stateless. Також було досліджено та підключено протокол OAuth 2.0 за допомогою `google-auth-library` для реалізації швидкого та безпечного входу через Google-акаунт.

**Узагальнення отриманих результатів:** Було створено повноцінний бекенд для власного вебзастосунку з максимальним спектром можливостей (виконано всі 20 поставлених завдань). Налаштовано комплексну валідацію вхідних даних, реалізовано механізм підтвердження email, додано систему ролей (admin/user), інтегровано захист від Brute-force атак та логування. Працездатність усіх API-маршрутів успішно перевірено та задокументовано за допомогою інструменту Thunder Client.