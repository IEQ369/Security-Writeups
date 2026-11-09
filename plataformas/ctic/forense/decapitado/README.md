# Write-up: Decapitado

## Reto

- Plataforma: CTIC

- Categoría: Forense

- Dificultad: Medium

- Fecha: 2026-09-01

- Estado: `resuelto`

- Tags: #forense #zip #magic-bytes #file-carving #zipcrypto #john #split


## Resumen

El reto entregaba un archivo llamado `Decapitado-chall.zip` que, pese a su extensión, no era reconocido inicialmente como un ZIP válido. El objetivo fue identificar la corrupción de la cabecera, reparar el archivo, recuperar su contraseña, reconstruir varios fragmentos internos y analizar el nuevo artefacto resultante. La reconstrucción produjo un segundo ZIP protegido con contraseña que finalmente contenía `flag.txt`.

## Reconocimiento

El primer paso fue inspeccionar los bytes iniciales del archivo:

```bash
xxd -l 16 Decapitado-chall.zip
```

La salida comenzaba con:

```text
00000000: 0304 0a00 0900 0000 d8ba 5154 8b31 e1a6
```

Un archivo ZIP normalmente comienza con la firma:

```text
50 4b 03 04
 P  K
```

El archivo comenzaba directamente por `03 04`, por lo que faltaban exactamente los dos primeros bytes `50 4b`, correspondientes a los caracteres ASCII `PK`. Esto encajaba además con el título del reto, `Decapitado`: al archivo le habían quitado literalmente la "cabeza".\
Se reparó anteponiendo esos dos bytes:

```bash
(printf 'PK'; cat Decapitado-chall.zip) > Decapitado-fixed.zip
```

Después se verificó:

```bash
file Decapitado-fixed.zip
unzip -t Decapitado-fixed.zip
zipinfo Decapitado-fixed.zip
```

`file` pasó a reconocerlo como:

```text
Zip archive data
```

El ZIP contenía cinco archivos:

```text
xaa
xab
xac
xad
xae
```

Los cinco estaban protegidos con contraseña. Los nombres `xaa`, `xab`, `xac`, etc. coinciden con el esquema de nombres que genera habitualmente la herramienta Unix `split`, por lo que existía una fuerte posibilidad de que fueran fragmentos consecutivos de otro archivo.

## Explotación / Procedimiento

Primero se extrajo el hash del ZIP para John the Ripper:

```bash
zip2john Decapitado-fixed.zip > decapitado.hash
```

El hash resultante comenzaba por `$pkzip$`, indicando que John reconocía correctamente el cifrado PKZIP/ZipCrypto.\
Se realizó un ataque de diccionario con `rockyou.txt`:

```bash
john --wordlist=rockyou.txt decapitado.hash
```

John encontró la contraseña:

```text
scoop
```

Con ella se extrajeron los cinco fragmentos:

```bash
mkdir decapitado_out
unzip -P scoop Decapitado-fixed.zip -d decapitado_out
cd decapitado_out
```

Debido al patrón `xaa`, `xab`, `xac`, `xad`, `xae`, se reconstruyó el archivo original concatenándolos exactamente en ese orden:

```bash
cat xaa xab xac xad xae > reconstruido.bin
```

Se volvió a realizar reconocimiento sobre el nuevo artefacto:

```bash
file reconstruido.bin
```

`file` reveló que el contenido reconstruido era otro archivo ZIP. La extensión `.bin` no tenía importancia para la identificación real del formato, por lo que se renombró únicamente por comodidad:

```bash
mv reconstruido.bin reconstruido.zip
```

El segundo ZIP también estaba protegido con contraseña. Se extrajo nuevamente su hash:

```bash
zip2john reconstruido.zip > reconstruido.hash
```

Se lanzó John con el mismo diccionario:

```bash
john --wordlist=rockyou.txt reconstruido.hash
```

La contraseña encontrada fue:

```text
myspace6
```

John indicó además que el archivo protegido dentro del ZIP era `flag.txt`.\
Se extrajo:

```bash
unzip reconstruido.zip
```

Introduciendo la contraseña:

```text
myspace6
```

Finalmente:

```bash
cat flag.txt
```

produjo:

```text
Morteruelo2022{ecc750bfe2cee389ac6ae4111a6b13f1}
```

El valor hexadecimal interno recuperado fue:

```text
ecc750bfe2cee389ac6ae4111a6b13f1
```

Para el formato utilizado por CITC/CTIC en este conjunto de retos, la flag comprobada es:

```text
cidsi{ecc750bfe2cee389ac6ae4111a6b13f1}
```

## Flag

`cidsi{ecc750bfe2cee389ac6ae4111a6b13f1}`

## Notas adicionales

El reto deja varios patrones forenses reutilizables. Una extensión de archivo no garantiza que su estructura sea válida, por lo que ante un archivo sospechoso conviene usar `file` y examinar los primeros bytes con `xxd`. Las magic bytes permiten detectar cabeceras dañadas o deliberadamente modificadas. En este caso, la ausencia de `50 4b` era una corrupción mínima y el nombre `Decapitado` funcionaba como pista directa.\
Los nombres `xaa`, `xab`, `xac`, etc. son además una señal importante porque son el esquema utilizado por defecto por `split`. Cuando aparecen varios archivos con ese patrón y tamaños similares, concatenarlos en orden con `cat` es una prueba barata y razonable.\
Otro aprendizaje importante es que un archivo recuperado no tiene por qué contener directamente la flag. Después de cada extracción, decodificación o reconstrucción conviene volver a iniciar el reconocimiento:

```text
file → magic bytes → strings/metadatos → estructura → siguiente acción
```

En este caso, la primera reconstrucción produjo otro ZIP completo. También se comprobó que cambiar la extensión de `.bin` a `.zip` no convirtió el archivo en ZIP; simplemente facilitó su manejo. El formato ya estaba determinado por sus bytes internos.\
Finalmente, RockYou funcionó para las contraseñas `scoop` y `myspace6`, pero esto no debe convertirse en una regla de aplicar diccionarios indiscriminadamente. Primero se identificó correctamente que los contenedores eran ZIP cifrados y después se utilizó `zip2john` y John sobre el artefacto correspondiente.

## Qué debo recordar para otro CTF

- Comparar extensión, magic bytes y estructura antes de cambiar nombres.
- Los fragmentos `xaa`, `xab`, `xac` sugieren la salida predeterminada de `split`.
- Cada archivo reconstruido vuelve a pasar por reconocimiento.

## Aporte para el arsenal

- Detectar cabeceras truncadas y proponer una reparación sobre una copia.
- Reconocer familias de fragmentos y validar el archivo concatenado.
