# SRP - The Code

```python {1-}{lines:true, maxHeight:'400px'}
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
