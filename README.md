# Django Channels Practice
 Practicing with Django Channels.

# Setup
## 1. Start by installing both `channels` and `channels_redis`:

```shell
pip install channels channels_redis
```

The package `channels` is the core Django Channels library.

The package `channels_redis` provides a channel layer backend using Redis.

## 2. Modify Django Project Settings

Update `settings.py` by adding it to `INSTALLED_APPS`:
```python
INSTALLED_APPS = [
    ...
    'channels',
    ...
]
```

## 3. Add Channel Layer Configurations
```python
# Use channels layer as the default backend for `asgi`
ASGI_APPLICATION = '<project_name_here>.routing.application'

# Channel layer settings
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            "hosts": [('127.0.0.1', 6379)],  # Pointing to local Redis server
        },
    },
}
```

## 4. Setup Routing
Create a `routing.py` file at the same level as `settings.py`. Uncomment and modify the below code when you create
your consumers (WebSocket views).
```python
from channels.routing import ProtocolTypeRouter, URLRouter
from django.urls import path

# Import your consumers here

application = ProtocolTypeRouter({
    # 'websocket': URLRouter(
    #     [
    #         path('ws/some_path/', YourConsumer.as_asgi()),
    #     ]
    # ),
})
```

## 5. Create Consumers
A consumer in Django Channels acts like a view but for WebSockets.

In your Django app, create a file named `consumers.py`:
```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer

class YourConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        await self.accept()

    async def disconnect(self, close_code):
        pass

    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        await self.send(text_data=json.dumps({
            'message': message
        }))
```

## 6. Install and Run Redis
You can install Redis [here](https://redis.io/download).

You can start Redis using:
```shell
redis-server
```

## 7. Run Development Server
Django Channels requires you to use the ASGI server instead of the usual WSGI. A good choice is `daphne`.
```shell
pip install daphne
```

Run:
```shell
daphne <your_project_name>.asgi:application
```

The project should now be running with Django Channels

# Adding Real Time Features
## 1. Define your Consumers
### Create consumers.py if Needed
Create a file named `consumers.py` in one of your Django apps:
```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        await self.accept()

    async def disconnect(self, close_code):
        pass

    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        await self.send(text_data=json.dumps({
            'message': message
        }))
```

In this basic consumer:
* `connect()`: Called when the websocket is handshaking as part of the connection process.
* `disconnect()`: Called when the WebSocket closes for any reason.
* `receive()`: Called when the server receives a message from WebSocket.

### Group Consumers
Django Channels provides a way to create groups of consumers. Groups are helpful if you want to broadcast
a message to multiple clients, like in a chat room.

```python
from channels.generic.websocket import AsyncWebsocketConsumer
import json

class RoomConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = f"chat_{self.room_name}"

        # Join room group
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )

        await self.accept()

    async def disconnect(self, close_code):
        # Leave room group
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )

    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        # Send message to room group
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message
            }
        )

    # Receive message from room group
    async def chat_message(self, event):
        message = event['message']

        # Send message to WebSocket
        await self.send(text_data=json.dumps({
            'message': message
        }))
```

In this example:
* We extract the `room_name` from the URL and create a group named based on that.
* When a user connects, they're added to the group.
* When a user sends a message, it's sent to the group and then relayed to all clients in the group.
* The `chat_message` method is a custom handler for the `chat_message` type. The `type` key corresponds to a method name in the consumer.

## 2. Connect them to Routes in routing.py
Wire up consumers in `routing.py`. From the above example using `RoomConsumer`:

```python
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<room_name>\w+)/$', consumers.RoomConsumer.as_asgi()),
]
```

Then inside of `routing.py`:
```python
from channels.routing import ProtocolTypeRouter, URLRouter
from your_app_name import routing

application = ProtocolTypeRouter({
    'websocket': URLRouter(routing.websocket_urlpatterns),
})
```

This will rout WebSocket requests to `RoomConsumer` based on the provided `room_name`.

## 3. Frontend Client