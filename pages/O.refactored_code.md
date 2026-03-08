# OCP - Виправлений код

Тут ми показуємо, як використовувати патерн Strategy та реєстр провайдерів (Registry Pattern). Код відкритий для розширення: щоб додати FedEx, треба просто створити новий клас і додати його в словник PROVIDERS, не змінюючи існуючий код.

```python {1-}{lines:true, maxHeight:'400px'}
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
