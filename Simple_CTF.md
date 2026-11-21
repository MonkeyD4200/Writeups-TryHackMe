# TryHackMe: Simple CTF

**Dificultad:** Fácil
**Sistema Operativo:** Linux (Ubuntu)
**Herramientas usadas:** Nmap, lftp, Hydra, GTFOBins

## Resumen

Acceso vía FTP anónimo para obtener una pista, fuerza bruta sobre SSH (puerto no estándar) para conseguir credenciales válidas, y escalada a `root` explotando un binario (`vim`) con permisos de `sudo` mal configurados, usando la técnica documentada en GTFOBins.

## 1. Reconocimiento

Se realizó un escaneo profundo con Nmap para descubrir todos los puertos abiertos:

```bash
nmap -sV -sC -p- -T4 <IP>
```

**Puertos clave descubiertos:**
- **21 (FTP)** — vsftpd 3.0.3 (permitía login anónimo).
- **80 (HTTP)** — Apache.
- **2222 (SSH)** — OpenSSH (puerto no estándar).

## 2. Acceso Inicial (FTP y SSH)

Se accedió al servidor FTP utilizando el usuario `anonymous`. Tras solucionar problemas de conexión (usando `lftp` en modo activo), se descargó el archivo `ForMitch.txt`.

El archivo contenía una pista: la contraseña del sistema era muy débil. Se procedió a realizar fuerza bruta sobre el servicio SSH (puerto 2222) para el usuario `mitch` con Hydra.

Tras encontrar la contraseña, se accedió al sistema:

```bash
ssh mitch@<IP> -p 2222
```

- **Bandera de usuario** (`user.txt`): `G00d j0b, keep up!`

## 3. Escalada de Privilegios

Estando dentro del sistema como `mitch`, se verificaron los privilegios de sudo:

```bash
sudo -l
```

El resultado mostró que el usuario podía ejecutar `/usr/bin/vim` como `root` sin contraseña. Se utilizó la técnica documentada en **GTFOBins** para obtener una shell a través de Vim:

```bash
sudo vim -c ':!/bin/sh'
```

Una vez dentro de la shell como `root`, se navegó al directorio protegido y se capturó la bandera final:

```bash
cd /root
cat root.txt
```

- **Bandera de root:** `W3ll d0n3. You made it!`

## Lecciones aprendidas

- Un servicio FTP con login anónimo habilitado puede filtrar archivos y pistas que no deberían ser públicas.
- Las contraseñas débiles siguen siendo uno de los vectores de acceso inicial más comunes, incluso detrás de un puerto SSH no estándar.
- Otorgar permisos de `sudo` sobre binarios como `vim`, `less`, `find` o `awk` (ver GTFOBins) sin restricciones es equivalente a dar acceso root directo.
