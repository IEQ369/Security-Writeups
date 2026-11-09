# Write-up: TriTriTri

## Reto

- Plataforma: CTIC

- Categoría: Crypto

- Dificultad: Medium

- Fecha: 2026-09-01

- Estado: `resuelto`

- Tags: #crypto #trifid #caesar #railfence #md5


## Resumen

El reto entrega un ciphertext y repite constantemente referencias al número tres. La solución encadena varios cifrados clásicos usando parámetros relacionados con `3`.

## Reconocimiento

El título `TriTriTri` y la frase `"Tres tristes tigres..."` apuntan al cifrado **Trifid**.\
Ciphertext: `xrnodijiptyoyticfxyipsaghffboyciefzjkfslxqhvnkvbjc+hvaobboxzultwvavkwmcdmxrsrvb+jkhzbcoi+pezwbty+epyiiqsk+cmvx+dvzlerbrmvh`

## Explotación / Procedimiento

1. Descifrar con **Trifid**, usando período/grupo `3`.

2. La siguiente capa se resuelve con **Caesar shift 3**.

3. El resultado entrega una pista relacionada con `"fences"`, indicando **Rail Fence**.

4. Aplicar Rail Fence con parámetros relacionados con `3`.

5. Se obtiene: `losclasicossiempresoninteresantes`

6. Calcular MD5: `printf %s 'losclasicossiempresoninteresantes' | md5sum`\
    Resultado: `dc4726805b1d4dbd7447e1520612c563`


## Flag

`dc4726805b1d4dbd7447e1520612c563`

## Notas adicionales

La descripción era parte fundamental del reconocimiento: `Tri` → Trifid, `3` → parámetros y `fences` → Rail Fence. Una capa también puede entregar directamente la pista para la siguiente.

## Qué debo recordar para otro CTF

- El nombre y la descripción pueden aportar tanto el algoritmo como sus parámetros.
- Una salida intermedia puede ser una instrucción para la siguiente capa.

## Aporte para el arsenal

- No probar cifrados clásicos sin límites; usar pistas y puntuación lingüística.
- Conservar parámetros de Trifid, Caesar y Rail Fence en la ruta.
