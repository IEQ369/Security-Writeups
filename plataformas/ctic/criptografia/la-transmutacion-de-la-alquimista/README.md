# Write-up: La transmutación de la alquimista

## Reto

- Plataforma: CTIC
- Categoría: Crypto
- Dificultad: Easy
- Fecha: 2026-09-01
- Estado: `resuelto`
- Tags: #crypto #hex #base64 #rot13 #swap

## Resumen

La descripción enumera cinco operaciones alquímicas que funcionan como pistas para varias transformaciones consecutivas.

## Reconocimiento

`Calcinación` menciona `16 espíritus`, apuntando a Base16/Hex. `Sublimación` menciona `13 ciclos`, apuntando a ROT13. `Putrefacción` dice que los pares intercambian sus lugares, indicando un swap por parejas.

## Explotación / Procedimiento

Ciphertext:\
`536b647353544e4d62456c4b626e4e5456556c31563246765a30464c63576c4662557471556c52615957637753327735616d383950513d3d`

1. Hex decode:\
    `SkdsSTNMbElKbnNTVUl1V2FvZ0FLcWlFbUtqUlRaYWcwS2w5am89PQ==`
2. Base64 decode:\
    `JGlI3LlIJnsSUIuWaogAKqiEmKjRTZag0Kl9jo==`
3. ROT13:\
    `WTyV3YyVWafFHVhJnbtNXdvRzXwEGMnt0Xy9wb==`
4. Intercambiar cada pareja de caracteres, ejemplo `ABCD -> BADC`:\
    `TWVyY3VyaWFfVHJhbnNtdXRvXzEwMGtnX09ybw==`\
    En CyberChef puede hacerse con `Swap endianness`, `Data format: Raw`, `Word length: 2 bytes`.
5. Base64 decode:\
    `Mercuria_Transmuto_100kg_Oro`

## Flag

`Mercuria_Transmuto_100kg_Oro`

## Notas adicionales

La transformación `swap pairs` es barata y útil para el triage general, pero no merece un script independiente. Este reto también demuestra que la descripción puede contener literalmente la receta.

## Qué debo recordar para otro CTF

- Traducir cada pista del enunciado a una operación candidata y respetar su orden.
- Intercambiar pares deja un último carácter intacto si la longitud es impar; el script debe avisarlo.

## Aporte para el arsenal

- Incluir intercambio por pares como transformación del decodificador por capas.
- Registrar parámetros y entradas exactas de cada paso.
