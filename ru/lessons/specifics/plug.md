---
version: 2.2.0
title: Plug
---

Если вы знакомы с Ruby, вы можете думать о Plug как о комбинации Rack и Sinatra.
Он предоставляет спецификацию для компонентов веб-приложений и адаптеров для веб-серверов.
Хотя Plug и не является частью ядра Elixir, это официальный проект от той же команды.

В этом уроке мы создадим простой HTTP-сервер с нуля, используя библиотеку Elixir PlugCowboy.
Cowboy - это простой HTTP-сервер для Erlang, и Plug предоставит нам адаптер подключения для этого веб-сервера.

После того, как мы настроим наше минимальное рабочее веб-приложение, мы узнаем о маршрутизаторе Plug и о том, как использовать несколько плагинов в одном веб-приложении.

{% include toc.html %}

## Перед установкой

Чтобы следовать инструкциям этого урока, вам понадобятся установленный Elixir версии 1.5 или выше и `mix`.

Мы начнем с создания нового OTP-проекта с деревом супервизора.

```shell
$ mix new example --sup
$ cd example
```

Нам нужно, чтобы наше приложение Elixir включало дерево супервизора, потому что мы будем использовать супервизор для запуска нашего сервера Cowboy2.

## Зависимости

Добавлять новые зависимости при помощи mix невероятно легко.
Чтобы использовать Plug в качестве интерфейса адаптера для веб-сервера Cowboy2, нам необходимо установить пакет PlugCowboy:

Добавьте в файл `mix.exs` следующее:

```elixir
def deps do
  [
    {:plug_cowboy, "~> 2.0"},
  ]
end
```

Выполните следующую команду в терминале, чтобы mix скачал и установил новые зависимости:

```shell
$ mix deps.get
```

## Спецификация

Чтобы создавать собственные Plug модули, нужно придерживаться спецификации.
К счастью, необходимо реализовать всего две функции: `init/1` и `call/2`.

Вот пример простого Plug модуля, который возвращает "Привет Мир!":

```elixir
defmodule Example.HelloWorldPlug do
  import Plug.Conn

  def init(options), do: options

  def call(conn, _opts) do
    conn
    |> put_resp_content_type("text/plain")
    |> send_resp(200, "Привет Мир!\n")
  end
end
```

Сохраним файл как `lib/example/hello_world_plug.ex`.

Функция `init/1` используется для инициализации параметров нашего модуля `Plug`.
Она вызывается супервизором, который мы увидим в следующей секции.
Пока что в качестве параметров будет пустой список.

Значение, возвращаемое `init/1`, передается в качестве второго аргумента в функцию `call/2`.

Функция `call/2` вызывается для каждого нового запроса, приходящего от веб-сервера Cowboy.
Она получает структуру `%Plug.Conn{}` в качества первого аргумента, и ожидается, что она также вернёт соединение (структуру того типа `%Plug.Conn{}`).

## Настройка Application-модуля приложения

Так как мы создаём Plug-приложение с нуля, нам придется создать еще и Application-модуль.
Добавим в `lib/example.ex` старт веб-сервера `Cowboy`:

```elixir
defmodule Example.Application do
  use Application
  require Logger

  def start(_type, _args) do
    children = [
      {Plug.Cowboy, scheme: :http, plug: Example.HelloWorldPlug, options: [port: 8080]}
    ]
    opts = [strategy: :one_for_one, name: Example.Supervisor]

    Logger.info("Starting application...")

    Supervisor.start_link(children, opts)
  end
end
```

Это запустит `Cowboy` под супервизором, который в свою очередь запустит `HelloWorldPlug` в качестве дочернего процесса.

В вызове `Plug.Adapters.Cowboy.child_spec/4` третий аргумент будет передан в `Example.HelloWorldPlug.init/1`.

Но это еще не все. Откроем `mix.exs` снова и найдем там функцию `applications`.
Нужно сделать так, чтобы наше приложение автоматически запускалось.

Для этого изменим файл следующим образом:
```elixir
def application do
  [
    extra_applications: [:logger],
    mod: {Example.Application, []}
  ]
end
```

Теперь всё готово к запуску нашего первого веб-приложения, созданного на базе `Plug`.
В командной строке выполним:

```shell
$ mix run --no-halt
```

Как только все скомпилируется, и выведется сообщение `[info]  Started app`, откройте
в браузере  <http://127.0.0.1:8080>.
Там должно появиться следующее:

```
Привет Мир!
```

## Использование Plug.Router

Для большинства приложений, таких как веб-сайты и REST API, понадобится что-то, что будет перенаправлять запросы к определенным ресурсам на соответствующие обработчики в коде.
Специально для этого в `Plug` существует маршрутизатор (или роутер). Как мы сейчас увидим, фреймворк типа `Sinatra` в `Elixir` не требуется, так как мы получаем его возможности вместе с `Plug`

Для начала создадим файл `lib/plug/router.ex` и скопируем в него следующий код:

```elixir
defmodule Example.Router do
  use Plug.Router

  plug :match
  plug :dispatch

  get "/" do
    send_resp(conn, 200, "Добро пожаловать")
  end

  match _ do
    send_resp(conn, 404, "Ой!")
  end
end
```

Это самая простая реализация модуля `Router`, её код довольно очевиден.
Мы подключили необходимые макросы с помощью инструкции `use Plug.Router` и задействовали встроенные модули `Plug`: `:match` и `:dispatch`. В коде задано два предопределённых пути маршрутизации: один, для обработки `GET`-запросов к родительскому узлу `'/'`, и второй, для обработки всех остальных запросов, возвращающий сообщение об ошибке `404`.

Вернемся теперь к `lib/example.ex` и добавим `Example.Router` к дочерним процессам веб-сервера.
Поменяем `Example.HelloWorldPlug` на наш новый роутер:

```elixir
def start(_type, _args) do
  children = [
    {Plug.Cowboy, scheme: :http, plug: Example.Router, options: [port: 8080]}
  ]
  opts = [strategy: :one_for_one, name: Example.Supervisor]

  Logger.info("Starting application...")

  Supervisor.start_link(children, opts)
end
```

Запустим веб-сервер (в случае, если предыдущий сервер еще работает, его можно остановить, дважды нажав `Ctrl+C`).

Теперь откроем <http://127.0.0.1:8080> в браузере.
Мы должны увидеть сообщение `Добро пожаловать`.
Попробуем открыть <http://127.0.0.1:8080/waldo> или любой другой ресурс.
Должна появиться 404 ошибка с текстом `Ой!`.

## Создание еще одного модуля Plug

Очень часто Plug-модули используются для обработки всех или части входящих запросов в соответствии с общей логикой.

Для примера создадим модуль `Plug`, проверяющий наличие всех заданных параметров у входящего запроса. Реализуя такую проверку в виде модуля `Plug`, мы можем быть уверены, что приложением будут обрабатываться только корректные запросы. Ожидается, что наш модуль будет инициализироваться с двумя аргументами: `:paths` и `:fields`. Первый будет содержать те пути запросов, к которым мы применяем нашу проверку, а второй &mdash; наличие каких именно параметров у входящего запроса требуется контролировать.

_Примечание_: модули `Plug` применяются ко всем запросам подряд, именно поэтому мы реализуем фильтрацию запросов и применяем нашу логику только к определённому их подмножеству. Чтобы проигнорировать запрос, мы просто передаём входящее соединение (структуру `%Plug.Conn`) далее без изменений.

Сначала мы покажем реализацию такого модуля Plug, а потом разберём его работу.
Создаём модуль в файле `lib/example/plug/verify_request.ex`:

```elixir
defmodule Example.Plug.VerifyRequest do
  defmodule IncompleteRequestError do
    @moduledoc """
    Если у запроса отсутствует один из требуемых параметров - возникает исключение.
    """

    defexception message: ""
  end

  def init(options), do: options

  def call(%Plug.Conn{request_path: path} = conn, opts) do
    if path in opts[:paths], do: verify_request!(conn.params, opts[:fields])
    conn
  end

  defp verify_request!(params, fields) do
    verified =
      params
      |> Map.keys()
      |> contains_fields?(fields)

    unless verified, do: raise(IncompleteRequestError)
  end

  defp contains_fields?(keys, fields), do: Enum.all?(fields, &(&1 in keys))
end
```

Первое, что необходимо отметить &mdash; мы определили новое исключение `IncompleteRequestError`, и что один из его параметров это `:plug_status`. Если этот параметр доступен, модуль `Plug` использует его, чтобы установить код состояния для `HTTP` ответа в случае возникновения исключения.

Вторая часть модуля, это функция `call/2`. Именно тут определяется, нужно ли вообще проверять данный запрос. Мы вызываем функцию `verify_request!/2` только в том случае, если путь запроса содержится в аргументе `:paths`.

Последняя часть описываемого модуля Plug &mdash; закрытая функция `verify_request!/2`, которая проверяет наличие у запроса всех требуемых параметров из аргумента `:fields`. В случае отсутствия любого из параметров, вызывается исключение `IncompleteRequestError`.

Мы настроили наш модуль `Plug` так, чтобы проверять, что все запросы к пути `/upload` содержат параметры `"content"` и `"mimetype"`. Только в случае прохождения этой проверки может быть выполнен код маршрутизатора, связанный с такими запросами.

Теперь нужно сообщить маршрутизатору о новом Plug-модуле.
Отредактируем `lib/example/router.ex` следующим образом:

```elixir
defmodule Example.Router do
  use Plug.Router

  alias Example.Plug.VerifyRequest

  plug Plug.Parsers, parsers: [:urlencoded, :multipart]
  plug VerifyRequest, fields: ["content", "mimetype"], paths: ["/upload"]
  plug :match
  plug :dispatch

  get "/" do
    send_resp(conn, 200, "Добро пожаловать")
  end

  get "/upload" do
    send_resp(conn, 201, "Загружено")
  end

  match _ do
    send_resp(conn, 404, "Ой!")
  end
end
```

With this code, we are telling our application to send incoming requests through the `VerifyRequest` plug _before_ running through the code in the router.
Via the function call:

```elixir
plug VerifyRequest, fields: ["content", "mimetype"], paths: ["/upload"]
```
We automatically invoke `VerifyRequest.init(fields: ["content", "mimetype"], paths: ["/upload"])`.
This in turn passes the given options to the `VerifyRequest.call(conn, opts)` function.

Let's take a look at this plug in action! Go ahead and crash your local server (remember, that's done by pressing `ctrl + c` twice).
Then restart the server (`mix run --no-halt`).
Now go to <http://127.0.0.1:8080/upload> in your browser and you'll see that the page simply isn't working. You'll just see a default error page provided by your browser.

Now let's add our required params by going to <http://127.0.0.1:8080/upload?content=thing1&mimetype=thing2>. Now we should see our 'Uploaded' message.

It's not great that when we raise an error, we don't get _any_ page. We'll look at how to handle errors with plugs later.

## Делаем HTTP порт конфигурируемым

Когда мы создавали наше приложение `Example`, HTTP порт был "зашит" в коде.
Считается хорошим тоном делать порт конфигурируемым при помощи файлов настроек.

Мы установим переменную среды приложения в `config/config.exs`

```elixir
use Mix.Config

config :example, cowboy_port: 8080
```
Затем нам нужно обновить `lib/example/application.ex`, прочитать значение конфигурации порта и передать его Cowboy.
Мы определим частную функцию, чтобы завершить эту ответственность.

Непосредственно нашего приложения касается строка `mod: {Example, []}`. Обратите внимание, что мы также запускаем приложения `cowboy`, `logger` и `plug`.
Далее необходимо добавить в файл `lib/example.ex` чтение номера порта из настроек и передачу его в `Cowboy`:

```elixir
defmodule Example.Application do
  use Application
  require Logger

  def start(_type, _args) do
    children = [
      {Plug.Cowboy, scheme: :http, plug: Example.Router, options: [port: cowboy_port()]}
    ]
    opts = [strategy: :one_for_one, name: Example.Supervisor]

    Logger.info("Starting application...")

    Supervisor.start_link(children, opts)
  end

  defp cowboy_port, do: Application.get_env(:example, :cowboy_port, 8080)
end
```

Третий аргумент в `Application.get_env` &mdash; это порт по умолчанию на случай, если настройка не объявлена.

Теперь для запуска приложения можно использовать команду:

```shell
$ mix run --no-halt
```

## Тестирование модуля Plug

Тестировать модули `Plug` легко благодаря наличию `Plug.Test`.
Этот модуль предоставляет множество функций для упрощения тестирования.

Напишем следующий тест в `test/example/router_test.exs`:

```elixir
defmodule Example.RouterTest do
  use ExUnit.Case
  use Plug.Test

  alias Example.Router

  @content "<html><body>Привет!</body></html>"
  @mimetype "text/html"

  @opts Router.init([])

  test "returns Добро пожаловать" do
    conn =
      :get
      |> conn("/", "")
      |> Router.call(@opts)

    assert conn.state == :sent
    assert conn.status == 200
  end

  test "returns Загружено" do
    conn =
      :get
      |> conn("/upload?content=#{@content}&mimetype=#{@mimetype}")
      |> Router.call(@opts)

    assert conn.state == :sent
    assert conn.status == 201
  end

  test "returns 404" do
    conn =
      :get
      |> conn("/missing", "")
      |> Router.call(@opts)

    assert conn.state == :sent
    assert conn.status == 404
  end
end
```

И запустим командой:

```shell
$ mix test test/example/router_test.exs
```

## Plug.ErrorHandler

Ранее мы заметили, что когда мы перешли на <http://127.0.0.1:8080/upload> без ожидаемых параметров, мы не получили удобную страницу с ошибкой или разумный статус HTTP - просто страницу с ошибкой c `500 Internal Server Error`.

Давайте исправим это сейчас, добавив [`Plug.ErrorHandler`](https://hexdocs.pm/plug/Plug.ErrorHandler.html).

Сначала откройте `lib/example/router.ex`, а затем напишите в этот файл следующее.

```elixir
defmodule Example.Router do
  use Plug.Router
  use Plug.ErrorHandler

  alias Example.Plug.VerifyRequest

  plug Plug.Parsers, parsers: [:urlencoded, :multipart]
  plug VerifyRequest, fields: ["content", "mimetype"], paths: ["/upload"]
  plug :match
  plug :dispatch

  get "/" do
    send_resp(conn, 200, "Добро пожаловать")
  end

  get "/upload" do
    send_resp(conn, 201, "Загружено")
  end

  match _ do
    send_resp(conn, 404, "Ой!")
  end

  defp handle_errors(conn, %{kind: kind, reason: reason, stack: stack}) do
    IO.inspect(kind, label: :kind)
    IO.inspect(reason, label: :reason)
    IO.inspect(stack, label: :stack)
    send_resp(conn, conn.status, "Something went wrong")
  end
end
```

You'll notice that at the top, we are now adding `use Plug.ErrorHandler`.

This plug catches any error, and then looks for a function `handle_errors/2` to call in order to handle it.

`handle_errors/2` just needs to accept the `conn` as the first argument and then a map with three items (`:kind`, `:reason`, and `:stack`) as the second.

You can see we've defined a very simple `handle_errors/2` function to see what's going on. Let's stop and restart our app again to see how this works!

Now, when you navigate to <http://127.0.0.1:8080/upload>, you'll see a friendly error message.

If you look in your terminal, you'll see something like the following:

```shell
kind: :error
reason: %Example.Plug.VerifyRequest.IncompleteRequestError{message: ""}
stack: [
  {Example.Plug.VerifyRequest, :verify_request!, 2,
   [file: 'lib/example/plug/verify_request.ex', line: 23]},
  {Example.Plug.VerifyRequest, :call, 2,
   [file: 'lib/example/plug/verify_request.ex', line: 13]},
  {Example.Router, :plug_builder_call, 2,
   [file: 'lib/example/router.ex', line: 1]},
  {Example.Router, :call, 2, [file: 'lib/plug/error_handler.ex', line: 64]},
  {Plug.Cowboy.Handler, :init, 2,
   [file: 'lib/plug/cowboy/handler.ex', line: 12]},
  {:cowboy_handler, :execute, 2,
   [
     file: '/path/to/project/example/deps/cowboy/src/cowboy_handler.erl',
     line: 41
   ]},
  {:cowboy_stream_h, :execute, 3,
   [
     file: '/path/to/project/example/deps/cowboy/src/cowboy_stream_h.erl',
     line: 293
   ]},
  {:cowboy_stream_h, :request_process, 3,
   [
     file: '/path/to/project/example/deps/cowboy/src/cowboy_stream_h.erl',
     line: 271
   ]}
]
```

В данный момент мы все еще возвращаем `500 Internal Server Error`. Мы можем настроить код состояния HTTP, добавив поле `: plug_status` к нашему исключению. Откройте `lib/example/plug/verify_request.ex` и добавьте следующее:

```elixir
defmodule IncompleteRequestError do
  defexception message: "", plug_status: 400
end
```

Перезагрузите сервер и обновите страницу, теперь вы получите ответ `400 Bad Request`.

This plug makes it really easy to catch the useful information needed for developers to fix issues, while being able to also give our end user a nice page so it doesn't look like our app totally blew up!

## Доступные модули Plug

Много модулей `Plug` доступно для использования сразу "из коробки".
Полный список можно найти в документации по `Plug` &mdash; [здесь](https://github.com/elixir-lang/plug#available-plugs).
