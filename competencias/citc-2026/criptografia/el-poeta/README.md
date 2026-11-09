# El Poeta

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Criptografía. Autor: C0nsult0r 03.
- Archivo: adjunto.txt (texto ASCII, 63 líneas, 1907 bytes).
- Estado: **aceptado según el usuario** («gracias funciona»), tras recibir la candidata `3s_3L_p000333t444`. No consta si la plataforma pidió algún envoltorio ni qué cadena exacta se envió.
- Registro anterior: STATE. El índice lo marcaba «pendiente»: **está desactualizado**.
- Nota: en esta carpeta también hay `home.html` y `cookies.txt`; son de otro reto web (Cazadores de biskut) y no se usan aquí.

> Un mensaje cifrado dentro de un poema, un «sello» y pistas ocultas. «Cada descubrimiento parece conducir al siguiente, revelando poco a poco el método de cifrado y las claves necesarias.»

## Resumen

El adjunto contiene una leyenda, un poema, un texto cifrado en hexadecimal (17 bytes) y un sello SHA-256. El poema describe en verso la sucesión de Fibonacci. Aplicando XOR entre el cifrado y los números de Fibonacci (módulo 256) aparece texto legible: `3s_3L_p000333t444`. El «sello» **no** coincide con ese texto con ninguna variante probada, así que su papel real queda sin explicar, pero el resultado fue aceptado.

## Qué recibimos

```bash
cd El-Poeta
file adjunto.txt
wc adjunto.txt
cat adjunto.txt
```

Estructura del archivo:

1. Una leyenda: Penélope «tejió su palabra con tres hilos»: el hilo de **Fibonacci**, el hilo **áureo** (la razón divina) y el hilo de los **eslabones**, «donde cada paso deja una sombra que alimenta al siguiente». Solo se descubren los dos primeros; el tercero se «desanda en orden inverso».
2. Un valor de 64 caracteres hexadecimales: «el último eslabón», con la advertencia «64 sílabas, ni una más, ni una menos».
3. Un poema titulado «El Hilo de Penélope».
4. Al final, dos bloques: `CIPHERTEXT (hex)` = `32725d3049577d2512076aa3da0d56ef09` y `SELLO DE VALIDACION (SHA-256 final)` = `5cc26c70…981ebe`.

## Hipótesis a partir del texto

La observación clave está en el poema:

| Verso | Qué sugiere |
|---|---|
| «En el principio era uno, y el uno miró al uno… dos con uno tejieron tres, y tres con dos engendraron cinco… al ocho» | **1, 1, 2, 3, 5, 8**: Fibonacci. Cada número suma los dos anteriores («sumando sus propias sombras»). |
| «uno coma seis uno ocho, y luego un tres» | **1,618 y luego un 3…**: los primeros dígitos de la razón áurea φ = 1,6180339887…. |
| «Cada letra… deja su sombra en la siguiente… encadenados» | Un cifrado que encadena un byte con el siguiente (como CBC). |

Con el cifrado de 17 bytes, la hipótesis más simple es: *cada byte se combina con un número de una secuencia*. Se probó con tres secuencias (Fibonacci, dígitos de φ, y mezclas de ambas), tres operaciones (XOR, suma, resta) y variantes de encadenamiento.

## Resolución paso a paso

### 1. Probar secuencia × operación

Se guardó el script descifrar_poeta.py. Ejecutarlo desde esta carpeta:

```bash
python3 scripts/descifrar_poeta.py adjunto.txt
```

La parte relevante del código Python:

```python
ct = bytes.fromhex(re.search(r'CIPHERTEXT \(hex\) ---\s*([0-9a-f]+)', t).group(1))
fib = [1, 1]                                   # 1, 1, 2, 3, 5, 8, ...
while len(fib) < len(ct):
    fib.append(fib[-1] + fib[-2])
texto = bytes(c ^ (f % 256) for c, f in zip(ct, fib))
```

`f % 256` reduce cada número de Fibonacci a un byte; `c ^ ...` hace XOR con el byte cifrado correspondiente. XOR se descifra con la misma operación con que se cifró.

Salida real (ejecutada el 4 de octubre):

```text
17 bytes -> 3s_3L_p000333t444
SHA-256 del texto : c9d140025a912ab1b9a415de1f0a7fe74553d49b4ad0534a753d872ad0ca80ab
sello del adjunto : 5cc26c70…981ebe (no coincide: ver nota)
```

**Cómo se supo que era el resultado correcto:** de todas las combinaciones probadas, solo «XOR con Fibonacci» produce texto ASCII legible; las demás dan basura. (Dos variantes con el mismo resultado, empezar con 0 o con 1, no son alternativas reales.) El resultado tiene aspecto de «leet»: `3` por `e`, `0` por `o`, `4` por `a`; se lee como «es el poeta» con las vocales `o`, `e`, `a` repetidas tres veces.

### 2. Intentar validar con el sello

Era lógico esperar `SHA-256(texto) == sello`. No coincidió. Se probó SHA-256 del texto con y sin prefijos (`AGETIC{}`, `citc{}`, `flag{}`, `CTF{}`, `penelope`…), con saltos de línea, en mayúsculas y minúsculas, iterado hasta 100 000 veces, y cadenas `h = sha256(h + letra)` con semillas de φ, de Fibonacci y de nombres. Tampoco se logró «deshacer» el sello como 64 símbolos con XOR, suma o resta consecutiva. **Ninguna dio el sello.**

Aviso: si eso es cierto, el sello y φ eran, o bien una validación de otra capa que no se reconstruyó, o bien parte del enunciado que no hacía falta. Aun así, la cadena que se entregó fue aceptada.

## Cadena de resolución

`adjunto.txt → poema describe 1,1,2,3,5,8 → XOR con Fibonacci mod 256 → texto ASCII legible (leet) → entregar`

## Resultado

```text
3s_3L_p000333t444
```

Estado: **aceptado según el usuario** (mensaje «gracias funciona»); no se registró el formato exacto de la entrega.

## Qué falta / dudas

- El sello SHA-256 no se explicó. No se descartó que correspondiera a la salida de la capa «de eslabones».
- No se verificó si la plataforma exigía un prefijo; el usuario indicó solo que funcionó.

## Qué debo recordar para otro CTF

- Cuando un poema enumera números, comprueba sucesiones conocidas (Fibonacci, primos, π, φ).
- Probar el cifrado más simple primero: XOR con una secuencia, sumas y restas. Un único resultado legible es la confirmación.
- Un «sello» que no valida no invalida un texto legible: puede ser un señuelo o una capa extra.
- Leet (`3`=e, `0`=o, `4`=a) es una pista de que el texto está bien.

## Aporte para el arsenal

- Automatizable: probar listas de secuencias (Fibonacci, primos, π, φ, naturales) con XOR/suma/resta, y puntuar por texto legible.
- Requiere criterio humano: reconocer la secuencia en el poema.
- Falso positivo: resultados que parecen leet sin ser la respuesta; cuidado al asumir un prefijo.


## Código de apoyo: descifrar_poeta.py

Guarda este bloque como `descifrar_poeta.py` si sigues los comandos que lo utilizan. Código del análisis original; no se ha vuelto a ejecutar para esta publicación.

```python
#!/usr/bin/env python3
"""El Poeta: descifra el texto del adjunto con XOR contra la secuencia de Fibonacci.
Uso: python3 scripts/descifrar_poeta.py adjunto.txt"""
import re, sys, hashlib

t = open(sys.argv[1]).read()
ct = bytes.fromhex(re.search(r'CIPHERTEXT \(hex\) ---\s*([0-9a-f]+)', t).group(1))
sello = re.search(r'SHA-256 final\) ---\s*([0-9a-f]{64})', t).group(1)

fib = [1, 1]                                  # 1, 1, 2, 3, 5, 8, ... (la primera estrofa del poema)
while len(fib) < len(ct):
    fib.append(fib[-1] + fib[-2])

texto = bytes(c ^ (f % 256) for c, f in zip(ct, fib))   # cada byte contra Fibonacci mod 256
print(len(ct), 'bytes ->', texto.decode())
print('SHA-256 del texto :', hashlib.sha256(texto).hexdigest())
print('sello del adjunto :', sello, '(no coincide: ver nota)')
```
