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

### Other repos
Django back end
https://bitbucket.org/sadistglf/websockets-basic-django-server/

Front end
https://bitbucket.org/sadistglf/websockets-basic-angular-client/src/master/

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

### Frontend ###
To show the complete functionality of the server, we will use a moder Javascript Framework. Popular ones are Angular, React, Vue, among others. We will use Angular in this example. 
For this example we need a simple component to show us a text field to enter our message, and a list of the incoming messages. We will receive the incoming messages via WebSockets, but we will send new messages using HTTP (we might want to send messages without the need of beign in the room). 

#### Features
Users can access a url like ```localhost:4300/rooms/roomid``` and they will be asked to add a name to join the Room. If the room doesnt exists, we expect the backend to be able to create one. A button will act as the trigger of the WS connection attempt. If succeded it will enable another text field to send messages, and it will show a list of the incomming messages received through Websockets. 

#### Creating the angular App
Be sure to have NodeJS and an @angular/cli package. There are huge inconsistences between angular and nodeJs, so you need to be carefull. Normally latest NodeJS will work ok with latest Angular. 
In my case I had to upgrade NodeJS and angular CLI. 
```python

# INSTALL/UPDATE NODE (instructions missing)
sudo npm -g uninstall angular-cli
sudo npm -g install @angular/cli
ng new client # enable angular routing
cd client 
ng serve
```
 With all of this you should have the basic Angular app working and serving!
 
#### Adding the room component
We want to be able to enter a url ```client.server/rooms/this-is-my-room/```  and be prompted with for our user name.

##### Adding FormsModule
This one is to use it in the TextField input. It will be use to bind the value in the textfield into a variable. 
```javascript
import { FormsModule } from '@angular/forms';
@NgModule({
  declarations: [
		..
  ],
  imports: [
	..
    FormsModule
  ],
	..
})
...

```
##### Creating component and enabling routing #####

```
ng generate component room
```
Add a username and room string into component class.  To enable routing, we nedd the following. We are not sending the data to create the connection yet. 

```
// room.component.ts
import { Component, OnInit } from '@angular/core';
import { Router, ActivatedRoute, ParamMap } from '@angular/router';

@Component({
  selector: 'app-room',
  templateUrl: './room.component.html',
  styleUrls: ['./room.component.css']
})
export class RoomComponent implements OnInit {
	name: string;
	username: string = "";

  constructor(private route: ActivatedRoute) { }

  ngOnInit(): void {
  	this.name = this.route.snapshot.paramMap.get('roomId');
  }

  connect() {

  }

}

```
Add a route to your ```app-routing.ts```

```
const routes: Routes = [
	{path: 'rooms/:roomId', component: RoomComponent}
];
```

##### Creating the websocket connection #####
```javascript
  // room.component.ts
  socket: any;
  WSURL = 'ws://127.0.0.1:8000/ws/chat/'

  setsocket() {
    this.socket = new WebSocket(this.WSURL+this.name+'/');
    this.socket.onopen = () => {
      console.log("Websockets connection created!");
    }
    // what will happen on message!
    this.socket.onmessage = (event) => {
      console.log(event);
    }
    if (this.socket.readyState == WebSocket.OPEN) {
      this.socket.onopen(null);
    }
  }
```
We will call this method when the user clicks the button. We are going to stay simple here, but we could (should?) implement the websocket connection in a service, and wrap everything in observables in order to publish updates from the service to the component.  

TODO : add html, and messages list. 

### Finishing the backend ###
We need an url and a view to manage new messages published into a room. We need ```roomId```, ```username``` and ```message```.  We could have everything in a POST request body, but we better mix the solution. We prefer using a POST rather than a GET just because is more conventional. 

##### Enabling CORS #####
You will likely use your server in a separate origin from your client server. So this is needed to allow different origins requests. 
```
pip install django-cors-headers
```
Add to your installed apps
```
INSTALLED_APPS = [
   ...
    'chat',
    'corsheaders'
]
```
Add to Middleware Stack
```
MIDDLEWARE = [
    ...
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    ...
]
```
Add basic configuration
```
# CORS
CORS_ORIGIN_ALLOW_ALL = True
```
##### Adding View and URL #####
Add a new path for your new view.
```python
# chat/urls.py
path('rooms/<str:roomId>/user/<str:username>/', views.postMessage, name='post-message')
```
Update your ```views.py``` to look like the following. We have to use ```csrf_exepmt``` for Django to accept the incoming POST method without a CSRF token. Take note, this is very liberal in permissions. 

```python
from django.shortcuts import render
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync
import json
# Create your views here.

def index(request):
    return JsonResponse({'foo': 'bar'})

@csrf_exempt
def postMessage(request, roomId, username):
	if request.method == 'GET':
		return JsonResponse({'status': 'error', 'msg': "GET method not supported", data: {}})

	channel_layer = get_channel_layer()
	chat_name = 'chat_'+roomId
	## If the post is sent as a form-data we use the following line
	# message = request.POST.get('message', "No message received")
	## if the post is sent as a raw-body we need to manually parse it
	body = json.loads(request.body.decode("utf-8"))
	message = body['message']
	response = {
		"text": message,
		"username": username
	}
	async_to_sync(channel_layer.group_send)(chat_name, {"type": "chat.message", "message": json.dumps(response)})
	return JsonResponse({'username': username, 'roomId': roomId})
```
Note that we are assuming three contracts here:
1. chat room names follow the next pattern ```"chat_{}".format(roomId)```
2. Request RAW-BODY has a message parameter. 
3. Server notifies the room observers with a message with the following format
```javascript
message = {
	... // several fields
	data: MESSAGE_AS_DUMPED_JSON //real data (the one you explicitly format as Json)
	type: MESSAGE.TYPE // this one you explicitly declare in the message as well
}
```

### Finishing the Frontend ###
##### Displaying incoming messages #####
Modify your room component, adding a method to handle incoming messages. 

```javascript
// room.component.ts
setsocket() {
    ...
    // what will happen on message!
    this.socket.onmessage = (event) => {
       this.handleMessage(event);
    }
	...
  }
	...
  handleMessage(event) {
  	let eventData = JSON.parse(event['data']);
  	let messageData = JSON.parse(eventData.message);
  	this.messages.push(messageData);
  }
```

##### Posting a new message #####
To keep it simple, we wont be using a service... but we should! And we should create a ```Message``` model to be able to use a stronger typing. 

Import ```HttpClientModule``` in the ```app.module.ts``` file
```javascript
// app.module.ts
...
import { HttpClientModule }    from '@angular/common/http';

@NgModule({
 	...
  imports: [
	...
    HttpClientModule
  ],
  ...
})
export class AppModule { }

```
Modify ```room.component.ts``` with the following hightlated changes
```javascript
import { HttpClient, HttpHeaders } from '@angular/common/http';

export class RoomComponent implements OnInit {
  ...
  connected: boolean = false;
  message: string = "";
  ...
	
  API_URL = 'http://127.0.0.1:8000/chat/';
  httpOptions = {
    headers: new HttpHeaders({ 'Content-Type': 'application/json' })
  };

  constructor(... , private http: HttpClient) { }

  sendMessage(){
  	let url = this.API_URL + 'rooms/' + this.name + '/user/' + this.username + '/';
  	let message = {
  		'message': this.message
  	}
  	this.http.post(url, message, this.httpOptions).subscribe(response => {
  		console.log(response);
  		// we might want to do something in case of errors here
  	})
  }
}
```
Finally, the entire ```room.component.html``` is here. 
```html
<p> Trying to access room {{name}}</p>
<div *ngIf="!connected">
	<p> Username </p>
	<input [(ngModel)]="username" placeholder="Nombre de jugador" type="text" name="playerName">
	<button (click)="connect();">Entrar</button>
</div>

<div *ngIf="connected">
	<span> <b>{{username}}: </b></span><input [(ngModel)]="message" placeholder="Message" type="text" name="message">
	<button (click)="sendMessage();">Enviar</button>
</div>

<h5> Messages </h5>
<p *ngIf="messages.length == 0"> No messages yet </p>
<p *ngFor="let message of messages"> <b>{{message.username }}:</b> {{message.text}} </p>
```

