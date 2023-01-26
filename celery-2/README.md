## Añadiendo Celery a nuestro proyecto Django

Tenemos que conectar nuestra aplicación Django como un "message producer" para nuestra cola de tareas. Dentro de nuestra carpeta principal añadimos el archivo `celery.py`:

```sample
djangoApp/
├── __init__.py
├── asgi.py
├── celery.py         <--- Nuevo archivo añadido
├── settings.py
├── urls.py
└── wsgi.py
```

La misma documentación de Celery nos recomienda crear el siguiente módulo:

> No olvidar reemplazar todo lo que diga `djangoApp`, por el nombre de la carpeta del proyecto.

```python
# djangoApp/celery.py

import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "djangoApp.settings")
app = Celery("djangoApp")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

-   Con `setdefault()` nos aseguremos que de `os.environ` que nuestros `settings.py` de nuestro proyecto sean accesibles con la key `DJANGO_SETTINGS_MODULE`
    
-   Creamos la app de Celery, pasando como argumento el nombre del módulo principal, que vendría a ser el nombre de nuestra carpeta donde se encuentra el archivo.
    
-   Con `config_from_object`definimos que nuestro archivo de configuración de Django funcione para Celery, con el `namespace` se indica que cada configuración de variables de Celery que se quieran realizar van a tener que ser de la siguiente forma:
    

```python
CELERY_NOMBRE_DE_LA_VARIABLE
CELERY_OTRA_VARIABLE
```

Siempre se va a tener que poner el nombre del `namespace` seguido de un subguión y el nombre de la variable.

-   Con `autodiscover_tasks()`, indicamos a Celery que encuentre todas las tareas de nuestra aplicación.

### Modificando settings.py

Luego, dentro de nuestro archivo de configuraciones añadimos las siguiente líneas.

```python
# Celery settings
CELERY_BROKER_URL = "redis://localhost:6379"
CELERY_RESULT_BACKEND = "redis://localhost:6379"
```

Con esto indicamos a Celery donde tiene que enviar los mensajes y donde se tienen que almacenar.

### Últimas modificaciones

Por último, tenemos que modificar el `__init__.py` de nuestra carpeta principal, dentro de este añadimos las siguiente líneas:

```python
# django_celery/__init__.py

from .celery import app as celery_app

__all__ = ("celery_app",)
```

Cargando Celery en el inicio de nuestra aplicación Django, nos aseguramos que el decorador `@shared_task` (que usaremos luego) funcione correctamente.

### Probando nuestro setup

Para que funcione correctamente, tenemos que recordar que 3 procesos deben estar ejecutándose simultáneamente:

1.  _Producer_: Django app
    
2.  _Message Broker_: El servidor Redis
    
3.  _Consumer_: Celery app
    

Ejecutamos nuestra app de Django:

```shell
python manage.py runserver
```

En caso se haya cerrado el terminal WSL con el servidor de Redis, abrimos uno nuevo y ejecutamos:

```shell
redis-server
```

Y finalmente ejecutamos Celery:

```shell
python -m celery -A djangoApp worker
```

Si todo está correcto, deberíamos obtener el siguiente mensaje:

![Celery funciona](https://photos.silabuz.com/uploads/big/fa2157ec95c78c1534cad752c151c919.PNG)

## Probando Celery

Para hacer prueba de que Celery funciona correctamente, añadiremos algunas tareas a nuestra aplicación:

Primero crearemos un form en `myapp/forms.py` que funcionará para simular el envío del libro a un correo:

```python
class InputForm(forms.Form):

    nombre = forms.CharField(max_length = 3)
    email = forms.EmailField()
```

Añadimos el form a nuestra vista de un solo libro `select_book`:

```python
def select_book(request, id):
    book = Book.objects.filter(bookID=id)
    request.session["authors"] = book[0].authors
    request.session["id"] = book[0].bookID
    context = {}
    context["book"] = book[0]
    context["form"] = InputForm()
    if request.method == "POST":
        form = InputForm(request.POST)
        if form.is_valid():
            print(form.cleaned_data["nombre"], form.cleaned_data["email"])
            return HttpResponse(form.cleaned_data["nombre"] + " " + form.cleaned_data["email"])
    return render(request, "oneBook.html", context)
```

Y modificamos nuestro template `oneBook.html`:

```html
{% extends 'base.html' %}

{% block content %}
<ul>

<li>{{book.title}}</li> 
<li><a href="{% url 'onlyAuthor' book.bookID%}">{{book.authors}}</a></li>
<li>{{book.average_rating}}</li>
<li>{{book.isbn}}</li>

</ul>
<form method="post">
    {% csrf_token %}
    {{ form }}
    <input type="submit">
</form>
{% endblock %}
```

Con esto añadiremos el nuevo form que utilizaremos para Celery:

Ahora añadiremos el archivo de task que necesita Celery:

En `myapp`, creamos el archivo `tasks.py`:

Los archivos de `myapp` deberías quedar se la siguiente forma:

```sample
myapp/
├── migrations/
├── templates/
├── __init__.py
├── admin.py
├── apps.py        
├── forms.py
├── models.py
├── tasks.py       <--- Nuevo archivo añadido
├── tests.py
├── urls.py
└── views.py
```

Dentro de la modificación a la vista, las siguientes líneas son las que refactorizaremos para crear un task.

```python
if form.is_valid():
    print(form.cleaned_data["nombre"], form.cleaned_data["email"])
    return HttpResponse(form.cleaned_data["nombre"] + " " + form.cleaned_data["email"])
```

Creamos la nueva función en `tasks.py` e importamos `shared_task`.

```python
from time import sleep

def send_book(nombre, mail):
    sleep(20)  # Simula operaciones muy pesadas que congelan a Django
    print(
        nombre + " " + mail
    )
```

Luego añadimos el decorados a nuestra función:

```python
from time import sleep
from celery import shared_task

def send_book(nombre, mail):
    sleep(20)  # Simula operaciones muy pesadas que congelan a Django
    print(
        nombre + " " + mail
    )
```

Con esto, importamos la nueva función en nuestro archivo de vistas y cambiamos el código de validación por `send_book`.

```python
from .tasks import send_book

def select_book(request, id):
    # ...
    if request.method == "POST":
        if form.is_valid():
            send_book.delay(form.cleaned_data["nombre"], form.cleaned_data["email"])
            return HttpResponse(form.cleaned_data["nombre"] + " " + form.cleaned_data["email"])
    return render(request, "oneBook.html", context)
```

Con `.delay` es la forma más rápida de enviar un mensaje a Celery, existen otras formas como `.apply_async()`, pero hacen más verboso nuestro código.

Ahora que tenemos nuestra tarea implementada, es tiempo de probarla.

Cerramos Celery con `CTRL + C` o simplemente cerrando el terminal, y ahora ejecutamos.

```shell
python -m celery -A djangoApp worker -l info
```

Con `-l info` listaremos todas las tareas descubiertas de nuestra aplicación.

![Task](https://photos.silabuz.com/uploads/big/c9eba033a13ad459e0fdb20c69fdc90c.PNG)

Verificamos que nuestra tarea ha sido reconocida:

Si todo está correcto, al momento de hacer uso de nuestro form, deberíamos ser redireccionados y en la consola de Celery, la tarea se debería completar luego de 20 segundos. Comprobando que es una tarea asíncrona a la ejecución de nuestra aplicación.

## Opcional - eventlet

Si los eventos de Celery no se muestran realizamos la siguiente modificación:

Instalamos eventlet.

```shell
pip install eventlet
```

Modificamos las rutas de celery en `settings.py`.

```python
# Celery settings
CELERY_BROKER_URL = "redis://127.0.0.1:6379"
CELERY_RESULT_BACKEND = "redis://127.0.0.1:6379"
```

Y ejecutamos:

```shell
python -m celery -A DjangoTemplates worker --loglevel=info -P eventlet
```

## Tarea Opcional

Crear nuevas tareas para Celery

-   Crear el envío del libro con `django.core.mail` (Investigar)
    
-   Realizar la modificación de los autores de un libro.
    
    -   Crear vista, url y form para la modificación.

LINKS:

[Diapositivas](https://docs.google.com/presentation/d/e/2PACX-1vTOSVjZQdeWkegR1S_rekUpjx5x-ip2icFVj6Sa4kn-_R7YMaDBh-41DqzX9MT56fhhNVSdmCRFuGFG/embed?start=false&loop=false&delayms=3000#slide=id.g143f30675af_0_0)

Videos:

[Teoria](https://www.youtube.com/watch?v=SvrZrv3FiI4&list=PLxI5H7lUXWhjV-yCSEuJXxsDmNESrvbw3&index=15&ab_channel=Silabuz)

[Practica](https://www.youtube.com/watch?v=J-RJL-SMZOU&list=PLxI5H7lUXWhjV-yCSEuJXxsDmNESrvbw3&index=16&ab_channel=Silabuz)

[REPOSITORIO](https://github.com/silabuzinc/TallerCelerySol/tree/master)