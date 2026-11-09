# Write-up: Cripto Avanzado

## Reto

- Plataforma: CTIC

- Categoría: Crypto

- Dificultad: Hard

- Fecha: 2026-09-01

- Estado: `resuelto`

- Tags: #crypto #base32 #md5 #xor #python #bruteforce


## Resumen

El reto entrega dos archivos, `ms1.txt` y `ms2.txt`, que deben combinarse para recuperar el mensaje final. `ms1.txt` contiene muchas líneas con apariencia de Base32 y `ms2.txt` contiene una secuencia binaria. La solución consiste en decodificar Base32, reconocer que los resultados son hashes MD5 de caracteres individuales, recuperar esos caracteres mediante un brute force muy pequeño y finalmente aplicar XOR entre dos secuencias binarias del mismo tamaño.

## Reconocimiento

Después de extraer el ZIP:

```bash
unzip Cripto_Avanzado.zip
```

se obtienen:

```text
ms1.txt
ms2.txt
```

Comprobamos los archivos:

```bash
file ms1.txt ms2.txt
```

Ambos son texto ASCII. Al observar `ms1.txt` aparecen líneas como:

```text
MI4WKY3FGE4GGOJVGBQWMYTGME3GEMDGMRRGMYJUMZTDOMZRMQZQU===
GI2TCMDDGM4TAMJRMM2WEZJXGA2DCOBSGQZDGZJTME3DSNLFHEYQU===
```

El alfabeto utilizado contiene `A-Z`, números `2-7` y padding `=`, lo cual es característico de **Base32**. Podemos comprobar una línea rápidamente:

```bash
echo 'MI4WKY3FGE4GGOJVGBQWMYTGME3GEMDGMRRGMYJUMZTDOMZRMQZQU===' | base32 -d
```

El resultado es:

```text
b9ece18c950afbfa6b0fdbfa4ff731d3
```

Tiene exactamente 32 caracteres hexadecimales, por lo que parece un hash **MD5**.

## Explotación / Procedimiento

### 1. Decodificar cada línea Base32

Cada línea de `ms1.txt` se decodifica de Base32 y produce un MD5:

```python
import base64
line = "MI4WKY3FGE4GGOJVGBQWMYTGME3GEMDGMRRGMYJUMZTDOMZRMQZQU==="
print(base64.b32decode(line).decode())
```

Salida:

```text
b9ece18c950afbfa6b0fdbfa4ff731d3
```

El problema ahora es descubrir qué representa cada MD5.

### 2. Notar que cada hash representa un solo carácter

No conviene intentar crackear cada MD5 con una wordlist enorme. Como el mensaje original probablemente está formado por caracteres ASCII, el espacio de búsqueda es muy pequeño: solo podemos probar los 256 posibles valores de un byte.\
La idea es calcular:

```text
MD5(0x00)
MD5(0x01)
MD5(0x02)
...
MD5(0xff)
```

y crear una tabla:

```text
hash MD5 -> carácter original
```

Por ejemplo:

```text
MD5("a") = 0cc175b9c0f1b6a831c399e269772661
```

Entonces, si uno de los hashes encontrados coincide con ese valor, sabemos que el carácter original era `a`.

### 3. Automatizar la recuperación de `ms1.txt`

Aquí sí merece la pena usar un script, porque hay cientos de líneas y hacerlo manualmente sería innecesariamente lento.

```python
import base64
import hashlib

# Crear tabla MD5 -> carácter
lookup = {}

for i in range(256):
    b = bytes([i])
    md5 = hashlib.md5(b).hexdigest()
    lookup[md5] = chr(i)

# Leer ms1.txt
with open("ms1.txt", "r") as f:
    lines = [line.strip() for line in f if line.strip()]

result = ""

for line in lines:
    # Base32 -> MD5
    md5_hash = base64.b32decode(line).decode()

    # MD5 -> carácter
    if md5_hash not in lookup:
        raise ValueError(f"No se pudo resolver: {md5_hash}")

    result += lookup[md5_hash]

print(result)
```

El resultado empieza con:

```text
TheFlagIsHiddenInsideThis:
```

y después contiene una larga secuencia de bits:

```text
TheFlagIsHiddenInsideThis:010100110011101100100011...
```

Esto indica que la información importante de `ms1.txt` no termina ahí: después de `:` hay un bitstream.

### 4. Separar el bitstream de `ms1`

Podemos separar ambas partes:

```python
prefix, bits1 = result.split(":", 1)

print(prefix)
print(len(bits1))
```

Resultado:

```text
TheFlagIsHiddenInsideThis
280
```

Tenemos exactamente **280 bits**.

### 5. Analizar `ms2.txt`

`ms2.txt` contiene únicamente unos y ceros. Sus líneas se concatenan:

```python
with open("ms2.txt", "r") as f:
    bits2 = "".join(line.strip() for line in f if line.strip())

print(len(bits2))
```

Resultado:

```text
280
```

Ahora tenemos:

```text
bits1 = 280 bits
bits2 = 280 bits
```

Que ambas secuencias tengan exactamente la misma longitud es una pista muy fuerte para probar **XOR**.

### 6. Aplicar XOR entre ambas secuencias

XOR funciona bit a bit:

```text
0 XOR 0 = 0
0 XOR 1 = 1
1 XOR 0 = 1
1 XOR 1 = 0
```

Podemos hacerlo con Python:

```python
xor_bits = "".join(
    "1" if a != b else "0"
    for a, b in zip(bits1, bits2)
)
```

### 7. Convertir los bits resultantes a ASCII

Como un byte son 8 bits, agrupamos de 8 en 8:

```python
plaintext = bytes(
    int(xor_bits[i:i+8], 2)
    for i in range(0, len(xor_bits), 8)
).decode()

print(plaintext)
```

Resultado:

```text
TheFlAgIs:BeThePersonWhoLovesCrypto
```

Por tanto, el mensaje final es:

```text
BeThePersonWhoLovesCrypto
```

### Script completo

```python
import base64
import hashlib

# ============================================================
# 1. Crear tabla MD5 -> byte
# ============================================================
lookup = {}

for i in range(256):
    value = bytes([i])
    lookup[hashlib.md5(value).hexdigest()] = chr(i)

# ============================================================
# 2. Resolver ms1.txt
# ============================================================
with open("ms1.txt", "r") as f:
    lines = [line.strip() for line in f if line.strip()]

decoded_ms1 = ""

for line in lines:
    md5_hash = base64.b32decode(line).decode()

    if md5_hash not in lookup:
        raise ValueError(f"Hash desconocido: {md5_hash}")

    decoded_ms1 += lookup[md5_hash]

print("[+] Resultado ms1:")
print(decoded_ms1)

# Separar el texto inicial del bitstream
prefix, bits1 = decoded_ms1.split(":", 1)

print(f"[+] Prefijo: {prefix}")
print(f"[+] Bits ms1: {len(bits1)}")

# ============================================================
# 3. Leer ms2.txt
# ============================================================
with open("ms2.txt", "r") as f:
    bits2 = "".join(line.strip() for line in f if line.strip())

print(f"[+] Bits ms2: {len(bits2)}")

if len(bits1) != len(bits2):
    raise ValueError("Los bitstreams no tienen la misma longitud")

# ============================================================
# 4. XOR
# ============================================================
xor_bits = "".join(
    "1" if a != b else "0"
    for a, b in zip(bits1, bits2)
)

# ============================================================
# 5. Binario -> ASCII
# ============================================================
plaintext = bytes(
    int(xor_bits[i:i+8], 2)
    for i in range(0, len(xor_bits), 8)
).decode()

print("[+] Mensaje final:")
print(plaintext)

# ============================================================
# 6. Calcular MD5 para formato de flag CTIC
# ============================================================
message = plaintext.split(":", 1)[1]

flag_hash = hashlib.md5(message.encode()).hexdigest()

print("[+] Mensaje:")
print(message)

print("[+] MD5:")
print(flag_hash)

print("[+] Flag:")
print(f"cidsi{{{flag_hash}}}")
```

Salida final:

```text
TheFlAgIs:BeThePersonWhoLovesCrypto
```

Mensaje utilizado para la flag:

```text
BeThePersonWhoLovesCrypto
```

MD5:

```text
4f4aa409bd95c74bfd79bbe48365c70a
```

## Flag

`cidsi{4f4aa409bd95c74bfd79bbe48365c70a}`

## Notas adicionales

La parte importante del reto es reconocer cada capa sin intentar ataques innecesarios. `A-Z`, `2-7` y `=` indican Base32; una salida de 32 caracteres hexadecimales sugiere MD5; muchos hashes MD5 distintos pueden representar caracteres individuales, por lo que un brute force de solo 256 bytes es suficiente; dos bitstreams exactamente del mismo tamaño hacen que XOR sea una prueba natural. Aquí el script sí es útil porque existen cientos de hashes y realizar las operaciones manualmente sería lento y propenso a errores. También es importante recordar que MD5 no se está “descifrando”: simplemente se calcula el hash de un dominio muy pequeño de posibles entradas y se comparan los resultados.

## Qué debo recordar para otro CTF

- Reconocer el alfabeto de Base32 antes de decodificar.
- Un hash puede recuperarse por comparación solamente cuando el espacio de entradas es pequeño y conocido.
- XOR entre secuencias requiere justificar longitud y alineación.

## Aporte para el arsenal

- Detectar listas de hashes y probar dominios pequeños configurables.
- Comparar bitstreams de igual longitud y reportar la alineación usada.
