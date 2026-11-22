# Пример API-теста

## Авторизация пользователя по email - позитивный сценарий

Пример демонстрирует реализацию API-теста и взаимодействие основных слоёв автоматизации на примере авторизации пользователя.

##  Архитектура API-тестов

API-автоматизация построена на слоистой архитектуре, в которой каждый слой отвечает за определённую часть взаимодействия с тестируемым приложением.

- Test — описывает тестовые сценарии и последовательность действий.
- Assertions — содержит проверки результатов выполнения тестов.
- Services — реализует бизнес-операции, управляет вызовами клиентов и обработкой ответов.
- Clients — инкапсулирует HTTP-взаимодействие с API.
- Models & DTO — описывает структуры данных, используемые при отправке запросов и обработке ответов.
- BaseApiClient и BaseApiService — базовые классы, содержащие общую логику и методы для клиентского и сервисного слоёв соответственно.

Такое разделение на слои позволяет разграничить ответственность компонентов, упростить поддержку и повторное использование кода.

## Test

Для изоляции тестов используется новый пользователь, который регистрируется и подтверждает email непосредственно перед выполнением теста. Вспомогательный метод возвращает DTO с учётными данными.
```
    private UserSignInDTO signUpAndConfirm() {
        InboxDto inbox = createInboxAndSaveId();  // создание почтового ящика

        UserSignUpRequest signUpRequest = SignUpRequestFactory.signUpRequest(inbox);
        userAuthService.signUp(signUpRequest);

        String confirmationCode = getConfirmationCode(inbox.getId().toString());
        userAuthService.confirmEmail(
                new ConfirmEmailRequest(confirmationCode)
        );

        emailsToDelete.add(inbox.getEmailAddress()); // добавление email в список для последующей очистки

        return new UserSignInDTO(
                signUpRequest.username(),
                inbox.getEmailAddress(),
                signUpRequest.password()
        );
    }
```

Тест:
```
    @Test
    public void shouldSignInViaEmail() {
        UserSignInDTO userData = signUpAndConfirm();

        session = userAuthService.signIn(
                new UserSignInRequest(
                        userData.email(),
                        userData.password()
                )
        );

        AuthAssertions.assertSignIn(session);
    }
```

## Assertions

Проверки успешной авторизации: наличие пользовательской сессии, access token и refresh cookie.

```
    public static void assertSignIn(UserSession response) {
        assertThat(response).isNotNull();
        assertThat(response.getAccessToken()).isNotBlank();
        assertThat(response.getRefreshCookie()).isNotNull();
    }
```

## Service

Сервисный слой отвечает за выполнение бизнес-операции: вызывает клиент, проверяет HTTP-статус ответа и преобразует ответ в объект пользовательской сессии.

```
    public UserSession signIn(UserSignInRequest request) {
        Response response = userAuthClient.signIn(request, UserAgent.DEFAULT_DEVICE);
        extract(
                response,
                SC_OK
        );
        return new UserSession(response);
    }
```

## Client

Клиент отвечает за отправку HTTP-запроса (определяя endpoint, тело запроса и параметры конфигурации) и получение ответа.

```
    public Response signIn(Object request, UserAgent userAgent) {
        return post(
                SIGN_IN_PATH,
                request,
                ApiConfig.publicJsonRequest(userAgent.getValue()));
    }
```

## Model & DTO

UserSignInRequest — модель тела запроса для авторизации по имени пользователя или email.

```
public record UserSignInRequest(
        String usernameOrEmail,
        String password
) {}
```

UserSignInDTO — DTO, объединяющий учётные данные зарегистрированного пользователя для последующего использования в тестах авторизации.

```
public record UserSignInDTO(
        String username,
        String email,
        String password
) {}
```