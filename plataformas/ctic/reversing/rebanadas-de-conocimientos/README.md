# Write-up: Rebanadas de Conocimientos

## Reto

- Plataforma: CTIC / AGETIC
- Nombre: Rebanadas de Conocimientos
- Categoría: Reversing
- Puntos: 300
- Archivo: `Rebanadas-de-Conocimientos.dat`
- Estado: `resuelto; flag verificada contra todas las condiciones del programa`

> Pon a prueba tus habilidades.

## Resumen

El archivo `.dat` es una línea de texto hexadecimal. Al convertirla a bytes aparece un programa Python que verifica la flag mediante varios *slices* o rebanadas.

Cada slice fija caracteres en determinadas posiciones de una cadena de longitud 26. Al colocar todas las restricciones en una tabla se reconstruye una única flag.

## Flag

```text
cidsi{5l1c35_0f_kn0wl3dg3}
```

## Herramientas utilizadas

- `file`: comprobar que el `.dat` es texto y no un ejecutable.
- `xxd`: reconocer y revertir hexadecimal plano.
- Editor de texto: leer el verificador Python.
- Una tabla de índices: reconstruir manualmente la cadena.
- Python: comprobar al final que todos los slices coinciden.

## Reconocimiento

```bash
file Rebanadas-de-Conocimientos.dat
```

Resultado:

```text
ASCII text, with very long lines, with no line terminators
```

La extensión `.dat` no determina el formato real. Al observar el inicio:

```bash
xxd -l 64 Rebanadas-de-Conocimientos.dat
```

Se ven caracteres como:

```text
666c6167203d2069...
```

La entrada está formada únicamente por parejas hexadecimales. Además:

```text
66 = f
6c = l
61 = a
67 = g
```

Los primeros bytes forman `flag`, así que hay evidencia suficiente para revertir toda la línea como hexadecimal.

## Decodificar la primera capa

```bash
xxd -r -p Rebanadas-de-Conocimientos.dat
```

Opciones:

- `-r`: operación inversa, de hexadecimal a bytes;
- `-p`: hexadecimal plano, sin offsets ni columnas.

El resultado es este verificador:

```python
flag = input('Enter flag: ')

def fail():
    print('Intenta de Nuevo!')
    exit(-1)

if len(flag) != 26: fail()
if flag[:6] != 'cidsi{': fail()
if flag[-1] != '}': fail()
if flag[5::3] != '{15fnlg': fail()
if flag[4::4] != 'i1_kl3': fail()
if flag[3::5] != 's100g': fail()
if flag[6::3] != '5c__033': fail()
if flag[7::3] != 'l30kwd}': fail()
```

## Entender los slices de Python

La sintaxis general es:

```python
texto[inicio:fin:salto]
```

Si `fin` está vacío, continúa hasta terminar la cadena. Por ejemplo:

```python
flag[5::3]
```

significa:

1. empezar en el índice 5;
2. tomar ese carácter;
3. avanzar tres posiciones cada vez;
4. continuar hasta el final.

Los índices comienzan en cero. Para una flag de 26 caracteres son:

```text
0, 1, 2, ..., 24, 25
```

## Colocar las condiciones simples

```python
len(flag) == 26
flag[:6] == 'cidsi{'
flag[-1] == '}'
```

Esto fija:

```text
Índice:  0 1 2 3 4 5                         25
Valor:   c i d s i {                          }
```

`flag[-1]` significa el último carácter.

## Resolver cada rebanada

### `flag[5::3] == '{15fnlg'`

Los índices son:

```text
5, 8, 11, 14, 17, 20, 23
```

Se asigna:

```text
5={, 8=1, 11=5, 14=f, 17=n, 20=l, 23=g
```

### `flag[4::4] == 'i1_kl3'`

Índices:

```text
4, 8, 12, 16, 20, 24
```

Valores:

```text
4=i, 8=1, 12=_, 16=k, 20=l, 24=3
```

### `flag[3::5] == 's100g'`

Índices:

```text
3, 8, 13, 18, 23
```

Valores:

```text
3=s, 8=1, 13=0, 18=0, 23=g
```

### `flag[6::3] == '5c__033'`

Índices:

```text
6, 9, 12, 15, 18, 21, 24
```

Valores:

```text
6=5, 9=c, 12=_, 15=_, 18=0, 21=3, 24=3
```

### `flag[7::3] == 'l30kwd}'`

Índices:

```text
7, 10, 13, 16, 19, 22, 25
```

Valores:

```text
7=l, 10=3, 13=0, 16=k, 19=w, 22=d, 25=}
```

## Tabla completa

| Índice | Carácter | Procedencia principal |
|---:|:---:|---|
| 0 | `c` | prefijo |
| 1 | `i` | prefijo |
| 2 | `d` | prefijo |
| 3 | `s` | prefijo / `[3::5]` |
| 4 | `i` | prefijo / `[4::4]` |
| 5 | `{` | prefijo / `[5::3]` |
| 6 | `5` | `[6::3]` |
| 7 | `l` | `[7::3]` |
| 8 | `1` | varias rebanadas |
| 9 | `c` | `[6::3]` |
| 10 | `3` | `[7::3]` |
| 11 | `5` | `[5::3]` |
| 12 | `_` | `[4::4]` / `[6::3]` |
| 13 | `0` | `[3::5]` / `[7::3]` |
| 14 | `f` | `[5::3]` |
| 15 | `_` | `[6::3]` |
| 16 | `k` | `[4::4]` / `[7::3]` |
| 17 | `n` | `[5::3]` |
| 18 | `0` | `[3::5]` / `[6::3]` |
| 19 | `w` | `[7::3]` |
| 20 | `l` | `[4::4]` / `[5::3]` |
| 21 | `3` | `[6::3]` |
| 22 | `d` | `[7::3]` |
| 23 | `g` | `[3::5]` / `[5::3]` |
| 24 | `3` | `[4::4]` / `[6::3]` |
| 25 | `}` | último / `[7::3]` |

Leyendo en orden del índice 0 al 25:

```text
cidsi{5l1c35_0f_kn0wl3dg3}
```

## Verificación final

Una vez reconstruida manualmente, se comprueban todas las condiciones:

```bash
python3 -c "f='cidsi{5l1c35_0f_kn0wl3dg3}'; print(len(f)); print(f[:6],f[-1],f[5::3],f[4::4],f[3::5],f[6::3],f[7::3])"
```

Resultado:

```text
26
cidsi{ } {15fnlg i1_kl3 s100g 5c__033 l30kwd}
```

Cada valor coincide exactamente con el verificador.

## Por qué pertenece a reversing

No siempre se entrega un ELF. Aquí se analiza un programa de validación y se trabaja hacia atrás desde sus condiciones hasta encontrar la única entrada aceptada. Esa reconstrucción de la entrada esperada es una forma de reversing.

## Qué enseña el reto

- La extensión de un archivo puede ser engañosa.
- Una cadena de caracteres hexadecimales no es lo mismo que un hexdump de bytes arbitrarios.
- Cómo funciona `texto[inicio:fin:salto]` en Python.
- Los índices empiezan en cero y los negativos cuentan desde el final.
- Varias restricciones parciales pueden reconstruir una cadena completa.
- Es mejor resolver primero con una tabla y usar un script solo como verificación.

## Aporte para el futuro arsenal

- Detectar texto compuesto casi exclusivamente por parejas hexadecimales.
- Decodificarlo y volver a clasificar el resultado.
- Reconocer verificadores con `len`, prefijos, sufijos y slices.
- Extraer restricciones posicionales y comprobar si producen una única solución.
- Informar conflictos entre condiciones en lugar de inventar caracteres.
