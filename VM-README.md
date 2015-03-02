# Sistema de prueba de MASCHE

Esta VM contiene todo lo necesario para recrear el ejemplo de cómo utilizar
MASCHE que se encuentra en el informe. Este tutorial discute los pasos
necesarios para reproducir las pruebas realizadas.

## Heartbleed linkeado dinámicamente

Se instaló la versión 1.0.1f de OpenSSL, utilizando los paquetes que se
encuentran en `$HOME/paquetes-vulnerables`. De este modo, cualquier programa
que haya sido linkeado dinámicamente con OpenSSL y utilice TLS será vulnerable
a Heartbleed.

Para demostrar esto se instaló la versión 1.4.6 de `nginx`, encontrada en los
repositorios oficiales de Ubuntu 14.04, y se la configuró para que escuche
únicamente el puerto 8000, sirviendo un sitio web mediante HTTPS.

Para comprobar el funcionamiento del servidor puede usarse `curl` con la opción
`-k`, la cual es necesaria para evitar verificar la cadena de certificados, ya
que se está utilizando un certificado SSL self-signed.

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

Para ver que efectivamente es vulnerable a Heartbleed puede utilizarse el
script `$HOME/heartbleedtest.py`.

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
recibe como parámetro una expresión regular que encuentre las librerías
relevantes, y retorna los procesos y librerías que coinciden.

```
$ sudo -E go run $GOPATH/src/github.com/mozilla/masche/examples/listlibs.go -r libssl
Processes matching: libssl
[1087] /usr/sbin/nginx
        /lib/x86_64-linux-gnu/libssl.so.1.0.0
[1088] /usr/sbin/nginx
        /lib/x86_64-linux-gnu/libssl.so.1.0.0
```

Nótese que se requiere correr este comando con permisos de superusuario, ya que
MASCHE accede a información de otros procesos, lo cual no esta permitido para
usuarios normales. El argumento `-E` de `sudo` es requerido por  `go`.

## Heartbleed linkeado estáticamente

Para la versión de Heartbleed con OpenSSL linkeado estáticamente se compiló
`nginx` con OpenSSL 1.0.1f, la última versión vulnerable. El script usado para
compilarlo se encuentra en `$HOME/static-nginx/build-nginx.sh`.  Nótese que
ejecutar este script simplemente compila los binarios necesarios, pero hace
falta configurar los mismos.

Para esto, se configuró también una copia de `nginx` compilado de este modo. La
misma se encuentra en `$HOME/static-nginx/nginx`, y sirve una web vulnerable en
el puerto 8080, como se puede comprobar utilizando los mismos scripts luego de
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
I'm a statically linked ngninx vulnerable to heartbleed &lt;3
</body>
</html>

$ python $HOME/heartbleedtest.py localhost -p 8080
0 hosts done
2015-03-01 17:24:46 localhost            Vulnerable
------------ summary -----------
1       Total
1       Vulnerable
```

Para detectar este proceso usando `Memsearch` se creó una firma como se explicó
en el informe, y se guardó en hexadecimal el código máquina de la función
desensamblada, como lo muestra `gdb`, en `$HOME/heartbleed-firm`.

Finalmente, podemos utilizar uno de los programas de ejemplo de MASCHE para
detectar la vulnerabilidad en el proceso.

```
$ sudo -E go run $GOPATH/src/github.com/mozilla/masche/examples/memsearch.go -action="file-search" -fileneedle=$HOME/heartbleed-firm -pid 15571
2015/03/01 17:35:54 Found in address: 4b61f0
```

Nótese que para esto es necesario conocer el PID del proceso, que puede
obtenerse con `$ pgrep nginx | tail -n 1`. En un caso real habría que examinar
todos los procesos del sistema.

## Otros ejemplos

En la carpeta `$GOPATH/src/github.com/mozilla/masche/example` se encuentran
tres programas que permiten realizar distintas pruebas con MASCHE.


