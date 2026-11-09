# Write-up: Explotación de estenografía
## Reto
- Plataforma: CTF CGII/AGETIC
- Nombre: Explotación de estenografía
- Categoría: Esteganografía
- Archivo/objetivo: `Explotacion-de-stenografia.jpg`
- Estado: `resuelto`
## Reconocimiento
```bash
file Explotacion-de-stenografia.jpg
exiftool Explotacion-de-stenografia.jpg
```
**Hallazgo:** El JPEG contiene una miniatura EXIF de 144868 bytes. El campo Artist dice `soy como un ogro`, una pista de que el contenido tiene capas.
## Hipótesis
- [x] La miniatura EXIF contiene otra capa de información.
- [x] La primera miniatura también contiene una miniatura anidada.
- [x] El mensaje final está presente visualmente, pero oculto por color o contraste.
## Comandos y resultados
```bash
exiftool -ThumbnailImage -b Explotacion-de-stenografia.jpg > miniatura.jpeg
exiftool -ThumbnailImage -b miniatura.jpeg > miniatura2.jpeg
file miniatura.jpeg miniatura2.jpeg
printf %s boomm | md5sum
```
**Resultado útil:** La segunda miniatura revela `boomm` después de ajustar los colores y el contraste. Su MD5 verificado es `1a0f70142919af1365e7b8b6de10d0f5`, por lo que la flag reportada es `cidsi{1a0f70142919af1365e7b8b6de10d0f5}`.
## Cadena de resolución
`JPEG → metadatos EXIF → thumbnail → thumbnail anidado → ajuste visual → boomm → MD5 → flag`
## Qué debo recordar para otro CTF
- Indicador que me hizo reconocer la técnica: `Thumbnail Offset` y `Thumbnail Length` en ExifTool.
- Herramienta que funcionó: `exiftool` con `-ThumbnailImage -b`.
- Comando mínimo: `exiftool -ThumbnailImage -b imagen.jpg > miniatura.jpg`.
- Error/trampa: detenerse en la primera miniatura; el procedimiento debe repetirse sobre cada artefacto recuperado.
- Pendiente de mejora: reproducir el ajuste de color localmente con parámetros exactos cuando se disponga del archivo original, evitando depender de una página web.

## Aporte para el arsenal

- Extraer miniaturas EXIF de forma recursiva, con límite de profundidad y evitando ciclos.
- Ejecutar el triage completo sobre cada miniatura recuperada.
- Generar variantes de contraste y canales como artefactos revisables, no como resultados confirmados.

## Capturas de la publicación original

<img width="1600" height="1600" alt="Explotacion-de-stenografia" src="https://github.com/user-attachments/assets/46da44df-1fa9-4e62-afea-998f3c3ecbb8" />

<img width="528" height="360" alt="miniatura" src="https://github.com/user-attachments/assets/a40a7079-23c5-4abf-a2ee-eacaacc510f7" />

<img width="1280" height="720" alt="miniatura2" src="https://github.com/user-attachments/assets/a68e814c-597a-4cce-8a78-369ebb5fc83f" />

<img width="1280" height="720" alt="miniatura2" src="https://github.com/user-attachments/assets/682e30e2-7144-41fc-9b63-0e09af4fc961" />
