# Señal Perdida

## Reto

- Plataforma: competencia AGETIC / CITC 2026.
- Categoría: Criptografía. Puntos: 90. Autor: Nicki Nicole.
- Estado: **parcial: texto legible recuperado, rechazado como respuesta**.

> Una transmisión de origen SECRETO. «No olvides que todo lo que va, regresa». El mensaje es IMPORTANTE.

## Entrada conservada

No había archivo: esta cadena venía en el enunciado y permite repetir el análisis completamente:

```text
MjEyMTIxNmE2ZTMzNmI3NzM0NzkzNDY1MzUzMzZjMzMzMTQzMzQ3MDMxNzE3MjQ1MzA2MTYxMzM1MTZhNmM3MjU5YzJhMWMyYTFjMmEx
```

## Resumen e hipótesis

Se encontraron capas Base64, hexadecimal, inversión, Vigenère con clave `SECRETO` y desplazamiento César. La frase final es coherente, pero el usuario confirmó su rechazo con y sin signos de admiración. No está demostrado que solo falte un formato de envío.

## Resolución paso a paso

En CyberChef:

1. **From Base64** produce una cadena compuesta por dígitos hexadecimales.
2. **From Hex** produce `!!!jn3kw4y4e53l31C4p1qrE0aa3QjlrY¡¡¡`.
3. **Reverse**, sobre caracteres, produce `¡¡¡YrljQ3aa0Erq1p4C13l35e4y4wk3nj!!!`. La frase «todo lo que va, regresa» justificaba probar inversión.
4. **Vigenère Decode**, clave `SECRETO`. Avanzar la clave solo al procesar letras ASCII, dejando números y puntuación intactos.
5. Rotación alfabética de **21** hacia adelante (equivalente a −5). Se encontró enumerando los 26 desplazamientos; no estaba explícita en el enunciado.

Resultado:

```text
¡¡¡BienH3ch0Hij1t4T13n35l4b4nd3ra!!!
```

## Validación y límites

Al invertir estas operaciones se reconstruye la entrada exactamente. Esto comprueba que implementamos transformaciones inversas consistentes, **no que sea la bandera que espera el juez**. Cualquier transformación reversible conserva esa propiedad.

Los MD5 propuestos después eran hipótesis, sin regla confirmada que los exigiera. No hay aceptación registrada de ninguno. Tampoco está demostrada una función de la palabra `IMPORTANTE`.

### Candidatas MD5 registradas (sin aceptación)

El registro anterior (SOLVE) probó la hipótesis `flag = citc{md5(mensaje)}` sobre cuatro variantes del texto, sin que ninguna tuviera confirmación:

| Variante del texto | MD5 |
|---|---|
| `¡¡¡BienH3ch0Hij1t4T13n35l4b4nd3ra!!!` | `38438e355afe9fd8d985cd4df02b31b7` |
| `BienH3ch0Hij1t4T13n35l4b4nd3ra` (sin signos) | `7d2151b471ac4ba986ff5fc93a26fbcb` |
| con espacios (forma no detallada en el registro) | `2b0187a0be97d3d6cca536d8b6688716` |
| con espacios y `¡¡¡…!!!` | `086cfd65f80af35ea47ca6a7f8cefc39` |

El registro también deja anotada, sin probar su MD5 ni enviarla, una variante en minúsculas: `¡¡¡bienh3ch0hij1t4t13n35l4b4nd3ra!!!`. Ninguna de estas cadenas ni sus MD5 tiene aceptación registrada; se conservan aquí para no repetir intentos.

## Cadena de resolución

`Base64 → hexadecimal → invertir caracteres → Vigenère(SECRETO) → César −5 → frase rechazada / pendiente`

## Flag

**No validada.** El texto de arriba es el resultado del análisis, no una bandera aceptada. Se conserva el registro anterior y `solve.py` como antecedentes.

## Qué debo recordar para otro CTF

Conservar leet, mayúsculas y puntuación originales. No aplicar MD5 automáticamente por haberlo usado en otra competencia.

## Aporte para el arsenal

Registrar la cadena de transformaciones y si la clave avanza sobre letras o sobre todas las posiciones. Diferenciar texto coherente, inversión matemática correcta y aceptación real.


## Código de apoyo: solve.py

Guarda este bloque como `solve.py` si sigues los comandos que lo utilizan. Código del análisis original; no se ha vuelto a ejecutar para esta publicación.

```python
import itertools
s="!!!jn3kw4y4e53l31C4p1qrE0aa3QjlrY¡¡¡"
core=s[3:-3]
for name,t in(("fwd",core),("rev",core[::-1])):
  for adv in("letters","all"):
    for sign in(1,-1):
      o="";k=0
      for c in t:
        if c.isalpha():
          b=65 if c.isupper() else 97
          sh=ord("SECRETO"[k%7])-65
          o+=chr((ord(c)-b+sign*sh)%26+b);k+=1
        else:
          o+=c
          if adv=="all":k+=1
      print(name,adv,sign,o)
print("----")
import re
L=str.maketrans("3401","eaoi")
words="el la de que flag bandera secret senal mensaje ctf regresa vuelve perdida nicki nicole clave".split()
outs=[]
for name,t in(("fwd",core),("rev",core[::-1])):
  for adv in("letters","all"):
    for sign in(1,-1):
      o="";k=0
      for c in t:
        if c.isalpha():
          b=65 if c.isupper() else 97
          o+=chr((ord(c)-b+sign*(ord("SECRETO"[k%7])-65))%26+b);k+=1
        else:
          o+=c
          if adv=="all":k+=1
      for r in range(26):
        p="".join(chr((ord(c)-(65 if c.isupper() else 97)+r)%26+(65 if c.isupper() else 97)) if c.isalpha() else c for c in o)
        d=p.translate(L).lower()
        sc=sum(w in d for w in words)
        print(sc,name,adv,sign,r,p)
```
