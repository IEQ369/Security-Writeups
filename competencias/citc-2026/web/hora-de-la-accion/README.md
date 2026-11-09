# Hora de la acción

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Web / Warmup. Puntos: 40. Autor: MauChuk.
- Objetivo histórico: `http://201.131.45.82:8125/`.
- Estado: **validado: el usuario confirmó que funcionó**.

> Practicar con unos y ceros; hacer las conversiones para hallar la bandera. La historia menciona militares aficionados a las matemáticas.

## Resumen

Cuatro conversiones desbloquean pistas. Los mensajes de éxito **y los enlaces para continuar** contienen palabras del alfabeto fonético militar. La última pista está en Base64. No hizo falta explotar una vulnerabilidad.

## Reconocimiento e hipótesis

La portada ofrece cuatro calentamientos. Conviene completarlos en la misma sesión del navegador y anotar todo el texto que aparece tras responder. La referencia a militares adquiere sentido cuando aparece `Charlie`.

## Resolución paso a paso

1. Hexadecimal `0x8E` a decimal: `8 × 16 + 14 = 142`. La respuesta aceptada es **142**. Aparecen `Charlie` y, en el enlace siguiente, `Oscar`: **CO**.
2. Decimal `619` a hexadecimal: `619 = 2 × 256 + 6 × 16 + 11`; 11 se representa como B. Enviar **26B**. `MikePapa` y `LimaEcho` forman **MPLE**.
3. Hexadecimal `26B` a binario: cada dígito aporta cuatro bits: `0010 0110 1011`. Enviar **001001101011**, sin espacios. El servidor rechazó `1001101011`, aunque representa el mismo número: su comprobación exige los ceros iniciales. `TangoAlpha` y `DeltaOscar` forman **TADO**.
4. Binario a octal: agrupar de tres en tres, `001 001 101 011` → `1 1 5 3`. Enviar **1153**.
5. El último mensaje es:

```text
X0ZveHRyb3RFY2hvTGltYUluZGlhQ2hhcmxpZUluZGlhRGVsdGFBbHBoYURlbHRhRWNob1NpZXJyYQ==
```

En CyberChef, **From Base64** devuelve:

```text
_FoxtrotEchoLimaIndiaCharlieIndiaDeltaAlphaDeltaEchoSierra
```

Tomar la inicial de cada palabra da `_FELICIDADES`. Concatenar con `CO`, `MPLE` y `TADO`. La propia página final indica el envoltorio `CITC{...}`.

## Cadena de resolución

`conversiones → pistas en mensajes y enlaces → Base64 → alfabeto fonético → bandera`

## Flag

```text
CITC{COMPLETADO_FELICIDADES}
```

## Qué debo recordar para otro CTF

- Leer los enlaces además del mensaje principal: allí estaban las letras que faltaban.
- Una conversión matemática correcta puede fallar por el formato de entrada.
- `Charlie` representa C; no es una contraseña ni una nueva capa de cifrado.

## Aporte para el arsenal

Conversión entre bases con opción de ancho fijo, Base64 y reconocimiento del alfabeto fonético. No inventar letras ausentes a partir de una frase probable.

## Archivos y evidencia

No hubo archivo descargable. Se conservaron respuestas textuales de la sesión en `evidencias/`; son registros históricos, no una copia ejecutable del servidor. No se volvió a contactar el servicio después de la competencia.
