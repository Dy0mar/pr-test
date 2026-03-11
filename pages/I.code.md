# The Code

```python {1-}{lines:true, maxHeight:'400px'}
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
