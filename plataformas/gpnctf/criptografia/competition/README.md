# Write-up: GPN24 - COMpetition

## Reto

- Plataforma: GPNCTF 2024
- Categoría: Criptografía / Reversing
- Estado: `resuelto`

**Categoría:** Crypto / Reversing
**Dificultad:** Media
**Conceptos Clave:** Cryptographic Commitment, Hash Collision, Input Parsing Exploitation, Length Extension-like Manipulation.
`#crypto`, `#ctf`, `#writeup`

---

## 1. Análisis del Reto
El reto nos conecta a un servicio remoto vía SSL que nos obliga a jugar 100 rondas de Piedra, Papel o Tijera contra un bot. Para evitar que hagamos trampa alterando nuestra jugada una vez que el bot elija la suya, el servidor implementa un esquema de **Commitment** (Compromiso Criptográfico).

### El Flujo Lógico (por ronda):
1. El usuario envía un `Commitment (hex)` (el hash de nuestra jugada secreta).
2. El bot elige y revela su jugada (`rock`, `paper` o `scissors`).
3. El usuario envía su jugada (`What did you choose?`) y una prueba (`Proof (hex)`).
4. El servidor valida la integridad de la jugada.

### La Vulnerabilidad (Source Code Analysis)
El fallo crítico se encuentra en la función de verificación del servidor:

```python
def verify(commitment, your_choice, unveil_info):
    r1, r2 = unveil_info  # Exige un desempaquetado de tupla (dos elementos)
    # Concatenación directa sin delimitadores seguros:
    return sha256(r1 + your_choice + r2).hexdigest() == commitment
```

Problema: El servidor divide el input de la prueba usando .split(), asignando los bloques a r1 y r2. Al reconstruir la cadena para calcular el SHA-256, simplemente hace r1 + your_choice + r2.

Al no haber delimitadores fijos (como un guion bajo o un caracter de control) entre las variables dentro del hash, el servidor realiza una concatenación plana. Esto nos permite controlar los extremos (r1 y r2) para "desplazar" dinámicamente el payload (your_choice) y alterar el significado de la cadena sin cambiar el hash final.
2. Estrategia de Explotación

Para ganar de forma consistente, necesitamos predecir el futuro. Generamos un Commitment base único por ronda (usando el prefijo R0XX_ para evitar el filtro de duplicados already_seen) y le agregamos un relleno controlado (PAD) al final para evitar que el string r2 quede vacío, lo cual rompería el desempaquetado de la tupla en Python (ValueError: not enough values to unpack).

Nuestra cadena base universal por ronda será:
base = b"R0XX_scissorsrockpaperPAD"

Calculamos el hash inicial de esa cadena y se lo enviamos como Commitment. Cuando el bot responde, adaptamos los valores de r1 y r2 usando el espacio como delimitador del split para reconstruir la misma cadena exacta:

|**Jugada Bot**|**Nuestro Contraataque**|**Bloque r1 (Bytes / Hex)**|**Jugada (Texto)**|**Bloque r2 (Bytes / Hex)**|**Cadena resultante en Servidor**|
|---|---|---|---|---|---|
|**rock**|`paper`|`R0XX_scissorsrock`|`paper`|`PAD`|`R0XX_scissorsrockpaperPAD`|
|**paper**|`scissors`|`R0XX_`|`scissors`|`rockpaperPAD`|`R0XX_scissorsrockpaperPAD`|
|**scissors**|`rock`|`R0XX_scissors`|`rock`|`paperPAD`|`R0XX_scissorsrockpaperPAD`|
Como el resultado de la concatenación en el servidor siempre es idéntico a la base, el SHA-256 coincide al 100%, el bypass es exitoso y ganamos la ronda de manera fraudulenta pero criptográficamente válida.

## 3. Script de Automatización (Exploit)
```python
from hashlib import sha256
import socket
import ssl
import time

HOST = "fermented-tempura-drizzled-with-candied-ponzu-gpu3.gpn24.ctf.kitctf.de"
PORT = 443

def obtener_datos_ronda(ronda, bot_choice):
    prefix = f"R{ronda:03d}_".encode()
    if bot_choice == "rock":
        return "paper", f"{(prefix + b'scissorsrock').hex()} 504144"
    elif bot_choice == "paper":
        return "scissors", f"{prefix.hex()} 726f636b7061706572504144"
    elif bot_choice == "scissors":
        return "rock", f"{(prefix + b'scissors').hex()} 7061706572504144"
    return None, None

def recv_until(s, ending):
    buffer = b""
    while ending not in buffer:
        data = s.recv(1)
        if not data: break
        buffer += data
    return buffer.decode()

def main():
    context = ssl.create_default_context()
    context.check_hostname = False
    context.verify_mode = ssl.CERT_NONE

    s = context.wrap_socket(socket.socket(socket.AF_INET, socket.SOCK_STREAM), server_hostname=HOST)
    s.connect((HOST, PORT))

    recv_until(s, b"I want to play a game...\n")

    for ronda in range(1, 101):
        recv_until(s, b"Commitment (hex): ")
        prefix = f"R{ronda:03d}_".encode()
        com_hex = sha256(prefix + b"scissorsrockpaperPAD").digest().hex()
        s.sendall(f"{com_hex}\n".encode())

        bot_line = recv_until(s, b"\n").strip()
        bot_choice = "rock" if "rock" in bot_line else "paper" if "paper" in bot_line else "scissors"

        your_choice, proof_hex = obtener_datos_ronda(ronda, bot_choice)

        recv_until(s, b"What did you choose? ")
        s.sendall(f"{your_choice}\n".encode())

        recv_until(s, b"Proof (hex): ")
        s.sendall(f"{proof_hex}\n".encode())
        time.sleep(0.005)

    print("\n[+] Explotación completada. Bandera obtenida:")
    print(s.recv(4096).decode())
    s.close()

if __name__ == "__main__":
    main()
```

## 4. Remediación (Mitigación)

Para evitar este tipo de ataques por manipulación de estructuras internas (parsing/boundary flaws), la concatenación para la verificación del hash **nunca debe ser plana**. Se debe forzar el uso de delimitadores seguros no controlables por el usuario o usar formatos de serialización rígidos (como JSON o Protocol Buffers):

```python
# Verificación Segura (Ejemplo)
def verify_fixed(commitment, your_choice, unveil_info):
    r1, r2 = unveil_info
    # El uso de delimitadores fijos rompe la capacidad de desplazar bloques libres
    return sha256(f"{r1}|{your_choice}|{r2}".encode()).hexdigest() == commitment
```

## Qué debo recordar para otro CTF

- Un hash no corrige una serialización ambigua: los límites entre campos deben ser inequívocos.
- La explotación debe explicarse a partir del parser y no solamente del script final.
