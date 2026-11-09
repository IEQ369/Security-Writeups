# Write-up: El tok del gato

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Forense
- Puntos: 300
- Archivo: `el_tok_del_gato.jpg`
- Estado: `resuelto`; flag aceptada por la plataforma

> “Susurros en la red hablan de un gato legendario, un felino que domina los secretos más profundos de la esteganografía. Se dice que esconde sus mensajes donde nadie pensaría buscar, en los espacios entre los espacios…”
>
> El mapa del gato:
>
> - Mira donde otros no ven — en los espacios entre los espacios.
> - Comienza tras la puerta 1000.
> - 200 pasos bastan.
> - 8 huellas, 1 palabra.
> - `00000000` marca el fin del camino.

## Resumen

Las pistas describen directamente un extractor de bits. Desde el offset decimal `1000` se toman como máximo `200` bytes del archivo. De cada byte se conserva su bit menos significativo o LSB, esos bits se agrupan de ocho en ocho en orden MSB-first y cada grupo se interpreta como un byte de texto. El primer byte nulo termina el mensaje.

El resultado exacto es:

```text
ViVVa BoliVia
```

## Reconocimiento inicial

```bash
file el_tok_del_gato.jpg
stat -c '%s bytes' el_tok_del_gato.jpg
exiftool el_tok_del_gato.jpg
binwalk el_tok_del_gato.jpg
strings -a -n 5 -t d el_tok_del_gato.jpg
```

Resultados relevantes:

- El archivo mide exactamente 25.000 bytes.
- Comienza con las firmas JPEG/JFIF `ff d8 ff e0` y termina con `ff d9`.
- `file` lo identifica como JPEG con datos EXIF/TIFF.
- ExifTool advierte `JPEG format error`.
- Los visualizadores y `steghide` no consiguen procesarlo como un JPEG válido.
- Binwalk no encuentra un archivo adicional incrustado que explique el reto.
- `strings` no revela el mensaje en texto claro.

Que la imagen esté dañada no impide estudiar sus bytes. Además, el enunciado proporciona offset, cantidad, agrupación y terminador; por tanto, insistir con diccionarios de Steghide o filtros visuales no sigue la pista disponible.

## Traducir “el mapa del gato” a operaciones

| Pista | Interpretación comprobada |
| --- | --- |
| «Espacios entre los espacios», «ruido» y «patrones» | Examinar bits no visibles de los bytes. |
| «Comienza tras la puerta 1000» | Empezar en el offset decimal `1000`. |
| «200 pasos bastan» | Procesar como máximo los siguientes `200` bytes. |
| «8 huellas, 1 palabra» | Cada byte de origen aporta un bit; ocho bits forman un carácter. |
| «El silencio del NULL» | Detenerse al reconstruir `00000000`, es decir, `0x00`. |

Aquí «1 palabra» debe entenderse como una unidad de ocho bits o byte, que en este mensaje representa un carácter.

## Descubrimiento manual

### 1. Mirar los bytes desde el offset 1000

El offset es decimal. En hexadecimal equivale a `0x3e8`:

```bash
xxd -g 1 -s 1000 -l 32 el_tok_del_gato.jpg
```

Los primeros 16 bytes de la región son:

```text
38 bf 62 69 bc 87 f7 4c  f4 d9 a9 26 91 1c 4e a5
```

### 2. Extraer el bit menos significativo

El LSB indica si cada byte es par (`0`) o impar (`1`). Para los primeros ocho bytes:

| Byte | Binario final | LSB |
| --- | --- | ---: |
| `38` | `...0` | 0 |
| `bf` | `...1` | 1 |
| `62` | `...0` | 0 |
| `69` | `...1` | 1 |
| `bc` | `...0` | 0 |
| `87` | `...1` | 1 |
| `f7` | `...1` | 1 |
| `4c` | `...0` | 0 |

Al unirlos en el mismo orden:

```text
01010110 → 0x56 → V
```

Los ocho bytes siguientes producen:

```text
01101001 → 0x69 → i
```

La pista queda validada inmediatamente porque los primeros dieciséis bytes de origen producen `Vi`.

### 3. Continuar hasta el byte nulo

Los primeros grupos reconstruidos son:

```text
01010110 01101001 01010110 01010110
01100001 00100000 01000010 01101111
01101100 01101001 01010110 01101001
01100001 00000000
```

Interpretados como bytes:

```text
56 69 56 56 61 20 42 6f 6c 69 56 69 61 00
 V  i  V  V  a     B  o  l  i  V  i  a NULL
```

El mensaje termina después de 13 caracteres. Para llegar hasta el terminador se consumieron `14 × 8 = 112` de los 200 bytes permitidos; los 88 restantes no pertenecen al mensaje y se ignoran porque la pista ordena detenerse en `00000000`.

## Extractor reproducible

El siguiente programa implementa exactamente las pistas, sin examinar píxeles ni probar combinaciones arbitrarias:

```python
from pathlib import Path

datos = Path("el_tok_del_gato.jpg").read_bytes()
region = datos[1000:1000 + 200]

bits = [byte & 1 for byte in region]
mensaje = bytearray()

for posicion in range(0, len(bits) - 7, 8):
    grupo = bits[posicion:posicion + 8]

    valor = 0
    for bit in grupo:
        valor = (valor << 1) | bit

    if valor == 0:
        break

    mensaje.append(valor)

print(mensaje.decode("ascii"))
```

Salida:

```text
ViVVa BoliVia
```

Qué hace el núcleo:

- `byte & 1` conserva únicamente el último bit.
- Se usan ocho LSB consecutivos por carácter.
- `valor = (valor << 1) | bit` agrega los bits de izquierda a derecha.
- `valor == 0` aplica el terminador indicado por el reto.
- Se decodifica como ASCII solamente después de obtener bytes completos.

## Verificación de orientación y offset

La misma región puede producir basura si se cambia una decisión sin evidencia:

- Empezar en `999` desplaza todos los grupos un bit.
- Empezar en `1001` también destruye la alineación.
- Usar `0x1000` significa offset decimal `4096`, que no es lo indicado.
- Invertir los ocho bits de cada grupo no produce texto coherente.
- Usar el segundo plano de bits en lugar del LSB tampoco produce el mensaje.

No se eligió la variante correcta solamente porque «se veía bonita»: coincide con todos los parámetros del enunciado, comienza a producir texto desde el primer grupo y termina con el byte nulo prometido.

## Mensaje y flag confirmada

Mensaje confirmado directamente por la evidencia:

```text
ViVVa BoliVia
```

Las mayúsculas y el espacio deben conservarse. Corregirlo a `Viva Bolivia`, quitar el espacio o cambiar el caso genera otro valor.

MD5 del mensaje exacto, sin salto de línea:

```bash
printf %s 'ViVVa BoliVia' | md5sum
```

Resultado:

```text
e09c1acc507ad88af1082ff3480978bf
```

La plataforma confirmó que esta edición utiliza el prefijo `citc` y el MD5 del mensaje exacto:

```text
citc{e09c1acc507ad88af1082ff3480978bf}
```

La aceptación también confirma que no debía corregirse la escritura a `Viva Bolivia`, retirar el espacio ni convertir el texto a minúsculas.

## Flag

```text
citc{e09c1acc507ad88af1082ff3480978bf}
```

## Cadena de resolución

`JPEG estructuralmente dañado → interpretar las pistas como parámetros → offset decimal 1000 → leer hasta 200 bytes → LSB de cada byte → grupos de 8 MSB-first → ASCII → detenerse en NULL → ViVVa BoliVia → MD5 si lo exige la plataforma`

## Qué debo recordar para otro CTF

- LSB no significa siempre «LSB de píxeles». También se puede ocultar información en los bits de los bytes crudos de cualquier archivo.
- `zsteg` está orientado principalmente a canales y estructuras PNG/BMP; no sustituye un extractor de bytes parametrizado para un JPEG dañado.
- Un offset puede estar expresado en decimal o hexadecimal. Se conserva literalmente lo indicado y se prueban vecinos solo como comprobación de alineación.
- Si cada byte aporta un bit, hacen falta ocho bytes de evidencia por byte reconstruido.
- El orden de bits importa: aquí se leen los LSB en orden de archivo y cada grupo representa el byte de más significativo a menos significativo.
- Un terminador nulo evita interpretar como mensaje el ruido situado después.
- Se debe conservar exactamente capitalización, espacios y caracteres de control antes de calcular un hash.
- Que un archivo no pueda renderizarse no significa que carezca de evidencia; `file`, `xxd` y lectura binaria todavía funcionan.
- Las herramientas automáticas comunes pueden fallar cuando el reto usa un esquema personalizado muy pequeño.

## Candidato para el arsenal forense

- Incorporar un extractor `raw_lsb` independiente de Pillow y del formato visual.
- Parámetros: offset decimal/hexadecimal, cantidad máxima de bytes, plano de bits, tamaño de grupo, orden MSB/LSB y terminador.
- Mostrar los primeros grupos como `bytes origen → bits → valor → carácter` para explicar el resultado.
- Probar offsets vecinos y ambos órdenes solamente como diagnósticos acotados, no recorrer todo el archivo sin límites.
- Detectar terminadores `00`, texto imprimible y magic bytes, conservando la salida cruda aunque no sea texto.
- Diferenciar LSB sobre bytes del archivo, LSB sobre píxeles decodificados y esteganografía sobre coeficientes JPEG: son canales distintos.
- Registrar offset, rango consumido, bit plane, orientación y condición de parada en cada candidato.
- Nunca normalizar mayúsculas o espacios antes de guardar y hashear el mensaje original.
