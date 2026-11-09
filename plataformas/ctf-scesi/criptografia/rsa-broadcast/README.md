# Write-up: RSA Broadcast / Håstad

## Reto

- Plataforma: scesi ctf

- Categoría: Crypto

- Dificultad: Medium

- Fecha: 2026-09-01

- Estado: `resuelto`

- Tags: #crypto #rsa #hastad #broadcast #crt


## Resumen

Se interceptaron tres cifrados RSA del mismo mensaje usando módulos distintos, el mismo exponente `e=3` y sin padding. Esto permite aplicar el ataque de Håstad.

## Reconocimiento

La descripción indicaba explícitamente que se envió **el mismo comunicado** a tres nodos distintos. Los datos tenían la forma `(n1,c1)`, `(n2,c2)`, `(n3,c3)` y `e=3`. El patrón clave es: mismo plaintext `m`, varios módulos `n`, mismo exponente pequeño y ausencia de padding.

## Explotación / Procedimiento

RSA calcula `c = m^e mod n`, por lo que aquí tenemos `c1=m³ mod n1`, `c2=m³ mod n2` y `c3=m³ mod n3`. Primero se verifica que los módulos sean coprimos con `gcd(ni,nj)=1`. Después se utiliza CRT para reconstruir un único valor `x` que cumple las tres congruencias. Como el mensaje es suficientemente pequeño, `x=m³` exactamente, por lo que basta calcular la raíz cúbica entera.\
Con el toolkit:\
`python3 rsa_broadcast.py`\
Configuración:

```python
E = 3
PAIRS = [
    (n1, c1),
    (n2, c2),
    (n3, c3),
]
```

Resultado recuperado:\
`umssctf{tres_mensajes_una_sola_raiz}`

## Flag

`umssctf{tres_mensajes_una_sola_raiz}`

## Notas adicionales

`e=3` por sí solo NO significa Håstad. Deben existir varios cifrados del mismo mensaje bajo módulos diferentes. Si solo hay un `(n,c)` con `e` pequeño, podría tratarse de un ataque small-e; si el módulo es el mismo pero cambian los exponentes, sería Common Modulus.

## Qué debo recordar para otro CTF

- Håstad requiere el mismo mensaje, varios módulos coprimos y el mismo exponente pequeño sin padding seguro.
- El ataque no se decide únicamente porque `e = 3`.

## Aporte para el arsenal

- Clasificar correctamente Håstad, small-e y common modulus según los datos disponibles.
- Verificar coprimalidad y exactitud de la raíz antes de mostrar plaintext.
