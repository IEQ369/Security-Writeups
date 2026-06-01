# Explotación de estenografía

- Plataforma: CTF CGII/AGETIC (ctf.cgii.gob.bo)
- Categoría: Esteganografía
- Puntos: 100
- Fecha: 2026-05-31
- Tags: #steganography #jpeg #exiftool #thumbnail #filters

## Resumen
La flag estaba oculta dentro de una imagen JPEG mediante miniaturas anidadas y filtros de color que ocultaban el texto real.

## Material
- `Explotacion-de-stenografia.jpg` (856 kB, 1600x1600)

**Imagen 1 — Archivo original `Explotacion-de-stenografia.jpg`**
![Archivo original](imagenes/01-original.jpg)

## Reconocimiento
```bash
exiftool Explotacion-de-stenografia.jpg
```
Datos relevantes:
- Artist: `soy como un ogro`
- Thumbnail Offset: 232
- Thumbnail Length: 144868 bytes
- El archivo contiene miniaturas incrustadas.

**Imagen 2 — Salida de exiftool mostrando metadatos EXIF y thumbnail**
![ExifTool](imagenes/02-exiftool.jpg)

## Extracción de miniaturas
```bash
exiftool -ThumbnailImage -b Explotacion-de-stenografia.jpg > miniatura
mv miniatura miniatura.jpeg
exiftool -ThumbnailImage -b miniatura.jpeg > miniatura2
mv miniatura2 miniatura2.jpeg
```

La segunda miniatura (`miniatura2.jpeg`) mostraba una imagen con filtros de color aplicados, ocultando el texto.

**Imagen 3 — miniatura2.jpeg con filtros aplicados (texto oculto)**
![Miniatura2 con filtros](imagenes/03-miniatura2-filtrada.jpg)

## Revelación de la flag
El texto estaba presente pero ofuscado por filtros. Usando una herramienta web de ajuste de color/contraste se reveló el contenido:

```
boomm
```

**Imagen 4 — Texto legible después de ajustar filtros/contraste**
![Flag revelada](imagenes/04-flag-revelada.jpg)

## Cálculo de la flag
Formato de entrega: `cidsi{md5}`

```bash
echo -n "boomm" | md5sum
```

```
1a0f70142919af1365e7b8b6de10d0f5
```

## Flag
```
cidsi{1a0f70142919af1365e7b8b6de10d0f5}
```

## Notas
- El reto se basó en extracción de miniaturas anidadas y no en `steghide`.
- La pista "Explotación de estenografía" apuntaba a EXIF/thumbnails más que a esteganografía clásica.
- Se usó una herramienta web para invertir/ajustar los filtros de la miniatura.
