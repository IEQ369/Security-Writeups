# Contra reloj

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Forense.
- Archivo: `incident.log` (200 600 líneas). **El original ya no está en Descargas** y no se localizó en la papelera; ver «Qué falta».
- Estado: **aceptado**. El usuario informó que la plataforma aceptó solo el usuario, sin fecha ni hora.
- Registro breve anterior: SOLVE.

> Un ransomware paralizó la empresa. Hay que reconstruir la línea de tiempo y encontrar al «Paciente Cero»: el primer usuario realmente comprometido y cuándo. Formato anunciado: `Usuario_fecha_hora`. Aviso del enunciado: «no te dejes engañar por los eventos posteriores».

## Resumen

El log mezcla unos 200 000 eventos normales con una campaña de correo: el mismo adjunto `Invoice_0904.zip` llegó a **seis** usuarios. Tres intentos fracasaron (antivirus o cancelación) y tres terminaron en PowerShell. De esos, el primero en el tiempo, y el único con conexión a un servidor externo y cifrado masivo inmediatos, es `juan.perez` en `NB-030`.

## Qué recibimos

Un archivo de texto plano con una línea por evento, en este formato:

```text
2026-09-04T17:17:38Z | PROCESS | host=NB-030 | user=juan.perez | pid=4019 | parent=wscript.exe | image=powershell.exe | cmd="..."
```

Los campos van separados por ` | `. El segundo campo es el **tipo de evento** (`DNS`, `FILE`, `FIREWALL`, `HTTP`, `PROCESS`, `SERVICE`, `AUTH`, `MAIL`, `AV`, `NET`). Detalle importante: **las líneas no están en orden cronológico**; la primera es de las 01:06 del día 5 y la segunda de las 19:51 del día 4. Por eso, para ver una secuencia hay que ordenar (`sort`) antes de leer.

## Reconocimiento

```bash
cd ./Descargas
wc -l incident.log
head -20 incident.log
```

**Hallazgo:** 200 600 líneas, mezcla de tipos y desordenadas. Leerlo entero no es viable; hay que buscar el evento que *no* es ruido.

## Resolución paso a paso

### 1. Buscar procesos que no sean rutina

**Observación:** casi todos los procesos nacen de `explorer.exe` (el usuario abre Chrome, Outlook…). Un ataque típico de adjunto malicioso rompe ese patrón: un script lanza una consola.

```bash
grep -E '\| PROCESS' incident.log | grep -oE 'image=[^ ]+' | sort | uniq -c | sort -rn
```

**Salida que importa** (las dos últimas líneas; el resto son miles de repeticiones de programas normales):

```text
      6 image=wscript.exe
      3 image=powershell.exe
```

Solo 6 `wscript.exe` y 3 `powershell.exe` en todo el log. Es la lista corta de candidatos. Los tres PowerShell tienen `parent=wscript.exe`: un script de Windows abriendo una consola, algo anómalo.

### 2. Listar esos eventos ordenados

```bash
grep -E 'wscript|powershell' incident.log | grep -v 'path=/assets' | sort
```

`sort` ordena por la marca de tiempo porque está al inicio de cada línea. Resultado: seis `wscript.exe` que ejecutan `Invoice_0904.lnk` (un acceso directo dentro del ZIP), y tres `powershell.exe`:

| Hora (UTC) | Usuario | Host | Proceso |
|---|---|---|---|
| 13:31:13 | andres_sanchez | WS-061 | wscript (sin PowerShell) |
| **17:17:33** | **juan.perez** | **NB-030** | wscript → **17:17:38 powershell `-nop -w hidden -enc`** |
| 17:47:13 | maria.paredes | PC-070 | wscript → 17:47:17 powershell `-version 2.0` |
| 18:08:29 | carmen_ramirez | LAP-045 | wscript (sin PowerShell) |
| 18:36:01 | sofia_torres | SRV-064 | wscript → 18:36:05 powershell `-version 2.0` |
| 20:05:07 | fnavarro | SRV-066 | wscript (sin PowerShell) |

**Por qué importa:** `-nop -w hidden -enc` significa «sin perfil, ventana oculta, comando codificado en Base64». Es la firma de un cargador malicioso; los otros dos usan `-version 2.0`, una técnica de degradación, y aparecen más tarde.

### 3. Seguir el adjunto hasta su origen

**Observación:** el archivo `Invoice_0904.lnk` viene de `Invoice_0904.zip`. Cada evento de proceso tiene un `file_ref=FILE-xxxxxx` que lo enlaza con un evento `FILE` y este con un `MAIL`.

```bash
grep -E 'Invoice_0904|FILE-(icct5h|9si5ga|obrcl4|g1x3j1|s8ke2w|st5j28)' incident.log | sort
```

Cadena para cada usuario: `MAIL` (llega el ZIP) → `FILE OPEN` (lo abre) → `PROCESS wscript` (ejecuta el `.lnk`). Los seis correos tienen asunto «Invoice September 2026» y remitentes de dominios distintos (`billing-services.net`, `account-verification.net`, `document-share.net`, `cloud-document.info`): una campaña de phishing.

### 4. Descartar los intentos que fallaron

**Observación:** que haya `wscript` no prueba infección. Hay que mirar qué pasó justo después.

```bash
grep -E '\| AV \||USER_CANCELLED' incident.log | sort
```

```text
2026-09-04T13:31:16Z | AV | host=WS-061 | user=andres_sanchez | detection=Suspicious.Script.Lnk | action=BLOCKED | source_pid=7204
2026-09-04T18:08:35Z | FILE | host=LAP-045 | user=carmen_ramirez | action=CLOSE | name=Invoice_0904.lnk | result=USER_CANCELLED
2026-09-04T20:05:10Z | AV | host=SRV-066 | user=fnavarro | detection=Suspicious.Script.Lnk | action=BLOCKED | source_pid=4972
```

- `andres_sanchez` es el más temprano (13:31) y el señuelo principal: parece el primero, pero el antivirus lo **bloqueó** tres segundos después y no hay PowerShell.
- `carmen_ramirez` canceló la ejecución.
- `fnavarro` también fue bloqueado.

Quedan `juan.perez`, `maria.paredes` y `sofia_torres`.

### 5. Elegir entre los tres y confirmar con consecuencias

Los tres tienen PowerShell, pero el primero cronológicamente es `juan.perez` (17:17). Se confirma con los eventos de su host:

```bash
grep -E 'NB-030|juan.perez' incident.log | sort | awk '$1>="2026-09-04T17:17" && $1<="2026-09-04T17:40"' | grep -v 'path=/assets'
```

El filtro `awk` compara el primer campo (la marca de tiempo, que se puede comparar como texto) para acotar la ventana. Las líneas útiles entre mucho ruido de otros «juan.perez1», «juan_perez»:

```text
17:17:00Z MAIL    to=juan.perez  attachment=Invoice_0904.zip
17:17:21Z FILE    OPEN Invoice_0904.zip
17:17:33Z PROCESS wscript.exe  cmd="Invoice_0904.lnk"
17:17:38Z PROCESS powershell.exe -nop -w hidden -enc...   (pid=4019)
17:17:45Z NET     pid=4019 dst=139.177.78.180:443 proto=TLS bytes_out=4297
17:18:01Z FILE    MASS_MODIFY pattern=*.locked count=169 source_pid=4019
17:18:18Z NET     SMB_SCAN source_pid=4019 targets=21
```

**Cómo interpretarlo:** todos los eventos finales tienen `pid=4019`, el mismo del PowerShell. Eso une el proceso con una conexión TLS saliente (servidor de control), el cifrado de 169 archivos `*.locked` 23 segundos después de iniciar PowerShell y un escaneo SMB de 21 destinos para propagarse. Es la secuencia completa de un ransomware. Los otros dos usuarios ejecutan PowerShell 30 y 79 minutos después; ya había propagación.

**Trampa de nombres:** abundan usuarios casi iguales (`juan.perez1`, `juan_perez`, `juan.p`, `juanperez`). Son ruido generado, no cuentas del incidente. El usuario del evento clave es exactamente `juan.perez`, en el host `NB-030`.

## Cadena de resolución

`log desordenado → tipos raros de PROCESS (wscript/powershell) → mismo adjunto a 6 usuarios → AV/cancelación descarta 3 → PowerShell más temprano → pid 4019 con C2 y cifrado → juan.perez`

## Respuesta

La candidata inicial fue `juan.perez_2026-09-04_17:17:38`, siguiendo el formato del enunciado. Según el usuario, la plataforma **aceptó solo el usuario**:

```text
juan.perez
```

El mensaje del usuario lo escribe `juanp.perez` (aparente error de tecleo); los registros y el índice de la competencia usan `juan.perez`, que es el usuario real del log. Hora del evento clave, solo como contexto: 2026-09-04 17:17:38Z.

## Corrección del registro anterior

El SOLVE llama «candidata» a esa respuesta y propone horas alternativas (17:17:33 y 17:17:45). Esas alternativas ya no hacen falta: la aceptación cubrió el usuario.

## Qué falta / límites

- `incident.log` no se conserva. No se pudo repetir ningún comando de esta nota. Las salidas son las del historial; las líneas citadas están en lineas-clave-incident.log (27 líneas, **extractos recuperados del historial, no una copia del log**).
- La carga `-enc` nunca se decodificó (el log la muestra truncada).

## Qué debo recordar para otro CTF

- Un log enorme se reduce contando valores raros: `uniq -c | sort -rn` deja arriba lo normal y abajo lo anómalo.
- Hay que ordenar antes de interpretar: el log estaba desordenado.
- El primer evento sospechoso no es necesariamente el primero real: comprobar si fue bloqueado o cancelado.
- El `pid` o `file_id` encadenan eventos de distinto tipo.

## Aporte para el arsenal

- Automatizable: agrupar por `file_ref`/`pid` y mostrar la cadena MAIL→FILE→PROCESS→AV/NET de cada usuario.
- Requiere criterio humano: decidir qué «bloqueado» o «cancelado» descarta a un candidato.
- Falso positivo: usuarios con nombre casi igual al candidato.
