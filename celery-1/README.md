## Repositorio a descargar

[Repositorio de referencia](https://github.com/silabuzinc/celery-first-steps)

[Experimentos a revisar en repositorio](https://github.com/silabuzinc/celery-first-steps/blob/main/docs/experimentos.org)

## Integrando Celery

Para realizar la primera instalación, dentro de nuestro entorno virtual, ejecutamos lo siguiente.

```shell
pip install celery
```

Si intentamos ejecutar celery, con:

```shell
python -m celery worker
```

Debería retornar un error, esto se debe a que necesita un "message broker". Para este caso haremos uso de Redis, para instalarlo dentro de sus máquinas, utilicen la siguiente documentación [Doc Redis](https://redis.io/docs/getting-started/installation/).

Luego de realizar la instalación, ejecutamos en nuestro terminal WSL:

```py
redis-server
```

> Redis va a funcionar de forma independiente a nuestro proyecto, por lo cual es necesario que mantengan abierta la ventana del terminal donde se encuentra ejecutando.

Esto nos debe retornar un mensaje en ASCII.

![Redis](https://photos.silabuz.com/uploads/big/315ad0963397269a04a74b79514d7378.PNG)

Luego, abrimos otra ventana del terminal de WSL o Ubuntu y ejecutamos el siguiente comando:

```shell
redis-cli
```

Esto nos hará ingresar al CLI de redis, en el cual probaremos si nuestra conexión funciona de forma correcta.

Dentro del CLI, ingresamos `ping` y deberíamos obtener el siguiente comportamiento.

```shell
127.0.0.1:6379> ping
PONG
127.0.0.1:6379>
```

Después de haber ejecutado el CLI de Redis, mandamos la palabra `ping` hacia el servidor de Redis, el cual nos responde automáticamente con un `PONG`, si obtuvimos esta respuesta nuestra instalación de Redis a sido correcta y Celery está habilitado de comunicarse con Redis.

Posterior a esto, podemos cerrar el CLI ejecutando `CTRL + C` y podemos cerrar la ventana del terminal donde está el CLI, el servidor lo dejamos abierto para realizar la comunicación con nuestra aplicación de Django.

Dentro de nuestro terminal donde tenemos nuestro entorno virtual ejecutándose, ingresamos.

```shell
python -m pip install redis
```

Con este comando damos a Python una interfaz para conectarse con Redis.

Luego de completar ambas instalaciones, tenemos configurado el "message broker". Incluso Celery funciona de forma correcta, pero nuestro proyecto todavía no está configurado. ¿Cómo comprobamos esto?

Ejecutamos el siguiente comando dentro de nuestro entorno virtual, donde `djangoAPP`, es la carpeta principal donde iniciamos nuestro proyecto de Django:

> No confundir con las aplicaciones que creamos.

```shell
> python -m celery -A djangoApp worker

Usage: python -m celery [OPTIONS] COMMAND [ARGS]...
Try 'python -m celery --help' for help.

Error: Invalid value for '-A' / '--app': 
Unable to load celery application.
Module 'djangoApp' has no attribute 'celery'
```

[REPOSITORIO](https://github.com/silabuzinc/TallerCelery)