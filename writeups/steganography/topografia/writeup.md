# Topografía

- Plataforma: CTF CGII/AGETIC (ctf.cgii.gob.bo)
- Categoría: Esteganografía
- Puntos: 300
- Fecha: 2026-05-31
- Tags: #steganography #jpeg #steghide #obfuscation

## Resumen
Reto de esteganografía donde la flag estaba oculta dentro de una imagen JPEG mediante `steghide`. La pista final implicaba eliminar el prefijo `CGII` de una cadena y convertir el resultado a MD5.

## Material
- `Topografia.jpg` (1200x1200, JPEG)

## Reconocimiento
```bash
exiftool Topografia.jpg
```
Datos relevantes:
- MIME type: `image/jpeg`
- Copyright: "Aqui no hay nada... AGETIC te puede ser de utilidad"

## Extracción
Ejecutamos `steghide` contra la imagen. Pide passphrase; en este reto la deja vacía.

```bash
steghide extract -sf Topografia.jpg
```
Archivo generado: `Oculto.txt`

## Contenido oculto
```
Que bueno eres!!!
Ahora toma tu premio...

INVCGIIESTCGIIIGCGIIAPCGIIARCGIIASCGIIERMCGIIEJCGIIORYPRCGIIACTCGIIICCGIIACCGIIADCGIIADCGIIIA

Lo puedes ver?
Puedes utilizar las iniciales del área Centro de Gestión de Insidentes Informáticos de la AGETIC para poder ver la Flag ;)
```

## Procedimiento
La cadena with `CGII` como prefijo repetido representa un cifrado por sustitución/inserción. Aplicando la pista del enunciado y eliminando el token `CGII` de toda la secuencia se obtiene:

```
INVESTIGAPARASERMEJORYPRACTICACADADIA
```

Según el formato requerido por la plataforma (`cidsi{hash}` o `citc{hash}`), convertimos el texto a MD5:

```
d58054ab9b51e1750e5c361aac27e8e0
```

## Flag
```
cidsi{d58054ab9b51e1750e5c361aac27e8e0}
citc{d58054ab9b51e1750e5c361aac27e8e0}
```

