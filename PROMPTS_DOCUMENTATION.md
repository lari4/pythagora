# Документация AI Промптов Pythagora

## Обзор

Pythagora использует GPT-4 для автоматической генерации и обработки тестов. Сами промпты находятся на сервере Pythagora API (https://github.com/Pythagora-io/api/tree/main/prompts), но этот документ описывает все API методы, которые используют промпты, и данные, которые в них отправляются.

## Конфигурация API

**Файл**: `src/helpers/api.js:4-10`

Приложение поддерживает два типа API ключей:
- OpenAI API Key - для прямого обращения к GPT-4
- Pythagora API Key - для обращения через сервер Pythagora

```javascript
function getApiConfig() {
    return {
        apiUrl: args.pythagora_api_server || PYTHAGORA_API_SERVER,
        apiKey: args.openai_api_key || args.pythagora_api_key,
        apiKeyType: args.openai_api_key ? 'openai' : 'pythagora'
    }
}
```

---

## 1. ГЕНЕРАЦИЯ ЮНИТ-ТЕСТОВ (Unit Tests Generation)

### 1.1. API Метод: `UnitTests.runProcessing()`

**Назначение**: Генерация юнит-тестов для JavaScript функций с использованием GPT-4.

**Файл**: `src/helpers/unitTests.js:16-31`

**Описание**: Этот метод является основным для генерации юнит-тестов. Он анализирует указанную функцию или файл, находит все связанные функции через AST (Abstract Syntax Tree) парсинг, и отправляет их на сервер Pythagora, который использует GPT-4 для генерации тестов.

**Входные параметры**:

```javascript
const unitTests = new UnitTests(
    {
        pathToProcess: args.path,        // Путь к файлу или директории
        pythagoraRoot: args.pythagora_root, // Корневая директория Pythagora
        funcName: args.func,             // Имя конкретной функции (опционально)
        force: args.force                // Перезаписать существующие тесты
    },
    Api,                                 // API инстанс для обращения к GPT-4
    {
        isSaveTests: true,               // Сохранять ли сгенерированные тесты
        screen,                          // UI screen для отображения прогресса
        spinner,                         // UI spinner
        scrollableContent                // UI scrollable content
    }
);
```

**Выходные данные**:

```javascript
const {errors, skippedFiles, testsGenerated} = await unitTests.runProcessing();
// errors - массив ошибок, возникших при генерации
// skippedFiles - массив файлов, которые были пропущены
// testsGenerated - массив сгенерированных тестов с путями
```

**Что отправляется в промпт GPT-4**:
1. Код функции, для которой нужно создать тесты
2. Все функции, которые вызываются из этой функции (найденные через AST парсинг)
3. Зависимости и импорты
4. Контекст использования функции

**Команда для запуска**:

```bash
# Для одной функции
npx pythagora --unit-tests --func <FUNCTION_NAME>

# Для файла
npx pythagora --unit-tests --path ./path/to/file.js

# Для директории
npx pythagora --unit-tests --path ./path/to/folder/
```

**Пример вызова в коде**:

```javascript
// src/helpers/unitTests.js:16-31
const unitTests = new UnitTests(
    {
        pathToProcess: args.path,
        pythagoraRoot: args.pythagora_root,
        funcName: args.func,
        force: args.force
    },
    Api,
    {
        isSaveTests: true,
        screen,
        spinner,
        scrollableContent
    }
);
const {errors, skippedFiles, testsGenerated} = await unitTests.runProcessing();
```

**Особенности**:
- Лучше всего работает для standalone функций (например, helper функций)
- Функция должна быть экспортирована из файла
- Результаты сохраняются в директории `./pythagora_tests/`
- Использует Jest в качестве фреймворка для тестов

---

## 2. РАСШИРЕНИЕ СУЩЕСТВУЮЩИХ ТЕСТОВ (Expand Unit Tests)

### 2.1. API Метод: `UnitTestsExpand.runProcessing()`

**Назначение**: Расширение существующих юнит-тестов для увеличения покрытия кода и добавления новых edge cases.

**Файл**: `src/helpers/unitTestsExpand.js:16-30`

**Описание**: Этот метод анализирует существующие тесты и генерирует дополнительные тестовые случаи, чтобы улучшить покрытие кода и протестировать больше граничных случаев. Использует GPT-4 для интеллектуального расширения тестового набора.

**Входные параметры**:

```javascript
const unitTestsExpand = new UnitTestsExpand(
    {
        pathToProcess: args.path,        // Путь к файлу или директории с тестами
        pythagoraRoot: args.pythagora_root, // Корневая директория Pythagora
        force: args.force                // Перезаписать существующие расширенные тесты
    },
    Api,                                 // API инстанс для обращения к GPT-4
    {
        isSaveTests: true,               // Сохранять ли сгенерированные тесты
        screen,                          // UI screen для отображения прогресса
        spinner,                         // UI spinner
        scrollableContent                // UI scrollable content
    }
);
```

**Выходные данные**:

```javascript
const {errors, skippedFiles, testsGenerated} = await unitTestsExpand.runProcessing();
// errors - массив ошибок, возникших при расширении
// skippedFiles - массив файлов, которые были пропущены
// testsGenerated - массив сгенерированных дополнительных тестов
```

**Что отправляется в промпт GPT-4**:
1. Существующий набор тестов
2. Исходный код функции, которая тестируется
3. Текущее покрытие кода (code coverage)
4. Запрос на генерацию дополнительных тестов для непокрытых путей и edge cases

**Команда для запуска**:

```bash
# Для одного файла с тестами
npx pythagora --expand-unit-tests --path <PATH_TO_YOUR_TEST_SUITE>

# Для директории с тестами
npx pythagora --expand-unit-tests --path ./path/to/tests/folder/
```

**Пример вызова в коде**:

```javascript
// src/helpers/unitTestsExpand.js:16-30
const unitTestsExpand = new UnitTestsExpand(
    {
        pathToProcess: args.path,
        pythagoraRoot: args.pythagora_root,
        force: args.force
    },
    Api,
    {
        isSaveTests: true,
        screen,
        spinner,
        scrollableContent
    }
);
const {errors, skippedFiles, testsGenerated} = await unitTestsExpand.runProcessing();
```

**Особенности**:
- Работает с существующими тестами, расширяя их
- Помогает увеличить code coverage
- Генерирует тесты для edge cases
- Может обрабатывать как отдельные файлы, так и целые директории с тестами

**Типичный use case**:
Когда у вас уже есть базовые тесты, но вы хотите:
- Увеличить покрытие кода
- Добавить тесты для граничных случаев
- Протестировать больше сценариев использования

---

## 3. ЭКСПОРТ ТЕСТОВ В JEST ФОРМАТ (Jest Export)

Pythagora может захватывать реальные API запросы во время выполнения приложения и конвертировать их в Jest тесты с помощью GPT-4. Этот процесс включает несколько API методов.

### 3.1. API Метод: `Api.isEligibleForExport(test)`

**Назначение**: Проверка, подходит ли тест для экспорта в Jest формат (проверка лимита токенов GPT-4).

**Файл**: `src/commands/export.js:53`

**Описание**: GPT-4 8k имеет ограничение на количество токенов в запросе. Этот метод проверяет, не превышает ли размер теста лимит токенов модели.

**Входные данные**:

```javascript
let test = convertOldTestForGPT(originalTest);
const isEligible = await Api.isEligibleForExport(test);
```

**Структура `test` объекта** (после преобразования через `convertOldTestForGPT`):

```javascript
{
    testId: "...",              // ID теста
    endpoint: "...",            // API endpoint
    method: "GET/POST/...",     // HTTP метод
    statusCode: 200,            // Код ответа
    response: {...},            // Тело ответа
    reqQuery: {...},            // Query параметры запроса
    reqBody: {...},             // Тело запроса
    mongoQueries: [{            // MongoDB запросы
        collection: "...",
        mongoOperation: "...",
        mongoQuery: {...},
        mongoOptions: {...},
        preQueryDocs: [...],
        postQueryDocs: [...],
        mongoResponse: [...]
    }]
}
```

**Выходные данные**:
- `true` - тест подходит для экспорта (не превышает лимит токенов)
- `false` - тест слишком большой для GPT-4 8k

**Пример вызова**:

```javascript
// src/commands/export.js:52-57
let test = convertOldTestForGPT(originalTest);
const isEligible = await Api.isEligibleForExport(test);

if (isEligible) {
    await exportTest(originalTest, exportsMetadata);
} else {
    testEligibleForExportLog(originalTest.endpoint, originalTest.id, isEligible);
}
```

---

### 3.2. API Метод: `Api.getJestTest(test)`

**Назначение**: Конвертация захваченного теста в Jest формат с помощью GPT-4.

**Файл**: `src/helpers/exports.js:104`

**Описание**: Основной метод для конвертации. Отправляет данные захваченного теста в GPT-4, который генерирует полноценный Jest тест, включая setup, mocks, assertions и cleanup.

**Входные данные**:

```javascript
let test = convertOldTestForGPT(originalTest);
let jestTest = await Api.getJestTest(test);
```

**Что отправляется в промпт GPT-4**:
1. **Endpoint информация**: метод, путь, query параметры
2. **Request данные**: headers, body, query parameters
3. **Response данные**: status code, body
4. **MongoDB запросы**: все запросы к базе данных, которые были выполнены во время обработки запроса
   - Состояние БД до запроса (`preQueryDocs`)
   - Сам MongoDB запрос и опции
   - Результат запроса (`mongoResponse`)
   - Состояние БД после запроса (`postQueryDocs`)

**Выходные данные**:
Полностью сформированный Jest тест в виде строки с кодом:

```javascript
const request = require('supertest');
const { MongoMemoryServer } = require('mongodb-memory-server');

describe('API Test - POST /endpoint', () => {
    // ... setup, mocks, test cases, cleanup ...
});
```

**Пример вызова**:

```javascript
// src/helpers/exports.js:98-114
async function exportTest(originalTest, exportsMetadata) {
    const { apiUrl, apiKey, apiKeyType } = getApiConfig();
    const Api = new API(apiUrl, apiKey, apiKeyType);

    testExportStartedLog();
    let test = convertOldTestForGPT(originalTest);
    let jestTest = await Api.getJestTest(test);
    let testName = await Api.getJestTestName(jestTest, Object.values(exportsMetadata).map(obj => obj.testName));

    if (!jestTest && !testName) return console.error('There was issue with getting GPT response. Make sure you have access to GPT4 with your API key.');

    fs.writeFileSync(`./${EXPORTED_TESTS_DATA_DIR}/${testName.replace('.test.js', '.json')}`, JSON.stringify(test.mongoQueries, null, 2));
    fs.writeFileSync(`./${EXPORTED_TESTS_DIR}/${testName}`, jestTest.replace(test.testId, testName));

    testExported(testName);
    saveExportJson(exportsMetadata, originalTest, testName);
}
```

**Особенности**:
- Генерирует полный Jest тест со всеми необходимыми imports
- Создает моки для MongoDB запросов
- Включает assertions для проверки ответа и состояния БД
- Обрабатывает сложные сценарии с множественными DB операциями

---

### 3.3. API Метод: `Api.getJestTestName(jestTest, existingNames)`

**Назначение**: Генерация читаемого и уникального имени для Jest теста.

**Файл**: `src/helpers/exports.js:105`

**Описание**: GPT-4 анализирует содержимое теста и генерирует осмысленное имя файла, которое отражает, что именно тестируется. Также проверяет, что имя уникально среди существующих тестов.

**Входные данные**:

```javascript
let jestTest = await Api.getJestTest(test); // Код Jest теста
let existingNames = Object.values(exportsMetadata).map(obj => obj.testName); // Существующие имена
let testName = await Api.getJestTestName(jestTest, existingNames);
```

**Что отправляется в промпт GPT-4**:
1. Полный код Jest теста
2. Список существующих имен тестов (для избежания дубликатов)

**Выходные данные**:
Строка с именем файла теста, например:
- `post-user-login-success.test.js`
- `get-products-with-filters.test.js`
- `delete-user-unauthorized.test.js`

**Пример вызова**:

```javascript
// src/helpers/exports.js:105
let testName = await Api.getJestTestName(
    jestTest,
    Object.values(exportsMetadata).map(obj => obj.testName)
);
```

**Особенности**:
- Генерирует семантически значимые имена
- Избегает дубликатов
- Следует конвенциям именования Jest тестов
- Имя отражает HTTP метод, endpoint и сценарий

---

### 3.4. API Метод: `Api.getJestAuthFunction(mongoQueries, requestBody, endpointPath)`

**Назначение**: Генерация функции аутентификации для Jest тестов на основе захваченного login теста.

**Файл**: `src/helpers/exports.js:66`

**Описание**: Многие API тесты требуют аутентификации. Этот метод анализирует, как происходит login в приложении, и генерирует вспомогательную функцию, которая будет использоваться во всех других тестах для получения auth токена.

**Входные данные**:

```javascript
let loginData = pythagoraMetadata.exportRequirements.login;
let code = await Api.getJestAuthFunction(
    loginData.mongoQueriesArray,  // MongoDB запросы во время login
    loginData.requestBody,         // Тело login запроса
    loginData.endpointPath         // Путь к login endpoint
);
```

**Что отправляется в промпт GPT-4**:
1. **Login endpoint**: путь к эндпоинту аутентификации
2. **Request body**: данные, отправляемые для login (email, password и т.д.)
3. **MongoDB запросы**: все DB операции, которые происходят во время аутентификации
   - Создание/поиск пользователя
   - Генерация токенов
   - Сохранение сессии

**Выходные данные**:
JavaScript код функции аутентификации, например:

```javascript
const request = require('supertest');

async function authenticate(app) {
    // Setup user in database
    // Make login request
    // Extract and return token
    return token;
}

module.exports = { authenticate };
```

**Пример вызова**:

```javascript
// src/helpers/exports.js:38-68
async function configureAuthFile(generatedTests) {
    const { apiUrl, apiKey, apiKeyType } = getApiConfig();
    const Api = new API(apiUrl, apiKey, apiKeyType);

    let pythagoraMetadata = require(`../${SRC_TO_ROOT}.pythagora/${METADATA_FILENAME}`);
    let loginPath = _.get(pythagoraMetadata, 'exportRequirements.login.endpointPath');
    let loginRequestBody = _.get(pythagoraMetadata, 'exportRequirements.login.requestBody');
    let loginMongoQueries = _.get(pythagoraMetadata, 'exportRequirements.login.mongoQueriesArray');

    if (!loginPath) {
        enterLoginRouteLog();
        process.exit(1);
    }

    if (!loginRequestBody || !loginMongoQueries) {
        let loginTest = generatedTests.find(t => t.endpoint === loginPath && t.method !== 'OPTIONS');
        if (loginTest) {
            _.set(pythagoraMetadata, 'exportRequirements.login.mongoQueriesArray', loginTest.intermediateData.filter(d => d.type === 'mongodb'));
            _.set(pythagoraMetadata, 'exportRequirements.login.requestBody', loginTest.body);
            updateMetadata(pythagoraMetadata);
        } else {
            pleaseCaptureLoginTestLog(loginPath);
            process.exit(1);
        }
    }

    let loginData = pythagoraMetadata.exportRequirements.login;
    let code = await Api.getJestAuthFunction(loginData.mongoQueriesArray, loginData.requestBody, loginData.endpointPath);

    fs.writeFileSync(path.resolve(args.pythagora_root, EXPORTED_TESTS_DIR, 'auth.js'), code);
}
```

**Куда сохраняется результат**:
- Файл: `./pythagora_tests/auth.js`
- Используется во всех экспортированных тестах

**Особенности**:
- Анализирует реальный процесс аутентификации приложения
- Генерирует переиспользуемую функцию для всех тестов
- Обрабатывает setup данных пользователя в БД
- Извлекает и возвращает токен/cookie для последующих запросов

---

## 4. REVIEW ИЗМЕНЕНИЙ (Test Review)

**Файл**: `src/scripts/review.js`

**Описание**: Функционал review не использует AI/GPT, но является важной частью workflow. Когда Pythagora запускается в режиме тестирования и обнаруживает изменения в поведении приложения (например, изменились данные в response или MongoDB запросы), эти изменения сохраняются для review.

### 4.1. Интерактивный Review Процесс

**Назначение**: Позволяет разработчику просматривать и принимать решения об изменениях в тестах.

**Что сохраняется для review**:
- Файл: `.pythagora/reviewData.json`
- Содержит все обнаруженные изменения между захваченными тестами и текущим выполнением

**Типы изменений, которые детектируются**:

1. **Изменения в Response данных**
   - Изменился status code
   - Изменилась структура или содержимое response body

2. **Изменения в MongoDB запросах**
   - Новые запросы, которых не было в оригинальном тесте (`mongoQueryNotFound`)
   - Запросы, которые не были выполнены (`mongoNotExecuted`)
   - Изменения в результатах запросов (`mongoResDiff`)

3. **Изменения в параметрах запроса**
   - Query параметры
   - Request body

**Доступные действия при review**:

```javascript
const changesActions = {
    A: {
        fn: 'acceptChange',
        msg: 'accept changes'        // Принять изменения и обновить тест
    },
    D: {
        fn: 'deleteChange',
        msg: 'delete test'           // Удалить тест полностью
    },
    S: {
        fn: 'skipChange',
        msg: 'skip test'             // Пропустить review этого теста
    },
    R: {
        fn: 'getRunCmd',
        msg: 'get command to run this test'  // Получить команду для запуска теста
    },
    Q: {
        fn: 'quitReview',
        msg: 'quit review'           // Выйти из review
    }
};
```

**Пример структуры изменения**:

```javascript
{
    id: "test-id-123",
    filename: "api-users.json",
    endpoint: "/api/users",
    method: "POST",
    statusCode: {
        capture: 200,    // Оригинальное значение в захваченном тесте
        test: 201        // Новое значение при текущем выполнении
    },
    responseData: {
        capture: {...},
        test: {...}
    },
    intermediateData: {  // MongoDB запросы
        capture: [...],
        test: [...]
    },
    errors: [{
        type: "mongoResultDifferent",
        message: "..."
    }]
}
```

**Команда для запуска review**:

```bash
npx pythagora --review
```

**Процесс работы**:

1. Система читает файл `.pythagora/reviewData.json`
2. Для каждого изменения:
   - Отображает оригинальные и новые значения
   - Показывает различия в MongoDB запросах
   - Предлагает выбрать действие (A/D/S/R/Q)
3. После принятия изменения (A):
   - Обновляет соответствующий тест файл
   - Удаляет изменение из reviewData.json
4. После удаления теста (D):
   - Удаляет тест из файла
   - Удаляет изменение из reviewData.json

**Особенности**:
- Не использует AI/GPT
- Полностью интерактивный процесс
- Помогает контролировать изменения в поведении приложения
- Предотвращает автоматическое принятие breaking changes

**Типичный use case**:
Когда вы изменили код приложения и хотите:
- Проверить, какие тесты были затронуты
- Решить, являются ли изменения ожидаемыми
- Обновить тесты в соответствии с новым поведением

---

## 5. ВСПОМОГАТЕЛЬНАЯ ФУНКЦИЯ: convertOldTestForGPT()

**Назначение**: Преобразование внутреннего формата теста в формат, подходящий для отправки в GPT-4.

**Файл**: `src/utils/legacy.js`

**Описание**: Эта функция не является AI промптом, но критически важна для подготовки данных перед отправкой в GPT-4. Она преобразует поля и структуру данных для обеспечения совместимости.

**Преобразования, которые выполняются**:

```javascript
function convertOldTestForGPT(originalTest) {
    let test = _.cloneDeep(originalTest);

    // Удаление ненужных полей
    delete test.url;
    delete test.trace;
    delete test.traceId;
    delete test.asyncStore;
    delete test.traceLegacy;
    delete test.createdAt;
    delete test.params;

    // Переименование полей
    test.testId = test.id;              // id -> testId
    delete test.id;

    test.response = JSON.parse(test.responseData);  // responseData -> response (parsed)
    delete test.responseData;

    test.mongoQueryNum = test.mongoQueriesCapture;  // mongoQueriesCapture -> mongoQueryNum
    delete test.mongoQueriesCapture;

    test.reqQuery = test.query;         // query -> reqQuery
    delete test.query;

    test.reqBody = test.body;           // body -> reqBody
    delete test.body;

    // Преобразование MongoDB запросов
    test.mongoQueries = test.intermediateData.map((item) => {
        item.preQueryDocs = item.preQueryRes;      // preQueryRes -> preQueryDocs
        delete item.preQueryRes;

        item.postQueryDocs = item.postQueryRes;    // postQueryRes -> postQueryDocs
        delete item.postQueryRes;

        item.mongoResponse = item.mongoRes;        // mongoRes -> mongoResponse
        delete item.mongoRes;

        item.mongoQuery = item.query;              // query -> mongoQuery
        delete item.query;

        item.mongoOptions = item.options;          // options -> mongoOptions
        delete item.options;

        item.mongoOperation = item.op;             // op -> mongoOperation
        delete item.op;

        return item;
    });
    delete test.intermediateData;

    return test;
}
```

**Когда используется**:
- Перед вызовом `Api.getJestTest()`
- Перед вызовом `Api.isEligibleForExport()`
- В любом месте, где нужно отправить тест данные в GPT-4

**Важность**:
Эта функция обеспечивает обратную совместимость и правильный формат данных для промптов GPT-4 на сервере Pythagora.

---

## 6. РЕЗЮМЕ: ВСЕ AI ПРОМПТЫ В СИСТЕМЕ

### Промпты для генерации тестов:
1. **UnitTests.runProcessing()** - Генерация юнит-тестов для функций
2. **UnitTestsExpand.runProcessing()** - Расширение существующих тестов

### Промпты для экспорта в Jest:
3. **Api.getJestTest()** - Конвертация захваченного теста в Jest
4. **Api.getJestTestName()** - Генерация имени для Jest теста
5. **Api.getJestAuthFunction()** - Генерация auth функции
6. **Api.isEligibleForExport()** - Проверка лимита токенов (не промпт, но проверка)

### Местонахождение фактических промптов:
Все фактические промпты находятся в репозитории Pythagora API:
- https://github.com/Pythagora-io/api/tree/main/prompts

### Модель AI:
- Используется: **GPT-4** (для всех операций)
- Ограничение: **8k tokens** для операций экспорта

### Типы данных, отправляемых в промпты:
1. **Исходный код** - функции JavaScript/TypeScript
2. **Зависимости** - импорты и вызываемые функции
3. **Тесты** - существующий код тестов
4. **API данные** - запросы, ответы, headers
5. **MongoDB данные** - запросы, состояние БД до/после
6. **Метаданные** - endpoints, методы, status codes

---

## 7. РЕКОМЕНДАЦИИ ПО ИСПОЛЬЗОВАНИЮ

### Для лучших результатов генерации юнит-тестов:
1. Функции должны быть экспортированы
2. Лучше всего работает для standalone/helper функций
3. Избегайте сложных зависимостей в функциях

### Для экспорта в Jest:
1. Убедитесь, что тест не превышает 8k tokens
2. Захватите login endpoint для генерации auth функции
3. Проверяйте сгенерированные тесты перед коммитом

### Для расширения тестов:
1. Используйте после создания базовых тестов
2. Помогает улучшить code coverage
3. Генерирует тесты для edge cases

---

