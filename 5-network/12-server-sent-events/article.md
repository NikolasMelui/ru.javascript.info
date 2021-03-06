# Server Sent Events

Спецификация [Server-Sent Events](https://html.spec.whatwg.org/multipage/comms.html#the-eventsource-interface) описывает встроенный класс `EventSource`, который позволяет поддерживать соединение с сервером и получать от него события.

Как и в случае с `WebSocket`, соединение постоянно.

Но есть несколько важных различий:

| `WebSocket` | `EventSource` |
|-------------|---------------|
| Двунаправленность: и сервер, и клиент могут обмениваться сообщениями | Однонаправленность: данные посылает только сервер |
| Бинарные и текстовые данные | Только текст |
| Протокол WebSocket | Обычный HTTP |

`EventSource` не настолько мощный способ коммуникации с сервером, как `WebSocket`.

Зачем нам его использовать?

Основная причина: он проще. Многим приложениям не требуется вся мощь `WebSocket`.

Если нам нужно получать поток данных с сервера: неважно, сообщения в чате или же цены для магазина - с этим легко справится `EventSource`. К тому же, он поддерживает автоматическое переподключение при потере соединения, которое, используя `WebSocket`, нам бы пришлось реализовывать самим. Кроме того, используется старый добрый HTTP, а не новый протокол.

## Получение сообщений

Чтобы начать получать данные, нам нужно просто создать `new EventSource(url)`.

Браузер установит соединение с `url` и будет поддерживать его открытым, ожидая события.

Сервер должен ответить со статусом 200 и заголовком `Content-Type: text/event-stream`, затем он должен поддерживать соединение открытым и отправлять сообщения в особом формате:

```
data: Сообщение 1

data: Сообщение 2

data: Сообщение 3
data: в две строки
```

- Текст сообщения указывается после `data:`, пробел после двоеточия необязателен.
- Сообщения разделяются двойным переносом строки `\n\n`.
- Чтобы разделить сообщение на несколько строк, мы можем отправить несколько `data:` подряд (третье сообщение).

На практике сложные сообщения обычно отправляются в формате JSON, в котором перевод строки кодируется как `\n`, так что в разделении сообщения на несколько строк обычно нет нужды.

Например:

```js
data: {"user":"Джон","message":"Первая строка*!*\n*/!* Вторая строка"}
```

...Так что можно считать, что в каждом `data:` содержится ровно одно сообщение.

Для каждого сообщения генерируется событие `message`:

```js
let eventSource = new EventSource("/events/subscribe");

eventSource.onmessage = function(event) {
  console.log("Новое сообщение", event.data);
  // этот код выведет в консоль 3 сообщения, из потока данных выше
};

// или eventSource.addEventListener('message', ...)
```

### Кросс-доменные запросы

`EventSource`, как и `fetch`, поддерживает кросс-доменные запросы. Мы можем использовать любой URL:

```js
let source = new EventSource("https://another-site.com/events");
```

Сервер получит заголовок `Origin` и должен будет ответить с заголовком `Access-Control-Allow-Origin`.

Чтобы послать авторизационные данные, следует установить дополнительную опцию `withCredentials`:

```js
let source = new EventSource("https://another-site.com/events", {
  withCredentials: true
});
```

Более подробное описание кросс-доменных заголовков вы можете прочитать в главе <info:fetch-crossorigin>.


## Переподключение

После создания `new EventSource` подключается к серверу и, если соединение обрывается, - переподключается.

Это очень удобно, так как нам не приходится беспокоиться об этом.

По умолчанию между попытками возобновить соединение будет небольшая пауза в несколько секунд.

Сервер может выставить рекомендуемую задержку, указав в ответе `retry:` (в миллисекундах):

```js
retry: 15000
data: Привет, я выставил задержку переподключения в 15 секунд
```

Поле `retry:` может посылаться как вместе с данными, так и отдельным сообщением.

Браузеру следует ждать именно столько миллисекунд перед новой попыткой подключения. Или дольше, например, если браузер знает (от операционной сисстемы) что соединения с сетью нет, то он может осуществить переподключение только когда оно появится.

- Если сервер хочет остановить попытки переподключения, он должен ответить со статусом 204.
- Если браузер хочет прекратить соединение, он может вызвать `eventSource.close()`:

```js
let eventSource = new EventSource(...);

eventSource.close();
```

Также переподключение не произойдёт, если в ответе указан неверный `Content-Type` или его статус отличается от 301, 307, 200 и 204. Браузер создаст событие `"error"` и не будет восстанавливать соединение.

```smart
После того как соединение окончательно закрыто, "переоткрыть" его уже нельзя. Если необходимо снова подключиться, просто создайте новый `EventSource`.
```

## Идентификатор сообщения

Когда соединение прерывается из-за проблем с сетью, ни сервер, ни клиент не могут быть уверены в том, какие сообщения были доставлены, а какие - нет.

Чтобы правильно возобновить подключение, каждое сообщение должно иметь поле `id`:

```
data: Сообщение 1
id: 1

data: Сообщение 2
id: 2

data: Сообщение 3
data: в две строки
id: 3
```

Получая сообщение с указанным `id:`, браузер:

- Установит его значение свойству `eventSource.lastEventId`.
- При переподключении отправит заголовок `Last-Event-ID` с этим `id`, чтобы сервер мог переслать последующие сообщения.

```smart header="Указывайте `id:` после `data:`"
Обратите внимание: `id` указывается сервером после данных `data` сообщения, чтобы обновление `lastEventId` произошло после того, как сообщение будет получено.
```

## Статус подключения: readyState

У объекта `EventSource` есть свойство `readyState`, имеющее одно из трёх значений:

```js no-beautify
EventSource.CONNECTING = 0; // подключение или переподключение
EventSource.OPEN = 1;       // подключено
EventSource.CLOSED = 2;     // подключение закрыто
```

При создании объекта и разрыве соединения оно автоматически устанавливается в значение `EventSource.CONNECTING` (равно `0`).

Мы можем обратиться к этому свойству, чтобы узнать текущее состояние `EventSource`.

## Типы событий

По умолчанию объект `EventSource` генерирует 3 события:

- `message` -- получено сообщение, доступно как `event.data`.
- `open` -- соединение открыто.
- `error` -- не удалось установить соединение, например, сервер вернул статус 500.

Сервер может указать другой тип события с помощью `event: ...` в начале сообщения.

Например:

```
event: join
data: Боб

data: Привет

event: leave
data: Боб
```

Чтобы начать слушать пользователькие события, нужно использовать `addEventListener`, а не `onmessage`:

```js
eventSource.addEventListener('join', event => {
  alert(`${event.data} зашёл`);
});

eventSource.addEventListener('message', event => {
  alert(`Сказал: ${event.data}`);
});

eventSource.addEventListener('leave', event => {
  alert(`${event.data} вышел`);
});
```

## Полный пример

В этом примере сервер посылает сообщения `1`, `2`, `3`, затем `пока-пока` и разрывает соединение.

После этого браузер автоматически переподключается.

[codetabs src="eventsource"]


## Итого

Объект `EventSource` автоматически устанавливает постоянное соединение и позволяет серверу отправлять через него сообщения.

Он предоставляет:
- Автоматическое переподключение с настраиваемой `retry` задержкой.
- Идентификаторы сообщений для восстановления соединения. Последний полученный идентификатор посылается в заголовке `Last-Event-ID` при пересоединении.
- Текущее состояние, записанное в свойстве `readyState`.

Это делает `EventSource` достойной альтернативой протоколу `WebSocket`, который сравнительно низкоуровневый и не имеет таких встроенных возможностей (хотя их и можно реализовать).

Для многих приложений возможностей `EventSource` вполне достаточно.

Поддерживается во всех современных браузерах (кроме Internet Explorer).

Синтаксис:

```js
let source = new EventSource(url, [credentials]);
```

Второй аргумент - необязательный объект с одним свойством: `{ withCredentials: true }`. Он позволяет отправлять авторизационные данные на другие домены.

В целом, кросс-доменная безопасность реализована так же как в `fetch` и других методов работы с сетью.

### Свойства объекта `EventSource`

`readyState`
: Текущее состояние подключения: `EventSource.CONNECTING (=0)`, `EventSource.OPEN (=1)` или `EventSource.CLOSED (=2)`.

`lastEventId`
: `id` последнего полученного сообщения. При переподключении браузер посылает его в заголовке `Last-Event-ID`.

### Методы

`close()`
: Закрывает соединение.

### События

`message`
: Сообщение получено, переданные данные записаны в `event.data`.

`open`
: Соединение установлено.

`error`
: В случае ошибки, включая как потерю соединения так и другие ошибки в нём. Мы можем обратиться к свойству `readyState`, чтобы проверить, происходит ли переподключение.

Сервер может выставить собственное событие с помощью `event:`. Такие события должны быть обработаны с помощью `addEventListener`, а не `on<event>`.

### Формат ответа сервера

Сервер посылает сообщения, разделённые двойным переносом строки `\n\n`.

Сообщение состоит из следующих полей:

- `data:` -- тело сообщения, несколько `data` подряд интерпретируются как одно сообщение, разделённое переносами строк `\n`.
- `id:` -- обновляет свойство `lastEventId`, отправляемое в `Last-Event-ID` при переподключении.
- `retry:` -- рекомендованная задержка перед переподключением в миллисекундах. Не может быть установлена с помощью JavaScript.
- `event:` -- имя пользовательского события, должно быть указано перед `data:`.

Сообщение может включать одно или несколько этих полей в любом порядке, но id обычно ставят в конце.
