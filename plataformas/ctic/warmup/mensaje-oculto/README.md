# Write-up: Mensaje Oculto

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Warmup / Esteganografía
- Archivo/objetivo: `JovenP.jpg`
- Estado: `resuelto`

## Descripción del reto

"Te hemos proporcionado una la imagen de una bella dama. Debes infiltrarte y extraer la información vital. ¿Estás listo para esta operación encubierta? ¡El tiempo corre!"" Autor: Franolig. "

`JovenP.jpg`

---

## Revision y Verificacion

* Usar `exiftool` en la imagen para observar pistas
* Verificar si tiene alguna clave oculta con `steghide`

```bash
steghide info JovenP.jpg
```

	En este caso se virifica que tiene dato oculto, le damos enter para ver si no lleva salvo conducto y extraer sus datos

---
* La imagen proporciona una nueva imagen usando el comando `steghide`, la imagen no tiene clave puede ser vacia.
* la imagen que deja es: `tarija.jpeg`
* hacemos lo mismo como la anterior

---
la imagen `tarija.jpeg` tiene un salvoconducto, usaremos la herramienta:
* `stegseek` para hacer una fuerza bruta con diccionario en la imagen usando **rockyou.txt**

```bash
stegseek --crack <imagen> <diccionario.txt>
```

* En este caso al clave que detecto es antonio y nos da un base 64

`YkRSZmJqSERzV0ZmWkROZk5YVTFYekJxTUhNPQ==`

* Usamos `CyberChef` para decifrar el mensaje
* Segun `CyberChef` usa dos veces seguidas base64, al final deja este mensaje:

**`l4_n1ña_d3_5u5_0j0s`**

**Convertimos el mensaje a md5** ->  **`0d37c9438c3aecf52d10c1acf5458b78`**

```bash
citc{0d37c9438c3aecf52d10c1acf5458b78}
```

## Cadena de resolución

`JovenP.jpg → Steghide sin contraseña → tarija.jpeg → StegSeek + RockYou → antonio → Base64 ×2 → texto → MD5 → flag`

## Flag

```text
citc{0d37c9438c3aecf52d10c1acf5458b78}
```

## Qué debo recordar para otro CTF

- Repetir el triage sobre cada imagen extraída.
- Steghide puede usar contraseña vacía; una segunda capa puede usar una contraseña distinta.
- Una salida Base64 puede contener otra capa Base64.

## Aporte para el arsenal

- Encadenar extracciones de Steghide con un límite de profundidad.
- Entregar cadenas extraídas al decodificador por capas.
