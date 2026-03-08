# DIP - The Code

```python {1-}{lines:true, maxHeight:'400px'}
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
