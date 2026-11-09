# Write-up: Fuga de Información Silenciosa

## Reto

- Plataforma: CITC

- Categoría: Forense

- Dificultad: Medium

- Fecha: 2026-09-01

- Estado: `resuelto`

- Tags: #forense #pcap #dns #dns-tunneling #exfiltration #base64 #wireshark #tshark


## Resumen

El reto proporciona una captura de tráfico `capture.pcapng` y pide recuperar una cadena de datos que fue robada de la red. El análisis revela consultas DNS anómalas hacia el dominio `exfil-data.net`, cuyos subdominios contienen fragmentos de información codificada. La clave del reto consiste en identificar el posible canal de exfiltración DNS, preservar el orden de las consultas, reconstruir los fragmentos y finalmente decodificar el contenido con Base64.

## Reconocimiento

La descripción menciona explícitamente actividad anómala en la red y una fuga de información, por lo que el primer objetivo es inspeccionar los protocolos presentes en la captura y buscar tráfico que pueda estar siendo utilizado como canal encubierto.\
Al revisar las consultas DNS aparecen dos dominios especialmente sospechosos:

```text
RE5TX1R1bm4zbDFuZ19FeGZpbHRyNHRpMG5fRDN0M2N0M.exfil-data.net
2Q.exfil-data.net
```

Ambas consultas utilizan el mismo dominio raíz:

```text
exfil-data.net
```

pero contienen datos diferentes dentro del subdominio.\
Una técnica común de DNS tunneling o DNS exfiltration consiste precisamente en introducir información dentro de los nombres consultados:

```text
<datos>.dominio-controlado.com
```

El servidor DNS controlado por el atacante puede registrar esas consultas y reconstruir los datos enviados.\
Además, el primer fragmento:

```text
RE5TX1R1bm4zbDFuZ19FeGZpbHRyNHRpMG5fRDN0M2N0M
```

utiliza únicamente caracteres compatibles con Base64, lo que da una pista adicional.

## Explotación / Procedimiento

Primero se puede filtrar el tráfico DNS en Wireshark:

```text
dns
```

Para reducir los resultados al dominio sospechoso:

```text
dns.qry.name contains "exfil-data.net"
```

También es posible extraer las consultas directamente con `tshark`:

```bash
tshark -r capture.pcapng \
  -Y 'dns.flags.response == 0 && dns.qry.name contains "exfil-data.net"' \
  -T fields \
  -e frame.number \
  -e dns.qry.name
```

Mostrar el número de frame es útil porque permite conservar el orden original de las consultas.\
Las consultas recuperadas son:

```text
RE5TX1R1bm4zbDFuZ19FeGZpbHRyNHRpMG5fRDN0M2N0M.exfil-data.net
2Q.exfil-data.net
```

Se elimina el dominio común `.exfil-data.net` y se conservan los fragmentos:

```text
RE5TX1R1bm4zbDFuZ19FeGZpbHRyNHRpMG5fRDN0M2N0M
2Q
```

Es importante no analizar únicamente el primer fragmento. Su longitud es 45 caracteres:

```text
45 mod 4 = 1
```

Una cadena Base64 completa no puede tener una longitud módulo 4 igual a 1. Esto es una señal de que la entrada probablemente está truncada o fragmentada.\
Al concatenar las consultas en el orden en que aparecieron:

```text
RE5TX1R1bm4zbDFuZ19FeGZpbHRyNHRpMG5fRDN0M2N0M2Q
```

la longitud pasa a ser compatible con una cadena Base64 a la que únicamente le falta padding:

```text
RE5TX1R1bm4zbDFuZ19FeGZpbHRyNHRpMG5fRDN0M2N0M2Q=
```

Se puede decodificar con:

```bash
echo 'RE5TX1R1bm4zbDFuZ19FeGZpbHRyNHRpMG5fRDN0M2N0M2Q=' | base64 -d
```

Resultado:

```text
DNS_Tunn3l1ng_Exfiltr4ti0n_D3t3ct3d
```

La plataforma utiliza para este reto el MD5 del contenido recuperado:

```bash
echo -n 'DNS_Tunn3l1ng_Exfiltr4ti0n_D3t3ct3d' | md5sum
```

Resultado:

```text
514ced0cecf02e505dd335043fd6e1e4
```

Por lo tanto, aplicando el formato de flag del reto:

```text
citc{514ced0cecf02e505dd335043fd6e1e4}
```

## Flag

`citc{514ced0cecf02e505dd335043fd6e1e4}`

## Notas adicionales

La principal lección del reto es que obtener texto legible después de una decodificación no significa necesariamente que los datos hayan sido reconstruidos completamente. Si se decodifica únicamente el primer fragmento mediante un Base64 tolerante se obtiene:

```text
DNS_Tunn3l1ng_Exfiltr4ti0n_D3t3ct
```

Aunque parece un resultado válido, está incompleto. El segundo query DNS contiene `2Q`, que completa el mensaje original hasta:

```text
DNS_Tunn3l1ng_Exfiltr4ti0n_D3t3ct3d
```

En exfiltración mediante DNS es importante conservar el orden temporal de los paquetes. Usar comandos como `sort | uniq` sin analizar primero la estructura puede destruir el orden de los fragmentos y producir una reconstrucción incorrecta.\
Patrón reusable:

```text
PCAP
→ identificar tráfico DNS sospechoso
→ detectar subdominios con datos codificados o alta entropía
→ agrupar consultas por dominio
→ preservar orden temporal
→ eliminar el dominio común
→ concatenar fragmentos
→ identificar encoding
→ decodificar
→ validar estructuralmente el resultado
```

Este reto también muestra una diferencia importante entre análisis forense y criptografía: la parte forense consiste en identificar y reconstruir correctamente los fragmentos enviados por DNS; Base64 es únicamente la capa de codificación aplicada después de recuperar los datos.

## Qué debo recordar para otro CTF

- Agrupar consultas por dominio y conservar orden temporal y dirección.
- No descartar fragmentos cortos: pueden completar el padding o los últimos bytes.
- Decodificar solamente después de reconstruir el canal.

## Aporte para el arsenal

- Detectar etiquetas DNS con alta entropía o alfabetos de codificación.
- Conservar dominio, paquete, tiempo y fragmento para evitar reconstrucciones falsas.
