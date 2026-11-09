# Los restos de una memoria

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Forense.
- Archivo/objetivo: consola web en `http://201.131.45.82:6008` (servicio en vivo, autorizado por el usuario según STATE). Materiales conservados en esta carpeta: swap.raw, strings.txt, x.sh, c.txt.
- Estado: **resuelto localmente, candidata pendiente de aceptación por el usuario**.

> Enunciado original: **pendiente** (no quedó registrado en SOLVE.md ni STATE.md). Por el contenido recuperado, el tema es un sistema que registra actividad de distintos «CREW» y guarda un volcado de memoria con datos que alguien intentó borrar.

## Resumen

El objetivo era una consola web que ejecuta comandos de sistema mediante una API (`POST /exec`). Listar el directorio de trabajo no mostraba todo: un archivo de manifiesto señalaba un volcado oculto (`swap.raw`) en otra ruta. Extraer las cadenas de texto de ese volcado reveló un registro de comandos en claro con una flag señuelo, y justo después, el mismo tipo de registro pero **cifrado con un César/Vigenère que desplaza letra por letra**. Usando fragmentos de texto conocido (crib dragging) contra la parte cifrada se recuperó una clave de 6 letras que descifra la última línea, con la flag candidata real.

## Reconocimiento

Herramienta usada para hablar con la consola: x.sh, un script de una línea que envía el comando como JSON a `/exec` reutilizando la cookie de sesión guardada en c.txt:

```bash
#!/bin/sh
# uso: ./x.sh "cmd"  (sesion por cookie)
curl -s -b c.txt -c c.txt -H 'Content-Type: application/json' \
  -d "$(python3 -c 'import json,sys;print(json.dumps({"cmd":sys.argv[1]}))' "$1")" \
  http://201.131.45.82:6008/exec
```

```bash
./x.sh "ls -la"
```

**Hallazgo:** el listado del directorio de trabajo no mostraba todos los archivos relevantes. Entre lo que sí aparecía había un `papelera.manifest` que apuntaba a un volcado oculto en otra ruta del sistema: `/var/.crash/swap.raw`.

## Hipótesis

- [x] El sistema guarda, en algún archivo no listado por defecto, restos de actividad que alguien intentó ocultar o borrar.
- [x] El volcado de memoria contiene tanto un señuelo en texto claro como un registro cifrado.
- [x] El cifrado del registro oculto es un desplazamiento de tipo César/Vigenère, descifrable por fragmentos de texto conocido (crib dragging), porque el registro cifrado conserva la misma estructura de líneas que el registro en claro (timestamps, nombre de comandos).
- [ ] La clave de 6 letras es constante en todo el texto cifrado — las pruebas registradas no terminan de confirmarlo (ver «Incertidumbre sobre la clave»).

## Resolución paso a paso

### 1. Encontrar el volcado oculto

```bash
./x.sh "cat papelera.manifest"
```

**Resultado útil:** el manifiesto señala la existencia de un volcado de memoria en `/var/.crash/swap.raw`, no listado en el directorio de trabajo normal.

### 2. Extraer el volcado y leer sus cadenas

```bash
./x.sh "cat /var/.crash/swap.raw" > swap.raw   # (o el comando equivalente para volcarlo)
strings /var/.crash/swap.raw > strings.txt
```

**Resultado útil** (contenido real guardado en strings.txt): entre ruido binario aparecen dos bloques de texto estructurado.

Primer bloque, en claro:

```text
01:04:12  login: SISTEMA
01:04:15  calibracion --rutina
01:06:40  diagnostico --completo
01:09:02  logout: SISTEMA
-- fragmento de comandos.log (rotado, sesion anterior) --
codigo de sesion: UTOPIA7{calibracion_de_rutina_completada}
```

**Hallazgo:** una flag con el formato correcto (`UTOPIA7{...}`), pero el propio texto la describe como «sesión anterior» y «rotada»: es consistente con un **señuelo**, no la respuesta del reto. Se descarta como tal.

Segundo bloque, con la misma estructura de líneas pero **cifrado** (cada letra desplazada, los números y símbolos de puntuación intactos):

```text
03:17:42  lrhpr: ERHX-07
03:18:01  tioouz_nycrg --fyeue
03:19:33  ufhgvou_pcitrlel
03:21:04  iucdql_tqd --fbuggl
03:21:17  vjnrcl_wshrumluait --dlthfne
03:21:19  opnswt: FSLA-07
-- hrdhtiptr slgwphshhq dh dvqcngpz.pqg.fpttnewp (lpkmlohhq 03:24:49) --
owsv etckjcs uaojv hgl pjzqq bdsymfo: gjhkpovupgq_071.tpq
...
gxdlis hn rheytnrdemsw: UWQTMJ7{lr_jmgrswg_xy_vivos_gaez_07}
```

**Interpretación:** la forma de las líneas (`HH:MM:SS  acción`) es idéntica a la del bloque en claro, y hasta el formato de la flag se reconoce desplazado: `UWQTMJ7{...}` tiene la misma longitud y posición de llaves que `UTOPIA7{...}`. Esto confirma la hipótesis de un cifrado de sustitución por desplazamiento (tipo César/Vigenère) aplicado letra por letra, dejando intactos dígitos y símbolos.

### 3. Recuperar la clave por fragmentos de texto conocido (crib dragging)

**Idea:** si se adivina qué texto en claro corresponde a un fragmento cifrado, restar (letra a letra) el cifrado menos el supuesto texto claro da el valor de la clave en esas posiciones. Se probó con nombres de comandos que, por el contexto (un sistema de diagnóstico/memoria), eran candidatos razonables.

Según SOLVE.md, se probaron tres fragmentos (cribs) contra distintas partes del texto cifrado:

| Crib (texto claro supuesto) | Resultado (clave derivada, como desplazamientos 0–25) | Como letras |
|---|---|---|
| `memory_guard --erase` | `7,4,2,0,3,1` | `HECADB` |
| `cache_sesion.dat` | `0,3,2,4,5,2` | *(sin letras anotadas en el registro)* |
| `diagnostico` | `0,3,2,4,4,9` | `ADCEEJ` |

**Resultado útil:** con la clave `ADCEEJ`, aplicada a la línea final en una fase concreta del ciclo de 6 letras (anotada en el registro original como «fase 3», sin más detalle sobre qué significa exactamente esa fase), la línea descifra a una frase coherente:

```text
diagnostico_071.tmp -- restos recuperados del mismo barrido / codigo de recuperacion: ...
```

## Incertidumbre sobre la clave

Los tres cribs probados **no dan la misma secuencia de 6 letras**: `HECADB`, una tercera sin nombrar letras, y `ADCEEJ`. Eso puede deberse a que la clave de 6 letras se lee en una fase distinta del ciclo según en qué posición del texto cae cada crib (un Vigenère de período 6 "rota" según el desplazamiento de arranque), pero **el registro original no explica esa reconciliación**, solo indica que `ADCEEJ` en una fase particular fue la que produjo texto legible en la línea final. No se vuelve a verificar aquí esa reconciliación entre cribs: queda documentada como estaba, sin resolver.

## Cadena de resolución

`consola web (/exec) → papelera.manifest → swap.raw oculto → strings → bloque en claro (señuelo UTOPIA7{...}) + bloque cifrado con misma estructura → crib dragging (3 fragmentos candidatos) → clave ADCEEJ (fase 3) → línea final descifrada → flag candidata`

## Flag

```text
UTOPIA7{lo_hiciste_tu_mismo_crew_07}
```

**Estado: candidata, resuelta localmente; pendiente de aceptación por el usuario.** No hay registro de que la plataforma la haya confirmado ni rechazado.

## Qué falta / pendiente

- El enunciado original del reto.
- La explicación completa de por qué los tres cribs no dan la misma secuencia de clave (fase del ciclo de 6 letras).
- El comando exacto usado para volcar `swap.raw` desde la consola remota (no quedó registrado literal, solo la ruta de origen `/var/.crash/swap.raw`).
- Confirmación de aceptación de la flag.

## Qué debo recordar para otro CTF

- Un listado de directorio «normal» puede no mostrar todo: un manifiesto de papelera o similar puede señalar rutas ocultas.
- Un volcado de memoria o swap suele mezclar señuelos en claro con contenido cifrado de estructura idéntica (mismos timestamps, mismo formato de líneas): esa similitud estructural es justamente lo que permite un ataque de texto conocido.
- El crib dragging contra un cifrado de sustitución por desplazamiento (César/Vigenère) funciona letra a letra: cifrado menos texto-claro-supuesto = clave, en las posiciones donde el crib encaja.
- Cuando distintos cribs dan claves distintas en un cifrado periódico, conviene comprobar si es solo un desfase de fase dentro del período antes de asumir que algo está mal.

## Aporte para el arsenal

- Qué parte puede automatizarse: localizar bloques de texto con la misma estructura de líneas (timestamps, formato repetido) en la salida de `strings`, y probar automáticamente una lista de cribs candidatos contra todas las posiciones del bloque cifrado.
- Qué debe seguir requiriendo interpretación humana: elegir qué palabras son cribs razonables según el contexto temático del reto, y decidir si una clave derivada es coherente o es ruido.
- Falso positivo que se debe evitar: tomar como flag un texto con formato correcto (`PREFIJO7{...}`) que el propio archivo describe como de una «sesión anterior» o «rotada».
