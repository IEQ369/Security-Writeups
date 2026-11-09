# Canario Filtrado

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Reversing / explotación binaria.
- Archivo: reportes (ELF 64 bits, PIE, enlazado dinámicamente, **sin símbolos** — `stripped`).
- Estado: **aceptado**. El usuario lo confirmó («sí funcionó el de canario») antes de pasar al siguiente reto.
- Registro breve anterior: SOLVE.

> No se conserva el enunciado textual; por el binario, el reto pide identificar una flag de verdad escondida detrás de una flag señuelo explícita y un volcado de depuración que filtra protecciones de pila.

## Resumen

El programa es un menú con cuatro opciones. Una opción de «Debug» imprime el valor real del canario de pila y una dirección de la propia función — suficiente para calcular, en teoría, la base de carga del binario (PIE). Otra opción muestra, en texto plano, una flag que se anuncia a sí misma como de «staging»: un señuelo explícito. La flag verdadera está en una **cuarta función que el menú nunca llama**: decodifica 38 bytes embebidos en el propio código con un XOR de un byte. Se recuperó leyendo el desensamblado a mano, sin ejecutar el binario (no corre en este Debian 12 por una versión de `glibc` distinta).

## Qué recibimos

Un único ejecutable, `reportes`.

## Reconocimiento

```bash
file reportes
objdump -d -M intel --no-show-raw-insn reportes > dis_full.txt
grep -n '^[0-9a-f]\{16\} <' dis_full.txt
```

**Salida que importa:**

```text
reportes: ELF 64-bit LSB pie executable, x86-64, ... dynamically linked ... stripped
...
00000000000010d0 <.text>:
000000000000143c <.fini>:
```

Sin símbolos: todo el código cae en un único bloque `.text`, sin nombres de función. Hay que ubicar las funciones por su posición y su contenido, no por su nombre. Antes de ejecutar nada (regla del equipo: análisis estático primero con binarios desconocidos), se intenta correrlo igualmente para confirmar si hace falta desensamblar a mano:

```bash
./reportes
```

```text
./reportes: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
```

El Debian 12 del equipo trae una `glibc` más antigua. **No se puede ejecutar sin un entorno con `glibc ≥ 2.38`** (un contenedor o Distrobox sí podría). Por eso todo lo que sigue es análisis estático puro.

## Resolución paso a paso

### 1. Leer las cadenas de texto (`.rodata`)

```bash
objdump -s -j .rodata reportes
```

```text
2008: citc{fl4g_d3_st4g1ng_n0_v4l1d4_3n_pr0d}
2030: CANARY: %#lx
2040: BASE_LEAK: %#lx
2050: Contenido del reporte:
2068: MENU\n1) Generar reporte\n2) Debug\n3) Ver flag de prueba\n4) Salir\n>
20ab: %d
```

**Interpretación:** hay un menú de 4 opciones. La opción «Ver flag de prueba» imprime directamente la cadena `citc{fl4g_d3_st4g1ng_n0_v4l1d4_3n_pr0d}`. El nombre («de prueba», «staging») y que esté en texto plano, visible con un simple `strings`, la delatan como señuelo: una flag real no se regala así. Hay además dos `printf` de depuración (`CANARY`, `BASE_LEAK`) que no deberían existir en un programa de producción: anuncian una vulnerabilidad deliberada.

### 2. Ubicar las cuatro funciones en el código

```bash
sed -n '76,295p' dis_full.txt   # el bloque .text completo
```

Se identifican por lo que hacen y por constantes de `.rodata`:

| Dirección | Qué hace | Identificación |
|---|---|---|
| `0x11b9` | Sin referencias desde el menú. Monta 38 bytes con varios `movabs` y los decodifica con XOR, imprimiendo byte a byte con `putchar` | **función oculta** (ver paso 4) |
| `0x127b` | `puts` de la cadena en `0x2008` | opción 3, «Ver flag de prueba» (señuelo) |
| `0x12b8` | dos `printf`: uno con un valor de `fs:0x28` (el canario real) y el formato `CANARY: %#lx`; otro con **su propia dirección** (`0x12b8`) y el formato `BASE_LEAK: %#lx` | opción 2, «Debug» |
| `0x132c` | `puts("Contenido del reporte:")`, luego `read(0, buffer, 0x200)` sobre un buffer de `rbp-0x50` | opción 1, «Generar reporte» |
| `0x13b4` | bucle: `printf` del menú, `scanf("%d", ...)`, `getchar()` para consumir el salto de línea, y despacha a 1/2/3 según el número | bucle principal (`main`) |

El buffer de la opción 1 está en `rbp-0x50` y el canario de pila en `rbp-0x8` (`mov rax, QWORD PTR fs:0x28` al principio de esa función). Entre ambos hay `0x50 - 0x8 = 0x48 = 72` bytes **utilizables** antes de pisar el canario: `read` acepta hasta 512 bytes (`0x200`), así que hay desbordamiento de pila si se escriben más de 72 bytes.

### 3. Entender la opción «Debug» como fuga de información

```text
12cf: mov rax, QWORD PTR fs:0x28        # el canario real del proceso
12e0: lea rdx, [rip+0xd49]              # "CANARY: %#lx"
12f2: call printf
12f7: lea rdx, [rip-0xffffffffffffffba] # 0x12b8: la propia dirección de esta función
12fe: lea rax, [rip+0xd39]              # "BASE_LEAK: %#lx"
1310: call printf
```

**Por qué importa:** en un binario PIE, las direcciones cambian en cada ejecución por aleatorización (ASLR). Conocer la dirección de una función en tiempo de ejecución permite calcular la **base de carga**: `base = BASE_LEAK − 0x12b8` (el desplazamiento fijo de esa función dentro del binario, visto en el propio desensamblado). Junto con el canario real, esto da todo lo necesario para construir un payload de desbordamiento que salte a una dirección exacta pese al ASLR y sin disparar `__stack_chk_fail`.

### 4. Decodificar la función oculta (sin ejecutar nada)

```bash
sed -n '/^    11b9:/,/^    127b:/p' dis_full.txt
```

El cuerpo relevante:

```text
11d0: movabs rax, 0x346e3921392e3339
11da: movabs rdx, 0x366b3c056a6b286e
11e4: mov QWORD PTR [rbp-0x30],rax
11e8: mov QWORD PTR [rbp-0x28],rdx
11ec: movabs rax, 0x6a34056a3e6e282e
11f6: movabs rdx, 0x693d692e6a282a05
1200: mov QWORD PTR [rbp-0x20],rax
1204: mov QWORD PTR [rbp-0x18],rdx
1208: movabs rax, 0x276e3e6e3405693d
1212: mov QWORD PTR [rbp-0x12],rax      # esta escritura SE SOLAPA con la anterior
1216: mov BYTE PTR [rbp-0x39],0x5a      # clave XOR = 0x5a
121a: mov QWORD PTR [rbp-0x38],0x0      # índice = 0
1224: lea rdx,[rbp-0x30]                # inicio del buffer
122f: movzx eax, BYTE PTR [rax]
1232: xor al, BYTE PTR [rbp-0x39]       # byte XOR clave
123a: call putchar
1244: cmp QWORD PTR [rbp-0x38],0x25     # 0x25 = 37 -> 38 bytes en total (0..37)
```

**Lo no evidente:** la penúltima escritura (`[rbp-0x18]`, 8 bytes) y la última (`[rbp-0x12]`, 8 bytes) se solapan en dos posiciones de memoria, porque `-0x12` cae dentro del rango que ya había escrito `-0x18` (`-0x18` ocupa hasta `-0x11`). La segunda escritura, al ser posterior, **gana** en esas dos posiciones. Simularlo a mano en papel es propenso a error; se automatizó con un script.

Script guardado en decodificar_flag.py (ejecutar con `python3 scripts/decodificar_flag.py` desde esta carpeta), que reproduce exactamente esas escrituras en un buffer de memoria simulado y aplica el XOR:

```python
mem = bytearray(0x40)
def put(off, val):                       # off = magnitud de rbp-off
    mem[0x40 - off : 0x40 - off + 8] = struct.pack('<Q', val)

put(0x30, 0x346e3921392e3339)
put(0x28, 0x366b3c056a6b286e)
put(0x20, 0x6a34056a3e6e282e)
put(0x18, 0x693d692e6a282a05)
put(0x12, 0x276e3e6e3405693d)             # se solapa con la escritura anterior

buf = bytes(mem[0x40-0x30 : 0x40-0x30+0x26])   # 38 bytes
print(bytes(b ^ 0x5a for b in buf).decode())
```

Salida (reproducida el 4 de octubre, sin ejecutar `reportes`):

```text
citc{c4n4r10_f1ltr4d0_n0_pr0t3g3_n4d4}
```

Coincide con la flag registrada en la competencia.

### 5. El exploit que el binario insinúa (no ejecutado)

Aunque no hizo falta correrlo para obtener la flag, el diseño del binario describe un exploit completo:

1. Opción 2 (Debug) para filtrar el canario real y `BASE_LEAK`; calcular `base = BASE_LEAK - 0x12b8`.
2. Opción 1 (Generar reporte) con un payload: 72 bytes de relleno + el canario filtrado (8 bytes) + 8 bytes cualesquiera (el `rbp` guardado) + `base + 0x11b9` (la dirección real de la función oculta) como dirección de retorno.
3. Al volver de la función, en vez de regresar a `main` salta a la función oculta, que imprime la flag por su cuenta.

Esto **no se ejecutó** (falta la `glibc` compatible); se describe porque explica por qué el programa expone el canario y una dirección: son las piezas que el reto espera que se filtren y se reutilicen.

## Cadena de resolución

`binario stripped → leer .rodata → señuelo "staging" descartado por evidente → Debug filtra canario+dirección (diseño de exploit) → función 0x11b9 nunca invocada desde el menú → desensamblar sus movabs → reconstruir 38 bytes (con solapamiento) → XOR 0x5a → flag`

## Flag

```text
citc{c4n4r10_f1ltr4d0_n0_pr0t3g3_n4d4}
```

Estado: aceptada. Obtenida por simulación estática del desensamblado; nunca se ejecutó el binario.

## Qué debo recordar para otro CTF

- Un binario sin símbolos sigue teniendo un único `.text`: localizar funciones por lo que hacen (llamadas a `printf`, constantes, patrones de prólogo/epílogo) y no por nombre.
- Una flag en texto plano accesible por un menú explícito («de prueba», «staging») es casi siempre señuelo.
- `printf` de un canario o de una dirección propia en un binario de reto son pistas de un exploit intencionado (leak + ROP/salto), no errores del autor.
- `movabs` seguido de `mov QWORD PTR [...]` repetido es una forma común de embeber bytes constantes sin que aparezcan como cadena en `.rodata` (evita que `strings` los vea).
- Escrituras de memoria que se solapan cambian el resultado final: conviene simularlas con un script en vez de leerlas de corrido.
- `glibc` nueva en el binario y vieja en el host: no se puede ejecutar sin contenedor; no es motivo para detenerse si el análisis estático basta.

## Aporte para el arsenal

- Automatizable: detectar funciones nunca alcanzadas desde `main` (análisis de referencias cruzadas) y secuencias `movabs`+`mov [mem]` seguidas de un bucle con una operación de un byte (candidato a XOR embebido).
- Requiere interpretación humana: decidir cuál «flag» es señuelo según el contexto narrativo (nombres, ubicación, facilidad de acceso).
- Falso positivo: aceptar la primera cadena con formato `citc{...}` sin revisar si hay una función oculta.


## Código de apoyo: decodificar_flag.py

Guarda este bloque como `decodificar_flag.py` si sigues los comandos que lo utilizan. Código del análisis original; no se ha vuelto a ejecutar para esta publicación.

```python
#!/usr/bin/env python3
"""Canario Filtrado: reconstruye los 38 bytes embebidos en la función oculta (0x11b9)
exactamente como los deja el binario (incluye la escritura superpuesta final) y aplica
el XOR de un byte. No ejecuta el binario."""
import struct

mem = bytearray(0x40)
def put(off, val):               # off = magnitud de rbp-off; val = qword little-endian
    mem[0x40 - off : 0x40 - off + 8] = struct.pack('<Q', val)

# valores tomados de los `movabs` en 0x11d0..0x1212 (ver evidencias/desensamblado-completo.txt)
put(0x30, 0x346e3921392e3339)
put(0x28, 0x366b3c056a6b286e)
put(0x20, 0x6a34056a3e6e282e)
put(0x18, 0x693d692e6a282a05)
put(0x12, 0x276e3e6e3405693d)   # esta escritura se solapa con la anterior: así queda en la pila real

buf = bytes(mem[0x40 - 0x30 : 0x40 - 0x30 + 0x26])   # 38 bytes (índice 0..0x25 inclusive)
clave = 0x5a                                          # mov BYTE PTR [rbp-0x39],0x5a
print(bytes(b ^ clave for b in buf).decode())
```
