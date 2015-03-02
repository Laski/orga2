# Sistema de prueba de MASCHE

Esta VM contiene todo lo necesario para recrear el ejemplo de cómo utilizar
MASCHE que se encuentra en el informe.

Se explican a continuación los pasos para hacerlo:

## Heartbleed linkeado dinámicamente

Se instaló una versión vieja de OpenSSL utilizando los paquetes que se
encuentran en $HOME/paquetes-vulnerables. De este modo, cualquier programa que
haya sido linkeado dinámicamente con OpenSSL y utilice TLS será vulnerable a
Heartbleed.

Para demostrar esto se instaló la versión más resiente de nginx encontrada en
los repositorios oficiales de Ubuntu 14.04, y se la configuro para que escuche
únicamente el puerto 8000 y sirviendo una sitio web mediante HTTPS en el mismo.

Para comprobar el funcionamiento del servidor puede usarse curl con la opción
-k, la cual es necesaria porque se esta utilizando un certificado SSL
self-signed.

```
$ curl https://localhost:8000 -k
<!DOCTYPE html>
<html>
<head>
<title>Masche demo</title>
</head>
<body>
I'm a dynamically linked nginx vulnerable to heartbleed &lt;3
</body>
</html>
```

Para ver que efectivamente es vulnerable a Heartbleed puede utilizarse el script
$HOME/heartbleedtest.py.

```
$ python $HOME/heartbleedtest.py localhost -p 8000
0 hosts done
2015-03-01 17:02:12 localhost            Vulnerable
------------ summary -----------
1       Total
1       Vulnerable
```

Teniendo todo en funcionamiento, se puede utilizar MASCHE para encontrar los
procesos vulnerables con un pequeño programa de ejemplo creado para esto, que
recibe como parámetro una expresión regular de las librerías que se están
buscando, y retorna los procesos y librerías de estos que coinciden.

```
$ sudo -E go run $GOPATH/src/github.com/mozilla/masche/examples/listlibs.go -r libssl
Processes matching: libssl
[1087] /usr/sbin/nginx
        /lib/x86_64-linux-gnu/libssl.so.1.0.0
[1088] /usr/sbin/nginx
        /lib/x86_64-linux-gnu/libssl.so.1.0.0
```

Notar que se requiere correr este comando con sudo, ya que MASCHE accede a
información de otros procesos, lo cual no esta permitido para usuarios
normales. El argumento -E de sudo se utiliza porque así go lo requiere.

## Heartbleed linkeado estáticamente

Para la versión de Heartbleed con OpenSSL linkeado estáticamente se compiló
nginx con OpenSSL 1.0.1f, la última versión vulnerable. Se encuentra en
$HOME/static-nginx/build-nginx.sh un script que se utilizo para compilarlo.
Notar que ejecutar este script simplemente compila los binarios necesarios,
pero hace falta configurar los mismos.

Para esto, configuró también esta versión de nginx, que se encuentra en
$HOME/static-nginx/nginx para que sirva una web vulnerable, pero esta vez en el
puerto 8080, como se puede comprobar utilizando los mismos scripts luego de
correrlo.

```
$ cd $HOME/static-nginx/nginx
$ ./nginx

$ curl https://localhost:8080 -k
<!DOCTYPE html>
<html>
<head>
<title>Masche demo</title>
</head>
<body>
I'm statically linked ngninx vulnerable to heartbleed &lt;3
</body>
</html>

$ python $HOME/heartbleedtest.py localhost -p 8080
0 hosts done
2015-03-01 17:24:46 localhost            Vulnerable
------------ summary -----------
1       Total
1       Vulnerable
```

Para detectar este proceso usando Memsearch se creó una firma de como se
explicó en el informe, y se guardo en hexadecimal el código maquina de la
función desensamblada, como lo muestra gdb, en $HOME/heartbleed-firm.

Finalmente, podemos utilizar uno de los programas de ejemplo de MASCHE para
poder detectar la vulnerabilidad en el proceso.

```
$ sudo -E go run $GOPATH/src/github.com/mozilla/masche/examples/memsearch.go -action="file-search" -fileneedle=$HOME/heartbleed-firm -pid 15571
2015/03/01 17:35:54 Found in address: 4b61f0
```

Notar que para esto es necesario conocer el PID del proceso, pero en un caso
real basta con iterar todos.

### Otros ejemplos

En la carpeta $GOPATH/src/github.com/mozilla/masche/example se encuentran tres
programas que permiten realizar distintas pruebas con MASCHE.
