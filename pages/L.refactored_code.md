# LSP - Виправлений код

Рішення: Ми розділяємо базові операції, які підтримують усі, і специфічні (як refund).

```python {1-}{lines:true, maxHeight:'400px'}
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
