# Movimientos

- Plataforma: CTF CGII/AGETIC (ctf.cgii.gob.bo)
- Categoría: Esteganografía
- Puntos: 100
- Dificultad: Easy
- Tags: #steganography #dancing-men #cipher #image

## Resumen
Reto de esteganografía donde el archivo entrega una imagen con un cifrado de símbolos tipo Dancing Men. No hay metadatos útiles ni archivos ocultos; la resolución depende de identificar el cifrado y traducir cada figura a su letra correspondiente.

## Material
- `Movimientos.jpg`

## Descripción del reto

"Desempolvando los pasos desde los 90'. formato de la flag cidsi{flag_en_md5}"

## Procedimiento

### Metadatos

En los metadatos no hay nada, así que el camino es analizar el contenido visual de la imagen.

### Análisis de símbolos

La imagen muestra figuras humanas con distintas posturas:
- Cada figura equivale a una letra del alfabeto.
- Se compara con la referencia de Dancing Men:
  - https://kryptografie.de/kryptografie/chiffre/dancing-men-code.htm
  - Específicamente la imagen del sitio que corresponde al reto.

Mapeo de símbolos a letras (según la referencia y las repeticiones observadas):

Secuencia leída de izquierda a derecha:
`r e k l a w n o o m`

### Inversión del mensaje

Al observar el sentido de las figuras, el mensaje está escrito al revés.

Cadena real:
`moonwalker`

### Cálculo de la flag

Formato: `cidsi{md5}`

```bash
echo -n "moonwalker" | md5sum
```

Resultado:
```bash
67192ced7898f434effd7c841c83e930
```

## Flag

```
cidsi{67192ced7898f434effd7c841c83e930}
```
