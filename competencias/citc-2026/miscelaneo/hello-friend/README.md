# hello, friend.

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Misceláneo.
- Archivo/objetivo: `fsociety00.zip` (entregado en Descargas; no se conserva su ruta original en los registros). Materiales en la subcarpeta `work/`, incluido el audio extraído.
- Estado: **parcial**. Hay una candidata descifrada capa por capa, pero las dos flags propuestas fueron **rechazadas** por la plataforma.
- Registros breves anteriores: SOLVE y STATE (este último marcado «PAUSADO»).

> Enunciado original: **pendiente** (no quedó registrado en SOLVE.md ni STATE.md).

## Resumen

El reto encadena varias capas: un ZIP con contraseña que contiene un PNG y un texto señuelo; el PNG guarda en sus metadatos EXIF la clave de una capa XOR final y, pegado tras su final de archivo, otro ZIP cifrado; ese segundo ZIP contiene un audio. El audio mezcla dos técnicas: un mensaje legible en el espectrograma y una secuencia de tonos DTMF que, decodificados y pasados por XOR, dan el resto del texto. La unión de ambas partes produjo dos candidatas de flag, **ninguna aceptada**.

## Reconocimiento

```bash
file fsociety00.zip
unzip -l fsociety00.zip
```

**Hallazgo:** el ZIP requiere contraseña y contiene al menos un texto (`leeme.txt`) y una imagen (`firma_fsociety.png`).

## Hipótesis

- [x] El ZIP inicial se abre con una contraseña relacionada con el tema del reto (`fsociety`).
- [x] `leeme.txt` es un mensaje de bienvenida, no la flag.
- [x] El PNG guarda una pista o una clave en metadatos.
- [x] Hay contenido adicional oculto tras el final del PNG.
- [x] El audio final combina más de una técnica esteganográfica.
- [ ] La flag es exactamente la concatenación literal del texto del espectrograma y el texto del DTMF descifrado — **sin confirmar**; ambos intentos con esta hipótesis fueron rechazados.

## Resolución paso a paso

### 1. Abrir el ZIP inicial

```bash
unzip -P fsociety fsociety00.zip
```

**Resultado útil:** la contraseña `fsociety` extrae `leeme.txt` y `firma_fsociety.png`.

### 2. Descartar el señuelo de texto

`leeme.txt` contiene una cadena en Base64 que decodifica a:

```text
citc{m3ns4j3_d3_bi3nv3nid4_no_3s_l4_fl4g}
```

**Resultado útil:** el propio contenido se declara señuelo («el mensaje de bienvenida no es la flag»). Se descarta y se continúa con la imagen.

### 3. Leer los metadatos del PNG

```bash
exiftool firma_fsociety.png
```

**Resultado útil:** el campo `Comment` dice «capa final: XOR de un byte. clave = 0x2B». Esto confirma que, más adelante, habrá que aplicar XOR con la clave `0x2B` a algún contenido, y que es la **última** capa del esquema.

### 4. Encontrar datos ocultos tras el PNG

```bash
binwalk firma_fsociety.png
```

**Resultado útil:** `binwalk` identifica un ZIP embebido después del final de bloque del PNG (tras `IEND`). Extraído, ese ZIP está cifrado.

### 5. Romper la contraseña del ZIP embebido

```bash
zip2john embedded.zip > embedded.hash
john --format=PKZIP embedded.hash --wordlist=rockyou.txt
```

**Resultado útil:** la contraseña es `elliot`. Con ella se extrae `transmision.wav` (17,4 segundos de audio).

### 6. Leer el espectrograma del audio

```bash
sox transmision.wav -n spectrogram -x 3000 -y 800 -z 130
```

**Resultado útil:** en los primeros segundos del espectrograma aparece el texto `citc{fs0ci3ty_`, cortado por un guion bajo final que indica que el mensaje continúa. **Nota de precaución dejada en los registros:** a simple vista, con menos resolución, la letra «i» de `fs0ci3ty` se confunde con un guion bajo o con otros caracteres (`o`/`0`, `e`/`3`); hace falta el zoom alto (`-x 3000 -y 800 -z 130`) indicado arriba para distinguirlos. Una primera lectura con menos cuidado produjo `fs0c_3ty_` en vez de `fs0ci3ty_`, y esa variante fue una de las dos candidatas rechazadas.

### 7. Decodificar los tonos DTMF del resto del audio

**Hallazgo:** después del tramo con el mensaje en espectrograma (aprox. desde t > 6,9 s) el audio contiene una secuencia de **42 tonos DTMF**. Se decodificaron con el algoritmo de Goertzel (detección de las frecuencias estándar de fila/columna de DTMF). El procedimiento no quedó documentado como script independiente; SOLVE.md lo describe como «decodificador DTMF inline en conversación» — **pendiente**: no se conserva ese código ni un archivo de salida intermedio con los 42 símbolos uno por uno.

**Resultado útil:** los 42 tonos, agrupados de a pares, dan un hexadecimal de 21 bytes. Aplicando XOR con la clave `0x2B` (la misma anunciada en el EXIF del PNG) se obtiene:

```text
dtmf_y_3sp3ctr0gr4m4}
```

### 8. Unir ambas mitades

```bash
printf '%s%s\n' 'citc{fs0ci3ty_' 'dtmf_y_3sp3ctr0gr4m4}'
```

**Resultado útil:** `citc{fs0ci3ty_dtmf_y_3sp3ctr0gr4m4}`, que se lee de forma coherente: «fsociety, dtmf y espectrograma».

## Inconsistencia detectada entre los registros

SOLVE.md tiene dos grafías distintas para el mismo prefijo en líneas diferentes del mismo archivo: el paso 7 escribe `citc{fs0c13ty_}` (con «1» y «3»), mientras que el paso 5 y la «flag candidata corregida» final usan `citc{fs0ci3ty_}` (con «i»). STATE.md, más reciente, usa consistentemente `fs0ci3ty_`. Se toma `fs0ci3ty_` como la lectura vigente por ser la de STATE.md (el registro de pausa, posterior) y la de la línea de «flag candidata corregida» en SOLVE.md, pero **esto no está verificado de nuevo contra el espectrograma** en este write-up (instrucción explícita: no ejecutar pruebas).

## Cadena de resolución

`fsociety00.zip (pw "fsociety") → leeme.txt (señuelo Base64) + firma_fsociety.png → EXIF "clave XOR 0x2B" + ZIP embebido tras IEND (binwalk) → embedded.zip (pw "elliot", crackeada con John/rockyou) → transmision.wav → espectrograma (prefijo "citc{fs0ci3ty_") + 42 tonos DTMF→hex→XOR 0x2B (sufijo "dtmf_y_3sp3ctr0gr4m4}") → concatenación`

## Flag

```text
citc{fs0ci3ty_dtmf_y_3sp3ctr0gr4m4}
```

**Estado: parcial, candidata rechazada.** Según STATE.md, la plataforma rechazó explícitamente dos variantes:

- `citc{fs0c_3ty_dtmf_y_3sp3ctr0gr4m4}` (primera lectura del espectrograma, con un guion bajo de más).
- `citc{fs0ci3ty_dtmf_y_3sp3ctr0gr4m4}` (lectura corregida del espectrograma, con «i»).

No se probaron más variantes después de ese segundo rechazo; el reto quedó pausado.

## Qué falta / pendiente

- **Enunciado original del reto:** no está en los registros.
- **Re-verificación símbolo por símbolo de los 42 tonos DTMF:** STATE.md deja como hipótesis abierta que uno o más tonos estén mal clasificados por confusión entre frecuencias cercanas de la tabla DTMF (filas 770/852/941 Hz, columnas 1209/1336/1477/1633 Hz). No se aisló ni volvió a escuchar/graficar cada pulso por separado.
- **Si la unión prefijo + sufijo es una simple concatenación:** no confirmado; es una hipótesis sin marcar en la lista de hipótesis.
- **Script del decodificador DTMF:** no se conserva como archivo; solo se ejecutó «inline» según SOLVE.md.
- **Corrección de la inconsistencia `fs0c13ty_` / `fs0ci3ty_`** dentro del propio SOLVE.md: no resuelta en este write-up.

## Qué debo recordar para otro CTF

- Un comentario EXIF puede anunciar una capa de cifrado que se aplica más adelante, no en la propia imagen.
- Un PNG puede llevar datos adicionales después de `IEND`; conviene revisarlo con `binwalk` aunque la imagen se vea normal.
- Un espectrograma necesita suficiente resolución (`-x`, `-y`, `-z` en `sox`) para distinguir caracteres parecidos (`i`/`_`, `0`/`o`, `3`/`e`).
- DTMF es una capa esteganográfica de audio que se decodifica por pares de frecuencias (fila/columna), no por un solo tono.
- Cuando una flag se arma combinando dos lecturas (visual + decodificada), un error en cualquiera de las dos invalida el conjunto; conviene aislar y confirmar cada mitad por separado antes de unirlas.

## Aporte para el arsenal

- Qué parte puede automatizarse: extracción de ZIP embebido tras `IEND` (binwalk + corte de offset), cracking de ZIP con John/rockyou, decodificación DTMF por Goertzel sobre pulsos detectados por energía.
- Qué debe seguir requiriendo interpretación humana: lectura fina del espectrograma (distinguir caracteres ambiguos) y decidir si el ensamblado de capas (prefijo + sufijo) es correcto.
- Falso positivo que se debe evitar: dar por buena una lectura visual del espectrograma sin el zoom adecuado; el propio registro documenta un error real de este tipo.
