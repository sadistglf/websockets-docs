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
ASGI_APPLICATION = "myproject.routing.application"
```


