# INCIDENT 42

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Forense.
- Archivo: `INCIDENT_42_participante.zip` (ya no está en Descargas). Contenido extraído en ![](imagenes/evidence.png): `evidence.png`, `workstation.mem`, `network.log`, `incident_note.txt`, `old_backup.zip`, `README.txt`. Los archivos del `old_backup.zip` y del ZIP escondido en el PNG están en `o/`.
- Estado: **resuelto localmente, con cabos sueltos**. Se recuperó un mensaje legible; no consta que la plataforma lo aceptara, no se sabe qué formato exigía y quedan dos pistas sin explicar.
- Registro breve anterior: SOLVE.

> Un equipo apagado misteriosamente. El dueño dice que borró todo lo relacionado con el incidente. «El equipo dejó más información de la que su dueño esperaba. Encuentra qué fue lo que realmente dejó oculto.»

## Resumen

Hay cinco artefactos: una imagen PNG, un volcado de memoria de 128 KiB, un registro de red, una nota y un ZIP viejo. La imagen es solo un «sobre» (esconde un ZIP cuya pista dice que mires la memoria). La memoria contiene un historial de comandos, una pista de clave (`ROTATE-3: N7-04-22-18`) y un bloque `STAGEv01` ofuscado con XOR. Con la clave `N7042218` el bloque da el mensaje `AccesoNoRegistrado`.

## Qué recibimos

```bash
cd Incident-42/work
cat README.txt incident_note.txt network.log
file *
unzip -l old_backup.zip
```

- `README.txt`: «Something in the evidence does not agree with the story. Find the hidden message.» No indica formato de flag.
- `incident_note.txt`: apagado a las 02:19, última actividad normal a las 02:17; «El operador insistía en que borrar un archivo bastaba. No era así». Pista de que lo borrado puede seguir en memoria.
- `network.log`: ocho consultas DNS entre 02:17:01 y 02:17:08, con nombres como `18.delta.lab`, `07.alpha.lab`, `04.gamma.lab`: un número y una letra griega.
- `evidence.png`: PNG de 640×420 (258 256 bytes). `workstation.mem`: datos (131 072 bytes). `old_backup.zip`: 394 bytes.

## Resolución paso a paso

### 1. Revisar los archivos «de portada»

```bash
mkdir -p ../o && cd ../o
unzip -o ../work/old_backup.zip
cat readme.old flag.txt notes.txt
```

Salida: `Archive from a previous incident.` / `Bicentenario2025` / `This archive predates the incident.` El `flag.txt` del ZIP viejo contiene `Bicentenario2025`, y `notes.txt` dice que es de un incidente anterior. **Señuelo**: el propio archivo dice que «predata» el incidente y no está relacionado con la hora del apagado.

### 2. La imagen: buscar datos tras el final del PNG

**Observación:** un PNG de 258 256 bytes con solo cuatro bloques de imagen es normal, pero `pngcheck` avisa de «additional data after IEND chunk».

```bash
binwalk evidence.png
pngcheck -v evidence.png
zsteg -a evidence.png | grep -v nothing
```

`pngcheck` lista los bloques: `IHDR`, tres `tEXt` (Author `Lab-Camera`, Description `warehouse inspection`, Date `2026-09-20 02:17:43`), cuatro `IDAT` e `IEND`, y «additional data after IEND». `zsteg` detalla: `[?] 310 bytes of extra data after image end (IEND), offset = 0x3ef9a` y `file: Zip archive`. Un PNG termina en `IEND`; lo que sigue es un ZIP añadido.

El script extraer_stage.py (ejecutar `python3 scripts/extraer_stage.py work` desde la carpeta del reto) automatiza este paso y el siguiente. Parte para el ZIP:

```python
png = open('work/evidence.png','rb').read()
fin = png.rindex(b'IEND') + 8                 # fin real del PNG
z = zipfile.ZipFile(io.BytesIO(png[fin:]))    # lo que sobra es un ZIP
```

También se puede cortar con la terminal: `dd if=evidence.png bs=1 skip=257946 of=png_tail.zip && unzip png_tail.zip`.

Dentro hay `clue.txt` y `decoy.txt`:

```text
clue.txt:  The picture was only the envelope.
           Look where the machine remembers what the network forgot.
decoy.txt: Nothing useful here.
```

**Cómo continúa:** «el sitio donde la máquina recuerda lo que la red olvidó» es la memoria RAM (`workstation.mem`). El `decoy.txt` se descarta por su propio nombre.

### 3. Explorar la memoria

```bash
strings -n 6 workstation.mem | less
grep -abo -E 'ROTATE[^ ]*|bash_history|STAGEv01-|ge6cso5t|rm -f /tmp' workstation.mem
```

`grep -abo` busca en binario (`-a`), muestra el offset en bytes (`-b`) y solo la parte coincidente (`-o`). Hallazgos con su offset:

| Offset | Contenido |
|---|---|
| 1470 | `ge6cso5t}` (texto suelto) |
| 10816 y siguientes | `bash_history`: `whoami`, `python3 /opt/rotate.py --slot 3`, `rm -f /tmp/stage.bin`, `clear` |
| 24352 | `ROTATE-3: N7-04-22-18` |
| 35344 | `deleted=/tmp/stage.bin` |
| 43008 | `STAGEv01-` seguido de bytes sin sentido |

**Interpretación:** el operador ejecutó un rotador con la «ranura 3» y borró `/tmp/stage.bin`. La memoria guarda el nombre del archivo borrado, una pista de clave (`ROTATE-3: N7-04-22-18`) y un bloque etiquetado `STAGEv01` (etapa 1). Con un bloque que no tiene sentido y una pista en forma de clave, la hipótesis natural es XOR.

### 4. Descifrar el bloque con XOR repetido

Una sola clave de un byte (256 pruebas) no dio texto legible; tampoco sumas o restas, ni los números del `network.log`. Entonces se probó la pista tal cual, sin guiones y como cadena que se repite: `N7042218`.

```python
mem = open('work/workstation.mem','rb').read()
clave = b'N7042218'
s = mem.index(b'STAGEv01') + 8                 # el cifrado empieza justo después de «STAGEv01»
bloque = mem[s:s + 61]
print(bytes(b ^ clave[k % 8] for k, b in enumerate(bloque)).decode())
```

Salida (reproducida el 4 de octubre):

```text
checksum=ok
message=AccesoNoRegistrado
note=the order matters
```

**Cómo leerlo:** XOR con una clave repetida: el byte `k` del bloque se combina con `clave[k mod 8]`. El texto legible confirma que la clave es correcta: un acierto produce líneas con la forma `campo=valor`.

> **Corrección del registro anterior:** el SOLVE decía «alineada +1», porque en la competencia se tomó el bloque desde justo después de `STAGEv01-` (con guion). Con el guion el primer byte quedaba como `h` y la `c` de `checksum` parecía perdida. En realidad el guion (`-`) ya es el primer byte cifrado: `'-' XOR 'N' = 'c'`. Empezando justo tras `STAGEv01`, la clave alinea desde cero y no hay desfase.

## Cadena de resolución

`PNG → ZIP tras IEND → clue.txt «donde la máquina recuerda» → memoria → ROTATE-3 + STAGEv01 → XOR con N7042218 → message=AccesoNoRegistrado`

## Resultado

```text
AccesoNoRegistrado
```

Estado: **resuelto localmente**. El enunciado pide «el mensaje oculto» y no da formato de flag; **no se comprobó** qué formato aceptaba la plataforma ni consta aceptación. En la competencia se propusieron `AGETIC{AccesoNoRegistrado}` y `citc{AccesoNoRegistrado}` solo como suposiciones por retos previos; no hay evidencia a favor de ninguna.

## Qué falta / cabos sueltos

- **`ge6cso5t}` en el offset 1470:** texto plano, con apariencia de final de flag (acaba en `}`), pero no se encontró el principio. El resultado descifrado insiste en que «el orden importa» (`note=the order matters`), y el `network.log` tiene ocho números con letras griegas en un orden aparentemente aleatorio: podría tratarse de un orden de reensamblado. **No se probó esa hipótesis**; está registrada solo como idea.
- `checksum=ok`: no se identificó qué comprueba.
- La relación entre `ROTATE-3` y `rotate.py --slot 3` solo se usó para obtener la clave.

## Qué debo recordar para otro CTF

- Si un archivo tiene más bytes de los que declara su formato (PNG tras `IEND`), buscar datos añadidos: `binwalk`, `pngcheck`, `zsteg`.
- Una pista tipo `N7-04-22-18` puede ser la clave sin guiones; probar variantes.
- Un resultado legible con forma `campo=valor` confirma un descifrado XOR. Calcular `byte ^ letra_conocida` ayuda a ver el desfase (aquí `'-' ^ 'c'`).
- Los archivos con nombres como `decoy`, `old_backup`, `predates` son señuelos casi siempre.

## Aporte para el arsenal

- Automatizable: detectar datos tras `IEND`, buscar marcas (`STAGE`, `ROTATE`) en volcados, probar claves derivadas de cadenas cercanas con XOR repetido y puntuar legibilidad.
- Requiere interpretación: decidir qué pista es una clave; dejar abierto lo que no se explica.
- Falso positivo: cadenas planas como `ge6cso5t}` que parecen flags incompletas.


## Código de apoyo: extraer_stage.py

Guarda este bloque como `extraer_stage.py` si sigues los comandos que lo utilizan. Código del análisis original; no se ha vuelto a ejecutar para esta publicación.

```python
#!/usr/bin/env python3
"""INCIDENT 42: 1) ZIP pegado tras el PNG; 2) bloque STAGEv01 de la memoria descifrado con XOR repetido.
Uso: python3 scripts/extraer_stage.py work/   (carpeta con evidence.png y workstation.mem)"""
import io, sys, zipfile
base = sys.argv[1]

png = open(f'{base}/evidence.png', 'rb').read()
fin = png.rindex(b'IEND') + 8                      # IEND (4) + tipo... + CRC (4): fin real del PNG
print('PNG termina en', fin, '- sobran', len(png) - fin, 'bytes, empiezan por', png[fin:fin + 4])
z = zipfile.ZipFile(io.BytesIO(png[fin:]))
for n in z.namelist():
    print('--', n, '->', z.read(n).decode().strip())

mem = open(f'{base}/workstation.mem', 'rb').read()
i = mem.index(b'ROTATE-3:')
print('pista:', mem[i:i + 22])                      # ROTATE-3: N7-04-22-18
clave = b'N7042218'                                 # la pista sin guiones
s = mem.index(b'STAGEv01') + 8                      # el bloque cifrado empieza justo después de la marca
bloque = mem[s:s + 61]
print(bytes(b ^ clave[k % len(clave)] for k, b in enumerate(bloque)).decode())
```
