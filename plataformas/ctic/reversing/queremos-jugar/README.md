# Write-up: Queremos Jugar

## Reto

- Plataforma: CTIC / AGETIC
- Nombre: Queremos Jugar
- Categoría: Reversing
- Puntos: 200
- Archivo: `keygen_gameDota.c`
- Autor: go2hack
- Estado: `resuelto y verificado localmente`

> Tenemos un ejecutable de un juego, el cual necesita una llave para poder acceder.

## Resumen

El archivo entregado contiene el código fuente C del verificador. La llave debe medir exactamente diez caracteres y cada posición sigue una regla distinta:

- índices pares: una letra calculada como `'A' + índice`;
- índices impares: un dígito calculado como `'0' + índice`.

Al evaluar las diez posiciones se obtiene:

```text
A1C3E5G7I9
```

El propio programa imprime esa llave como flag y la solución fue verificada compilando y ejecutando el código.

## Flag

```text
A1C3E5G7I9
```

No se añade `citc{...}` porque el programa llama directamente a `print_flag(input)` y muestra la entrada sin ningún prefijo o envoltura adicional.

## Herramientas utilizadas

- `file`: comprobar que el archivo es código fuente C y no un ejecutable.
- Editor de texto: leer `verify_key()` y seguir la condición.
- `gcc`: compilar el programa para verificar la llave.
- Binary Ninja: no es necesario al disponer del código fuente; sería útil si solo se entregara el ejecutable compilado.

## Reconocimiento

```bash
file keygen_gameDota.c
```

Resultado:

```text
C source, Unicode text, UTF-8 text
```

Esto cambia el flujo de trabajo: no hace falta descompilar un binario. La forma más directa y humana es leer primero `main()` y después la función que decide si la llave es válida.

## Seguir el flujo desde `main`

El programa lee como máximo 63 caracteres:

```c
scanf("%63s", input);
```

Luego llama a:

```c
if (verify_key(input)) {
    print_flag(input);
} else {
    printf("Invalid key!\n");
}
```

Por tanto, la pregunta importante es: **¿qué debe devolver verdadero dentro de `verify_key()`?**

## Primera condición: longitud exacta

```c
if (strlen(key) != 10) {
    return 0;
}
```

La llave tiene exactamente diez caracteres. Una entrada más corta o más larga se rechaza antes de revisar su contenido.

## Segunda condición: patrón por posición

El bucle visita los índices del `0` al `9`:

```c
for (int i = 0; i < 10; i++) {
    if ((i % 2 == 0 && key[i] != 'A' + i) ||
        (i % 2 != 0 && key[i] != '0' + i)) {
        return 0;
    }
}
```

### Entender `i % 2`

El operador `%` obtiene el resto de una división:

- `i % 2 == 0`: el índice es par (`0`, `2`, `4`, `6`, `8`);
- `i % 2 != 0`: el índice es impar (`1`, `3`, `5`, `7`, `9`).

### Entender `'A' + i`

En C, un carácter también tiene un valor numérico ASCII. `'A'` vale 65:

```text
'A' + 0 = 65 = A
'A' + 2 = 67 = C
'A' + 4 = 69 = E
'A' + 6 = 71 = G
'A' + 8 = 73 = I
```

Como esta regla solamente se utiliza en índices pares, produce `A`, `C`, `E`, `G`, `I`.

### Entender `'0' + i`

El carácter `'0'` vale 48 en ASCII. Sumar un número entre cero y nueve produce el carácter de ese dígito:

```text
'0' + 1 = 49 = '1'
'0' + 3 = 51 = '3'
'0' + 5 = 53 = '5'
'0' + 7 = 55 = '7'
'0' + 9 = 57 = '9'
```

No es el número entero `1`: es el carácter ASCII `'1'`, que es lo que se almacena en una cadena.

## Construcción manual de la llave

| Índice `i` | Tipo | Operación | Carácter esperado |
|---:|---|---|:---:|
| 0 | par | `'A' + 0` | `A` |
| 1 | impar | `'0' + 1` | `1` |
| 2 | par | `'A' + 2` | `C` |
| 3 | impar | `'0' + 3` | `3` |
| 4 | par | `'A' + 4` | `E` |
| 5 | impar | `'0' + 5` | `5` |
| 6 | par | `'A' + 6` | `G` |
| 7 | impar | `'0' + 7` | `7` |
| 8 | par | `'A' + 8` | `I` |
| 9 | impar | `'0' + 9` | `9` |

Leyendo la última columna de arriba hacia abajo:

```text
A1C3E5G7I9
```

## Por qué la condición parece complicada

La condición está escrita desde el punto de vista del rechazo:

```c
(índice par Y carácter incorrecto)
O
(índice impar Y carácter incorrecto)
```

Si cualquiera de esas situaciones ocurre, retorna `0`. Para llegar al `return 1`, cada posición debe coincidir con su carácter esperado.

Una forma equivalente y más fácil de pensar es:

```text
si el índice es par, necesito 'A' + i;
si el índice es impar, necesito '0' + i.
```

## Verificación

Se compila sin optimizaciones para conservar una estructura fácil de estudiar:

```bash
gcc -Wall -Wextra -O0 -g keygen_gameDota.c -o keygen_gameDota
```

Después:

```bash
./keygen_gameDota
```

Entrada:

```text
A1C3E5G7I9
```

Salida:

```text
Enter the key: Congratulations! Here is your flag: A1C3E5G7I9
```

## Cómo se vería si solo entregaran el binario

En Binary Ninja el recorrido sería:

1. abrir `main`;
2. seguir la llamada que recibe la entrada;
3. entrar a `verify_key`;
4. identificar la comparación de longitud con `10`;
5. reconocer el bucle de diez iteraciones;
6. observar una rama para índices pares y otra para impares;
7. traducir las constantes ASCII `0x41` (`A`) y `0x30` (`0`);
8. construir la tabla de posiciones y verificar la llave.

Aunque se podría parchear el retorno de `verify_key()` para que siempre sea verdadero, eso no sería suficiente para conocer la llave: `print_flag()` imprime exactamente la entrada proporcionada. Aquí es mejor reconstruir el patrón real.

## Qué enseña el reto

- En C, los caracteres se pueden sumar porque internamente son valores numéricos.
- Los índices de una cadena comienzan en cero.
- `% 2` permite separar posiciones pares e impares.
- Las validaciones suelen escribirse como condiciones de rechazo, lo que puede hacerlas parecer más complicadas.
- Antes de usar un descompilador se debe comprobar si se recibió código fuente o un ejecutable.
- Parchear una validación no siempre revela la flag si el programa imprime la entrada del usuario.
