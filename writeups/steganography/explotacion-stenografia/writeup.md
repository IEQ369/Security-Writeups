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
<img width="1600" height="1600" alt="Explotacion-de-stenografia" src="https://github.com/user-attachments/assets/46da44df-1fa9-4e62-afea-998f3c3ecbb8" />

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
<img width="528" height="360" alt="miniatura" src="https://github.com/user-attachments/assets/a40a7079-23c5-4abf-a2ee-eacaacc510f7" />

## Extracción de miniaturas
```bash
exiftool -ThumbnailImage -b Explotacion-de-stenografia.jpg > miniatura
mv miniatura miniatura.jpeg
exiftool -ThumbnailImage -b miniatura.jpeg > miniatura2
mv miniatura2 miniatura2.jpeg
```

La segunda miniatura (`miniatura2.jpeg`) mostraba una imagen con filtros de color aplicados, ocultando el texto.

**Imagen 3 — miniatura2.jpeg con filtros aplicados (texto oculto)**
<img width="1280" height="720" alt="miniatura2" src="https://github.com/user-attachments/assets/a68e814c-597a-4cce-8a78-369ebb5fc83f" />

## Revelación de la flag
El texto estaba presente pero ofuscado por filtros. Usando una herramienta web de ajuste de color/contraste se reveló el contenido:

```
boomm
```

**Imagen 4 — Texto legible después de ajustar filtros/contraste**
<img width="1280" height="720" alt="miniatura2" src="https://github.com/user-attachments/assets/682e30e2-7144-41fc-9b63-0e09af4fc961" />


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
