# Tráfico Sospechoso

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Forense / red. Autor: drei.
- Archivo: `trafico_docker.zip` (ya no está en Descargas). Contenido extraído en  (`docker-compose.yml`, `Dockerfile`, `README.txt`, `files/beacon.py`).
- Estado: **aceptado**. Tras entregar la flag, el usuario escribió «bien bien es correcto».
- Registro breve anterior: SOLVE.

> Un host interno exfiltra datos. Se entrega un entorno Docker que genera tráfico en vivo; hay que capturarlo, identificar el canal encubierto real y recuperar la flag. «No hay uno solo: el tráfico trae más de un candidato con la misma pinta, y solo uno decodifica a algo que realmente tiene sentido.»

## Resumen

El contenedor ejecuta `beacon.py`, que manda consultas DNS con datos escondidos en el subdominio. Hay **tres canales** (`metrics`, `telemetry`, `diagnostics`). Dos decodifican a flags con formato correcto pero falsas; el canal real está cifrado con un XOR de un byte que se descubre probando las 256 claves. La resolución de competencia **no levantó Docker**: se leyó el script, porque trae los fragmentos de los tres canales. Esta nota explica ambos caminos: el que se hizo y el de captura en vivo que pedía el enunciado (este último **no se ejecutó**).

## Qué recibimos

```bash
cd Trafico-Sospechoso/work/trafico_docker
cat README.txt docker-compose.yml Dockerfile
```

- `README.txt`: levantar con `docker compose up -d --build`, capturar con `tcpdump -i eth0 -w /tmp/captura.pcap` 2–3 minutos dentro del contenedor, copiar con `docker cp` y analizar.
- `docker-compose.yml`: contenedor `reto4-host-interno` en una red `10.20.30.0/24` con IP `10.20.30.55`.
- `Dockerfile`: Debian con Python, `tcpdump` y `ping`; ejecuta `/opt/beacon.py`.
- `files/beacon.py`: el programa que genera el tráfico. Es la pieza clave.

## Reconocimiento

**Observación:** antes de ejecutar un entorno ajeno, se lee su código. Las reglas del equipo piden análisis estático primero. Un script de 100 líneas se lee en minutos.

```bash
cat files/beacon.py
```

Lo que revela:

- Destino de todas las consultas: `10.20.30.1:53` (UDP).
- Una lista de dominios de ruido (`www.google.com`, `api.github.com`…).
- Tres listas de fragmentos hexadecimales: `METRICS_FRAGS`, `TELEMETRY_FRAGS`, `DIAGNOSTICS_FRAGS`.
- Los comentarios del propio script describen los roles: `METRICS` es «hex plano… flag con formato válido y FALSA»; `TELEMETRY` es «el canal real, con un XOR de un byte»; `DIAGNOSTICS` es el «señuelo gemelo del real… con OTRA clave».
- `make_schedule()` construye nombres de consulta así: `NN.<fragmento hex>.<canal>.citc-ctf.lab`, los mezcla con ruido y los **baraja** (`random.shuffle`). El número `NN` es el índice del fragmento.

**Cuidado:** que los comentarios digan cuál es real no se toma como una prueba; se confirma con el contenido de cada canal.

## Resolución paso a paso

### Camino que se siguió: decodificar desde el script

**Qué hace:** extraer cada lista de fragmentos con una expresión regular, unirla en orden, pasar de hexadecimal a bytes y probar las 256 claves XOR posibles. Solo se imprimen resultados que son texto imprimible.

```bash
python3 - <<'E'
import re
src = open('files/beacon.py').read()
def g(n):                     # une los fragmentos hex de una lista y los convierte a bytes
    lst = re.search(n + r'_FRAGS = \[(.*?)\]', src, re.S).group(1)
    return bytes.fromhex(''.join(re.findall(r'"([0-9a-f]+)"', lst)))
for n in ('METRICS', 'TELEMETRY', 'DIAGNOSTICS'):
    b = g(n); print(n, b)
    for k in range(256):
        t = bytes(x ^ k for x in b)
        if all(32 <= c < 127 for c in t): print(' ', k, t)
E
```

**Salida que importa** (hay muchas líneas de basura imprimible; solo unas pocas tienen sentido):

```text
METRICS b'citc{c4n4l_equivoc4d0_sigu3_busc4nd0}'          (clave 0: sin cifrar)
TELEMETRY ... 90 b'citc{dns_tunn3l1ng_3s_un_r13sg0_r34l}'
DIAGNOSTICS ... clave 0x3c: citc{ru1d0_disfr4z4d0_d3_s3n4l}
```

**Interpretación:**

| Canal | Cifrado | Texto | Veredicto |
|---|---|---|---|
| `metrics` | ninguno (hex plano) | `citc{c4n4l_equivoc4d0_sigu3_busc4nd0}` | señuelo: «canal equivocado, sigue buscando» |
| `diagnostics` | XOR `0x3c` | `citc{ru1d0_disfr4z4d0_d3_s3n4l}` | señuelo gemelo: «ruido disfrazado de señal» |
| `telemetry` | XOR `0x5a` (90) | `citc{dns_tunn3l1ng_3s_un_r13sg0_r34l}` | **real**: habla del propio ataque, DNS tunneling |

El filtro «formato `citc{...}` y texto imprimible» da **dos** candidatos (`telemetry` y `diagnostics`); el enunciado lo advierte. Se distinguen por lo que *dicen*: los dos señuelos se describen a sí mismos como falsos, y el tercero describe la técnica real.

### Camino de captura en vivo (no ejecutado)

Esto es lo que pedía el README. No se hizo durante la competencia ni en este archivado (Docker tampoco se levantó para este informe), así que **la salida de los pasos con captura real no se conserva**. Con `docker compose` instalado:

```bash
docker compose up -d --build
docker exec -it reto4-host-interno bash
tcpdump -i eth0 -w /tmp/captura.pcap     # 2–3 minutos, luego Ctrl+C; después: exit
docker cp reto4-host-interno:/tmp/captura.pcap .
docker compose down
```

Extraer los nombres consultados y reconstruir los canales con el script reconstruir_canales.py:

```bash
tshark -r captura.pcap -Y 'dns.flags.response==0' -T fields -e dns.qry.name \
  | python3 scripts/reconstruir_canales.py
```

El script (Python) lee los nombres, descarta los de ruido, agrupa por canal usando la tercera etiqueta, **ordena los fragmentos por su índice `NN`** (porque llegan barajados) y prueba las 256 claves:

```python
canales[p[2]][int(p[0])] = p[1]          # canal -> {indice: hex}
datos = bytes.fromhex(''.join(frags[i] for i in sorted(frags)))
for k in range(256):
    t = bytes(b ^ k for b in datos)
    if all(32 <= c < 127 for c in t): print(...)
```

**Comprobado hoy sin Docker:** se generaron 43 consultas con la propia función `make_schedule()` de `beacon.py` (guardadas en consultas-simuladas.txt, son **simuladas, no una captura**) y el script recuperó los tres mensajes anteriores. Esto prueba que el procesamiento funciona sobre nombres con el mismo formato, no que se hiciera la captura.

## Cadena de resolución

`Docker + beacon.py → leer script (sin ejecutar) → 3 canales DNS NN.hex.canal → hex plano / XOR 0x3c / XOR 0x5a por fuerza bruta → elegir por significado → flag`

## Flag

```text
citc{dns_tunn3l1ng_3s_un_r13sg0_r34l}
```

Estado: aceptada (confirmación del usuario). Obtenida del script, no de una captura viva.

## Corrección del registro anterior

El SOLVE y el índice la llamaban «candidata». El usuario confirmó después que era correcta.

## Qué debo recordar para otro CTF

- Si el reto entrega el código del emisor, leerlo ahorra la captura. Pero no es la ruta que el autor describe; en otro reto el script podría no traer los datos.
- Dos resultados «válidos» obligan a leer el contenido: el formato no basta.
- En DNS tunneling el dato viaja en la etiqueta del subdominio. Los índices numéricos indican orden de reensamblado.
- XOR de un byte: probar las 256 claves y filtrar por texto imprimible.

## Aporte para el arsenal

- Automatizable: extraer qnames de un PCAP, agrupar por dominio base, reordenar por índice, probar XOR de 1 byte.
- Requiere criterio humano: decidir cuál de varios textos válidos tiene sentido.
- Falso positivo: cualquier flag con el formato esperado dentro de un canal que se describe como señuelo.


## Código de apoyo: beacon.py

Guarda este bloque como `beacon.py` si sigues los comandos que lo utilizan. Código del análisis original; no se ha vuelto a ejecutar para esta publicación.

```python
#!/usr/bin/env python3
import random
import socket
import struct
import subprocess
import threading
import time

DEST_IP = "10.20.30.1"
DEST_PORT = 53

NOISE_DOMAINS = [
    "update.citc.gob.bo", "translate.google.com", "www.linkedin.com",
    "fonts.googleapis.com", "actualizaciones.microsoft.com", "www.instagram.com",
    "api.github.com", "ntp.citc.gob.bo", "cdn.cloudflare.com",
    "s3.amazonaws.com", "www.google.com", "www.wikipedia.org",
    "docs.citc.gob.bo", "www.facebook.com", "www.youtube.com",
    "correo.citc.gob.bo",
]

# Canal senuelo: hex plano (sin XOR). Decodifica directo pero da una flag
# con formato valido y FALSA.
METRICS_FRAGS = [
    "63697463", "7b63346e", "346c5f65", "71756976", "6f633464",
    "305f7369", "6775335f", "62757363", "346e6430", "7d",
]

# Canal real: los mismos bytes de la flag real, ya con un XOR de un byte
# aplicado de antemano. La clave nunca se escribe en ningun lado de este
# entorno (ni aca ni en la imagen) -- se recupera por fuerza bruta de las
# 256 posibilidades sobre la captura.
TELEMETRY_FRAGS = [
    "39332e39", "213e3429", "052e2f34", "3469366b", "343d0569",
    "29052f34", "05286b69", "293d6a05", "28696e36", "27",
]

# Canal señuelo GEMELO del real: mismo aspecto (hex + XOR de un byte, con
# OTRA clave), pero decodifica a una flag con formato valido y FALSA. Se
# agrega para que "fuerza bruta de 256 + quedarse con el primer resultado
# que da texto imprimible con formato citc{...}" ya no alcance por si
# sola -- ahora ese filtro da DOS resultados validos y hay que leer el
# contenido para decidir cual tiene sentido tematico.
DIAGNOSTICS_FRAGS = [
    "5f55485f", "474e490d", "580c6358", "554f5a4e", "08460858",
    "0c63580f", "634f0f52", "085041",
]


def build_dns_query(qname: str, txid: int) -> bytes:
    flags = 0x0100  # standard query, recursion deseada
    header = struct.pack(">HHHHHH", txid, flags, 1, 0, 0, 0)
    labels = b"".join(
        struct.pack("B", len(label)) + label.encode("ascii")
        for label in qname.split(".")
    )
    question = labels + b"\x00" + struct.pack(">HH", 1, 1)  # QTYPE=A QCLASS=IN
    return header + question


def make_schedule():
    events = []
    for i, frag in enumerate(METRICS_FRAGS):
        events.append(f"{i:02d}.{frag}.metrics.citc-ctf.lab")
    for i, frag in enumerate(TELEMETRY_FRAGS):
        events.append(f"{i:02d}.{frag}.telemetry.citc-ctf.lab")
    for i, frag in enumerate(DIAGNOSTICS_FRAGS):
        events.append(f"{i:02d}.{frag}.diagnostics.citc-ctf.lab")
    events += random.choices(NOISE_DOMAINS, k=15)
    random.shuffle(events)
    return events


def beacon_loop():
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    schedule = make_schedule()
    txid = random.randint(0, 0xFFFF)
    while True:
        for qname in schedule:
            txid = (txid + 1) & 0xFFFF
            try:
                sock.sendto(build_dns_query(qname, txid), (DEST_IP, DEST_PORT))
            except OSError:
                pass
            time.sleep(random.uniform(1.0, 2.5))


def noise_ping_loop():
    while True:
        time.sleep(random.uniform(4.0, 8.0))
        subprocess.run(
            ["ping", "-c", "1", "-W", "1", DEST_IP],
            stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
        )


if __name__ == "__main__":
    threading.Thread(target=noise_ping_loop, daemon=True).start()
    beacon_loop()
```

## Código de apoyo: reconstruir_canales.py

Guarda este bloque como `reconstruir_canales.py` si sigues los comandos que lo utilizan. Código del análisis original; no se ha vuelto a ejecutar para esta publicación.

```python
#!/usr/bin/env python3
"""Tráfico Sospechoso: de nombres DNS (uno por línea) a los mensajes de cada canal.
Uso:  tshark -r captura.pcap -Y 'dns.flags.response==0' -T fields -e dns.qry.name | python3 reconstruir_canales.py
Cada consulta tiene la forma  NN.<hex>.<canal>.citc-ctf.lab ."""
import sys, collections

canales = collections.defaultdict(dict)          # canal -> {indice: hex}
for linea in sys.stdin:
    p = linea.strip().lower().split('.')
    if len(p) >= 4 and p[-2:] == ['citc-ctf', 'lab'] and p[0].isdigit():
        canales[p[2]][int(p[0])] = p[1]          # el índice evita depender del orden de llegada

for canal, frags in sorted(canales.items()):
    datos = bytes.fromhex(''.join(frags[i] for i in sorted(frags)))
    print(f'== {canal}: {len(frags)} fragmentos, {len(datos)} bytes')
    for k in range(256):                          # XOR de 1 byte: probar las 256 claves
        t = bytes(b ^ k for b in datos)
        if all(32 <= c < 127 for c in t):         # solo texto imprimible
            print(f'   clave {k:#04x} ({k:3d}): {t.decode()}')
```
