# Write-up: El Rastro del Paquete Fantasma

## Reto

- Plataforma: CTF SCESI
- Nombre: El Rastro del Paquete Fantasma
- Categoría: Forensics
- Puntos: 150
- Archivo/objetivo: `e79fbd68258cd5c3f1e667be066286e6.pcap`
- Estado: `resuelto`

> El equipo de monitoreo de red capturó tráfico sospechoso de una estación de trabajo comprometida. Se sospecha que un atacante usó un canal encubierto para exfiltrar información sensible camuflándola entre tráfico "normal". Analiza la captura, identifica el canal de exfiltración, reconstruye los datos robados y recupera la flag.

## Reconocimiento

```bash
file e79fbd68258cd5c3f1e667be066286e6.pcap
capinfos e79fbd68258cd5c3f1e667be066286e6.pcap
tshark -r e79fbd68258cd5c3f1e667be066286e6.pcap -q -z io,phs
```

**Hallazgo:** Es un PCAP válido. Contiene solamente 66 paquetes capturados durante 46,67 segundos. Todos son consultas DNS por UDP desde `192.168.56.101` hacia `192.168.56.1`; no hay respuestas DNS en la captura.

## Descubrimiento humano, paso a paso

### 1. Mirar primero qué protocolos existen

En Wireshark, `Statistics → Protocol Hierarchy` muestra que los 66 paquetes son DNS. Por tanto, no hace falta perseguir HTTP, TCP ni archivos transferidos: el canal encubierto debe estar en algún campo de las consultas DNS.

Se aplica el filtro:

```text
dns
```

Al seleccionar un paquete y desplegar `Domain Name System → Queries → Name`, se ve el nombre que el equipo intentó resolver.

### 2. Comparar los nombres consultados

La mayoría parecen dominios normales usados como ruido:

```text
s3.amazonaws.com
clients4.google.com
api.spotify.com
www.wikipedia.org
```

Sin embargo, seis nombres comparten una estructura completamente distinta:

```text
c01-thwzdoonpx.exfil.umssctf.local
c03-lom5pwe2lf.exfil.umssctf.local
c04-nzpwgylnov.exfil.umssctf.local
c02-i5lonyzwym.exfil.umssctf.local
c05-tgyndegb6q.exfil.umssctf.local
c00-ovwxg43dor.exfil.umssctf.local
```

Son sospechosos porque:

- Comparten el sufijo artificial `exfil.umssctf.local`.
- El primer componente tiene un contador `c00`–`c05`.
- Después del guion hay bloques de diez caracteres con apariencia codificada.
- La palabra `exfil` confirma de forma bastante directa el tema indicado por el enunciado.

En Wireshark se pueden aislar con:

```text
dns.qry.name contains ".exfil.umssctf.local"
```

El canal de exfiltración es, por tanto, el **nombre de las consultas DNS**: los datos fueron introducidos en el subdominio izquierdo.

### 3. Elegir el orden usando evidencia, no la posición del paquete

El orden de captura no es el orden del mensaje. Los fragmentos llegaron como `c01`, `c03`, `c04`, `c02`, `c05`, `c00`. El prefijo de cada consulta proporciona el orden real:

| Índice | Frame | Tiempo relativo | Fragmento |
| --- | ---: | ---: | --- |
| `c00` | 45 | 33,631079 s | `ovwxg43dor` |
| `c01` | 3 | 1,689061 s | `thwzdoonpx` |
| `c02` | 24 | 19,076416 s | `i5lonyzwym` |
| `c03` | 6 | 4,929646 s | `lom5pwe2lf` |
| `c04` | 16 | 12,809262 s | `nzpwgylnov` |
| `c05` | 43 | 32,584355 s | `tgyndegb6q` |

Después de ordenar por `c00`–`c05` y retirar el contador, el guion y el dominio fijo se obtiene:

```text
ovwxg43dorthwzdoonpxi5lonyzwymlom5pwe2lfnzpwgylnovtgyndegb6q
```

### 4. Reconocer y normalizar Base32

La cadena utiliza solamente letras y los dígitos `2`–`7`, el alfabeto característico de Base32 estándar. En DNS no importa que las letras del dominio estén en mayúsculas o minúsculas, pero algunos decodificadores Base32 sí distinguen el caso. Por eso se convierte la cadena a mayúsculas sin alterar su contenido:

```text
OVWXG43DORTHWZDOONPXI5LONYZWYMLOM5PWE2LFNZPWGYLNOVTGYNDEGB6Q
```

La cadena extraída del DNS termina en `6q`: **no contiene ningún signo `=`**. Esto es normal en canales DNS, donde suele omitirse el padding porque `=` no forma parte de la sintaxis habitual de un nombre de host y, además, se puede deducir posteriormente.

Los signos `=` no contienen información del mensaje; solamente completan la representación para los decodificadores que exigen bloques Base32 de ocho caracteres. La evidencia tiene 60 caracteres y `60 mod 8 = 4`, por lo que se crea una **copia de trabajo** a la que se añaden cuatro signos:

```text
OVWXG43DORTHWZDOONPXI5LONYZWYMLOM5PWE2LFNZPWGYLNOVTGYNDEGB6Q====
```

La línea anterior es la entrada normalizada para el decodificador, no el nombre capturado. Al decodificarla aparece:

```text
umssctf{dns_tunn3l1ng_bien_camufl4d0}
```

### 5. Por qué dCode y CyberChef podían mostrar resultados distintos

El problema no era que existieran dos respuestas Base32 correctas. La evidencia extraída estaba **en minúsculas y sin padding**. dCode pudo inferir el padding que no venía en el DNS; con otros decodificadores es necesario añadirlo a la copia de trabajo.

La implementación actual de `From Base32` de CyberChef utiliza por defecto el alfabeto `A-Z2-7=` y tiene activada la opción `Remove non-alphabet chars`. Si se pega directamente la cadena DNS en minúsculas, las letras no pertenecen a ese alfabeto y pueden ser retiradas antes de decodificar. Existe además una propuesta todavía abierta para mejorar su manejo de entradas Base32 sin padding.

Referencias del proyecto CyberChef:

- [Código de From Base32](https://github.com/gchq/CyberChef/blob/master/src/core/operations/FromBase32.mjs)
- [Propuesta para mejorar el manejo de padding](https://github.com/gchq/CyberChef/pull/2082)

dCode fue tolerante con la representación recibida, pero no conviene depender de qué página sea más permisiva. El método reproducible es:

1. Conservar la cadena original como evidencia.
2. Confirmar que sus caracteres son compatibles con Base32.
3. Convertir solamente las letras a mayúsculas.
4. Si el decodificador lo exige, agregar el padding deducido solamente a una copia de trabajo, dejando claro que no estaba en la captura.
5. Verificar la salida con dos implementaciones y con la estructura esperada del resultado.

En este caso GNU `base32` y Python producen exactamente los mismos 37 bytes y la misma flag.

## Automatización con TShark, después del descubrimiento manual

Primero se extraen únicamente los nombres sospechosos, todavía con su frame para mantener trazabilidad:

```bash
tshark -r e79fbd68258cd5c3f1e667be066286e6.pcap \
  -Y 'dns.qry.name contains ".exfil.umssctf.local"' \
  -T fields -E separator='|' \
  -e frame.number -e frame.time_relative -e dns.qry.name
```

Resultado:

```text
3|1.689061000|c01-thwzdoonpx.exfil.umssctf.local
6|4.929646000|c03-lom5pwe2lf.exfil.umssctf.local
16|12.809262000|c04-nzpwgylnov.exfil.umssctf.local
24|19.076416000|c02-i5lonyzwym.exfil.umssctf.local
43|32.584355000|c05-tgyndegb6q.exfil.umssctf.local
45|33.631079000|c00-ovwxg43dor.exfil.umssctf.local
```

Una vez entendida la estructura, se puede automatizar la copia y el ordenamiento:

```bash
tshark -r e79fbd68258cd5c3f1e667be066286e6.pcap \
  -Y 'dns.qry.name contains ".exfil.umssctf.local"' \
  -T fields -e dns.qry.name \
  | sort -t- -k1,1V \
  | sed -E 's/^c[0-9]+-([^.]+)\..*/\1/' \
  | tr -d '\n' \
  | tr '[:lower:]' '[:upper:]'
```

Qué hace cada parte:

- TShark entrega un nombre DNS por línea.
- `sort -t- -k1,1V` usa `-` como separador y ordena el primer campo (`c00`, `c01`…) numéricamente.
- `sed` conserva solamente el fragmento situado después de `cNN-` y antes del primer punto.
- El primer `tr` une las líneas; el segundo normaliza las letras a mayúsculas.

Finalmente se agrega el padding justificado y se decodifica:

```bash
printf '%s\n' 'OVWXG43DORTHWZDOONPXI5LONYZWYMLOM5PWE2LFNZPWGYLNOVTGYNDEGB6Q====' \
  | base32 -d
```

Comprobación independiente con Python:

```bash
python3 -c 'import base64; print(base64.b32decode("OVWXG43DORTHWZDOONPXI5LONYZWYMLOM5PWE2LFNZPWGYLNOVTGYNDEGB6Q====").decode())'
```

## Cadena de resolución

`PCAP corto → inventario de protocolos → solo DNS → comparar Query Name → sufijo exfil anómalo → separar contador y fragmento → ordenar por c00–c05 → concatenar → normalizar Base32 → decodificar → flag`

## Flag

`umssctf{dns_tunn3l1ng_bien_camufl4d0}`

## Qué debo recordar para otro CTF

- DNS puede transportar información en los subdominios, no solamente en respuestas TXT.
- Un dominio largo, repetitivo o de alta entropía no basta por sí solo para afirmar exfiltración; aquí también existían un sufijo común, contadores y bloques uniformes.
- El orden temporal es el valor inicial que debe conservarse, pero un índice explícito puede definir el orden lógico real.
- Nunca retirar prefijos o sufijos antes de guardar también frame, tiempo y nombre original.
- Los nombres DNS no distinguen mayúsculas de minúsculas, pero un decodificador sí puede hacerlo.
- Si una herramienta web falla y otra funciona, primero se comparan alfabeto, mayúsculas, espacios y padding. No se elige una salida solamente porque parece legible.
- Una flag con formato correcto es una validación fuerte, pero también conviene confirmar la transformación con más de una implementación.

## Candidato para el arsenal forense

- Inventariar consultas y respuestas DNS, preservando frame, tiempo, origen, destino, tipo y nombre.
- Separar cada nombre en etiquetas y detectar sufijos repetidos con etiquetas variables.
- Marcar contadores como `c00`, `part3`, `03` o similares y proponer reconstrucciones tanto por tiempo como por índice.
- Puntuar etiquetas por longitud, alfabeto, entropía y proporción de caracteres Base32/Base64/hex, mostrando siempre por qué fueron marcadas.
- Reconstruir sin perder duplicados hasta comprobar si son retransmisiones o fragmentos repetidos intencionalmente.
- Normalizar una copia para cada decodificador, conservando intacto el valor capturado.
- Validar padding y probar decodificaciones justificadas mediante el motor común de capas.
