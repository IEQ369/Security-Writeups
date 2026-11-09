# Write-up: El Cementerio de Bytes
## Reto
- Plataforma: CTF SCESI
- Nombre: El Cementerio de Bytes
- Categoría: Forensics
- Puntos: 200
- Archivo/objetivo: `6daefb26a764cc5c3774510f1ed22df5.raw`
- Estado: `resuelto`
## Reconocimiento
```bash
file 6daefb26a764cc5c3774510f1ed22df5.raw
mmls 6daefb26a764cc5c3774510f1ed22df5.raw
fsstat 6daefb26a764cc5c3774510f1ed22df5.raw
```
**Hallazgo:** Es un volcado de 4 MiB clasificado simplemente como `data`. `mmls` no reconoce una tabla de particiones y `fsstat` no reconoce un sistema de archivos; advierte entropía alta. Por tanto, intentar montarlo no es la primera opción: corresponde buscar firmas y tallar archivos (*file carving*).
## Descubrimiento humano, paso a paso
### 1. Buscar firmas internas
```bash
binwalk 6daefb26a764cc5c3774510f1ed22df5.raw
```
Resultado relevante:
```text
500000   0x7A120   PNG image, 8 x 8, RGB
2600000  0x27AC40  Zip archive data, encrypted, name: flag.txt
2600205  0x27AD0D  End of Zip archive
```
No son coincidencias aisladas: aparecen una cabecera PNG completa y un ZIP con su directorio central y registro final.
### 2. Recuperar los objetos
```bash
foremost -i 6daefb26a764cc5c3774510f1ed22df5.raw -o recuperados
file recuperados/png/* recuperados/zip/*
```
Foremost recupera:
- Un PNG válido de 120 bytes y 8×8 píxeles.
- Un ZIP válido de 227 bytes con `flag.txt`, cifrado y sin comprimir.

Que la imagen sea diminuta no permite descartarla. Al contrario, un contenedor tan pequeño puede existir solamente para transportar metadatos o unos pocos bits.
### 3. Revisar la estructura de la imagen antes de buscar LSB
```bash
exiftool recuperados/png/00000976.png
pngcheck -v recuperados/png/00000976.png
```
ExifTool muestra:
```text
Comment : bzFhNGVsX3M0YWduZno0XzIwMjY=
```
`pngcheck` ubica ese valor en un chunk `tEXt` con palabra clave `Comment`. `zsteg` también puede mostrar `meta Comment`, pero se trata del mismo metadato, no de una segunda prueba ni de información extraída mediante LSB. Ejecutar todas sus combinaciones con `zsteg -a` produce mucho ruido innecesario para este PNG.
### 4. Decodificar la primera capa
La cadena tiene alfabeto Base64, longitud múltiplo de cuatro y padding `=`:
```bash
printf '%s' 'bzFhNGVsX3M0YWduZno0XzIwMjY=' | base64 -d
```
Resultado:
```text
o1a4el_s4agnfz4_2026
```
El resultado es totalmente imprimible y tiene forma de contraseña, pero no forma palabras claras. Probarlo directamente contra el ZIP falla, por lo que Base64 era solamente una capa.
### 5. Reconocer ROT13 sin depender de una prueba aleatoria
Hay varias pistas:
- Los `_` separan aparentes palabras.
- `2026` parece intencional y debe conservarse.
- ROT13 modifica solamente letras; no altera dígitos ni guiones bajos.
- La salida de Base64 parece texto desplazado, no datos cifrados de alta entropía.

Se prueba entonces ROT13:
```bash
printf '%s' 'o1a4el_s4agnfz4_2026' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```
Resultado:
```text
b1n4ry_f4ntasm4_2026
```
Ahora se reconocen `binary_fantasma_2026` escrito en *leet*, una frase coherente con el tema del reto.
### 6. Validar la contraseña contra el segundo artefacto
```bash
unzip -P 'b1n4ry_f4ntasm4_2026' -p recuperados/zip/00005078.zip flag.txt
```
Resultado:
```text
umssctf{c4rv1ng_y_m3t4d4t0s_ftw}
```
Esta prueba convierte una cadena plausible en contraseña confirmada. El texto extraído tiene 33 bytes, coincide con el tamaño declarado para `flag.txt` y posee el formato correcto de la competencia.
## Cadena de resolución
`RAW sin sistema de archivos → firmas PNG/ZIP → carving → chunk PNG tEXt/Comment → Base64 → texto casi legible → ROT13 → contraseña leet → abrir ZIP → flag`
## Flag
`umssctf{c4rv1ng_y_m3t4d4t0s_ftw}`
## Qué debo recordar para otro CTF
- Un `.raw` no montable todavía puede contener archivos completos recuperables por sus firmas.
- Primero se inventarían todas las firmas; no se investiga solo el primer objeto recuperado.
- El tamaño visual de una imagen no determina cuánta importancia tienen sus metadatos.
- ExifTool y `zsteg meta` pueden estar mostrando exactamente el mismo chunk; dos herramientas no implican dos hallazgos independientes.
- Una decodificación correcta no siempre termina la cadena. Si produce texto imprimible con estructura pero sin semántica clara, debe reevaluarse con transformaciones baratas y justificables.
- ROT13 conserva números, `_` y signos. Esa invariancia ayuda a reconocer contraseñas con años o separadores.
- Una frase que parece contraseña sigue siendo candidata hasta que abre el contenedor. La validación cruzada entre artefactos es más fuerte que cualquier puntuación lingüística.
## Hallazgo y corrección de `crypto_triage.py`
Antes de la corrección, al entregarle solamente el comentario Base64, el script sí generaba internamente:
```text
Base64 decode → ROT13 → b1n4ry_f4ntasm4_2026
```
Sin embargo, la ruta obtuvo una puntuación de `158.2` y quedó fuera de los 15 resultados mostrados. Varias cadenas basura imprimibles recibieron aproximadamente `199.2`. El problema observado era de ranking y presentación, no la ausencia del decodificador ROT13.

Corrección aplicada al arsenal el 2026-09-03:
- `Base64 decode → ROT13` aparece ahora como resultado fuerte con la salida original intacta.
- Explica las interpretaciones `b1n4ry→binary` y `f4ntasm4→fantasma` como señales usadas solamente para *scoring*.
- `1` genera alternativas acotadas `i/l`; los números también se conservan como literales para no alterar términos como `Base64`, hashes, versiones o años.
- Una sola palabra reconocida no basta para declarar texto fuerte, y una cadena *leet* aleatoria sigue siendo rechazada.
- Se eliminó `XOR 0x00`, que no cambiaba el contenido, y se corrigió una iteración adicional innecesaria. Este caso bajó aproximadamente de 14 a 3,4 segundos.
- Se agregó el reto como prueba de regresión y todas las pruebas anteriores continúan pasando.

Actualización del 2026-09-06: `crypto-general/layered_decode.py` ya existe y reutiliza el motor de `crypto_triage.py` mediante funciones. Conecta capas textuales, matrices QR y contenedores OpenSSL `Salted__`; la extracción independiente de un `crypto_core/` todavía queda como refactor futuro. Este reto continúa cubierto por la ruta directa `Base64 decode → ROT13`, y su prueba de regresión sigue pasando.
## Candidato para el arsenal forense
- El triage de RAW debe ejecutar identificación de particiones/sistema de archivos, búsqueda de firmas y carving controlado.
- Cada objeto recuperado debe conservar offset, tamaño, hash y relación con el RAW padre.
- Aplicar automáticamente metadatos y validación estructural según el tipo recuperado; para PNG, diferenciar chunks de metadatos de resultados LSB.
- Pasar cadenas de metadatos al motor común de decodificación por capas.
- Construir relaciones entre artefactos hermanos: una cadena obtenida del PNG puede probarse de forma segura como contraseña del ZIP recuperado.
- Conservar la cadena original, cada transformación y la prueba que convierte un candidato en hallazgo confirmado.
