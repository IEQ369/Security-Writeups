# Iconografía galletaria

- Plataforma: CTF CGII/AGETIC (ctf.cgii.gob.bo)
- Categoría: Esteganografía
- Puntos: 100
- nivel: Facil
- Fecha: 2026-06-07
- Tags: #steganography #binwalk #squashfs #png #visual

## Resumen
El archivo entregado es un contenedor que oculta un sistema de archivos. Dentro de él hay una imagen PNG con la flag escrita directamente en el diseño.

## Material
- `Iconografia_galletaria`

## Reconocimiento

Tipo de archivo:

```bash
file Iconografia_galletaria
```

Resultado: `HIT archive data`.

Extracción del contenido embebido:

```bash
binwalk -e Iconografia_galletaria
```

Se genera `_Iconografia_galletaria.extracted/124718/squashfs-root/`.

## Procedimiento

Ruta de interés dentro del filesystem extraído:

```bash
find _Iconografia_galletaria.extracted/124718/squashfs-root/usr/www/images -type f
```

Archivo relevante:
- `ui-icons_cd0a0a_256x240.png`

Se abre la imagen para inspección visual directa y se identifica la flag escrita en el PNG.

<img width="256" height="240" alt="ui-icons_cd0a0a_256x240" src="https://github.com/user-attachments/assets/c8e464ae-bf48-4843-bc1b-1a66c181f037" />


## Flag

```
cidsi{b3bc634eaecec30651ca8b55eff512fe}
```

## Notas
- No hubo esteganografía avanzada: la flag estaba visible en la imagen final.
- `binwalk -e` alcanzó para exponer el sistema de archivos oculto.
- La ruta `/usr/www/images` coincide con la idea del reto de "galletitas sin fondo" al tratarse de recursos web.
