# Phase 09: Cross-App Features

Standard features that every app should include. Add these after the core build is complete.

**Prerequisite:** Phase 08 complete (app containerized and deployable).

---

## In-App Feedback via GitHub Issues

Give users a simple way to report bugs or request features without leaving the app.

### How It Works

1. User clicks a feedback button in the app header
2. A modal collects a title and description
3. The backend calls the GitHub REST API to create an issue with a "feedback" label
4. The user sees a success confirmation

### Backend

#### Config additions

Add to `api/config.py` Settings class:

```python
from typing import Optional

# In-app feedback → GitHub Issues
feedback_github_token: Optional[str] = None
feedback_github_repo_owner: Optional[str] = None
feedback_github_repo_name: Optional[str] = None
```

Add to `.env.example`:

```
FEEDBACK_GITHUB_TOKEN=
FEEDBACK_GITHUB_REPO_OWNER=
FEEDBACK_GITHUB_REPO_NAME=
```

#### Service

Create `api/services/github_service.py`:

```python
import httpx
import logging
from api.config import get_settings

logger = logging.getLogger(__name__)

class GitHubService:
    @staticmethod
    async def create_issue(title: str, body: str, labels: list[str] | None = None) -> dict | None:
        settings = get_settings()
        if not all([settings.feedback_github_token, settings.feedback_github_repo_owner, settings.feedback_github_repo_name]):
            logger.warning("GitHub feedback not configured")
            return None

        url = f"https://api.github.com/repos/{settings.feedback_github_repo_owner}/{settings.feedback_github_repo_name}/issues"
        headers = {
            "Authorization": f"Bearer {settings.feedback_github_token}",
            "Accept": "application/vnd.github.v3+json",
        }
        payload = {"title": title, "body": body}
        if labels:
            payload["labels"] = labels

        async with httpx.AsyncClient() as client:
            response = await client.post(url, json=payload, headers=headers)
            if response.status_code == 201:
                data = response.json()
                return {"number": data["number"], "url": data["html_url"]}
            logger.error("GitHub API error: %s %s", response.status_code, response.text)
            return None


async def create_feedback_issue(title: str, description: str, username: str | None = None) -> dict | None:
    body_parts = []
    if username:
        body_parts.append(f"### Submitted By\n{username}")
    body_parts.append(f"### Feedback\n{description}")
    body_parts.append("*Submitted via in-app feedback*")
    body = "\n\n".join(body_parts)
    return await GitHubService.create_issue(title, body, labels=["feedback"])
```

#### Route

Create `api/routes/feedback.py`:

```python
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, Field
from typing import Optional
from api.utils.auth import get_current_user
from api.services.github_service import create_feedback_issue

router = APIRouter()

class FeedbackRequest(BaseModel):
    title: str = Field(..., max_length=200)
    description: str

class FeedbackResponse(BaseModel):
    success: bool
    issue_number: Optional[int] = None
    issue_url: Optional[str] = None
    message: str

@router.post("", response_model=FeedbackResponse)
async def submit_feedback(data: FeedbackRequest, user=Depends(get_current_user)):
    result = await create_feedback_issue(data.title, data.description, username=user.username)
    if result:
        return FeedbackResponse(
            success=True, issue_number=result["number"],
            issue_url=result["url"], message="Feedback submitted"
        )
    raise HTTPException(status_code=500, detail="Failed to submit feedback")
```

Register in `api/main.py`:

```python
from api.routes import feedback
app.include_router(feedback.router, prefix="/api/feedback", tags=["feedback"])
```

### Frontend

Add a `FeedbackModal` component with a title input and description textarea. Wire a button in the app header to open it. On submit, call `api.post("/api/feedback", { title, description })` and show a success message before auto-closing.

### GitHub Token

Create a fine-grained personal access token with **Issues: Read and Write** permission scoped to the target repository. Add it as `FEEDBACK_GITHUB_TOKEN` in your environment.

Ensure `httpx` is in your `pyproject.toml` dependencies (it should be from Phase 01).

---

## Checklist

- [ ] GitHub feedback config added to Settings and `.env.example`
- [ ] `api/services/github_service.py` created
- [ ] `api/routes/feedback.py` created and registered in `main.py`
- [ ] Feedback modal component added to the frontend header
- [ ] GitHub PAT created with Issues permission and added to environment
- [ ] Feedback submission works end-to-end
