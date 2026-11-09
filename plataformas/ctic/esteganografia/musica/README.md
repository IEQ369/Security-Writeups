# Write-up: Música
## Reto
- Plataforma: CTIC / AGETIC
- Nombre: Música
- Categoría: Esteganografía
- Archivo/objetivo: `Musica.zip` → `Digital Pop sin copyright.mp3`
- Estado: `resuelto`
## Reconocimiento
```bash
unzip -l Musica.zip
unzip Musica.zip
file 'Digital Pop sin copyright.mp3'
```
**Hallazgo:** El ZIP contiene un MP3 válido de 44.1 kHz. No aparece una pista textual útil en el contenedor.
## Hipótesis
- [x] El mensaje está codificado visualmente en el espectrograma del audio.
## Comandos y resultados
```bash
audacity 'Digital Pop sin copyright.mp3'
printf %s 3spectr0gram4 | md5sum
```
**Resultado útil:** Al cambiar la pista a vista de espectrograma aparece `3spectr0gram4`. Su MD5 verificado es `7e8019168654eae555a5bdf91ede0f9e`. El repositorio conserva los formatos `cidsi{7e8019168654eae555a5bdf91ede0f9e}` y `citc{7e8019168654eae555a5bdf91ede0f9e0}`; el segundo contiene un carácter adicional y debe considerarse un error tipográfico. El hash verificado termina en `f9e`.
## Cadena de resolución
`ZIP → MP3 → vista de espectrograma → 3spectr0gram4 → MD5 → flag`
## Qué debo recordar para otro CTF
- Indicador que me hizo reconocer la técnica: audio entregado en un reto de esteganografía sin otra capa evidente.
- Herramienta que funcionó: Audacity en vista de espectrograma.
- Comando mínimo: abrir el audio y cambiar la visualización de forma de onda a espectrograma.
- Error/trampa: escuchar la canción completa cuando el mensaje se encuentra en el dominio de frecuencia.
## Aporte para el arsenal

- Generar automáticamente un espectrograma estático con `sox` o `ffmpeg`.
- Entregar la imagen al OCR como sugerencia, sin afirmar que cualquier texto detectado sea una flag.
- Conservar la escucha manual como segunda ruta cuando el espectrograma no explique el audio.

## Capturas de la publicación original

<img width="1886" height="559" alt="260531_21h53m45s_screenshot" src="https://github.com/user-attachments/assets/7954cc28-9d57-48e2-9f5f-e1cf144ea04f" />

<img width="392" height="333" alt="260531_21h54m01s_screenshot" src="https://github.com/user-attachments/assets/23aa9679-afa0-4e7f-acb4-52a0c7fdf7a8" />
