# Write-up: Iconografía galletaria
## Reto
- Plataforma: CTF CGII/AGETIC
- Nombre: Iconografía galletaria
- Categoría: Esteganografía
- Archivo/objetivo: `Iconografia_galletaria`
- Estado: `resuelto`
## Reconocimiento
```bash
file Iconografia_galletaria
binwalk Iconografia_galletaria
```
**Hallazgo:** El archivo es reportado como `HIT archive data` y contiene un sistema de archivos SquashFS.
## Hipótesis
- [x] El contenedor contiene recursos web que deben inspeccionarse visualmente.
## Comandos y resultados
```bash
binwalk -e Iconografia_galletaria
find _Iconografia_galletaria.extracted -path '*/usr/www/images/*' -type f
```
**Resultado útil:** Dentro de `squashfs-root/usr/www/images/` aparece `ui-icons_cd0a0a_256x240.png`. La flag está escrita directamente en la imagen: `cidsi{b3bc634eaecec30651ca8b55eff512fe}`.
## Cadena de resolución
`archivo sin extensión → identificación de contenedor → binwalk → SquashFS → inventario de imágenes web → inspección visual → flag`
## Qué debo recordar para otro CTF
- Indicador que me hizo reconocer la técnica: una firma de archivo contenedor en lugar de una imagen normal.
- Herramienta que funcionó: `binwalk -e` y `find`.
- Comando mínimo: `binwalk -e Iconografia_galletaria`.
- Error/trampa: buscar esteganografía avanzada antes de revisar visualmente los archivos extraídos.

## Aporte para el arsenal

- Identificar contenedores aunque no tengan extensión y enumerar los archivos extraídos por tipo real.
- Crear una galería o lista corta de imágenes internas para inspección humana antes de técnicas más costosas.

## Capturas de la publicación original

<img width="256" height="240" alt="ui-icons_cd0a0a_256x240" src="https://github.com/user-attachments/assets/c8e464ae-bf48-4843-bc1b-1a66c181f037" />
