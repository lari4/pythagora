# Документация Пайплайнов Pythagora Agent

## Обзор

Этот документ описывает все возможные пайплайны (workflows) работы Pythagora Agent. Каждый пайплайн показывает, какие промпты вызываются, в каком порядке, и какие данные передаются между этапами.

## Содержание

1. [Пайплайн: Генерация Юнит-Тестов](#1-пайплайн-генерация-юнит-тестов)
2. [Пайплайн: Расширение Тестов](#2-пайплайн-расширение-тестов)
3. [Пайплайн: Экспорт в Jest](#3-пайплайн-экспорт-в-jest)
4. [Пайплайн: Review Изменений](#4-пайплайн-review-изменений)

---

## 1. ПАЙПЛАЙН: ГЕНЕРАЦИЯ ЮНИТ-ТЕСТОВ

**Команда**: `npx pythagora --unit-tests --func <FUNCTION_NAME>` или `--path <PATH>`

**Файл entry point**: `src/scripts/unit.js`

**Основной handler**: `src/helpers/unitTests.js`

### 1.1. ASCII Диаграмма Потока

```
┌─────────────────────────────────────────────────────────────────┐
│                    ПОЛЬЗОВАТЕЛЬ                                 │
│  npx pythagora --unit-tests --func myFunction                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              src/scripts/unit.js                                │
│  • Читает аргументы командной строки                            │
│  • Вызывает generateTestsForDirectory(args)                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         src/helpers/unitTests.js :: generateTestsForDirectory() │
│  1. Получить API конфигурацию (OpenAI/Pythagora key)           │
│  2. Инициализировать UI (screen, spinner, scrollableContent)    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              Создание UnitTests объекта                         │
│  const unitTests = new UnitTests(                               │
│    { pathToProcess, pythagoraRoot, funcName, force },           │
│    Api,                                                          │
│    { isSaveTests, screen, spinner, scrollableContent }          │
│  )                                                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│       unitTests.runProcessing() - ОСНОВНАЯ ОБРАБОТКА            │
│  [Этот метод из пакета @pythagora.io/js-code-processing]       │
│                                                                  │
│  Шаги (внутри пакета):                                          │
│  1. AST парсинг файлов                                          │
│     └─> Найти целевую функцию                                   │
│     └─> Найти все зависимости (вызываемые функции)             │
│     └─> Извлечь импорты и exports                               │
│                                                                  │
│  2. Подготовка данных для GPT-4                                 │
│     └─> Собрать код функции                                     │
│     └─> Собрать код всех зависимостей                           │
│     └─> Подготовить контекст использования                      │
│                                                                  │
│  3. Отправка запроса в API                                      │
│     └─> POST request to Pythagora API Server                    │
│         или OpenAI API (в зависимости от apiKeyType)            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PYTHAGORA API SERVER                         │
│                    (или OpenAI API)                             │
│                                                                  │
│  Входные данные (JSON):                                         │
│  {                                                               │
│    "function": "код целевой функции",                           │
│    "dependencies": ["код зависимости 1", "код зависимости 2"],  │
│    "imports": ["import statements"],                            │
│    "context": "дополнительный контекст"                         │
│  }                                                               │
│                                                                  │
│  ╔══════════════════════════════════════════════════╗           │
│  ║          ПРОМПТ GPT-4 (на сервере)              ║           │
│  ║  Сгенерируй Jest юнит-тесты для функции:        ║           │
│  ║  - Покрой различные сценарии использования       ║           │
│  ║  - Добавь edge cases                             ║           │
│  ║  - Создай моки для зависимостей                  ║           │
│  ║  - Используй Jest синтаксис (describe, it, etc) ║           │
│  ╚══════════════════════════════════════════════════╝           │
│                                                                  │
│  Выходные данные (string):                                      │
│  - Полный Jest тестовый файл с кодом                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│           ОБРАБОТКА РЕЗУЛЬТАТА (в runProcessing)                │
│  • Получить сгенерированный код тестов                          │
│  • Сохранить в файл: ./pythagora_tests/unit/<path>/<file>.test.js│
│  • Обновить UI (показать прогресс)                              │
│  • Добавить результат в массив testsGenerated                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              ВОЗВРАТ РЕЗУЛЬТАТОВ                                │
│  const {errors, skippedFiles, testsGenerated} =                 │
│        await unitTests.runProcessing()                          │
│                                                                  │
│  • errors: [] - ошибки при генерации                            │
│  • skippedFiles: [] - пропущенные файлы                         │
│  • testsGenerated: [{testPath, ...}] - созданные тесты          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│               ВЫВОД РЕЗУЛЬТАТОВ ПОЛЬЗОВАТЕЛЮ                    │
│  • Если errors.length > 0:                                      │
│    └─> Сохранить в errorLogs.log                                │
│    └─> Вывести путь к логам                                     │
│  • Если skippedFiles.length > 0:                                │
│    └─> Показать сообщение о пропущенных файлах                  │
│  • Если testsGenerated.length > 0:                              │
│    └─> Вывести пути к сгенерированным тестам                    │
│    └─> "X unit tests generated!"                                │
│  • Выход из процесса (process.exit(0))                          │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Детальное Описание Этапов

#### Этап 1: Инициализация
- **Файл**: `src/scripts/unit.js`
- **Действие**: Парсинг аргументов командной строки
- **Данные**: `args.path`, `args.func`, `args.force`, `args.pythagora_root`

#### Этап 2: Получение API конфигурации
- **Файл**: `src/helpers/api.js:4-10`
- **Функция**: `getApiConfig()`
- **Выход**:
  ```javascript
  {
    apiUrl: string,      // URL Pythagora API или OpenAI
    apiKey: string,      // API ключ
    apiKeyType: string   // 'openai' или 'pythagora'
  }
  ```

#### Этап 3: Создание UnitTests инстанса
- **Пакет**: `@pythagora.io/js-code-processing`
- **Класс**: `UnitTests`
- **Конфигурация**:
  ```javascript
  {
    pathToProcess: string,     // Путь к файлу/директории
    pythagoraRoot: string,     // Корневая директория проекта
    funcName?: string,         // Имя конкретной функции (опционально)
    force: boolean            // Перезаписать существующие тесты
  }
  ```

#### Этап 4: AST Парсинг (внутри runProcessing)
- **Инструмент**: Abstract Syntax Tree парсер
- **Цель**: Найти функцию и все её зависимости
- **Выход**: Дерево зависимостей функции

#### Этап 5: Подготовка данных для GPT-4
- **Что собирается**:
  1. Исходный код целевой функции
  2. Код всех вызываемых функций (зависимости)
  3. Import statements
  4. Export statements
  5. Контекст использования

#### Этап 6: Запрос к GPT-4
- **API Endpoint**: Pythagora API Server или OpenAI
- **Модель**: GPT-4
- **Промпт**: Находится на сервере Pythagora API
- **Формат запроса**: JSON с кодом и контекстом

#### Этап 7: Получение и сохранение результата
- **Формат ответа**: JavaScript код Jest тестов
- **Путь сохранения**: `./pythagora_tests/unit/<original-file-path>/<file>.test.js`
- **Структура теста**:
  ```javascript
  const functionToTest = require('../../src/path/to/file');

  describe('functionName', () => {
    it('should handle basic case', () => {
      // test code
    });

    it('should handle edge case', () => {
      // test code
    });
  });
  ```

#### Этап 8: Вывод результатов
- **Успех**: Показать пути к сгенерированным тестам
- **Ошибки**: Сохранить в `errorLogs.log` и показать путь
- **Пропущенные**: Показать количество и причину (файлы уже существуют)

### 1.3. Структура Данных На Каждом Этапе

**Вход (аргументы командной строки)**:
```javascript
{
  path: './src/utils/helper.js',
  func: 'calculateTotal',
  force: false,
  pythagora_root: './'
}
```

**После AST парсинга**:
```javascript
{
  targetFunction: {
    name: 'calculateTotal',
    code: 'function calculateTotal(items) { ... }',
    params: ['items'],
    location: { file: '...', line: 15 }
  },
  dependencies: [
    {
      name: 'formatCurrency',
      code: 'function formatCurrency(amount) { ... }'
    },
    {
      name: 'validateItems',
      code: 'function validateItems(items) { ... }'
    }
  ],
  imports: [
    "const _ = require('lodash');"
  ]
}
```

**Запрос к GPT-4**:
```javascript
{
  function: "function calculateTotal(items) { ... полный код ... }",
  dependencies: [
    "function formatCurrency(amount) { ... }",
    "function validateItems(items) { ... }"
  ],
  imports: ["const _ = require('lodash');"],
  context: "This function calculates total price from items array"
}
```

**Ответ от GPT-4**:
```javascript
// Строка с полным кодом теста
`const { calculateTotal } = require('../../src/utils/helper');

describe('calculateTotal', () => {
  it('should calculate total for valid items', () => {
    const items = [{ price: 10 }, { price: 20 }];
    expect(calculateTotal(items)).toBe(30);
  });

  it('should handle empty array', () => {
    expect(calculateTotal([])).toBe(0);
  });

  // ... более тестов
});`
```

**Финальный результат**:
```javascript
{
  errors: [],
  skippedFiles: [],
  testsGenerated: [
    {
      testPath: './pythagora_tests/unit/src/utils/helper.test.js',
      functionName: 'calculateTotal',
      testsCount: 5
    }
  ]
}
```

---

## 2. ПАЙПЛАЙН: РАСШИРЕНИЕ ТЕСТОВ

**Команда**: `npx pythagora --expand-unit-tests --path <PATH_TO_TEST_SUITE>`

**Файл entry point**: `src/scripts/expandUnit.js`

**Основной handler**: `src/helpers/unitTestsExpand.js`

### 2.1. ASCII Диаграмма Потока

```
┌─────────────────────────────────────────────────────────────────┐
│                    ПОЛЬЗОВАТЕЛЬ                                 │
│  npx pythagora --expand-unit-tests --path ./tests/helper.test.js│
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              src/scripts/expandUnit.js                          │
│  • Читает аргументы (path к существующим тестам)                │
│  • Вызывает expandTestsForDirectory(args)                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│    src/helpers/unitTestsExpand.js :: expandTestsForDirectory()  │
│  1. Получить API конфигурацию                                   │
│  2. Инициализировать UI (screen, spinner)                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│           Создание UnitTestsExpand объекта                      │
│  const unitTestsExpand = new UnitTestsExpand(                   │
│    { pathToProcess, pythagoraRoot, force },                     │
│    Api,                                                          │
│    { isSaveTests, screen, spinner, scrollableContent }          │
│  )                                                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│     unitTestsExpand.runProcessing() - ОСНОВНАЯ ОБРАБОТКА        │
│  [Из пакета @pythagora.io/js-code-processing]                  │
│                                                                  │
│  Шаги:                                                           │
│  1. Чтение существующих тестов                                  │
│     └─> Парсинг тестового файла                                 │
│     └─> Извлечение всех test cases                              │
│     └─> Анализ покрытия кода                                    │
│                                                                  │
│  2. Чтение исходного кода тестируемой функции                   │
│     └─> Найти файл с исходным кодом                             │
│     └─> Извлечь функцию, которая тестируется                    │
│     └─> Найти зависимости                                       │
│                                                                  │
│  3. Анализ покрытия                                             │
│     └─> Определить, какие пути кода не покрыты                  │
│     └─> Определить, какие edge cases отсутствуют                │
│     └─> Определить, какие сценарии можно добавить               │
│                                                                  │
│  4. Подготовка данных для GPT-4                                 │
│     └─> Существующие тесты (код)                                │
│     └─> Исходный код функции                                    │
│     └─> Анализ непокрытых путей                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PYTHAGORA API SERVER                         │
│                    (или OpenAI API)                             │
│                                                                  │
│  Входные данные (JSON):                                         │
│  {                                                               │
│    "existingTests": "код существующих тестов",                  │
│    "functionCode": "код тестируемой функции",                   │
│    "uncoveredPaths": ["путь 1", "путь 2", ...],                 │
│    "dependencies": ["код зависимостей"],                        │
│    "coverageReport": { ... }                                    │
│  }                                                               │
│                                                                  │
│  ╔══════════════════════════════════════════════════╗           │
│  ║          ПРОМПТ GPT-4 (на сервере)              ║           │
│  ║  Расширь существующий набор тестов:              ║           │
│  ║  - Проанализируй текущие тесты                   ║           │
│  ║  - Определи непокрытые пути кода                 ║           │
│  ║  - Добавь тесты для edge cases                   ║           │
│  ║  - Добавь тесты для новых сценариев              ║           │
│  ║  - Сохрани существующий стиль тестов             ║           │
│  ╚══════════════════════════════════════════════════╝           │
│                                                                  │
│  Выходные данные (string):                                      │
│  - Дополнительные test cases (только новые it() блоки)          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│           ОБРАБОТКА РЕЗУЛЬТАТА (в runProcessing)                │
│  • Получить новые test cases от GPT-4                           │
│  • Объединить с существующими тестами                           │
│  • Сохранить расширенный файл (или создать новый)               │
│  • Обновить UI (показать прогресс)                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              ВОЗВРАТ РЕЗУЛЬТАТОВ                                │
│  const {errors, skippedFiles, testsGenerated} =                 │
│        await unitTestsExpand.runProcessing()                    │
│                                                                  │
│  • errors: [] - ошибки при расширении                           │
│  • skippedFiles: [] - пропущенные файлы                         │
│  • testsGenerated: [{testPath, ...}] - расширенные тесты        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│               ВЫВОД РЕЗУЛЬТАТОВ ПОЛЬЗОВАТЕЛЮ                    │
│  • Если errors.length > 0:                                      │
│    └─> Сохранить в errorLogs.log                                │
│  • Если testsGenerated.length > 0:                              │
│    └─> Вывести пути к расширенным тестам                        │
│    └─> "X unit tests generated!" (дополнительные тесты)         │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2. Сравнение с Генерацией Тестов

| Аспект | Генерация Тестов | Расширение Тестов |
|--------|------------------|-------------------|
| **Вход** | Исходный код функции | Существующие тесты + исходный код |
| **Цель** | Создать тесты с нуля | Добавить больше тестов |
| **Анализ** | AST парсинг функции | Анализ покрытия + AST |
| **Выход** | Полный тестовый файл | Дополнительные test cases |
| **Когда использовать** | Нет тестов | Есть базовые тесты |

### 2.3. Структура Данных На Каждом Этапе

**Вход (существующий тест файл)**:
```javascript
// ./tests/helper.test.js
describe('calculateTotal', () => {
  it('should calculate total for valid items', () => {
    const items = [{ price: 10 }, { price: 20 }];
    expect(calculateTotal(items)).toBe(30);
  });
});
```

**После анализа**:
```javascript
{
  existingTests: {
    file: './tests/helper.test.js',
    testCases: [
      {
        description: 'should calculate total for valid items',
        code: '...',
        coveredPaths: ['basic-case']
      }
    ],
    totalTests: 1
  },
  sourceFunction: {
    name: 'calculateTotal',
    code: 'function calculateTotal(items) { ... }',
    branches: ['empty-array', 'null-input', 'invalid-items', 'basic-case'],
    coveredBranches: ['basic-case'],
    uncoveredBranches: ['empty-array', 'null-input', 'invalid-items']
  }
}
```

**Запрос к GPT-4**:
```javascript
{
  existingTests: "describe('calculateTotal', () => { ... })",
  functionCode: "function calculateTotal(items) { ... }",
  uncoveredPaths: ['empty-array', 'null-input', 'invalid-items'],
  coverageReport: {
    totalBranches: 4,
    coveredBranches: 1,
    coveragePercent: 25
  }
}
```

**Ответ от GPT-4** (новые test cases):
```javascript
`
  it('should return 0 for empty array', () => {
    expect(calculateTotal([])).toBe(0);
  });

  it('should throw error for null input', () => {
    expect(() => calculateTotal(null)).toThrow();
  });

  it('should handle items with invalid prices', () => {
    const items = [{ price: 10 }, { price: 'invalid' }];
    expect(calculateTotal(items)).toBe(10);
  });
`
```

**Финальный результат** (объединённый файл):
```javascript
describe('calculateTotal', () => {
  // Существующий тест
  it('should calculate total for valid items', () => {
    const items = [{ price: 10 }, { price: 20 }];
    expect(calculateTotal(items)).toBe(30);
  });

  // НОВЫЕ ТЕСТЫ от GPT-4
  it('should return 0 for empty array', () => {
    expect(calculateTotal([])).toBe(0);
  });

  it('should throw error for null input', () => {
    expect(() => calculateTotal(null)).toThrow();
  });

  it('should handle items with invalid prices', () => {
    const items = [{ price: 10 }, { price: 'invalid' }];
    expect(calculateTotal(items)).toBe(10);
  });
});
```

### 2.4. Ключевые Отличия в Промпте

**Генерация тестов** (промпт):
- "Создай тесты для этой функции"
- Фокус на полном покрытии с нуля

**Расширение тестов** (промпт):
- "Вот существующие тесты: [код]"
- "Вот что НЕ покрыто: [список]"
- "Добавь тесты для непокрытых путей"
- "Сохрани стиль существующих тестов"
- Фокус на дополнении, а не замене

---

