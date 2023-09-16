## DDD
Заметки о Domain driver design

**Domain driven design** (предметно-ориентированное проектирование) - набор практик разработки ПО, фокусирующийся на понятиях
предметной области (домена). Может включать в себя поддомены.

**Ubiquitous language** (вездесущий язык) - некий словарь терминов нашего домена. Формируется в результате обсуждения 
бизнес-области с аналитиками и доменными экспертами. 

**Bounded context** (ограниченный контекст) - подобласть конкректного домена, где используется один ubiquitous language.
Например, в компании может быть несколько отделов: отдел зарплат, отдел кадров и т.д. И каждому отделу интересны только 
отдельные характеристики сотрудников в этих отделах. Но при этом в каждом отделе используется одно понятие - "сотрудник".

Отличие поддоментов и bounded context: в идеале каждый поддомент = один bounded context. Однако может быть что большой поддомен
включает в себя несколько bounded context. Либо один bounded context включает в себя несколько поддоменов.

**Entities** (сущности) - модели, обитатели конкректного bounded context. Можно проверять как наличие ее в UI пользователя.
У каждой сущности должен быть уникальный идентификатор - ID. Может создаваться на стороне сервисов (application generated ID - 
java.util.UUID), базы (sequence), может приходить из другого bounded context (скорее всего это будет отдельный микросервис).

Раняя и поздняя генерация ID: sequence-формирование ID происходит в 2 запроса к базе - при первом мы его получаем, при втором выполняем
сохранение. Однако есть вариант и с автоикрементов колонок - ID приходит после сохранения. Это важно в случае event-driven систем,
когда важен порядок прихода событий.

**Value object** (объекты-значения) - описывает entities, количественное его измерение. Аналогия ссылочного типа данных в Java у объекта.

**Aggregated** (агрегаты) - представляет из себя кластер сущностей, логически связанных друг с другом. Изменение сущностей из агрегата мы делегируем 
самому агрегату. Создано преимущественно для одновременного обновления и атомарности. Пример из интернет-магазина: корзина и товар. 
Можем смоделировать так, что это два разных агрегата, две разных таблицы в бд. У каждого товара в этом случае ссылка на корзину.
Вдруг понадобилось иметь счетчик в каждой корзине. Тогда если 3 товара в корзине = счетчик корзины равен 3. В этом случае обновление
базы требует двух отдельных операций. По какой-то причине запрос может упасть. Получим неконсистентность данных.

**Invarians** (инварианты) - условия, которые всегда должны быть выполнены в нашей системы. Их нарушение приводит к неконсистентности
данных. Тогда мы ходим сгруппировать обновление счетчика и добавление корзины в атомарную операцию. Это приведет 
к согласованности данных. Таким образом, агрегат = граница консистентности, согласованности данных внутри агрегата.

**Изоляция транзакций** (transactional isolation) - методы разрешения одновременного выполнения транзакций. Их есть 4 уровня:
read uncommited, read commited, repeatable read, serialization. Есть базы, в которых нет уровня изоляций (Mongo DB). В этом случае 
можно применить isolation with optimistic locking. В этом случае можно каждому агрегату поставить ID, при считвании 
увеличивать на один выполнять изменение, и далее снова считывать версию, и если ранее считанный ID + 1 равен текущему + 1,
то сохранять изменения.

Не стоит делать агрегаты слишком большими, ибо тем больше агрегат, тем больше вероятность того что потоки с ним будут выполняться
параллельно. Это сильно нагружает оперативную память и сказывается на производительности. 

**Strict and eventual consistency** (строгая и конечная согласованность) - разные виды консистенции. Агрегат - строгая консистенция.
Если операция выполнена, то все сущности находятся в консистентности мгновенно. Конечная же согласованность - согласованность 
данных будет достигнута, но в конечном итоге. Т.е. может существовать временной промежуток, в котором сущности могут быть
в unconsistency состоянии. Пример: служба доставки. Два разных контекста - сервис доставки и сервис заказов товара, работающих с разными БД. 
Понятно, что в рамках одной транзакции мы не сможем обновить и статус доставки, и статус заказа, т.к. как минимум разные базы.

Размер кластера агрегата можно выбирать 2 путями: от одного большого, либо постепенным объединением сущностей.
Нам выгодно чтобы агрегаты были как можно меньшего размера => плюс производительности. Однако же бизнес может
требовать одновременного обновления, требуя строгую консистентность. 

## EDA
Заметки о Event-driven architecture

**Event-driven architecture** - подход к проектированию системы, основанный на событиях, или ивентах. Систему предлагается строить
как набор компонентов, производящих и потребляющих события. Eda - асинхронной подход к проектированию.

**Event (событие)** - структура данных, которая содержит информацию о том, что в системе произошло изменение.
Event - сообщение о происхождении события. Сообщение как бы несет в себе событие. При этом у сообщения  есть адресат,
а у события нет.

**Особенности и преимущества EDA**:
- Смена направленности коммуникаций. В синхронных сервисах сервис знает что дергать дальше по бизнес цепочке.
В асинхронных системах же сервис лишь испускает событие, не зная, что с ним случится дальше. Здесь API сервиса - набор 
событий, который они генерируют.

- Надежность системы - нам без разницы, доступен ли сервис, который должен обрабатывать event. 

- Разделение обязанностей CQRS - command and query responsibility segregation. Здесь запросы на чтение и запись разделяются.
Можно в зависимости от преобладающих операций масштабировать или сервисы на чтение, или на запись. Также можно поддерживать
собственные базы данных.

**Query** - запросы на чтение данных к агрегатам (сервисам).

**Command** - запросы на изменение данных в агрегатах (создание, изменение, удаление). Обычно порождают
события для eventual consistency всей системы. Команда может порождать один или несколько ивентов.

Важно обеспечить атомарность двух операций команды - изменение данных в бд и отправкой ивента. Также может нарушиться
порядок ивентов, которые летят в брокер сообщений. Т.е. в базе статус сначала меняется на in-progress, review, но события
уходят в обратном порядке. Таким образом, у подписчиков может сложиться другая картина произошедших действий. На помощь
приходит transactional outbox pattern.

**Transactional outbox pattern** - в рамках одной транзакции в бд мы и обновляем агрегат, и сохраняем набор ивентов для
этого обновления. Помимо этого нужен будет еще background-процесс, который будет смотреть на таблицу с ивентами и заниматься
их отправлением в БД. Он должен отправлять эти ивенты в нужном порядке в message-broker и отметить их отправнными. 
Порядок считывания нужно также гарантировать консюмерами, на стороне других агрегатов (осторожно с параллельностью на сервисе-консюмере).
Также message-broker надо выбрать, который поддерживает порядок сообщений.

Мы сказали, что в рамках одной транзакции и обновляем таблицу с агрегатом, и таблицу с ивентами. Однако не все БД поддерживают
мультитабличную атомарность (ACID-транзакции). Тогда нужно надеяться что в базе есть атомарное обновление одной сущности.
Атомарное обновление сущности - это когда есть документ, или рекорд, который мы набором действий можем изменить. Тогда мы включаем
список изменений прямо в документ помимо состояние агрегата. Следующий шаг - **event sourcing**, или хранение только ивентов агрегата
без его состояния. 


