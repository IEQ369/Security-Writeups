# Write-up: Filtros

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Esteganografía
- Archivo/objetivo: `Filtros.zip` con 38 imágenes JPEG
- Estado: `resuelto`

## Resumen

Cada imagen contenía en el metadato `Artist` un byte hexadecimal. Al extraer ese campo de todos los JPEG en el orden correcto y convertir los bytes a ASCII se reconstruyó la flag.

## Análisis inicial

- Se recibió un `.zip` llamado `Filtros.zip` con **38 archivos JPEG**.

- Sin descripción adicional más allá de “Filtros Filtros Filtros”.


## Inspección de los archivos

- Se ejecutó `exiftool` para revisar los metadatos:
```bash
exiftool *
```
- La mayoría de las imágenes tenían un tag `Artist` con valores hexadecimales (`0x..`).

- Otros metadatos como `File Name`, `File Size` y `User Comment` estaban presentes pero no eran relevantes para el flag.
## Primer intento de filtrar

- Se intentó usar:
```bash
exiftool * | sort | uniq -u
```
Esto **quitó los valores repetidos**, pero como el flag necesitaba todos los valores en orden, este paso fue un error: perdías parte del flag.

## Extraer solamente `Artist` en orden
* Se encontró la forma correcta de extraer **solo los valores de Artist** sin perder el orden:
```bash
exiftool -s -s -s -Artist *.jpg
```
Esto mostró únicamente los valores hexadecimales en cada línea:
```bash
0x63
0x69
0x74
0x63
0x7b
...
0x7d
```
- Cada línea representaba un carácter ASCII del flag.

## Traducir de hexadecimal a ASCII

- Se revisaron varias formas de traducir los hex a texto:

    - Usando `CyberChef`.

    - Usando `xxd`

    - Con scripts en Python o Perl
- Finalmente se entendió la idea principal: cada `0xHH` es un byte ASCII que, al concatenarse en orden, forma el flag.


## Flag

- Al convertir todos los hex en ASCII en el orden correcto, se obtuvo el flag:
```bash
citc{16cffa7b89682e0d784c246ec78eacf5}
```

## Cadena de resolución

`ZIP → 38 JPEG → metadato Artist en orden de archivo → bytes 0xHH → ASCII → flag`

## Qué debo recordar para otro CTF

- Extraer solamente el campo útil reduce el ruido sin alterar los datos.
- `sort | uniq -u` habría destruido repeticiones necesarias para reconstruir la flag.
- Ordenar candidatos y reconstruir un mensaje son problemas distintos.

## Aporte para el arsenal

- Extraer el mismo metadato de una colección preservando nombre y orden.
- Detectar secuencias `0xHH` y proponer ASCII sin eliminar duplicados.
