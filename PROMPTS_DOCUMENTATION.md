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

