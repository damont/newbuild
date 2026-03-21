# Reference: File Storage

Implementation guide for apps that need to store user-uploaded files or generated assets (images, PDFs, exports, etc.). Uses a storage abstraction that supports local filesystem and Azure Blob Storage.

**Pull this in when:** your app needs file upload/download capabilities.

---

## Storage Abstraction

Create `api/services/storage.py`:

```python
import logging
from pathlib import Path
from abc import ABC, abstractmethod

logger = logging.getLogger(__name__)


class StorageBackend(ABC):
    """Abstract file storage interface."""

    @abstractmethod
    async def save(self, key: str, data: bytes, content_type: str | None = None) -> str:
        """Save file data under the given key. Returns the storage path/URL."""
        ...

    @abstractmethod
    async def load(self, key: str) -> bytes | None:
        """Load file data by key. Returns None if not found."""
        ...

    @abstractmethod
    async def delete(self, key: str) -> bool:
        """Delete a file by key. Returns True if deleted."""
        ...

    @abstractmethod
    async def exists(self, key: str) -> bool:
        """Check if a file exists."""
        ...


class LocalStorageBackend(StorageBackend):
    """Stores files on the local filesystem. Use for development and Pi deployment."""

    def __init__(self, base_dir: str = "./uploads"):
        self.base_dir = Path(base_dir)
        self.base_dir.mkdir(parents=True, exist_ok=True)

    async def save(self, key: str, data: bytes, content_type: str | None = None) -> str:
        path = self.base_dir / key
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_bytes(data)
        return str(path)

    async def load(self, key: str) -> bytes | None:
        path = self.base_dir / key
        if path.exists():
            return path.read_bytes()
        return None

    async def delete(self, key: str) -> bool:
        path = self.base_dir / key
        if path.exists():
            path.unlink()
            return True
        return False

    async def exists(self, key: str) -> bool:
        return (self.base_dir / key).exists()


class AzureBlobStorageBackend(StorageBackend):
    """Stores files in Azure Blob Storage. Use for Azure deployment."""

    def __init__(self, connection_string: str, container_name: str = "uploads"):
        from azure.storage.blob.aio import BlobServiceClient
        self.client = BlobServiceClient.from_connection_string(connection_string)
        self.container_name = container_name

    async def save(self, key: str, data: bytes, content_type: str | None = None) -> str:
        container = self.client.get_container_client(self.container_name)
        kwargs = {}
        if content_type:
            from azure.storage.blob import ContentSettings
            kwargs["content_settings"] = ContentSettings(content_type=content_type)
        await container.upload_blob(key, data, overwrite=True, **kwargs)
        return f"{self.container_name}/{key}"

    async def load(self, key: str) -> bytes | None:
        container = self.client.get_container_client(self.container_name)
        try:
            blob = await container.download_blob(key)
            return await blob.readall()
        except Exception:
            return None

    async def delete(self, key: str) -> bool:
        container = self.client.get_container_client(self.container_name)
        try:
            await container.delete_blob(key)
            return True
        except Exception:
            return False

    async def exists(self, key: str) -> bool:
        container = self.client.get_container_client(self.container_name)
        try:
            await container.get_blob_properties(key)
            return True
        except Exception:
            return False
```

---

## Configuration

Add to `api/config.py` Settings class:

```python
from typing import Optional

# File storage
storage_backend: str = "local"  # "local" or "azure_blob"
upload_dir: str = "./uploads"   # Local backend only
azure_storage_connection_string: Optional[str] = None  # Azure backend only
azure_storage_container: str = "uploads"  # Azure backend only
```

Add to `.env.example`:

```
STORAGE_BACKEND=local
UPLOAD_DIR=./uploads
# AZURE_STORAGE_CONNECTION_STRING=  # Azure only
# AZURE_STORAGE_CONTAINER=uploads   # Azure only
```

---

## Lifespan Initialization

In `api/main.py`, initialize the storage backend during lifespan:

```python
from api.services.storage import LocalStorageBackend, AzureBlobStorageBackend, StorageBackend

storage: StorageBackend | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global storage
    settings = get_settings()

    if settings.storage_backend == "azure_blob":
        storage = AzureBlobStorageBackend(
            settings.azure_storage_connection_string,
            settings.azure_storage_container,
        )
    else:
        storage = LocalStorageBackend(settings.upload_dir)

    # ... rest of existing lifespan (MongoDB init, etc.)
    yield
    # ... existing shutdown
```

---

## Docker Compose

Mount a volume so uploads persist across container restarts:

```yaml
services:
  api:
    volumes:
      - uploads:/app/uploads
    environment:
      - STORAGE_BACKEND=local
      - UPLOAD_DIR=/app/uploads

volumes:
  uploads:
```

---

## Key Rules

- **Never hardcode a storage path in route or service code.** Always go through the storage backend.
- **Use structured keys** that include the project/user context: `{project_id}/{artifact_id}/{filename}`.
- **Add `azure-storage-blob[aio]` to `pyproject.toml`** only when the app actually needs Azure blob support. The local backend has no extra dependencies.
- **For Pi deployment**, the Docker volume is sufficient. For Azure, use a Storage Account and pass the connection string as a secret through the Terraform/GitHub Environments pipeline.
