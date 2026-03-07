# LSP - The Code

```python {1-}{lines:true, maxHeight:'400px'}
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
