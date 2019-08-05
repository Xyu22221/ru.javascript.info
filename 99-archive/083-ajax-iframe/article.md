archive:
  ref: null

---

# IFRAME для AJAX и COMET

Эта глава посвящена `IFRAME` -- самому древнему и кросс-браузерному способу AJAX-запросов.

Сейчас он используется, разве что, для поддержки кросс-доменных запросов в IE7- и, что чуть более актуально, для реализации COMET в IE9-.

Для общения с сервером создается невидимый `IFRAME`. В него отправляются данные, и в него же сервер пишет ответ.

## Введение

Сначала -- немного вспомогательных функций и особенности работы с `IFRAME`.

### Двуличность IFRAME: окно+документ

Что такое IFRAME? На этот вопрос у браузера два ответа

1. IFRAME -- это HTML-тег: <code>&lt;iframe&gt;</code> со стандартным набором свойств.

	- Тег можно создавать в JavaScript
	- У тега есть стили, можно менять.
	- К тегу можно обратиться через `document.getElementById` и другие методы.
2. IFRAME -- это окно браузера, вложенное в основное

	- IFRAME -- такое же по функционалу окно браузера, как и основное, с адресом и т.п.
        - Если документ в `IFRAME` и внешнее окно находятся на разных доменах, то прямой вызов методов друг друга невозможен.
	- Ссылку на это окно можно получить через `window.frames['имя фрейма']`.

Для достижения цели мы будем работать как с тегом, так и с окном. Они, конечно же, взаимосвязаны.

**В теге `<iframe>` свойство `contentWindow` хранит ссылку на окно.**

Окна также содержатся в коллекции `window.frames`.

Например:

```js
// Окно из ифрейма
var iframeWin = iframe.contentWindow;

// Можно получить и через frames, если мы знаем имя ифрейма (и оно у него есть)
var iframeWin = window.frames[iframe.name];
iframeWin.parent == window; // parent из iframe указывает на родительское окно

// Документ не будет доступен, если iframe с другого домена
var iframeDoc = iframe.contentWindow.document;
```

Больше информации об ифреймах вы можете получить в главе <info:iframes>.

### IFRAME и история посещений

`IFRAME` -- полноценное окно, поэтому навигация в нём попадает в историю посещений.

Это означает, что при нажатии кнопки "Назад" браузер вернёт посетителя назад не в основном окне, а в ифрейме. В лучшем случае -- браузер возьмёт предыдущее состояние ифрейма из кэша и посетитель просто подумает, что кнопка не сработала. В худшем -- в ифрейм будет сделан предыдущий запрос, а это уже точно ни к чему.

**Наши запросы в ифрейм -- служебные и для истории не предназначены. К счастью, есть ряд техник, которые позволяют обойти проблему.**

- Ифрейм нужно создавать динамически, через JavaScript.
- Когда ифрейм уже создан, то единственный способ поменять его `src` без попадания запроса в историю посещений:

    ```js
    // newSrc - новый адрес
    iframeDoc.location.replace(newSrc);
    ```

    Вы можете возразить: "но ведь `iframeDoc` не всегда доступен! `iframe` может быть с другого домена -- как быть тогда?". Ответ: вместо смены `src` этого ифрейма -- создать новый, с новым `src`.
- POST-запросы в `iframe` всегда попадают в историю посещений.
- ... Но если `iframe` удалить, то лишняя история тоже исчезнет :). Сделать это можно по окончании запроса.

**Таким образом, общий принцип использования `IFRAME`: динамически создать, сделать запрос, удалить.**

Бывает так, что удалить по каким-то причинам нельзя, тогда возможны проблемы с историей, описанные выше.

### Функция createIframe

Приведенная ниже функция `createIframe(name, src, debug)` кросс-браузерно создаёт ифрейм с данным именем и `src`.

Аргументы:

`name`
: Имя и `id` ифрейма

`src`
: Исходный адрес ифрейма. Необязательный параметр.

`debug`
: Если параметр задан, то ифрейм после создания не прячется.

```js
function createIframe(name, src, debug) {
  src = src || 'javascript:false'; // пустой src

  var tmpElem = document.createElement('div');

  // в старых IE нельзя присвоить name после создания iframe
  // поэтому создаём через innerHTML
  tmpElem.innerHTML = '<iframe name="' + name + '" id="' + name + '" src="' + src + '">';
  var iframe = tmpElem.firstChild;

  if (!debug) {
    iframe.style.display = 'none';
  }

  document.body.appendChild(iframe);

  return iframe;
}
```

Ифрейм здесь добавляется к `document.body`. Конечно, вы можете исправить этот код и добавлять его в любое другое место документа.

Кстати, при вставке, если не указан `src`, тут же произойдёт событие `iframe.onload`. Пока обработчиков нет, поэтому оно будет проигнорировано.

### Функция postToIframe

Функция `postToIframe(url, data, target)` отправляет POST-запрос в ифрейм с именем `target`, на адрес `url` с данными `data`.

Аргументы:

`url`
: URL, на который отправлять запрос.

`data`
: Объект содержит пары `ключ:значение` для полей формы. Значение будет приведено к строке.

`target`
: Имя ифрейма, в который отправлять данные.

```js
// Например: postToIframe('/vote', {mark:5}, 'frame1')

function postToIframe(url, data, target) {
  var phonyForm = document.getElementById('phonyForm');
  if (!phonyForm) {
    // временную форму создаем, если нет
    phonyForm = document.createElement("form");
    phonyForm.id = 'phonyForm';
    phonyForm.style.display = "none";
    phonyForm.method = "POST";
    document.body.appendChild(phonyForm);
  }

  phonyForm.action = url;
  phonyForm.target = target;

  // заполнить форму данными из объекта
  var html = [];
  for (var key in data) {
    var value = String(data[key]).replace(/"/g, "&quot;");
    // в старых IE нельзя указать name после создания input
    // поэтому используем innerHTML вместо DOM-методов
    html.push("<input type='hidden' name=\"" + key + "\" value=\"" + value + "\">");
  }
  phonyForm.innerHTML = html.join('');

  phonyForm.submit();
}
```

Эта функция формирует форму динамически, но, конечно, это лишь один из возможных сценариев использования.

В `IFRAME` можно отправлять и существующую форму, включающую файловые и другие поля.

## Запросы GET и POST

Общий алгоритм обращения к серверу через ифрейм:

1. Создаём `iframe` со случайным именем `iframeName`.
2. Создаём в основном окне объект `CallbackRegistry`, в котором в `CallbackRegistry[iframeName]` сохраняем функцию, которая будет обрабатывать результат.
3. Отправляем GET или POST-запрос в него.
4. Сервер отвечает как-то так:

    ```html no-beautify
    <script>
      parent.CallbackRegistry[window.name]({данные});
    </script>
    ```

    ...То есть, вызывает из основного окна функцию обработки (`window.name` в ифрейме -- его имя).
5. Дополнительно нужен обработчик `iframe.onload` -- он сработает и проверит, выполнилась ли функция `CallbackRegistry[window.name]`. Если нет, значит какая-то ошибка. Сервер при нормальном потоке выполнения всегда отвечает её вызовом.

Подробнее можно понять процесс, взглянув на код.

Мы будем использовать в нём две функции -- одну для GET, другую -- для POST:

- `iframeGet(url, onSuccess, onError)` -- для GET-запросов на `url`. При успешном запросе вызывается `onSuccess(result)`, при неуспешном: `onError()`.
- `iframePost(url, data, onSuccess, onError)` -- для POST-запросов на `url`. Значением `data` должен быть объект `ключ:значение` для пересылаемых данных, он конвертируется в поля формы.

Пример в действии, возвращающий дату сервера при GET и разницу между датами клиента и сервера при POST:

[codetabs src="date"]

Прямой вызов функции внешнего окна из ифрейма отлично работает, потому что они с одного домена. Если с разных, то нужны дополнительные действия, например:

- В IE8+ есть интерфейс [postMessage](https://developer.mozilla.org/en-US/docs/DOM/window.postMessage) для общения между окнами с разных доменов.
- В любых, даже самых старых IE, можно обмениваться данными через `window.name`. Эта переменная хранит "имя" окна или фрейма, которое не меняется при перезагрузке страницы.

    Поэтому если мы сделали `POST` в `<iframe>` на другой домен и он поставил `window.name = "Вася"`, а затем сделал редирект на основной домен, то эти данные станут доступны внешней странице.
- Также в совсем старых IE можно обмениваться данными через хеш, то есть фрагмент URL после `#`. Его изменение доступно между ифреймами с разных доменов и не приводит к перезагрузке страницы. Таким образом они могут передавать данные друг другу. Есть готовые библиотеки, которые реализуют этот подход, например [Porthole](http://ternarylabs.github.io/porthole/).

## IFRAME для COMET

Бесконечный IFRAME -- самый старый способ организации COMET. Когда-то он был основой AJAX-приложений, а сейчас -- используется лишь в случаях, когда браузер не поддерживает современный стандарт WebSocket, то есть для IE9-.

Этот способ основан на том, что браузер читает страницу последовательно и обрабатывает все новые теги по мере того, как сервер их присылает.

Классическая реализация -- это когда клиент создает невидимый IFRAME, ведущий на служебный URL. Сервер, получив соединение на этот URL, не закрывает его, а
время от времени присылает блоки сообщений <code>&lt;script&gt;...javascript...&lt;/script&gt;</code>. Появившийся в IFRAME'е javascript тут же выполняется браузером, передавая информацию на основную страницу.

Таким образом, для передачи данных используется "бесконечный" ифрейм, через который сервер присылает все новые данные.

Схема работы:

1. Создаётся `<iframe src="COMET_URL">`, по адресу `COMET_URL` расположен сервер.
2. Сервер выдаёт начало ("шапку") документа и останавливается, оставляя соединение активным.
3. Когда сервер хочет что-то отправить -- он пишет в соединение <code>&lt;script&gt;parent.onMessage(сообщение)&lt;/script&gt;</code> Браузер тут же выполняет этот скрипт -- так сообщение приходит на клиент.
4. Ифрейм, в теории, грузится бесконечно. Его завершение означает обрыв канала связи. Его можно поймать по `iframe.onload` и заново открыть соединение (создать новый `iframe`).

Также ифрейм можно пересоздавать время от времени, для очистки памяти от старых сообщений.

![](comet.png)

Ифрейм при этом работает только на получение данных с сервера, как альтернатива [Server Sent Events](/server-sent-events). Для запросов используется обычный `XMLHttpRequest`.

## Обход проблем с IE

Такое использование ифреймов является хаком. Поэтому есть ряд проблем:

1. Показывается индикатор загрузки, "курсор-часики".
2. При POST в `<iframe>` раздаётся звук "клика".
3. Браузер буферизует начало страницы.

Мы должны эти проблемы решить, прежде всего, в IE, поскольку в других браузерах есть [WebSocket](/websockets) и [Server Sent Events](/server-sent-events) .

Проще всего решить последнюю -- IE не начинает обработку страницы, пока она не загрузится до определенного размера.

Поэтому в таком `IFRAME` первые несколько сообщений задержатся:

```html no-beautify
<!DOCTYPE HTML>
<html>
  <body>
  <script>parent.onMessage("привет");</script>
  <script>parent.onMessage("от сервера");</script>
  ...
```

Решение -- забить начало ифрейма чем-нибудь, поставить, например, килобайт пробелов в начале:

```html
<!DOCTYPE HTML>
<html>

<body>
  ******* 1 килобайт пробелов, а потом уже сообщения ******
  <script>
    parent.onMessage("привет");
  </script>
  <script>
    parent.onMessage("от сервера");
  </script>
  ...
```

Для решения проблемы с индикацией загрузки и клика мы можем использовать безопасный ActiveX-объект `htmlfile`. IE не требует разрешений на его создание. Фактически, это независимый HTML-документ.

Оказывается, если `iframe` создать в нём, то никакой анимации и звуков не будет.

Итак, схема:

1. Основное окно `main` создаёт вспомогательный объект: `new ActiveXObject("htmlfile")`. Это HTML-документ со своим `window`, похоже на встроенный `iframe`.
2. В `htmlfile` записывается `iframe`.
3. Цепочка общения: основное окно - `htmlfile` - ифрейм.

### iframeActiveXGet

На самом деле всё еще проще, если посмотреть на код:

Метод `iframeActiveXGet` по существу идентичен обычному `iframeGet`, которое мы рассмотрели. Единственное отличие -- вместо `createIframe` используется особый метод `createActiveXFrame`:

```js
function iframeActiveXGet(url, onSuccess, onError) {

  var iframeOk = false;

  var iframeName = Math.random();
*!*
  var iframe = createActiveXFrame(iframeName, url);
*/!*

  CallbackRegistry[iframeName] = function(data) {
    iframeOk = true;
    onSuccess(data);
  }

  iframe.onload = function() {
    iframe.parentNode.removeChild(iframe); // очистка
    delete CallbackRegistry[iframeName];
    if (!iframeOk) onError(); // если колбэк не вызвался - что-то не так
  }

}
```

### createActiveXFrame

В этой функции творится вся IE-магия:

```js
function createActiveXFrame(name, src) {
  // (1)
  var htmlfile = window.htmlfile;
  if (!htmlfile) {
    htmlfile = window.htmlfile = new ActiveXObject("htmlfile");
    htmlfile.open();
    // (2)
    htmlfile.write("<html><body></body></html>");
    htmlfile.close();
    // (3)
    htmlfile.parentWindow.CallbackRegistry = CallbackRegistry;
  }

  // (4)
  src = src || 'javascript:false';
  htmlfile.body.insertAdjacentHTML('beforeEnd',
    "<iframe name='" + name + "' src='" + src + "'></iframe>");
  return htmlfile.body.lastChild;
}
```

1. Вспомогательный объект `htmlfile` будет один и он будет глобальным. Можно и спрятать переменную в замыкании. Смысл в том, что в один `htmlfile` можно записать много ифреймов, так что не будем множить сущности и занимать ими лишнюю память.
2. В `htmlfile` можно записать любой текст и, при необходимости, через  `document.write('<script>...<\/script>)`. Здесь мы делаем пустой документ.
3. Когда загрузится `iframe`, он сделает вызов:

    ```html
    <script>
      parent.CallbackRegistry[window.name](объект с данными);
    </script>
    ```

    Здесь `parent'ом` для `iframe'а` будет `htmlfile`, т.е. `CallbackRegistry` будет искаться среди переменных соответствующего ему окна, а вовсе не верхнего `window`.

    Окно для `htmlfile` доступно как `htmlfile.parentWindow`, копируем в него ссылку на реестр колбэков `CallbackRegistry`. Теперь ифрейм его найдёт.
4. Далее вставляем ифрейм в документ. В старых `IE` нельзя поменять `name` ифрейму через DOM, поэтому вставляем строкой через `insertAdjacentHTML`.

Пример в действии (только IE):

[codetabs src="date-activex"]

Запрос, который происходит, полностью незаметен.

Метод POST делается аналогично, только форму нужно добавлять не в основное окно, а в `htmlfile`, через вызов `htmlfile.appendChild`. В остальном -- всё так же, как и при обычной отправке через ифрейм.

Впрочем, для COMET нужен именно GET.

Можно и сочетать эти способы: если есть ActiveX: `if ("ActiveXObject" in window)` -- используем методы для IE, описанные выше, а иначе -- обычные методы.

Вот мини-приложение с сервером на Node.JS, непрерывно получающее текущее время с сервера через `<iframe>`, сочетающее эти подходы:

[codetabs src="date-comet"]

Ещё раз заметим, что обычно такое сочетание не нужно, так как если не IE9-, то можно использовать более современные средства для COMET.

## Итого

- Iframe позволяет делать "AJAX"-запросы и хитро обходить кросс-доменные ограничения в IE7-. Обычно для этого используют либо `window.name` с редиректом, либо хеш с библиотекой типа [Porthole](https://github.com/ternarylabs/porthole).
- В IE9- iframe можно использовать для COMET. В IE10 уже есть WebSocket.

Существует ряд уже готовых клиент-серверных библиотек, которые реализуют AJAX/COMET, в том числе и через iframe, мы рассмотрим их позже. Поэтому совсем не обязательно делать "с нуля". Хотя, как можно видеть из главы, это совсем несложно.