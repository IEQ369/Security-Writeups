# Write-up: Piensa

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Forense
- Puntos: 330
- Archivo: `Piensa.xlsx`
- Tamaño: 40.324 bytes
- Estado: `resuelto; flag aceptada por la plataforma`

> El anverso de la moneda tiene un mensaje especial. Lo único que tienes que hacer es pensar con la cabeza y ordenar los cuadros.

## Resumen

Aunque el archivo tiene extensión `.xlsx`, no es un libro de Excel. Sus seis primeros bytes están dañados y comienzan con una imitación incompleta de la firma ELF. El resto de la estructura coincide con un GIF89a animado de 400×400: descriptor de pantalla, tabla global de 256 colores, extensiones GIF, marcador `NETSCAPE2.0` y seis fotogramas.

Después de restaurar la firma `GIF89a`, los seis fotogramas muestran fragmentos hexadecimales rotados. Al orientarlos y ordenarlos en sentido inverso, `5 → 4 → 3 → 2 → 1 → 0`, la concatenación decodifica como:

```text
It's pronounced JIF not GIF
```

## Reconocimiento inicial

```bash
file Piensa.xlsx
xxd -g 1 -l 128 Piensa.xlsx
unzip -t Piensa.xlsx
```

Resultados:

- `file` responde únicamente `data`.
- `unzip` indica que no existe una estructura ZIP válida.
- Un `.xlsx` normal debe comenzar con la firma ZIP `50 4b 03 04`; este archivo no la posee.
- Los primeros bytes son:

```text
7f 45 4c 3c 19 61 90 01 90 01 f7 00 00
```

## Por qué no es un ELF

Un ELF válido comienza con:

```text
7f 45 4c 46
         └─ F
```

El archivo comienza con:

```text
7f 45 4c 3c
         └─ no es F
```

Los tres primeros bytes parecen elegidos para sugerir ELF, pero falta el cuarto byte obligatorio y los campos siguientes tampoco forman una cabecera ELF válida. Cambiar la extensión a `.elf` no repararía el contenido.

## Cómo reconocer el GIF aunque su cabecera esté dañada

La firma normal de GIF89a ocupa seis bytes:

```text
ASCII: GIF89a
Hex:   47 49 46 38 39 61
```

Justo después de los seis bytes dañados aparecen:

```text
90 01 90 01 f7 00 00
```

Interpretación según la estructura GIF:

- `90 01`: ancho little-endian `0x0190` = 400.
- `90 01`: alto little-endian `0x0190` = 400.
- `f7`: existe una tabla global de colores de 256 entradas.
- `00`: índice del color de fondo.
- `00`: relación de aspecto sin especificar.

Desde el offset 13 aparece una tabla RGB en escala de grises:

```text
00 00 00
01 01 01
02 02 02
03 03 03
...
ff ff ff
```

Cada color tiene tres componentes: rojo, verde y azul. Que los tres valores sean iguales produce tonos de gris.

Después de las 256 entradas aparecen marcadores todavía más específicos:

```text
21 f9 04                 → Graphic Control Extension
21 ff 0b NETSCAPE2.0     → extensión de animación en bucle
2c                       → Image Descriptor de un fotograma
```

Esta combinación permite identificar el formato aunque `file` no pueda hacerlo. No se adivinó únicamente por los bytes iniciales: se comprobó la estructura completa.

## Reparar solamente una copia

Se conserva intacto el archivo original y se reemplazan únicamente sus seis primeros bytes por:

```text
47 49 46 38 39 61
 G  I  F  8  9  a
```

La copia reparada se reconoce como:

```text
GIF image data, version 89a, 400 x 400
```

Sus metadatos indican:

- seis fotogramas;
- 400×400 píxeles;
- 256 colores;
- duración total de 2,40 segundos;
- repetición infinita.

La pista «pensar con la cabeza» puede leerse como una referencia a la **cabecera** del archivo. «Ordenar los cuadros» apunta a separar y ordenar los fotogramas del GIF.

## Los seis fragmentos

Al separar los fotogramas y rotar el tercero 180 grados para leerlo correctamente, se obtienen:

| Fotograma | Fragmento leído |
| ---: | --- |
| 0 | `420474946` |
| 1 | `46206E6F7` |
| 2 | `564204A49` |
| 3 | `6F756E636` |
| 4 | `070726F6E` |
| 5 | `497427732` |

Si se extraen con FFmpeg:

```bash
ffmpeg -i Piensa_reparado.gif frame-%04d.png
```

FFmpeg comienza la numeración de los archivos en `0001`, mientras que el índice interno del GIF comienza en `0`. La equivalencia es:

| Índice interno | Archivo de FFmpeg | Fragmento |
| ---: | --- | --- |
| 0 | `frame-0001.png` | `420474946` |
| 1 | `frame-0002.png` | `46206E6F7` |
| 2 | `frame-0003.png` | `564204A49` |
| 3 | `frame-0004.png` | `6F756E636` |
| 4 | `frame-0005.png` | `070726F6E` |
| 5 | `frame-0006.png` | `497427732` |

Por tanto, expresado mediante los nombres creados por FFmpeg, el orden correcto es:

```text
frame-0006 → frame-0005 → frame-0004 → frame-0003 → frame-0002 → frame-0001
```

Se invierte el orden de los **fotogramas completos**. No se invierte el texto dentro de cada fragmento.

Cada fragmento tiene nueve dígitos hexadecimales. Como un byte necesita dos dígitos hexadecimales, no se deben decodificar los fragmentos por separado: cada uno deja medio byte pendiente que debe completarse con el fragmento correcto.

## Ordenarlos como una persona

El fotograma 5 proporciona un comienzo reconocible:

```text
49 74 27 73 2
 I  t  '  s  └─ medio byte pendiente
```

Para completar un espacio ASCII `0x20`, el siguiente fragmento debe comenzar con `0`. El fotograma 4 cumple esa condición:

```text
2 + 0 → 20 → espacio
70 72 6f 6e → pron
```

Después se continúa usando texto ASCII coherente y los medios bytes pendientes:

```text
fotograma 5 → It's + medio byte 2
fotograma 4 → 0 completa 20; después pron
fotograma 3 → ounc + medio byte 6
fotograma 2 → 5 completa 65 = e; después d JI
fotograma 1 → F no + medio byte 7
fotograma 0 → 4 completa 74 = t; después  GIF
```

El orden completo es:

```text
5 → 4 → 3 → 2 → 1 → 0
```

La concatenación hexadecimal queda:

```text
497427732070726F6E6F756E636564204A4946206E6F7420474946
```

## Decodificación

```bash
printf '%s' '497427732070726F6E6F756E636564204A4946206E6F7420474946' \
  | xxd -r -p
```

Resultado exacto, incluyendo el apóstrofo ASCII `0x27`:

```text
It's pronounced JIF not GIF
```

No existe otra carga añadida después del GIF: el archivo termina correctamente en el trailer `0x3b`. Tampoco aparecen comentarios o metadatos con otra flag. Por tanto, el mensaje reconstruido es el resultado sustentado por el archivo.

## Flag

Mensaje original:

```text
It's pronounced JIF not GIF
```

La plataforma aplica MD5 exacto sin salto de línea y utiliza el envoltorio `cidsi{}`:

```bash
printf %s "It's pronounced JIF not GIF" | md5sum
```

produce:

```text
84bc48897ee3a9cf66a5e2ac03e341b8
```

Flag aceptada:

```text
cidsi{84bc48897ee3a9cf66a5e2ac03e341b8}
```

## Cadena de resolución

`extensión .xlsx falsa → no existe ZIP → falsa apariencia ELF → estudiar bytes posteriores → estructura interna GIF89a → reparar firma en una copia → extraer seis frames → orientar el texto → fragmentos hex de nueve dígitos → encadenar medios bytes → orden 5,4,3,2,1,0 → From Hex → It's pronounced JIF not GIF`

## Qué debo recordar para otro CTF

- La extensión es solo un nombre; no determina el tipo real del archivo.
- Un `.xlsx` moderno es un contenedor ZIP y normalmente comienza con `PK`, bytes `50 4b 03 04`.
- La firma ELF necesita los cuatro bytes `7f 45 4c 46`; tres coincidencias no bastan.
- Una cabecera dañada puede identificarse estudiando dimensiones, paletas, marcadores internos y terminadores.
- Se debe reparar una copia, nunca sobrescribir la evidencia original.
- GIF puede contener varios fotogramas aunque al abrirlo solo parezca mostrar una imagen cambiante.
- Dos dígitos hexadecimales forman un byte. Fragmentos con cantidad impar de dígitos pueden dividir bytes entre piezas.
- Antes de automatizar todas las permutaciones, se pueden usar ASCII, espacios `20`, letras frecuentes y continuidad semántica para ordenar los fragmentos.
- Se valida el mensaje comprobando que todos los bytes resultantes sean imprimibles y formen una frase completa, no solo por una coincidencia aislada.

## Candidato para el arsenal forense

- Comparar extensión, salida de `file`, magic bytes y estructura esperada.
- Reconocer cabeceras parcialmente dañadas mediante marcadores internos de PNG, JPEG, GIF, ZIP, PDF, ELF y otros formatos frecuentes.
- Proponer reparaciones mínimas sobre una copia y mostrar los bytes cambiados.
- Para GIF/APNG/video, contar y extraer fotogramas con orden, duración y disposición.
- Detectar texto hexadecimal por fotograma y advertir cuando un fragmento tiene longitud impar.
- Probar orden temporal, orden inverso y permutaciones acotadas, conservando la procedencia de cada fragmento.
- Puntuar decodificaciones por ASCII imprimible, estructura de flag y lenguaje, sin declarar automáticamente correcta la primera cadena legible.
