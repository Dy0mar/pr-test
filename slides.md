---
theme: seriph
background: https://cover.sli.dev
title: Performance Review - SOLID
info: |
  ## SOLID Principles Interview
  Assess engineering thinking through code discussion
class: text-center
transition: slide-left
---

<style>
pre { font-size: 10px !important; line-height: 1.2 !important; max-height: 90vh !important; overflow-y: auto !important; }
code { font-size: 10px !important; }
ul, ol { font-size: 12px !important; }
li { margin-bottom: 2px !important; }
p { margin-bottom: 4px !important; }
h1, h2, h3 { font-size: 24px !important; }
</style>

# Performance Review

Engineering Thinking Assessment

---

# SRP - The Code

```python
import httpx
from ninja import Router
from django.utils import timezone
from django.core.mail import send_mail
from django.shortcuts import get_object_or_404
from .models import Subscription, User
from .schemas import SubscriptionRenewSchema

router = Router()

@router.post("/subscriptions/renew")
def renew_subscription(request, payload: SubscriptionRenewSchema):
    user = get_object_or_404(User, id=payload.user_id)
    subscription = get_object_or_404(Subscription, user=user, is_active=True)

    base_price = subscription.plan.price
    discount = 0.0

    if user.registration_date.year < 2022:
        discount += 0.10
    if subscription.auto_renew_enabled:
        discount += 0.05

    amount_to_charge = base_price * (1.0 - discount)
    amount_to_charge = round(amount_to_charge * 1.20, 2) 

    with httpx.Client() as client:
        response = client.post(
            "https://api.payment-provider.com/charge",
            json={
                "customer_id": user.payment_customer_id,
                "amount": amount_to_charge,
                "currency": "USD"
            },
            headers={"Authorization": f"Bearer {payload.gateway_token}"}
        )
        response.raise_for_status()

    subscription.expires_at = timezone.now() + timezone.timedelta(days=30)
    subscription.save(update_fields=["expires_at"])

    send_mail(
        subject="Subscription Renewed",
        message=f"Thank you! You were charged ${amount_to_charge}.",
        from_email="billing@myapp.com",
        recipient_list=[user.email],
        fail_silently=True,
    )

    return {"status": "success", "new_expiry": subscription.expires_at, "charged": amount_to_charge}
```

---
class: text-sm px-4
---

# SRP - Розбір проблем та Погляд AI

Головні проблеми:
- God Function: Цей ендпоінт робить 5 різних речей: дістає дані з БД, рахує бізнес-логіку (знижки та податки), робить зовнішній HTTP-запит, оновлює БД, відправляє імейл. Це класичне порушення SRP на рівні функції.

- Нетестованість доменної логіки: математика розрахунку ціни (знижка за рік реєстрації, податок 20%) зашита прямо в контролері (роуті). Щоб протестувати тільки правильність обчислення amount_to_charge, доведеться підіймати тестову БД, мокати httpx та send_mail.

- Обробка помилок (Resilience): Якщо відправка імейлу впаде (навіть з fail_silently, наприклад, через мережу), транзакція з платіжкою вже пройшла, але клієнт може отримати 500 помилку, якщо щось піде не так до return. Немає атомарності процесу.

Як AI бачить цей код:
"Код має надмірну зв'язність (High Coupling). Доменна логіка змішана з інфраструктурною (HTTP-клієнт, ORM, SMTP). Контекст для написання unit-тестів занадто великий. Щоб згенерувати один тест на знижку, мені доведеться написати 15 рядків моків (patch('httpx.Client'), patch('send_mail')). Бізнес-правила (податок 20%, знижка 10%) є магічними числами (magic numbers) і приховані глибоко в роуті. Рефакторинг обов'язковий."

---

# SRP - Виправлений код

```python
from decimal import Decimal
from ninja import Router
from django.shortcuts import get_object_or_404
from .models import Subscription, User
from .schemas import SubscriptionRenewSchema
from .services import PaymentGateway, NotificationService
from .domain import calculate_renewal_price

router = Router()

# 1. Pure Domain Logic (can be in domain.py) - легко тестувати без БД!
def calculate_renewal_price(base_price: Decimal, reg_year: int, auto_renew: bool) -> Decimal:
    discount = Decimal("0.0")
    if reg_year < 2022:
        discount += Decimal("0.10")
    if auto_renew:
        discount += Decimal("0.05")
        
    discounted = base_price * (Decimal("1.0") - discount)
    with_tax = discounted * Decimal("1.20")
    return round(with_tax, 2)


# 2. Application Service (can be in services.py)
def process_subscription_renewal(user: User, subscription: Subscription, token: str):
    amount = calculate_renewal_price(
        base_price=subscription.plan.price,
        reg_year=user.registration_date.year,
        auto_renew=subscription.auto_renew_enabled
    )
    
    # Інтеграції винесені в окремі класи/адаптери
    PaymentGateway.charge(user.payment_customer_id, amount, token)
    
    subscription.extend_by_days(30) # Логіка оновлення інкапсульована в моделі
    subscription.save()
    
    # Відправка асинхронно або через ізольований сервіс
    NotificationService.send_receipt(user.email, amount)
    
    return subscription, amount


# 3. Router (Controller) - лише координація HTTP
@router.post("/subscriptions/renew")
def renew_subscription(request, payload: SubscriptionRenewSchema):
    user = get_object_or_404(User, id=payload.user_id)
    subscription = get_object_or_404(Subscription, user=user, is_active=True)

    updated_sub, charged_amount = process_subscription_renewal(
        user=user, 
        subscription=subscription, 
        token=payload.gateway_token
    )

    return {"status": "success", "new_expiry": updated_sub.expires_at, "charged": charged_amount}
```

---
class: px-2
---

# OCP - The Code

```python
from ninja import Router
from django.shortcuts import get_object_or_404
from .models import Order
from .schemas import DeliveryRequestSchema

router = Router()

class DeliveryCostCalculator:
    def apply_delivery_cost(self, order: Order, provider: str, metrics: dict) -> float:
        cost = 0.0
        
        match provider:
            case "ups":
                volumetric = (metrics.get("l", 0) * metrics.get("w", 0) * metrics.get("h", 0)) / 5000
                cost = max(volumetric, metrics.get("weight", 0)) * 2.5
            case "dhl":
                cost = metrics.get("weight", 0) * 4.2 + 15.0
                if order.user.is_premium:
                    cost *= 0.85
            case "local_courier":
                base_rate = 70.0 if metrics.get("city_type") == "urban" else 100.0
                cost = base_rate + (metrics.get("weight", 0) * 10.0)
            case _:
                raise ValueError(f"Unknown provider: {provider}")

        order.delivery_cost = cost
        order.save(update_fields=["delivery_cost"])
        
        return cost

@router.post("/delivery/calculate")
def calculate_delivery(request, payload: DeliveryRequestSchema):
    order = get_object_or_404(Order, id=payload.order_id)
    calculator = DeliveryCostCalculator()
    
    final_cost = calculator.apply_delivery_cost(order, payload.provider, payload.metrics)
    
    return {"order_id": order.id, "delivery_cost": final_cost}
```

---
class: text-sm px-4
---

# OCP - Розбір проблем та Погляд AI

Головні проблеми:
- Порушення OCP: Клас DeliveryCostCalculator не закритий для модифікації. Якщо бізнес вирішить додати нового перевізника (наприклад, fedex), розробнику доведеться лізти в існуючий метод apply_delivery_cost і додавати новий case. Це порушує принцип OCP і створює ризик зламати існуючу логіку розрахунків для інших провайдерів.

- Витік доменної логіки (Domain Leakage): Логіка знижки для преміум-користувачів (order.user.is_premium) зашита виключно в гілку dhl. Калькулятору взагалі не потрібно знати про об'єкт order або user, він має отримувати лише абстрактні параметри.

- Порушення SRP (як бонус): Метод називається "калькулятор", але він ще й оновлює стан об'єкта та зберігає його в БД (order.save()).

Як AI бачить цей код:
"Я бачу високу цикломатичну складність (Cyclomatic Complexity), яка буде невпинно зростати. Цей метод — антипатерн. Щоб написати тести на цей код, мені доведеться створювати моки для Order та User для кожної гілки match/case. При додаванні нового провайдера, мені доведеться переписувати цей клас, що створює ризик регресії. Архітектура не підтримує поліморфізм або патерн Strategy, що є стандартом для таких задач."

---

# OCP - Виправлений код

Тут ми показуємо, як використовувати патерн Strategy та реєстр провайдерів (Registry Pattern). Код відкритий для розширення: щоб додати FedEx, треба просто створити новий клас і додати його в словник PROVIDERS, не змінюючи існуючий код.

```python
from typing import Protocol
from ninja import Router
from django.shortcuts import get_object_or_404
from .models import Order
from .schemas import DeliveryRequestSchema

router = Router()

# 1. Interface (Strategy)
class DeliveryStrategy(Protocol):
    def calculate(self, metrics: dict, is_premium: bool) -> float:
        ...

# 2. Concrete Strategies
class UPSStrategy:
    def calculate(self, metrics: dict, is_premium: bool) -> float:
        volumetric = (metrics.get("l", 0) * metrics.get("w", 0) * metrics.get("h", 0)) / 5000
        return float(max(volumetric, metrics.get("weight", 0)) * 2.5)

class DHLStrategy:
    def calculate(self, metrics: dict, is_premium: bool) -> float:
        cost = metrics.get("weight", 0) * 4.2 + 15.0
        return cost * 0.85 if is_premium else cost

class LocalCourierStrategy:
    def calculate(self, metrics: dict, is_premium: bool) -> float:
        base_rate = 70.0 if metrics.get("city_type") == "urban" else 100.0
        return float(base_rate + (metrics.get("weight", 0) * 10.0))

# 3. Registry
PROVIDERS: dict[str, DeliveryStrategy] = {
    "ups": UPSStrategy(),
    "dhl": DHLStrategy(),
    "local_courier": LocalCourierStrategy(),
}

# 4. Application Logic
def process_delivery_calculation(order: Order, provider_name: str, metrics: dict) -> float:
    strategy = PROVIDERS.get(provider_name)
    if not strategy:
        raise ValueError(f"Unknown provider: {provider_name}")
        
    cost = strategy.calculate(metrics=metrics, is_premium=order.user.is_premium)
    
    order.delivery_cost = cost
    order.save(update_fields=["delivery_cost"])
    return cost

# 5. Controller
@router.post("/delivery/calculate")
def calculate_delivery(request, payload: DeliveryRequestSchema):
    order = get_object_or_404(Order, id=payload.order_id)
    
    final_cost = process_delivery_calculation(
        order=order, 
        provider_name=payload.provider, 
        metrics=payload.metrics
    )
    
    return {"order_id": order.id, "delivery_cost": final_cost}
```

---
class: px-2
---

# LSP - The Code

```python
from typing import List
from ninja import Router
from pydantic import BaseModel

router = Router()

class RefundRequestSchema(BaseModel):
    transaction_ids: List[str]

class PaymentGateway:
    def process_payment(self, amount: float) -> str:
        raise NotImplementedError

    def refund_payment(self, transaction_id: str) -> bool:
        raise NotImplementedError

class StripeGateway(PaymentGateway):
    def process_payment(self, amount: float) -> str:
        return f"txn_stripe_{amount}"

    def refund_payment(self, transaction_id: str) -> bool:
        return True

class CryptoGateway(PaymentGateway):
    def process_payment(self, amount: float, wallet_address: str) -> str:
        return f"txn_crypto_{amount}_{wallet_address}"

    def refund_payment(self, transaction_id: str) -> bool:
        raise Exception("Crypto transactions are immutable and cannot be refunded")

def process_bulk_refunds(gateways: List[PaymentGateway], transaction_ids: List[str]) -> int:
    successful_refunds = 0
    for gateway, txn_id in zip(gateways, transaction_ids):
        if gateway.refund_payment(txn_id):
            successful_refunds += 1
    return successful_refunds

@router.post("/refunds/bulk")
def bulk_refund_endpoint(request, payload: RefundRequestSchema):
    gateways = [StripeGateway(), CryptoGateway()] 
    
    success_count = process_bulk_refunds(gateways, payload.transaction_ids)
    
    return {"successful_refunds": success_count}
```

---
class: text-sm px-4
---

# LSP - Розбір проблем та Погляд AI

Головні проблеми:
- Зміна сигнатури методу: CryptoGateway.process_payment вимагає додатковий аргумент wallet_address. Якщо система спробує викликати цей метод через абстракцію PaymentGateway, програма впаде з TypeError.

- Порушення контракту (Unexpected Exceptions): Базовий клас обіцяє, що refund_payment повертає bool. Натомість CryptoGateway викидає Exception. Функція process_bulk_refunds нічого не знає про це виключення і просто впаде, зупинивши весь процес масового повернення коштів. Це класичне порушення LSP — підклас не може замінити базовий клас без руйнування логіки.

Як AI бачить цей код:
"Статичні аналізатори коду (Mypy) одразу підсвітять помилку сигнатури в CryptoGateway. З точки зору архітектури, порушено контракт інтерфейсу. Базовий клас дає хибну обіцянку, що всі платіжні шлюзи підтримують повернення коштів. Щоб AI міг безпечно рефакторити цей код, йому доведеться додавати потворні перевірки типів на кшталт if isinstance(gateway, CryptoGateway), що автоматично порушить ще й OCP. Потрібно розділити інтерфейси."

---

# LSP - Виправлений код

Рішення: Ми розділяємо базові операції, які підтримують усі, і специфічні (як refund).

```python
from typing import List, Protocol
from ninja import Router
from .schemas import RefundRequestSchema

router = Router()

# 1. Segregated Interfaces
class PaymentProcessor(Protocol):
    def process_payment(self, payload: dict) -> str:
        ...

class Refundable(Protocol):
    def refund_payment(self, transaction_id: str) -> bool:
        ...

# 2. Implementations
class StripeGateway:
    def process_payment(self, payload: dict) -> str:
        return f"txn_stripe_{payload['amount']}"

    def refund_payment(self, transaction_id: str) -> bool:
        return True

class CryptoGateway:
    def process_payment(self, payload: dict) -> str:
        return f"txn_crypto_{payload['amount']}_{payload['wallet_address']}"
    # No refund_payment method here at all!

def process_bulk_refunds(gateways: List[Refundable], transaction_ids: List[str]) -> int:
    successful_refunds = 0
    for gateway, txn_id in zip(gateways, transaction_ids):
        if gateway.refund_payment(txn_id):
            successful_refunds += 1
    return successful_refunds

@router.post("/refunds/bulk")
def bulk_refund_endpoint(request, payload: RefundRequestSchema):
    # Тут можуть бути лише ті шлюзи, які імплементують Refundable
    gateways: List[Refundable] =[StripeGateway()] 
    
    success_count = process_bulk_refunds(gateways, payload.transaction_ids)
    return {"successful_refunds": success_count}
```

---
class: px-2
---

# ISP - The Code

```python
from abc import ABC, abstractmethod
from ninja import Router

router = Router()

class CloudResourceManager(ABC):
    @abstractmethod
    def upload_file(self, file_data: bytes) -> str:
        pass

    @abstractmethod
    def delete_file(self, file_id: str) -> None:
        pass

    @abstractmethod
    def generate_cdn_url(self, file_id: str) -> str:
        pass

    @abstractmethod
    def analyze_content_toxicity(self, file_id: str) -> float:
        pass

class AWSS3Manager(CloudResourceManager):
    def upload_file(self, file_data: bytes) -> str:
        return "s3_file_id"

    def delete_file(self, file_id: str) -> None:
        pass

    def generate_cdn_url(self, file_id: str) -> str:
        return f"https://cdn.aws.com/{file_id}"

    def analyze_content_toxicity(self, file_id: str) -> float:
        return 0.05 

class LocalStorageManager(CloudResourceManager):
    def upload_file(self, file_data: bytes) -> str:
        return "local_file_id"

    def delete_file(self, file_id: str) -> None:
        pass

    def generate_cdn_url(self, file_id: str) -> str:
        raise NotImplementedError("Local storage does not support CDN")

    def analyze_content_toxicity(self, file_id: str) -> float:
        raise NotImplementedError("AI analysis is not available for local storage")
```

---
class: text-sm px-4
---

# ISP - Розбір проблем та Погляд AI

Головні проблеми:
- "Товстий" інтерфейс (Fat Interface): CloudResourceManager змушує всі класи-нащадки реалізовувати методи, які їм можуть бути непотрібні (CDN, AI-аналіз).
Зайві залежності клієнтів: Якщо якомусь сервісу в системі потрібно тільки завантажувати файли, він все одно отримує об'єкт з методами analyze_content_toxicity і generate_cdn_url. Це порушує принцип найменшого здивування.

- Використання NotImplementedError як затички: Якщо клієнт викликає generate_cdn_url для LocalStorageManager, програма падає. Це тісно пов'язано з порушенням LSP, але корінь проблеми — у погано спроектованому базовому інтерфейсі.

Як AI бачить цей код:
"Інтерфейс має низьку когезію (Low Cohesion). Він поєднує три різні домени: Storage, CDN та ML-аналіз. Для генерації тестів це означає, що створення моку для CloudResourceManager вимагатиме реалізації 4 методів, хоча тестована функція може використовувати лише один. Це порушує модульність системи. Набагато ефективніше розбити його на маленькі сфокусовані Protocol-класи."

---

# ISP - Виправлений код

Рішення: Замість одного великого інтерфейсу ми створюємо кілька вузьконаправлених.

```python
from typing import Protocol
from ninja import Router

router = Router()

class FileStorage(Protocol):
    def upload_file(self, file_data: bytes) -> str: ...
    def delete_file(self, file_id: str) -> None: ...

class CDNDistributor(Protocol):
    def generate_cdn_url(self, file_id: str) -> str: ...

class ContentAnalyzer(Protocol):
    def analyze_content_toxicity(self, file_id: str) -> float: ...

# Implementations only do what they actually support
class AWSS3Manager:
    def upload_file(self, file_data: bytes) -> str:
        return "s3_file_id"
        
    def delete_file(self, file_id: str) -> None:
        pass

    def generate_cdn_url(self, file_id: str) -> str:
        return f"https://cdn.aws.com/{file_id}"

    def analyze_content_toxicity(self, file_id: str) -> float:
        return 0.05 

class LocalStorageManager:
    def upload_file(self, file_data: bytes) -> str:
        return "local_file_id"

    def delete_file(self, file_id: str) -> None:
        pass

# Будь-яка функція тепер вимагає тільки потрібний їй інтерфейс:
def process_upload(storage: FileStorage, data: bytes):
    return storage.upload_file(data)
```

---

# DIP - The Code

```python
import smtplib
from ninja import Router
from django.shortcuts import get_object_or_404
from .models import User
from .schemas import MarketingRequestSchema

router = Router()

class PostgresUserRepository:
    def get_active_users(self):
        return User.objects.filter(is_active=True)

class SMTPEmailSender:
    def send_email(self, to_address: str, content: str):
        server = smtplib.SMTP("smtp.provider.com")
        server.sendmail("noreply@app.com", to_address, content)
        server.quit()

class WeeklyNewsletterService:
    def __init__(self):
        self.db = PostgresUserRepository()
        self.mailer = SMTPEmailSender()

    def broadcast_newsletter(self, content: str):
        users = self.db.get_active_users()
        for user in users:
            self.mailer.send_email(user.email, content)

@router.post("/marketing/broadcast")
def trigger_broadcast(request, payload: MarketingRequestSchema):
    service = WeeklyNewsletterService()
    service.broadcast_newsletter(payload.content)
    
    return {"status": "broadcast_initiated"}
```

---
class: text-sm px-4
---

# DIP - Розбір проблем та Погляд AI

Головні проблеми:
- Жорстка зв'язність (Tightly Coupled): Високорівнева бізнес-логіка (WeeklyNewsletterService) напряму створює екземпляри низькорівневих модулів (PostgresUserRepository та SMTPEmailSender) у своєму __init__.

- Неможливість підміни (No Dependency Injection): Якщо ми захочемо відправити імейли через SendGrid замість SMTP, або дістати користувачів з кешу Redis замість Postgres, нам доведеться переписувати код самого WeeklyNewsletterService.

- Жахіття для тестування: Цей сервіс неможливо протестувати юніт-тестами без використання unittest.mock.patch на рівні модулів. Це робить тести крихкими (brittle).

Як AI бачить цей код:
"Відсутність Інверсії Контролю (Inversion of Control). Високорівневий клас диктує, які саме низькорівневі інструменти він буде використовувати, замість того, щоб просто вимагати абстрактні інструменти ззовні. Це робить код негнучким. Якби AI генерував тести для цього класу, йому довелося б мокати внутрішні виклики імпортів і бібліотеку smtplib, що є антипатерном у тестуванні. Набагато краще передавати залежності через конструктор."

---

# DIP - Виправлений код

Рішення: Ми впроваджуємо Dependency Injection (DI). Сервіс тепер залежить від абстракцій (Protocols), а не від конкретних класів.

```python
from typing import Protocol, List
from ninja import Router
from .models import User
from .schemas import MarketingRequestSchema

router = Router()

# 1. Abstractions
class UserRepository(Protocol):
    def get_active_users(self) -> List[User]: ...

class EmailSender(Protocol):
    def send_email(self, to_address: str, content: str) -> None: ...

# 2. High-level module depends on ABSTRACTIONS
class WeeklyNewsletterService:
    def __init__(self, db: UserRepository, mailer: EmailSender):
        self.db = db
        self.mailer = mailer

    def broadcast_newsletter(self, content: str):
        users = self.db.get_active_users()
        for user in users:
            self.mailer.send_email(user.email, content)

# 3. Low-level modules implement abstractions
class PostgresUserRepository:
    def get_active_users(self) -> List[User]:
        return User.objects.filter(is_active=True)

class SMTPEmailSender:
    def send_email(self, to_address: str, content: str) -> None:
        pass # SMTP logic here

# 4. Controller wires them together (Dependency Injection)
@router.post("/marketing/broadcast")
def trigger_broadcast(request, payload: MarketingRequestSchema):
    # In a real app, you might use a DI container (like dependency-injector)
    db_repo = PostgresUserRepository()
    email_sender = SMTPEmailSender()
    
    service = WeeklyNewsletterService(db=db_repo, mailer=email_sender)
    service.broadcast_newsletter(payload.content)
    
    return {"status": "broadcast_initiated"}
```

---

# Summary

**SOLID Principles:**

| Principle | Problem | Solution |
|-----------|---------|----------|
| **SRP** | God Function | Separate domain logic |
| **OCP** | Modify for new features | Strategy Pattern |
| **LSP** | Subclass breaks contract | Interface Segregation |
| **ISP** | Fat interface | Small focused protocols |
| **DIP** | Tight coupling | Dependency Injection |

---

# Your Turn

Add your own code examples for discussion

---

# Discussion Summary

Key takeaways from this interview:

-

-

-

