# TryHackMe: Pickle Rick

**Dificultad:** Fácil
**Sistema Operativo:** Linux
**Herramientas usadas:** Gobuster, navegador (código fuente), panel web vulnerable (Command Injection)

## Resumen

Explotación de una vulnerabilidad de **Command Injection** en un panel de administración web, tras obtener credenciales filtradas en el código fuente de la página y en `robots.txt`. Escalada a `root` aprovechando una configuración de `sudo` sin restricciones.

## 1. Reconocimiento (Recon)

Se identificó un servidor web Apache en el puerto 80. Al inspeccionar el código fuente de la página principal (`Ctrl + U`), se encontró un comentario HTML con un nombre de usuario oculto: `R1ckRul3s`.

Usando Gobuster se descubrió el archivo `robots.txt`:

```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
```

El archivo `robots.txt` contenía una contraseña: `Wubbalubbadubdub`.

## 2. Acceso Inicial (Command Injection)

Con las credenciales obtenidas (`R1ckRul3s` : `Wubbalubbadubdub`), se accedió al panel de administración (`/login.php`). Este panel resultó ser vulnerable a **Command Injection**, permitiendo ejecutar comandos del sistema operativo a través de un campo de texto.

Se listaron los archivos del servidor:

```bash
ls
```

Se encontró el archivo `Sup3rS3cretPickl3Ingred.txt`. Como el comando `cat` estaba bloqueado por el sistema, se utilizó un bypass con `less`:

```bash
less Sup3rS3cretPickl3Ingred.txt
```

- **Primer ingrediente:** `mr. meeseek hair`

## 3. Post-Explotación y Escalada de Privilegios

Se leyó el archivo `clue.txt`, que indicaba revisar los directorios de usuario. Navegando a la carpeta de Rick:

```bash
ls /home/rick
less "/home/rick/second ingredients"
```

- **Segundo ingrediente:** `1 jerry tear`

Para obtener acceso como `root`, se verificaron los privilegios de `sudo`:

```bash
sudo -l
```

El servidor estaba mal configurado y permitía al usuario actual (`www-data`) ejecutar cualquier comando como root sin contraseña. Se procedió a leer la carpeta protegida:

```bash
sudo ls /root
sudo less /root/3rd.txt
```

- **Tercer ingrediente:** `fleeb juice`

## Lecciones aprendidas

- El código fuente y archivos como `robots.txt` pueden filtrar credenciales sensibles: nunca deben contener información secreta.
- Un panel de administración que ejecuta comandos del sistema sin sanitizar la entrada es una puerta directa a Command Injection.
- Una mala configuración de `sudo` (permitir todos los comandos sin contraseña) anula cualquier otra medida de seguridad y lleva al compromiso total del sistema.
