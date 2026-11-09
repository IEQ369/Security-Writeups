# Write-up: El Laberinto de Knossos

## Reto

- Plataforma: CTIC / AGETIC
- Nombre: El Laberinto de Knossos
- Categoría: Reversing
- Puntos: 240
- Archivos: `main`, `laberinto.png`
- Estado: `resuelto y verificado localmente`

> Debes elegir un camino para que Asterión pueda salir del laberinto. El primer carácter indica una entrada (`A` a `L`) y los siguientes representan izquierda, derecha, abajo y arriba.

## Resumen

El ejecutable valida una entrada de exactamente 19 caracteres. Dentro de la función de comprobación construye una secuencia ofuscada y aplica XOR con `0x0b` a cada byte antes de compararla con la entrada.

Al deshacer el XOR se recupera:

```text
Jllllldrdlddrrrrrrd
```

La ruta empieza en `J`, atraviesa el laberinto y llega a la salida 20. Al introducirla, otra función descifra e imprime la flag.

## Flag

```text
citc{8b67f5c6e802cd5858833e997f509e81}
```

## Herramientas utilizadas

- `file`: identificar el ELF y comprobar si conserva símbolos.
- `strings`: localizar los mensajes de entrada, éxito y error.
- Binary Ninja: seguir referencias, recuperar el flujo real y estudiar la comparación.
- `objdump`: comprobar direcciones e instrucciones concretas.
- CyberChef: deshacer XOR sobre los bytes de la ruta, aunque también puede hacerse manualmente.
- La imagen del laberinto: verificar que la secuencia representa un camino real.

## Reconocimiento inicial

```bash
file main
```

Resultado relevante:

```text
ELF 64-bit LSB pie executable, x86-64, dynamically linked, stripped
```

La palabra `stripped` indica que el binario perdió nombres como `main` o `verify_path`. A diferencia de [La Clave Perdida](../la-clave-perdida/README.md), no podemos depender de una lista de funciones descriptivas.

Se revisan las cadenas:

```bash
strings -a -n 4 main | less
```

Se encuentran:

```text
Ingrese los movimientos:
Nos ha salvado estamos agradecidos
Incorrecto, ni modo, con mas calma el siguiente semestre
```

La flag no aparece porque está almacenada de forma codificada.

## Encontrar `main` en Binary Ninja

Una forma cómoda cuando faltan símbolos es:

1. abrir **Strings**;
2. seleccionar `Ingrese los movimientos:`;
3. abrir sus referencias cruzadas o **xrefs**;
4. saltar a la función que utiliza la cadena;
5. renombrarla manualmente como `main`.

La función hace aproximadamente esto:

```c
puts("Ingrese los movimientos:");
fgets(entrada, 30, stdin);
longitud = strcspn(entrada, "\n");
resultado = verificar_ruta(entrada, longitud);

if (resultado == 0)
    puts("Nos ha salvado estamos agradecidos");
else
    puts("Incorrecto...");
```

`strcspn(entrada, "\n")` localiza el salto de línea introducido al pulsar Enter. Por eso la longitud enviada al verificador no incluye ese salto.

La llamada inmediatamente anterior a la decisión de éxito/error conduce a la función importante. Conviene renombrarla como `verificar_ruta`.

## La longitud exigida

Dentro del verificador aparece:

```asm
cmp rsi, 0x13
jne fallo
```

En la convención de llamadas x86-64 de Linux:

- `rdi` recibe el primer argumento: la dirección de la entrada;
- `rsi` recibe el segundo: su longitud.

`0x13` hexadecimal equivale a 19 decimal. Por tanto, la entrada tiene:

- una letra inicial (`A` a `L`);
- 18 movimientos.

## Reconocer la trampa de anti-desensamblado

El binario contiene repetidamente construcciones como:

```asm
je  destino
jne destino
; bytes basura que parecen instrucciones
destino:
```

`je` significa “salta si son iguales” y `jne`, “salta si no son iguales”. Las dos condiciones son opuestas y apuntan al mismo destino: necesariamente uno de los dos saltos se ejecutará.

Por tanto, los bytes colocados entre esos saltos y el destino nunca se ejecutan. Su propósito es confundir desensambladores lineales para que muestren llamadas imposibles o instrucciones rotas.

En el grafo de Binary Ninja se debe seguir el bloque alcanzable y no interpretar esos bytes muertos como lógica verdadera.

## Recuperar la ruta ofuscada

Siguiendo únicamente el flujo real, la función construye estos 19 bytes en la pila:

```text
41 67 67 67 67 67 6f 79 6f 67 6f 6f 79 79 79 79 79 79 6f
```

Después, para cada posición, hace:

```asm
mov  r10d, 0x0e
sub  r10,  0x03
xor  rbx,  r10
```

Primero calcula:

```text
0x0e - 0x03 = 0x0b
```

Así que cada byte guardado se transforma con XOR `0x0b` y luego se compara con el carácter introducido por el usuario.

Los primeros ejemplos son:

```text
0x41 XOR 0x0b = 0x4a = J
0x67 XOR 0x0b = 0x6c = l
0x6f XOR 0x0b = 0x64 = d
0x79 XOR 0x0b = 0x72 = r
```

Solo aparecen cuatro valores codificados, que producen exactamente los símbolos necesarios para el reto: `J`, `l`, `d` y `r`.

### Decodificarlo con CyberChef

Se pega la secuencia hexadecimal y se añaden:

1. **From Hex**;
2. **XOR**, clave `0B` interpretada como hexadecimal.

Resultado:

```text
Jllllldrdlddrrrrrrd
```

No fue necesario escribir un script: el ensamblador ya revela la operación y CyberChef permite comprobarla rápidamente.

## Verificar el camino en la imagen

La primera letra no es un movimiento; indica el punto de inicio:

```text
J | lllll | d | r | d | l | dd | rrrrrr | d
```

Interpretación:

1. iniciar en `J`;
2. avanzar cinco casillas a la izquierda;
3. bajar;
4. avanzar una casilla a la derecha;
5. bajar;
6. avanzar una casilla a la izquierda;
7. bajar dos casillas;
8. avanzar seis casillas a la derecha;
9. bajar hacia la salida 20.

La imagen funciona como confirmación visual. El camino no se dedujo adivinando entre todas las posibilidades: primero se recuperó la secuencia que valida el ejecutable y después se comprobó sobre el laberinto.

## Ejecutar y obtener la flag

```bash
./main
```

Entrada:

```text
Jllllldrdlddrrrrrrd
```

Salida:

```text
citc{8b67f5c6e802cd5858833e997f509e81}
Ingrese los movimientos:
Nos ha salvado estamos agradecidos
```

La flag aparece antes del mensaje debido a que una función interna la escribe directamente mediante la llamada al sistema `write`, mientras `puts` utiliza el búfer de la biblioteca estándar. Al redirigir la entrada, ambos mecanismos pueden vaciar su salida en un orden visual diferente.

## Cómo se oculta la flag

Cuando las 19 comparaciones son correctas, el verificador llama a otra función. Esta construye 40 bytes codificados, aplica nuevamente XOR `0x0b` a cada uno y los imprime uno por uno mediante `syscall`.

Eso explica por qué:

- la flag no aparece con `strings`;
- la ruta y la flag usan la misma constante XOR;
- el programa puede imprimir la flag antes del mensaje almacenado por `puts`.

## Qué enseña el reto

- Cómo encontrar una función desde las referencias de una cadena cuando el binario está `stripped`.
- Cómo reconocer argumentos en `rdi` y `rsi` bajo x86-64 Linux.
- Que `0x13` es 19 decimal, no trece.
- Cómo identificar XOR a partir de instrucciones simples.
- Cómo reconocer pares de saltos opuestos que siempre conducen al mismo destino.
- Por qué no todo lo que muestra un desensamblado lineal es código alcanzable.
- Cómo combinar reversing con una comprobación visual del problema original.

## Checklist reutilizable

Cuando un binario stripped solicita una entrada:

1. ejecutar `file` y `strings`;
2. abrir una cadena de éxito o error en Binary Ninja;
3. seguir sus xrefs hasta la función principal;
4. localizar la función que decide la rama;
5. reconocer comparaciones de longitud;
6. separar código alcanzable de bytes basura;
7. localizar constantes y operaciones reversibles como XOR;
8. reconstruir la entrada esperada;
9. verificarla contra el ejecutable.
