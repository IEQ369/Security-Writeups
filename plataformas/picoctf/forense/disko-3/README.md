# Write-up: Disko 3

## Reto

- Plataforma: PicoCTF
- Categoría: Forense
- Autor: Darkraicg492
- Archivo/objetivo: `disko-3.dd`
- Estado: `resuelto`

## Descripción del Reto

Se proporciona una imagen de disco llamada `disko-3.dd`. El objetivo es encontrar la flag oculta dentro de esta imagen. A diferencia de DISKO 2, donde era necesario identificar y montar particiones Linux, en este caso nos encontramos con un sistema de archivos FAT32 más simple.

La clave del reto es **explorar el contenido** de la imagen y localizar cualquier archivo sospechoso.

---
## Reconocimiento Inicial

Listamos y verificamos el tipo de la imagen:
```bash
file disko-3.dd
```
Salida relevante:
```bash
disko-3.dd: DOS/MBR boot sector ... FAT (32 bit)
```
Esto confirma que el archivo es una imagen con sistema de archivos **FAT32**, por lo que no es necesario extraer particiones ni utilizar herramientas como `mmls`.

---
## Análisis y Extracción de Contenido

Para inspeccionar posibles archivos internos, usamos:
```bash
binwalk -e disko-3.dd
```
Opcionalmente, para buscar archivos recuperables automáticamente (aunque en este caso no era necesario):
```bash
foremost disko-3.dd
```
Esto solo devolvió algunos archivos `.bmp`, pero ninguno era la flag.

El enfoque más directo es listar la estructura del directorio extraído:
```bash
tree extractions/disko-3.dd.extracted
```
Dentro de la estructura aparece un archivo sospechoso:
```bash
.
└── 0
    └── rootfs
        └── log
            ├── alternatives.log.2.gz
            ├── apt
            │   ├── eipp.log.xz
            │   ├── history.log.1.gz
            │   ├── history.log.2.gz
...

            ├── flag.gz
...
```

---
## Obtención de la Flag
Descomprimimos:
```bash
gunzip flag.gz
```
Luego visualizamos el contenido:
```bash
cat flag
```
Flag Encontrada:
```bash
picoCTF{n3v3r_z1p_2_h1d3_26d4f233}
```

## Qué debo recordar para otro CTF
- `file` confirmó que FAT32 comienza en el offset 0; por eso no hizo falta seleccionar una partición con `mmls`.
- Cada elemento recuperado debe volver a pasar por triage: aquí el artefacto de interés era `rootfs/log/flag.gz`.
- La nota anterior confundía `flag.gz` con `flag.gzip` y ejecutaba `tree` sobre la imagen DD en lugar del directorio extraído.
- Si Binwalk no extrae el filesystem, se puede enumerar con The Sleuth Kit o montar la imagen en modo de solo lectura.
- Fuente: https://github.com/IEQ369/Security-Writeups/tree/main/writeups/forense/disko-3

## Cadena de resolución

`imagen FAT32 → enumeración/extracción → rootfs/log/flag.gz → gunzip → flag`

## Aporte para el arsenal

- Detectar si el sistema de archivos comienza en el offset cero o dentro de una partición.
- Enumerar archivos comprimidos encontrados y volver a analizarlos como artefactos hijos.
