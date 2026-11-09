# Write-up: Capas de la Cebolla

## Reto

- Plataforma: scesi ctf
- Categoría: Crypto
- Dificultad: Easy
- Fecha: 2026-09-01
- Estado: `resuelto`
- Tags: #crypto #base64 #xor #reverse #encoding

## Resumen

El reto entrega una cadena sometida a varias transformaciones consecutivas. La solución consiste en reconocer qué operación produce una salida cada vez más estructurada.

## Reconocimiento

Ciphertext:\
`XtUSDx0SY1URep1QYlUdLZkRFh0TJV3TOVXWLp1SJFFTelUWZd0X`\
La cadena parece compatible con Base64, pero decodificarla directamente produce basura. Al invertir primero la cadena, la posterior decodificación Base64 genera texto completamente imprimible, indicando que la ruta es correcta.

## Explotación / Procedimiento

1. Reverse:

```
data = data[::-1]
```

2. Base64 decode:

```
import base64
data = base64.b64decode(data)
```

Resultado intermedio:\
`_GYYI^LQIKZKYuNOuIOHEFFKuIXCZ^EMXKLCIKW`\
3. El resultado parece texto ASCII alterado. Se prueba XOR de un byte. La clave correcta es `42` (`0x2a`):

```
data = bytes(x ^ 42 for x in data)
```

Resultado:\
`umssctf{capas_de_cebolla_criptografica}`\
Ruta completa:\
`reverse -> Base64 -> XOR 0x2a`

## Flag

`umssctf{capas_de_cebolla_criptografica}`

## Notas adicionales

Que una operación decodifique sin error no significa que sea correcta. La salida buena normalmente aumenta la estructura: texto imprimible, otro encoding reconocible o una flag. Este reto motivó la creación de `crypto_triage.py`.

## Qué debo recordar para otro CTF

- Una decodificación válida puede ser un falso camino si su salida pierde estructura.
- Reverse, Base64 y XOR de un byte son transformaciones baratas que pueden explorarse con límites.

## Aporte para el arsenal

- Puntuar cada estado por imprimibilidad, codificaciones reconocibles y formatos de flag.
- Conservar la ruta exacta y evitar ciclos entre transformaciones.
