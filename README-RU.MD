# MSC MP — Руководство по серверу и скриптам

Полный туториал по созданию и запуску сервера, сборке скриптов и справочник по API. Для README мода (клиента) см. корневой репозиторий/релиз.

## Быстрый старт (сервер)
- Распакуйте сервер в папку `MSC-MP-Server/`.
- Запустите `MSC.MP.Server.exe` один раз — появится `server.cfg`.
- Остановите сервер, отредактируйте `server.cfg` и снова запустите.
- Проверка статуса: `http://<ip>:<http_port>/` (возвращает JSON с именем и количеством игроков).

## Требования
- Windows.
- .NET 8 Runtime (для запуска), .NET 8 SDK (для сборки из исходников).

## Структура сервера
```
MSC-MP-Server/
├── MSC.MP.Server.exe
├── MSC.MP.Server.dll
├── server.cfg              # конфиг сервера (создаётся при первом запуске)
└── scripts/                # DLL скриптов (необязательно)
    └── SampleMode.dll      # пример скрипта (если собираете из исходников)
```

## Конфигурация — server.cfg
```
name=My Cool Server   # имя сервера (отображается в HTTP JSON)
port=26015            # TCP порт сервера
http_port=26016       # порт HTTP-статуса
```
- Редактируйте файл и перезапускайте сервер для применения.
- TCP порт можно переопределить аргументом запуска: `MSC.MP.Server.exe 26020`.

## Запуск сервера
- Двойной клик по `MSC.MP.Server.exe`.
- Либо PowerShell из папки сервера:
```
.\MSC.MP.Server.exe
```

## HTTP‑статус
- Адрес: `http://<ip>:<http_port>/`
- Пример ответа:
```
{"name":"My Cool Server","players":3}
```

## Сборка из исходников
### Сервер
```
dotnet build "MSC-MP/Server/MSC.MP.Server.csproj" -c Release -v minimal
```
Артефакты: `MSC-MP/Server/bin/Release/`

### Скрипты (пример SampleMode)
```
dotnet build "MSC-MP/Server/scripts/SampleMode/SampleMode.csproj" -c Release -v minimal
```
Скопируйте полученный DLL в `MSC-MP-Server/scripts/` и перезапустите сервер.

## Как написать свой скрипт
Скрипт — это .NET 8 Class Library (DLL) с реализацией `MSC.MP.Abstractions.IServerScript`.

### Колбэки `IServerScript`
- `void OnServerStart(IServerApi api)` — старт сервера, выдаётся `api`.
- `void OnPlayerConnect(int id, string name)` — игрок подключился.
- `void OnPlayerDisconnect(int id, string name)` — игрок отключился.
- `bool OnPlayerText(int id, string name, string msg)` — текст, вернуть `false` чтобы подавить дефолтный бродкаст.
- `bool OnPlayerCommand(int id, string name, string cmd, string args)` — команда `/cmd args`, вернуть `true` если обработано.

Пример:
```
using MSC.MP.Abstractions;

public class MyMode : IServerScript
{
    private IServerApi _api;

    public void OnServerStart(IServerApi api)
    {
        _api = api;
        _api.SetServerName("My Server");
    }

    public void OnPlayerConnect(int id, string name)
    {
        _api.MsgTo(id, $"Welcome, {name}!", "#ffffff");
    }

    public void OnPlayerDisconnect(int id, string name) { }

    public bool OnPlayerText(int id, string name, string msg)
    {
        return true; // пропустить в общий чат
    }

    public bool OnPlayerCommand(int id, string name, string cmd, string args)
    {
        if (string.Equals(cmd, "id", System.StringComparison.OrdinalIgnoreCase))
        {
            _api.MsgTo(id, $"ID: {id}", "#aaaaaa");
            return true;
        }
        return false;
    }
}
```

## Справочник по API (`IServerApi`)
- Сообщения
  - `void MsgAll(string message, string color = null)` — всем.
  - `void MsgTo(int id, string message, string color = null)` — одному игроку.
- Поиск/инфо
  - `int? FindIdByName(string name)` — найти ID по нику (без регистра).
  - `int GetPlayerCount()` — число игроков онлайн.
  - `void SetServerName(string name)` — задать имя сервера (перезапишет `server.cfg`, попадёт в HTTP/`server_info`).
- Карта (debug‑объекты)
  - `int MapSpawn(string shape, float x, float y, float z, float sx, float sy, float sz, string color = null)` — заспавнить примитив, вернуть `mapId`.
  - `void MapClear(int mapId)` — удалить объект по `mapId`.
  - `void MapClearAll()` — удалить все объекты.
- Персистентное состояние
  - `void SetState(string key, string value)` — сохранить и разослать `state_set`, записать в `state.txt`.
  - `void RemoveState(string key)` — удалить и разослать `state_remove`.
  - `string GetState(string key)` — получить из памяти.

## Протокол
- Сервер → Клиент: `welcome`, `server_info`, `join`, `leave`, `chat`, `pos`, `map_spawn`, `map_clear`, `map_clear_all`, `state_set`, `state_remove`.
- Клиент → Сервер: `hello` (имя), `chat` (сообщения/команды), `pos` (позиция/углы).

## Отладка
- Консоль: `[id] name connected/disconnected`, `<name> msg`.
- HTTP ошибки: `HTTP status endpoint error: ...`.
- `state.txt` — персистентные пары ключ/значение рядом с exe.

## Релизы
- Для серверного пакета достаточно: `MSC.MP.Server.exe`, `MSC.MP.Server.dll`, `server.cfg`, `scripts/` (если используете).

## Лицензия
All rights reserved.
