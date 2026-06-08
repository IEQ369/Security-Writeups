# Los 5 secretos de Juan

- Plataforma: CTF CGII/AGETIC (ctf.cgii.gob.bo)
- Categoría: Forense
- Puntos: 190
- Dificultad: Medium
- Tags: #forense #zip #steghide #john

## Resumen
Reto de forense con un ZIP protegido. Contiene cinco archivos: cuatro son distractores y en el quinto (`secret4.jpeg`) está la flag real, oculta con esteganografía.

## Material
- `forensic_sh4m4n.zip`

## Descripción del reto

"A lo largo de los años, Juan ha desarrollado una obsesión con las combinaciones de 5 elementos. Cada vez que quiere ocultar algo importante, siempre recurre a su número favorito: el 5. En este desafío, tendrás que desentrañar los secretos que ha dejado escondidos en los archivos. Pero cuidado, no todo lo que parece simple lo es. Necesitarás utilizar varias herramientas para descubrir la verdad, ya que a veces lo que parece obvio puede ser una distracción. Recuerda que Juan es muy astuto y adora ocultar sus pistas entre falsas verdades. ¿Podrás descifrar los 5 secretos que Juan ha ocultado? Autor: 3LH3ch1z3r0 La bandera esta con el siguiente formato CITC2024{bandera}, deberás llevarla al formato de la competencia citc{bandera_md5}"

## Metodología / Pasos seguidos

### 1. Revisión inicial del archivo

El archivo `forensic_sh4m4n.zip` es un ZIP protegido con contraseña.
Contenido del ZIP con `unzip -l forensic_sh4m4n.zip`:

	secret1.txt
	secret2.txt
	secret3.txt
	secret4.txt
	.secret5.txt

Intentos de extracción directa fallidos debido a la contraseña.

### 2. Extracción de hash

Se extrajo el hash de uno de los archivos para iniciar el crackeo:

```bash
zip2john -o secret1.txt forensic_sh4m4n.zip > hash_secret1.txt
```

Nota: Todos los archivos usan la misma contraseña, aunque se prepararon con hashes separados, probablemente para distraer al usuario.

### 3. Crackeo de la contraseña

Se utilizó John the Ripper en modo incremental:

```bash
john --incremental hash_secret1.txt
```

Resultado exitoso:

```bash
`forensic_sh4m4n.zip/secret1.txt:zz102:secret1.txt:forensic_sh4m4n.zip::forensic_sh4m4n.zip`
```

Contraseña encontrada: `zz102`.

### 4. Próximo paso

Con la contraseña `zz102`, se pueden extraer todos los archivos del ZIP:

```bash
unzip -P zz102 forensic_sh4m4n.zip
```

### 5. Análisis de secretos y flags obtenidas

Tras extraer los 5 archivos con la contraseña `zz102`, se procedió a inspeccionar cada uno:

- `secret1.txt` → texto UTF-8, líneas muy largas, secuencias de A–F (hexadecimal). Contiene una flag: `CITC2024{Th3_fl@g_0f_Th3_k0@L@}`
- `secret2.txt` → texto UTF-8, líneas muy largas. Contiene una flag en Base64: `CITC2024{RXN0YV9Ob19Fc19sYV9GbGFnCg==}` traducido del base64 queda `Esta_No_Es_la_Flag`
- `secret3.txt` → texto UTF-8, líneas muy largas. Contiene una flag: `CITC2024{3st4_n0_3s_L4_fl@g}`
- `secret4.jpg` → JPEG (antes `.txt`), aún no analizada con herramientas de imagen. Contiene información que puede ser relevante para la flag.
- `secret5.txt` → texto UTF-8, líneas muy largas. Contiene una flag: `CITC2024{R3To_F0R3nCiNg_F4lse}`

Observaciones importantes:

- Ninguna de las flags obtenidas corresponde a la flag real del reto.
- La flag en Base64 (`RXN0YV9Ob19Fc19sYV9GbGFnCg==`) decodificada no corresponde a la flag final; solo dice `"Esta_No_Es_la_Flag"`.
- La distribución de las flags falsas sigue un patrón:
  - `secret1.txt` → primera flag
  - `secret2.txt` → segunda flag
  - `secret3.txt` → tercera flag
  - `secret4.txt` → imagen (aún no analizada)
  - `secret5.txt` → cuarta flag
- Esto concuerda con el enunciado: Juan oculta 5 secretos, algunos de los cuales son pistas falsas.
- Se intentó convertir cada flag a MD5, pero ninguna correspondía a la flag final.

**Conclusión parcial:** Hasta este punto, se han extraído todas las flags de texto y se ha identificado la presencia de la imagen (`secret4.txt`) como posible fuente de la flag real. La pista principal son los contenidos de los archivos, aunque muchos son decoy. La investigación deberá continuar analizando la imagen para descubrir la flag final.

## Análisis de `secret4`

### 1. Archivo

- Nombre: `secret4.jpg`
- Observación: Imagen de Snoopy, tamaño 728x546, RGB, no entrelazada.

### 2. Comandos y resultados

#### 3.1. Identificación del archivo

```bash
file secret4.jpg
# Resultado: secret4.jpg: JPEG image data, JFIF standard 1.01
```

#### 3.2. Inspección visual

La imagen es una foto de Snoopy. Se procede a probar esteganografía.

#### 3.3. Extracción

```bash
steghide extract -sf secret4.jpg
```

Resultado:
```
wrote extracted data to "secret.txt"
```

#### 3.4. Lectura del contenido extraído

```bash
cat secret.txt
```

Resultado: `CITC2024{Th3_Sh4m4n_s3cr3t_fl@g}`

## Flag

```cidsi{Th3_Sh4m4n_s3cr3t_fl@g}
```

Formato bruto:
```CITC2024{Th3_Sh4m4n_s3cr3t_fl@g}```
