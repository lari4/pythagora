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

