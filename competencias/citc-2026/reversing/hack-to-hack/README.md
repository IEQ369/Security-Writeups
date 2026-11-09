# Hack to Hack

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Reversing / criptografía simple. Puntos: 450. Autor: C0nsult0r 03.
- Archivos: WannaFlag_Decryptor.exe (PE de 64 bits para Windows, 271 660 bytes) y flag_real.txt.enc (29 bytes). Las copias están en esta carpeta; en la competencia se trabajó sobre copias de Descargas, sin modificar los originales.
- Estado: **resuelto localmente**. El archivo se descifra y empieza por `flag{`; no consta aceptación de la plataforma ni se ejecutó el `.exe`.
- Registros breves anteriores: SOLVE y STATE.

> Un grupo de ransomware dejó un `.txt` cifrado y el ejecutable que usaban para recuperarlo. El programa acepta `--key`, pero no se sabe la clave correcta. Objetivo: encontrar la clave, usarla con el ejecutable y descifrar el archivo.

## Resumen

El `.exe` no compara la clave con una contraseña fija: **la convierte en un solo byte** (suma de sus bytes, módulo 256, XOR `0x34`) y usa ese byte para generar una llave de tres valores que se repite. Hay muchísimas claves válidas. Además, contiene una clave trampa (`1A2B3C`) que imprime una flag falsa. Como el archivo cifrado debe empezar por `flag{`, se puede recuperar ese byte sin ejecutar nada y descifrar con un script de cuatro líneas.

## Qué recibimos y qué buscaba el reto

Un ejecutable y un archivo binario. El enunciado habla de «descubrir la clave» y de «usarla con el ejecutable», lo que sugiere que la clave importa. Pero la entrega es descifrar el contenido; la clave es un medio.

## Reconocimiento

```bash
cd Hack-to-Hack
file WannaFlag_Decryptor.exe flag_real.txt.enc
xxd flag_real.txt.enc
strings -n 5 WannaFlag_Decryptor.exe | grep -iE 'key|flag|usage|Uso|xor|\.enc'
```

**Salida que importa:**

```text
WannaFlag_Decryptor.exe: PE32+ executable (console) x86-64, for MS Windows
flag_real.txt.enc:       data
00000000: 1b52 9b1a 45c9 0e4a ce49 0ace 220d c94e  .R..E..J.I.."..N
00000010: 4da5 310a ce49 0aa5 3b52 9b1a 43         M.1..I..;R..C
--key
Uso: %s --key <clave>
flag{t3_cr3ist3_3l_c3b0_d3_l4_IA}
_Z14derive_xor_keyPKc   _Z17decrypt_and_printPKch   _Z11try_decryptPKc
```

**Interpretación:**

- `file` dice «data»: el `.enc` no tiene cabecera reconocible; son bytes cifrados.
- Mide 29 bytes, igual que una cadena corta del tipo `flag{...}`.
- Los nombres `derive_xor_key` (derivar clave XOR) y `decrypt_and_print` delatan el esquema: **XOR**. Son nombres de símbolos C++ que el compilador dejó.
- `strings` muestra ya una cadena `flag{t3_cr3ist3...}` dentro del programa. Una flag visible sin descifrar nada es sospechosa: el archivo se llama `flag_real`, de modo que esa debe ser un señuelo. Se confirma en el paso 1.
- Si el binario no se puede ejecutar de forma segura, se desensambla. Aquí se hizo **análisis estático**; no hay `wine` en la máquina y las reglas del equipo piden aislamiento para ejecutar desconocidos.

## Resolución paso a paso

### 1. Desensamblar y ubicar las funciones

```bash
objdump -d -M intel --no-show-raw-insn WannaFlag_Decryptor.exe > /tmp/h2h.txt
awk '/<_Z23verificar_clave_secretaPKc>:/,/<_Z14derive_xor_keyPKc>:/' /tmp/h2h.txt | head -60
objdump -s -j .rdata --start-address=0x14000b000 --stop-address=0x14000b0f0 WannaFlag_Decryptor.exe
```

`objdump -d` desensambla; `-M intel` usa la sintaxis Intel, más legible; el `awk` imprime solo el rango entre dos etiquetas. En `verificar_clave_secreta` se ve una llamada a `strcmp` contra una cadena de `.rdata`, y el volcado de `.rdata` muestra:

```text
14000b000  31413242 33430000 666c6167 7b74335f   1A2B3C..flag{t3_
14000b010  63723369 7374335f 336c5f63 3362305f   cr3ist3_3l_c3b0_
14000b020  64335f6c 345f4941 7d00                d3_l4_IA}.
```

**Conclusión:** si la clave es exactamente `1A2B3C`, el programa imprime «Acceso concedido» y la flag `flag{t3_cr3ist3_3l_c3b0_d3_l4_IA}` y **sale**. Es el señuelo: se encuentra rápido con `strings`, y no sale del archivo `.enc`. No usarla.

### 2. Ver qué hace `try_decrypt`

Esta función encadena: `verificar_clave_secreta` (señuelo) → `check_attempts` → `derive_xor_key(clave)` → `decrypt_and_print(clave_derivada)`. `check_attempts` limita a unos pocos intentos por sesión («Demasiados intentos…») y hace esperas. **Consecuencia:** probar claves a mano contra el `.exe` no es práctico; hay que entender el cálculo.

### 3. Entender `derive_xor_key`

El desensamblado es largo y está lleno de operaciones que no sirven para nada (constantes como `0x9e3779b1`, `0xcafebabe`, `0xdeadbeef`, `ror`, `shr`). Se guardan en variables que luego no se usan: es ofuscación. La parte relevante se identifica así:

- Hay un bucle sobre los caracteres de la clave con `add BYTE PTR [rbp-0x1], al`: **suma cada byte de la clave en un acumulador de 8 bits** (por eso es módulo 256).
- Al final: `xor eax, 0x34` y devuelve ese valor: `k = (suma de bytes mod 256) ^ 0x34`.
- Hay tres comprobaciones que parecen condiciones («si ... entonces»), pero siempre se cumplen: `(n·n + n) & 1 == 0` (producto de dos consecutivos es par), `(c³ − c) % 6 == 0` (producto de tres consecutivos) y `(s⁴ − s²) % 12 == 0`. Son «predicados opacos»: engañan al lector, pero nunca desvían el flujo. Sin ese análisis se perdería tiempo buscando una rama que nunca ocurre.

### 4. Ver cómo se usa `k` en `decrypt_and_print`

Dentro del bucle sobre el archivo se ve una división por 3 (`imul`/`shr`/`add rax,rax`… calculan `i mod 3`) y tres variantes del byte de llave, con `shr al,1` y `add eax,eax`:

| `i mod 3` | Byte de llave |
|---|---|
| 0 | `k` |
| 1 | `k >> 1` |
| 2 | `(k * 2) & 0xFF` |

Cada byte cifrado se hace XOR con ese byte. Al final hay una `memcmp` con `flag{` (en `.rdata`), es decir, el programa comprueba que el resultado empiece por `flag{`.

### 5. Recuperar `k` sin conocer la clave

**Observación:** XOR es reversible: `texto = cifrado XOR llave`. Se conoce el principio del texto: `flag{`. Con el primer byte (`i = 0`, llave = `k`): `k = 0x1b XOR 'f'`.

Con este script (guardado como verificar_clave.py; ejecutar con `python3 verificar_clave.py` desde esta carpeta):

```python
d = open('flag_real.txt.enc', 'rb').read()

def descifrar(d, k):
    ks = [k, k >> 1, (k * 2) & 0xFF]          # llave por posición: i % 3
    return bytes(b ^ ks[i % 3] for i, b in enumerate(d))

for k in range(256):                           # probar los 256 valores de k
    t = descifrar(d, k)
    if t.startswith(b'flag{'):
        print('k =', hex(k), t)
        print('suma de bytes de la clave mod 256 =', k ^ 0x34)

for clave in ('I', '1A2B3C'):
    print(repr(clave), '-> k =', hex((sum(clave.encode()) % 256) ^ 0x34))
```

El script resuelve dos cosas: (1) probar los 256 valores posibles de `k` sin depender de adivinar nada, y (2) calcular qué clave de texto produce ese `k`. Salida real (ejecutada el 4 de octubre):

```text
k = 0x7d b'flag{3st4444_333s_L4444_Flag}'
suma de bytes de la clave mod 256 = 73 -> I (un solo carácter sirve)
'I' -> k = 0x7d
'1A2B3C' -> k = 0x68
```

Solo `k = 0x7d` da un texto con sentido; el resto no empieza por `flag{`. El registro original usaba `solve.py`, que hace lo mismo con la hipótesis `k = primer byte ^ 'f'`.

### 6. La clave para el ejecutable

`k = 0x7d` equivale a una suma de bytes `0x7d ^ 0x34 = 0x49 = 73`, que coincide con el carácter `I`. Es decir, `--key I` **debería** producir el mismo descifrado, igual que cualquier cadena con suma congruente a 73 módulo 256. No se ejecutó el `.exe`; esta parte es una deducción desde el código.

## Cadena de resolución

`strings: flag falsa visible → desensamblar → 1A2B3C = señuelo → derive_xor_key = suma mod 256 ^ 0x34 → keystream de 3 bytes → conocer flag{ → k = 0x7d → descifrar`

## Flag

```text
flag{3st4444_333s_L4444_Flag}
```

Estado: **resuelta localmente** (el texto descifrado empieza por `flag{`). No se probó con el `.exe` y no consta aceptación. Mismas letras repetidas (`4444`, `333`) parecen intencionales del reto, no un fallo del descifrado.

## Qué debo recordar para otro CTF

- Una flag legible dentro del binario, o una clave «demasiado fácil» (`1A2B3C`), suele ser señuelo.
- XOR con prefijo conocido (`flag{`): se recupera la llave sin saber la clave.
- Si el desensamblado tiene mucho ruido (constantes mágicas, condiciones que nunca cambian), busca lo que **sí** se usa al final: el valor devuelto.
- Un bucle `add BYTE PTR [...]` es una suma módulo 256; muchas claves distintas valen igual.

## Aporte para el arsenal

- Automatizable: probar los 256 bytes de llave para formatos `flag{`/`citc{` y para llaves periódicas de 1 a 4 bytes.
- Requiere criterio humano: reconocer predicados opacos y distinguir el señuelo.
- Falso positivo: dar por buena la flag que `strings` encuentra en `.rdata`.


## Código de apoyo: verificar_clave.py

Guarda este bloque como `verificar_clave.py` si sigues los comandos que lo utilizan. Código del análisis original; no se ha vuelto a ejecutar para esta publicación.

```python
# Reproduce el algoritmo del .exe (ver nota "Hack to Hack") sin ejecutarlo.
d = open('flag_real.txt.enc', 'rb').read()

def descifrar(d, k):
    ks = [k, k >> 1, (k * 2) & 0xFF]          # llave por posición: i % 3
    return bytes(b ^ ks[i % 3] for i, b in enumerate(d))

# 1) Probar los 256 valores posibles de k y quedarse con los que dan "flag{"
for k in range(256):
    t = descifrar(d, k)
    if t.startswith(b'flag{'):
        print('k =', hex(k), t)
        # 2) k = (suma de bytes de la clave mod 256) ^ 0x34
        suma = k ^ 0x34
        print('suma de bytes de la clave mod 256 =', suma, '->', chr(suma), '(un solo carácter sirve)')

# 3) Comprobar que la clave "I" produce ese k, y que "1A2B3C" (señuelo) no
for clave in ('I', '1A2B3C'):
    print(repr(clave), '-> k =', hex((sum(clave.encode()) % 256) ^ 0x34))
```

## Código de apoyo: solve.py

Guarda este bloque como `solve.py` si sigues los comandos que lo utilizan. Código del análisis original; no se ha vuelto a ejecutar para esta publicación.

```python
d=open('flag_real.txt.enc','rb').read()
k=d[0]^ord('f')
ks=[k,k>>1,(k*2)&255]
print(hex(k), 'suma bytes clave mod 256 =', k^0x34)
print(bytes(b^ks[i%3] for i,b in enumerate(d)))
```
