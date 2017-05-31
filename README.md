# LIKES. High-load applications.


Мы рассматриваем архитектуру счётчиков на высоконагруженных интернет приложениях. 

Счётчики - это лайки, просмотры, посещения, репосты, количество новых сообщений, оповещений и т.д.

## Задача

Необходимо считывать количество лайков с данной страницы (поста) и предоставлять информацию пользователю в «реальном времени», т.е. время отклика сервера должно быть минимально.


## Архитектура

Различают монолитную архитектуру и архитектуру микросервисов. В первом случае выше надежность, согласованность кода, данных, во втором случае выше гибкость, доступность. Многие высоконагруженные приложения – социальные сети, используют микросервисную архитектуру. 

Мы выделили систему обработки счётчиков в отдельный микросервис, который снимает часть нагрузки с других сервисов, занимаясь хранением внутренних данных.

Данные счётчиков во многих социальных сетях актуальны постоянно. Поэтому прямое обращение к базе данных за каждым небольшим объёмом информации может стать узким горлом в работе системы. Memcache - эффиктивная технология для работы с key-value базами данных. Эта система позволит обращаться к актуальным данным значительно быстрее.


## Ограничения
### Отображение лайков
При достижении N лайков переходим к обобщенному отображению (например, 1.5К) на клиенте, но продолжаем увеличивать счетчик.
### Хранение счетчика лайков для поста
На начальном этапе проектирования микросервиса предполагаем, что пользователей будет немного и типа INT [диапазон без знака от 0 до 4294967295, 32-бит для MySQL, MongoDB] для поля счетчика будет достаточно.
Однако при приближении значения счетчика к предельной величине (с учетом, что таких постов будет немного) можно использовать следующий алгоритм.

Идея состоит в использовании нескольких полей для счетчика и суммировании значений из них.

Для поля счетчика можно хранить список ссылок на связанные с ним поля (пустой, если поле одно) и индекс в списке для данного поля в базе данных. При приближении к пределу будем использовать новое поле и указывать ссылку на него. Список можно хранить только с первым значением, для других указывать ссылку на первое поле и индекс в списке.
Также необходимо реализовать механизм отслеживания «приближений к пределу».
### Используемая память

Необходимо определить, сколько занимает БД с счетчиком лайков. На каждый лайк в БД хранится post_id, поле-счётчик, список пользователей. Общий размер списка пользователей - количество пар (post_id, user_id). Каждая занимает 1 int (4 байт), так как хранится только user_id в списке. Размером счётчика и идентификатора поста можно принебречь.
Таким образом можем рассчитать вместимость memcached или жесткого диска - т.е. количество записей, которое можем хранить в оперативной памяти или на одном жестком диске.
Например, для сервера с 48 Гб оперативной памяти получим 13 миллиардов записей.

## Межсервисное API

Микросервис лайков в большинстве сценариев будет взаимодействовать с другими микросервисами. Будут предоставляться следующие базовые функции.

#### Метод постановки лайка
|**Параметр**|**Описание**|
|------------|------------|
|post_id     |id поста, на который ставится лайк|
|post_owner_id|id владельца поста|
|like_owner_id|id пользователя, поставившего лайк|

Метод возвращает информацию об успехе или неудаче установки лайка. Этот метод API подразумевает, что параметры уже были провалидированы до вызова. Например функцию валидации можно возложить на FRONT SERVER или на отдельный микросервис безопасности.

Запрос из браузера попадает на FRONT-SERVER. Оттуда запрос идёт в POSTS microservice с валидацией отправленных данных ( есть ли такой пост, действительно ли указан id владельца поста). После успешной валидации запрос отправляется в LIKES microservice, откуда идут уведомления о лайке владельцу поста ( --> USERS microservice) и читателям поста (--> POSTS microservice). Далее вычисляется post_id_hash  (hash(post_id) % server_amount) и по нему идёт сохранение лайка на каком-то из master-серверов(MEMCACHE Master на схеме), ближайших к геолокации поста. Далее эта информация сначала поступает на "локальные" реплики, а затем и во все остальные центры, откуда впоследствии будет выдаваться пользователям.

#### Метод снятия лайка
|**Параметр**|**Описание**|
|------------|------------|
|post_id     |id поста, на который ставится лайк|
|post_owner_id|id владельца поста|
|like_owner_id|id пользователя, поставившего лайк|

Метод возвращает информацию об успехе или неудаче снятия лайка.

(аналогично описанию метода постановки лайка)

#### Метод получения количество лайков поста
|**Параметр**|**Описание**|
|------------|------------|
|post_id     |id поста, на который ставится лайк|

Метод возвращает число лайков у запрошенного поста. Если в хранилищах микросервиса лайков нет информации о лайках к этому посту, то возвращается 0, так как это означает, что посту еще не было поставлено не одного лайка.

Запрос из браузера попадает в POSTS microservice, в котором после успешной валидации мы находим "интересующий" нас пост (по post_id). Из POSTS microservice идём в LIKES microservice, оттуда в slave-реплики нашего хранилища, в которых по post_id получаем количество лайков(LIKES_COUNTER). 

#### Метод получения списка пользователей, поставивших лайк посту
|**Параметр**|**Описание**|
|------------|------------|
|post_id     |id поста, для которого запрашивается список|

Возвращает список, в котором содержаться id пользователей, поставивших лайк на запрошенный пост. В случае, если пост не найден в хранилищах микросервиса лайков будет возвращен пустой список.

(аналогично описанию метода получения количества лайков поста, только из хранилища по post_id получаем список пользователей, лайкнувших пост(LIKES_OWNERS_LIST))

#### Метод удаления информации о лайках поста
|**Параметр**|**Описание**|
|------------|------------|
|post_id     |id поста, для которого необходимо удалить всю информацию о лайках|

Данный метод может использоваться для освобождения памяти при полном удалении поста из системы.

Запрос из браузера попадает в POSTS microservice, в котором после успешной валидации мы находим пост, информацию о лайках которого мы хотим удалить. Из POSTS microservice идём в LIKES microservice, оттуда обращаемся в Master-реплику нашего хранилища, и для данного post_id обнуляем LIKES_COUNTER и LIKES_OWNERS_LIST. Далее это отправляем в "локальные" реплики, а дальше во все остальные.


## Проблемы

##### Проблема падения сервера memcache

Будем рассматривать кластер memcache, каждый сервер обрабатывает группу с номером hash(post_id) % server_amount. Проблема заключается в том, что в memcache могут содержаться несохранённые в основной базе данные, когда сервер падает.

Имеет смысл логирование операций до попадания их в базу данных. В случае отключения можно равномерно распределить пространство хэшей потерянного сервера между оставшимися машинами кластера и восстановить данные из лога.

При этом у нас усложняется алгоритм распределения хэшей по серверам. Это не большая проблема, если организовано запланированное перехэширование данных в момент спада нагрузки.

Восстановление из лога может являться самым долгим процессом. Но его можно выполнять параллельно с основной работой. Если пользователь обратится к потерянным хэшам на запись, то есть возможность записать данные, а по окончании восстановления объединить старые и новые данные. При обращении на чтение можно подгрузить данные из основной базы. В данном случае мы нарушаем consistency из CAP теоремы. Но это временные меры, которые могут не быть проблемными при решении некоторых задач.

##### Проблема 10 друзей

Рассматривается применение схемы в социальной сети. Требуется, чтобы читатели поста, приняв уведомление о поставленном лайке, знали, является ли его владелец другом или нет? Аналогичная ситуация при запросе поста с сервера. В таком случае проверяются все владельцы лайков для одного поста.

Предлагается следующее решение. Когда в микросервисе постов формируется уведомление (или пост), инициализируется некоторая структура  для хранения отношений между пользователями. Такая структура может быть двудольным графом отношений, либо булева матрица, неполная матрицы смежности: на строках которой стоят читатели поста, по колонкам - владельцы лайков. Отправляется запрос к микросервису пользователей, в котором проверяется для каждой ячейки матрицы, является ли пара пользователей друзьями. Микросервис постов, приняв эту информацию, в запросе к клиентам добавляет булево поле: друг или нет.

Возникает вопрос о времени выполнения этой проверки. Возможно имеет смысл разослать уведомления всем читателям поста, не дожидаясь микросервис пользователей, а информацию о друзьях отправить с некоторой задержкой.

##### Master - slave

Положим, что случились проблемы с master сервером (или кластером), после которых он не доступен для остальных участников процесса. При этом остались неотправленные данные в slave хранилища. 

Аналогично решению проблемы падения сервера memcache, можно применить логирование операций. Это позволит восстановить потерянную информацию на другом кластере и установить его как master. 

В качестве оптимизации можно использовать спецальный slave, который будет забирать себе информацию, как нормальный slave, но в случае падения мастера, может стать сам мастером, подгрузив недоотправленную инфу из лога. Это позволит не восстанавливать всю инфомацию с других хранилищ, а только из лога.

Можно сделать пул кластеров, которые будут работать по протоколу RAFT.



---

* [Алёна Смирнова](https://github.com/alsmirnova)
* [Георгий Кубрин](https://github.com/Georgiy0)
* [Владислав Анисимов](https://github.com/ShittyWizard)
* [Никита Супротивный](https://github.com/NSuprotivniy)

