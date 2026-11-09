# Write-up: Nimrod, la computadora malvada

## Reto

- Plataforma: CTIC / AGETIC
- Nombre: Nimrod, la computadora malvada
- Categoría: Reversing
- Puntos: 390
- Archivo: `Nimrod`
- Estado: `resuelto y verificado localmente con GDB`

> Nimrod es una computadora conocida por hacer trampa en los juegos. ¿Puedes ganarle?

Formato indicado:

```text
cidsi{bandera_en_md5}
```

## Resumen

El programa presenta ocho cajas. La caja ganadora es la número `7`, pero Nimrod hace trampa: después de reconocer que ganamos consulta una variable global llamada `boleto_entrada`. Su valor es `2`, aunque la rama que entrega la recompensa exige que sea `1`.

La propia salida ofrece la pista:

```text
Si ganar quieres.
Debuggear debes.
-Yoda
```

Se coloca un breakpoint justo antes de la comparación, se cambia `boleto_entrada` de `2` a `1` y se continúa. El programa entra entonces en `dar_recompensa` y muestra la flag verdadera.

## Flag

```text
cidsi{896aeb7e677433b7aed20756096d8d61}
```

## Herramientas utilizadas

- `file`: identificar el ejecutable.
- `strings`: localizar mensajes y detectar una flag falsa.
- `nm`: recuperar nombres conservados por el binario.
- Binary Ninja: seguir la selección de la caja y la condición de la recompensa.
- GDB: observar y modificar `boleto_entrada` durante la ejecución.

No fue necesario automatizar el juego ni aplicar fuerza bruta.

## Reconocimiento inicial

```bash
file Nimrod
```

Resultado:

```text
ELF 64-bit LSB executable, x86-64, statically linked, not stripped
```

Lo importante es:

- `ELF 64-bit`: ejecutable Linux x86-64;
- `statically linked`: incluye el runtime dentro del archivo y por eso contiene muchas funciones;
- `not stripped`: conserva símbolos y algunos nombres originales.

En este caso fue escrito en **Nim**. Se reconoce por símbolos como:

```text
NimMain
NimMainModule
dar_recompensa__juego_u87
boleto_entrada__juego_u104
```

El runtime de Nim hace que `NimMainModule` sea grande y algo ruidoso, pero los nombres conservados ayudan mucho.

## La flag falsa

Al revisar cadenas:

```bash
strings -a -n 4 Nimrod | less
```

aparece:

```text
cidsi{shame_on_you!_fake_flag}
```

No debe aceptarse automáticamente por dos motivos:

1. contiene literalmente `fake_flag`;
2. el enunciado exige 32 caracteres hexadecimales, propios de un MD5, dentro de `cidsi{...}`.

`shame_on_you!_fake_flag` no cumple ese formato. La descripción del reto sirve aquí como regla de validación y evita el falso positivo de `strings`.

## Ejecutar el juego normalmente

Si falta el permiso de ejecución:

```bash
chmod u+x Nimrod
./Nimrod
```

El programa pregunta si queremos jugar y muestra ocho cajas. Al elegir la caja `7` sucede esto:

```text
¡¡¡Ganaste!!!
Espera, ¿qué?
¡¡¡¡No puedes ganarle a la computadora!!!!

Procediendo a hacer chicanerías y otras cosas turbias...
Mmmmmmm haré de cuenta que no escogiste esa caja...

Si ganar quieres.
Debuggear debes.
-Yoda
```

Esto permite deducir dos cosas antes de abrir el desensamblador:

- la caja correcta sí es la `7`;
- el objetivo no es adivinar otra caja, sino alterar el estado del programa con un depurador.

## Encontrar la lógica en Binary Ninja

Como los símbolos existen:

1. abrir `Nimrod` en Binary Ninja;
2. buscar la función `NimMainModule`;
3. buscar dentro de ella la constante `7` o las referencias a los textos `¡¡¡Ganaste!!!`;
4. seguir el bloque que se ejecuta después de leer el número.

La primera decisión importante equivale a:

```c
if (user_number != 7)
    ir_a_caja_perdedora();
```

En ensamblador:

```asm
cmp rax, 7
jne rama_perdedora
```

`rax` contiene el número introducido. Si no es `7`, salta al resultado perdedor. Si es `7`, continúa por el bloque que inicialmente muestra `¡¡¡Ganaste!!!`.

Más adelante aparece la verdadera trampa:

```asm
cmp qword [boleto_entrada__juego_u104], 1
je  dar_recompensa
```

En este binario concreto las direcciones son:

```text
comparación:     0x415c29
boleto_entrada:  0x41e198
rama premio:     0x415f00
```

La variable vale `2` al llegar a esa comparación. Por tanto, la condición `boleto_entrada == 1` es falsa y el programa ejecuta la secuencia donde Nimrod hace trampa.

## Por qué las direcciones pueden usarse directamente

El encabezado ELF indica:

```text
Type: EXEC
```

Eso significa que no es PIE. Sus direcciones de código son fijas, por lo que `0x415c29` puede utilizarse directamente en GDB para este archivo.

En un ejecutable PIE las direcciones cambiarían al cargarlo y sería preferible usar símbolos, offsets o desactivar temporalmente ASLR dentro del depurador.

## Resolverlo manualmente con GDB

Se abre el programa:

```bash
gdb ./Nimrod
```

Dentro de GDB se coloca un breakpoint antes de comprobar el boleto:

```gdb
break *0x415c29
run
```

Se interactúa normalmente con el programa:

```text
y
7
```

Después de mostrar `¡¡¡Ganaste!!!`, GDB se detiene antes de esta instrucción:

```asm
cmp qword [0x41e198], 1
```

Se inspecciona el valor actual:

```gdb
x/gd 0x41e198
```

- `x`: examinar memoria;
- `/g`: leer ocho bytes, un *giant word* de 64 bits;
- `d`: mostrarlo en decimal.

Resultado esperado:

```text
0x41e198 <boleto_entrada__juego_u104>: 2
```

Ahora se cambia a `1`:

```gdb
set {long long}0x41e198 = 1
```

Se comprueba:

```gdb
x/gd 0x41e198
```

Debe mostrar:

```text
0x41e198 <boleto_entrada__juego_u104>: 1
```

Finalmente:

```gdb
continue
```

Salida:

```text
¿Pero qué haces aquí?
Okay eres más tiburón de lo que pensaba
cidsi{896aeb7e677433b7aed20756096d8d61}
```

## Qué modificamos realmente

No cambiamos la comparación ni saltamos directamente a la función ganadora. Modificamos el dato que la comparación consulta:

```text
antes: boleto_entrada = 2 → 2 == 1 es falso → Nimrod hace trampa
después: boleto_entrada = 1 → 1 == 1 es verdadero → recompensa
```

Este detalle es importante para aprender depuración: un debugger no sirve únicamente para avanzar instrucción por instrucción. También permite observar registros y memoria, comprobar una hipótesis y cambiar temporalmente el estado del proceso.

El archivo original no queda modificado. Al cerrar GDB y ejecutar de nuevo, `boleto_entrada` vuelve a valer `2`.

## Alternativa: cambiar el salto en Binary Ninja

También sería posible parchear el salto condicional situado después de la comparación:

```asm
je 0x415f00
```

En Binary Ninja se puede seleccionar ese salto y utilizar **Patch → Always Branch**. Así la ejecución iría siempre hacia la recompensa.

Sin embargo, la solución con GDB encaja mejor con la pista explícita del reto y enseña la diferencia entre:

- modificar temporalmente memoria durante una ejecución;
- modificar permanentemente las instrucciones del archivo.

## La función de recompensa

La rama correcta llama a:

```text
dar_recompensa__juego_u87
```

Esta función prepara una serie de valores y llama a otra función de nombre extraño:

```text
ABACADSFAFZ9877__juego_u5
```

La segunda función transforma los valores mediante restas y XOR antes de imprimirlos. Por eso la flag verdadera no aparece completa mediante `strings`.

No es necesario reconstruir manualmente esa codificación: el reto pide vencer la condición imposible mediante depuración, y al alcanzar legítimamente la rama de recompensa el propio programa realiza el descifrado.

## Qué enseña el reto

- Cómo reconocer un ejecutable compilado en Nim.
- La ventaja de que un binario sea `not stripped`.
- Cómo usar el formato esperado para descartar una flag falsa.
- Cómo encontrar una selección correcta a partir de `cmp` y `jne`.
- Cómo distinguir “ganar el juego” de superar una segunda condición oculta.
- Qué es un breakpoint y por qué se coloca antes de la comparación.
- Cómo leer y modificar una variable global con GDB.
- Que modificar memoria durante la depuración no altera el ejecutable original.
- La diferencia entre depurar una condición y parchear un salto.

## Checklist reutilizable

Cuando un juego parece impedir una victoria válida:

1. probar el flujo normal y anotar exactamente cuándo cambia el resultado;
2. buscar las cadenas de victoria y derrota;
3. revisar símbolos con `nm` antes de asumir que el binario está stripped;
4. localizar la comparación de la opción elegida;
5. seguir la rama ganadora y buscar condiciones adicionales;
6. observar los valores reales con un breakpoint;
7. cambiar solo el dato o registro necesario;
8. continuar hasta que el propio programa muestre la recompensa;
9. validar la flag contra el formato indicado en el enunciado;
10. documentar por qué la ejecución normal nunca podía cumplir la condición.
