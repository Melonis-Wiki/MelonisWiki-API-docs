# MelonisWiki API

Официальная русскоязычная документация публичного API [MelonisWiki](https://melonis.wiki).

API предоставляет чтение статей и разделов в формате JSON. Авторизация и API-ключ не требуются. Основной эндпоинт:

```text
GET https://melonis.wiki/api/fetch
```

API работает только на чтение. Для внешних интеграций следует использовать именно `/api/fetch`: у него единый формат ответа, поддержка пакетного получения статей, пагинация, CORS и явные лимиты запросов.

## Содержание

- [Быстрый старт](#быстрый-старт)
- [Режимы запросов](#режимы-запросов)
- [Параметры](#параметры)
- [Форматы данных](#форматы-данных)
- [Примеры ответов](#примеры-ответов)
- [Ошибки и HTTP-статусы](#ошибки-и-http-статусы)
- [Ограничение частоты запросов](#ограничение-частоты-запросов)
- [Кэширование](#кэширование)
- [CORS](#cors)
- [Примеры интеграции](#примеры-интеграции)
- [Рекомендации клиентам](#рекомендации-клиентам)

## Быстрый старт

Получить список разделов:

```bash
curl -sS 'https://melonis.wiki/api/fetch?sections'
```

Найти статьи:

```bash
curl -sS --get 'https://melonis.wiki/api/fetch' \
  --data-urlencode 'q=спавн'
```

Получить полную статью по числовому идентификатору:

```bash
curl -sS 'https://melonis.wiki/api/fetch?id=9'
```

Получить список статей раздела:

```bash
curl -sS 'https://melonis.wiki/api/fetch?section=information&limit=20&offset=0'
```

Каждый успешный ответ содержит `"ok": true` и поле `mode`, определяющее форму остальных данных.

## Режимы запросов

| Задача | Пример URL | `mode` в ответе |
|---|---|---|
| Получить разделы | `https://melonis.wiki/api/fetch?sections` | `sections` |
| Найти статьи | `https://melonis.wiki/api/fetch?q=экономика` | `search` |
| Найти статью по заголовку | `https://melonis.wiki/api/fetch?title=Melonis` | `article` |
| Получить статью по ID | `https://melonis.wiki/api/fetch?id=9` | `article` |
| Получить несколько статей | `https://melonis.wiki/api/fetch?ids=9,39` | `articles` |
| Получить статью по slug | `https://melonis.wiki/api/fetch?section=information&slug=melonis` | `article` |
| Получить статьи раздела | `https://melonis.wiki/api/fetch?section=information` | `section_list` |

### Приоритет режимов

Если в одном URL переданы параметры нескольких режимов, сервер выбирает первый подходящий режим в таком порядке:

1. `sections` или `mode=sections`;
2. `title`;
3. `q` или `query`;
4. `ids`;
5. `id`;
6. комбинация `section` + `slug`;
7. список по `section`.

Передавать параметры разных режимов одновременно не рекомендуется. `section` без `slug` означает список статей раздела. Один `slug` без `section` не создаёт режим и при отсутствии других параметров приведёт к ошибке `400`.

## Параметры

### Выбор режима

| Параметр | Формат | Назначение |
|---|---|---|
| `sections` | наличие параметра | Возвращает индекс разделов. Значение не учитывается: `?sections`, `?sections=1` и `?sections=false` включают один режим. |
| `mode` | `sections` | Альтернативная форма запроса списка разделов. |
| `q`, `query` | строка от 2 символов | Полнотекстовый поиск. После удаления пробелов по краям сервер использует не более 200 символов. Если есть оба параметра, используется `q`. |
| `title` | строка от 2 символов | Нечёткий поиск одной статьи по заголовку. Используется не более 200 символов. |
| `id` | целое число от 1 | Получение одной статьи по ID. |
| `ids` | ID через запятую или пробел | Пакетное получение до 50 уникальных статей. Например, `ids=9,39,42`. |
| `section` | строка | Slug раздела. Без `slug` возвращает список статей. |
| `slug` | строка | Slug статьи; применяется только вместе с `section`. |

Строковые значения необходимо кодировать как UTF-8 URL encoding. Например, безопаснее формировать поиск через `URLSearchParams`, а в `curl` использовать `--data-urlencode`.

### Управление составом ответа

| Параметр | Значения | Поведение |
|---|---|---|
| `fields` | `summary`, `full` | Выбирает краткую или полную статью. По умолчанию `full` для одной статьи и пакетного запроса; `summary` — для поиска и списка раздела. |
| `expand` | `1`, `true`, `yes` | Раскрывает wiki-ссылки в полном `content` и добавляет исходный текст в `content_raw`. Регистр не учитывается. |

Особенности:

- `fields=full` разрешён для поиска;
- `fields=full` запрещён для списка статей раздела и вернёт `400`;
- `expand` влияет только на ответы, содержащие полный текст;
- `fields` не меняет форму индекса разделов.

### Пагинация и сортировка списка раздела

Эти параметры используются только при запросе `?section=...` без `slug`:

| Параметр | Допустимые значения | По умолчанию |
|---|---|---|
| `limit` | целое число; фактический диапазон 1–100 | `50` |
| `offset` | целое число от 0 | `0` |
| `sort` | `updated`, `title`, `created` | `updated` |

`limit=0` приводится к `1`, а значения больше `100` — к `100`. Сортировка `updated` и `created` идёт от новых к старым, `title` — по возрастанию заголовка.

В ответе `section_list` поле `count` — общее количество статей раздела, а не размер текущей страницы. Следующую страницу можно запрашивать с `offset + articles.length`.

Параметры `limit`, `offset` и `sort` игнорируются в остальных режимах.

## Форматы данных

### Краткая статья (`summary`)

| Поле | Тип | Описание |
|---|---|---|
| `id` | `number` | Уникальный числовой ID статьи. |
| `slug` | `string` | Slug статьи внутри раздела. |
| `title` | `string` | Заголовок. |
| `section_slug` | `string` | Slug раздела. |
| `section_name` | `string` | Отображаемое название раздела. |
| `author_username` | `string` | Имя автора. |
| `created_at` | `string` | Дата создания, обычно ISO 8601 в UTC. |
| `updated_at` | `string` | Дата последнего изменения, обычно ISO 8601 в UTC. |
| `version` | `number` | Номер версии статьи. |
| `url` | `string` | Абсолютный URL или путь к статье. См. примечание ниже. |
| `score` | `number`, необязательное | Релевантность результата поиска; присутствует в кратких результатах поиска. |

`url` следует считать URI-ссылкой, которая может быть как абсолютной, так и относительной. Универсальное преобразование в абсолютный адрес:

```javascript
const absoluteArticleUrl = new URL(article.url, "https://melonis.wiki").href;
```

### Полная статья (`full`)

Полная статья содержит все поля `summary` и дополнительно:

| Поле | Тип | Описание |
|---|---|---|
| `content` | `string` | Текст статьи в Markdown. При `expand=1` wiki-ссылки в нём раскрыты. |
| `content_raw` | `string`, необязательное | Исходный Markdown до раскрытия wiki-ссылок; появляется только при `expand=1`. |

Контент следует считать недоверенным пользовательским вводом. При выводе HTML клиент обязан использовать безопасный Markdown-рендерер и санитизацию.

### Раздел

| Поле | Тип | Описание |
|---|---|---|
| `id` | `number` | ID раздела. |
| `slug` | `string` | Стабильный идентификатор раздела в URL. |
| `name` | `string` | Отображаемое название. |
| `description` | `string \| null` | Описание раздела. |
| `icon` | `string \| null` | Идентификатор иконки. |
| `article_count` | `number` | Количество статей в разделе. |

## Примеры ответов

Значения статей и счётчиков ниже иллюстративны и могут измениться вместе с содержимым вики.

### Индекс разделов

Запрос:

```http
GET /api/fetch?sections HTTP/2
Host: melonis.wiki
Accept: application/json
```

Ответ `200 OK`:

```json
{
  "ok": true,
  "mode": "sections",
  "sections": [
    {
      "id": 6,
      "slug": "information",
      "name": "Информация",
      "description": "О сервере Melonis: правила, IP, версия",
      "icon": "server",
      "article_count": 6
    }
  ]
}
```

### Поиск

Запрос:

```text
https://melonis.wiki/api/fetch?q=Melonis
```

Краткий ответ:

```json
{
  "ok": true,
  "mode": "search",
  "query": { "q": "Melonis" },
  "count": 1,
  "articles": [
    {
      "id": 9,
      "slug": "melonis",
      "title": "Melonis",
      "section_slug": "information",
      "section_name": "Информация",
      "author_username": "example",
      "created_at": "2026-05-31T12:17:36.000Z",
      "updated_at": "2026-06-04T21:36:40.000Z",
      "version": 5,
      "url": "/article/9",
      "score": 100
    }
  ]
}
```

Количество найденных статей возвращается в `count`. Отдельной пагинации поиска в текущем контракте нет.

### Одна полная статья

Запрос:

```text
https://melonis.wiki/api/fetch?id=9
```

Ответ:

```json
{
  "ok": true,
  "mode": "article",
  "article": {
    "id": 9,
    "slug": "melonis",
    "title": "Melonis",
    "section_slug": "information",
    "section_name": "Информация",
    "author_username": "example",
    "created_at": "2026-05-31T12:17:36.000Z",
    "updated_at": "2026-06-04T21:36:40.000Z",
    "version": 5,
    "url": "/article/9",
    "content": "# Melonis\n\nТекст статьи в Markdown."
  }
}
```

Для краткой формы добавьте `fields=summary`:

```text
https://melonis.wiki/api/fetch?id=9&fields=summary
```

### Статья с раскрытыми wiki-ссылками

```text
https://melonis.wiki/api/fetch?id=9&expand=1
```

В ответе:

```json
{
  "ok": true,
  "mode": "article",
  "article": {
    "id": 9,
    "content": "Текст с раскрытым содержимым связанной статьи.",
    "content_raw": "Текст с [[wiki-ссылкой]]."
  }
}
```

Остальные стандартные поля статьи в примере сокращены. Раскрытие ссылок увеличивает объём ответа и стоимость обработки, поэтому его стоит включать только при необходимости.

### Пакет статей

Запрос:

```text
https://melonis.wiki/api/fetch?ids=9,39,999999&fields=summary
```

Ответ:

```json
{
  "ok": true,
  "mode": "articles",
  "count": 2,
  "articles": [
    { "id": 9, "title": "Melonis", "url": "/article/9" },
    { "id": 39, "title": "melonis.wiki", "url": "/article/39" }
  ],
  "not_found": [999999]
}
```

Остальные стандартные поля статей в примере сокращены. Порядок найденных статей соответствует порядку ID в запросе. Дубликаты ID удаляются. `not_found` появляется только тогда, когда хотя бы один ID не найден. Отсутствие отдельных статей не превращает пакетный запрос в ошибку `404`.

### Список статей раздела

Запрос:

```text
https://melonis.wiki/api/fetch?section=information&limit=2&offset=0&sort=title
```

Ответ:

```json
{
  "ok": true,
  "mode": "section_list",
  "section": {
    "slug": "information",
    "name": "Информация"
  },
  "count": 6,
  "limit": 2,
  "offset": 0,
  "articles": [
    { "id": 9, "slug": "melonis", "title": "Melonis" },
    { "id": 39, "slug": "meloniswiki", "title": "melonis.wiki" }
  ]
}
```

Остальные стандартные поля `summary` в примере сокращены.

## Ошибки и HTTP-статусы

Прикладная ошибка обычно имеет форму:

```json
{
  "ok": false,
  "error": "Статья не найдена",
  "code": "NOT_FOUND"
}
```

В ошибке валидации может присутствовать подсказка:

```json
{
  "ok": false,
  "error": "Неверный id",
  "hint": "положительное целое"
}
```

| HTTP-статус | Значение | Действие клиента |
|---:|---|---|
| `200` | Запрос выполнен. | Проверить `ok` и обработать данные. |
| `204` | Успешный preflight `OPTIONS`. | Тело отсутствует. |
| `400` | Нет обязательных параметров или параметр некорректен. | Исправить запрос; повтор без изменений бессмысленен. |
| `404` | Статья или раздел не найдены. | Не повторять автоматически. |
| `429` | Превышен лимит частоты. | Подождать число секунд из `Retry-After`. |
| `500` | Внутренняя ошибка, обработанная API; обычно `code: "INTERNAL"`. | Повторить позже с ограниченным backoff. |
| `503` | Ошибка маршрута или инфраструктуры, например недоступная БД. | Повторить позже; тело может иметь другую форму. |

Не следует считать, что любое ошибочное тело содержит `ok`, `code` или даже корректный JSON: сбой reverse proxy или платформы может произойти до запуска обработчика API. Надёжный клиент сначала проверяет HTTP-статус и `Content-Type`, а затем разбирает JSON.

Пример ответа `429`:

```http
HTTP/2 429 Too Many Requests
Retry-After: 17
Content-Type: application/json

{
  "ok": false,
  "error": "Слишком много запросов",
  "retry_after_sec": 17
}
```

## Ограничение частоты запросов

| Тип запросов | Лимит |
|---|---:|
| Поиск через `q` или `query` | 120 запросов в минуту на ключ клиента |
| Остальные режимы | 300 запросов в минуту на ключ клиента |

Лимитер работает в памяти процесса и является защитой best effort, а не гарантированной глобальной квотой между всеми экземплярами сервера. Клиентам всё равно необходимо:

- ограничивать параллелизм;
- кэшировать повторяющиеся чтения;
- при `429` соблюдать `Retry-After`;
- для `500` и `503` применять exponential backoff с jitter и ограничением числа повторов.

## Кэширование

API возвращает `Cache-Control` в зависимости от режима:

| Режим | Значение | Максимальный срок |
|---|---|---:|
| `sections` | `public, max-age=300, s-maxage=300` | 5 минут |
| `section_list` | `public, max-age=120` | 2 минуты |
| `article`, `articles` | `public, max-age=60` | 1 минута |
| `search` | `no-store` | не кэшировать |

Учитывайте эти заголовки в CDN и клиентском кэше. API не предоставляет отдельный параметр для принудительного обхода серверного или промежуточного кэша.

## CORS

Успешные ответы, прикладные ошибки, rate-limit ответы и ошибки маршрута, перехваченные обработчиком `/api/fetch`, содержат:

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Headers: Content-Type, Accept
```

Preflight-запрос поддерживается:

```bash
curl -i -X OPTIONS 'https://melonis.wiki/api/fetch'
```

Он возвращает `204 No Content`. Cookie и авторизационные заголовки API не нужны. При инфраструктурной ошибке до запуска маршрута CORS-заголовки не гарантируются.

## Примеры интеграции

### JavaScript / TypeScript

```javascript
const API_URL = "https://melonis.wiki/api/fetch";

async function searchMelonisWiki(query) {
  const url = new URL(API_URL);
  url.searchParams.set("q", query);

  const response = await fetch(url, {
    headers: { Accept: "application/json" },
  });

  const contentType = response.headers.get("content-type") ?? "";
  const data = contentType.includes("application/json")
    ? await response.json()
    : null;

  if (!response.ok || !data?.ok) {
    throw new Error(data?.error ?? `MelonisWiki API: HTTP ${response.status}`);
  }

  return data.articles.map((article) => ({
    ...article,
    url: new URL(article.url, "https://melonis.wiki").href,
  }));
}

const articles = await searchMelonisWiki("Melonis");
console.log(articles);
```

### Python

Требуется пакет `requests`:

```python
from urllib.parse import urljoin

import requests

BASE_URL = "https://melonis.wiki"

response = requests.get(
    f"{BASE_URL}/api/fetch",
    params={"section": "information", "limit": 20, "offset": 0},
    headers={"Accept": "application/json"},
    timeout=15,
)
response.raise_for_status()
data = response.json()

if not data.get("ok"):
    raise RuntimeError(data.get("error", "Неизвестная ошибка API"))

for article in data["articles"]:
    print(article["title"], urljoin(BASE_URL, article["url"]))
```

### Node.js: пакетная загрузка

```javascript
const url = new URL("https://melonis.wiki/api/fetch");
url.searchParams.set("ids", [9, 39].join(","));
url.searchParams.set("fields", "full");

const response = await fetch(url);
const data = await response.json();

if (!response.ok || !data.ok) {
  throw new Error(data.error ?? `HTTP ${response.status}`);
}

console.log(`Получено статей: ${data.count}`);
console.log(`Не найдены: ${(data.not_found ?? []).join(", ")}`);
```

## Рекомендации клиентам

1. Всегда задавайте тайм-аут сетевого запроса.
2. Проверяйте и HTTP-статус, и `ok` в JSON.
3. Кодируйте пользовательский ввод через `URLSearchParams`, `params` клиента или `--data-urlencode`.
4. Не полагайтесь на фиксированный порядок полей JSON и допускайте появление новых полей.
5. Не сохраняйте `score` как стабильную характеристику статьи: это метрика конкретного поискового запроса.
6. Разрешайте `article.url` относительно `https://melonis.wiki`, поскольку оно может быть относительным.
7. Используйте `fields=summary`, если текст статьи не нужен, и пакетный `ids` вместо множества одиночных запросов.
8. Соблюдайте `Cache-Control` и `Retry-After`.
9. Безопасно отображайте Markdown из `content`; не вставляйте преобразованный HTML без санитизации.
10. Не используйте непубличные маршруты сайта как стабильный API-контракт.

## Совместимость

У `/api/fetch` нет параметра версии в URL. Интеграции должны быть tolerant readers: игнорировать неизвестные поля и опираться только на документированные поля. Потенциально несовместимые изменения следует отслеживать по истории этого репозитория.

## Связанные ссылки

- Сайт: [https://melonis.wiki](https://melonis.wiki)
- API: [https://melonis.wiki/api/fetch?sections](https://melonis.wiki/api/fetch?sections)
- Исходный проект MelonisWiki: [github.com/ClapFy/MelonisWiki](https://github.com/ClapFy/MelonisWiki)

