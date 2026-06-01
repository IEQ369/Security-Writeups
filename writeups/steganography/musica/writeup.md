# Musica

- Plataforma: Custom
- Categoría: Esteganografía
- Puntos: 500
- Fecha: 2026-05-31
- Tags: #steganography #audio #espectrograma #mp3 #audacity

## Resumen
Reto de esteganografía en formato de audio. Se entrega un archivo ZIP que contiene un MP3. La flag se encuentra oculta en el espectrograma del audio. No es necesario reproducir la pista completa; basta con visualizar la sección del espectrograma que contiene el mensaje oculto.

## Material
- `Musica.zip`
- `Digital Pop sin copyright.mp3` (extraído del ZIP)

## Reconocimiento
Listar archivos en directorio:

```bash
ls
```

```
Musica.zip
```

Extracción del ZIP:

```bash
unzip Musica.zip
```

```
Archive:  Musica.zip
  inflating: Digital Pop sin copyright.mp3
```

Validar tipo de archivo:

```bash
file Digital\ Pop\ sin\ copyright.mp3
```

```
Digital Pop sin copyright.mp3: Audio file with ID3 version 2.4.0, contains: MPEG ADTS, layer III, v1, 128 kbps, 44.1 kHz, JntStereo
```

## Análisis del espectrograma
Se abre el MP3 en Audacity. No hace falta escuchar toda la canción; en una sección específica del espectrograma aparece el texto `3spectr0gram4`.

<img width="1886" height="559" alt="260531_21h53m45s_screenshot" src="https://github.com/user-attachments/assets/7954cc28-9d57-48e2-9f5f-e1cf144ea04f" />

<img width="392" height="333" alt="260531_21h54m01s_screenshot" src="https://github.com/user-attachments/assets/23aa9679-afa0-4e7f-acb4-52a0c7fdf7a8" />


## Procedimiento
1. Extraer `Musica.zip` para obtener el MP3.
2. Abrir el MP3 en Audacity.
3. Cambiar la pista a vista de espectrograma.
4. Navegar a la sección que contiene `3spectr0gram4`.
5. Calcular el MD5 de esa cadena:

```bash
echo -n "3spectr0gram4" | md5sum
```

```
7e8019168654eae555a5bdf91ede0f9e
```

## Flag
```bash
cidsi{7e8019168654eae555a5bdf91ede0f9e}
citc{7e8019168654eae555a5bdf91ede0f9e}
```

## Notas
- Este reto se resolvió íntegramente por inspección visual del espectrograma; no requirió decodificación adicional ni herramientas CLI de esteganografía.
- El mensaje `3spectr0gram4` apareció en una sección puntual del audio; la escucha completa no aportó información.
- Herramientas usadas: `unzip`, `file` y Audacity.
