# Оптимизация CRUD-тестирования административного API

## Контекст и задача

Административный API предоставляет возможность управления заданиями различных игр и упражнений посредством CRUD-операций.

- Проект поддерживает несколько игр и других упражнений.
- Для каждой игры необходимо реализовать стандартные CRUD-операции.
- Создание отдельных клиентов и сервисов для каждой игры приводит к дублированию однотипной CRUD-логики.

## Принятое решение

Между базовым API-клиентом (BaseApiClient) и специализированными клиентами игр введён дополнительный слой AdminCrudClient, содержащий общие методы CRUD-операций без привязки к конкретной игре.

Аналогичный подход применён к сервисному слою: AdminCrudService инкапсулирует общую логику выполнения CRUD-операций и работы с моделями.

Таким образом, общая функциональность реализуется один раз, а специализированные игровые клиенты и сервисы определяют только необходимые параметры, endpoint'ы и типы моделей.

## Архитектура
```
BaseApiClient
      |
AdminCrudClient
      |
      +-- Game1AdminClient
      +-- Game2AdminClient
      +-- Game3AdminClient
      +-- ...
```

```
BaseApiService
      |
AdminCrudService
      |
      +-- Game1AdminService
      +-- Game2AdminService
      +-- Game3AdminService
      +-- ...
```

## Применение

#### Общий CRUD Client

AdminCrudClient предоставляет общие методы для выполнения CRUD-операций:
- Создание сущности
- Получение списка сущностей
- Получение сущности по идентификатору
- Обновление сущности
- Удаление сущности

Клиент содержит несколько наборов методов:
- JSON - для стандартных запросов с JSON-телом
- Multipart - для запросов с передачей файлов и других multipart-данных
- Raw - для негативных тестов, требующих отправки произвольного содержимого запроса

Конструктор класса принимает параметры, определяющие endpoint'ы и имя параметра идентификатора:
```
    protected AdminCrudClient(
            String idParam,
            String createPath,
            String getAllPath,
            String getByIdPath,
            String updatePath,
            String deletePath) {

        this.idParam = idParam;
        this.createPath = createPath;
        this.getAllPath = getAllPath;
        this.getByIdPath = getByIdPath;
        this.updatePath = updatePath;
        this.deletePath = deletePath;
    }
```

Пример метода класса:
```
  public Response create(
          AdminSession session,
          Object body
  ) {
      return post(
          requireConfigured(createPath, "create-path"),
          body,
          ApiConfig.adminAccessTokenJsonSpec(session)
      );
  }
```

#### Game Client

Специализированный клиент игры наследуется от AdminCrudClient и определяет необходимые endpoint'ы.

Клиент игры не содержит собственной реализации CRUD-методов и использует функциональность базового CRUD-клиента.

Пример Client-класса игры:
```
public class AnagramAdminClient extends AdminCrudClient {

    private static final String ANAGRAMS = ...;
    private static final String ID = ...;
    private static final String ANAGRAMS_BY_ID = ...;

    public AnagramAdminClient() {
        super(
                ID,
                ANAGRAMS,
                ANAGRAMS,
                ANAGRAMS_BY_ID,
                ANAGRAMS_BY_ID,
                ANAGRAMS_BY_ID
        );
    }
}
```

#### Общий CRUD Service
AdminCrudService обеспечивает типизированный доступ к CRUD-операциям и инкапсулирует общую логику взаимодействия с клиентом.

Ответственность:
- Взаимодействие с CRUD-клиентом.
- Проверка ожидаемых HTTP-статусов.
- Преобразование ответов в соответствующие модели.
- Предоставление типизированных операций для тестов.

Для работы с различными моделями используются три generic-параметра:

- REQUEST — модель запроса на создание сущности.
- ITEM — модель ответа для отдельной сущности.
- ITEMS — модель ответа со списком сущностей.

Конструктор класса:
```
    protected AdminCrudService(
            AdminCrudClient client,
            Class<REQUEST> requestClass,
            Class<ITEM> itemClass,
            Class<ITEMS> itemsClass
    ) {
        this.client = client;
        this.requestClass = requestClass;
        this.itemClass = itemClass;
        this.itemsClass = itemsClass;
    }
```

Пример метода класса:
```
    public ITEM create(AdminSession session, REQUEST request) {
        return extract(
                client.create(session, request),
                SC_CREATED,
                itemClass
        );
    }
```

#### Game Service

Специализированный сервис игры наследуется от AdminCrudService и определяет конкретный клиент и типы используемых моделей.

Сервис игры не содержит собственной реализации CRUD-методов.

Пример Service-класса игры:
```
public class AnagramAdminService extends AdminCrudService<AnagramAdminTask, AnagramAdminTask, AnagramAdminTasks> {

    public AnagramAdminService() {
        super(
                new AnagramAdminClient(),
                AnagramAdminTask.class,
                AnagramAdminTask.class,
                AnagramAdminTasks.class
        );
    }
}
```
#### Пример использования в тесте

Благодаря общей реализации CRUD-операций тесты используют специализированные сервисы, не завися от внутренней реализации HTTP-взаимодействия.

```
    @Test
    public void shouldCreateTask() {
        AnagramAdminTask request = AnagramTaskFactory.task();
        AnagramAdminTask response = anagramAdminService.create(
                adminSession,
                request
        );
        savedId.add(response.id());

        AnagramAdminAssertions.assertCreateTask(response, request);
    }
```

#### Добавление новой игры
При подключении новой игры клиент и сервис добавляются в виде двух небольших классов:

* Client:
  - Указать endpoint'ы CRUD-операций
  - Определить имя параметра идентификатора id
  
* Service:
  - Указать специализированный клиент
  - Определить модель запроса на создание задания
  - Определить модель ответа для отдельного задания
  - Определить модель ответа со списком заданий

Реализовывать CRUD-методы повторно не требуется: они наследуются из общих клиентского и сервисного слоёв.

(Остальные инфраструктурные слои - модели, генерация тестовых данных и проверки - добавляются индивидуально для каждой игры, согласно её специфике)

## Преимущества
* Уменьшение дублирования кода - общая реализация CRUD-операций используется всеми играми.
* Ускорение разработки - подключение новой игры требует минимального количества кода.
* Упрощение сопровождения - изменения общей CRUD-логики выполняются централизованно.
* Исключение случайных ошибок при копировании или написании кода для каждой игры отдельно.
* Единообразие реализации - все игровые клиенты и сервисы используют общий механизм взаимодействия с административным API.
* Масштабируемость - добавление новых игр не приводит к пропорциональному увеличению объёма инфраструктурного кода.

## Результаты внедрения

#### Уменьшение количества кода 

Сравнение количества строк кода до и после применения механизма на примере типовой игры.

Подсчёт выполнен по непустым строкам исходного кода без комментариев. Строки, содержащие только открывающие или закрывающие фигурные скобки, не учитывались. Отдельные строки с параметрами методов учитывались. Для обеих версий классов применялись одинаковые правила подсчёта.

| Класс|До| После |Сокращение|
|-|-|-|-|
| Admin Client|42     |12|71,4%|
| Admin Service|62     |7|88,7%|


#### Масштабирование

Решение изначально было разработано для унификации CRUD-тестирования административного API 7 игр.

На текущий момент механизм используется при тестировании 13 игр и упражнений.

Количество использований будет увеличиваться по мере расширения проекта.