# Write-up: Nested Frames

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Esteganografía
- Puntos: 350
- Archivo: `Nested-Frames.mp3`
- Estado: `resuelto`; el archivo demuestra el mensaje final, pero esta nota no inventa un envoltorio que no figura en la descripción

> Múltiples capas de imágenes denominadas atbash

## Resumen

El archivo tiene una extensión falsa: no es un MP3, sino un PNG animado o **APNG** con 13 frames. Las imágenes extraídas no contienen el mensaje visualmente. La información está en la duración de cada frame.

La herramienta `apngdis` extrae tanto las imágenes como pequeños archivos de texto que conservan esa duración. Los numeradores de las 13 duraciones son valores ASCII:

`85 79 90 84 58 52 84 51 71 82 88 53 50`

Al convertirlos a texto aparece `UOZT:4T3GRX52`. La descripción entrega explícitamente la última transformación, Atbash, que produce:

`FLAG:4G3TIC52`

## 1. No confiar en la extensión

El primer paso es preguntar qué contiene realmente el archivo:

```bash
file Nested-Frames.mp3
```

Resultado real:

```text
Nested-Frames.mp3: PNG image data, 512 x 512, 8-bit/color RGBA, non-interlaced
```

La extensión `.mp3` es una distracción. La firma inicial del archivo es la de PNG.

Aquí aparece además una lección importante: ExifTool interpreta este archivo como MP3, mientras que `file` y `pngcheck` reconocen la estructura PNG. Cuando dos herramientas discrepan, hay que contrastar la evidencia en vez de escoger automáticamente la salida que más convenga.

## 2. Descubrir que es un PNG animado

```bash
pngcheck -v Nested-Frames.mp3
```

Entre los chunks aparecen:

```text
acTL
fcTL
fdAT
```

Estos nombres indican un APNG:

- `acTL` controla la animación y declara cuántos frames existen.
- `fcTL` controla un frame: tamaño, posición, duración y forma de combinarlo.
- `fdAT` contiene datos de los frames posteriores al primero.

La versión local de `pngcheck` los muestra como chunks privados desconocidos porque no los interpreta por completo, pero sí permite ver sus nombres y que `fcTL` se repite.

Para confirmarlo con una herramienta que reconoce el formato:

```bash
ffprobe -v error \
  -show_entries stream=codec_name,width,height \
  -of default=noprint_wrappers=1 \
  Nested-Frames.mp3
```

Resultado:

```text
codec_name=apng
width=512
height=512
```

## 3. Usar la herramienta diseñada para APNG

`apngdis` necesita trabajar con el archivo como PNG. Se crea una copia con la extensión correcta; cambiar la extensión no transforma los bytes:

```bash
cp Nested-Frames.mp3 Nested-Frames.png
apngdis Nested-Frames.png
```

La herramienta informa:

```text
extracting frame 1 of 13
...
extracting frame 13 of 13
```

Y crea dos tipos de archivo por cada frame:

```text
apngframe01.png
apngframe01.txt
apngframe02.png
apngframe02.txt
...
apngframe13.png
apngframe13.txt
```

Los PNG son las imágenes visibles. Los TXT conservan parámetros necesarios para volver a montar la animación, incluida su duración.

## 4. Revisar primero las imágenes

Las imágenes muestran distintos memes de Pepe. No forman letras, no contienen un QR y no revelan una flag al verlas en orden.

Esto evita una conclusión equivocada: «Nested Frames» sí indicaba que debíamos separar los frames, pero no necesariamente que el secreto estuviera dibujado en ellos.

También conviene evitar este comando sin opciones adicionales:

```bash
ffmpeg -i Nested-Frames.mp3 frame-%02d.png
```

Como cada frame dura varios segundos, FFmpeg puede repetirlo muchas veces para mantener una tasa de imágenes constante. En esta evidencia produjo cientos de PNG, aunque el APNG solo tiene 13 frames reales. Para este reto, `apngdis` conserva mejor la estructura original.

## 5. Leer los parámetros que extrajo apngdis

No hace falta crear un parser. Basta con mirar los TXT:

```bash
cat apngframe*.txt
```

Salida:

```text
delay=85/10
delay=79/10
delay=90/10
delay=84/10
delay=58/10
delay=52/10
delay=84/10
delay=51/10
delay=71/10
delay=82/10
delay=88/10
delay=53/10
delay=50/10
```

Una duración de APNG es una fracción:

`delay_num / delay_den`

Aquí el denominador siempre es 10. Los numeradores cambian y todos caen en el rango de caracteres ASCII imprimibles. Además, son duraciones absurdamente largas para una animación normal: `85/10` equivale a 8,5 segundos.

Por eso la hipótesis razonable es interpretar los **numeradores** como ASCII.

## 6. Convertir los números a ASCII

La conversión puede hacerse con una tabla ASCII o en CyberChef usando **From Decimal**:

| Número | ASCII |
| ---: | :--- |
| 85 | U |
| 79 | O |
| 90 | Z |
| 84 | T |
| 58 | : |
| 52 | 4 |
| 84 | T |
| 51 | 3 |
| 71 | G |
| 82 | R |
| 88 | X |
| 53 | 5 |
| 50 | 2 |

Resultado:

```text
UOZT:4T3GRX52
```

Esto todavía no parece una flag, pero ya tiene estructura: cuatro letras, dos puntos y un valor alfanumérico.

## 7. Aplicar la pista Atbash

El enunciado dice literalmente `atbash`. Atbash cambia cada letra por su opuesta en el alfabeto:

```text
A ↔ Z
B ↔ Y
C ↔ X
...
```

Las primeras cuatro letras permiten validar la hipótesis:

```text
U → F
O → L
Z → A
T → G
```

Por tanto:

```text
UOZT:4T3GRX52
        ↓ Atbash
FLAG:4G3TIC52
```

Los números y los dos puntos no cambian.

## Resultado

El mensaje demostrado por el archivo es:

```text
FLAG:4G3TIC52
```

El valor recuperado es:

```text
4G3TIC52
```

### Flag
```text
cidsi{290947209de781300346c9772ffa51f9}
```
## Cadena de resolución

`extensión MP3 falsa → PNG → chunks de APNG → apngdis → 13 frames + 13 TXT → numeradores de delay → ASCII → UOZT:4T3GRX52 → Atbash → FLAG:4G3TIC52`

## Qué debo recordar para otro CTF

- `file` descubre que la extensión no coincide con el contenido.
- `acTL`, `fcTL` y `fdAT` son indicadores de APNG.
- `apngdis` conserva imágenes y parámetros de cada frame.
- El secreto de una animación también puede estar en tiempos, offsets, dimensiones u otros campos de control.
- Una secuencia de números en el rango 32–126 merece una prueba como ASCII, pero solo se acepta si produce estructura coherente.
- La palabra Atbash ya estaba en la descripción; no era necesario adivinar el cifrado.
- Un parser propio era innecesario porque `apngdis` ya exponía `delay_num/delay_den`.

## Aporte para el arsenal

- Detectar `acTL`, `fcTL` y `fdAT`.
- Enumerar frames reales sin duplicarlos según su duración.
- Recoger delays, dimensiones y offsets mediante herramientas existentes.
- Proponer ASCII cuando varios valores consecutivos sean imprimibles.
- Exigir coherencia global antes de marcar una secuencia como mensaje.
- Mostrar Atbash como transformación prioritaria únicamente cuando exista una pista o cuando produzca una estructura claramente superior.
