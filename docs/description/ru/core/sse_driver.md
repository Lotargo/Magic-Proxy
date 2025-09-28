# Техническая Документация: Модуль sse_driver.py

## 1. Краткое Резюме

`sse_driver.py` — это специализированный модуль, реализующий надежный и масштабируемый real-time коммуникационный мост между асинхронными фоновыми процессами и конечным клиентом. Он использует паттерн "Издатель-Подписчик" на базе Redis Pub/Sub для передачи событий и предоставляет их клиенту через стандартный протокол Server-Sent Events (SSE). Модуль инкапсулирует всю сложность управления жизненным циклом потоковой передачи, включая "рукопожатие" для синхронизации, обработку таймаутов и отказоустойчивое завершение соединения.

## 2. Архитектурная Роль и Решаемая Проблема

В распределенной архитектуре, где прием запроса (веб-сервер) и его обработка (фоновый воркер) разделены через Kafka, возникает проблема: как доставить результаты от воркера обратно клиенту, который ожидает ответа по одному HTTP-соединению?

`sse_driver.py` решает эту проблему, создавая непрямой, высокопроизводительный канал связи через Redis. Он позволяет воркерам публиковать события в централизованный брокер, а веб-серверам — подписываться на них и транслировать клиентам, эффективно "сшивая" асинхронный бэкенд с интерактивным фронтендом.

## 3. Ключевые Функции и Архитектурные Паттерны

### 3.1. Паттерн "Издатель-Подписчик" (Pub/Sub)

*   **`publish_event` (Издатель):** Эта функция предоставляет простой интерфейс для фоновых воркеров. Воркер вызывает ее, чтобы опубликовать JSON-сообщение вида `{"event_type": ..., "payload": ...}` в уникальный для каждой сессии канал Redis (`sse_session:{session_id}`).
*   **`sse_event_streamer` (Подписчик):** Эта асинхронная генераторная функция используется веб-сервером в `StreamingResponse` FastAPI. Она подписывается на тот же сессионный канал и асинхронно ожидает поступления сообщений.

### 3.2. Синхронизация через "Рукопожатие" (ACK Handshake)

`sse_event_streamer` не начинает немедленно слушать основной поток данных. Сначала он в течение 10 секунд ожидает специального события-подтверждения — `worker_ack`. Этот механизм решает критическую проблему состояния гонки (race condition) и делает коммуникацию значительно более надежной.

### 3.3. Управление Жизненным Циклом Потока

Модуль включает надежную обработку таймаутов, корректное завершение по событию `FinalAnswerStreamEnd` и отказоустойчивый парсинг сообщений, что делает систему устойчивой к распространенным проблемам в распределенных средах.

## 4. Интеграция с Клиентским Приложением (Frontend)

Основное преимущество архитектуры прокси заключается в том, что вся сложность потоковой передачи данных скрыта на сервере. Клиентскому приложению остается лишь подключиться к стандартному SSE-эндпоинту и обрабатывать структурированные JSON-события.

Ниже приведен пример на JavaScript, демонстрирующий, как легко можно интегрировать фронтенд-приложение.

### 4.1. Пример на JavaScript с использованием fetch

Современные браузеры позволяют обрабатывать потоковые ответы от POST-запросов с помощью `fetch` API. Это предпочтительный метод для взаимодействия с эндпоинтом `/v1/react/sessions`.

```javascript
// Функция для обработки потока событий от прокси
async function consumeProxyStream(requestBody) {
  try {
    const response = await fetch('/v1/react/sessions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(requestBody),
    });

    if (!response.ok) {
      // Обработка ошибок, если запрос не прошел (например, 4xx, 5xx)
      const errorData = await response.json();
      console.error('Error starting stream:', errorData.detail);
      // Здесь можно обновить UI, показав ошибку
      return;
    }

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) {
        console.log('Stream finished.');
        break;
      }

      // Декодируем полученный чанк и добавляем в буфер
      buffer += decoder.decode(value, { stream: true });

      // SSE-сообщения разделены двойным переводом строки
      const parts = buffer.split('\n\n');
      buffer = parts.pop(); // Последняя часть может быть неполной, оставляем ее в буфере

      for (const part of parts) {
        if (part.startsWith('data: ')) {
          const jsonData = part.substring(6);
          handleStreamEvent(jsonData);
        }
      }
    }
  } catch (error) {
    console.error('An error occurred while consuming the stream:', error);
  }
}

// Функция-диспетчер для обработки разных типов событий
function handleStreamEvent(jsonData) {
  try {
    const event = JSON.parse(jsonData);
    const eventType = event.event_type;
    const payload = event.payload;

    // Используем switch для обработки разных событий от бэкенда
    switch (eventType) {
      case 'worker_ack':
        console.log('Backend worker has acknowledged the session.');
        break;
      case 'AgentThoughtStream':
        // Добавляем символ к "мыслям" агента в UI
        document.getElementById('thought-display').textContent += payload.char;
        break;
      case 'AgentToolCallStart':
        console.log(`Agent is calling tool: ${payload.tool_name}`);
        break;
      case 'FinalAnswerStream':
        // Добавляем символ к финальному ответу в UI
        document.getElementById('answer-display').textContent += payload.char;
        break;
      case 'FinalAnswerStreamEnd':
        console.log('Final answer received. Closing connection.');
        // Здесь можно активировать кнопки, убрать лоадер и т.д.
        break;
      case 'error':
        console.error('An error event was received:', payload.error);
        break;
      default:
        // Неизвестный тип события
        break;
    }
  } catch (e) {
    console.error('Failed to parse JSON from stream:', jsonData);
  }
}

// Пример вызова
const userRequest = {
  user_query: "What is the capital of France and what is the weather there right now?",
  model_alias: "smart_agent",
  // ... другие параметры
};

consumeProxyStream(userRequest);
```

### 4.2. Ключевые шаги для клиента

1.  **Отправить POST-запрос:** Использовать `fetch` для отправки запроса на эндпоинт, который инициирует сессию.
2.  **Получить `ReadableStream`:** Работать с `response.body`, которое является потоком для чтения.
3.  **Декодировать и буферизовать:** Использовать `TextDecoder` для преобразования байтов в текст и накапливать данные в буфере до получения полного SSE-сообщения (заканчивается на `\n\n`).
4.  **Парсить JSON:** Каждое полученное сообщение (`data: {...}`) является JSON-строкой. Ее необходимо парсить с помощью `JSON.parse()`.
5.  **Обрабатывать события:** Использовать `switch` по полю `event_type` внутри JSON-объекта для выполнения различных действий в интерфейсе.
6.  **Завершить обработку:** Прекратить чтение потока при получении события `FinalAnswerStreamEnd` или по сигналу `done` от ридера.

Этот подход показывает, как `sse_driver` и архитектура прокси берут на себя всю сложную логику, предоставляя клиенту простой и структурированный поток событий.

## 5. Архитектурная Ценность и Бизнес-Эффект

*   **Полное Разделение Компонентов (Decoupling):** Веб-серверы и воркеры остаются полностью независимыми.
*   **Высокая Производительность:** Redis Pub/Sub обеспечивает передачу событий с минимальной задержкой.
*   **Исключительный User Experience:** Именно этот модуль обеспечивает "магию" интерактивности.
*   **Надежность:** Продуманная логика управления жизненным циклом потока делает систему устойчивой к проблемам в распределенных средах.