# Write-up: Spy Games

## Reto

- Plataforma: CTIC / AGETIC
- Nombre: Spy Games
- Categoría: Reversing
- Puntos: 300
- Archivo: `spy_games`
- Estado: `resuelto y verificado localmente`

> Se ha conseguido una extraña cadena codificada en una partición oculta de un disco duro. ¿Puedes acceder a la información que está en él?

## Resumen

Aunque el archivo no tiene extensión y `file` solo lo clasifica como `data`, no es un ejecutable. Contiene una cadena de bits escrita como texto UTF-16BE.

Al retirar esa capa y convertir cada grupo de ocho bits a un byte aparece texto UTF-8 formado por caracteres como `䰀愀`. Estos caracteres tampoco son un idioma ni un cifrado: sus unidades UTF-16 tienen los bytes invertidos. Al corregir la endianess aparece el mensaje y la flag.

## Flag

```text
cidsi{3nc0d1ng_15_b1t_r3m4pp1ng}
```

## Herramientas utilizadas

- `file`: comprobar que no era un ELF ni otro formato reconocido.
- `xxd`: inspeccionar los bytes reales.
- `iconv`: convertir correctamente entre UTF-8, UTF-16BE y UTF-16LE.
- `basenc`: convertir una cadena binaria MSB-first a bytes.
- CyberChef: alternativa visual para reproducir las mismas capas.

No fue necesario Binary Ninja porque el archivo no contiene código ejecutable.

## 1. No confiar en el nombre ni en la categoría

Se empieza identificando el archivo:

```bash
file spy_games
```

Resultado:

```text
spy_games: data
```

`data` no significa que esté cifrado. Solo indica que `file` no reconoció una firma conocida. Antes de probar herramientas al azar se observan sus primeros bytes:

```bash
xxd -g 1 -l 64 spy_games
```

Salida relevante:

```text
00 31 00 31 00 31 00 30 00 30 00 31 00 30 00 30
00 31 00 30 00 31 00 31 00 30 00 30 00 30 00 30
```

Los bytes `31` y `30` son los caracteres ASCII `1` y `0`. Cada uno está precedido por `00`:

```text
00 31 = "1"
00 30 = "0"
```

Un carácter de dos bytes con el byte nulo a la izquierda es una señal típica de UTF-16 **big-endian**. No existe BOM al principio, así que `file` no pudo nombrarlo automáticamente; lo deducimos por el patrón.

## 2. Retirar UTF-16BE

Se convierte el texto a UTF-8 normal:

```bash
iconv -f UTF-16BE -t UTF-8 spy_games > capa1.bits
```

- `-f UTF-16BE`: formato de entrada (*from*).
- `-t UTF-8`: formato de salida (*to*).

Al observar el comienzo:

```bash
head -c 96 capa1.bits
```

aparece:

```text
111001001011000010000000111001101000010010000000...
```

La salida contiene solamente `0` y `1`, tiene 2424 caracteres y su longitud es divisible entre ocho. Por tanto, es razonable agruparla en bytes.

## 3. Convertir los bits a bytes

GNU Coreutils incluye `basenc`, que evita escribir un script:

```bash
basenc --base2msbf -d capa1.bits > capa2.utf8
```

Las opciones significan:

- `--base2msbf`: Base2 con el bit más significativo primero;
- `-d`: decodificar.

Se usa MSB-first porque cada grupo está escrito en el orden binario habitual. Por ejemplo:

```text
11100100 = 0xe4
```

Si se muestra `capa2.utf8`, aparecen caracteres extraños parecidos a:

```text
䰀愀 椀渀昀漀爀洀愀挀椀漀渀...
```

Esto es progreso: `basenc` produjo una secuencia UTF-8 válida. No conviene asumir que es chino ni probar César, Base64 o XOR; primero hay que inspeccionar su representación.

## 4. Reconocer el intercambio de bytes

Se convierten esos caracteres UTF-8 a unidades UTF-16 big-endian y se observan:

```bash
iconv -f UTF-8 -t UTF-16BE capa2.utf8 | xxd -g 2 | head
```

El comienzo es:

```text
4c00 6100 2000 6900 6e00 6600 6f00 7200...
```

Los valores ASCII conocidos están ahí, pero con `00` a la derecha:

```text
4c = L
61 = a
20 = espacio
69 = i
6e = n
```

Si `4c00` se interpreta como UTF-16BE representa `U+4C00`, el carácter `䰀`. Si los mismos dos bytes se interpretan como UTF-16LE representan `U+004C`, la letra `L`.

Por eso el problema no es una sustitución criptográfica: se usó el orden de bytes equivocado.

## 5. Corregir la endianess

Primero se materializan las unidades UTF-16BE:

```bash
iconv -f UTF-8 -t UTF-16BE capa2.utf8 > capa3.utf16
```

Después se interpretan esos mismos bytes como UTF-16LE:

```bash
iconv -f UTF-16LE -t UTF-8 capa3.utf16
```

Resultado:

```text
La informacion que necesitas para obtener el flag es
cidsi{3nc0d1ng_15_b1t_r3m4pp1ng}
Buen trabajo!
```

La pareja de comandos no “traduce chino”. Reescribe los caracteres como unidades de dos bytes y luego cambia cuál byte se considera más significativo.

## Comando completo

Una vez entendidas y comprobadas las capas, se pueden encadenar:

```bash
iconv -f UTF-16BE -t UTF-8 spy_games \
  | basenc --base2msbf -d \
  | iconv -f UTF-8 -t UTF-16BE \
  | iconv -f UTF-16LE -t UTF-8
```

Este comando es la versión resumida del procedimiento, no el punto de partida del análisis.

## Alternativa con CyberChef

La misma lógica puede reproducirse con operaciones equivalentes:

1. **Decode text** como `UTF-16BE`;
2. **From Binary**, usando ocho bits por byte;
3. decodificar esos bytes como `UTF-8`;
4. volver a codificar el texto como `UTF-16BE`;
5. decodificar los mismos bytes como `UTF-16LE`.

Según la versión de CyberChef, los nombres exactos de las operaciones de texto pueden variar. La señal que permite comprobar cada paso es:

```text
00 31 / 00 30 → bits → caracteres U+xx00 → ASCII legible
```

## Flujo de resolución

```text
archivo sin extensión
→ caracteres 0/1 en UTF-16BE
→ cadena binaria
→ bytes UTF-8
→ caracteres U+xx00
→ reinterpretar UTF-16BE como UTF-16LE
→ texto y flag
```

## Qué enseña el reto

- Una extensión ausente y la salida `data` no dicen qué transformación usar.
- Los bytes nulos alternados ayudan a reconocer UTF-16 y su endianess.
- Una cadena compuesta por `0` y `1` debe validarse por longitud antes de tratarla como bytes.
- Texto Unicode extraño puede contener un patrón útil en sus puntos de código.
- UTF-16BE y UTF-16LE usan los mismos bytes en distinto orden.
- Corregir una interpretación de texto no es lo mismo que romper un cifrado.
- La categoría Reversing también puede exigir reconstruir una cadena de transformaciones, aunque no exista un programa que desensamblar.

## Checklist reutilizable

Cuando `file` devuelve solamente `data`:

1. mirar el inicio con `xxd`;
2. buscar patrones de bytes nulos;
3. identificar si el contenido visible usa un alfabeto limitado;
4. comprobar si su longitud encaja con bloques de 8, 16 u otro tamaño;
5. decodificar una sola capa a la vez;
6. volver a usar `file`, `xxd` y una vista textual después de cada capa;
7. si aparece Unicode extraño, revisar sus unidades o puntos de código;
8. documentar el orden exacto de las transformaciones;
9. verificar que el resultado final es texto coherente y contiene la flag completa.
