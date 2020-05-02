# README #

### What is this repository for? ###
The idea behind this repo is to guide you through the steps needed to implement a websocket communication using a Django based backend. It will be noise free, in the sense of not including unneeded libraries.  There will be included examples of how to develop and how to deploy. Please note this guide is not for complete begginers, so for simplicity while writing I might omit littles details, such as showing an explicit virtual environment, or navigating through folders. 

### Basic Example App ###
Most basic Websocket application is a Chat Room. 
Basic features required:

+ Beign able to send messages to a chat room
+ Receive updates from a room

### Example Architecture
+ Django backend 
+ Django channels as websocket layer
+ Angular app for client logic. 

# DEVELOPMENT 

### NOTES
Versions updates
There might be conflicts between dependencies versions. 
Currently using
python 3.6
Django 3.0.5
Channels 2.4.0

Due to versions inconsistencies, as channels beign documented for django 1. X, there are some imports that could be updated to the newer Django version. 

### Backend
#### Create Django Project

Create virtualenvironment. 
Install django
```
django-admin startproject backend
python manage.py migrate
```
Just for the sake of easy of mind:
```
python manage.py runserver
```

#### Install Redis server
A redis server is needed to be used as a Channel layer sistem. 
You can install redis using your favorite procedure. 
Channels recommends using a Docker (veery easy). 
The important thing is to have it running at the right port!

```docker run -p 6379:6379 -d redis:5```

In case you dont use docker, be sure you have the redis server up and running. 
```
redis-cli
> PING
PONG
```
If it didnt work out, you may have to use
```sudo service redis start```
#### Install and configure django-channels ####

https://channels.readthedocs.io/en/latest/installation.html

##### Summary #####

###### Installation ######
```
pip install django-channels
```
Add to your ```INSTALLED_APPS``` in ```settings.py```

###### Basic routing
Create a ```routing.py``` file in you project folder (alongside ```manage.py```)
```python
from channels.routing import ProtocolTypeRouter

application = ProtocolTypeRouter({
    # Empty for now (http->django views is added by default)
})
```
###### Enable ASGI routing
Add the following to ```settings.py```
```python
ASGI_APPLICATION = "backend.routing.application"
```
#### Chat application ####
Create app and delete unneeded files. Just keep views.py

##### ```consumer.py``` (the handlers)#####
Create ```consumer.py``` alonside your app's ```views.py```.  The consumer implements simple methods to create the connection between the client and the server. Once the conection is made, every message will be handled in the consumer. 

The next consumer just echoes every message that a user send. 

```
import json
from channels.generic.websocket import WebsocketConsumer

class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.accept()

    def disconnect(self, close_code):
        pass

    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        self.send(text_data=json.dumps({
            'message': message
        })
```
##### app-level ```routing.py``` (the dispatchers)#####
Routing is the simil to urls.py. Here we provide the urls that will be handled by the consumers. 
Create it alongside your ```consumers.py```

Note that since our consumer accepts every connection atempt, a user will be able to join to any chat room. 

```
from django.urls import re_path

from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<room_name>\w+)/$', consumers.ChatConsumer),
]
```
##### Modify module ```routing.py``` (root routing)
This is the one next to ```settings.py```
```python
from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
import chat.routing

application = ProtocolTypeRouter({
    # (http->django views is added by default)
    'websocket': AuthMiddlewareStack(
        URLRouter(
            chat.routing.websocket_urlpatterns
        )
    ),
})
```

##### Channel Layer #####
From channels documentations, the channel layer is a kind of communication system.

	A channel is a mailbox where messages can be sent to. Each channel has a name. Anyone who has the name of a channel can send a message to the channel. A group is a group of related channels. A group has a name. Anyone who has the name of a group can add/remove a channel to the group by name and send a message to all channels in the group. It is not possible to enumerate what channels are in a particular group. In our chat application we want to have multiple instances of ChatConsumer in the same room communicate with each other. To do that we will have each ChatConsumer add its channel to a group whose name is based on the room name. That will allow ChatConsumers to transmit messages to all other ChatConsumers in the same room.

To use a Channel Layer, install ```channels-redis``` and then add the following to ```settings.py```

```
ASGI_APPLICATION = 'backend.routing.application'
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            "hosts": [('127.0.0.1', 6379)],
        },
    },
}
```

##### A better ```ChatConsumer.py``` #####
```
import json
from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer

class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = 'chat_%s' % self.room_name
```
        # Join room group
        async_to_sync(self.channel_layer.group_add)(
            self.room_group_name,
            self.channel_name
        )

        self.accept()

    def disconnect(self, close_code):
        # Leave room group
        async_to_sync(self.channel_layer.group_discard)(
            self.room_group_name,
            self.channel_name
        )

    # Receive message from WebSocket
    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        # Send message to room group
        async_to_sync(self.channel_layer.group_send)(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message
            }
        )

    # Receive message from room group
    def chat_message(self, event):
        message = event['message']

        # Send message to WebSocket
        self.send(text_data=json.dumps({
            'message': message
        }))
