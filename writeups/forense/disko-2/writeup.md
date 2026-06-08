# Disko 2

- Plataforma: PicoCTF
- Categoría: Forense
- Dificultad: Medium
- Tags: #forense #dd #partition #strings #grep #linux

## Resumen
Reto de análisis forense donde se entrega una imagen de disco comprimida con dos particiones: una Linux y una FAT32. La flag se encuentra dentro de la partición Linux y puede obtenerse con `strings` y `grep`.

## Material
- `disko-2.dd.gz`

## Descripción del reto

"Can you find the flag in this disk image? The right one is Linux! One wrong step and its all gone!"

La imagen contiene dos particiones: una Linux y otra FAT32. Solo la partición Linux lleva a la flag. Un paso incorrecto durante el análisis puede hacer que se pierda acceso a la información relevante.

### 1. Descompresión

```bash
gunzip disko-2.dd.gz
```

Archivo generado: `disko-2.dd`

### 2. Identificación del tipo de archivo

```bash
file disko-2.dd
```

Resultado: imagen de disco con tabla de particiones DOS/MBR.

### 3. Reconocimiento de particiones

Con `mmls`:

```bash
mmls disko-2.dd
```

Salida:
```
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000053247   0000051200   Linux (0x83)
003:  000:001   0000053248   0000118783   0000065536   Win95 FAT32 (0x0b)
004:  -------   0000118784   0000204799   0000000001   Unallocated
```

Con `fdisk`:

```bash
fdisk -l disko-2.dd
```

Salida relevante:
```
Disco disko-2.dd: 100 MiB, 104857600 bytes, 204800 sectores
Unidades: sectores de 1 * 512 = 512 bytes

Disposit.   Inicio Comienzo  Final Sectores Tamaño Id Tipo
disko-2.dd1            2048  53247    51200    25M 83 Linux
disko-2.dd2           53248 118783    65536    32M  b W95 FAT32
```

Particiones relevantes:
- `disko-2.dd1`: Linux, inicio en sector 2048, tamaño 51200 sectores.
- `disko-2.dd2`: FAT32, inicio en sector 53248, tamaño 65536 sectores.

### 4. Extracción de la partición Linux

```bash
dd if=disko-2.dd of=LinuxPartition.img bs=512 skip=2048 count=51200
```

Salida:
```
51200+0 records in
51200+0 records out
26214400 bytes (26 MB, 25 MiB) copied, 0,037567 s, 698 MB/s
```

### 5. Búsqueda de la flag

Búsqueda directa con `strings` filtrada por `pico`:

```bash
strings LinuxPartition.img | grep "pico"
```

Salida relevante:
```
picoCTF{4_P4Rt_1t_i5_055dd175}
piconv
:/icons/appicon
...
```

### 6. Otras formas de exploración

También es posible usar The Sleuth Kit para listar archivos y extraerlos:

```bash
fls -o 2048 disko-2.dd
```

Y para obtener un archivo concreto:

```bash
icat -o 2048 disko-2.dd <inodo> > archivo_salida
```

## Flag

```
picoCTF{4_P4Rt_1t_i5_055dd175}
```

## Notas
- La vía más rápida fue filtrar cadenas de texto con `strings | grep picoCTF` directamente sobre la partición Linux extraída.
- La partición FAT32 no era necesaria para la flag, pero era relevante descartarla primero.
- Herramientas usadas: `gunzip`, `file`, `mmls`, `fdisk`, `dd`, `strings`, `grep`.
