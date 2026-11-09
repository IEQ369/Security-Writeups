# Write-up: Cazafantasmas

## Reto

- Plataforma: CTIC
- Categoría: Crypto
- Dificultad: Easy
- Fecha: 2026-09-01
- Estado: `resuelto`
- Tags: #crypto #audio #morse #rot13 #hex

## Resumen

Se entrega un WAV acompañado de pistas sobre voces “al derecho y al revés” y un “viernes 13”. La solución combina transformación del audio, Morse, ROT13 y hexadecimal.

## Reconocimiento

Al escuchar/analizar el audio se identifican tonos compatibles con Morse. La frase “al derecho y al revés” indica invertir físicamente el audio, mientras que “viernes 13” sugiere ROT13.

## Explotación / Procedimiento

1. Abrir `Cazafantasmas.wav` en Audacity.
2. Aplicar reverse al audio y exportarlo.
3. Decodificar el audio invertido como Morse. Resultado:\
    `756R345S6964345S795S756R345S7675656P7434`
4. Aplicar ROT13:\
    `756E345F6964345F795F756E345F7675656C7434`
5. La nueva cadena solo utiliza caracteres hexadecimales, por lo que se aplica `From Hex`.\
    Resultado:\
    `un4_id4_y_un4_vuelt4`

## Flag

`un4_id4_y_un4_vuelt4`

## Notas adicionales

Invertir el audio antes de Morse NO equivale a invertir posteriormente el texto. Un símbolo Morse como `.-` se transforma en `-.` al invertir físicamente la señal, cambiando incluso la letra resultante. Es un patrón útil también para futuros retos de stego/forense.

## Qué debo recordar para otro CTF

- Transformar la señal y transformar el texto obtenido no son operaciones equivalentes.
- Las pistas del enunciado pueden indicar una operación y su orden.

## Aporte para el arsenal

- Analizar audio normal e invertido con límites, guardando cuál produjo cada resultado.
- Entregar texto Morse, ROT y hexadecimal al decodificador por capas.
