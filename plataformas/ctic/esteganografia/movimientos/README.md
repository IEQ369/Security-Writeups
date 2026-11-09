# Write-up: Movimientos

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Esteganografía
- Archivo/objetivo: `Movimientos.jpg`
- Estado: `resuelto`

## Descripción
Desempolvando los pasos desde los 90'. formato de la flag cidsi{flag_en_md5}

archivo: `Movimientos.jpg`
La imagen del reto contiene una secuencia de figuras del cifrado Dancing Men.

---
## Analisis
En los metadatos no hay nada asi que solo queda entender cada simbolo
encontre este link:
https://kryptografie.de/kryptografie/chiffre/dancing-men-code.htm
* Especificamente esta imagen
  ![](https://kryptografie.de/kryptografie/chiffre/images/dancing-men-code3.png)

Descifrarlo seria:
`reklawnoom` Pero la frase realmente es al reves -> `moonwalker`

### Flag MD5
* md5 -> `67192ced7898f434effd7c841c83e930`
* **flag:**
```bash
cidsi{67192ced7898f434effd7c841c83e930}
```

Fuente contrastada: https://github.com/IEQ369/Security-Writeups/tree/main/writeups/steganography/movimientos

## Cadena de resolución

`imagen → símbolos Dancing Men → reklawnoom → invertir → moonwalker → MD5 → flag`

## Qué debo recordar para otro CTF

- Algunos retos usan alfabetos visuales conocidos y no esteganografía de píxeles.
- Si una traducción produce una palabra casi legible al revés, invertir es una prueba pequeña y justificable.

## Aporte para el arsenal

- Clasificar símbolos visuales como una ruta humana; evitar afirmar automáticamente un alfabeto sin coincidencias suficientes.

## Capturas de la publicación original

<img width="455" height="180" alt="dancing-men-code3" src="https://github.com/user-attachments/assets/8241e218-f9ca-4a09-805b-dbf87eda690b" />
