## Тема, мета та посилання

**Тема:** СТВОРЕННЯ БАЗИ ДАНИХ У MYSQL. ПІДКЛЮЧЕННЯ NODE.JS ДО MYSQL. РОБОТА З ORM SEQUELIZE.

**Мета:** Навчитися створювати базу даних у MySQL, освоїти виконання SQL-запитів (SELECT, INSERT, UPDATE, DELETE), навчитися підключати серверну програму на Node.js до бази даних та використовувати ORM Sequelize для роботи з БД, а також реалізувати зв'язок One-to-Many між таблицями.

**Посилання на виконані завдання:**
* **Репозиторій власного веб-застосунку (GitHub):** [https://github.com/lerakovaliuk/ecoplant-shop]
* **Власний веб-застосунок (Жива сторінка):** [https://lerakovaliuk.github.io/ecoplant-shop/]
* **Репозиторій звітного документа (GitHub):** [https://github.com/lerakovaliuk/IS-32_appRECORD-KovaliukValeriia-FIOT-2026]
* **Звітний документ (Жива сторінка):** [https://lerakovaliuk.github.io/IS-32_appRECORD-KovaliukValeriia-FIOT-2026/]

---

## 1. Створення бази даних та робота з драйвером mysql2

На першому етапі за допомогою інструменту phpMyAdmin (у пакеті XAMPP) було створено нову базу даних `web_backend_lab`. Для прямої взаємодії з нею без використання ORM було застосовано драйвер `mysql2`. 

Створено скрипт, який підключається до бази даних, автоматично створює таблицю `users` та виконує базові SQL-запити (SELECT, INSERT, UPDATE) за допомогою методів асинхронного виконання (`execute`).

### Лістинг файлу raw-sql.js (Завдання 3-5)
```javascript
const mysql = require('mysql2/promise');

async function testDatabase() {
    const connection = await mysql.createConnection({
        host: 'localhost',
        user: 'root',
        password: '',
        database: 'web_backend_lab'
    });

    console.log("Успішно підключено до бази даних через mysql2!");

    await connection.execute(`
        CREATE TABLE IF NOT EXISTS users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            email VARCHAR(255) NOT NULL
        )
    `);

    await connection.execute(
        'INSERT INTO users (name, email) VALUES (?, ?)', 
        ['Олена', 'olena@gmail.com']
    );

    await connection.execute(
        'UPDATE users SET name = ? WHERE email = ?', 
        ['Олена Ковалюк', 'olena@gmail.com']
    );

    const [rows] = await connection.execute('SELECT * FROM users');
    console.log("Отримані дані (SELECT):", rows);

    await connection.end();
}

testDatabase();
```

---

## 2. Робота з ORM Sequelize та створення моделей

На наступному етапі було підключено сучасну ORM-бібліотеку Sequelize для роботи з реляційною базою даних через JavaScript-об'єкти. 

Було створено дві моделі: `User` (покупець) та `Post` (відгук). Між ними встановлено зв'язок типу "один до багатьох" (One-to-Many), де один користувач може мати кілька відгуків. Цей зв'язок реалізовано за допомогою методів `hasMany` та `belongsTo`.

### Конфігурація підключення (config/database.js)
```javascript
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize('web_backend_lab', 'root', '', {
    host: 'localhost',
    dialect: 'mysql'
});

module.exports = sequelize;
```

### Модель користувача (models/User.js)
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const User = sequelize.define('User', {
    name: {
        type: DataTypes.STRING,
        allowNull: false
    },
    email: {
        type: DataTypes.STRING,
        allowNull: false
    }
});

module.exports = User;
```

### Модель відгуку (models/Post.js)
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Post = sequelize.define('Post', {
    title: {
        type: DataTypes.STRING,
        allowNull: false
    },
    content: {
        type: DataTypes.TEXT,
        allowNull: false
    }
});

module.exports = Post;
```

### Головний файл сервера та синхронізація БД (server.js)
```javascript
const sequelize = require('./config/database');
const User = require('./models/User');
const Post = require('./models/Post');

// Зв'язок One-to-Many
User.hasMany(Post);
Post.belongsTo(User);

async function startApp() {
    try {
        await sequelize.authenticate();
        console.log("Sequelize успішно підключився до БД!");

        await sequelize.sync({ force: true }); 
        console.log("Таблиці очищено і створено заново!");

        const user1 = await User.create({ name: 'Іван', email: 'ivan@gmail.com' });
        await Post.create({ 
            title: 'Мій перший відгук', 
            content: 'Дуже гарні рослини у вашому магазині!', 
            UserId: user1.id 
        });

        const user2 = await User.create({ name: 'Олена', email: 'olena@ukr.net' });
        await Post.create({ 
            title: 'Питання про фікус', 
            content: 'Підкажіть, як часто його поливати взимку?', 
            UserId: user2.id 
        });

        const user3 = await User.create({ name: 'Марія', email: 'maria@ecoplant.com' });
        await Post.create({ 
            title: 'Супер сервіс', 
            content: 'Замовляла монстеру, приїхала ціла і дуже швидко.', 
            UserId: user3.id 
        });

        console.log("Всіх користувачів успішно 'жорстко' записано в базу!");

    } catch (error) {
        console.error("Помилка:", error);
    }
}

startApp();
```

---

## 3. Скріншоти результатів виконання

### 1. Виконання raw-sql.js у терміналі
*(Тут видно повідомлення про успішне створення таблиці та виконання запитів)*
![Термінал mysql2](/assets/labs/lab-2/mysql2-terminal.png)

### 2. Запуск сервера Sequelize (server.js) у терміналі
*(Відображено успішне підключення, синхронізацію та запис нових користувачів)*
![Термінал Sequelize](/assets/labs/lab-2/sequelize-terminal.png)

### 3. База даних web_backend_lab у phpMyAdmin
*(Відображено наявність автоматично створених таблиць Users та Posts)*
![Таблиці в БД](/assets/labs/lab-2/db-tables.png)

### 4. Дані у таблиці Users (phpMyAdmin)
*(Видно 3 створених користувачів із автоматичними полями createdAt та updatedAt)*
![Дані Users](/assets/labs/lab-2/db-users.png)

### 5. Дані у таблиці Posts (phpMyAdmin)
*(Видно пости та їх прив'язку до користувачів через UserId)*
![Дані Posts](/assets/labs/lab-2/db-posts.png)

---

## Висновки

Під час виконання лабораторної роботи було успішно налаштовано локальний сервер бази даних MySQL та здійснено підключення до нього програмного середовища Node.js. 

**Набуті навички:** На практиці закріплено розуміння роботи драйвера `mysql2`, за допомогою якого були написані "сирі" (raw) SQL-запити для виконання базових операцій CRUD (створення таблиць, додавання, оновлення та вибірка даних). Також було успішно інтегровано та налаштовано сучасну бібліотеку ORM Sequelize, що дозволила спростити роботу з реляційною базою даних шляхом її відображення у вигляді JavaScript-об'єктів. 

**Узагальнення отриманих результатів:** Було спроєктовано структуру бази даних для інтернет-магазину. У рамках роботи створено моделі `User` та `Post` і успішно реалізовано зв'язок `One-to-Many` (один до багатьох) між цими сутностями. За допомогою методу синхронізації `sync()` таблиці були автоматично згенеровані на стороні сервера MySQL. Написано скрипт для "жорсткого" наповнення (сідінгу) бази даних тестовими користувачами та їхніми відгуками, результати якого підтверджено візуально через інтерфейс phpMyAdmin. Роботу можна вважати повністю виконаною згідно із завданням.