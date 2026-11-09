# Write-up: Timeline 1

## Reto

- Plataforma: picoCTF 2026

- Nombre: Timeline 1

- Categoría: Forensics

- Archivo/objetivo: `partition4.img.gz`

- Estado: `resuelto`


## Reconocimiento

```
gunzip partition4.img.gz

file partition4.img
```

La evidencia entregada es una imagen de un sistema de archivos. Por el nombre del reto y las pistas, el objetivo es reconstruir una timeline de actividad utilizando The Sleuth Kit.

Se genera un body file:

```
fls -r -m / partition4.img > timeline.body
```

Y posteriormente una timeline ordenada:

```
mactime -b timeline.body > timeline.txt
```

**Hallazgo:**

El reto indica prestar atención a timestamps recientes y a posibles acciones anti-forenses.

En la salida de `mactime`, las letras:

- `m` = Modified

- `a` = Accessed

- `c` = Changed

- `b` = Birth


representan los timestamps MACB de un archivo.

Un archivo que presenta las cuatro marcas simultáneamente puede resultar interesante, especialmente en un reto relacionado con manipulación de timestamps.

## Hipótesis

- La flag está almacenada dentro de un archivo del filesystem.

- El archivo fue ocultado mediante manipulación de timestamps.

- Los eventos `macb` recientes permiten localizar el archivo sospechoso.

- El contenido encontrado está codificado en Base64.


## Comandos y resultados

```
grep "macb" timeline.txt | tail -n 30
```

**Resultado útil:**

Entre los archivos recientes aparece:

```
49 macb r/rrw-r--r-- 0 0 32716 /etc/chat
```

El archivo `/etc/chat` resulta sospechoso porque:

- `/etc` normalmente contiene configuraciones del sistema.

- `chat` no parece un archivo estándar del sistema.

- Tiene solamente 49 bytes.

- Aparece entre los archivos con timestamps `macb`.


El inode del archivo es:

```
32716
```

Se extrae directamente desde la imagen:

```
icat partition4.img 32716
```

Resultado:

```
NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK
```

La cadena tiene características de Base64:

```
echo 'NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK' | base64 -d
```

Resultado:

```
573417h13r_7h4n_7h3_1457_58527bb222
```

Aplicando el formato requerido por picoCTF:

```
picoCTF{573417h13r_7h4n_7h3_1457_58527bb222}
```

## Cadena de resolución

`partition4.img.gz → gunzip → imagen de filesystem → fls → body file → mactime → MAC timeline → grep macb → /etc/chat → inode 32716 → icat → Base64 → flag`

## Qué debo recordar para otro CTF

- Indicador que me hizo reconocer la técnica:

    - El nombre `Timeline`.

    - Pistas relacionadas con timestamps.

    - Referencias a acciones anti-forenses o timestomping.

    - Las marcas `m`, `a`, `c`, `b` de Sleuth Kit.

- Herramienta que funcionó:

    - The Sleuth Kit:

        - `fls`

        - `mactime`

        - `icat`

- Comando mínimo:


```
fls -r -m / image.img > body
mactime -b body > timeline
grep "macb" timeline | tail
icat image.img <inode>
```

- Error/trampa:

    - Buscar directamente la cadena `picoCTF` puede no funcionar porque la flag puede estar codificada.

    - No asumir que timestamps normales implican actividad legítima.

    - Un atacante puede hacer timestomping para esconder un archivo en otra época.

    - El nombre y ubicación de un archivo pueden ser tan importantes como su timestamp.

    - `fls` permite identificar el inode; `icat` permite recuperar directamente el contenido usando ese inode.


## Flag

```
picoCTF{573417h13r_7h4n_7h3_1457_58527bb222}
```
