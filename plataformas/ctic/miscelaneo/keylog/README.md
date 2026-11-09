# Keylog — Write-up CTF

## Reto y estado

- Plataforma: CITC

- Nombre: Keylog

- Categoría: Misceláneo

- Puntos: 200

- Autor: gp

- Archivo: `keylog.txt`

- Estado técnico: resuelto

- Aceptación en plataforma: pendiente de comprobar


## Reconocimiento inicial

El enunciado proporciona dos datos conocidos:

```
Usuario:    tonystark
Contraseña: Password321
```

Como el archivo procede de un keylogger programado incorrectamente, lo primero es comprobar si esos datos aparecen transformados en el registro.

Identifiqué el archivo real:

```
file -k keylog.txt
wc -c -l -w keylog.txt
exiftool keylog.txt
```

Salida real relevante:

```
keylog.txt: Unicode text, UTF-8 text
```

```
42 254 1807 keylog.txt
```

```
File Size       : 1807 bytes
File Type       : TXT
MIME Type       : text/plain
MIME Encoding   : utf-8
Byte Order Mark : No
Newlines        : Unix LF
Line Count      : 42
Word Count      : 254
```

El archivo es texto UTF-8 normal. No contiene una cabecera falsa, datos binarios ni metadatos útiles. El problema se encuentra en la representación de las teclas.

Para conservar el número de cada línea:

```
nl -ba keylog.txt
```

Las dos primeras líneas son:

```
1  yrbfoyapt
2  Laoo,rpe321
```

## Observaciones reales

Las longitudes y los patrones coinciden con las credenciales conocidas:

```
yrbfoyapt   → tonystark
Laoo,rpe321 → Password321
```

Comparé carácter por carácter:

```
paste \
  <(printf '%s' yrbfoyapt | fold -w1) \
  <(printf '%s' tonystark | fold -w1)
```

Salida real:

```
y    t
r    o
b    n
f    y
o    s
y    t
a    a
p    r
t    k
```

Con la contraseña:

```
paste \
  <(printf '%s' 'Laoo,rpe321' | fold -w1) \
  <(printf '%s' 'Password321' | fold -w1)
```

Salida real:

```
L    P
a    a
o    s
o    s
,    w
r    o
p    r
e    d
3    3
2    2
1    1
```

No parece César, ROT ni una sustitución aleatoria. Las correspondencias coinciden con las posiciones físicas de un teclado QWERTY interpretadas mediante la distribución Dvorak.

Por ejemplo:

```
Tecla física QWERTY: q w e r t y u i o p
Carácter Dvorak:     ' , . p y f g c r l
```

Eso explica correspondencias como:

```
y → t
r → o
b → n
f → y
, → w
```

## Hipótesis y por qué surgió

Hipótesis:

> El keylogger registró caracteres según la distribución Dvorak mientras el programador escribía pensando en QWERTY.

No basta con que las credenciales coincidan. Había que buscar una confirmación independiente dentro del archivo.

La línea 4 contiene:

```
ekrpat
```

Aplicando el mismo mapeo:

```
e → d
k → v
r → o
p → r
a → a
t → k
```

Resultado:

```
dvorak
```

La palabra `dvorak` aparece dentro del propio contenido decodificado. Esto confirma la hipótesis y descarta que la coincidencia con las credenciales sea accidental.

La definición local del teclado también puede consultarse con:

```
rg -n 'xkb_symbols "dvorak"' /usr/share/X11/xkb/symbols/us
sed -n '203,255p' /usr/share/X11/xkb/symbols/us
```

La salida muestra oficialmente las posiciones:

```
key <AD01> { [ apostrophe, quotedbl ] };
key <AD02> { [ comma, less ] };
key <AD03> { [ period, greater ] };
key <AD04> { [ p, P ] };
key <AD05> { [ y, Y ] };
...
key <AC01> { [ a, A ] };
key <AC02> { [ o, O ] };
key <AC03> { [ e, E ] };
...
key <AB01> { [ semicolon, colon ] };
key <AB02> { [ q, Q ] };
key <AB03> { [ j, J ] };
```

## Comprobación paso a paso

### 1. Construir el mapeo

Las letras QWERTY y los caracteres obtenidos en Dvorak son:

```
Dvorak registrado:
',.pyfgcrlaoeuidhtn;qjkxbm

QWERTY pretendido:
qwertyuiopasdfghjklzxcvbnm
```

La transformación puede realizarse con `tr`, por lo que no hace falta crear un script.

### 2. Verificar varias líneas

Apliqué el mapeo a varias líneas reales del archivo:

```
printf '%s\n' \
  'yrbfoyapt' \
  'Laoo,rpe321' \
  'Drf .ml.j. lprxaber gb lapo.p pálcer lapa JOKo' \
  'ekrpat' \
  'lfydrb' \
  'e.u jn.ab_pr,o(eaya):' \
  'unai --> U.ncjceae.oLrp.nXcj.by.bapcrXrnckca' |
tr \
  "',.pyfgcrlaoeuidhtn;qjkxbmPYFGCRLAOEUIDHTNQJKXBM" \
  "qwertyuiopasdfghjklzxcvbnmRTYUIOPASDFGHJKLXCVBNM"
```

Salida real:

```
tonystark
Password321
Hoy empece probando un parser rápido para CSVs
dvorak
python
def clean_rows(data):
flag --> FelicidadesPorelBicentenarioBolivia
```

El resultado explica simultáneamente:

- las credenciales del enunciado;

- frases en español;

- nombres de lenguajes;

- fragmentos de código;

- la palabra `dvorak`;

- una línea marcada explícitamente como `flag`.


### 3. Localizar la línea de la flag

La línea 17 original es:

```
unai --> U.ncjceae.oLrp.nXcj.by.bapcrXrnckca
```

La primera palabra se transforma directamente:

```
u n a i
f l a g
```

La parte restante se separa así:

```
U.ncjceae.o → Felicidades
Lrp.n       → Porel
Xcj.by.bapcr → Bicentenario
Xrnckca     → Bolivia
```

Resultado exacto:

```
flag --> FelicidadesPorelBicentenarioBolivia
```

Es importante conservar `Porel` tal como fue extraído. Aunque estilísticamente podría escribirse `PorEl`, no debemos modificar una posible flag para mejorar su gramática o capitalización.

## Consideración sobre la puntuación

En Dvorak, algunas posiciones alfabéticas producen signos como:

```
' , . ;
```

Esos mismos signos también pueden existir como puntuación literal en el texto original. Por eso una traducción ciega de todo el documento puede deformar puntos, comas o fragmentos de código.

Esto no afecta a la flag porque:

- comienza con el marcador reconocible `unai → flag`;

- el resto forma una frase continua y coherente;

- la misma tabla ya fue validada con las credenciales;

- aparece en una línea dedicada y señalada con una flecha.


La interpretación humana sigue siendo necesaria para reconstruir perfectamente algunos fragmentos de código, pero no para identificar el valor señalado como flag.

## Resultado

Cadena de resolución:

```
archivo UTF-8
→ usar las credenciales como texto conocido
→ comparar caracteres
→ reconocer posiciones Dvorak/QWERTY
→ confirmar con “ekrpat” → “dvorak”
→ traducir la línea “unai”
→ recuperar el valor de la flag
```

Valor extraído y confirmado:

```
FelicidadesPorelBicentenarioBolivia
```

## Flag confirmada o candidata

El contenido de la flag está confirmado por el archivo:

```
FelicidadesPorelBicentenarioBolivia
```

El enunciado no especifica un contenedor como `flag{...}` o `CITC{...}`. Por tanto, la candidata de envío más fiel es exactamente:

```
FelicidadesPorelBicentenarioBolivia
```

No se marca como aceptada por la plataforma hasta realizar el envío.

## Qué aprendí

- Las credenciales conocidas pueden utilizarse como un ataque de texto conocido.

- Un keylogger puede registrar caracteres incorrectos si interpreta teclas físicas con otra distribución.

- Los signos extraños no siempre indican cifrado: pueden corresponder a posiciones de teclado.

- Una hipótesis debe explicar más de un dato. Aquí explica las credenciales, el código y la palabra `dvorak`.

- No se debe corregir la ortografía o capitalización de una flag extraída.


## Errores y falsos caminos

- Probar César, ROT13 o sustitución monoalfabética sin utilizar las credenciales.

- Tratar el contenido como texto cifrado aleatoriamente.

- Analizar solamente frecuencias de letras.

- Corregir `Porel` a `PorEl` por intuición.

- Traducir toda la puntuación automáticamente sin considerar sus posibles ambigüedades.

- Declarar aceptación sin haber enviado la respuesta.


## Aporte para el arsenal

Una herramienta para retos de teclado podría:

- recibir texto conocido y texto registrado;

- mostrar correspondencias carácter por carácter;

- comparar distribuciones QWERTY, Dvorak, AZERTY y QWERTZ;

- preservar mayúsculas;

- señalar caracteres ambiguos;

- mostrar el texto original y traducido en paralelo;

- permitir correcciones manuales de puntuación;

- no seleccionar una flag únicamente porque aparece una palabra reconocible.


Técnica reutilizable:

```
texto aparentemente ilegible
→ buscar palabras conocidas del enunciado
→ comparar posiciones físicas
→ identificar distribución de teclado
→ validar con otra palabra independiente
→ traducir conservando exactamente el resultado
```
