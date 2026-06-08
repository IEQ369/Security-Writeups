# Los 5 secretos de Juan

- Plataforma: CTF CGII/AGETIC (ctf.cgii.gob.bo)
- Categoría: Forense
- Puntos: 190
- Fecha: 2026-06-07
- Tags: #forense #zip #steghide #ctf #steganography

## Resumen
Reto forense con un ZIP protegido que contiene varios archivos de texto y una imagen con esteganografía.Solo `secret4` lleva a la flag real; el resto son distractores. La imagen se extrajo con `steghide` usando la clave `aazip`, obteniendo un texto con la flag final.

## Material
- `forensic_sh4m4n.zip`

## Reconocimiento

Identificar tipo de archivo:

```bash
file forensic_sh4m4n.zip
```

Resultado: ZIP archive data.

## Procedimiento

Extraer el ZIP con contraseña `aazip`:

```bash
unzip -P 'aazip' forensic_sh4m4n.zip
```

Archivos generados:
- secret1.txt
- secret2.txt
- secret3.txt
- secret4.jpeg
- .secret5.txt

Inspección rápida de los textos: contienen flags falsas y pistas de distracción.

La imagen objetivo es `secret4.jpeg`. Se valida su formato:

```bash
file secret4.jpeg
```

Resultado: JPEG image.

Extraer información oculta con `steghide`:

```bash
steghide extract -sf secret4.jpeg -p 'aazip'
```

Archivo extraído:
- secret.txt

Leer el contenido:

```bash
cat secret.txt
```

Resultado: `CITC2024{Th3_Sh4m4n_s3cr3t_fl@g}`

## Flag

```cidsi{Th3_Sh4m4n_s3cr3t_fl@g}
```

Formato bruto:
```CITC2024{Th3_Sh4m4n_s3cr3t_fl@g}```

## Notas
- El reto tiene 5 secretos, pero solo `secret4` contiene la flag válida.
- Los archivos `.txt` iniciales son distractores.
- La clave `aazip` no se obtuvo por fuerza bruta ni lista de palabras durante el reto.
- Herramientas usadas: `unzip`, `file`, `steghide`, `cat`.
