# ISP - Виправлений код

Рішення: Замість одного великого інтерфейсу ми створюємо кілька вузьконаправлених.

```python {1-}{lines:true, maxHeight:'400px'}
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
