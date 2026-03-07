# DIP - Виправлений код

Рішення: Ми впроваджуємо Dependency Injection (DI). Сервіс тепер залежить від абстракцій (Protocols), а не від конкретних класів.

```python {1-}{lines:true, maxHeight:'400px'}
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
    db_repo = PostgresUserRepository()
    email_sender = SMTPEmailSender()
    
    service = WeeklyNewsletterService(db=db_repo, mailer=email_sender)
    service.broadcast_newsletter(payload.content)
    
    return {"status": "broadcast_initiated"}
```
