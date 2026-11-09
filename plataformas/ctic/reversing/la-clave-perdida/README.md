# Write-up: La Clave Perdida

## Reto

- Plataforma: CTIC / AGETIC
- Nombre: La Clave Perdida
- Categoría: Reversing
- Puntos: 140
- Archivo: `la-clave-perdida`
- Estado: `resuelto y verificado localmente`

> ¿Serás capaz de desbloquear sus secretos?

## Resumen

El ejecutable construye la contraseña correcta carácter por carácter dentro de una función llamada `build_correct_password()`. Como el binario no está *stripped*, ese nombre se conserva y permite llegar rápidamente a la validación desde Binary Ninja.

Hay dos formas educativas de resolverlo:

1. recuperar la contraseña leyendo los valores ASCII usados por `build_correct_password()`;
2. parchear el salto condicional que conduce al mensaje de error.

La primera explica qué espera realmente el programa. La segunda coincide con la pista del enunciado sobre “saltar algo en el código”.

## Flag

```text
citc{si_tu_lo_encontraste_magic_zsa}
```

## Herramientas utilizadas

- `file`: identificar formato, arquitectura y características generales.
- `strings`: localizar mensajes y posibles prefijos de flag.
- `nm -C`: listar símbolos C++ con nombres legibles.
- Binary Ninja: seguir `main`, estudiar el pseudocódigo y parchear el salto.
- `objdump`: comprobación alternativa del ensamblador.

## Reconocimiento inicial

Primero se identifica el archivo:

```bash
file la-clave-perdida
```

Resultado relevante:

```text
ELF 64-bit LSB pie executable, x86-64, dynamically linked, not stripped
```

Esto significa:

- `ELF`: ejecutable de Linux;
- `64-bit` y `x86-64`: arquitectura que debe seleccionar el analizador;
- `PIE`: sus direcciones pueden cambiar al ejecutarse;
- `not stripped`: conserva símbolos como nombres de funciones, una ventaja enorme para el análisis estático.

Después se revisan las cadenas:

```bash
strings -a -n 4 la-clave-perdida | less
```

Entre ellas aparecen:

```text
Ingrese la clave:
Comprobando la clave....
citc{
Error, la clave no es la correcta
```

Esto confirma que es un *crackme*: recibe una clave, la compara y toma una rama de éxito o error.

## Resolución con Binary Ninja

### 1. Abrir `main`

Se abre el binario y se espera a que termine el análisis. En la lista de funciones se selecciona `main` y se cambia a HLIL para ver una representación parecida a C.

La lógica importante es aproximadamente:

```cpp
std::string entrada;
std::string correcta = build_correct_password();

std::cin >> entrada;

if (entrada == correcta) {
    std::cout << "citc{" << correcta << "}";
} else {
    std::cout << "Error, la clave no es la correcta";
}
```

Aquí ya sabemos que la flag se imprime usando la cadena devuelta por `build_correct_password()`. Por eso esa función es el siguiente objetivo.

### 2. Entrar a `build_correct_password()`

El binario conserva el símbolo:

```text
build_correct_password[abi:cxx11]()
```

La función almacena números como estos:

```text
0x73, 0x69, 0x5f, 0x74, 0x75, 0x5f, ...
```

Después recorre 30 posiciones y añade el byte bajo de cada número a un `std::string`. Los valores son códigos ASCII:

```text
0x73 = s
0x69 = i
0x5f = _
0x74 = t
0x75 = u
```

Continuando con todos los valores se reconstruye:

```text
si_tu_lo_encontraste_magic_zsa
```

No existe una operación criptográfica: la clave está dividida en constantes numéricas para que no aparezca completa en `strings`.

### 3. Verificar la contraseña

Si el archivo perdió el permiso de ejecución al descargarse:

```bash
chmod +x la-clave-perdida
```

Se ejecuta:

```bash
./la-clave-perdida
```

Entrada:

```text
si_tu_lo_encontraste_magic_zsa
```

Salida:

```text
Ingrese la clave:
Comprobando la clave....
citc{si_tu_lo_encontraste_magic_zsa}
```

## Ruta alternativa: parchear el salto

Después de comparar ambas cadenas, `main` comprueba el resultado y ejecuta un salto condicional hacia la rama de error:

```asm
test al, al
je   rama_de_error
```

`al` contiene el resultado booleano de la comparación:

- `al = 1`: las cadenas son iguales y el programa continúa hacia el éxito;
- `al = 0`: `test` activa la bandera de cero y `je` salta al error.

En la vista de ensamblador de Binary Ninja se puede seleccionar ese `je` y usar:

```text
Patch → Never Branch
```

Al impedir el salto, el flujo cae siempre en la rama que imprime la flag, incluso con una entrada incorrecta. Esta es la técnica clásica de los antiguos *crackmes*: modificar una decisión del programa sin conocer inicialmente la clave.

Parchear resuelve el control de flujo, pero analizar `build_correct_password()` enseña de dónde proviene realmente la respuesta. Conviene entender ambas rutas.

## Qué enseña el reto

- Empezar por `file` y `strings` reduce mucho el espacio de búsqueda.
- Un binario `not stripped` conserva nombres que pueden revelar la intención de cada función.
- Las cadenas ocultas no siempre están cifradas; pueden construirse carácter por carácter.
- Los números hexadecimales entre `0x20` y `0x7e` suelen representar ASCII imprimible.
- Seguir la rama de éxito y la rama de error permite localizar la validación.
- Un salto condicional puede invertirse, forzarse o eliminarse para estudiar el comportamiento del programa.

## Checklist reutilizable

```bash
file reto
strings -a -n 4 reto | less
nm -C reto | less
```

En Binary Ninja:

1. abrir `main`;
2. buscar la entrada del usuario;
3. localizar la comparación;
4. seguir las funciones que generan la clave esperada;
5. revisar las ramas de éxito y error;
6. verificar la hipótesis ejecutando el programa;
7. solo después, practicar el parcheo del salto.
