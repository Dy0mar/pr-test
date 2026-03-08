# SRP - Виправлений код

```python {1-}{lines:true, maxHeight:'400px'}
# 1. Pure Domain Logic (can be in domain.py) - легко тестувати без БД!
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
