# Write-up: Cripto del bueno

## Reto

- Plataforma: CTIC
- Categoría: Crypto
- Dificultad: Medium
- Fecha: 2026-09-01
- Estado: `resuelto`
- Tags: #crypto #brainfuck #caesar #base32 #rot47

## Resumen

El archivo contiene inicialmente código Brainfuck. Su salida parece Base32, pero decodificarla inmediatamente produce basura. La solución requiere una transformación César previa y finalmente ROT47.

## Reconocimiento

El archivo contiene únicamente instrucciones típicas de Brainfuck: `+ - < > [ ] .`. Al ejecutarlo se obtiene:\
`JCFHNPVHJEDXTSCZJX======`\
La cadena cumple formalmente el alfabeto Base32, pero Base32 directo genera bytes binarios sin sentido, indicando que falta una transformación.

## Explotación / Procedimiento

1. Ejecutar Brainfuck:\
    `JCFHNPVHJEDXTSCZJX======`
2. Aplicar Caesar `-3`:\
    `GZCEKMSEGBAUQPZWGU======`
3. Base32 decode:\
    `6DE2D0AH?65`
4. La salida contiene letras, números y símbolos imprimibles, haciendo ROT47 un candidato fuerte. Aplicar ROT47:\
    `estas_pwned`\
    Ruta completa:\
    `Brainfuck -> Caesar -3 -> Base32 -> ROT47`

## Flag

`estas_pwned`

## Notas adicionales

Este reto demuestra que una cadena puede ser Base32 válida sin que Base32 sea necesariamente la siguiente capa correcta. Si una decodificación produce basura, conviene probar transformaciones baratas como Caesar, reverse, Atbash o ROT antes de descartarla.

## Qué debo recordar para otro CTF

- Aceptar formalmente una codificación no demuestra que sea la capa correcta.
- La salida debería aumentar la estructura o producir otra pista reconocible.

## Aporte para el arsenal

- Explorar transformaciones baratas con profundidad y número de estados limitados.
- Evitar ciclos y mostrar por qué una ruta recibió mejor puntuación.
