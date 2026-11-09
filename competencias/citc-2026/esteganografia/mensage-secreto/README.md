# Mensage_Secreto

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Esteganografía. Puntos: 80. Autor: GK4ME.
- Archivo: ![mensaje_secreto.jpg](imagenes/mensaje-secreto.jpg).
- Estado: **parcial; no se recuperó un mensaje oculto**.

> La clave está en los colores: seis símbolos después de un `#`. Prestar atención a aquello que ocupa el último lugar.

## Reconocimiento e hipótesis

La pista sugiere un código hexadecimal RGB y quizá el último píxel o el bit menos significativo. Son hipótesis; ninguna permite inventar una bandera sin comprobar el archivo.

## Análisis realizado

```bash
file mensaje_secreto.jpg
exiftool mensaje_secreto.jpg
binwalk mensaje_secreto.jpg
```

Comprobar color y variaciones:

```bash
python3 - <<'PY'
from PIL import Image
im = Image.open('mensaje_secreto.jpg').convert('RGB')
colors = im.getcolors(im.width * im.height)
print('Tamaño:', im.size)
print('Colores:', colors)
print('Último píxel:', im.getpixel((im.width-1, im.height-1)))
PY
```

En la copia analizada había un millón de píxeles iguales: RGB **(254, 0, 0)**, hexadecimal **#FE0000**. El último píxel no aportaba una diferencia. No se identificaron datos anexados o metadatos secretos.

## Resultado y límites

Se propusieron precipitadamente `FE0000` y `FF0000` como respuesta. Un color uniforme no demuestra una bandera. La posibilidad de que una recomprensión JPEG destruyera el contenido **no fue demostrada**; no debe registrarse como causa confirmada.

## Cadena de resolución

`JPEG → color uniforme y último píxel → sin mensaje recuperado → pendiente de nueva evidencia`

## Flag

**No recuperada.**

## Qué debo recordar / aporte para el arsenal

Contar colores antes de buscar LSB por todo el archivo. Distinguir píxeles uniformes de pruebas concluyentes sobre todas las formas de esteganografía JPEG.
