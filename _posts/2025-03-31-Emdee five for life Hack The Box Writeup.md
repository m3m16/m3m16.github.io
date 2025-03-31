---
layout: single
title: Emdee five for life - Hack The Box
excerpt: "Emdee Five For Life es un reto sencillo que requiere que el jugador lea una cadena, la cifre con MD5 y la envíe de vuelta a la instancia remota mediante un script."
date: 2025-03-31
classes: wide
header:
  teaser: /assets/images/CAP.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - infosec
tags:  
  - python
  - web
  - scripting
  - linux
  - web requests
---

![foto3.png](/assets/images/Emdee/foto3.png)

--------
# ENUMERACIÓN

- Como podemos ver este `Challenge` solo es accesible a través de una instancia remota así que echemos un vistazo:
![foto1.png](/assets/images/Emdee/foto1.png)

- Nos esta devolviendo una `string` y nos quiere decir que la encriptemos y se la pasemos por el cuadro de texto, lo que pasa que cuando hacemos eso nos devuelve el siguiente error:
![foto2.png](/assets/images/Emdee/foto2.png)

-------------
# SOLUCIÓN

- Tenemos que enviar una petición GET para leer la cadena que se nos pide cifrar a MD5.
- Encriptar dicha cadena.
- Enviar una petición POST con la solución.
- Obtener el resultado.

-----------
## PASO 1: IMPORTAR LIBRERIAS

```python
#!/usr/bin/env python3

import requests
import hashlib
import re
```

- **`requests`**: Para hacer peticiones HTTP (`GET` y `POST`) a la web.
    
- **`hashlib`**: Para calcular el **hash MD5** de la cadena que nos da la web.
    
- **`re`**: Para buscar patrones en el HTML con **expresiones regulares (regex)**.

------------
## PASO 2: CREAR UNA SESION HTTP

```python
# URL de la web
url = "http://IP:PORT"

# Iniciar sesión HTTP
s = requests.Session()
```

- Crea una sesión persistente, lo que permite reutilizar cookies y optimizar conexiones HTTP.

----------------
## PASO 3: HACER UNA PETICION GET

```python
# Obtener la página
rget = s.get(url)
```

- Envía una solicitud `GET` a la URL para obtener el código fuente de la web.
    
- `rget.text` contiene el **HTML** de la página en formato texto.
    
- `rget.content` es lo mismo, pero en **bytes** (útil para manipular datos binarios).

--------------
## PASO 4: EXTRAER LA CADENA

```python
# Buscar la cadena dentro del HTML
encode_this = rget.content.split(b"<h3 align='center'>")[1].split(b'<')[0]
```

- Busca el texto dentro de `<h3 align='center'>...</h3>`.
    
- `split(b"<h3 align='center'>")[1]` → Divide el HTML y toma la parte después de `<h3 align='center'>`.
    
- `split(b'<')[0]` → Luego, corta el texto justo antes del siguiente `<` (el fin del `<h3>`).
    
- `encode_this` ahora contiene **la cadena que debemos encriptar**.

-------------
## PASO 5: CALCULAR EL HASH MD5

```python
# Generar el hash MD5
res = hashlib.md5(encode_this).hexdigest()
```

- **Convierte la cadena a MD5** usando `hashlib.md5()`.
    
- `.hexdigest()` devuelve el hash en formato hexadecimal.

------------
## PASO 6: ENVIAR EL HASH AL SERVER

```python
# Enviar el hash mediante POST
rpost = s.post(url, data={'hash': res})
```

- Hace una petición `POST` a la misma URL.
    
- Envía el hash en un formulario con `data={'hash': res}`.
    
- El servidor lo valida y responde con el resultado.

-----------
## PASO 7: PRINTEO DE FLAG POR PANTALLA

```python
# Mostrar la respuesta del servidor
match = re.search(r"<p align='center'>(HTB\{.*?\})</p>", rpost.text)
if match:
    print(f"🚩 Bandera encontrada: {match.group(1)}")
else:
    print("No se encontró la bandera en la respuesta del servidor.")
```

- Usa una **expresión regular** para buscar el texto dentro de `<p align='center'>...</p>`.
    
- `HTB\{.*?\}` busca cualquier texto que empiece con `HTB{` y termine con `}`.
    
- Si la bandera está en la respuesta, `match.group(1)` contendrá el resultado.

------------
# SCRIPT

```python
#!/usr/bin/env python3

import requests
import hashlib
import re

# URL de la web
url = "http://IP:PORT"

# Iniciar sesión HTTP
s = requests.Session()

# Obtener la página
rget = s.get(url)

# Buscar la cadena dentro del HTML
encode_this = rget.content.split(b"<h3 align='center'>")[1].split(b'<')[0]

# Generar el hash MD5
res = hashlib.md5(encode_this).hexdigest()

# Enviar el hash mediante POST
rpost = s.post(url, data={'hash': res})

# Mostrar la respuesta del servidor
match = re.search(r"<p align='center'>(HTB\{.*?\})</p>", rpost.text)
if match:
    print(f"🚩 Bandera encontrada: {match.group(1)}")
else:
    print("No se encontró la bandera en la respuesta del servidor.")
```
