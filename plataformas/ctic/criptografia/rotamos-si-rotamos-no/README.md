# Write-up: Rotamos si rotamos no

## Reto

- Plataforma: AGETIC / CTIC / CIDSI
- Categoría: Criptografía
- Estado: `resuelto`

## Descripción

Cifrados de cajon.

638CK370A320638

---

rotar el codigo con el siguiente orden:
ROT47->ROT13:
salida: `rotemos_con_rot`

md5 -> `fb297129c28923015076ff9bf921f244`

CTF->  `citc{fb297129c28923015076ff9bf921f244}`

## Cadena de resolución

`638CK370A320638 → ROT47 → ROT13 → rotemos_con_rot → MD5 → flag`

## Flag

```text
citc{fb297129c28923015076ff9bf921f244}
```

## Qué debo recordar para otro CTF

- Las transformaciones encadenadas deben registrarse en orden.
- Una salida intermedia legible todavía puede requerir el formato indicado por el reto.

## Aporte para el arsenal

- Probar rotaciones baratas dentro del decodificador por capas y mostrar la ruta completa.
