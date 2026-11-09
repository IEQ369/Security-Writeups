# Señal Interceptada

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría de trabajo: esteganografía de audio. Puntos: 80. Autor: drei.
- Archivo original: `senal_interceptada.wav` (**no recuperado**).
- Estado: **mensaje extraído localmente durante la competencia; aceptación no registrada**.

> Audio interceptado en los canales de Allsafe con firma de fsociety. La banda más evidente del espectrograma no es la única que importa; ni el espectro más nítido enseña toda la señal.

## Resumen

El WAV contenía varias pistas visuales y metadatos. El mensaje más concluyente apareció en el LSB de las muestras PCM, con una cabecera que indicaba correctamente su longitud.

## Reconocimiento

```bash
file senal_interceptada.wav
exiftool senal_interceptada.wav
ffprobe -v error -show_streams senal_interceptada.wav
```

La salida histórica identifica **PCM de 16 bits little-endian, mono, 96000 Hz, 18,32 segundos**. Una muestra ocupa dos bytes; no debemos confundir su LSB con tomar el último bit de todos los bytes del archivo, pues eso incluiría cabeceras y componentes equivocados.

## Hipótesis

La advertencia sobre lo que el ojo no ve justifica continuar tras inspeccionar el espectrograma. En PCM, modificar el bit de menor peso altera muy poco la amplitud: puede almacenar datos que no formen letras visibles en el espectro.

## Resolución paso a paso

1. Revisar los metadatos: había Base64 que llevaba a mensajes de distracción, incluido `l4_m4sc4r4_no_es_l4_fl4g`.
2. Abrir el WAV en Audacity, cambiar la pista a espectrograma y revisar diferentes bandas y contraste. Esos mensajes no bastaban para cerrar el reto. Se guardaron vistas recuperadas en `evidencias/`.
3. Extraer un bit por muestra, empezando por el LSB. Agrupar cada ocho bits en un byte, de más significativo a menos significativo.

Si se recupera el WAV, este bloque se pega **entero en la terminal**; `python3` ejecuta el código entre ambos `PY`:

```bash
python3 - <<'PY'
import wave
with wave.open('senal_interceptada.wav', 'rb') as w:
    assert w.getsampwidth() == 2 and w.getnchannels() == 1
    pcm = w.readframes(w.getnframes())
bits = [pcm[i] & 1 for i in range(0, len(pcm), 2)]
data = bytes(sum(bits[i+j] << (7-j) for j in range(8))
             for i in range(0, len(bits)-7, 8))
n = int.from_bytes(data[:4], 'big')
assert n <= len(data) - 4
print('Longitud:', n)
print(data[4:4+n].decode('ascii'))
PY
```

4. La extracción histórica comienza con `00 00 00 20`. Interpretado como entero big-endian equivale a **32**, exactamente los bytes del mensaje siguiente. Esa coincidencia respalda la estructura, no demuestra por sí sola aceptación del juez.

## Cadena de resolución

`WAV PCM → metadatos y espectrograma → LSB de muestras → cabecera de longitud → texto`

## Flag / resultado

```text
h3ll0_fr13nd_w3lc0m3_t0_fs0c13ty
```

No se conserva confirmación de un envoltorio `fsociety{...}` o `citc{...}`. Esas propuestas durante la competencia fueron candidatas, no resultados aceptados.

## Qué debo recordar para otro CTF

Un espectrograma no muestra toda la información binaria del audio. Respetar el formato PCM, el tamaño de muestra y el orden de bits.

## Aporte para el arsenal

Probar LSB por muestra, informar tamaño y canales, reconocer longitudes y separar mensajes de distracción de candidatas. No tratar un resultado de `binwalk` como archivo real sin validar su estructura.

## Archivos y límites

Se recuperaron siete vistas del espectrograma y dos salidas de extracción desde el historial local. **No equivalen al WAV** ni permiten reconstruir sus muestras originales. El bloque anterior reproduce el algoritmo registrado; no pudo probarse otra vez contra el audio ausente.
