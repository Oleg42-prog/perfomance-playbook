# Request ID и тайминги

[Содержание](../../README.md) · [Браузер](<../2. Браузер/2.1. Frontend.md>)

Request ID связывает один HTTP-запрос с записями Nginx, Apache и PHP. Каждый AJAX-запрос имеет свой ID. Для всей карточки CRM смотреть цепочку запросов в DevTools.

Ниже — изменения конфигурации и кода, не команды чтения состояния. Перед применением сохранить исходные файлы. Диагностическим логам настроить ротацию.

## Nginx

В `http` добавить формат и маску REST-путей: URL вебхуков содержит секреты даже без query string.

```nginx
map $uri $perf_path {
    default $uri;
    ~^/rest/ /rest/REDACTED;
}

log_format b24perf
    'time=$time_iso8601 rid=$request_id '
    'method=$request_method path=$perf_path '
    'status=$status upstream_status=$upstream_status '
    'rt=$request_time uct=$upstream_connect_time '
    'uht=$upstream_header_time urt=$upstream_response_time';
```

В нужном `server` / `location` добавить `access_log`, а в проксирующем `location` — передачу ID и заголовок ответа:

```nginx
access_log /var/log/nginx/b24_perf.log b24perf;
proxy_set_header X-Request-ID $request_id;
add_header X-Request-ID $request_id always;
```

Для прямого FastCGI вместо `proxy_set_header`:

```nginx
fastcgi_param HTTP_X_REQUEST_ID $request_id;
```

Сохранить существующие `proxy_set_header`, `fastcgi_param` и `add_header`: добавление директив в дочернем контексте может изменить наследование. Проверить все используемые маршруты, включая AJAX. Маску расширить, если другие пути тоже содержат секреты.

```bash
sudo nginx -t && sudo systemctl reload nginx
```

В DevTools → Network → Headers взять `X-Request-ID` и найти запись:

```bash
grep -F 'rid=0123456789abcdef0123456789abcdef' /var/log/nginx/b24_perf.log
```

`rt`, `uct`, `uht`, `urt` — секунды. Значения upstream могут быть `-` или содержать несколько попыток.

## Apache

В конфигурации нужного virtual host:

```apache
LogFormat "rid=%{X-Request-ID}i duration_us=%D status=%>s" b24perf
CustomLog /var/log/httpd/b24_perf.log b24perf
```

```bash
sudo httpd -t && sudo systemctl reload httpd
grep -F 'rid=0123456789abcdef0123456789abcdef' /var/log/httpd/b24_perf.log
```

`%D` — микросекунды, включая обработку PHP/backend. Имена службы и пути для Debian-подобных систем заменить на свои.

## PHP и SQL

Измерять выбранный фрагмент в закрытом диагностическом обработчике после проверки доступа. Не добавлять трекинг ко всем хитам через `init.php`. Пример требует PHP 7.3+ и штатного соединения Bitrix; не совмещать с уже работающим SQL Tracker.

В `try` поместить исследуемую операцию:

```php
$connection = \Bitrix\Main\Application::getConnection();
$tracker = $connection->startTracker(true);
$startedAt = hrtime(true);

try {
    // Здесь выполняется исследуемая операция.
} finally {
    $appMs = (hrtime(true) - $startedAt) / 1e6;
    $connection->stopTracker();
    $sqlMs = $tracker->getTime() * 1000;
    $sqlCount = $tracker->getCounter();
    $requestId = $_SERVER['HTTP_X_REQUEST_ID'] ?? '';
    $requestId = preg_match('/\A[0-9a-f]{32}\z/', $requestId) ? $requestId : '-';

    if (!headers_sent()) {
        header(sprintf(
            'Server-Timing: app;dur=%.3F, sql;dur=%.3F, sql_count;desc="%d"',
            $appMs,
            $sqlMs,
            $sqlCount
        ), false);
    }

    error_log(sprintf(
        '[B24PERF] rid=%s app_ms=%.3F sql_ms=%.3F sql_count=%d',
        $requestId,
        $appMs,
        $sqlMs,
        $sqlCount
    ));
}
```

`app` — время фрагмента, не всего PHP-запроса. SQL измерен только после старта трекера и только на этом соединении. `finally` выполняется при исключениях, но `exit` / fatal могут прервать сбор.

В DevTools → Network → Timing появятся метрики Server Timing, если заголовки ещё не отправлены. В shutdown-функции гарантировать заголовок уже нельзя. Для полного запроса и внешних API использовать профилировщик либо [трассировку](<../4. Мониторинг/4.1. Мониторинг.md#трассировка>).

## Как читать времена

| Измерение | Единицы | Включает |
| --- | --- | --- |
| Nginx `rt` | с | Обработку запроса и отправку клиенту |
| Nginx `urt` | с | Ответ backend |
| Apache `%D` | мкс | Обработку PHP/backend |
| Server-Timing `app` | мс | Измеренный фрагмент, включая SQL |
| Server-Timing `sql` | мс | Отслеженные обращения к БД, включая клиентские ожидания |

Вложенные интервалы не складываются. `rt − urt` не равно собственному времени Nginx, а разница браузера и Nginx не равна времени VPN. Для сети сравнивать [LAN и VPN](<../2. Браузер/2.1. Frontend.md#сравнение-lan-и-vpn>).

Для отключения убрать добавленные директивы и PHP-фрагменты, проверить конфигурации и выполнить reload.

[Nginx log_format](https://nginx.org/en/docs/http/ngx_http_log_module.html) · [Apache LogFormat](https://httpd.apache.org/docs/2.4/mod/mod_log_config.html) · [SQL Tracker](https://docs.1c-bitrix.ru/pages/database/sql-tracker.html) · [Server-Timing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Server-Timing)
