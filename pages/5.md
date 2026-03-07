# OCP - The Code

```python {1-}{lines:true, maxHeight:'400px'}
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
