# Простой пример разработки System Design для ETL системы

## Первоначальная постановка задачи (выдержка из ТЗ)
**Функционал и описание**

Продукт обеспечивает доступ пользователей Заказчиков к исходным данным, которые генерируются в процессе работы «ХХХ”. 

Доступ к данным осуществляется через абстрактное представление – «Витрина данных» (или по тексту далее – витрина).

Данные реплицируется в витрину из основной СУБД «ХХХ”, согласно заданным правилам.

Пользователи витрины: программист  DevExpress,  программист SQL, специалист по BI-аналитики.

**Цель продукта**
* обеспечить доступ Заказчика к данным «ХХХ”, для разработки кастомных отчётов силами Заказчика.
* оградить Заказчика от доступа к секретным данным СУБД «ХХХ” (полям и таблицам), которые представляют коммерческую тайну.
* исключить негативное влияние Заказчика на СУБД «ХХХ” и работоспособность «ХХХ” в целом, которое Заказчик может оказать в результате неконтролируемого прямого доступа к СУБД «ХХХ”.

**Требования к архитектуре ПО**

* При разработке архитектуры необходимо локализовать ПО в виде микросервиса для «ХХХ”.
* Планируется, что продукт будет работать на серверном оборудовании совместно с ПО «ХХХ”, либо на отдельном сервере (виртуальном или физическом), выделенным для витрины данных. Доступ Заказчика к витрине данных будет производится, в форме подключения к стандартной реляционной базе данных. Извлечение данных при помощи языка запросов SQL. 

Предлагается разделить витрину данных на две части:

* Первая «оперативная» часть – исходные данные в прямом виде, как и в СУБД «ХХХ”, для работы с оперативной информацией для простых запросов (которые используют малые объёмы данных ли имеют простую логику), планируемая глубина архива 3 месяцев до 1 года.
* Вторая «аналитическая» часть – преобразованные данные для работы с большим объёмом  данных (большое число записей, большой период выборки, сложные запросы с одновременным выводом множества значений). Предполагается, что данная часть витрины будет представлять из себя технологию OLAP (куб или столбцовая система управления базами данных или другое). Также данная часть будет использована для BI-аналитики в разрезе нескольких лет. Планируемая глубина архива 3 года.

Необходимо определить механизм повторной загрузки данных из СУБД «ХХХ” в витрину, при изменении данных в СУБД «ХХХ”. Но если, мы не сможем решить проблему с автоматической загрузкой измененных данных в витрину, то потребуется frontend для заказчика, для ручной миграции данных из СУБД «ХХХ” в витрину данных.

Разделение витрины данных на две части обусловлено технологическими ограничениями OLTP классического способа организации базы данных. 

Все технологии, используемые в продукте, должны быть на основе открытой лицензии.

На этапе технической проработки продукта необходимо понять все ограничения, которые будут наложены на конечный продукт, в зависимости от выбранной технологии.

*(**Важно: спроектировать дизайн без уточняющих требований, AS IS**)*


# System Design
## Первоначальный анализ системы “Витрина Данных”

Предложенная нотация C4

Сильно глубоко не проектировал, нет схемы выкладки на сервера с оркестрацией контейнеров.


### Компонентная диаграмма*
Состоит из:
* Сама система XXX
* Витрина данных 
* Пользователи системы XXX

![](https://github.com/masterofgears/sysd1/blob/main/xxx-1.png)

---

### Контейнерная диаграмма

Состоит из:
* Система ХХХ
* Механизм репликации данных
* Витрина данных
* Микросервисов, обеспечивающих работу
* Пользователей системы

![](https://github.com/masterofgears/sysd1/blob/main/xxx-2.png)

---

### LLD

Здесь описана более детальная архитектура предложенного решения

![](https://github.com/masterofgears/sysd1/blob/main/xxx-3.png)

---

### Основаная последовательность работы

![](https://github.com/masterofgears/sysd1/blob/main/xxx-main-seq.png)

---

### Обоснование

**Используемое ПО**
* Apache Kafka - транспорт для CDC и некое временное хранилище для архива записей с глубиной от 3 мес до 3 лет
* Debezium - CDC инструмент
* Source- и sink- Kafka Connector - обеспечивает взаимодействие Kafka и БД
* Apache Airflow - ETL механизм, позволяющей настраивать правила трансормации
* Python - неплохой язык, плюс “дружит” с Airflow
* HA-Proxy - балансировщик запросов от пользователей, по-факту интерфейс доступа

**Обеспечение стабильности**

Используется подход Change Data Capture - OutBox, который позволяет инкрементально “забирать” изменения в БД ХХХ, далее продюсировать их в топик Kafka, где Apache Airflow производит ETL процесс.

А именно:

* обфускация данных, для предотвращения попадания в Витрину секретных данных
* трансформация данных для предопределенных запросов BI
* Формирования вьюх (при необходимости)

**Обоснование:**

* в случае выхода  из строя или профилактической работы “Хранилища для витрины данных” все изменения сохраняются в Реплицирующем механизме. Далее, по восстановлении работы, данные передаются в Хранилище
* в случае  выхода из строя или проф. работа реплицирующиеся механизма, “Витрина данных” продолжает работу как независимый элемент. Далее, по восстановлении работы, данные передаются в Хранилище

Восстановление данных происходит из топиков Kafka по последнему сохраненному offset в служебном топике


**Ограничения**

* необходим коннектор к конкретным БД для Debezium
* необходим коннектор Airflow для конкретных БД
* вероятно будет нужна библиотека Python для трансформации стандартного SQL в формат Аналитической БД, в случае прямого запроса к аналитической БД, трансформация не нужна
* выбор языка Python обусловлен использованием ПО в конкретной реализации


