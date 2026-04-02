---
name: django-realtime-communication
description: >
  Apply this skill when implementing any real-time or event-driven communication:
  WebSockets (Django Channels), Server-Sent Events (SSE), Webhooks (inbound/outbound),
  long-polling. Includes protocol selection guide, Django Channels consumers, channel
  layers, Redis pub/sub, webhook HMAC verification, and SSE async streaming.
  Use when asked about real-time features, live updates, push notifications, or
  integrating with external webhook-based services.
---

# Django Real-Time Communication Skill

> FIRST: For Django Channels, query Context7 for `channels` and `daphne` latest docs.
> For Django async, use Context7 with `/websites/djangoproject_en_6_0`.

---

## 1. Communication Protocol Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│              WHEN TO USE WHICH PROTOCOL                         │
├──────────────────┬──────────────────────────────────────────────┤
│ REST API (HTTP)  │ Request-response. Client initiates.          │
│                  │ CRUD, queries, commands.                     │
│                  │ Use: 99% of API calls                        │
├──────────────────┼──────────────────────────────────────────────┤
│ WebSocket        │ Persistent bidirectional. Both sides push.   │
│                  │ Use: Chat, collaborative editing,            │
│                  │      multiplayer, live trading               │
│                  │ Cost: High — connection held open per client │
├──────────────────┼──────────────────────────────────────────────┤
│ SSE              │ Persistent unidirectional (server→client).   │
│                  │ Use: Live feeds, progress bars, dashboards,  │
│                  │      notifications, log streaming            │
│                  │ Cost: Medium — connection per client         │
│                  │ Pro: Auto-reconnect, works over HTTP/2       │
├──────────────────┼──────────────────────────────────────────────┤
│ Webhook          │ Server-to-server event notification.         │
│                  │ Use: Payment events, CI/CD, external         │
│                  │      integrations (Stripe, GitHub, etc.)     │
│                  │ Cost: Low — stateless HTTP POST              │
├──────────────────┼──────────────────────────────────────────────┤
│ Long Polling     │ Client polls and server holds response.      │
│                  │ Use: LAST RESORT — when WebSocket/SSE        │
│                  │      not supported by client                 │
│                  │ Cost: Very high — thread per poll            │
└──────────────────┴──────────────────────────────────────────────┘
```

---

## 2. Django Channels — WebSocket Setup

### Installation & ASGI Configuration
```python
# config/asgi.py
from __future__ import annotations
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.security.websocket import AllowedHostsOriginValidator
from apps.core.middleware import JWTAuthMiddlewareStack

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.production")

# IMPORTANT: Initialize Django BEFORE importing consumers
django_asgi_app = get_asgi_application()

from config.websocket_router import websocket_urlpatterns  # noqa: E402

application = ProtocolTypeRouter({
    "http": django_asgi_app,
    "websocket": AllowedHostsOriginValidator(
        JWTAuthMiddlewareStack(
            URLRouter(websocket_urlpatterns)
        )
    ),
})

# config/websocket_router.py
from django.urls import re_path
from apps.chat.consumers import ChatConsumer
from apps.notifications.consumers import NotificationConsumer

websocket_urlpatterns = [
    re_path(r"ws/chat/(?P<room_name>\w+)/$", ChatConsumer.as_asgi()),
    re_path(r"ws/notifications/$", NotificationConsumer.as_asgi()),
]
```

### Channel Layers (Redis)
```python
# config/settings/base.py
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [env("REDIS_URL", default="redis://localhost:6379/1")],
            "capacity": 1500,       # Max messages per channel
            "expiry": 10,           # Message TTL in seconds
            "group_expiry": 86400,  # Group membership TTL (1 day)
        },
    },
}
```

---

## 3. WebSocket Consumers

### JWT Auth Middleware for WebSockets
```python
# apps/core/middleware.py
from __future__ import annotations
from typing import Any
from channels.middleware import BaseMiddleware
from channels.auth import AuthMiddlewareStack
from django.contrib.auth import get_user_model
from django.contrib.auth.models import AnonymousUser
from rest_framework_simplejwt.tokens import AccessToken
from rest_framework_simplejwt.exceptions import InvalidToken, TokenError
import logging

logger = logging.getLogger(__name__)
User = get_user_model()


class JWTAuthMiddleware(BaseMiddleware):
    """Authenticate WebSocket connections via JWT in query string."""

    async def __call__(self, scope: dict[str, Any], receive: Any, send: Any) -> None:
        scope["user"] = await self._get_user(scope)
        await super().__call__(scope, receive, send)

    async def _get_user(self, scope: dict[str, Any]) -> Any:
        from urllib.parse import parse_qs
        query_string = scope.get("query_string", b"").decode()
        params = parse_qs(query_string)
        token_list = params.get("token", [])
        if not token_list:
            return AnonymousUser()
        try:
            token = AccessToken(token_list[0])
            user_id = token.get("user_id")
            from asgiref.sync import sync_to_async
            return await sync_to_async(
                User.objects.select_related().get
            )(pk=user_id, is_active=True)
        except (InvalidToken, TokenError, User.DoesNotExist) as exc:
            logger.debug("WS JWT auth failed: %s", exc)
            return AnonymousUser()


def JWTAuthMiddlewareStack(inner: Any) -> JWTAuthMiddleware:
    return JWTAuthMiddleware(AuthMiddlewareStack(inner))
```

### Chat Consumer (Bidirectional)
```python
# apps/chat/consumers.py
from __future__ import annotations
import json
import logging
from typing import Any
from channels.generic.websocket import AsyncWebsocketConsumer
from asgiref.sync import sync_to_async

logger = logging.getLogger(__name__)


class ChatConsumer(AsyncWebsocketConsumer):
    """
    WebSocket consumer for room-based chat.
    Group naming: chat_{room_name}
    """

    async def connect(self) -> None:
        self.room_name: str = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name: str = f"chat_{self.room_name}"
        self.user = self.scope["user"]

        # Reject unauthenticated connections
        if not self.user or not self.user.is_authenticated:
            await self.close(code=4001)
            return

        # Verify room access
        if not await self._user_can_access_room(self.room_name):
            await self.close(code=4003)
            return

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()
        logger.info("WS connected: user=%s room=%s", self.user.pk, self.room_name)

    async def disconnect(self, close_code: int) -> None:
        if hasattr(self, "room_group_name"):
            await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data: str | None = None, bytes_data: bytes | None = None) -> None:
        if not text_data:
            return
        try:
            data = json.loads(text_data)
        except json.JSONDecodeError:
            await self.send(text_data=json.dumps({"type": "error", "message": "Invalid JSON"}))
            return

        message_type = data.get("type")
        if message_type == "chat.message":
            await self._handle_chat_message(data)
        elif message_type == "chat.typing":
            await self._handle_typing(data)

    async def _handle_chat_message(self, data: dict[str, Any]) -> None:
        content = str(data.get("content", "")).strip()
        if not content or len(content) > 2000:
            await self.send(text_data=json.dumps({
                "type": "error", "message": "Invalid message content"
            }))
            return

        # Persist message
        message = await self._save_message(content)

        # Broadcast to room group
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                "type": "chat.message",  # Maps to chat_message method
                "message_id": str(message.pk),
                "content": content,
                "user_id": str(self.user.pk),
                "user_name": self.user.get_full_name(),
            },
        )

    async def _handle_typing(self, data: dict[str, Any]) -> None:
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                "type": "chat.typing",
                "user_id": str(self.user.pk),
                "is_typing": bool(data.get("is_typing", False)),
            },
        )

    # Handlers for group messages (method name = event type with "." → "_")
    async def chat_message(self, event: dict[str, Any]) -> None:
        await self.send(text_data=json.dumps({
            "type": "chat.message",
            "message_id": event["message_id"],
            "content": event["content"],
            "user_id": event["user_id"],
            "user_name": event["user_name"],
        }))

    async def chat_typing(self, event: dict[str, Any]) -> None:
        await self.send(text_data=json.dumps({
            "type": "chat.typing",
            "user_id": event["user_id"],
            "is_typing": event["is_typing"],
        }))

    @sync_to_async
    def _save_message(self, content: str) -> Any:
        from apps.chat.models import Message
        return Message.objects.create(
            room_name=self.room_name,
            user=self.user,
            content=content,
        )

    @sync_to_async
    def _user_can_access_room(self, room_name: str) -> bool:
        from apps.chat.models import ChatRoom
        return ChatRoom.objects.filter(
            name=room_name,
            members=self.user
        ).exists()
```

### Notification Consumer (Server-Push Only)
```python
# apps/notifications/consumers.py
from __future__ import annotations
import json
from typing import Any
from channels.generic.websocket import AsyncWebsocketConsumer


class NotificationConsumer(AsyncWebsocketConsumer):
    """One-way notification push. Server sends, client only receives."""

    async def connect(self) -> None:
        self.user = self.scope["user"]
        if not self.user.is_authenticated:
            await self.close(code=4001)
            return

        self.group_name = f"notifications_{self.user.pk}"
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code: int) -> None:
        if hasattr(self, "group_name"):
            await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def receive(self, text_data: str | None = None, bytes_data: bytes | None = None) -> None:
        # Ignore client messages — this is server-push only
        pass

    async def notify(self, event: dict[str, Any]) -> None:
        """Handler for 'notify' group events."""
        await self.send(text_data=json.dumps({
            "type": "notification",
            "notification_id": event.get("notification_id"),
            "title": event.get("title"),
            "body": event.get("body"),
            "action_url": event.get("action_url"),
        }))


# Utility to send notification from anywhere (Celery task, service, etc.)
async def push_notification_to_user(
    user_id: str,
    title: str,
    body: str,
    action_url: str = "",
) -> None:
    from channels.layers import get_channel_layer
    channel_layer = get_channel_layer()
    await channel_layer.group_send(
        f"notifications_{user_id}",
        {
            "type": "notify",  # Maps to notify() handler
            "title": title,
            "body": body,
            "action_url": action_url,
        },
    )
```

---

## 4. Server-Sent Events (SSE)

```python
# apps/core/sse.py
"""
SSE via Django async views.
Use for: live dashboards, progress tracking, activity feeds.
Advantages over WebSocket: simpler, HTTP/2 multiplexed, auto-reconnect.
"""
from __future__ import annotations
import asyncio
import json
from typing import AsyncIterator, Any
from django.http import StreamingHttpResponse, HttpRequest


class SSEResponse(StreamingHttpResponse):
    """HTTP response that streams SSE events."""

    def __init__(self, streaming_content: AsyncIterator[str], **kwargs: Any) -> None:
        super().__init__(
            streaming_content=streaming_content,
            content_type="text/event-stream",
            **kwargs,
        )
        self["Cache-Control"] = "no-cache"
        self["X-Accel-Buffering"] = "no"  # Disable Nginx buffering
        self["Access-Control-Allow-Origin"] = "*"


def sse_event(
    data: Any,
    event: str | None = None,
    event_id: str | None = None,
    retry: int | None = None,
) -> str:
    """Format a single SSE event string."""
    lines: list[str] = []
    if event_id:
        lines.append(f"id: {event_id}")
    if event:
        lines.append(f"event: {event}")
    if retry:
        lines.append(f"retry: {retry}")
    if isinstance(data, (dict, list)):
        lines.append(f"data: {json.dumps(data)}")
    else:
        lines.append(f"data: {data}")
    lines.append("")  # Empty line = end of event
    lines.append("")
    return "\n".join(lines)


# apps/jobs/views.py
async def job_progress_stream(request: HttpRequest, job_id: str) -> SSEResponse:
    """Stream job progress to client via SSE."""
    from apps.jobs.selectors import get_job_or_404

    if not request.user.is_authenticated:
        from django.http import HttpResponseForbidden
        return HttpResponseForbidden()

    async def event_generator() -> AsyncIterator[str]:
        # Send initial retry interval
        yield sse_event(data="connected", event="connected", retry=5000)

        last_status = None
        max_polls = 300  # 5 minutes at 1s interval
        for _ in range(max_polls):
            from asgiref.sync import sync_to_async
            job = await sync_to_async(get_job_or_404)(job_id)

            if job.status != last_status:
                last_status = job.status
                yield sse_event(
                    data={"status": job.status, "progress": job.progress, "message": job.message},
                    event="progress",
                    event_id=str(job.updated_at.timestamp()),
                )

            if job.status in ("completed", "failed"):
                yield sse_event(data={"status": job.status}, event="done")
                break

            await asyncio.sleep(1)

    return SSEResponse(event_generator())
```

---

## 5. Webhooks — Inbound (Receiving External Events)

```python
# apps/integrations/views.py
"""
Inbound webhook endpoint rules:
1. ALWAYS verify HMAC signature before processing
2. Return 200 immediately — process async via Celery
3. Idempotency — use event_id to avoid double processing
4. Log ALL incoming webhook payloads for debugging
"""
from __future__ import annotations
import hashlib
import hmac
import logging
from django.conf import settings
from django.http import HttpRequest, HttpResponse, HttpResponseBadRequest
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_POST

logger = logging.getLogger(__name__)


@csrf_exempt
@require_POST
def stripe_webhook(request: HttpRequest) -> HttpResponse:
    """Receive and process Stripe webhook events."""
    payload = request.body
    sig_header = request.META.get("HTTP_STRIPE_SIGNATURE", "")

    if not _verify_stripe_signature(payload, sig_header):
        logger.warning("Invalid Stripe webhook signature")
        return HttpResponseBadRequest("Invalid signature")

    import json
    try:
        event = json.loads(payload)
    except json.JSONDecodeError:
        return HttpResponseBadRequest("Invalid JSON")

    event_id = event.get("id")
    event_type = event.get("type")

    logger.info("Stripe webhook received", extra={"event_id": event_id, "type": event_type})

    # Idempotency check
    from apps.integrations.models import WebhookEvent
    if WebhookEvent.objects.filter(external_id=event_id).exists():
        logger.info("Duplicate webhook event %s — skipping", event_id)
        return HttpResponse(status=200)

    # Persist and dispatch
    from django.db import transaction
    with transaction.atomic():
        webhook_event = WebhookEvent.objects.create(
            source="stripe",
            external_id=event_id,
            event_type=event_type,
            payload=event,
        )
        transaction.on_commit(
            lambda: _process_stripe_event.delay(str(webhook_event.pk))
        )

    return HttpResponse(status=200)


def _verify_stripe_signature(payload: bytes, sig_header: str) -> bool:
    """Verify Stripe webhook signature using HMAC-SHA256."""
    webhook_secret = settings.STRIPE_WEBHOOK_SECRET
    try:
        import stripe
        stripe.WebhookSignature.verify_header(
            payload.decode("utf-8"),
            sig_header,
            webhook_secret,
        )
        return True
    except Exception:
        return False


def _verify_hmac_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Generic HMAC-SHA256 verification for custom webhooks."""
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256,
    ).hexdigest()
    # Use hmac.compare_digest to prevent timing attacks
    return hmac.compare_digest(expected, signature)
```

---

## 6. Webhooks — Outbound (Sending Events to Clients)

```python
# apps/webhooks/services.py
"""
Outbound webhook delivery with:
- Retry with exponential backoff
- Signature signing (so recipients can verify)
- Delivery logging
"""
from __future__ import annotations
import hashlib
import hmac
import json
import logging
import time
from typing import Any
import httpx
from django.conf import settings

logger = logging.getLogger(__name__)


def deliver_webhook(
    *,
    url: str,
    event_type: str,
    payload: dict[str, Any],
    secret: str,
    delivery_id: str,
) -> bool:
    """Deliver an outbound webhook. Returns True on success."""
    body = json.dumps({"event": event_type, "data": payload, "delivery_id": delivery_id})
    timestamp = str(int(time.time()))
    signature = _sign_payload(body, secret, timestamp)

    headers = {
        "Content-Type": "application/json",
        "X-Webhook-Signature": f"t={timestamp},v1={signature}",
        "X-Webhook-Event": event_type,
        "X-Delivery-ID": delivery_id,
    }

    try:
        with httpx.Client(timeout=30) as client:
            response = client.post(url, content=body, headers=headers)
            response.raise_for_status()
            logger.info(
                "Webhook delivered",
                extra={"url": url, "event": event_type, "status": response.status_code}
            )
            return True
    except httpx.HTTPStatusError as exc:
        logger.warning(
            "Webhook delivery failed (HTTP %d)",
            exc.response.status_code,
            extra={"url": url, "event": event_type},
        )
        return False
    except httpx.RequestError as exc:
        logger.warning("Webhook delivery network error: %s", exc, extra={"url": url})
        return False


def _sign_payload(body: str, secret: str, timestamp: str) -> str:
    signed_content = f"{timestamp}.{body}"
    return hmac.new(secret.encode(), signed_content.encode(), hashlib.sha256).hexdigest()
```

---

## 7. Long Polling (Last Resort)

```python
# apps/notifications/views.py
"""
Long polling: Use ONLY when SSE/WebSocket is not viable.
The client sends a request, server holds it until new data arrives or timeout.
Requires ASGI for efficiency — blocks a thread in WSGI.
"""
import asyncio
from django.http import HttpRequest, JsonResponse


async def long_poll_notifications(request: HttpRequest) -> JsonResponse:
    """Return new notifications or empty after 30s timeout."""
    if not request.user.is_authenticated:
        return JsonResponse({"error": "Unauthorized"}, status=401)

    last_id = request.GET.get("last_id", "0")
    max_wait = 30  # seconds
    poll_interval = 1
    elapsed = 0

    while elapsed < max_wait:
        from asgiref.sync import sync_to_async
        notifications = await sync_to_async(
            lambda: list(
                request.user.notifications
                .filter(pk__gt=last_id, is_read=False)
                .values("id", "title", "body")[:10]
            )
        )()

        if notifications:
            return JsonResponse({"notifications": notifications})

        await asyncio.sleep(poll_interval)
        elapsed += poll_interval

    return JsonResponse({"notifications": []})  # Timeout — client should re-poll
```

---

## 8. Production Deployment for Channels

```python
# Daphne (ASGI server for Channels)
# Run with: daphne -b 0.0.0.0 -p 8000 config.asgi:application

# Or Uvicorn (preferred for pure ASGI)
# Run with: uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --workers 4

# Nginx config for WebSocket proxying:
# location /ws/ {
#     proxy_pass http://django;
#     proxy_http_version 1.1;
#     proxy_set_header Upgrade $http_upgrade;
#     proxy_set_header Connection "upgrade";
#     proxy_set_header Host $host;
#     proxy_read_timeout 86400;  # Keep WS alive
# }

# Key settings for Channels in production:
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [{"address": env("REDIS_URL")}],
            "capacity": 1500,
            "expiry": 10,
        },
    }
}
```

---

## 9. WebSocket Message Schema Standards

```python
# All WebSocket messages follow this schema:
# CLIENT → SERVER:
{
    "type": "chat.message",     # namespace.action format
    "payload": {                # Actual data
        "content": "Hello!",
        "room_id": "abc123"
    },
    "request_id": "uuid"        # Optional: for client-side correlation
}

# SERVER → CLIENT:
{
    "type": "chat.message",
    "payload": {...},
    "timestamp": 1234567890,    # Unix milliseconds
    "event_id": "uuid"          # For deduplication on reconnect
}

# Error:
{
    "type": "error",
    "code": "invalid_payload",
    "message": "Content cannot be empty."
}

# Server-initiated disconnect:
{
    "type": "connection.close",
    "code": 4003,
    "reason": "Forbidden"
}
```
