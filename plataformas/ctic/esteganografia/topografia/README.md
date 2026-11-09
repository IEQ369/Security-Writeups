# Write-up: Topografía
## Reto
- Plataforma: CTF CGII/AGETIC
- Nombre: Topografía
- Categoría: Esteganografía
- Archivo/objetivo: `Topografia.jpg`
- Estado: `resuelto`
## Reconocimiento
```bash
file Topografia.jpg
exiftool Topografia.jpg
```
**Hallazgo:** Es una imagen JPEG. El campo Copyright contiene `Aqui no hay nada... AGETIC te puede ser de utilidad`, que señala la contraseña `AGETIC`.
## Hipótesis
- [x] La pista de los metadatos funciona como passphrase de `steghide`.
- [x] El texto extraído contiene un token repetido que debe eliminarse.
## Comandos y resultados
```bash
steghide extract -sf Topografia.jpg -p AGETIC
cat Oculto.txt
printf %s INVESTIGAPARASERMEJORYPRACTICACADADIA | md5sum
```
**Resultado útil:** `steghide` extrae `Oculto.txt`. Al eliminar cada aparición de `CGII` de la cadena oculta se obtiene `INVESTIGAPARASERMEJORYPRACTICACADADIA`; su MD5 verificado es `d58054ab9b51e1750e5c361aac27e8e0`. El repositorio conserva los formatos `cidsi{d58054ab9b51e1750e5c361aac27e8e0}` y `citc{d58054ab9b51e1750e5c361aac27e8e0}`; el prefijo debe validarse según la edición de la competencia.
## Cadena de resolución
`JPEG → EXIF Copyright → passphrase AGETIC → steghide → Oculto.txt → eliminar CGII → MD5 → flag`
## Qué debo recordar para otro CTF
- Indicador que me hizo reconocer la técnica: una frase llamativa en Copyright que coincide con el nombre de la organización.
- Herramienta que funcionó: `exiftool` y `steghide`.
- Comando mínimo: `steghide extract -sf Topografia.jpg -p AGETIC`.
- Error/trampa: ignorar metadatos textuales o confundir la inserción repetida con un cifrado complejo.

## Aporte para el arsenal

- Recoger textos de EXIF como candidatos de contraseña y probarlos con su forma original y variaciones mínimas.
- Si Steghide extrae texto, buscar tokens repetidos sin eliminarlos automáticamente: mostrar ambas versiones y preservar el orden.

## Capturas de la publicación original

<img width="1200" height="1200" alt="Topografia" src="https://github.com/user-attachments/assets/e6ff9875-b6c5-4512-9cc8-4e6e4b288c80" />
