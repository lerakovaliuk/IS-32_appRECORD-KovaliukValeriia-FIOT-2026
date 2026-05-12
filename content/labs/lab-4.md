## Тема, мета та посилання

**Тема:** РОЗШИРЕНІ МОЖЛИВОСТІ NODE.JS-ДОДАТКІВ: ЛОГУВАННЯ, ЗАВАНТАЖЕННЯ ФАЙЛІВ, МОНІТОРИНГ ПРОДУКТИВНОСТІ.

**Мета:** Ознайомитися з розширеними можливостями серверних застосунків на базі Node.js. Набути практичних навичок з реалізації логування НТТР-запитів і подій, організації безпечного завантаження файлів на сервер, а також моніторингу продуктивності та стабільності роботи застосунку за допомогою вбудованих інструментів та менеджерів процесів.

**Посилання на виконані завдання:**
* **Репозиторій власного веб-застосунку (GitHub):** [https://github.com/lerakovaliuk/ecoplant-shop]
* **Власний веб-застосунок (Жива сторінка):** [https://lerakovaliuk.github.io/ecoplant-shop/]
* **Репозиторій звітного документа (GitHub):** [https://github.com/lerakovaliuk/IS-32_appRECORD-KovaliukValeriia-FIOT-2026]
* **Звітний документ (Жива сторінка):** [https://lerakovaliuk.github.io/IS-32_appRECORD-KovaliukValeriia-FIOT-2026/]


---

## 1. Теоретичний опис та архітектура рішення

Під час виконання лабораторної роботи існуючий REST API (створений у попередній роботі) був суттєво розширений для роботи в умовах, наближених до реального production-середовища. 

Було інтегровано такі ключові модулі:
1. **Morgan** — для автоматичного логування всіх вхідних HTTP-запитів.
2. **Winston** — для професійного гнучкого логування подій та помилок із записом у зовнішні файли (`app.log` та `error.log`).
3. **Multer** — middleware для обробки `multipart/form-data` запитів, що дозволило реалізувати завантаження файлів на сервер із попередньою валідацією їх типу та розміру.
4. **PM2** — менеджер процесів, що забезпечує роботу сервера у фоновому режимі, автоматичний перезапуск у разі збоїв та зручний моніторинг ресурсів.

---

## 2. Реалізація завдань з програмним кодом

Відповідно до методичних вказівок, наведено виконання кожного з поставлених завдань із відповідними фрагментами коду з файлу `server.js`.

### Завдання 1. Ініціалізація проєкту
Встановлено необхідні пакети та налаштовано базовий сервер на Express.
```javascript
const express = require("express");
const fs = require("fs");
const app = express();
const port = 3000;

app.use(express.json());
```

### Завдання 2. Логування HTTP-запитів
Підключено бібліотеку Morgan для виведення інформації про всі HTTP-запити в консоль.
```javascript
const morgan = require('morgan');
app.use(morgan('combined'));
```

### Завдання 3. Файлове логування подій
Інтегровано Winston. Налаштовано запис подій рівня `info` у файл `app.log`, а помилок — у файл `error.log`.
```javascript
const winston = require('winston');

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: 'error.log', level: 'error' }),
        new winston.transports.File({ filename: 'app.log' })
    ]
});
```

### Завдання 9. Вимірювання часу відповіді
Створено middleware для обчислення тривалості обробки кожного запиту та запису цих даних у лог.
```javascript
app.use((req, res, next) => {
    const start = Date.now();
    res.on('finish', () => {
        const duration = Date.now() - start;
        logger.info(`${req.method} ${req.url} - ${duration}ms`);
    });
    next();
});
```

### Завдання 5, 6, 7. Завантаження та валідація файлів
Підключено Multer. Реалізовано збереження в папку `uploads/`, додано валідацію за типом файлу (тільки JPG, PNG, PDF) та обмеження розміру до 2 МБ.
```javascript
const multer = require('multer');

const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        const uploadPath = 'uploads/';
        if (!fs.existsSync(uploadPath)) fs.mkdirSync(uploadPath);
        cb(null, uploadPath);
    },
    filename: (req, file, cb) => {
        cb(null, Date.now() + '-' + file.originalname);
    }
});

const fileFilter = (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
    if (allowedTypes.includes(file.mimetype)) {
        cb(null, true);
    } else {
        cb(new Error('Недопустимий формат файлу. Дозволено: JPG, PNG, PDF.'), false);
    }
};

const upload = multer({ 
    storage: storage,
    limits: { fileSize: 2 * 1024 * 1024 }, 
    fileFilter: fileFilter
});

// Завдання 5: Завантаження одного файлу
app.post('/upload', upload.single('file'), (req, res) => {
    res.json({ message: 'Файл успішно завантажено', file: req.file });
});

// Завдання 6: Завантаження кількох файлів
app.post('/upload-multiple', upload.array('files', 5), (req, res) => {
    res.json({ message: 'Файли успішно завантажено', files: req.files });
});
```

### Завдання 8. Моніторинг стану сервера
Створено маршрут для отримання інформації про використання пам'яті та час безперебійної роботи (uptime).
```javascript
app.get('/status', (req, res) => {
    res.json({
        uptime: process.uptime(),
        memoryUsage: process.memoryUsage(),
        timestamp: new Date()
    });
});
```

### Завдання 4. Обробка помилок
Створено тестовий маршрут та реалізовано глобальний middleware для перехоплення помилок, їх логування через Winston та повернення JSON-відповіді клієнту.
```javascript
app.get('/test-error', (req, res, next) => {
    next(new Error('Тестова помилка сервера!')); 
});

app.use((err, req, res, next) => {
    logger.error(`${err.message} - ${req.originalUrl}`);
    res.status(err.status || 500).json({ error: err.message });
});
```

### Завдання 10. Інтеграція менеджера процесів
Для забезпечення стабільності сервер запущено через PM2 за допомогою команд у терміналі:
```bash
npm install -g pm2
pm2 start server.js --name "ecoplant-api"
pm2 logs
```

---

## 3. Тестування API (Скріншоти виконання)

Для тестування розроблених функцій використовувались браузер, інструмент Postman (для надсилання `multipart/form-data` запитів) та термінал.

### 1. Моніторинг статусу та обробка помилок
*(GET запит на `/status` з відображенням `uptime` та `memoryUsage`, а також перевірка `/test-error`)*
![Статус сервера](/assets/labs/lab-4/api-status-error.png)

### 2. Записи у файлах логів
*(Демонстрація файлів `app.log` та `error.log`, які згенерувала бібліотека Winston)*
![Файли логів](/assets/labs/lab-4/winston-logs.png)

### 3. Завантаження одного файлу (Multer)
*(Успішний POST-запит через Postman на `/upload` з демонстрацією збереженого файлу в папці `uploads/`)*
![Завантаження файлу](/assets/labs/lab-4/upload-single.png)

### 4. Валідація та завантаження кількох файлів
*(POST-запит на `/upload-multiple` та перевірка обмежень за форматом/розміром)*
![Завантаження кількох файлів](/assets/labs/lab-4/upload-multiple.png)

### 5. Інтеграція PM2
*(Скріншоти терміналу: успішний запуск процесу `ecoplant-api` та виведення логів через `pm2 logs`)*
![Запуск PM2](/assets/labs/lab-4/pm2-status.png)

---

### Бонусне завдання. Ротація логів, API логів та Панель моніторингу
У `winston` налаштовано механізм ротації: при досягненні розміру файлу 5 МБ створюється новий файл (максимум 5 файлів). Створено маршрут `GET /logs` для отримання останніх 50 логів. Розроблено та інтегровано клієнтську веб-панель моніторингу (маршрут `GET /dashboard`), яка за допомогою JavaScript кожні 3 секунди асинхронно запитує метрики сервера (пам'ять, uptime) та відображає їх у зручному графічному інтерфейсі разом із системними логами.
![dashboard](/assets/labs/lab-4/bonus-dashboard.png)

---

## Висновки

Під час виконання лабораторної роботи було детально розглянуто та впроваджено розширені можливості Node.js-додатків, критично важливі для розробки надійних production-серверів. 

**Набуті навички:** Було опановано роботу з професійними системами логування. Інтеграція `Morgan` та `Winston` дозволила організувати ефективний запис HTTP-запитів і відстеження внутрішніх помилок із поділом на рівні (`info`, `error`) та збереженням у файли. Налаштовано глобальний перехоплювач помилок (Error Handler Middleware). Також на практиці було досліджено архітектуру запитів `multipart/form-data` і налаштовано безпечне завантаження файлів за допомогою бібліотеки `Multer` (включно із захистом сервера шляхом перевірки MIME-типів файлів та їх розміру).

**Узагальнення отриманих результатів:** Проєкт було доповнено інструментами моніторингу: написано код для контролю споживання оперативної пам'яті (`process.memoryUsage`) та вимірювання часу обробки запитів. Як фінальний етап, застосунок було переведено під управління менеджера процесів `PM2`, що забезпечило можливість моніторингу навантаження в реальному часі та гарантію автоматичного відновлення роботи сервера після критичних збоїв. Усі поставлені завдання успішно виконані та протестовані.