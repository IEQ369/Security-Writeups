# El secreto de Fondo de Bikini

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Esteganografía. Puntos: 160. Autor: Nath Barros.
- Archivo: EL_SECRETO_DE_FONDO_DE_BIKINI.zip.
- Estado: **resuelto localmente; aceptación en la plataforma no registrada**.

> «No todo lo que ves está realmente ahí». La respuesta está en la imagen, pero no en la forma en que parece. No confíes en una sola observación.

## Resumen

Cada canal RGB contiene una pista en su bit menos significativo. El azul conduce al rojo; el rojo conduce al verde. El mensaje del verde está invertido. Cada extracción empieza con una longitud de cuatro bytes.

## Reconocimiento

Desde la carpeta donde esté el ZIP:

```bash
unzip EL_SECRETO_DE_FONDO_DE_BIKINI.zip
cd EL_SECRETO_DE_FONDO_DE_BIKINI_CORREGIDO
file FONDO_BIKINI.png
exiftool FONDO_BIKINI.png
zsteg FONDO_BIKINI.png
```

La imagen es un PNG: sus píxeles permiten examinar LSB sin la pérdida propia de recomprimir un JPEG. Las pistas sobre observación justifican separar canales, pero el resultado de la extracción es la evidencia decisiva.

## Hipótesis

- [x] Hay texto en los LSB de canales separados.
- [x] Los textos forman una cadena de instrucciones.
- [ ] La plataforma aceptó un prefijo determinado: no consta.

## Resolución paso a paso

### 1. Extraer el azul y seguir su pista

```bash
zsteg -E b1,b,lsb,xy FONDO_BIKINI.png | xxd -l 80
```

`b1` extrae un bit por componente; `b` elige azul; `lsb` selecciona el menos significativo; `xy` recorre filas. Aparece una cabecera `00 00 00 40`, seguida de:

```text
PRIMERA PISTA: SEPARA LOS CANALES RGB. EL ROJO GUARDA EL CAMINO.
```

`0x40 = 64`: hay exactamente 64 bytes de texto.

### 2. Extraer el rojo

```bash
zsteg -E b1,r,lsb,xy FONDO_BIKINI.png | xxd -l 80
```

Cabecera `00 00 00 3d` (61), seguida de:

```text
SEGUNDA PISTA: EL VERDE TIENE EL MENSAJE, PERO ESTA AL REVES.
```

### 3. Extraer exactamente el mensaje del verde

```bash
zsteg -E b1,g,lsb,xy FONDO_BIKINI.png | xxd -l 48
```

Los primeros cuatro bytes son `00 00 00 24`: **36** en decimal. Para quitar la cabecera y leer solo esos 36 bytes:

```bash
zsteg -E b1,g,lsb,xy FONDO_BIKINI.png | dd bs=1 skip=4 count=36 status=none
```

Resultado:

```text
inikiBeDodnoFeDoterceSlE_AJNOPSE_BOB
```

### 4. Invertir el texto

```bash
printf '%s\n' 'inikiBeDodnoFeDoterceSlE_AJNOPSE_BOB' | rev
```

Resultado: `BOB_ESPONJA_ElSecretoDeFondoDeBikini`.

**Corrección de la explicación de competencia:** `strings` también mostraba un `$` porque `0x24` es un carácter imprimible. Ese byte pertenecía a la cabecera de longitud; **no era un terminador del mensaje**. Leer la longitud elimina la ambigüedad.

## Cadena de resolución

`ZIP → PNG → azul LSB → rojo LSB → verde LSB → longitud de 4 bytes → invertir texto`

## Flag / resultado

```text
BOB_ESPONJA_ElSecretoDeFondoDeBikini
```

Contenido recuperado y comprobado de nuevo el 3 de octubre. No añadir un prefijo ni un MD5 sin una regla confirmada del reto.

## Qué debo recordar para otro CTF

`strings` sirve para orientarse, pero puede unir la cabecera con el mensaje. Un carácter visible como `$` no es automáticamente parte de la respuesta.

## Aporte para el arsenal

Extraer RGB por separado, detectar cabeceras de longitud y mostrar tanto bytes crudos como texto. Mantener las pistas y los resultados como hallazgos distintos.
