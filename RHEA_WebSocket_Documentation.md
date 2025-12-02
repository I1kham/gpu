# Полная документация на WebSocket-протокол проекта RHEA

## Общая информация

RHEA - это система управления кофеваркой с графическим интерфейсом, предназначенная для международного использования. Система использует WebSocket-соединение для обмена данными между веб-интерфейсом и серверной частью (GPU/SMU).

Система поддерживает до 48 выборов (RHEA_NUM_MAX_SELECTIONS=48), каждый из которых может иметь различные параметры: доступность, цену, поддержку свежего молока и т.д.

## Подключение к WebSocket

### Параметры подключения
- **Адрес**: `ws://127.0.0.1:2280/`
- **Протокол**: `binary`
- **Тип соединения**: WebSocket с бинарными данными

## Процесс Hand Shake (установление соединения)

### 1. Установка соединения
При подключении к WebSocket-серверу на порту 2280 происходит следующая последовательность:

1. Клиент устанавливает соединение с сервером
2. Сервер может сразу отправить события (например, статус CPU) без предварительного запроса
3. Клиент отправляет запрос на получение ID-кода для идентификации

### 2. Запрос ID-кода при подключении
После установки соединения клиент отправляет запрос для получения уникального ID-кода:

```javascript
// Запрос ID-кода
var buffer = new Uint8Array(4);
buffer[0] = RHEA_CLIENT_INFO__API_VERSION;  // 0x01
buffer[1] = RHEA_CLIENT_INFO__APP_TYPE;     // 0x01
buffer[2] = RHEA_CLIENT_INFO__UNUSED2;      // 0x00
buffer[3] = RHEA_CLIENT_INFO__UNUSED3;      // 0x00
this.sendGPUCommand("I", buffer, 0, 0);
```

### 3. Идентификация после получения ID-кода
После получения ID-кода клиент идентифицируется в системе:

```javascript
// Отправка ID-кода для идентификации
var buffer = new Uint8Array(8);
buffer[0] = RHEA_CLIENT_INFO__API_VERSION;  // 0x01
buffer[1] = RHEA_CLIENT_INFO__APP_TYPE;     // 0x01
buffer[2] = RHEA_CLIENT_INFO__UNUSED2;      // 0x00
buffer[3] = RHEA_CLIENT_INFO__UNUSED3;      // 0x00
buffer[4] = this.idCode_0;  // MSB ID-кода
buffer[5] = this.idCode_1;
buffer[6] = this.idCode_2;
buffer[7] = this.idCode_3;  // LSB ID-кода
this.sendGPUCommand("W", buffer, 0, 0);
```

### 4. Запрос начального состояния
После успешной идентификации клиент запрашивает начальное состояние системы:

```javascript
// Запросы начального состояния
this.requestGPUEvent(RHEA_EVENT_SELECTION_AVAILABILITY_UPDATED);
this.requestGPUEvent(RHEA_EVENT_SELECTION_PRICES_UPDATED);
this.requestGPUEvent(RHEA_EVENT_CREDIT_UPDATED);
this.requestGPUEvent(RHEA_EVENT_CPU_STATUS);
```

## Формат команд

### Структура команды
Каждая команда имеет следующий формат:

```
[35] [commandChar] [requestID] [length_MSB] [length_LSB] [data...] [ck_MSB] [ck_LSB]
```

- `35` - ASCII символ '#' (0x23)
- `commandChar` - ASCII символ команды (например 'A', 'E', 'I', 'W', 'F')
- `requestID` - идентификатор запроса (0-255)
- `length_MSB/length_LSB` - длина данных в байтах (16-битное значение)
- `data...` - полезная нагрузка
- `ck_MSB/ck_LSB` - контрольная сумма (16-битная сумма всех байтов)

### Типы команд

| Команда | Код | Назначение |
|---------|-----|------------|
| 'A' | 65 | AJAX-запрос |
| 'E' | 69 | Событие/команда CPU |
| 'I' | 73 | Запрос ID-кода |
| 'W' | 87 | Идентификация клиента |
| 'F' | 70 | Файловый трансфер |

## Формат ответов

### AJAX-ответы
Ответы на AJAX-запросы имеют следующий формат:

```
[35] [65] [74] [requestID] [106] [97] [payload...]
```

- `35` - '#'
- `65` - 'A'
- `74` - 'J'
- `requestID` - идентификатор запроса
- `106` - 'j'
- `97` - 'a'
- `payload...` - полезная нагрузка в формате UTF-8

### События от GPU
События от GPU имеют следующий формат:

```
[35] [101] [86] [110] [eventTypeID] [eventSeqNum] [payloadLen_MSB] [payloadLen_LSB] [payload...]
```

- `35` - '#'
- `101` - 'e'
- `86` - 'V'
- `110` - 'n'
- `eventTypeID` - тип события (см. ниже)
- `eventSeqNum` - номер последовательности
- `payloadLen_MSB/payloadLen_LSB` - длина полезной нагрузки
- `payload...` - полезная нагрузка

## Типы событий

| ID | Константа | Назначение |
|----|-----------|------------|
| 70 | RHEA_EVENT_START_SELECTION_FORCE_JUG | Запуск выбора с принудительным использованием кувшина |
| 72 | RHEA_EVENT_SELECTION_PROGRESS | Прогресс выполнения выбора |
| 77 | RHEA_EVENT_SET_GUI_URL | Установка URL GUI |
| 71 | RHEA_EVENT_CPU_ACTIVATE_BUZZER | Активация зуммера CPU |
| (неизвестен) | RHEA_EVENT_START_SEL_ALREADY_PAID | Запуск выбора с уже оплаченной суммой |
| 97 | RHEA_EVENT_SELECTION_AVAILABILITY_UPDATED | Обновление доступности выбора |
| 98 | RHEA_EVENT_SELECTION_PRICES_UPDATED | Обновление цен на выбор |
| 99 | RHEA_EVENT_CREDIT_UPDATED | Обновление кредита |
| 100 | RHEA_EVENT_CPU_MESSAGE | Сообщение от CPU |
| 101 | RHEA_EVENT_SELECTION_REQ_STATUS | Статус запроса выбора |
| 102 | RHEA_EVENT_START_SELECTION | Начало выбора |
| 103 | RHEA_EVENT_STOP_SELECTION | Остановка выбора |
| 104 | RHEA_EVENT_CPU_STATUS | Статус CPU |
| 105 | RHEA_EVENT_ANSWER_TO_IDCODE_REQUEST | Ответ на запрос ID-кода |
| 108 | RHEA_EVENT_READ_DATA_AUDIT | Чтение аудита данных |
| 115 | RHEA_EVENT_SEND_BUTTON | Нажатие кнопки |
| 116 | RHEA_EVENT_SEND_PARTIAL_DA3 | Частичная отправка DA3 |

## Примеры команд

### Примеры событий, связанных с выбором

#### RHEA_EVENT_SELECTION_PROGRESS (ID: 72)
Событие отправляется во время выполнения выбора, показывает прогресс:

```javascript
// Пример обработки события прогресса выбора
rhea.onEvent_selectionProgress = function(progressPercentage) {
  console.log("Прогресс выбора: " + progressPercentage + "%");
};
```

#### RHEA_EVENT_SELECTION_REQ_STATUS (ID: 101)
Событие запрашивает статус определенного выбора:

```javascript
// Запрос статуса конкретного выбора
var buffer = new Uint8Array(2);
buffer[0] = RHEA_EVENT_SELECTION_REQ_STATUS;  // 101
buffer[1] = selectionNumber;  // номер выбора
this.sendGPUCommand("E", buffer, 0, 0);
```

#### RHEA_EVENT_START_SELECTION_FORCE_JUG (ID: 70)
Запуск выбора с принудительным использованием кувшина:

```javascript
// Запуск выбора с принудительным использованием кувшина
var buffer = new Uint8Array(2);
buffer[0] = RHEA_EVENT_START_SELECTION_FORCE_JUG;  // 70
buffer[1] = selectionNumber;  // номер выбора (1-48)
this.sendGPUCommand("E", buffer, 0, 0);
```

### Пример AJAX-запроса
```javascript
// Запрос времени
var commandString = "time";
var plainJSObject = {};
var jo = JSON.stringify(plainJSObject);

var bytesNeeded = 1 + commandString.length + 2 + jo.length;
var buffer = new Uint8Array(bytesNeeded);
var t = 0;
buffer[t++] = (commandString.length & 0x00FF);
for (var i = 0; i < commandString.length; i++)
    buffer[t++] = commandString.charCodeAt(i);
buffer[t++] = ((jo.length & 0xFF00) >> 8);
buffer[t++] = (jo.length & 0x00FF);
for (var i = 0; i < jo.length; i++)
    buffer[t++] = jo.charCodeAt(i);

return this.sendGPUCommand("A", buffer, 1, 4000);
```

### Пример отправки события
```javascript
// Запрос статуса CPU
var buffer = new Uint8Array(1);
buffer[0] = RHEA_EVENT_CPU_STATUS;
this.sendGPUCommand("E", buffer, 0, 0);
```

### Пример запуска выбора
```javascript
// Запуск выбора с номером 5
var buffer = new Uint8Array(2);
buffer[0] = RHEA_EVENT_START_SELECTION;
buffer[1] = 5;
this.sendGPUCommand("E", buffer, 0, 0);
```

### Примеры команд управления выбором

#### START_SELECTION - Запуск выбора
```javascript
// Запуск выбора с номером N
var buffer = new Uint8Array(2);
buffer[0] = RHEA_EVENT_START_SELECTION;  // 102
buffer[1] = N;  // номер выбора (1-48)
this.sendGPUCommand("E", buffer, 0, 0);
```

#### START_SELECTION_FORCE_JUG - Запуск выбора с принудительным использованием кувшина
```javascript
// Запуск выбора с принудительным использованием кувшина
var buffer = new Uint8Array(2);
buffer[0] = RHEA_EVENT_START_SELECTION_FORCE_JUG;  // 70
buffer[1] = N;  // номер выбора (1-48)
this.sendGPUCommand("E", buffer, 0, 0);
```

#### START_SEL_ALREADY_PAID - Запуск выбора с уже оплаченной суммой
```javascript
// Запуск выбора с уже оплаченной суммой
var buffer = new Uint8Array(6);
buffer[0] = RHEA_EVENT_START_SEL_ALREADY_PAID;  // (неизвестный ID, но используется в системе)
buffer[1] = N;  // номер выбора (1-48)
buffer[2] = paymentType;  // тип оплаты
// байты 3-4: цена в виде 16-битного числа (старший байт, младший байт)
buffer[5] = forceJUG;  // флаг принудительного использования кувшина (0 или 1)
this.sendGPUCommand("E", buffer, 0, 0);
```

#### STOP_SELECTION - Остановка текущего выбора
```javascript
// Остановка текущего выбора
var buffer = new Uint8Array(1);
buffer[0] = RHEA_EVENT_STOP_SELECTION;  // 103
this.sendGPUCommand("E", buffer, 0, 0);
```

## Файловый трансфер

### Формат сообщений файлового трансфера
Сообщения файлового трансфера начинаются с:

```
[35] [102] [84] [114] [size_MSB] [size_LSB] [opcode] [payload...]
```

- `35` - '#'
- `102` - 'f'
- `84` - 'T'
- `114` - 'r'
- `size_MSB/size_LSB` - размер полезной нагрузки
- `opcode` - код операции
- `payload...` - полезная нагрузка

### Оpcodes файлового трансфера
- `0x02` - Подтверждение начала передачи
- `0x04` - Подтверждение получения пакета
- `0x51` - Запрос на скачивание файла
- `0x52` - Подтверждение начала скачивания
- `0x53` - Данные файла
- `0x54` - ACK для скачивания

## Состояния CPU

| ID | Состояние | Описание |
|----|-----------|----------|
| 2 | READY | Готов |
| 3 | DRINK PREP | Подготовка напитка |
| 4 | PROGR | Программирование |
| 5 | INI CHECK | Проверка инициализации |
| 6 | ERROR | Ошибка |
| 7 | RINSING | Промывка |
| 8 | AUTO WASHING | Автоматическая мойка |
| 10 | WAIT TEMP | Ожидание температуры |
| 13 | DISINSTALLATION | Демонтаж |
| 15 | END DISINSTALLATION | Завершение демонтажа |
| 20 | BREWER CLEANING | Очистка варочной системы |
| 21 | TEST SEL | Тест выбора |
| 22 | TEST MODEM | Тест модема |
| 23 | CLEANING MILKER | Очистка молочной системы (VENTURI) |
| 24 | CLEANING MILKER | Очистка молочной системы (INDUX) |
| 26 | DESCALING | Дескалинг |
| 30 | MILKER DESCALING | Дескалинг молочной системы |
| 31 | SYRUP CLEANING | Очистка сиропа |
| 101 | COM_ERROR | Ошибка связи |
| 102 | GRINDER OPENING | Открытие кофемолки |
| 103 | COMPATIBILITY CHECK | Проверка совместимости |
| 104 | CPU_NOT_SUPPORTED | CPU не поддерживается |
| 105 | DA3_SYNC | Синхронизация DA3 |
| 106 | GRINDER SPEED TEST | Тест скорости кофемолки |

## Обработка ошибок

### Таймауты
- Таймаут подключения: 5 секунд
- Таймаут ожидания ID-кода: 10 секунд
- Таймаут AJAX-запросов: 4 секунды (настраиваемый)

### Обработка ошибок WebSocket
При ошибках WebSocket браузер перенаправляется на страницу `startup.html` через 2 секунды.

## Примеры использования

### Пример подключения
```javascript
var rhea = new Rhea();
rhea.webSocket_connect()
  .then(function() {
    console.log("Подключено к RHEA");
    // Здесь можно начинать использовать систему
  })
  .catch(function(error) {
    console.log("Ошибка подключения: " + error);
  });
```

### Пример обработки событий
```javascript
// Обработчик обновления статуса CPU
rhea.onEvent_cpuStatus = function(statusID, statusStr, flag16) {
  console.log("Статус CPU: " + statusStr + " (" + statusID + ")");
};

// Обработчик сообщений от CPU
rhea.onEvent_cpuMessage = function(msg, importanceLevel) {
  console.log("Сообщение от CPU: " + msg);
};
```

## Вспомогательные функции

### sendGPUCommand
Основная функция для отправки команд GPU:
```javascript
sendGPUCommand(commandChar, bufferArrayIN, bReturnAPromise, promiseTimeoutMSec)
```

### ajax
Функция для выполнения AJAX-запросов через WebSocket:
```javascript
ajax(commandString, plainJSObject)
```

### requestGPUEvent
Запрос события от GPU:
```javascript
requestGPUEvent(eventTypeID)
```

## Система выбора (Selection System)

### Основные параметры выбора
Каждый из 48 возможных выборов имеет следующие параметры:
- **selNum**: номер выбора (1-48)
- **enabled**: статус доступности (0 - недоступен, 1 - доступен)
- **price**: цена в формате строки (например, "1.50")
- **withFreshMilk**: поддержка свежего молока (0 - нет, 1 - да)

### Управление выбором
Система позволяет:
- Запускать выбор с помощью команд START_SELECTION, START_SELECTION_FORCE_JUG или START_SEL_ALREADY_PAID
- Останавливать текущий выбор с помощью STOP_SELECTION
- Отслеживать прогресс выполнения выбора через событие SELECTION_PROGRESS
- Запрашивать статус конкретного выбора через SELECTION_REQ_STATUS
- Обновлять доступность и цены выборов через соответствующие события

## Заключение

WebSocket-протокол RHEA представляет собой надежный механизм для двунаправленного обмена данными между веб-интерфейсом и системой управления кофеваркой. Протокол поддерживает как синхронные (AJAX-подобные) запросы, так и асинхронные события, что позволяет эффективно управлять всеми аспектами работы кофеварки.