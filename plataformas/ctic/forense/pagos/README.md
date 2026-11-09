# Write-up: Pagos

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Forense
- Puntos: 500
- Archivo: `Pagos.pcapng`
- Estado: `resuelto y validado en la plataforma`

> Se sospecha que un empleado está robando información y vendiéndola a compradores de China y Rusia. Debemos conseguir información que permita identificar o capturar al responsable.

## Resumen

La captura contiene casi exclusivamente SMB2 entre `192.168.27.100` y `192.168.27.206`. La mayor parte del tráfico corresponde a enumeración de directorios y constituye ruido. Desde **Archivo → Exportar objetos → SMB**, Wireshark reconstruye dos archivos relevantes: `transfer.info` y `wallet.enc`.

`transfer.info` contiene Base64. Al decodificarlo aparece un largo Lorem Ipsum con una secuencia Morse insertada. El Morse indica que se debe usar el nombre de usuario para extraer la dirección de una billetera. La autenticación NTLM de las sesiones SMB revela el usuario `sniper`.

`wallet.enc` comienza con `Salted__`, cabecera característica de `openssl enc` con salt. Se descifra con AES-256-CBC, derivación tradicional de OpenSSL y la contraseña `sniper`. El resultado todavía está codificado en Base64; su decodificación revela el mensaje final y el contenido de la flag.

## Reconocimiento inicial

```bash
file Pagos.pcapng
capinfos Pagos.pcapng
tshark -r Pagos.pcapng -q -z io,phs
```

Resultados relevantes:

- PCAPNG válido, capturado con Wireshark 4.0.7 sobre Windows.
- 8.384 paquetes y 2.033.636 bytes de archivo.
- Duración de aproximadamente 35,87 segundos.
- Dos participantes principales: `192.168.27.100` y `192.168.27.206`.
- Casi todo el tráfico es TCP/445 con NBSS y SMB2.
- Existen tres conexiones SMB2 principales.
- Hay aproximadamente 700 operaciones `Create` y 1.300 `Find`, pero solamente ocho lecturas SMB2.

La diferencia entre miles de paquetes y muy pocas lecturas reales permite reducir rápidamente el ruido.

## Resolución humana en Wireshark

### 1. Identificar SMB2

En:

```text
Estadísticas → Jerarquía de protocolos
```

se observa que casi toda la captura pertenece a SMB2. En:

```text
Estadísticas → Conversaciones → TCP
```

aparecen tres conversaciones grandes entre:

```text
192.168.27.100 → 192.168.27.206:445
```

El puerto `445` y el disector SMB2 indican acceso remoto a archivos y directorios.

### 2. Usar la función incorporada para exportar objetos

La ruta utilizada durante la resolución fue:

```text
Archivo → Exportar objetos → SMB
```

La ventana muestra cinco objetos exportables:

- tres objetos `srvsvc`, pertenecientes a una tubería/servicio RPC;
- `\Downloads\transfer.info`;
- `\Downloads\wallet.enc`.

Al seleccionar un objeto, Wireshark salta al paquete que contiene o completa su reconstrucción. Así se llegó humanamente al `Read Response` relevante; no fue necesario recorrer cronológicamente los 8.384 paquetes.

Los nombres exportados en Linux aparecen como:

```text
%5cDownloads%5ctransfer.info
%5cDownloads%5cwallet.enc
```

`%5c` es la codificación URL del carácter `\`. No forma parte del contenido ni indica cifrado.

### 3. Método equivalente mediante filtros

Las lecturas SMB2 se aíslan con:

```text
smb2.cmd == 8 && smb2.flags.response == 1
```

En esta captura, una reducción todavía más directa es:

```text
smb2 && data.data
```

Solo quedan dos respuestas con datos genéricos reconstruidos:

| Frame | Stream | Tamaño de datos | Archivo asociado |
| ---: | ---: | ---: | --- |
| 5306 | 3 | 8.393 bytes | `Downloads\transfer.info` |
| 8109 | 5 | 144 bytes | `Downloads\wallet.enc` |

`data.data` no es una regla universal para todo SMB: funciona aquí porque Wireshark entrega el contenido de esas dos respuestas como datos genéricos. **Export Objects** es el método principal para recuperar los archivos completos.

### 4. Decodificar `transfer.info`

El archivo es texto ASCII formado por caracteres Base64 y saltos de línea. Se decodifica así:

```bash
base64 -d '%5cDownloads%5ctransfer.info' > transfer.txt
```

El resultado es una línea extensa de Lorem Ipsum. Dentro aparece una secuencia formada por puntos, guiones y separadores:

```text
-. --- - . ---... ..- ... . ....... -.-- --- ..- .-. ....... ..- ... . .-. -. .- -- .
....... - --- ....... . -..- - .-. .- -.-. - ....... - .... .
....... .-- .- .-.. .-.. . - ....... .- -.. -.. .-. . ... ...
```

Los grupos de siete puntos representan espacios de palabras en esta construcción. La decodificación Morse produce:

```text
NOTE: USE YOUR USERNAME TO EXTRACT THE WALLET ADDRESS
```

El Lorem Ipsum es cobertura y la frase Morse es una instrucción intermedia, no la flag.

### 5. Recuperar el nombre de usuario SMB

Se filtran los campos de autenticación NTLM:

```text
ntlmssp.auth.username
```

Aparecen tres autenticaciones, una por cada sesión principal:

| Frame | Stream | Dominio | Usuario |
| ---: | ---: | --- | --- |
| 20 | 1 | `WORKGROUP` | `sniper` |
| 2796 | 3 | `WORKGROUP` | `sniper` |
| 5603 | 5 | `WORKGROUP` | `sniper` |

La repetición en sesiones distintas confirma el valor:

```text
sniper
```

No se refiere al usuario local del participante ni al de la plataforma CTF: es la identidad autenticada capturada en NTLM.

### 6. Identificar `wallet.enc`

La extensión `.enc` es genérica. Primero se debe identificar el contenido real:

```bash
file '%5cDownloads%5cwallet.enc'
xxd -l 32 '%5cDownloads%5cwallet.enc'
strings -a -n 6 '%5cDownloads%5cwallet.enc'
```

Resultados:

```text
openssl enc'd data with salted password

00000000: 5361 6c74 6564 5f5f ae7d 34f8 26c9 73b9  Salted__.}4.&.s.
00000010: e031 66ce e134 7d8c 2426 9a09 5234 55be  .1f..4}.$&..R4U.
```

La estructura inicial es:

```text
bytes 0–7   → "Salted__"
bytes 8–15  → salt: ae 7d 34 f8 26 c9 73 b9
bytes 16…   → ciphertext
```

El salt no es la contraseña ni el IV. OpenSSL combina el password con ese valor para derivar la clave y el IV. Su función es evitar que dos cifrados del mismo contenido con la misma contraseña produzcan exactamente los mismos bytes.

`strings` no ayuda después de la cabecera porque un ciphertext correcto debe parecer bytes aleatorios.

### 7. Descifrar con la contraseña encontrada

La combinación válida para este archivo es:

```bash
openssl enc -d -aes-256-cbc \
  -in '%5cDownloads%5cwallet.enc' \
  -out wallet.b64 \
  -pass pass:sniper
```

Significado:

- `openssl enc`: operación de cifrado simétrico sobre un archivo;
- `-d`: descifrar;
- `-aes-256-cbc`: algoritmo y modo empleados;
- `-in`: archivo cifrado de entrada;
- `-out`: resultado descifrado;
- `-pass pass:sniper`: contraseña obtenida de la autenticación SMB.

No se utiliza `-a` porque `wallet.enc` ya está en binario. Tampoco se utiliza `-pbkdf2`: el archivo fue creado con la derivación tradicional de OpenSSL. La advertencia sobre una derivación obsoleta describe cómo fue creado el reto, no un fallo del descifrado.

### 8. Reconocer la última capa

`wallet.b64` ya no es ciphertext. Es texto ASCII con apariencia Base64:

```text
RG8gdGhlIHRyYW5zZmVyIHRvIHRoaXMgQml0Y29pbiB3YWxsZXQK...
```

Se decodifica:

```bash
base64 -d wallet.b64
```

Resultado:

```text
Do the transfer to this Bitcoin wallet

flagHunters{827b86172465ff938f962b010d7b544f}
```

El texto llama «Bitcoin wallet» al identificador, aunque el contenido de 32 dígitos hexadecimales no tiene la estructura habitual de una dirección Bitcoin. Debe tratarse como parte de la narrativa/flag, no como una dirección válida demostrada.

## Flag validada

La plataforma CIDSI acepta el contenido con su envoltorio actual:

```text
cidsi{827b86172465ff938f962b010d7b544f}
```

El envoltorio `flagHunters{...}` pertenece al texto original del reto. El valor interno se conserva y el prefijo se adapta al formato exigido por la plataforma.

## ¿Cómo se supo que era AES-256-CBC?

La cabecera `Salted__` identifica el contenedor de `openssl enc`, pero **no almacena el algoritmo**, el modo ni la función de derivación. En el PCAP tampoco aparecen cadenas como `AES`, `CBC`, `OpenSSL`, `PBKDF2` o `cipher`.

Por tanto, AES-256-CBC no se deduce de manera única a partir de la evidencia. Es una combinación común en ejemplos y retos CTF, y una prueba controlada resultó correcta. Esta parte está insuficientemente indicada por el reto.

La validación no consiste solamente en que OpenSSL finalice sin error. El resultado forma Base64 válido y, después de decodificarlo, produce lenguaje natural y un marcador estructurado. Esa cadena de coherencia confirma la combinación.

## Cadena de resolución

`PCAPNG → Protocol Hierarchy: SMB2 → Export Objects: transfer.info + wallet.enc → Base64 → Lorem Ipsum → detectar Morse → instrucción sobre username → NTLMSSP username = sniper → file/xxd sobre wallet.enc → Salted__ = openssl enc → AES-256-CBC + password sniper → Base64 → mensaje y flag → adaptar envoltorio CIDSI`

## Qué debo recordar para otro CTF

- **Export Objects** permite pasar de miles de paquetes a unos pocos archivos reconstruidos y conserva un vínculo con el paquete correspondiente.
- En SMB2, `Session Setup` identifica la cuenta, `Create` abre un archivo, `Read` pide bloques y `Read Response` entrega sus bytes.
- Un archivo `.enc` no define un formato ni un algoritmo. Se comienza con `file`, magic bytes, tamaño y `xxd`.
- `Salted__` identifica el formato de `openssl enc` con salt, no AES específicamente.
- Base64 es una codificación reversible sin clave; AES es cifrado y exige clave/contraseña, algoritmo, modo y parámetros compatibles.
- El resultado de descifrar puede contener otra codificación. Siempre se vuelve a ejecutar triage sobre cada nueva capa.
- El ciphertext normalmente no produce strings útiles; una salida aparentemente aleatoria es esperable.
- Un `bad decrypt` puede significar password, cifrado, digest o derivación incorrectos; no demuestra por sí solo cuál falló.
- El reto debe proporcionar o permitir deducir el algoritmo. Cuando no lo hace, se documenta claramente qué parte fue una prueba razonable.
- La validez final se apoya en varias señales: descifrado sin error, Base64 válido, lenguaje natural, marcador de flag y aceptación de la plataforma.

## Candidato para el arsenal forense

- Inventariar operaciones y objetos SMB/SMB2.
- Extraer username, dominio, workstation, share, filename, File ID, offset y longitud.
- Separar enumeración de directorios de lecturas y escrituras con datos.
- Exportar objetos SMB y conservar frame, stream y ruta original.
- Detectar Base64 y Morse dentro de objetos de texto sin aceptar automáticamente cualquier coincidencia.
- Reconocer `Salted__` y explicar que el algoritmo no está almacenado.
- Probar una lista pequeña y explícita de combinaciones OpenSSL comunes, registrando cipher, KDF y digest, pero sin afirmar que se dedujeron del archivo.
- Volver a ejecutar triage sobre cada capa obtenida: tipo, magic bytes, strings y formatos de flag.
