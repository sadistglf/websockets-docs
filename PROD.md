## Introduccion
En este doc se anotaran los pasos necesarios para llevar este proyecto a un entorno en la nube accesible por distintos usuarios. 

## Pasos previos
Se asumira que se cuenta con una instancia cloud en algun proveedor y que tambien se tiene un dominio.  Se asumira que el dominio esta configurado con DNS para poder apuntar a la IP de la maquia virtual. 

Se asumira que se cuenta con el desarrollo disponible en repositorios. 

## Componentes claves
En general, se levantaran 5 servicios:
1. nginx (webserver; redirecciona todo el trafico)
2. daphne (websocket layer; backend, recibe el trafico de websockets)
3. gunicorn (http layer; backend, recibe el trafico a traves de http)
4. redis (channel layer; backend, mantiene la capa de consumers)
5. base de datos (opcional: Mantiene todo el backend, por defecto es SQLite, pero podria usarse algo mas robusto como MySQL o Postgresql)

Aca se detalla 1, 2, y 3. Para 4 se puede revisar el tutorial de Django-channels y montar Redis usando Docker (mucho mas facil). 
Para dejar andando todos los servicios se puede usar un script de startup, pero aca de detalla el proceso manual para el cual implicitamente se usa ```screen``` para poder mantener varios terminales en una misma conexion. 

## Front-end 
### Prod-like V1: Servir el entorno de desarrollo: ```ng-serve```
##### Ingresar a la maquina virtual
Ingresar a la maquina virtual. Metodo preferido es por ssh. El proveedor del servicio deberia tener un recordatorio de como ingresar en caso de dudas. 

Instalar/upgrade de node js

```
sudo npm cache clean -f
sudo npm install -g n
sudo n stable
```
Upgrade de version de angular. Quizas no es necesario ya que uno dsps hace npm install de todas formas en el proyecto. 

```
npm uninstall -g angular-cli
npm install -g @angular/cli@latest
```
Clonar proyecto

```
git clone url-a-repo
```
Instalar node modelues
```
npm install
```
Si la instancia en la nube tiene el puerto 4200 abierto, se puede probar que todo funcione usando el siguiente comando. 

```
ng serve --host 0.0.0.0 --port 4200 --disableHostCheck
```

### Prod-like V2: Deploy to webserver ```ng-build``` ###
 Este metodo permite que no tengas que instalar angular ni node en el servidor. En general el procedimiento es hacer un build del proyecto en una rama de desarrollo-server en un entorno local. Esta rama debe apuntar a los servidores de backend (i.e modificar API_URL  y similares).  Luego, se copian los archivos del build (normalmente en carpeta /dist en el proyecto angular) a un webserver, y se configura el webserver para que redirija los request a dicha carpeta. 
##### build local
```
(local)$ ng build --prod 
```

##### scp transfer to webserver
```
scp -i PATH/TO/KEY.pem -r PATH/TO/LOCAL/dist/ user@host.domain:~/PATH/TO/SERVER/dist/
``` 

##### Serving the dist folder
Se puede escoger cualquier webserver. Yo prefiero ```nginx```.  Mas adelante se mostrara como servir ```django``` y ```angular``` desde el mismo webserver pero por ahora se deja un ```nginx.conf``` sencillo solo para angular.  OJO! Los path deben ser modificados segun sea necesario. Si los archivos de logs no existen, deben ser creados. Por ejemplo
```
$ cd /home/user/log/
$ touch nginx.access.log
$ touch nginx.error.log
```

```
worker_processes 1;

user nobody nogroup;
# 'user nobody nobody;' for systems with 'nobody' as a group instead
pid /tmp/nginx.pid;
error_log /home/user/log/nginx.error.log; 

events {
  worker_connections 1024; # increase if you have lots of clients
  accept_mutex off; 
}

http {
  include /etc/nginx/mime.types;
  # fallback in case we can't determine a type
  default_type application/octet-stream;
  access_log /home/user/log/nginx.access.log combined;
  sendfile on;

  server {
    # if no Host match, close the connection to prevent host spoofing
    listen 80 default_server;
    return 444;
  }

  server {
    listen 80;

    # set the correct host(s) for your site
    # this is useful if someone else is pointing your server!
    # DNS whould be your A registry in your DNS/DOMAIN provider. 
    server_name DNS.DOMAIN.com; 

    keepalive_timeout 5;
       
    # ANGULAR STUFF GOES HERE
    root /PATH/TO/dist/client; 

    # INDEX LOCATION
    location / {
      try_files $uri /index.html;
    }
     
  }
}

```
## Backend
Este proyecto usa python 3.6, hay que instalarlo en el servidor. 
```
$ sudo add-apt-repository ppa:deadsnakes/ppa
$ sudo apt update
$ sudo apt install python3.6
```
Hay que crear un nuevo entorno virtual e instalar las dependencias. 
Hay que instalar redis para la comunicacion con channel layer. 

##### Configurar nginx.conf
```
worker_processes 1;

user nobody nogroup;
# 'user nobody nobody;' for systems with 'nobody' as a group instead
pid /tmp/nginx.pid;
error_log /home/ubuntu/log/nginx.error.log; # error_log /home/sadistglf/log/nginx.error.log;

events {
  worker_connections 1024; # increase if you have lots of clients
  accept_mutex off; 
}

http {
  include /etc/nginx/mime.types;
  # fallback in case we can't determine a type
  default_type application/octet-stream;
  access_log /home/ubuntu/log/nginx.access.log combined; 
  sendfile on;

  ## A NORMAL DJANGO - GUNICORN APP PORT
  upstream app_server {
    server 127.0.0.1:8000 fail_timeout=0;
  }

  server {
    # if no Host match, close the connection to prevent host spoofing
    listen 80 default_server;
    return 444;
  }

  server {
    listen 80;

    # set the correct host(s) for your site
    # this is useful if someone else is pointing your server!
    server_name DNS.DOMAIN.COM; 

    keepalive_timeout 5;
       
    # ANGULAR STUFF GOES HERE
    root /PATH/TO/dist/client; 

    # INDEX LOCATION
    location / {
      # checks for static file, if not found proxy to app
      # try_files $uri @proxy_to_app; # de esta forma haciamos escuchar a django
      try_files $uri /index.html;
    }
   
    # DJANGO URL PREFIX (IF NEEDED)
    location /apirest/ {
      try_files $uri @proxy_to_app;
    }

    # DJANGO STATIC FILES LOCATION (point to your django static folder)
    location /static {
      alias /PATH/TO/DJANGO/static; 
     }

    location @proxy_to_app {
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
      proxy_redirect off;
      proxy_pass http://app_server;
    }
  }
}

```
##### backend settings changes
Add host to allowed hosts. 
Add nginx location to urls.py root (refer to nginx.conf django location setting) 
```
# nginx.cof
# DJANGO URL PREFIX (IF NEEDED)
    location /apirest/ {
      try_files $uri @proxy_to_app;
    }
```
Depende del proyecto en si y sus url. 
```    
# django root url settings
urlpatterns = [
    url(r'^apirest/admin/', admin.site.urls),
    url(r'^apirest/api/', include('app.api.urls')),
]

```
##### backend static files
add to settings.py
```
# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/2.1/howto/static-files/

STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```
Collect static in server
```
(server) $ python manage.py collectstatic
```
add static folder to nginx.conf
```
# DJANGO STATIC FILES LOCATION
    location /static {
      alias /PATH/TO/static; 
     }

```
##### serving wsgi (http -> gunicorn)
In this case, we cd to our project root folder (next to manage.py) and run. Be sure to look for the real name of the ```.wsgi``` file next to your ```settings.py``` file. 
```
gunicorn app.wsgi
```
and that's everything. You now should be able to navigate a little in the django views, and trying to enter a room but wont be able to make a WS connection.  
NOTE: Its not necessary to create any new file for this. 

##### serving asgi (ws -> daphne)
In this case we need to create a new file called ```asgi.py``` next to ```wsgi.py```
```
import os
import django
from channels.routing import get_default_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "dixit.settings")
django.setup()
application = get_default_application()
```
cd to root folder (next to ```manage.py```) and run 
```
daphne -p 8001 myproject.asgi:application

```
update ```nginx.conf``` to redirect to daphne.
```
# BLOQUE PARA WEBSOCKET USANDO DAPHNE
    location /ws/ {
      proxy_pass http://0.0.0.0:8001;
      proxy_http_version 1.1;

      proxy_read_timeout 86400;
      proxy_redirect off;
      
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarder-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Host $server_name;
    }
``` 
Note: Nginx will redirect traffic after coming through ```ws://DNS.DOMAIN.COM/ws/...```
Be sure to have just that in ```routing.py```
```
websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<room_name>\w+)/$', consumers.ChatConsumer),
]
```
And in the client
```
WSURL = 'ws://DNS.DOMAIN.com/ws/chat/';
```

#### Continuos deployment  ####

##### Frontend

```
$ git checkout master
$ nano somefile.ts  # make some changes
$ git commit -a -m "updated client"
$ git checkout server
$ git merge master
$ sudo ng build 
$ scp dist/ destination-remote-folder
```

##### backend
Similar approach.
```
$ git checkout master
$ nano somefile.py  # make some changes
$ git commit -a -m "updated client"
$ git checkout server
$ git merge master
(remote)$ git checkout server
(remote)$ git pull 
$ gunicorn dixit.wsgi
```
