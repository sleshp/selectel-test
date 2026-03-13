# Отчет по отладке проекта Громаков Даниил Алексеевич


1. При первой попытке запуска приложения через Docker возникла ошибка подключения к базе данных.

Причина оказалась в опечатке в файле `config.py`:

    validation_alias="DATABSE_URL"

Нужно DATABASE

Параметр `validation_alias` также оказался избыточным, так как имя
переменной окружения совпадает с именем поля DATABASE_URL.
Поэтому данный параметр был полностью удалён.

2. При следующем запуске возникла ошибка в парсере.

Поле `city` в данных API иногда может иметь значение `None`.
При этом код пытался обратиться к атрибуту `name`:

    item.city.name

Это приводило к ошибке:

    AttributeError: 'NoneType' object has no attribute 'name'

Исправление --- добавлена проверка:

    city_name = item.city.name.strip() if item.city and item.city.name else None

После этого парсинг стал выполняться корректно.


3. Хотя вакансии успешно парсились, парсер запускался слишком часто и не
соответствовал значению, указанному в `.env`.

В файле `scheduler.py` интервал был указан в секундах:

    seconds=settings.parse_schedule_minutes

При этом переменная называется `parse_schedule_minutes` и содержит
значение в минутах.

Исправление:

    minutes=settings.parse_schedule_minutes

4. В парсере использовался `httpx.AsyncClient`, который не закрывался.

Было:

    client = httpx.AsyncClient(timeout=timeout)

Исправление --- использование контекстного менеджера:

    async with httpx.AsyncClient(timeout=timeout) as client:

5. Проблема что фоновая джоба считается успешно выполненной даже если возникло исключение
~~~
async def _run_parse_job() -> None:
    try:
        async with async_session_maker() as session:
            await parse_and_store(session)
    except Exception as exc:
        logger.exception("Ошибка фонового парсинга: %s", exc)
~~~
обязательно нужно добавить в конец `raise`
6. При попытке создать вакансию с уже существующим `external_id` API
возвращал статус `200 OK`.

Это некорректно, так как ресурс уже существует.

Исправление возвращается код:

    409 Conflict

7 В коде была логическая ошибка:

    if external_ids:
        existing_ids = set(...)
    else:
        existing_ids = {}

В одной ветке `existing_ids` имел тип `set`, а в другой --- `dict`.

Исправление:

    existing_ids = set()



8 В коде использовалось условие:

    ext_id = payload["external_id"]

    if ext_id and ext_id in existing_ids:

Если `external_id` равен `0`, условие не срабатывало.

Исправление:

    if ext_id is not None and ext_id in existing_ids:


## Итог

В ходе анализа и отладки проекта было найдено и исправлено **8 ошибок**.
После исправления всех проблем приложение корректно запускается и
функционирует.
