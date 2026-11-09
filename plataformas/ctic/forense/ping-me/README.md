# Write-up: Ping me

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Forense
- Puntos: 300
- Archivo: `Ping-me.pcap`
- Estado: `resuelto`

> Te acaban de contratar en una empresa de ciberseguridad y estás en un SOC mirando el tráfico de red que se dirigía hacia fuera de la empresa cliente. Dicho tráfico, aunque parezca habitual, por alguna razón te hace sospechar...
>
> Demuéstrale a tu jefe que tienes vista de halcón.

## Resumen

La captura contiene una secuencia muy rápida de mensajes ICMP Echo. Cada solicitud saliente lleva solamente dos bytes de datos y su respuesta devuelve exactamente los mismos bytes. Al conservar solo las solicitudes y concatenar sus payloads se obtiene una cadena UTF-16LE, un carácter por paquete. La cadena reconstruida contiene la bandera original entre llaves.

## Reconocimiento

```bash
file Ping-me.pcap
capinfos Ping-me.pcap
tshark -r Ping-me.pcap -q -z io,phs
```

Resultados importantes:

- PCAP clásico válido con encapsulación Ethernet.
- 102 paquetes y 6.156 bytes de archivo.
- Duración total de 1,919710 segundos.
- 98 paquetes ICMP, 2 ARP y 2 STP.
- 49 ICMP Echo Request y 49 Echo Reply.
- Todos los mensajes ICMP transportan exactamente dos bytes en `data.data`.

La jerarquía de protocolos concentra casi toda la captura en ICMP. El título `Ping me.` y la descripción sobre tráfico saliente refuerzan que debemos comparar las solicitudes Echo enviadas con sus respuestas.

## Descubrimiento humano, paso a paso

### 1. Separar solicitudes y respuestas

En Wireshark se comienza con:

```text
icmp
```

Los paquetes aparecen en pares:

- Tipo `8`: Echo Request, desde `192.168.122.90` hacia `192.168.122.203`.
- Tipo `0`: Echo Reply, en la dirección contraria.

La especificación de ICMP indica que los datos recibidos en un Echo Request se devuelven en el Echo Reply. Por eso ambos integrantes de cada par contienen el mismo payload; si se extraen los dos sentidos, cada carácter queda duplicado.

Referencia: [RFC 792 — Echo or Echo Reply Message](https://www.rfc-editor.org/rfc/rfc792.html#page-14).

Como el enunciado habla del tráfico que salía de la empresa, se conservan únicamente las solicitudes:

```text
icmp.type == 8
```

### 2. Inspeccionar el campo Data

Al abrir el primer Echo Request, Wireshark muestra:

```text
Internet Control Message Protocol
    Type: 8 (Echo (ping) request)
    Data (2 bytes)
        Data: 4d00
        Length: 2
```

Los siguientes payloads son:

```text
4d00 6f00 7200 7400 6500 7200 7500 6500 6c00 6f00 ...
```

Si se mira solamente el primer byte de cada bloque:

```text
4d 6f 72 74 65 72 75 65 6c 6f
 M  o  r  t  e  r  u  e  l  o
```

Esto revela texto de inmediato.

### 3. Entender correctamente los dos bytes

ICMP **no está limitado a dos bytes de datos**. Un Echo Request posee una zona `Data` de longitud variable; en esta captura el creador decidió enviar exactamente dos bytes por solicitud.

El patrón completo es compatible con UTF-16LE:

```text
4d 00 → U+004D → M
6f 00 → U+006F → o
72 00 → U+0072 → r
```

En UTF-16LE, cada carácter básico ocupa un código de 16 bits escrito con el byte menos significativo primero. Para letras ASCII, el segundo byte suele ser `00`. Por eso tomar el penúltimo byte de cada frame funciona en este reto, pero no es una regla de ICMP ni una técnica universal.

El último payload es:

```text
0a00 → U+000A → salto de línea
```

Esto confirma aún más la interpretación: no es simplemente «un carácter más relleno», sino una secuencia textual de unidades de dos bytes.

### 4. Conservar solo una copia de cada carácter

Hay 49 solicitudes y 49 respuestas. Cada respuesta invierte origen y destino, cambia el tipo de `8` a `0` y refleja los mismos datos.

Si se procesan todos los ICMP, el inicio sería:

```text
MMoorrtteerruueelloo...
```

Al filtrar únicamente `icmp.type == 8`, la secuencia se conserva una sola vez y en el orden correcto.

## Extracción con TShark

Primero se crea una tabla que todavía conserva la procedencia de cada fragmento:

```bash
tshark -r Ping-me.pcap \
  -Y 'icmp.type == 8' \
  -T fields -E header=y -E separator='|' \
  -e frame.number -e frame.time_relative \
  -e ip.src -e ip.dst -e icmp.ident -e icmp.seq \
  -e data.len -e data.data
```

Las primeras filas son:

```text
frame.number|frame.time_relative|ip.src|ip.dst|icmp.ident|icmp.seq|data.len|data.data
4|1.090978000|192.168.122.90|192.168.122.203|15433|1|2|4d00
6|1.093004000|192.168.122.90|192.168.122.203|352|1|2|6f00
8|1.094208000|192.168.122.90|192.168.122.203|27809|1|2|7200
10|1.095878000|192.168.122.90|192.168.122.203|40517|1|2|7400
```

Después de verificar que todos los payloads relevantes miden dos bytes y siguen el mismo patrón, se concatenan y decodifican como UTF-16LE:

```bash
tshark -r Ping-me.pcap \
  -Y 'icmp.type == 8 && data.len == 2' \
  -T fields -e data.data \
  | tr -d ':\n' \
  | xxd -r -p \
  | iconv -f UTF-16LE -t UTF-8
```

Qué hace cada parte:

- TShark extrae `data.data` solamente de los Echo Request.
- `tr` une las líneas y elimina posibles separadores `:` del hexadecimal.
- `xxd -r -p` convierte el texto hexadecimal en los bytes originales.
- `iconv` interpreta esos bytes como UTF-16LE y produce texto UTF-8 legible.

Resultado:

```text
Morteruelo2022{1d7a90a63039831c7fcaa53b766d5b2d}
```

La secuencia contiene también un salto de línea después de `}`, procedente del último payload `0a00`.

### Método corto que funciona aquí, pero es menos general

También se puede seleccionar solo el primer byte de cada payload:

```bash
tshark -r Ping-me.pcap \
  -Y 'icmp.type == 8 && data.len == 2' \
  -T fields -e data.data \
  | cut -c1-2 \
  | tr -d '\n' \
  | xxd -r -p
```

Esto equivale a descartar cada `00`. Funciona porque todos los caracteres pertenecen al rango ASCII, pero perdería información si aparecieran caracteres UTF-16LE cuyo byte alto no fuera cero. Interpretar los dos bytes es una solución más correcta y reutilizable.

## Cadena de resolución

`PCAP → jerarquía de protocolos → 98 ICMP → pares Request/Reply → conservar tráfico saliente tipo 8 → Data de 2 bytes → reconocer patrón carácter/00 → concatenar → UTF-16LE → marcador con flag`

## Flag

Marcador original reconstruido:

```text
Morteruelo2022{1d7a90a63039831c7fcaa53b766d5b2d}
```

Contenido situado dentro de las llaves y utilizado como respuesta del reto:

```text
1d7a90a63039831c7fcaa53b766d5b2d
```

Si una edición de la plataforma exige el envoltorio CIDSI, la representación correspondiente sería `cidsi{1d7a90a63039831c7fcaa53b766d5b2d}`; el valor probado por la evidencia es el contenido hexadecimal anterior.

## Qué debo recordar para otro CTF

- Los Echo Request pueden incluir una carga útil de longitud variable; dos bytes no son un límite de ICMP.
- Un Echo Reply devuelve los datos recibidos. Antes de concatenar hay que separar tipos o direcciones para no duplicar cada fragmento.
- `icmp.type == 8` selecciona solicitudes; `icmp.type == 0`, respuestas.
- No se debe asumir que `00` es relleno. Un patrón `letra 00 letra 00` puede indicar UTF-16LE.
- El «penúltimo byte del frame» funcionó por la construcción específica del reto. Es más seguro extraer `data.data`, estudiar su estructura e interpretar el encoding completo.
- Una secuencia de pings extremadamente rápida, payloads diminutos y uniformes, identificadores cambiantes y texto al reconstruir forman una evidencia conjunta mucho más fuerte que una sola rareza.
- Se guarda siempre frame, tiempo, tipo, origen, destino, identificador, secuencia, longitud y payload antes de transformar los datos.

## Candidato para el arsenal forense

- Inventariar tipos, direcciones, tamaños e intervalos temporales del tráfico ICMP.
- Emparejar Echo Request y Echo Reply mediante direcciones, identificador, secuencia y payload.
- Mostrar por separado las reconstrucciones de solicitudes, respuestas y ambos sentidos, avisando cuando una es duplicación reflejada.
- Agrupar por longitud de `data.data` y detectar posiciones de byte constantes, como todos los bytes pares iguales a `00`.
- Probar interpretaciones justificadas: bytes crudos, ASCII/UTF-8, UTF-16LE, UTF-16BE y hexadecimal, sin destruir la evidencia original.
- Detectar caracteres o archivos distribuidos por paquete y conservar el orden temporal salvo que exista un índice explícito.
- Explicar por qué se seleccionó una interpretación y puntuar estructura de texto, magic bytes y formatos de flag sin afirmar automáticamente que cualquier texto legible es correcto.
