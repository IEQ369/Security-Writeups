# Write-up: Scan Surprise

## Reto

- Plataforma: PicoCTF
- Categoría: Forense
- Archivo/objetivo: `challenge.zip` y la imagen que contiene
- Estado: `resuelto`

## Descripción original

| I've gotten bored of handing out flags as text. Wouldn't it be cool if they were an image instead? You can download the challenge files here:<br><br>    challenge.zip<br><br>Additional details will be available after launching your challenge instance. | Me cansé de repartir las banderas en formato de texto. ¿No sería genial que fueran imágenes? Puedes descargar los archivos del desafío aquí:<br><br>challenge.zip<br><br>Encontrarás más detalles después de iniciar el desafío. |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

es un zip y solo es un QR al que hay que escannear

Yo use esta pagina
+ https://4qrcode.com/scan-qr-code.php?lang=es
```bash
picoCTF{p33k_@_b00_0194a007}
```

## Cadena de resolución

`ZIP → extraer → inspección visual → código QR → flag`

## Flag

```text
picoCTF{p33k_@_b00_0194a007}
```

## Qué debo recordar para otro CTF

- Inspeccionar visualmente antes de probar técnicas avanzadas.
- Un QR visible puede leerse localmente con zbarimg; no es obligatorio subirlo a una web.

## Aporte para el arsenal

- Detectar QR y devolver su contenido al análisis por capas.
- No ejecutar esteganálisis costoso si ya existe un hallazgo visual claro.
