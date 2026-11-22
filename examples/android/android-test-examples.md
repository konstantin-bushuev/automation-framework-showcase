# Пример Android-теста

## Авторизация пользователя по email - позитивный сценарий    

Пример демонстрирует реализацию Android UI-теста на примере авторизации пользователя.

## Выбор окружения

Параметры запуска Android-тестов задаются через application.properties и local.properties: application.properties содержит общие настройки и настройки эмулятора, а local.properties — параметры конкретного реального устройства и окружения.

В конфигурации можно определить:
- тип устройства: эмулятор или реальное устройство
- параметры подключения к устройству
- платформу и версию Android
- параметры запуска Appium
- настройки приложения (путь к APK или идентификатор установленного приложения)

Локальные параметры устройства хранятся в local.properties и не добавляются в репозиторий.

## Особенности реализации

* Page Object Model — каждый экран представлен отдельным классом, содержащим локаторы и методы взаимодействия с элементами.
* BaseUiObject — общие методы взаимодействия с UI-элементами, ожидания и работа с драйвером вынесены в базовый класс.
* Fluent Interface — методы экранов возвращают текущий или следующий экран, позволяя строить последовательные цепочки вызовов.
* Тестовые данные — учетные данные пользователя вынесены в отдельный источник и не дублируются в тестовых сценариях.

## Test

Для тестирования авторизации через UI используется заранее созданный пользователь.

```
    @Test
    public void loginViaEmailTest() {

        LoginScreen login =
                new StartScreen(driver)
                        .start()
                        .clickLogin();

        ProfileScreen profileScreen = login
                .clickEmail()
                .waitEmailMode()
                .enterEmail(Users.USER.email())
                .enterPassword(Users.USER.password())
                .submit();

        assertTrue(profileScreen.isLoaded());
    }
```

## LoginScreen

```
    
    private final By regButton =...; 
    private final By emailButton =...; 
    private final By emailField = ...;
    private final By passwordField = ...;
    private final By passwordRecoveryButton = ...;
    private final By submitButton = ...;

    public LoginScreen(AndroidDriver driver) {
        super(driver);
        waitUntilLoaded();
    }

    private void waitUntilLoaded() {
        waitVisible(regButton);
    }

    public boolean isLoaded() {
        return waitVisible(regButton).isDisplayed();
    }

    public LoginScreen clickEmail() {
        click(emailButton);
        return this;
    }

    public LoginScreen waitEmailMode() {
        try {
            wait.until(ExpectedConditions.and(
                    ExpectedConditions.elementToBeClickable(emailField),
                    ExpectedConditions.elementToBeClickable(passwordField),
                    ExpectedConditions.not(ExpectedConditions.elementToBeClickable(submitButton))
            ));
            return this;
        } catch (TimeoutException e) {
            throw new AssertionError("Email mode didn't appear", e);
        }
    }

    public LoginScreen enterEmail(String userEmail) {
        type(emailField, userEmail);
        driver.hideKeyboard();
        return this;
    }

    public LoginScreen enterPassword(String userPassword) {
        type(passwordField, userPassword);
        driver.hideKeyboard();
        return this;
    }

    public ProfileScreen submit() {
        click(submitButton);
        return new ProfileScreen(driver);
    }
```

