# InternetArchiveUploader
GUI for uploading to the Internet Archive

Original Reddit thread: https://www.reddit.com/r/internetarchive/comments/1wl1c14/i_made_a_simple_gui_uploader_for_large_files_to/

Modified from u/Motorboy1975's original to fix an issue where files split into more than 1000 parts would not complete.


## Original readme:

Internet Archive IAS3 — Parallel Multipart Uploader

README / Руководство пользователя

Автор / Author: Motorboy

This README is bilingual: the Russian section comes first, followed by
the English section.

------------------------------------------------------------------------

РУССКАЯ ВЕРСИЯ

1. Что это за программа

InternetArchiveUploader.py — небольшой графический загрузчик для отправки больших
файлов в Internet Archive через его S3-совместимый интерфейс IAS3.

Программа предназначена прежде всего для больших архивов и других
файлов, для которых обычная загрузка через браузер неудобна или
ненадёжна.

Зачем вообще был создан этот загрузчик

Главная практическая задача программы — увеличить скорость загрузки
больших файлов в Internet Archive.

Обычная последовательная передача одного большого файла может
использовать соединение далеко не на полную мощность. В этом случае один
запрос передаёт данные со своей собственной скоростью, и итоговая
скорость может оказаться очень низкой.

Multipart Upload позволяет одновременно передавать несколько разных
частей одного и того же файла. Их скорости складываются, поэтому общая
скорость загрузки может вырасти в несколько раз.

На практике во время разработки этого загрузчика было получено
показательное сравнение на одном и том же соединении и файле:

  Параллельность     Наблюдаемая скорость
  ---------------- ----------------------
  1 поток                  около 256 KB/s
  10 потоков               около 2.5 MB/s

То есть в данном случае переход от 1 к 10 параллельным загрузкам дал
примерно десятикратное увеличение скорости.

Это не гарантированный результат для каждого пользователя: реальная
скорость зависит от интернет-канала, расстояния до серверов, текущей
нагрузки Internet Archive и допустимого сервером количества
одновременных запросов. Но именно возможность суммировать пропускную
способность нескольких параллельных запросов является главной причиной,
ради которой этот загрузчик использует Multipart Upload.

Важно понимать разницу:

-   разбиение на части само по себе не ускоряет загрузку;
-   параллельная передача нескольких частей позволяет увеличить
    суммарную скорость;
-   слишком большое количество потоков может упереться в ограничения
    сервера и вместо ускорения вызвать ошибки 503 SlowDown.

Главная особенность — Multipart Upload, то есть один большой файл
передаётся не одним огромным запросом, а по частям. Несколько частей
можно передавать одновременно.

Важно: программа не создаёт на диске отдельные файлы частей. Исходный
файл остаётся одним файлом. Программа читает из него нужные диапазоны
данных и отправляет их на сервер независимо друг от друга.

После передачи всех частей Internet Archive автоматически собирает их
обратно в один конечный файл.

------------------------------------------------------------------------

2. Для чего нужно разбитие на части

Представьте файл размером 10 GB.

Если отправлять его одним запросом, то один обрыв соединения может
испортить всю попытку загрузки, и придётся начинать заново.

При Multipart Upload файл, например, разбивается логически на части по
50 MiB:

    Большой файл
    │
    ├── Часть 1
    ├── Часть 2
    ├── Часть 3
    ├── ...
    └── Последняя часть

Каждая часть имеет свой номер. Она передаётся отдельно, и сервер
запоминает, какие части уже приняты.

Поэтому при продолжении загрузки программе не нужно передавать уже
загруженные части повторно.

Например:

    Всего:             192 части
    Уже принято:       155 частей
    Осталось передать:  37 частей

Программа продолжит с оставшихся частей.

Почему это особенно полезно для больших файлов

-   обрыв отдельной части не требует повторной передачи всего архива;
-   уже принятые сервером части сохраняются;
-   можно остановить программу и продолжить позже;
-   несколько частей можно передавать одновременно;
-   после завершения сервер собирает части в исходный файл.

------------------------------------------------------------------------

3. Как это работает внутри

Упрощённо алгоритм выглядит так:

    1. Выбирается исходный файл.
    2. Программа определяет его размер.
    3. Выбранный размер части используется для расчёта количества частей.
    4. Создаётся Multipart Upload на Archive.org.
    5. Для каждой части формируется отдельный запрос PUT.
    6. Несколько частей передаются параллельно.
    7. Как только одна часть заканчивается, на её место запускается следующая.
    8. Программа проверяет, какие части сервер действительно принял.
    9. Состояние сохраняется в JSON.
    10. После передачи всех частей выполняется Complete Multipart Upload.
    11. Archive.org собирает части в один конечный файл.
    12. После успешного завершения JSON состояния удаляется.

Параллельная загрузка здесь динамическая. Программа не ждёт завершения
всех текущих частей. Если один поток освободился, следующая ожидающая
часть запускается сразу.

------------------------------------------------------------------------

4. Что нужно для работы

Нужны:

-   Windows с установленным Python 3.x;
-   Tkinter (обычно входит в стандартную установку Python для Windows);
-   доступ в интернет;
-   учётная запись Internet Archive;
-   Access Key и Secret Key Internet Archive.

Библиотека requests отдельно устанавливать вручную не требуется: если
она отсутствует, программа пытается установить её автоматически при
запуске.

------------------------------------------------------------------------

5. Получение Access Key и Secret Key

В программе есть кнопка «Получить ключи» / “Get keys”.

Она открывает страницу ключей Internet Archive в браузере.

Перед этим необходимо войти в свою учётную запись на archive.org.

Ключи вводятся в поля:

-   Access Key
-   Secret Key

Секретный ключ в окне программы отображается скрытым.

Безопасность ключей

Файл состояния конкретной загрузки не содержит Access Key и Secret Key.

Последняя конфигурация интерфейса сохраняется отдельно в:

    ia_upload_states\last_upload_config.json

В Windows ключи сохраняются в этом файле в зашифрованном виде с
использованием Windows DPAPI. Это сделано для того, чтобы программа
могла восстановить последнюю конфигурацию без записи ключей в открытом
виде.

Не публикуйте свои Access Key / Secret Key вместе с программой, README,
скриншотами или исходным кодом.

------------------------------------------------------------------------

6. Поля программы

Файл

Полный путь к локальному файлу, который нужно загрузить.

Кнопка «Выбрать…» / “Browse…” открывает стандартный диалог выбора файла.

После выбора файла имя удалённого файла автоматически заполняется его
исходным именем.

Identifier

Идентификатор (identifier) элемента Internet Archive, в который будет
загружен файл.

Например:

    my-large-archive-2026

Identifier должен соответствовать элементу, который вы хотите
использовать на Archive.org.

Имя файла в Archive.org

Имя, под которым файл будет находиться внутри элемента Internet Archive.

Например:

    MyArchive.7z

Оно не обязано совпадать с локальным именем файла.

Размер части (MiB)

Размер одной multipart-части в MiB.

По умолчанию:

    50 MiB

Для обычного пользователя 50 MiB — хороший исходный вариант, и менять
это значение без необходимости не требуется.

Общее количество частей рассчитывается автоматически:

    Количество частей = ceil(размер файла / размер части)

Последняя часть может быть меньше остальных.

Параллельные загрузки

Это количество частей, которые программа пытается передавать
одновременно.

Например, при значении:

    6

одновременно работают до шести частей.

При значении:

    10

одновременно работают до десяти частей.

Программа поддерживает значения от 1 до 64, но возле поля специально
указано предупреждение:

  Рекомендуется не более 10 — при большем числе возможны ошибки сервера.

Это не технический запрет. Большие значения можно указать, но сервер
Internet Archive может начать возвращать ошибки ограничения
параллельности, например HTTP 503 SlowDown.

------------------------------------------------------------------------

7. Сколько потоков ставить

Для человека, который просто хочет надёжно загрузить большой архив,
удобно придерживаться следующего подхода:

  -----------------------------------------------------------------------
  Значение                            Что означает
  ----------------------------------- -----------------------------------
  1                                   Последовательная загрузка. Минимум
                                      нагрузки на сервер.

  2–4                                 Осторожная параллельная загрузка.

  6                                   Хороший базовый вариант.

  8–10                                Более высокая параллельность. Часто
                                      полезна для повышения скорости.

  11–64                               Разрешено программой, но возрастает
                                      вероятность серверных ограничений.
  -----------------------------------------------------------------------

Главное правило

Не следует считать, что больше потоков всегда означает больше скорости.

В какой-то момент упираются в возможности сервера, канала или сетевого
соединения. Если увеличить число параллельных загрузок слишком сильно,
скорость может не вырасти, а наоборот появятся ошибки 503 SlowDown.

Поэтому для начала разумно использовать:

    6

а затем при необходимости попробовать:

    8–10

Если цель — именно получить максимальную скорость, имеет смысл сравнить
6, 8 и 10 потоков на своём соединении. В некоторых условиях разница
может быть очень заметной — как в приведённом выше примере с ростом
примерно с 256 KB/s до 2.5 MB/s.

Если сервер начинает ругаться на слишком большую параллельность —
количество потоков нужно уменьшить.

------------------------------------------------------------------------

8. Что происходит при ошибке 503 SlowDown

Archive.org может временно ограничить количество одновременно
выполняемых запросов.

Тогда сервер может вернуть, например:

    HTTP 503
    SlowDown
    Please reduce your request concurrency.

Программа умеет автоматически повторять временные ошибки. Для кодов 429,
502, 503 и 504 используется повторная попытка с увеличивающейся
задержкой.

Поэтому одиночная временная ошибка сервера не обязательно означает
провал всей загрузки.

Несмотря на это, постоянно держать слишком большое количество потоков не
рекомендуется.

------------------------------------------------------------------------

9. Что такое ETA

ETA означает Estimated Time of Arrival, то есть приблизительное
оставшееся время загрузки.

Например:

    ETA: 00:12:35

означает, что по текущей средней скорости остаётся примерно 12 минут 35
секунд.

Это именно оценка. Она может меняться в зависимости от скорости
соединения и нагрузки Archive.org.

------------------------------------------------------------------------

10. Скорость загрузки

Программа показывает скорость передачи данных за текущий запуск.

Это важно при возобновлении старой загрузки.

Например, если до запуска программы уже было загружено 6 GB, эти 6 GB не
будут ошибочно включены в расчёт скорости текущего сеанса.

При продолжении программа рассчитывает скорость по данным, которые были
переданы во время текущего запуска.

------------------------------------------------------------------------

11. Возобновление незавершённой загрузки

Одна из главных функций программы — продолжение уже начатой загрузки.

Для этого рядом со скриптом автоматически создаётся папка:

    ia_upload_states

В ней находятся JSON-файлы состояния незавершённых заданий.

В состоянии сохраняется информация, необходимая для продолжения работы,
в том числе:

-   локальный файл;
-   Identifier;
-   удалённое имя;
-   размер файла;
-   размер части;
-   UploadId;
-   уже завершённые части.

При запуске программы

Если найдено одно незавершённое задание, его параметры восстанавливаются
автоматически.

Если найдено несколько незавершённых заданий, программа показывает
список и предлагает выбрать нужное.

Если незавершённых заданий нет, поля остаются пустыми.

Проверка состояния на сервере

Перед продолжением программа использует ListParts и проверяет, какие
части действительно уже приняты Archive.org.

Это важно: локальный JSON не считается единственным источником истины.
Если сервер уже получил часть, повторно отправлять её не нужно.

------------------------------------------------------------------------

12. Можно ли закрыть программу и продолжить завтра?

Да.

При остановке программы состояние сохраняется в JSON. После следующего
запуска незавершённую загрузку можно продолжить.

То же относится к аварийному завершению, если уже сохранённое состояние
осталось на диске и Multipart Upload на сервере всё ещё существует.

Важное условие

Не меняйте исходный файл во время незавершённой загрузки.

Программа продолжает читать части именно из этого файла. Если заменить
файл другим содержимым, особенно файлом того же размера, уже загруженные
части могут соответствовать старому содержимому, а новые — новому.

Если файл был намеренно заменён, безопаснее начинать новое задание.

------------------------------------------------------------------------

13. Совместимость со старым консольным загрузчиком

Программа умеет подхватывать состояние старого консольного загрузчика,
если рядом со скриптом присутствует старый JSON состояния, например:

    ia_upload_state.json

Это позволяет продолжить старую загрузку без повторной передачи уже
принятых частей.

При обнаружении подходящего старого UploadId программа сначала проверяет
его на Archive.org.

Если старый UploadId больше не существует, создаётся новый Multipart
Upload.

------------------------------------------------------------------------

14. Что такое UploadId

При создании Multipart Upload Archive.org выдаёт специальный
идентификатор — UploadId.

Он связывает все отдельные части с одной будущей конечной загрузкой.

Упрощённо:

    Identifier + имя файла
            │
            └── Multipart Upload
                  │
                  ├── Part 1
                  ├── Part 2
                  ├── Part 3
                  ├── ...
                  └── Part N

Только после команды завершения multipart-загрузки сервер собирает эти
части в итоговый файл.

------------------------------------------------------------------------

15. Что делать, если программа показывает ошибку worker

Иногда передача данных заканчивается, но HTTP-ответ от сервера приходит
с задержкой. В таких случаях программа не должна сразу считать часть
потерянной.

После завершения worker программа:

1.  ждёт небольшое время, чтобы получить его итоговое сообщение;
2.  проверяет ListParts;
3.  при необходимости повторяет проверку несколько раз;
4.  считает часть успешно принятой, если Archive.org уже показывает её в
    списке частей.

Это особенно важно для больших частей и нестабильной связи, когда данные
уже дошли до сервера, а подтверждение пришло чуть позже.

------------------------------------------------------------------------

16. Как читать информацию на экране

Во время загрузки программа показывает:

Загружено

    Загружено: 7.20 GB / 9.35 GB (77.00%)

Общий объём данных, который программа считает уже переданным или
подтверждённым.

Скорость

Текущая скорость загрузки за текущий запуск.

ETA

Оценка оставшегося времени.

Частей

Например:

    Частей: 150 / 192

Количество уже завершённых частей из общего количества.

Активные

Пример:

    Активные: [151:32.00 MB] [152:18.50 MB] [153:49.00 MB]

Это показывает, какие части сейчас передаются и сколько данных каждой из
них уже отправлено в текущей попытке.

JSON

Показывает путь к файлу состояния текущего задания.

------------------------------------------------------------------------

17. Цвета журнала

В нижней части окна находится журнал.

-   Белый — обычные сообщения;
-   зелёный — успешно принятые части и успешное завершение;
-   красный — ошибки;
-   серый — информационные сообщения.

Журнал полезен при диагностике проблем и особенно при публикации отчёта
об ошибке.

------------------------------------------------------------------------

18. Остановка загрузки

Кнопка «Остановить» / “Stop” не удаляет уже загруженные части.

Программа завершает текущую работу настолько аккуратно, насколько это
возможно, сохраняет JSON состояния и позволяет продолжить позже.

При следующем запуске программа снова проверит сервер и не будет
передавать уже принятые части повторно.

------------------------------------------------------------------------

19. Что будет после завершения всех частей

После того как все части переданы, программа не просто закрывает окно.

Она выполняет финальный этап:

    Complete Multipart Upload

Перед этим ещё раз получает список частей через ListParts и проверяет,
что все необходимые части действительно присутствуют.

Затем формируется XML со списком номеров частей и их ETag, который
отправляется Archive.org.

После подтверждённого завершения:

-   конечный файл считается собранным;
-   состояние задания удаляется;
-   программа сообщает об успешном завершении.

------------------------------------------------------------------------

20. Что делать после завершения

Не стоит сразу считать загрузку проверенной только потому, что программа
написала «Загрузка завершена».

Для важных архивов рекомендуется открыть страницу элемента на
Archive.org и:

1.  найти загруженный файл;
2.  дождаться, пока Archive.org закончит обработку, если она ещё идёт;
3.  скачать файл обратно или хотя бы проверить его доступность;
4.  для особенно важных архивов распаковать скачанную копию и проверить
    её содержимое.

Это позволяет убедиться не только в том, что multipart-загрузка
завершилась, но и в том, что итоговый файл действительно пригоден для
использования.

------------------------------------------------------------------------

21. Типичный порядок работы

Для первого запуска можно просто сделать так:

    1. Запустите InternetArchiveUploader.py
    2. Выберите файл
    3. Проверьте Identifier
    4. Проверьте удалённое имя файла
    5. Введите Access Key
    6. Введите Secret Key
    7. Оставьте размер части 50 MiB
    8. Установите 6 потоков
    9. Нажмите «Загрузить»

Если всё работает стабильно и хочется попробовать увеличить скорость,
можно проверить 8 или 10 потоков.

Не нужно менять сразу все параметры. Если задача — просто надёжно
загрузить большой архив, сначала лучше использовать значения по
умолчанию и посмотреть на результат.

------------------------------------------------------------------------

22. Рекомендации по размеру частей

50 MiB выбран как удобное значение по умолчанию.

В общем случае:

-   слишком маленькие части означают больше отдельных запросов;
-   слишком большие части означают, что при неудаче придётся повторно
    передавать больший объём данных;
-   при большом количестве потоков слишком маленький размер части также
    создаёт больше запросов к серверу.

Поэтому без конкретной причины менять размер части не стоит.

------------------------------------------------------------------------

23. Типичные проблемы

«No existing file» / «Не выбран существующий файл»

Путь к локальному файлу неверен или файл был перемещён.

«Identifier is not specified»

Не указан Identifier.

«Access Key and Secret Key are required»

Не введены оба ключа.

HTTP 503 SlowDown

Сервер временно ограничивает количество параллельных запросов. Программа
умеет повторять временные ошибки, но при регулярном появлении SlowDown
уменьшите число потоков.

«UploadId is no longer valid»

Старый Multipart Upload больше недоступен на сервере. Программа создаёт
новый Upload и начинает передачу отсутствующих частей заново.

При этом старый UploadId может остаться незавершённым на стороне
сервера, если его больше нельзя использовать.

Скорость внезапно меняется

Это нормально. Скорость зависит от интернет-канала, нагрузки сервера и
количества активных запросов.

Один worker завершился, но часть не видна сразу

Программа выполняет дополнительные проверки ListParts, потому что
подтверждение сервером может появиться не мгновенно.

------------------------------------------------------------------------

24. Файлы и папки, создаваемые программой

Рядом со скриптом создаётся:

    ia_upload_states\

В этой папке находятся:

    last_upload_config.json

Последняя конфигурация интерфейса. В Windows ключи хранятся в
зашифрованном виде.

И файлы состояния отдельных заданий, например:

    some-identifier__archive.7z__xxxxxxxxxxxx.json

В них хранится информация для возобновления загрузки.

После успешного завершения конкретной загрузки её JSON состояния
удаляется.

------------------------------------------------------------------------

25. Ограничения и важные замечания

-   Это не универсальный S3-клиент. Программа рассчитана именно на
    используемый интерфейс Internet Archive IAS3.
-   Программа зависит от доступности соответствующего S3 endpoint и от
    поведения Multipart API Internet Archive.
-   Серверные ограничения по параллельности могут меняться.
-   Слишком большое количество потоков не гарантирует более высокую
    скорость.
-   Не заменяйте исходный файл во время незавершённой загрузки.
-   Не публикуйте свои Access Key и Secret Key.
-   Для критически важных архивов всегда проверяйте итоговый файл после
    завершения загрузки.

------------------------------------------------------------------------

ENGLISH VERSION

1. What is this program?

InternetArchiveUploader.py is a small graphical uploader for sending large files
to the Internet Archive through its S3-compatible IAS3 interface.

It is primarily intended for large archives and other files for which a
single browser upload is inconvenient or unreliable.

Why was this uploader created?

The main practical purpose of this program is to increase the upload
speed of large files to Internet Archive.

A normal sequential upload may fail to use the available connection
capacity efficiently. In that case, one request transfers data at its
own limited rate, and the resulting upload speed can be surprisingly
low.

Multipart Upload makes it possible to transfer several different parts
of the same file at the same time. Their throughput can add up, so the
total upload speed can become several times higher.

During development, the following real-world comparison was observed on
the same connection and file:

  Parallel uploads     Observed speed
  ------------------ ----------------
  1 upload             about 256 KB/s
  10 uploads           about 2.5 MB/s

In this case, increasing parallelism from 1 to 10 produced an
approximately tenfold increase in upload speed.

This is not a guaranteed result for every user. Actual throughput
depends on the internet connection, network path, current Internet
Archive load, and the number of concurrent requests accepted by the
server. The key reason for using Multipart Upload here is the ability to
combine the throughput of multiple simultaneous requests.

The important distinction is:

-   splitting the file into parts does not make the upload faster by
    itself;
-   uploading several parts in parallel can increase the total
    throughput;
-   too many parallel uploads can hit server limits and cause errors
    such as 503 SlowDown instead of improving speed.

The main feature is Multipart Upload: instead of sending one huge
request, the file is transferred as a sequence of numbered parts.
Several parts can be uploaded at the same time.

Important: the program does not create separate part files on your disk.
The original file stays intact. The program reads the required ranges
from that file and uploads them independently.

After all parts have been uploaded, Internet Archive combines them into
the final file.

------------------------------------------------------------------------

2. Why split a large file into parts?

Imagine a 10 GB file.

With a single upload request, one broken connection can ruin the whole
attempt and force you to start again.

With Multipart Upload, the file is logically divided into smaller parts,
for example 50 MiB each:

    Large file
    │
    ├── Part 1
    ├── Part 2
    ├── Part 3
    ├── ...
    └── Last part

Each part has its own number. It is uploaded separately, and the server
keeps track of the parts it has accepted.

This means that when an upload is resumed, already accepted parts do not
have to be transferred again.

For example:

    Total parts:        192
    Already accepted:   155
    Remaining:           37

The program continues with the remaining parts.

Why this is useful for large files

-   a failed part does not require re-uploading the entire archive;
-   already accepted parts remain available to the multipart upload;
-   the program can be stopped and resumed later;
-   several parts can be transferred simultaneously;
-   the server assembles the parts into the final file at the end.

------------------------------------------------------------------------

3. How does it work?

The simplified workflow is:

    1. Select the local file.
    2. Determine its size.
    3. Calculate the number of parts from the selected part size.
    4. Create a Multipart Upload on Archive.org.
    5. Create a separate PUT request for each part.
    6. Upload several parts in parallel.
    7. Start the next waiting part immediately when a slot becomes free.
    8. Check which parts were actually accepted by the server.
    9. Save the state to JSON.
    10. After all parts are present, send Complete Multipart Upload.
    11. Archive.org assembles the final file.
    12. Remove the job state JSON after successful completion.

The upload pool is dynamic. The program does not wait for all currently
active parts to finish before starting new ones.

------------------------------------------------------------------------

4. Requirements

You need:

-   Windows with Python 3.x installed;
-   Tkinter (normally included with the standard Python installer for
    Windows);
-   an internet connection;
-   an Internet Archive account;
-   an Internet Archive Access Key and Secret Key.

You do not normally need to install requests manually. If it is missing,
the program tries to install it automatically at startup.

------------------------------------------------------------------------

5. Getting Access Key and Secret Key

The program has a “Get keys” button.

It opens the Internet Archive key-management page in your browser.

You must be signed in to your archive.org account first.

Enter:

-   Access Key
-   Secret Key

The Secret Key field is masked in the GUI.

Key security

The JSON file for an individual upload does not contain your Access Key
or Secret Key.

The last GUI configuration is stored separately in:

    ia_upload_states\last_upload_config.json

On Windows, the keys are stored there in encrypted form using Windows
DPAPI. This allows the program to restore the previous configuration
without writing the raw keys to the file.

Do not publish your Access Key or Secret Key with the program, README,
screenshots, or source code.

------------------------------------------------------------------------

6. Program fields

File

The full path to the local file to be uploaded.

The Browse… button opens the standard file selection dialog.

After selecting a file, the remote filename is automatically filled with
the local filename.

Identifier

The Internet Archive item identifier where the file will be uploaded.

For example:

    my-large-archive-2026

Remote filename on Archive.org

The filename that will appear inside the Internet Archive item.

For example:

    MyArchive.7z

It does not have to be identical to the local filename.

Part size (MiB)

The size of each multipart part.

Default:

    50 MiB

For a normal user, 50 MiB is a good starting value, and changing it is
usually unnecessary.

The total number of parts is calculated automatically:

    Number of parts = ceil(file size / part size)

The last part may be smaller than the others.

Parallel uploads

This is the maximum number of parts the program attempts to upload at
the same time.

For example:

    6

means up to six active parts.

    10

means up to ten active parts.

The program accepts values from 1 to 64, but the GUI recommends 10 or
fewer because higher concurrency can trigger server-side throttling such
as HTTP 503 SlowDown.

This is not a hard program limit. Higher values are allowed, but they
may be less reliable.

------------------------------------------------------------------------

7. How many parallel uploads should I use?

A practical starting guide is:

  -----------------------------------------------------------------------
  Value                               Meaning
  ----------------------------------- -----------------------------------
  1                                   Sequential upload. Lowest server
                                      load.

  2–4                                 Conservative parallel upload.

  6                                   Good default starting point.

  8–10                                Higher concurrency; may improve
                                      throughput.

  11–64                               Allowed by the program, but server
                                      throttling becomes more likely.
  -----------------------------------------------------------------------

The important rule

More parallel uploads do not automatically mean more speed.

At some point the bottleneck becomes the server, your internet
connection, or the network path. Excessive concurrency can reduce
reliability and cause errors such as 503 SlowDown instead of increasing
throughput.

A sensible progression is:

    6 → 8 → 10

If your main goal is maximum throughput, compare 6, 8, and 10 on your
own connection. In some environments the difference can be very large —
as in the example above, where throughput increased from about 256 KB/s
to about 2.5 MB/s.

Use a higher number only when it is actually beneficial in your
environment.

------------------------------------------------------------------------

8. What happens with HTTP 503 SlowDown?

Internet Archive may temporarily limit the number of concurrent
requests.

The server may return something like:

    HTTP 503
    SlowDown
    Please reduce your request concurrency.

The program automatically retries temporary errors. Codes 429, 502, 503,
and 504 are retried with increasing delays.

Therefore, one temporary server-side throttling event does not
necessarily fail the whole upload.

However, if SlowDown happens repeatedly, reduce the number of parallel
uploads.

------------------------------------------------------------------------

9. What does ETA mean?

ETA means Estimated Time of Arrival — the estimated remaining upload
time.

For example:

    ETA: 00:12:35

means that, according to the current session throughput, about 12
minutes and 35 seconds remain.

It is only an estimate and can change with network conditions and server
load.

------------------------------------------------------------------------

10. Upload speed

The displayed speed is calculated from data transferred during the
current program run.

This matters when resuming an old upload.

For example, if 6 GB had already been uploaded before starting the
program, those 6 GB are not incorrectly counted as data transferred
during the current session.

The speed reflects the data transferred during the current run.

------------------------------------------------------------------------

11. Resuming an unfinished upload

One of the main features is the ability to continue an interrupted
multipart upload.

The program automatically creates:

    ia_upload_states

next to the script.

This directory contains JSON state files for unfinished jobs.

The state stores information such as:

-   local file;
-   Identifier;
-   remote filename;
-   file size;
-   part size;
-   UploadId;
-   completed parts.

At startup

If one unfinished job is found, its parameters are restored
automatically.

If multiple unfinished jobs are found, the program displays a selection
window.

If no unfinished jobs are found, the fields remain empty.

Server-side verification

Before resuming, the program uses ListParts to determine which parts
Internet Archive has actually accepted.

This is important because the local JSON is not treated as the only
source of truth. If the server already has a part, the program does not
transfer it again.

------------------------------------------------------------------------

12. Can I stop the program and continue tomorrow?

Yes.

When the upload is stopped, the state is saved to JSON. After starting
the program again, the unfinished upload can be resumed.

The same principle also applies after an unexpected program termination,
provided the saved state remains on disk and the multipart upload still
exists on the server.

Important condition

Do not replace or modify the source file during an unfinished upload.

The program continues reading data from that file. If the file is
replaced with different content — especially with another file of the
same size — previously uploaded parts can belong to the old content
while new parts belong to the new content.

If the source file has intentionally been replaced, start a new upload
job.

------------------------------------------------------------------------

13. Compatibility with the old console uploader

The GUI can import state from the older console uploader when a legacy
state file is present next to the script, for example:

    ia_upload_state.json

This makes it possible to continue an existing upload without
retransmitting parts that Archive.org has already accepted.

When a suitable old UploadId is found, the program verifies it with
Archive.org first.

If the old UploadId no longer exists, a new Multipart Upload is created.

------------------------------------------------------------------------

14. What is an UploadId?

When a Multipart Upload is created, Archive.org returns a special
identifier called an UploadId.

It associates all individual parts with one future final file.

Simplified:

    Identifier + remote filename
            │
            └── Multipart Upload
                  │
                  ├── Part 1
                  ├── Part 2
                  ├── Part 3
                  ├── ...
                  └── Part N

Only after the multipart completion request does the server assemble the
final file.

------------------------------------------------------------------------

15. What if a worker finishes but the part is not immediately visible?

Sometimes data transfer completes before the final HTTP response or
server-side part listing becomes visible.

The program therefore does not immediately assume that the part was
lost.

After a worker finishes, it can:

1.  briefly wait for the worker’s final queue message;
2.  check ListParts;
3.  repeat the check several times if necessary;
4.  accept the part as successful when Archive.org reports it in the
    part list.

This protects against timing races where the data has reached the server
but confirmation becomes visible slightly later.

------------------------------------------------------------------------

16. Understanding the status area

During upload the program shows:

Uploaded

    Uploaded: 7.20 GB / 9.35 GB (77.00%)

The amount of data currently considered transferred or server-confirmed.

Speed

The upload speed for the current run.

ETA

Estimated remaining time.

Parts

For example:

    Parts: 150 / 192

The number of completed parts out of the total number of parts.

Active

For example:

    Active: [151:32.00 MB] [152:18.50 MB] [153:49.00 MB]

This shows which parts are currently being transferred and how much data
of the current attempt has been sent for each active part.

JSON

Shows the path to the state file for the current upload job.

------------------------------------------------------------------------

17. Log colors

The lower part of the window contains the log.

-   White — normal messages;
-   green — accepted parts and successful completion;
-   red — errors;
-   gray — informational messages.

The log is useful for troubleshooting and for providing a clear error
report if something goes wrong.

------------------------------------------------------------------------

18. Stopping an upload

The Stop button does not delete parts that have already been uploaded.

The program stops as cleanly as possible, saves the JSON state, and
allows the job to be resumed later.

On the next run, the program checks the server again and skips parts
that are already present.

------------------------------------------------------------------------

19. What happens after all parts are uploaded?

The program does not simply stop after transferring the last part.

It performs the final multipart step:

    Complete Multipart Upload

Before that, it obtains the part list again using ListParts and checks
that all required parts are present.

Then it sends an XML document containing the part numbers and their
ETags to Internet Archive.

After successful confirmation:

-   the multipart upload is completed;
-   the final file is assembled by the server;
-   the job state file is removed;
-   the program reports successful completion.

------------------------------------------------------------------------

20. What should I do after completion?

For important files, do not rely only on the program’s success message.

Open the Internet Archive item page and:

1.  find the uploaded file;
2.  wait for any remaining Archive.org processing;
3.  download the file again or at least verify that it is accessible;
4.  for important archives, unpack the downloaded copy and verify the
    contents.

This checks not only that the multipart operation completed, but also
that the final file is actually usable.

------------------------------------------------------------------------

21. Typical first-time workflow

For a first upload, you can simply do this:

    1. Run InternetArchiveUploader.py
    2. Select the file
    3. Check the Identifier
    4. Check the remote filename
    5. Enter Access Key
    6. Enter Secret Key
    7. Leave Part size at 50 MiB
    8. Set Parallel uploads to 6
    9. Click Upload

If the upload is stable and you want to test higher throughput, try 8 or
10 parallel uploads.

There is usually no need to change all parameters at once.

------------------------------------------------------------------------

22. Part-size recommendations

50 MiB is used as the default because it is a practical compromise.

In general:

-   very small parts mean more individual requests;
-   very large parts mean more data has to be retransmitted when a part
    fails;
-   with many parallel workers, very small parts also create more server
    requests.

Unless you have a specific reason to change it, keeping the default is
recommended.

------------------------------------------------------------------------

23. Common problems

“No existing file”

The local file path is invalid or the file has been moved.

“Identifier is not specified”

The Identifier field is empty.

“Access Key and Secret Key are required”

One or both keys are missing.

HTTP 503 SlowDown

The server is temporarily limiting concurrent requests. The program
retries temporary errors, but if SlowDown happens repeatedly, reduce the
number of parallel uploads.

“UploadId is no longer valid”

The previous Multipart Upload is no longer available on the server. The
program creates a new upload and transfers the missing parts again.

The old UploadId may remain as an unfinished server-side upload if it
can no longer be resumed.

Upload speed changes unexpectedly

This is normal. Throughput depends on your internet connection, server
load, and the number of active requests.

A worker finished but the part is not visible immediately

The program performs additional ListParts checks because server
confirmation may not become visible instantly.

------------------------------------------------------------------------

24. Files and folders created by the program

The program creates this directory next to the script:

    ia_upload_states\

It contains:

    last_upload_config.json

The last GUI configuration. On Windows, credentials are stored there in
encrypted form.

It also contains state files for individual jobs, for example:

    some-identifier__archive.7z__xxxxxxxxxxxx.json

These contain the information required to resume an upload.

After a successful upload, the state JSON for that job is deleted.

------------------------------------------------------------------------

25. Limitations and important notes

-   This is not a general-purpose S3 client. It is designed for the
    Internet Archive IAS3 workflow used by this uploader.
-   The program depends on the availability and behavior of the Internet
    Archive S3 endpoint and Multipart API.
-   Server-side concurrency limits may change over time.
-   Increasing the number of parallel workers does not guarantee higher
    speed.
-   Do not replace the source file during an unfinished upload.
-   Do not publish your Access Key or Secret Key.
-   For important archives, always verify the final file after the
    upload completes.

------------------------------------------------------------------------

License / Лицензия

No separate license is declared in this README. Add your preferred
license here before publishing the script publicly.
