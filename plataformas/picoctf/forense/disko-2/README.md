# Write-up: Disko 2

## Reto

- Plataforma: PicoCTF
- Categoría: Forense
- Archivo/objetivo: `disko-2.dd`
- Estado: `resuelto`

## Descripción
| Can you find the flag in this disk image? The right one is Linux! One wrong step and its all gone! Download the disk image here. | ¡¿Puedes encontrar la bandera en esta imagen de disco? ¡La correcta es Linux! ¡Un paso en falso y lo perderás todo! Descarga la imagen de disco aquí. |
| -------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
### Pistas
¿Cómo se puede extraer/aislar una partición?

## Ver particiones (mmls / fdisk / parted)

Usa `mmls` (sleuthkit) para ver particiones:
```bash
mmls disko-2.dd
```
o tambien para ver info:
```bash
fdisk -l disko-2.dd
```
## Extraer la particion de linux

```bash
dd if=disko-2.dd of=LinuxPartition.img bs=512 skip=2048 count=51200
```

#### Antes de realizar cualquier tecnica avanzada, buscar cadenas simples o un GREP
```bash
strings LinuxPartition.img | grep 'picoCTF'
```
**Resultado:**
```bash
picoCTF{4_P4Rt_1t_i5_055dd175}
```
 ---
 #### Otra manera de usarse con estas herramientas
 **Ejemplo:**
 montar
```bash
fdisk -l <nombre>
```

```bash
fls -o 2048 disko-2.dd
# 2048 es el sector Start obtenido con mmls o fdisk
```
**Para extraer archivos**
```bash
icat -o 2048 disko-2.dd <inodo> > archivo_salida
```

La vía más rápida en este reto fue extraer la partición Linux y buscar cadenas. `fls` e `icat` quedan como alternativa cuando sea necesario identificar y recuperar un archivo concreto.

Fuente contrastada: https://github.com/IEQ369/Security-Writeups/tree/main/writeups/forense/disko-2

## Cadena de resolución

`imagen de disco → tabla de particiones → offset 2048 → partición Linux → strings/grep → flag`

## Flag

```text
picoCTF{4_P4Rt_1t_i5_055dd175}
```

## Qué debo recordar para otro CTF

- El valor `2048` proviene del sector inicial mostrado por `mmls` o `fdisk`; no es una constante universal.
- Antes del carving avanzado, buscar cadenas conocidas puede resolver el reto de manera barata.
- Trabajar sobre una copia y montar solamente en modo lectura.

## Aporte para el arsenal

- Enumerar particiones y explicar cómo convertir sectores en offsets.
- Buscar formatos de flag conservando su ubicación dentro de la evidencia.
