# Microkernel Architecture (Архитектура с микроядром)

## Что такое Microkernel Architecture?

Microkernel Architecture (архитектура с микроядром) — это архитектурный паттерн, в котором минимальное ядро системы содержит только базовую функциональность, а дополнительные возможности реализуются через плагины (plug-ins). Эта архитектура также известна как Plug-in Architecture.

```
┌─────────────────────────────────────────────────────────────────┐
│                   Microkernel Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│   │ Plugin  │  │ Plugin  │  │ Plugin  │  │ Plugin  │           │
│   │    A    │  │    B    │  │    C    │  │    D    │           │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘           │
│        │            │            │            │                 │
│        └────────────┼────────────┼────────────┘                 │
│                     │            │                              │
│                     ▼            ▼                              │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                        Core System                       │  │
│   │                       (Микроядро)                        │  │
│   │                                                          │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │  │
│   │   │   Plugin    │  │   Event     │  │   Service   │     │  │
│   │   │  Registry   │  │   System    │  │   Locator   │     │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘     │  │
│   │                                                          │  │
│   │   ┌─────────────────────────────────────────────────┐   │  │
│   │   │           Minimal Core Functionality             │   │  │
│   │   │     (Базовая функциональность системы)          │   │  │
│   │   └─────────────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Компоненты архитектуры

### 1. Core System (Ядро системы)

Ядро содержит минимальную функциональность и механизмы расширения.

```python
from abc import ABC, abstractmethod
from typing import Dict, List, Any, Optional, Type
from dataclasses import dataclass
import importlib
import os


@dataclass
class PluginInfo:
    """Информация о плагине"""
    name: str
    version: str
    description: str
    author: str
    dependencies: List[str]


class Plugin(ABC):
    """Базовый интерфейс для плагинов"""

    @property
    @abstractmethod
    def info(self) -> PluginInfo:
        """Информация о плагине"""
        pass

    @abstractmethod
    def initialize(self, core: 'CoreSystem') -> None:
        """Инициализация плагина"""
        pass

    @abstractmethod
    def shutdown(self) -> None:
        """Завершение работы плагина"""
        pass

    def on_event(self, event_name: str, data: Any) -> None:
        """Обработка события (опционально)"""
        pass


class PluginRegistry:
    """Реестр плагинов"""

    def __init__(self):
        self._plugins: Dict[str, Plugin] = {}
        self._plugin_classes: Dict[str, Type[Plugin]] = {}

    def register_class(self, name: str, plugin_class: Type[Plugin]) -> None:
        """Зарегистрировать класс плагина"""
        self._plugin_classes[name] = plugin_class

    def create_instance(self, name: str) -> Optional[Plugin]:
        """Создать экземпляр плагина"""
        if name not in self._plugin_classes:
            return None

        plugin = self._plugin_classes[name]()
        self._plugins[name] = plugin
        return plugin

    def get(self, name: str) -> Optional[Plugin]:
        """Получить плагин по имени"""
        return self._plugins.get(name)

    def get_all(self) -> List[Plugin]:
        """Получить все загруженные плагины"""
        return list(self._plugins.values())

    def unregister(self, name: str) -> bool:
        """Удалить плагин"""
        if name in self._plugins:
            del self._plugins[name]
            return True
        return False


class EventSystem:
    """Система событий для коммуникации между ядром и плагинами"""

    def __init__(self):
        self._handlers: Dict[str, List[callable]] = {}

    def subscribe(self, event_name: str, handler: callable) -> None:
        """Подписаться на событие"""
        if event_name not in self._handlers:
            self._handlers[event_name] = []
        self._handlers[event_name].append(handler)

    def unsubscribe(self, event_name: str, handler: callable) -> None:
        """Отписаться от события"""
        if event_name in self._handlers:
            self._handlers[event_name].remove(handler)

    def emit(self, event_name: str, data: Any = None) -> None:
        """Отправить событие"""
        if event_name in self._handlers:
            for handler in self._handlers[event_name]:
                try:
                    handler(data)
                except Exception as e:
                    print(f"Error in event handler: {e}")


class ServiceLocator:
    """Локатор сервисов для зависимостей между плагинами"""

    def __init__(self):
        self._services: Dict[str, Any] = {}

    def register(self, name: str, service: Any) -> None:
        """Зарегистрировать сервис"""
        self._services[name] = service

    def get(self, name: str) -> Optional[Any]:
        """Получить сервис"""
        return self._services.get(name)

    def has(self, name: str) -> bool:
        """Проверить наличие сервиса"""
        return name in self._services


class CoreSystem:
    """
    Ядро системы — минимальная функциональность + механизмы расширения.
    """

    def __init__(self):
        self.registry = PluginRegistry()
        self.events = EventSystem()
        self.services = ServiceLocator()
        self._initialized = False

        # Регистрируем базовые сервисы
        self.services.register("events", self.events)
        self.services.register("registry", self.registry)

    def load_plugin(self, plugin_class: Type[Plugin]) -> bool:
        """Загрузить и инициализировать плагин"""
        try:
            # Создаём экземпляр
            plugin = plugin_class()
            info = plugin.info

            # Проверяем зависимости
            for dep in info.dependencies:
                if not self.registry.get(dep):
                    print(f"Missing dependency: {dep}")
                    return False

            # Регистрируем
            self.registry.register_class(info.name, plugin_class)
            instance = self.registry.create_instance(info.name)

            # Инициализируем
            instance.initialize(self)

            # Уведомляем о загрузке
            self.events.emit("plugin.loaded", info)

            print(f"Loaded plugin: {info.name} v{info.version}")
            return True

        except Exception as e:
            print(f"Failed to load plugin: {e}")
            return False

    def unload_plugin(self, name: str) -> bool:
        """Выгрузить плагин"""
        plugin = self.registry.get(name)
        if not plugin:
            return False

        try:
            plugin.shutdown()
            self.registry.unregister(name)
            self.events.emit("plugin.unloaded", {"name": name})
            return True
        except Exception as e:
            print(f"Error unloading plugin: {e}")
            return False

    def load_plugins_from_directory(self, directory: str) -> int:
        """Загрузить все плагины из директории"""
        loaded = 0

        for filename in os.listdir(directory):
            if filename.endswith("_plugin.py"):
                module_name = filename[:-3]
                try:
                    spec = importlib.util.spec_from_file_location(
                        module_name,
                        os.path.join(directory, filename)
                    )
                    module = importlib.util.module_from_spec(spec)
                    spec.loader.exec_module(module)

                    # Ищем класс плагина
                    for attr_name in dir(module):
                        attr = getattr(module, attr_name)
                        if (isinstance(attr, type) and
                            issubclass(attr, Plugin) and
                            attr is not Plugin):
                            if self.load_plugin(attr):
                                loaded += 1

                except Exception as e:
                    print(f"Error loading {filename}: {e}")

        return loaded

    def start(self) -> None:
        """Запуск ядра системы"""
        self._initialized = True
        self.events.emit("core.started")

    def shutdown(self) -> None:
        """Остановка системы"""
        self.events.emit("core.stopping")

        # Выгружаем все плагины
        for plugin in self.registry.get_all():
            plugin.shutdown()

        self._initialized = False
        self.events.emit("core.stopped")
```

### 2. Plugin (Плагин)

Плагины расширяют функциональность системы.

```python
# Пример: Текстовый редактор с плагинами

# Базовый плагин для форматирования текста
class TextFormatterPlugin(Plugin):
    """Плагин форматирования текста"""

    @property
    def info(self) -> PluginInfo:
        return PluginInfo(
            name="text_formatter",
            version="1.0.0",
            description="Basic text formatting",
            author="Example",
            dependencies=[]
        )

    def initialize(self, core: CoreSystem) -> None:
        self._core = core

        # Регистрируем сервис
        core.services.register("formatter", self)

        # Подписываемся на события
        core.events.subscribe("document.save", self.on_document_save)

    def shutdown(self) -> None:
        self._core.events.unsubscribe("document.save", self.on_document_save)

    def format_bold(self, text: str) -> str:
        """Жирный текст"""
        return f"**{text}**"

    def format_italic(self, text: str) -> str:
        """Курсив"""
        return f"*{text}*"

    def format_code(self, text: str) -> str:
        """Код"""
        return f"`{text}`"

    def on_document_save(self, data):
        """Обработка сохранения документа"""
        print(f"Document saved: {data.get('filename')}")


# Плагин проверки орфографии
class SpellCheckerPlugin(Plugin):
    """Плагин проверки орфографии"""

    @property
    def info(self) -> PluginInfo:
        return PluginInfo(
            name="spell_checker",
            version="1.0.0",
            description="Spell checking",
            author="Example",
            dependencies=[]
        )

    def initialize(self, core: CoreSystem) -> None:
        self._core = core
        self._dictionary = self._load_dictionary()

        core.services.register("spell_checker", self)
        core.events.subscribe("text.changed", self.on_text_changed)

    def shutdown(self) -> None:
        self._core.events.unsubscribe("text.changed", self.on_text_changed)

    def check(self, text: str) -> List[dict]:
        """Проверить текст на ошибки"""
        words = text.split()
        errors = []

        for i, word in enumerate(words):
            clean_word = word.strip(".,!?;:").lower()
            if clean_word and clean_word not in self._dictionary:
                errors.append({
                    "word": word,
                    "position": i,
                    "suggestions": self._get_suggestions(clean_word)
                })

        return errors

    def on_text_changed(self, data):
        """Проверка при изменении текста"""
        errors = self.check(data.get("text", ""))
        if errors:
            self._core.events.emit("spell_checker.errors", errors)

    def _load_dictionary(self) -> set:
        # Загрузка словаря
        return {"hello", "world", "example", "text", "editor"}

    def _get_suggestions(self, word: str) -> List[str]:
        # Простой алгоритм предложений
        return []


# Плагин автосохранения
class AutoSavePlugin(Plugin):
    """Плагин автосохранения"""

    @property
    def info(self) -> PluginInfo:
        return PluginInfo(
            name="auto_save",
            version="1.0.0",
            description="Automatic document saving",
            author="Example",
            dependencies=[]
        )

    def initialize(self, core: CoreSystem) -> None:
        self._core = core
        self._interval = 60  # секунд
        self._timer = None
        self._dirty = False

        core.events.subscribe("text.changed", self._on_change)
        core.events.subscribe("document.saved", self._on_saved)

        self._start_timer()

    def shutdown(self) -> None:
        if self._timer:
            self._timer.cancel()
        self._core.events.unsubscribe("text.changed", self._on_change)
        self._core.events.unsubscribe("document.saved", self._on_saved)

    def _on_change(self, data):
        self._dirty = True

    def _on_saved(self, data):
        self._dirty = False

    def _start_timer(self):
        import threading

        def auto_save():
            if self._dirty:
                self._core.events.emit("document.auto_save")
            self._timer = threading.Timer(self._interval, auto_save)
            self._timer.start()

        self._timer = threading.Timer(self._interval, auto_save)
        self._timer.start()


# Плагин экспорта в разные форматы
class ExportPlugin(Plugin):
    """Плагин экспорта документов"""

    @property
    def info(self) -> PluginInfo:
        return PluginInfo(
            name="export",
            version="1.0.0",
            description="Export to various formats",
            author="Example",
            dependencies=["text_formatter"]  # Зависит от форматтера
        )

    def initialize(self, core: CoreSystem) -> None:
        self._core = core
        self._formatter = core.services.get("formatter")

        core.services.register("export", self)

    def shutdown(self) -> None:
        pass

    def export_html(self, content: str) -> str:
        """Экспорт в HTML"""
        # Преобразуем markdown-подобный формат в HTML
        html = content
        html = html.replace("**", "<strong>", 1).replace("**", "</strong>", 1)
        html = html.replace("*", "<em>", 1).replace("*", "</em>", 1)
        html = html.replace("`", "<code>", 1).replace("`", "</code>", 1)

        return f"<!DOCTYPE html><html><body>{html}</body></html>"

    def export_pdf(self, content: str) -> bytes:
        """Экспорт в PDF"""
        # Заглушка для PDF экспорта
        return b"PDF content"

    def export_markdown(self, content: str) -> str:
        """Экспорт в Markdown"""
        return content
```

## Пример: Система обработки платежей

```python
# Система обработки платежей с плагинами для разных провайдеров

class PaymentProcessor:
    """Базовый интерфейс обработчика платежей"""

    @abstractmethod
    def process(self, amount: float, currency: str,
                payment_details: dict) -> dict:
        pass

    @abstractmethod
    def refund(self, transaction_id: str, amount: float) -> dict:
        pass

    @abstractmethod
    def get_status(self, transaction_id: str) -> dict:
        pass


class PaymentCore(CoreSystem):
    """Ядро платёжной системы"""

    def __init__(self):
        super().__init__()
        self._processors: Dict[str, PaymentProcessor] = {}

    def register_processor(self, name: str, processor: PaymentProcessor) -> None:
        """Зарегистрировать обработчик платежей"""
        self._processors[name] = processor
        self.services.register(f"payment.{name}", processor)

    def get_processor(self, name: str) -> Optional[PaymentProcessor]:
        """Получить обработчик"""
        return self._processors.get(name)

    def process_payment(self, processor_name: str, amount: float,
                        currency: str, details: dict) -> dict:
        """Обработать платёж"""
        processor = self.get_processor(processor_name)
        if not processor:
            raise ValueError(f"Unknown processor: {processor_name}")

        self.events.emit("payment.processing", {
            "processor": processor_name,
            "amount": amount,
            "currency": currency
        })

        result = processor.process(amount, currency, details)

        self.events.emit("payment.completed", result)
        return result


# Плагин для Stripe
class StripePaymentPlugin(Plugin):
    """Плагин для оплаты через Stripe"""

    @property
    def info(self) -> PluginInfo:
        return PluginInfo(
            name="stripe_payment",
            version="1.0.0",
            description="Stripe payment integration",
            author="Example",
            dependencies=[]
        )

    def initialize(self, core: PaymentCore) -> None:
        self._core = core
        self._api_key = os.environ.get("STRIPE_API_KEY")

        processor = StripeProcessor(self._api_key)
        core.register_processor("stripe", processor)

    def shutdown(self) -> None:
        pass


class StripeProcessor(PaymentProcessor):
    """Обработчик платежей Stripe"""

    def __init__(self, api_key: str):
        self._api_key = api_key

    def process(self, amount: float, currency: str,
                payment_details: dict) -> dict:
        # Интеграция с Stripe API
        return {
            "success": True,
            "transaction_id": f"stripe_{uuid.uuid4().hex}",
            "amount": amount,
            "currency": currency,
            "processor": "stripe"
        }

    def refund(self, transaction_id: str, amount: float) -> dict:
        return {"success": True, "refund_id": f"refund_{uuid.uuid4().hex}"}

    def get_status(self, transaction_id: str) -> dict:
        return {"status": "completed"}


# Плагин для PayPal
class PayPalPaymentPlugin(Plugin):
    """Плагин для оплаты через PayPal"""

    @property
    def info(self) -> PluginInfo:
        return PluginInfo(
            name="paypal_payment",
            version="1.0.0",
            description="PayPal payment integration",
            author="Example",
            dependencies=[]
        )

    def initialize(self, core: PaymentCore) -> None:
        self._core = core
        self._client_id = os.environ.get("PAYPAL_CLIENT_ID")
        self._secret = os.environ.get("PAYPAL_SECRET")

        processor = PayPalProcessor(self._client_id, self._secret)
        core.register_processor("paypal", processor)

    def shutdown(self) -> None:
        pass


class PayPalProcessor(PaymentProcessor):
    """Обработчик платежей PayPal"""

    def __init__(self, client_id: str, secret: str):
        self._client_id = client_id
        self._secret = secret

    def process(self, amount: float, currency: str,
                payment_details: dict) -> dict:
        # Интеграция с PayPal API
        return {
            "success": True,
            "transaction_id": f"paypal_{uuid.uuid4().hex}",
            "amount": amount,
            "currency": currency,
            "processor": "paypal"
        }

    def refund(self, transaction_id: str, amount: float) -> dict:
        return {"success": True}

    def get_status(self, transaction_id: str) -> dict:
        return {"status": "completed"}


# Использование
payment_system = PaymentCore()
payment_system.load_plugin(StripePaymentPlugin)
payment_system.load_plugin(PayPalPaymentPlugin)
payment_system.start()

# Обработка платежа
result = payment_system.process_payment(
    processor_name="stripe",
    amount=99.99,
    currency="USD",
    details={"card_token": "tok_xxx"}
)
```

## Архитектурная диаграмма

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Application Layer                                   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                          UI / API                                │   │
│   └──────────────────────────────┬──────────────────────────────────┘   │
│                                  │                                       │
│   ┌──────────────────────────────┼──────────────────────────────────┐   │
│   │                         Plugins                                  │   │
│   │                                                                  │   │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│   │  │Export   │  │Spell    │  │Theme    │  │Analytics│            │   │
│   │  │Plugin   │  │Checker  │  │Plugin   │  │Plugin   │            │   │
│   │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘            │   │
│   │       │            │            │            │                   │   │
│   └───────┼────────────┼────────────┼────────────┼───────────────────┘   │
│           │            │            │            │                       │
│           └────────────┴────────────┴────────────┘                       │
│                                  │                                       │
│   ┌──────────────────────────────┼──────────────────────────────────┐   │
│   │                         Core System                              │   │
│   │                                                                  │   │
│   │   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐      │   │
│   │   │    Plugin     │  │    Event      │  │   Service     │      │   │
│   │   │   Registry    │  │    System     │  │   Locator     │      │   │
│   │   └───────────────┘  └───────────────┘  └───────────────┘      │   │
│   │                                                                  │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │                  Core Functionality                      │   │   │
│   │   │  • Document Management                                   │   │   │
│   │   │  • Basic CRUD Operations                                 │   │   │
│   │   │  • Configuration                                         │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                  │   │
│   └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Плюсы и минусы

### Плюсы

1. **Расширяемость** — легко добавлять новую функциональность
2. **Изоляция** — плагины изолированы друг от друга
3. **Гибкость** — разные конфигурации для разных пользователей
4. **Maintainability** — обновление плагинов без изменения ядра
5. **Тестируемость** — плагины тестируются независимо

### Минусы

1. **Сложность** — нужна продуманная система плагинов
2. **Производительность** — overhead на динамическую загрузку
3. **Совместимость** — версионирование API плагинов
4. **Безопасность** — плагины могут быть небезопасными
5. **Зависимости** — сложные зависимости между плагинами

## Когда использовать

### Используйте Microkernel когда:

- Приложение с расширяемой функциональностью (IDE, браузеры)
- Разные пользователи нуждаются в разных функциях
- Третьи стороны должны расширять систему
- Функциональность развивается независимо
- Нужна модульность и изоляция

### Не используйте когда:

- Простое приложение без расширений
- Функциональность чётко определена и стабильна
- Высокие требования к производительности
- Нет ресурсов на поддержку API плагинов

## Примеры использования

- **IDE**: VS Code, Eclipse, IntelliJ IDEA
- **Браузеры**: Chrome, Firefox (расширения)
- **CMS**: WordPress, Drupal (плагины)
- **Редакторы**: Vim, Emacs (плагины)
- **Игры**: модификации через плагины

## Заключение

Microkernel Architecture идеальна для приложений, которые должны быть расширяемыми. Ядро остаётся стабильным и минимальным, а вся дополнительная функциональность добавляется через плагины. Это позволяет создавать гибкие системы, адаптируемые под нужды разных пользователей.
