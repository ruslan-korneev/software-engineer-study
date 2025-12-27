# Функциональное тестирование

## Что такое функциональное тестирование?

**Функциональное тестирование** (Functional Testing) — это тип тестирования программного обеспечения, который проверяет, что система выполняет свои функции в соответствии с требованиями. Тесты фокусируются на том, **что** система делает, а не на том, **как** она это делает.

### Ключевые характеристики:

- **Чёрный ящик** — тестируется внешнее поведение без знания внутренней реализации
- **Бизнес-требования** — тесты основаны на функциональных спецификациях
- **End-to-End** — проверяется полный пользовательский сценарий
- **Реальное окружение** — максимально приближено к продакшену

---

## Место в пирамиде тестирования

```
           /\
          /  \
         / E2E\         <- Функциональные тесты
        /  UI  \           (мало, медленные, дорогие)
       /--------\
      /API Tests \      <- Интеграционные тесты
     /------------\
    /  Unit Tests  \    <- Модульные тесты
   /----------------\      (много, быстрые, дешёвые)
```

Функциональные тесты находятся на вершине пирамиды. Их должно быть меньше всего, но они критически важны для проверки реальных сценариев использования.

---

## Отличия от других видов тестирования

| Аспект | Unit | Integration | Functional |
|--------|------|-------------|------------|
| Фокус | Отдельный метод | Взаимодействие модулей | Бизнес-сценарий |
| Перспектива | Разработчик | Разработчик | Пользователь |
| Данные | Моки | Тестовые | Реалистичные |
| Окружение | Изолированное | Частичное | Полное |
| Скорость | Очень быстрые | Средние | Медленные |

---

## Виды функционального тестирования

### 1. Smoke Testing (Дымовое тестирование)

Быстрая проверка основной функциональности после деплоя.

```python
# test_smoke.py
import pytest
import requests

class TestSmoke:
    """Базовые проверки работоспособности системы"""

    BASE_URL = "https://api.example.com"

    def test_api_is_alive(self):
        """API отвечает на запросы"""
        response = requests.get(f"{self.BASE_URL}/health")
        assert response.status_code == 200

    def test_database_connection(self):
        """БД доступна"""
        response = requests.get(f"{self.BASE_URL}/health/db")
        assert response.json()["database"] == "connected"

    def test_authentication_works(self):
        """Авторизация работает"""
        response = requests.post(f"{self.BASE_URL}/auth/login", json={
            "email": "test@example.com",
            "password": "testpassword"
        })
        assert response.status_code in [200, 401]  # Работает, даже если неверные данные

    def test_main_page_loads(self):
        """Главная страница загружается"""
        response = requests.get(f"{self.BASE_URL}/")
        assert response.status_code == 200
        assert "Welcome" in response.text or response.json()
```

### 2. Sanity Testing (Проверка здравомыслия)

Проверка конкретной функциональности после изменений.

```python
class TestSanity:
    """Проверка после обновления модуля оплаты"""

    def test_payment_page_accessible(self):
        """Страница оплаты доступна"""
        pass

    def test_payment_form_validation(self):
        """Валидация формы работает"""
        pass

    def test_payment_processing(self):
        """Платёж обрабатывается"""
        pass
```

### 3. Regression Testing (Регрессионное тестирование)

Проверка, что новый код не сломал существующую функциональность.

```python
# test_regression.py
@pytest.mark.regression
class TestUserRegistration:
    """Регрессионные тесты регистрации пользователей"""

    def test_user_can_register_with_email(self):
        """Пользователь может зарегистрироваться через email"""
        pass

    def test_user_receives_confirmation_email(self):
        """Пользователь получает письмо подтверждения"""
        pass

    def test_user_can_login_after_registration(self):
        """Пользователь может войти после регистрации"""
        pass
```

---

## End-to-End тестирование API

### Полный сценарий с pytest

```python
# test_e2e_order.py
import pytest
import requests
from datetime import datetime

class TestOrderE2E:
    """
    End-to-End тест полного цикла заказа:
    Регистрация -> Вход -> Добавление в корзину -> Заказ -> Оплата
    """

    BASE_URL = "http://localhost:8000/api"

    @pytest.fixture(autouse=True)
    def setup(self):
        """Подготовка тестовых данных"""
        self.test_email = f"test_{datetime.now().timestamp()}@example.com"
        self.test_password = "SecurePassword123!"
        self.session = requests.Session()

    def test_complete_order_flow(self):
        # === STEP 1: Регистрация пользователя ===
        register_response = self.session.post(
            f"{self.BASE_URL}/auth/register",
            json={
                "email": self.test_email,
                "password": self.test_password,
                "name": "Тестовый Пользователь"
            }
        )
        assert register_response.status_code == 201, \
            f"Регистрация не удалась: {register_response.text}"

        user_id = register_response.json()["id"]

        # === STEP 2: Вход в систему ===
        login_response = self.session.post(
            f"{self.BASE_URL}/auth/login",
            json={
                "email": self.test_email,
                "password": self.test_password
            }
        )
        assert login_response.status_code == 200, \
            f"Вход не удался: {login_response.text}"

        token = login_response.json()["access_token"]
        self.session.headers.update({"Authorization": f"Bearer {token}"})

        # === STEP 3: Получение списка товаров ===
        products_response = self.session.get(f"{self.BASE_URL}/products")
        assert products_response.status_code == 200

        products = products_response.json()["items"]
        assert len(products) > 0, "Нет доступных товаров"

        selected_product = products[0]

        # === STEP 4: Добавление товара в корзину ===
        cart_response = self.session.post(
            f"{self.BASE_URL}/cart/items",
            json={
                "product_id": selected_product["id"],
                "quantity": 2
            }
        )
        assert cart_response.status_code == 201, \
            f"Не удалось добавить в корзину: {cart_response.text}"

        # === STEP 5: Проверка корзины ===
        cart_check = self.session.get(f"{self.BASE_URL}/cart")
        assert cart_check.status_code == 200

        cart_data = cart_check.json()
        assert len(cart_data["items"]) == 1
        assert cart_data["items"][0]["quantity"] == 2

        # === STEP 6: Оформление заказа ===
        order_response = self.session.post(
            f"{self.BASE_URL}/orders",
            json={
                "shipping_address": {
                    "city": "Москва",
                    "street": "Тверская",
                    "building": "1",
                    "apartment": "10"
                },
                "payment_method": "card"
            }
        )
        assert order_response.status_code == 201, \
            f"Не удалось создать заказ: {order_response.text}"

        order = order_response.json()
        order_id = order["id"]
        assert order["status"] == "pending"

        # === STEP 7: Имитация оплаты ===
        payment_response = self.session.post(
            f"{self.BASE_URL}/orders/{order_id}/pay",
            json={
                "card_number": "4111111111111111",
                "expiry": "12/25",
                "cvv": "123"
            }
        )
        assert payment_response.status_code == 200, \
            f"Оплата не удалась: {payment_response.text}"

        # === STEP 8: Проверка статуса заказа ===
        order_status = self.session.get(f"{self.BASE_URL}/orders/{order_id}")
        assert order_status.status_code == 200

        final_order = order_status.json()
        assert final_order["status"] == "paid"
        assert final_order["payment_status"] == "completed"

        # === STEP 9: Корзина должна быть пустой ===
        empty_cart = self.session.get(f"{self.BASE_URL}/cart")
        assert len(empty_cart.json()["items"]) == 0

        print(f"✓ Заказ #{order_id} успешно создан и оплачен")
```

---

## Behavior-Driven Development (BDD)

### Использование pytest-bdd

```bash
pip install pytest-bdd
```

```gherkin
# features/login.feature
Feature: Авторизация пользователя
    Как пользователь системы
    Я хочу иметь возможность войти в свой аккаунт
    Чтобы получить доступ к персональным данным

    Background:
        Given пользователь зарегистрирован в системе

    Scenario: Успешный вход с правильными данными
        Given пользователь находится на странице входа
        When пользователь вводит корректный email "user@example.com"
        And пользователь вводит корректный пароль "password123"
        And пользователь нажимает кнопку "Войти"
        Then пользователь видит свой профиль
        And в системе создаётся сессия

    Scenario: Неудачный вход с неверным паролем
        Given пользователь находится на странице входа
        When пользователь вводит корректный email "user@example.com"
        And пользователь вводит неверный пароль "wrongpassword"
        And пользователь нажимает кнопку "Войти"
        Then пользователь видит сообщение об ошибке "Неверный email или пароль"
        And пользователь остаётся на странице входа

    Scenario Outline: Валидация формы входа
        Given пользователь находится на странице входа
        When пользователь вводит email "<email>"
        And пользователь вводит пароль "<password>"
        And пользователь нажимает кнопку "Войти"
        Then пользователь видит ошибку валидации "<error>"

        Examples:
            | email           | password | error                    |
            |                 | pass123  | Email обязателен         |
            | invalid-email   | pass123  | Неверный формат email    |
            | user@test.com   |          | Пароль обязателен        |
```

```python
# tests/step_defs/test_login.py
import pytest
from pytest_bdd import given, when, then, scenario, parsers

@scenario('../features/login.feature', 'Успешный вход с правильными данными')
def test_successful_login():
    pass

@scenario('../features/login.feature', 'Неудачный вход с неверным паролем')
def test_failed_login():
    pass

# --- Given steps ---

@given('пользователь зарегистрирован в системе')
def registered_user(test_client):
    test_client.post('/api/auth/register', json={
        'email': 'user@example.com',
        'password': 'password123'
    })

@given('пользователь находится на странице входа')
def on_login_page(browser):
    browser.navigate_to('/login')

# --- When steps ---

@when(parsers.parse('пользователь вводит корректный email "{email}"'))
def enter_email(browser, email):
    browser.fill('email', email)

@when(parsers.parse('пользователь вводит корректный пароль "{password}"'))
def enter_password(browser, password):
    browser.fill('password', password)

@when(parsers.parse('пользователь вводит неверный пароль "{password}"'))
def enter_wrong_password(browser, password):
    browser.fill('password', password)

@when(parsers.parse('пользователь нажимает кнопку "{button}"'))
def click_button(browser, button):
    browser.click_button(button)

# --- Then steps ---

@then('пользователь видит свой профиль')
def see_profile(browser):
    assert browser.current_url.endswith('/profile')
    assert browser.find_element('#user-name').is_displayed()

@then('в системе создаётся сессия')
def session_created(browser):
    assert browser.get_cookie('session_id') is not None

@then(parsers.parse('пользователь видит сообщение об ошибке "{message}"'))
def see_error_message(browser, message):
    error_element = browser.find_element('.error-message')
    assert message in error_element.text

@then('пользователь остаётся на странице входа')
def still_on_login_page(browser):
    assert browser.current_url.endswith('/login')
```

---

## Тестирование UI с Selenium

### Настройка

```bash
pip install selenium webdriver-manager
```

```python
# conftest.py
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager

@pytest.fixture(scope="function")
def browser():
    """Создаёт браузер для каждого теста"""
    options = webdriver.ChromeOptions()
    options.add_argument("--headless")  # Без GUI
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")

    service = Service(ChromeDriverManager().install())
    driver = webdriver.Chrome(service=service, options=options)
    driver.implicitly_wait(10)

    yield driver

    driver.quit()

@pytest.fixture
def logged_in_browser(browser, base_url, test_user):
    """Браузер с авторизованным пользователем"""
    browser.get(f"{base_url}/login")
    browser.find_element("id", "email").send_keys(test_user["email"])
    browser.find_element("id", "password").send_keys(test_user["password"])
    browser.find_element("id", "login-button").click()
    return browser
```

### Page Object Pattern

```python
# pages/base_page.py
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.by import By

class BasePage:
    def __init__(self, driver, base_url):
        self.driver = driver
        self.base_url = base_url
        self.wait = WebDriverWait(driver, 10)

    def navigate_to(self, path):
        self.driver.get(f"{self.base_url}{path}")

    def find_element(self, locator):
        return self.wait.until(EC.presence_of_element_located(locator))

    def click(self, locator):
        element = self.wait.until(EC.element_to_be_clickable(locator))
        element.click()

    def fill(self, locator, text):
        element = self.find_element(locator)
        element.clear()
        element.send_keys(text)

    def get_text(self, locator):
        return self.find_element(locator).text


# pages/login_page.py
class LoginPage(BasePage):
    # Локаторы
    EMAIL_INPUT = (By.ID, "email")
    PASSWORD_INPUT = (By.ID, "password")
    LOGIN_BUTTON = (By.ID, "login-button")
    ERROR_MESSAGE = (By.CLASS_NAME, "error-message")

    def __init__(self, driver, base_url):
        super().__init__(driver, base_url)
        self.navigate_to("/login")

    def login(self, email, password):
        self.fill(self.EMAIL_INPUT, email)
        self.fill(self.PASSWORD_INPUT, password)
        self.click(self.LOGIN_BUTTON)

    def get_error_message(self):
        return self.get_text(self.ERROR_MESSAGE)

    def is_login_successful(self):
        return "/dashboard" in self.driver.current_url


# pages/dashboard_page.py
class DashboardPage(BasePage):
    USER_NAME = (By.ID, "user-name")
    LOGOUT_BUTTON = (By.ID, "logout")

    def get_user_name(self):
        return self.get_text(self.USER_NAME)

    def logout(self):
        self.click(self.LOGOUT_BUTTON)


# test_login_ui.py
class TestLoginUI:

    def test_successful_login(self, browser, base_url, test_user):
        # Arrange
        login_page = LoginPage(browser, base_url)

        # Act
        login_page.login(test_user["email"], test_user["password"])

        # Assert
        assert login_page.is_login_successful()

        dashboard = DashboardPage(browser, base_url)
        assert dashboard.get_user_name() == test_user["name"]

    def test_login_with_invalid_credentials(self, browser, base_url):
        # Arrange
        login_page = LoginPage(browser, base_url)

        # Act
        login_page.login("wrong@email.com", "wrongpassword")

        # Assert
        assert not login_page.is_login_successful()
        assert "Неверный email или пароль" in login_page.get_error_message()
```

---

## Тестирование с Playwright

Современная альтернатива Selenium.

```bash
pip install playwright pytest-playwright
playwright install
```

```python
# conftest.py
import pytest
from playwright.sync_api import Page

@pytest.fixture
def authenticated_page(page: Page, base_url: str, test_user: dict) -> Page:
    """Страница с авторизованным пользователем"""
    page.goto(f"{base_url}/login")
    page.fill("#email", test_user["email"])
    page.fill("#password", test_user["password"])
    page.click("#login-button")
    page.wait_for_url("**/dashboard")
    return page

# test_dashboard.py
def test_dashboard_shows_user_info(authenticated_page: Page):
    # Assert
    expect(authenticated_page.locator("#user-name")).to_be_visible()
    expect(authenticated_page.locator("#user-email")).to_contain_text("@")

def test_user_can_update_profile(authenticated_page: Page):
    # Arrange
    authenticated_page.click("#edit-profile")

    # Act
    authenticated_page.fill("#name", "Новое Имя")
    authenticated_page.click("#save-profile")

    # Assert
    expect(authenticated_page.locator(".success-message")).to_be_visible()
    expect(authenticated_page.locator("#user-name")).to_have_text("Новое Имя")

def test_navigation_menu(authenticated_page: Page):
    # Act & Assert
    authenticated_page.click("text=Заказы")
    expect(authenticated_page).to_have_url(re.compile(r".*/orders"))

    authenticated_page.click("text=Настройки")
    expect(authenticated_page).to_have_url(re.compile(r".*/settings"))
```

---

## Тестовые данные и фикстуры

### Фабрики тестовых данных

```python
# factories.py
from faker import Faker
from dataclasses import dataclass
from typing import Optional

fake = Faker('ru_RU')

@dataclass
class TestUser:
    email: str
    password: str
    name: str

    @classmethod
    def create(cls, **overrides) -> 'TestUser':
        defaults = {
            "email": fake.email(),
            "password": "TestPassword123!",
            "name": fake.name()
        }
        return cls(**{**defaults, **overrides})

@dataclass
class TestProduct:
    name: str
    price: float
    description: str

    @classmethod
    def create(cls, **overrides) -> 'TestProduct':
        defaults = {
            "name": fake.word().capitalize(),
            "price": round(fake.pyfloat(min_value=100, max_value=10000), 2),
            "description": fake.text(max_nb_chars=200)
        }
        return cls(**{**defaults, **overrides})

@dataclass
class TestOrder:
    user: TestUser
    products: list[TestProduct]
    address: dict

    @classmethod
    def create(cls, user: Optional[TestUser] = None) -> 'TestOrder':
        return cls(
            user=user or TestUser.create(),
            products=[TestProduct.create() for _ in range(3)],
            address={
                "city": fake.city(),
                "street": fake.street_name(),
                "building": fake.building_number()
            }
        )
```

### Загрузка тестовых данных

```python
# conftest.py
import json
import pytest

@pytest.fixture(scope="session")
def test_data():
    """Загружает тестовые данные из JSON"""
    with open("tests/fixtures/test_data.json") as f:
        return json.load(f)

@pytest.fixture
def test_user(test_data):
    return test_data["users"]["standard_user"]

@pytest.fixture
def admin_user(test_data):
    return test_data["users"]["admin_user"]
```

```json
// tests/fixtures/test_data.json
{
    "users": {
        "standard_user": {
            "email": "user@test.com",
            "password": "password123",
            "name": "Тестовый Пользователь"
        },
        "admin_user": {
            "email": "admin@test.com",
            "password": "adminpass",
            "name": "Администратор"
        }
    },
    "products": [
        {"id": 1, "name": "Товар 1", "price": 1000},
        {"id": 2, "name": "Товар 2", "price": 2000}
    ]
}
```

---

## Лучшие практики

### 1. Независимость тестов

```python
# Плохо - тесты зависят друг от друга
def test_create_order():
    global order_id
    order_id = create_order()

def test_pay_order():
    pay_order(order_id)  # Зависит от предыдущего теста!

# Хорошо - каждый тест автономен
def test_complete_order_flow():
    order_id = create_order()
    pay_order(order_id)
    verify_order_paid(order_id)
```

### 2. Читаемые утверждения

```python
# Плохо
assert response.status_code == 200

# Хорошо - понятное сообщение об ошибке
assert response.status_code == 200, \
    f"Ожидался статус 200, получен {response.status_code}. Ответ: {response.text}"

# Ещё лучше - использовать библиотеку
from hamcrest import assert_that, equal_to, has_key

assert_that(response.status_code, equal_to(200))
assert_that(response.json(), has_key("user_id"))
```

### 3. Стабильные селекторы

```python
# Плохо - хрупкие селекторы
browser.find_element(By.XPATH, "//div[3]/span[2]/button")
browser.find_element(By.CLASS_NAME, "btn-primary-large-v2")

# Хорошо - стабильные селекторы
browser.find_element(By.ID, "submit-order-button")
browser.find_element(By.CSS_SELECTOR, "[data-testid='submit-order']")
```

### 4. Ожидания вместо sleep

```python
# Плохо
import time
time.sleep(5)
element = browser.find_element(By.ID, "result")

# Хорошо
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(browser, 10)
element = wait.until(EC.presence_of_element_located((By.ID, "result")))
```

---

## Типичные ошибки

### 1. Слишком много E2E тестов

```
Неправильно:                      Правильно:
        /\                               /\
       /  \                             /  \
      / E2E\  <- Много                 / E2E\  <- Мало
     /------\                         /------\
    /  Int   \  <- Мало              /  Int   \  <- Средне
   /----------\                     /----------\
  /   Unit     \ <- Мало           /   Unit     \ <- Много
 /--------------\                 /--------------\
```

### 2. Игнорирование очистки

```python
# Плохо - данные остаются между тестами
def test_create_user():
    create_user(email="test@test.com")

def test_another():
    # test@test.com уже существует - может сломать тест

# Хорошо - очистка после каждого теста
@pytest.fixture(autouse=True)
def cleanup():
    yield
    clean_test_database()
```

### 3. Нестабильные тесты (Flaky tests)

```python
# Плохо - тест иногда падает
def test_async_operation():
    start_async_operation()
    result = get_result()  # Может не успеть выполниться

# Хорошо - явное ожидание
def test_async_operation():
    start_async_operation()
    result = wait_for_result(timeout=10)
    assert result is not None
```

---

## Запуск функциональных тестов

```bash
# Все функциональные тесты
pytest tests/functional/ -v

# Только smoke-тесты
pytest -m smoke

# С отчётом Allure
pytest --alluredir=allure-results
allure serve allure-results

# В разных браузерах (Playwright)
pytest --browser chromium
pytest --browser firefox
pytest --browser webkit

# Параллельный запуск
pytest tests/functional/ -n 4

# С записью видео (Playwright)
pytest --video=on
```

---

## Отчёты о тестировании

### Allure Reports

```python
import allure

@allure.feature("Авторизация")
@allure.story("Вход пользователя")
@allure.severity(allure.severity_level.CRITICAL)
def test_user_login():
    with allure.step("Открыть страницу входа"):
        login_page.open()

    with allure.step("Ввести учётные данные"):
        login_page.enter_credentials("user@test.com", "password")

    with allure.step("Нажать кнопку входа"):
        login_page.submit()

    with allure.step("Проверить успешный вход"):
        assert dashboard_page.is_displayed()

    allure.attach(
        browser.get_screenshot_as_png(),
        name="screenshot",
        attachment_type=allure.attachment_type.PNG
    )
```

---

## Полезные ресурсы

- [pytest-bdd документация](https://pytest-bdd.readthedocs.io/)
- [Selenium Python](https://selenium-python.readthedocs.io/)
- [Playwright Python](https://playwright.dev/python/)
- [Allure Framework](https://docs.qameta.io/allure/)
- [Page Object Pattern](https://martinfowler.com/bliki/PageObject.html)
