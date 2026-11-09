# Write-up: Secuestro del michi

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Forense
- Archivo/objetivo: `forence.txt`
- Estado: `resuelto`

## Descripción del reto

"Secuestraron al gato de una persona importante (no diremos quien es por seguridad), ayúdalo a reunirse de nuevo con su dueño que lo extraña mucho. Autor:Fabian Rieral"

`forence.txt`

---
### Primeros pasos
* Determinar el tipo de archivo que es `forence.txt`, en este caso resulta que es un .zip
* lo descomprimimos con `unzip`

```bash
unzip forence.txt
```

* El archivo descomprimido es `secuestro`, si usamos "file" es de tipo boot
* Usamos para ver listar el contenido
```bash
fdisk -l secuestro
```
**Salida:**
```bash
Disk secuestro: 128 MiB, 134217728 bytes, 262144 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xc5ce543f

Device     Boot Start    End Sectors  Size Id Type
secuestro1       2048 262143  260096  127M  b W95 FAT32
```
* Lo importante es el Start que es: 2048 para la herramienta "fls"
### Analizar con "fls"
* Usamos el siguiente linea
```bash
fls -o 2048 secuestro
```
**Salida:**
```bash
d/d 5:  Documentations
d/d 7:  Files
d/d 9:  WebSites
v/v 4096995:    $MBR
v/v 4096996:    $FAT1
v/v 4096997:    $FAT2
V/V 4096998:    $OrphanFiles
```
### 1) Listar el contenido de un directorio concreto (usando su inodo)

Para ver qué hay dentro del directorio con inodo `5` (Documentations):
```bash
fls -o 2048 secuestro 5
```
**Salida:**
```bash
r/r 25: tartes_flambee_a_volonte_francais_2013.pdf
r/r 28: mangeur-de-cigogne (1).pdf
r/r * 32:       La rÃ©sistance Ã©lectronique.pdf
r/r 34: Menu AC.pdf
r/r 54726:      brasserie_jo_dinner_menu.pdf
r/r 54729:      Courba13-01.pdf
r/r 54731:      m-flamm.pdf
r/r 54734:      Barbey_Cigognes_BDC.pdf
r/r * 54738:    Anarchie, indolence & synarchie.pdf
r/r * 178246:   anarchistscookbookv2000.pdf
r/r 178250:     texte_migration_des_cigognes.pdf
r/r 178253:     mangeur-de-cigogne.pdf
```

* En este caso nos interesa `7` (files)
```bash
fls -o 2048 secuestro 7
```
```bash
r/r * 246775:   revendications.odt
r/r 246778:     421_20080208011.doc
r/r 246780:     Coker.doc
r/r 246784:     DataSanitizationTutorial.odt
r/r 363796:     Creer_votre_association.doc
```

* Sacamos los archivos con `icat`, sacaremos el revendications.odt
```bash
icat -o 2048 secuestro 246775 > revendications.odt
```

### ¿Por que un archivo .odt?
**Ademas de que tenga un "*" en el principio, significa que fue borrado antes

* Los .odt se pueden abrir como texto comprimido .zip
* es básicamente un **documento de texto ZIP comprimido**
asi que usaremos `unzip` para listar o descomprimirlo directo

**Listar**
```bash
unzip -l revendications.odt
```
**Descomprimir**
```bash
unzip revendications.odt -d extract
```

```bash
Archive:  revendications.odt
  Length      Date    Time    Name
---------  ---------- -----   ----
       39  2013-07-22 21:25   mimetype
    41974  2013-07-22 21:25   Thumbnails/thumbnail.png
  2291450  2013-07-22 21:25   Pictures/1000000000000CC000000990038D2A62.jpg
     4752  2013-07-22 21:25   content.xml
    11677  2013-07-22 21:25   styles.xml
     9882  2013-07-22 21:25   settings.xml
     1040  2013-07-22 21:25   meta.xml
      899  2013-07-22 21:25   manifest.rdf
        0  2013-07-22 21:25   Configurations2/accelerator/current.xml
        0  2013-07-22 21:25   Configurations2/toolpanel/
        0  2013-07-22 21:25   Configurations2/statusbar/
        0  2013-07-22 21:25   Configurations2/progressbar/
        0  2013-07-22 21:25   Configurations2/toolbar/
        0  2013-07-22 21:25   Configurations2/images/Bitmaps/
        0  2013-07-22 21:25   Configurations2/popupmenu/
        0  2013-07-22 21:25   Configurations2/floater/
        0  2013-07-22 21:25   Configurations2/menubar/
     1200  2013-07-22 21:25   META-INF/manifest.xml
---------                     -------
  2362913                     18 files
```

* Nos vamos a Pictures/ y escaneamos el .jpg
* Eso nos dan los metadatos de la imagen y encontraremos GPS
```bash
exiftool extract/Pictures/1000000000000CC000000990038D2A62.jpg
```
```bash
GPS Altitude                    : 16.7 m Above Sea Level
GPS Latitude                    : 47 deg 36' 16.15" N
GPS Longitude                   : 7 deg 24' 52.48" E
```

* Pero nos interesa Latitud y longitud, lo pegamos en google maps
```
47°36'16.15"N 7°24'52.48"E
```
* Y nos lleva a Francia, Especificamente a:

```
Helfrantzkirch, Francia
```

* Convertimos a md5 > `HelfrantzkirchFrancia` -> `f21dc310eaf04ab1aefd513bb961accb`

```
citc{f21dc310eaf04ab1aefd513bb961accb}
```

Fuente contrastada: https://github.com/IEQ369/Security-Writeups/tree/main/writeups/forense/secuestro-michi

## Cadena de resolución

`archivo con extensión engañosa → ZIP → imagen de disco FAT32 → fls/icat → ODT → imagen interna → GPS EXIF → ubicación → MD5 → flag`

## Flag

```text
citc{f21dc310eaf04ab1aefd513bb961accb}
```

## Qué debo recordar para otro CTF

- Un ODT es un contenedor ZIP con XML, miniaturas e imágenes internas.
- El sector inicial de la partición debe provenir de la tabla, no de una cifra memorizada.
- Las coordenadas EXIF pueden ser la pista final y deben conservarse exactamente.

## Aporte para el arsenal

- Enumerar contenedores ofimáticos e imágenes internas sin abrirlos a ciegas.
- Extraer GPS y mostrar formatos útiles para validación manual.

## Capturas de la publicación original

<img width="3264" height="2448" alt="1000000000000CC000000990038D2A62" src="https://github.com/user-attachments/assets/ebb9abcd-a4bc-4dc5-8063-22caf1725122" />

<img width="1380" height="673" alt="260531_21h16m18s_screenshot" src="https://github.com/user-attachments/assets/fc6c9cb4-e5ff-4ce1-a9b2-87066b30a253" />
