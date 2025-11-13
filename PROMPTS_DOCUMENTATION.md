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

