# Write-up: Escape de ASCIITRAZ

## Reto

- Plataforma: CTIC / AGETIC
- Nombre: Escape de ASCIITRAZ
- Categoría: Reversing
- Puntos: 500
- Archivo: `Escape_de_asciitraz`
- Estado: `resuelto y verificado localmente`

> Estás en ASCIITRAZ. ¿Podrás encontrar la salida?

## Resumen

El programa presenta primero una suma sencilla dibujada con caracteres ASCII y después pide escoger cinco puertas. La animación es decorativa: el ejecutable guarda las cinco elecciones en un arreglo y las compara con valores calculados mediante operaciones aritméticas y XOR.

La ruta correcta es:

```text
1 1 1 2 5
```

Después muestra solamente el principio de la flag y solicita una contraseña. Las comparaciones carácter por carácter revelan `liberty`. Si la contraseña es correcta, otra función descifra e imprime la parte restante de la flag.

## Flag

```text
cidsi{052c3eaaa25e4b148110b00345be17fd}
```

## Herramientas utilizadas

- `file`: identificar el tipo de ejecutable.
- `strings`: localizar mensajes interesantes y comprobar que la flag completa no está en texto claro.
- Binary Ninja: seguir referencias, analizar la función que valida las puertas y entender el descifrado final.
- `objdump`: confirmar instrucciones y constantes concretas.
- Ejecución local: verificar la ruta, la contraseña y la flag final.

No fue necesario escribir un script para resolverlo.

## Reconocimiento inicial

Primero se identifica el archivo:

```bash
file Escape_de_asciitraz
```

Resultado relevante:

```text
ELF 64-bit LSB executable, x86-64, statically linked, stripped
```

Esto nos dice:

- es un ejecutable Linux de 64 bits;
- `statically linked` significa que incluye gran parte de las bibliotecas dentro del propio binario, por eso pesa más y contiene muchas funciones;
- `stripped` significa que se eliminaron nombres útiles como `main` o `check_doors`.

Si no tiene permiso de ejecución:

```bash
chmod u+x Escape_de_asciitraz
./Escape_de_asciitraz
```

Al ejecutarlo se observa el flujo real:

1. muestra dos números en arte ASCII y pide su suma;
2. solicita una puerta entre 1 y 5 durante cinco fases;
3. comprueba la secuencia completa;
4. si es correcta, pide una contraseña.

La suma inicial cambia entre ejecuciones porque se generan dos números aleatorios del 0 al 9. No es una clave oculta: basta con leer los dibujos y sumar.

## Buscar los puntos importantes

Una primera revisión ayuda a encontrar los mensajes:

```bash
strings -a -n 5 Escape_de_asciitraz | less
```

Entre ellos aparecen:

```text
Fase %d, escoge una puerta:
Escogiste las puertas:
cidsi{052c3e
Ingrese la contraseña:
Contraseña incorrecta
```

La cadena `cidsi{052c3e` no es la flag completa. El programa la imprime como primera mitad y genera el resto después de validar la contraseña.

## Llegar a la función principal en Binary Ninja

Como el archivo está `stripped`, es más práctico entrar mediante una cadena que buscar `main` por nombre:

1. abrir el binario en Binary Ninja;
2. abrir **Strings**;
3. buscar `Fase %d, escoge una puerta:`;
4. abrir sus referencias cruzadas o **xrefs**;
5. saltar a la función que usa esa cadena;
6. renombrar esa función como `main` para trabajar cómodamente.

En la vista de pseudocódigo se distingue un bucle que se repite cinco veces. Cada entrada válida se almacena aproximadamente así:

```c
puertas[indice] = opcion;
indice++;
```

Después del bucle, el arreglo `puertas` se entrega a otra función. Esa función devuelve `1` para la ruta correcta y `0` para una incorrecta. Conviene renombrarla como `validar_puertas`.

## Reconstruir las cinco puertas

La función comienza con una constante base:

```c
base = 2;
valor1 = base + 1;
valor2 = base * 2;
valor3 = base << 2;
valor4 = base * 3 + 1;
valor5 = base * 4 + 1;
```

`base << 2` desplaza dos bits a la izquierda, lo que equivale a multiplicar por cuatro. Por tanto:

```text
valor1 = 3
valor2 = 4
valor3 = 8
valor4 = 7
valor5 = 9
```

Luego compara cada elemento del arreglo con esos valores tras aplicar XOR:

```c
puertas[0] == (valor1 ^ 2)
puertas[1] == (valor2 ^ 5)
puertas[2] == (valor3 ^ 9)
puertas[3] == (valor4 ^ 5)
puertas[4] == (valor5 ^ 12)
```

Resolviendo las operaciones:

| Fase | Cálculo | Puerta |
|---:|---|---:|
| 1 | `3 XOR 2` | `1` |
| 2 | `4 XOR 5` | `1` |
| 3 | `8 XOR 9` | `1` |
| 4 | `7 XOR 5` | `2` |
| 5 | `9 XOR 12` | `5` |

Por ejemplo, en binario:

```text
3 = 0011
2 = 0010
    ---- XOR
1 = 0001
```

La secuencia correcta queda:

```text
1 1 1 2 5
```

## Recuperar la contraseña

Al seguir la rama de éxito de `validar_puertas`, el programa imprime el fragmento:

```text
cidsi{052c3e
```

Luego lee una palabra y compara sus bytes individualmente. En Binary Ninja se observa el equivalente a:

```c
entrada[0] == 'l'
entrada[1] == 'i'
entrada[2] == 'b'
entrada[3] == 'e'
entrada[4] == 'r'
entrada[5] == 't'
entrada[6] == 'y'
```

Leyendo los caracteres en orden:

```text
liberty
```

Aquí no hay que descifrar un hash ni adivinar una wordlist: la contraseña está repartida entre comparaciones directas.

## Entender la segunda mitad de la flag

Si la contraseña es correcta, se llama a una última función. Esta construye los siguientes bytes ofuscados:

```text
2f 2f 2f 73 74 33 75 2e 70 75 79 70 70 71
2e 71 71 72 75 74 2e 33 70 76 32 30 3c
```

La rutina de decodificación realiza dos pasos sobre cada byte:

1. XOR con `0x41`;
2. ROT13 si el resultado es una letra.

Un ejemplo con el primer byte:

```text
0x2f XOR 0x41 = 0x6e = n
ROT13(n) = a
```

Al aplicar la misma lógica a toda la secuencia se obtiene:

```text
aaa25e4b148110b00345be17fd}
```

Esta no es una flag independiente. Se concatena con la parte impresa anteriormente:

```text
cidsi{052c3e + aaa25e4b148110b00345be17fd}
```

Resultado:

```text
cidsi{052c3eaaa25e4b148110b00345be17fd}
```

## Verificación normal

Se ejecuta:

```bash
./Escape_de_asciitraz
```

El procedimiento es:

1. resolver la suma aleatoria mostrada al principio;
2. introducir, una por una, las puertas `1`, `1`, `1`, `2`, `5`;
3. introducir `liberty` cuando pida la contraseña.

La salida relevante es:

```text
Escogiste las puertas:
1 1 1 2 5

Lo lograste!!!
cidsi{052c3e

Ingrese la contraseña:
Felicitaciones
aaa25e4b148110b00345be17fd}
```

Unidas, ambas líneas producen la flag verificada.

## Qué enseña el reto

- Cómo entrar a un ELF `stripped` por las referencias de una cadena.
- Cómo distinguir animaciones y pausas de la lógica que realmente decide el resultado.
- Cómo reconocer un arreglo de cinco enteros en el pseudocódigo.
- Cómo simplificar aritmética, desplazamientos y XOR.
- Cómo recuperar una contraseña comparada carácter por carácter.
- Que una cadena con forma de flag puede ser solo un fragmento.
- Cómo seguir una rutina de decodificación compuesta por XOR y ROT13.
- Por qué hay que verificar el resultado recorriendo el flujo completo del programa.

## Checklist reutilizable

Ante un crackme con menús o fases visuales:

1. ejecutar una vez para entender qué entradas solicita;
2. buscar los textos de entrada, éxito y error;
3. usar sus xrefs para encontrar la función principal;
4. localizar dónde se guardan todas las decisiones;
5. seguir la llamada que valida el arreglo completo;
6. simplificar constantes y operaciones una por una;
7. revisar si existe una segunda validación después del primer éxito;
8. seguir las funciones que imprimen o transforman la flag;
9. ejecutar el programa con los valores recuperados.
