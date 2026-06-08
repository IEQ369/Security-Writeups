# Disko 3

- Plataforma: PicoCTF
- Categoría: Forense
- Dificultad: Medium
- Tags: #forense #dd #fat32 #binwalk #gzip

## Resumen
Reto de análisis forense con una imagen de disco en formato FAT32. Aunque el enunciado advierte que no es tan directo como parece, la flag se encuentra comprimida dentro de los logs del sistema, accesible tras extraer el sistema de archivos con `binwalk`.

## Material
- `disko-3.dd`

## Descripción del reto

"Can you find the flag in this disk image? This time, its not as plain as you think it is!"

La pista sugiere que la flag no está en texto plano directamente accesible, sino que requiere un paso intermedio de extracción o desempaquetado.

## Procedimiento

### 1. Identificación del tipo de archivo

```bash
file disko-3.dd
```

Salida:
```
disko-3.dd: DOS/MBR boot sector, code offset 0x58+2, OEM-ID "mkfs.fat", Media descriptor 0xf8, sectors/track 32, heads 8, sectors 204800 (volumes > 32 MB), FAT (32 bit), sectors/FAT 1576, serial number 0x49838d0b, unlabeled
```

Confirmado: sistema de archivos **FAT32**. No requiere análisis de particiones complejo como en retos anteriores con imágenes MBR.

### 2. Extracción del sistema de archivos

Con `binwalk`:

```bash
binwalk -e disko-3.dd
```

Salida relevante:
```
DECIMAL                            HEXADECIMAL                        DESCRIPTION
-----------------------------------------------------------------------------------------------------------------------------
0                                  0x0                                FAT file system, type: FAT32, total size: 104857600 bytes

[+] Extraction of fat data at offset 0x0 completed successfully
```

Se genera el directorio:
```
extractions/disko-3.dd.extracted/
```

### 3. Exploración del contenido extraído

```bash
cd extractions/disko-3.dd.extracted
```

Estructura del directorio raíz extraído:
```
0/
└── rootfs/
    └── log/
        ├── alternatives.log.2.gz
        ├── apt/
        ├── dpkg.log.1
        ├── dpkg.log.4.gz
        ├── dpkg.log.5.gz
        ├── flag.gz
        ├── kern.log.3.gz
        ├── kern.log.4.gz
        ├── ...
```

Archivo sospechoso: `flag.gz`

### 4. Extracción de la flag

Intento inicial erróneo:
```bash
gzip -d flag.gzip
```

Error:
```
gzip: flag.gzip.gz: No such file or directory
```

El nombre correcto es `flag.gz`, no `flag.gzip.gz`.

Extracción correcta:
```bash
gunzip flag.gz
```

Archivo generado: `flag`

Lectura del contenido:
```bash
cat flag
```

Salida:
```
Here is your flag
picoCTF{n3v3r_z1p_2_h1d3_26d4f233}
```

## Flag

```
picoCTF{n3v3r_z1p_2_h1d3_26d4f233}
```

Formato del reto:
```
picoCTF{n3v3r_z1p_2_h1d3_26d4f233}
```

## Notas
- La advertencia "its not as plain as you think it is" se debe a que la flag no está en texto plano directamente en la raíz del sistema, sino comprimida dentro del directorio `log/`.
- `binwalk -e` alcanzó para exponer todo el sistema de archivos FAT32 sin necesidad de `mmls` ni `fls`.
- El error inicial con `flag.gzip` vs `flag.gz` es un detalle común al confundir la extensión real del archivo comprimido.
- Herramientas usadas: `file`, `binwalk`, `gunzip`, `cat`.
