# Último Umbral

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Esteganografía. Puntos: 160. Autor: F7ASH.
- Archivo: ![ultimo_umbral.png](imagenes/ultimo-umbral.png).
- Estado: **parcial; segunda carga extraída, sin descifrar**.

> Sombra archivaba cada operación con las siglas de la campaña seguidas del año, un guion y el nombre de operación en minúsculas, sin tildes y con guiones. En su siguiente envío dejó una pista real y una trampa para quien confirmara la primera respuesta con forma de flag.

## Resumen

El plano rojo de bit 1 muestra `TINIEBLA` y `citc2026-ultimo-umbral`. No son banderas aceptadas. `TINIEBLA` sí sirve para extraer dos cargas del LSB azul: una secuencial, que es un señuelo legible, y otra con posiciones reordenadas, de 68 bytes todavía cifrados.

## Reconocimiento e hipótesis

```bash
file ultimo_umbral.png
exiftool ultimo_umbral.png
binwalk ultimo_umbral.png
zsteg ultimo_umbral.png
```

PNG RGB de 1600 × 1000. No había datos tras IEND ni metadatos de texto. Las detecciones genéricas de OpenPGP de `zsteg` no se validaron y no deben considerarse archivos recuperados.

## Resolución parcial paso a paso

### 1. Leer el plano visible

En StegOnline: abrir imagen → **Browse Bit Planes** → canal rojo, bit 1 contando desde 0. Se lee **TINIEBLA**, singular, y abajo `citc2026-ultimo-umbral`. La primera interpretación de esos textos como respuesta final fue incorrecta.

### 2. Extraer el LSB azul secuencial

```bash
zsteg -E b1,b,lsb,xy ultimo_umbral.png | xxd -l 48
```

La cabecera `00 27` indica 39 bytes. Aplicando XOR repetido con `TINIEBLA` a esos 39 bytes aparece:

```text
citc{c451_l0_l0gr45_s1gu3_1nt3nt4nd0lo}
```

Es el **señuelo**, no la respuesta. El contenido ya dice «casi lo logras, sigue intentándolo».

### 3. Recuperar el segundo bloque

El azul también tenía cambios dispersos. Se probó una permutación reproducible de los índices de píxel con `random.Random('TINIEBLA').shuffle`. Sus primeras posiciones concentran los cambios y generan una cabecera válida: `00 44`, longitud **68**.

Este bloque reproduce ambas extracciones sin modificar la imagen:

```bash
python3 - <<'PY'
from PIL import Image
import random
pixels = list(Image.open('ultimo_umbral.png').convert('RGB').getdata())
def extraer(indices, cantidad):
    bits = [pixels[i][2] & 1 for i in indices[:cantidad*8]]
    return bytes(sum(bits[i+j] << (7-j) for j in range(8))
                 for i in range(0, len(bits)-7, 8))
indices = list(range(len(pixels)))
data = extraer(indices, 41)
n = int.from_bytes(data[:2], 'big')
key = b'TINIEBLA'
print('Señuelo:', bytes(v ^ key[i % len(key)]
      for i, v in enumerate(data[2:2+n])).decode())
random.Random('TINIEBLA').shuffle(indices)
data = extraer(indices, 70)
n = int.from_bytes(data[:2], 'big')
print('Segunda longitud:', n)
print('Segunda carga:', data[2:2+n].hex())
PY
```

Carga obtenida:

```text
3c87197c39af3cf2a3bedea21eb1084153792def0d64fcb78eda67fb31cb013c73979848f9cdfd59c6ce566b94b16dc82e36b4b167ddfe1323d4c0ba1761693654f18a2b
```

### 4. Punto pendiente

El XOR repetido con los textos visibles y varias variantes de SHA-256/MD5 no produjo texto claro. Tampoco se validaron los ensayos de AES-GCM/ChaCha20-Poly1305 con las disposiciones probadas. **Eso no identifica el cifrado ni descarta todos sus parámetros.** El siguiente trabajo debe centrarse en justificar el cifrado y la derivación de clave, no volver a probar las banderas rechazadas.

## Cadena de resolución

`PNG → plano rojo → TINIEBLA → azul secuencial (señuelo) / azul permutado (68 bytes) → descifrado pendiente`

## Flag

**No recuperada.** No presentar la carga de 68 bytes, su MD5 ni el señuelo como solución.

## Qué debo recordar para otro CTF

Una palabra visible puede ser una semilla o clave. Encontrar `citc{...}` no basta: el reto advertía explícitamente sobre una trampa.

## Aporte para el arsenal

Separar texto visible, carga secuencial y carga permutada. Registrar semilla, canal, recorrido, cabecera y bytes exactos para retomar sin repetir pruebas.
