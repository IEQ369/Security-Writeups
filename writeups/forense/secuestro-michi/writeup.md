# El secuestro del michi

- Plataforma: CTF CGII/AGETIC (ctf.cgii.gob.bo)
- Categoría: Forense
- Puntos: 290
- Fecha: 2026-05-31
- Tags: #forense #zip #odt #exiftool #gps #fls #icat

## Resumen
Reto de análisis forense donde la flag estaba oculta dentro de una imagen JPEG embebida en un documento ODT, a través de metadatos GPS extraídos con ExifTool.

## Material
- `forence.txt` (fichero ZIP disfrazado)

## Reconocimiento

Determinar el tipo de archivo:

```bash
file forence.txt
```

Resultado: ZIP archive data.

Descompresión:

```bash
unzip forence.txt
```

Archivo generado: `secuestro`

Tipo del disco:

```bash
file secuestro
```

Resultado: DOS/MBR boot sector. Se trata de una imagen de disco.

Particiones:

```bash
fdisk -l secuestro
```

- Partición 1: ID=0xb (W95 FAT32)
- Inicio: sector 2048
- Tamaño: 127M

## Análisis del sistema de archivos

Listar contenido raíz del filesystem:

```bash
fls -o 2048 secuestro
```

Directorios relevantes:
- `5`: Documentations
- `7`: Files
- `9`: WebSites

Listar contenido del directorio `Files` (inodo 7):

```bash
fls -o 2048 secuestro 7
```

Archivos relevantes:
- `246775`: `revendications.odt` (marcado como borrado `*`)
- otros archivos: `421_20080208011.doc`, `Coker.doc`, etc.

## Extracción del documento ODT

```bash
icat -o 2048 secuestro 246775 > revendications.odt
```

`revendications.odt` es un documento ODT, que en realidad es un ZIP comprimido.

Listar contenido:

```bash
unzip -l revendications.odt
```

Archivo relevante:
- `Pictures/1000000000000CC000000990038D2A62.jpg`

Extraer todo:

```bash
unzip revendications.odt -d extract
```

## Análisis de la imagen embebida

Ruta de la imagen:
- `Pictures/1000000000000CC000000990038D2A62.jpg`

<img width="3264" height="2448" alt="1000000000000CC000000990038D2A62" src="https://github.com/user-attachments/assets/ebb9abcd-a4bc-4dc5-8063-22caf1725122" />


Metadatos EXIF:

```bash
exiftool extract/Pictures/1000000000000CC000000990038D2A62.jpg
```

Datos relevantes:
- Cámara: iPhone 4S
- Fecha: 2013-03-11 11:47:07
- GPS Latitude: 47 deg 36' 16.15" N
- GPS Longitude: 7 deg 24' 52.48" E
- GPS Altitude: 16.7 m Above Sea Level

## Ubicación GPS

Coordenadas:
```
47°36'16.15"N 7°24'52.48"E
```

Coordenadas decimales:
- Latitud: 47.604486
- Longitud: 7.414022

Lugar identificado: Helfrantzkirch, Francia

<img width="1380" height="673" alt="260531_21h16m18s_screenshot" src="https://github.com/user-attachments/assets/fc6c9cb4-e5ff-4ce1-a9b2-87066b30a253" />


## Cálculo de la flag

Formato de entrega: `citc{md5}`

Palabra base: `HelfrantzkirchFrancia`

```bash
echo -n "HelfrantzkirchFrancia" | md5sum
```

```
f21dc310eaf04ab1aefd513bb961accb
```

## Flag
```
citc{f21dc310eaf04ab1aefd513bb961accb}
```

## Notas
- El archivo `forence.txt` tenía extensión engañosa; era un ZIP que contenía una imagen de disco.
- El archivo `secuestro` se analizó como imagen de disco con `fdisk` y `fls`.
- El archivo `revendications.odt` estaba marcado como borrado en el filesystem.
- Los ODT son ZIP; se pueden listar y extraer con `unzip`.
- La flag se obtuvo a partir de coordenadas GPS en metadatos EXIF de una foto embebida en el documento.
- Autor del reto: Fabian Rieral.
