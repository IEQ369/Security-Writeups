# Operación Espejo Roto v2

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Forense. Autor: Psicohack.
- Archivo entregado: `forense.zip` (277 034 bytes; contenía `dist/evidence.raw` y `dist/SHA256SUMS`). **El ZIP ya no está en Descargas.** Se conserva el disco extraído: evidence.raw (203 MB), cuyo SHA-256 coincide con el `SHA256SUMS` que traía el ZIP (comprobado el 4 de octubre).
- Estado: **resuelto localmente**. La flag se descifró y el hash del resultado coincide con el que anunciaba el propio caso. El usuario respondió después con «llevas mucha racha», pero no consta una confirmación explícita de la plataforma.
- Registro breve anterior: SOLVE.

> Una historia de un sobre negro y un espejo roto. «Adéntrate en el caso y recupera la bandera con formato AGETIC.»

El texto del enunciado incluía por error una frase de otro reto («el nombre del reto es Contra reloj»). Se ignoró.

## Resumen

La imagen de disco guarda un caso de respuesta a incidentes (`IR-204`) «cerrado como falso positivo». Un analista subió a un servidor un objeto cifrado. La receta de la clave está en un manifiesto **borrado** del disco, y las piezas de la clave están en el historial de Firefox, **también borrado**, pero aún presente en el archivo WAL de su base de datos. Reconstruyendo ese historial se arma la clave y se descifra el objeto, que contiene la flag.

## Qué recibimos

Un ZIP con una imagen de disco de unos 194 MiB. El disco usa tabla de particiones GPT con una partición ext4 llamada `IRIS_DATA`.

## Reconocimiento

```bash
cd Espejo-Roto-v2/work/dist
sha256sum -c SHA256SUMS
mmls evidence.raw
fsstat -o 2048 evidence.raw | head -8
```

`SHA256SUMS` es el control que entrega el propio reto: dice que la imagen está intacta. `mmls` muestra la tabla de particiones:

```text
GUID Partition Table (EFI)
004:  000       0000002048   0000395263   0000393216   IRIS_EVIDENCE
```

La partición empieza en el **sector 2048**. Ese número se pasa como `-o 2048` a todos los comandos de The Sleuth Kit. `fsstat` confirma `File System Type: Ext4`, volumen `IRIS_DATA`. (Resultados guardados: mmls-fsstat.txt.)

## Resolución paso a paso

### 1. Listar archivos, incluidos los borrados

```bash
fls -r -p -o 2048 evidence.raw | grep -v cache2/entries | grep -v '^d/d'
```

`fls` lista archivos; `-r` recursivo; `-p` ruta completa. Los borrados aparecen con un `*`. El filtro oculta 73 entradas de la caché de Firefox y los directorios para ver lo no estándar:

```text
r/r * 144: home/analyst/.cache/.objects/a2184cb287bb5e.json
r/r * 145: home/analyst/.cache/.objects/f85ee3711f4b23.tmp
r/r * 146: home/analyst/.cache/.objects/8869ba2a5125a1.tmp
r/r   128: home/analyst/.local/share/sync-agent/objects/8fffd093392af040.obj
r/r   129: home/analyst/.mozilla/firefox/k4m2.default-release/places.sqlite
r/r   130: home/analyst/.mozilla/firefox/k4m2.default-release/places.sqlite-wal
r/r   132: home/analyst/Documents/IR-204.txt
r/r * 150: home/intern/.cache/.objects/ade6d0709c0757.tmp
r/r   142: var/log/sync-agent/activity.log
```

El número tras `r/r` es el **inodo** (`icat` lo usa para leer el archivo). Resultado completo en fls-sin-cache.txt.

### 2. Leer el contexto humano antes de lo técnico

Para leer todo se extrajo cada archivo con `icat` hacia una carpeta `work/x` (lectura sin montar el disco):

```bash
fls -rp -o 2048 evidence.raw | grep -E '^r/r' | while IFS=$'\t' read a p; do
  i=${a#r/r }; i=${i#\* }; i=${i%:}
  mkdir -p "x/$(dirname "$p")"; icat -o 2048 evidence.raw "$i" > "x/$p"
done
```

Lo que dicen los textos:

- `.bash_history` del analista: `/usr/local/bin/cache-sync --resume R-C2CF5DA6`. Un identificador de ejecución: **R-C2CF5DA6**.
- `IR-204.txt`: «Estado visible: cerrado como falso positivo… no confiar en esta conclusión sin verificarla». Indica que hay que mirar la imagen, no el ticket.
- `var/log/sync-agent/activity.log`: el analista (uid 1000) ejecutó `run=R-C2CF5DA6`, con `object=8fffd093392af040.obj` «sealed» y una subida (`upload=drop-gw...:8443 bytes=304`). Es el objeto que importa.
- Hay otra ejecución, `LAB-031`, del *interno* (uid 1001), marcada `training`. Es un señuelo de práctica.

### 3. Recuperar el manifiesto borrado

**Observación:** hay archivos `.tmp` borrados en `.objects`, y el objeto cifrado `8fffd093392af040.obj` no se puede descifrar sin saber cómo se cifró. Se prueba el inodo 145:

```bash
icat -o 2048 evidence.raw 145 | zcat | python3 -m json.tool
```

`zcat` descomprime gzip (el `.tmp` empieza con la firma `1f 8b`). El resultado, guardado como manifiesto-IR-204-borrado.json, es la receta:

- Cifrado `aes-256-cbc`, derivación `pbkdf2` con `sha256` y **326 894 iteraciones**, formato `openssl enc` («Salted__»).
- Clave = valores del parámetro `etag` de las visitas de Firefox con `sid=R-C2CF5DA6`, en orden de `visit_date`, decodificados como base64url, concatenados, más `:` más `/etc/machine-id[8:20]`.
- `receipt_count: 12`, `source: "Firefox WAL committed state before cleanup"`: **12 recibos**, y estado del WAL «antes de la limpieza».
- Hash del objeto y del texto claro final (`plaintext_sha256 = 9ca499a1…`). Sirve para validar el descifrado.

Los otros dos `.tmp`: `8869ba…` es el objeto cifrado de otro caso, y el del interno (`ade6d0…`, caso `LAB-031`) tiene como clave una frase literal `training-…`; no es la ruta correcta (otro caso, otra ejecución).

### 4. Buscar las 12 visitas: SQLite normal no las muestra

```bash
cd work/ff2   # copia de places.sqlite y places.sqlite-wal
python3 -c "
import sqlite3
c=sqlite3.connect('places.sqlite')
for r in c.execute(\"select v.id,v.visit_date,p.url from moz_historyvisits v join moz_places p on p.id=v.place_id where p.url like '%sid=%' order by v.visit_date\"): print(r)"
```

(No hay `sqlite3` en el sistema, por eso se usa Python.) Aparecen visitas a `https://static-cache.intra.invalid/pixel.gif?sid=R-8D83C822&etag=...` y otros `sid` (señuelos), pero **ninguna con `R-C2CF5DA6`**. Abrirla con SQLite aplica el WAL, y el WAL borró las visitas. Aviso aprendido en el trabajo: SQLite fusiona y borra el archivo `-wal` de la copia; hay que trabajar sobre copias y quitar permisos de escritura.

### 5. Leer el WAL a mano

El WAL (*write-ahead log*) guarda páginas modificadas antes de volcarlas a la base. Cada transacción termina con un frame «de commit». Un análisis del encabezado y de los frames (tamaño de página 4096, 14 frames) mostró:

```text
frame página commit contiene "C2CF5DA6"
  4     8     0      True
  5     9     0      True
  6    10    10      False      <- commit de la 1.ª transacción
  ...
 13     9    10      False      <- commit de la 2.ª transacción (limpieza)
```

Dos transacciones: la primera (frames 0–6) añade las visitas, y la segunda (7–13) las elimina. El manifiesto pedía «estado confirmado **antes** de la limpieza», es decir, aplicar solo los frames de la primera transacción.

### 6. Reconstruir la base y armar la clave

El script resolver_espejo.py hace todo el recorrido desde la imagen. Guardarlo y ejecutarlo así (necesita `icat` de TSK y `openssl`):

```bash
cd Espejo-Roto-v2
python3 scripts/resolver_espejo.py work/dist/evidence.raw /tmp/espejo_out
```

Las partes relevantes del código Python:

```python
# aplicar solo los frames de la 1.ª transacción sobre la base original
for i in range(nfr):
    o = 32 + i * (ps + 24)                          # cabecera WAL 32 B; cada frame = 24 B + página
    pg, commit = struct.unpack('>II', wal[o:o + 8])
    db[(pg - 1) * ps:pg * ps] = wal[o + 24:o + 24 + ps]
    if commit:                                      # primer commit: parar
        db = db[:commit * ps]                       # tamaño de la base tras la transacción
        break
```

El segundo campo de la cabecera de cada frame decide el final de transacción: `commit` distinto de 0 indica que ahí termina una. Resultado: base reconstruida con 85 visitas; con el WAL completo quedaron 73 y ninguna de `R-C2CF5DA6` (comprobado).

```python
for vid, vd, url in c.execute("select ... order by v.visit_date, v.id"):
    q = urllib.parse.parse_qs(urllib.parse.urlparse(url).query)
    if q.get('sid') == [sid]:
        e = q['etag'][0]
        partes.append(base64.urlsafe_b64decode(e + '=' * (-len(e) % 4)).decode())
clave = ''.join(partes) + ':' + mid[8:20]
```

Los 12 `etag` (ids de visita 74–85) son: `MDkwNg NGQxZA ZjNkMw MDFmYQ OGNkNA MGUwYg OGRiNQ NWVmMQ MWNhMw ZDFiMA ZTc2Yw YWFjZA`. **Error que ocurrió:** unir los 12 en una sola cadena y decodificarla da basura (`0906\x03FC\x16@...`) porque cada trozo termina con su propio relleno omitido. Cada `etag` se decodifica **por separado** (`0906`, `4d1d`, `f3d3`…) y luego se concatenan. Resultado:

```text
09064d1df3d301fa8cd40e0b8db55ef11ca3d1b0e76caacd:a14c3e4982b9
```

El sufijo es `machine-id[8:20]` de `97ee8e6aa14c3e4982b98261fc7caf84`.

### 7. Descifrar el objeto

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -iter 326894 -md sha256 -pass file:pw.txt \
  -in 8fffd093392af040.obj -out plain.bin
sha256sum plain.bin
zcat plain.bin > inner && tar -xvf inner
cat export/IR-204/report.json
```

`-pass file:pw.txt` toma la clave del archivo (sin salto de línea). Antes, el intento con la cadena unida dio `bad decrypt`: es la señal de que la clave es incorrecta. Con la correcta, `sha256sum` da `9ca499a1…`, **igual** al `plaintext_sha256` del manifiesto. Eso demuestra la clave sin depender de ver una flag. Después, `gzip` → `tar` → `report.json` (report.json).

El 4 de octubre se repitió el recorrido completo con `resolver_espejo.py`; coincidió el hash y apareció la misma flag.

## Cadena de resolución

`GPT → ext4 (-o 2048) → archivos borrados → manifiesto (inodo 145) → receta de clave → WAL de Firefox (1.ª transacción) → 12 etags base64url por separado + machine-id → openssl aes-256-cbc/pbkdf2 → gzip → tar → report.json`

## Flag

```text
AGETIC{analyst_0218_d47891415bed}
```

Estado: resuelta localmente, verificada por hash.

## Qué debo recordar para otro CTF

- «Cerrado como falso positivo» en una imagen forense es una invitación a mirar lo borrado.
- En `fls`, `*` marca borrados. `icat` los lee por inodo.
- Un `places.sqlite-wal` puede contener lo que la base ya no tiene. No lo abras con SQLite sin copiarlo antes.
- Un manifiesto con un hash final permite saber si la clave es correcta antes de buscar la flag.
- Decodificar Base64 por partes cuando cada parte tiene su propio relleno.

## Aporte para el arsenal

- Automatizable: extraer todo el ext4 con `icat`, listar WAL por frames y transacciones, buscar un manifiesto JSON con `key_recipe`.
- Requiere interpretación: elegir qué transacción es el estado correcto y qué `sid` pertenece al caso.
- Falso positivo: los `sid` de las otras ejecuciones y el objeto de entrenamiento `LAB-031`.


## Código de apoyo: resolver_espejo.py

Guarda este bloque como `resolver_espejo.py` si sigues los comandos que lo utilizan. Código del análisis original; no se ha vuelto a ejecutar para esta publicación.

```python
#!/usr/bin/env python3
"""Espejo Roto v2: de evidence.raw a report.json.
Uso: python3 resolver_espejo.py work/dist/evidence.raw SALIDA/
Requiere The Sleuth Kit (icat) y openssl. Solo lee la imagen."""
import base64, gzip, io, json, os, struct, sqlite3, subprocess, sys, tarfile, urllib.parse

IMG, OUT = sys.argv[1], sys.argv[2]
OFFSET = '2048'                  # sector de inicio de la partición ext4 (mmls)
# números de inodo vistos con `fls -r -p -o 2048` (ver la nota del reto)
INODOS = {'manifiesto': 145, 'places': 129, 'wal': 130, 'machine-id': 49, 'objeto': 128}
os.makedirs(OUT, exist_ok=True)

def icat(ino):
    return subprocess.run(['icat', '-o', OFFSET, IMG, str(ino)], check=True, capture_output=True).stdout

# 1) Manifiesto borrado: gzip con JSON que explica cómo se cifró el objeto
man = json.loads(gzip.decompress(icat(INODOS['manifiesto'])))
rec = man['key_recipe']
sid = rec['filter_query_parameter']['sid']
vq = rec['value_query_parameter']
it = man['crypto']['iterations']
print('manifiesto:', man['crypto']['cipher'], man['crypto']['kdf'], it, 'sid =', sid, 'campo =', vq)

# 2) places.sqlite + WAL: aplicar solo la 1.ª transacción (frames hasta el primer commit)
db = bytearray(icat(INODOS['places'])); wal = icat(INODOS['wal'])
ps = struct.unpack('>I', wal[8:12])[0]
nfr = (len(wal) - 32) // (ps + 24)
maxpg = 0
for i in range(nfr):
    o = 32 + i * (ps + 24)
    pg, commit = struct.unpack('>II', wal[o:o + 8])
    db[(pg - 1) * ps:pg * ps] = wal[o + 24:o + 24 + ps]
    maxpg = max(maxpg, pg)
    if commit:                   # primer frame de commit: se detiene aquí
        db = db[:commit * ps]    # 'commit' = tamaño de la BD en páginas tras la transacción
        print('primer commit en el frame', i, '-> BD de', commit, 'páginas')
        break
open(os.path.join(OUT, 'committed1.sqlite'), 'wb').write(db)

# 3) Etags de las visitas de esa sesión, en orden de fecha; cada uno es base64url independiente
c = sqlite3.connect(os.path.join(OUT, 'committed1.sqlite'))
partes = []
for vid, vd, url in c.execute("select v.id, v.visit_date, p.url from moz_historyvisits v "
                               "join moz_places p on p.id=v.place_id order by v.visit_date, v.id"):
    q = urllib.parse.parse_qs(urllib.parse.urlparse(url).query)
    if q.get('sid') == [sid]:
        e = q[vq][0]
        partes.append(base64.urlsafe_b64decode(e + '=' * (-len(e) % 4)).decode())
print(len(partes), 'etags ->', ''.join(partes))
mid = icat(INODOS['machine-id']).decode().strip()
clave = ''.join(partes) + ':' + mid[8:20]
open(os.path.join(OUT, 'pw.txt'), 'w').write(clave)

# 4) Descifrar el objeto subido
obj = os.path.join(OUT, 'objeto.obj'); open(obj, 'wb').write(icat(INODOS['objeto']))
plain = subprocess.run(['openssl', 'enc', '-d', '-aes-256-cbc', '-pbkdf2', '-iter', str(it), '-md', 'sha256',
                        '-pass', 'file:' + os.path.join(OUT, 'pw.txt'), '-in', obj],
                       check=True, capture_output=True).stdout
import hashlib
print('sha256 plaintext:', hashlib.sha256(plain).hexdigest(), '(manifiesto:', man['plaintext_sha256'] + ')')
tar = tarfile.open(fileobj=io.BytesIO(gzip.decompress(plain)))
for m in tar.getmembers():
    print(m.name); print(tar.extractfile(m).read().decode())
```
