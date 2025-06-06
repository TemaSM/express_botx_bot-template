# Bot Template Tutorial
 
## Введение

> :information_source: Инфо  
> Для взаимодействия с платформой **botx** используется библиотека **[pybotx](https://github.com/ExpressApp/pybotx)**. В **[документации](https://pypi.org/project/pybotx/)** можно посмотреть примеры её использования. Перед прочтением данного туториала следует с ней ознакомиться.

----

Вне зависимости от решаемых ботами задач, во всех повторяется один и тот же код.

Чтобы разработчикам не приходилось писать базовый однообразный код для каждого нового бота, существует шаблонный проект - Bot Template. Он задает структуру проекта и основной стек технологий. Это значительно упрощает разработку, позволяя сконцентрироваться на реализации бота.

----


## 1. Развертывание из шаблона и структура проекта

Для развертывания проекта необходимо установить [copier](https://github.com/copier-org/copier) с расширением `copier-templates-extensions`.
Пример:
1. [Устанавливаем](https://docs.astral.sh/uv/getting-started/installation/) улучшенный  менеджер пакетов `uv`/`uvx`:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```
2. Устанавливаем `copier` + `copier-templates-extensions`:
```bash
uv tool install copier --with copier-templates-extensions
```
### Инициируем проект из этого шаблона, используя copier:
```bash
copier copy https://github.com/ExpressApp/bot-template.git mybot --trust
```  

Структура шаблонного бота состоит из нескольких следующих пакетов и модулей:

```
.
├── app
│   ├── api            - реализация http роутов для приложения, включая необходимые для бота
│   ├── bot            - команды бота и вспомогательные функции для них
│   ├── caching        - классы и функции для работы с in-memory БД
│   ├── db             - модели, функции для работы с БД и миграции
│   ├── resources      - текстовые или файловые ресурсы бота
│   ├── schemas        - сериализаторы, енамы, доменные модели
│   ├── services       - сервисы с логикой (бизнес-логика)
│   ├── logger.py      - логгер
│   ├── main.py        - запуск сервера с инициализацией необходимых сервисов
│   └── settings.py    - настройки приложения
├── scripts            - скрипты для запуска тестов, форматеров, линтеров
├── tests              - тесты, структура которых соответствует структуре проекта, и хелперы для них
├── poetry.lock        - конфигурация текущих зависимостей. используется для их установки
├── pyproject.toml     - конфигурация зависимостей, мета информация проекта (название, версия, авторы и т.п.)
└── setup.cfg          - конфигурация линтеров и тестов
```

## 2. Запуск проекта

### Настройка окружения

1. Устанавливаем зависимости проекта через [poetry](https://github.com/python-poetry/poetry#poetry-dependency-management-for-python):
```bash
$ poetry install
```
2. Определяем переменные окружения в файле **`.env`**. Примеры переменных окружения находятся в файле **`example.env`**.
3. Запускаем `postges` и `redis` используя [docker-compose](https://docs.docker.com/compose/):
```bash
$ docker-compose -f docker-compose.dev.yml up -d
```
4. Применяем все миграции для инициализации таблиц с помощью [alembic](https://alembic.sqlalchemy.org/en/latest/tutorial.html):
```bash
$ alembic upgrade head
```
5. Запускаем бота как приложение [FastAPI](https://fastapi.tiangolo.com/tutorial/) через [gunicorn](https://fastapi.tiangolo.com/deployment/server-workers/?h=gunicorn#run-gunicorn-with-uvicorn-workers).
Флаг `--reload` используется только при разработке для автоматического перезапуска сервера при изменениях в коде:
```bash
$ gunicorn "app.main:get_application()" --worker-class uvicorn.workers.UvicornWorker
```
По необходимости добавить флаг `--workers` и их колличество, в данном случае 4 рабочих процесса:
```bash
$ gunicorn "app.main:get_application()" --worker-class uvicorn.workers.UvicornWorker --workers 4
```

----

## 3. Добавление нового функционала

### 3.1. Команды бота

#### Структура пакета команд
Команды бота находятся в пакете **`app.bot.commands`** и группируются в отдельные модули в зависимости от логики. Команды добавляются с помощью [коллекторов pybotx](https://expressapp.github.io/pybotx/development/collector/). 

Основные команды, такие как `/help` и системные команды, находятся в модуле **`common.py`**. Для команд, относящихся к определенной задаче, создается свой модуль. Например, для интеграции с Atlassian Jira будет создан модуль **`jira.py`**. В результате структура пакета **`app.bot.commands`** будет выглядеть так:

```
bot
├── commands
│   ├── common.py
│   ├── jira.py 
```

Если в модуле становится слишком много команд, следует разбить его на новые модули и сложить в один пакет с названием старого модуля. Например, так:

```
bot
├── commands
│   ├── common.py
│   ├── jira
│       ├── projects.py
│       ├── issues.py
```


#### Регистрация команд
Для добавления модуля с командами нужно импортировать `collector` в **`app/bot/bot.py`** и добавить его в инстанс бота: 
```python3
from app.bot.commands import common 

bot.include_handlers(common.collector)
```

----

### 3.2. Взаимодействие с БД

#### Создание новых моделей

Взаимодействовать с новыми таблицами можно через модели [sqlalchemy](https://www.sqlalchemy.org/). С примерами использования можно ознакомиться [тут](https://www.sqlalchemy.org/library.html#tutorials). Модели располагаются в пакете **`app.db.package_name`**. Там же хранятся `crud` функции и [репозитории](https://gist.github.com/maestrow/594fd9aee859c809b043). Структура пакета выглядит следующим образом: 
```
├── app
│   ├── db 
│       ├── migrations
│       ├── exampleapp
│           ├── repo.py - репозиторий/crud функции
│           ├── models.py - модели таблиц
```

Пример модели:
``` python
from sqlalchemy import Column, Integer, String
from app.db.sqlalchemy import Base

class ExampleModel(Base):
    __tablename__ = "examples"

    id: int = Column(Integer, primary_key=True, autoincrement=True)
    text: str = Column(String)
```  

Пример репозитория:
``` python
from sqlalchemy import insert
from app.db.sqlalchemy import session
from app.db.example.models import ExampleModel

class ExampleRepo:
    async def create(self, text: str) -> None:
        query = insert(ExampleModel).values(text=text)
        async with session.begin():
            await session.execute(query)
```

#### Создание новых миграций

Для генерации миграций используется [alembic](https://alembic.sqlalchemy.org/en/latest/). Все файлы миграции хранятся в директории **`app.db.migrations`**. Для генерации новой миграции необходимо создать модель `sqlalchemy` и выполнить команду:

```bash
$ alembic revision --autogenerate -m "migration message"
```

Новый файл миграции будет создан в следующей директории:
```
├── app
│   ├── db 
│       ├── migrations
│           ├── versions
│               ├── 0123456789ab_migration_message.py
```

Чтобы применить все миграции, следует выполнить команду:
```bash
$ alembic upgrade head
```
или:
```bash
$ alembic upgrade 1
```
для применения только одной миграции.

Для отмены одной миграции необходимо выолнить:
```bash
$ alembic downgrade -1
```

----

### 3.3. Сервисы и бизнес-логика

Вся бизнес-логика проекта выносится в пакет  **`app.services`**. Бизнес-логика - логика, характерная только для данного проекта. Туда же выносятся запросы, клиенты для использования API сторонних сервисов, обработка данных по заданным (в ТЗ) правилам.

Структура следующая:
```
├── app
│   ├── services
│   │     ├── errors.py - исключения, вызываемые в клиенте
│   │     ├── client.py - клиент для обращения к стороннему сервису 
```

----

### 3.4. Конфиги и переменные среды

Новые переменные среды можно добавить в класс `AppSettings` из файла `app/settings.py`. Если у переменной нет значения по умолчанию, то оно будет браться из файла `.env`.
Чтобы использовать эту переменную в боте, необходимо:
``` python
from app.settings import settings
...
settings.MY_VAR
```

> :information_source: Инфо
> Через переменные среды можно указывать окружения, в которых будет запускаться бот. `test`, `dev` или `prod`. Просто добавьте в файл `.env` переменную `APP_ENV=prod`.

----

## 4. Линтеры и форматирование кода

#### Запуск
Для запуска всех форматеров необходимо выполнить скрипт:
```bash
$ ./scripts/format
```

Для запуска всех линтеров необходимо выполнить скрипт:
```bash
$ ./scripts/lint
```

#### Описание 
* [black](https://github.com/psf/black)

Используется для форматирования кода к единому стилю: разбивает длинные строки, следит за отступами и импортами.

> :warning: Примечание  
> В некоторых моментах isort конфликтует с black. Конфликт решается настройкой файла конфигурации **`setup.cfg`**.

* [isort](https://github.com/timothycrosley/isort)

Используется для сортировки импортов. Сначала импорты из стандартных библиотек python, затем из внешних библиотек и в конце из модулей данного проекта.
Между собой импорты сортируются по алфавиту.

* [autoflake](https://github.com/myint/autoflake)

Используется для удаления неиспользуемых импортов и переменных.

* [mypy](https://github.com/python/mypy)

Используется для проверки типов. Помогает находить некоторые ошибки еще на стадии разработки. 

> :warning: Примечание  
> К сожалению, не все библиотеки поддерживают типизацию. Чтобы подсказать это **mypy** необходимо добавить следующие строки в файл конфигурации **`setup.cfg`**:

```
[mypy]

# ...

[mypy-your_library_name.*]
ignore_missing_imports = True
```

Некоторые же наоборот имеют специальные плагины для **mypy**, например **pydantic**:

```
[mypy]
plugins = pydantic.mypy

...

[pydantic-mypy]
init_forbid_extra = True
init_typed = True
warn_required_dynamic_aliases = True
warn_untyped_fields = True
```

* [wemake-python-styleguide](https://github.com/wemake-services/wemake-python-styleguide)

Используется для комплексной проверки. Анализирует допустимые имена перменных и их длину, сложность вложенных конструкций, правильную обработку исключений и многое другое. Для каждого типа ошибок есть свой уникальный номер, объяснение, почему так делать не стоит, и объяснение, как делать правильно. Список ошибок можно посмотреть [тут](https://wemake-python-stylegui.de/en/latest/pages/usage/violations/index.html).

> :information_source: Инфо  
> В некоторых редких случаях можно игнорировать правила линтера. Для этого необходимо либо прописать комментарий с меткой `noqa` на проблемной строке:
> ```python3
> var = problem_function()  # noqa: WPS999 
> ```
> либо указать `ignore` ошибки в **`setup.cfg`**:
> ```
> [flake8]
> # ...
> ignore =
>     # f-strings are useful
>     WPS305,
> ```
> Также можно исключать модули и пакеты.


----


## 5. Тестирование



### 5.1. Запуск и добавление тестов

Все тесты пишутся с помощью библиотеки [pytest](https://docs.pytest.org/en/latest/). Запустить тесты можно командой:

```bash
$ pytest
```

Во время тестирования поднимается docker-контейнер с БД. Порт выбирается свободный, поэтому запущенная локально БД не будет мешать. Если вы хотите запускать тесты используя вашу локальную БД, необходимо добавить в **`.env`** переменную `DB=1`, либо выполнить команду:
```bash
$ DB=1 pytest
```


> :information_source: Инфо  
> Поскольку **pytest** не умеет в асинхронные тесты, для работы с ними ему необходим плагин **[pytest-asyncio](https://github.com/pytest-dev/pytest-asyncio)**.

### 5.2. Покрытие

Покрытие показывает процент протестированного исходного кода, как всего, так и отдельных модулей. Покрытие помогает определить какие фрагменты кода не запускались в тестах. Для генерации отчетов покрытия используется плагин [pytest-cov](https://pytest-cov.readthedocs.io/en/latest/reporting.html).

Чтобы не прописывать все флаги каждый раз, можно использовать эти скрипты:
```bash
$ ./scripts/test
$ ./scripts/html-cov-test
```

Первый выводит отчет в терминале, второй генерирует отчет в виде `html`страниц с подсветкой непокрытых участков кода.
