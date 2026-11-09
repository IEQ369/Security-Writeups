# Write-up: Command & Control - Level 2

## Reto

**Plataforma:** Root-Me\
**Categoría:** Forense / Análisis de memoria\
**Reto:** Command & Control - Level 2\
**Puntuación:** 15 puntos\
**Objetivo:** recuperar el **hostname de la estación de trabajo** a partir de un volcado de memoria.
**Estado:** `resuelto`

El reto proporciona un dump de memoria de una máquina Windows. La bandera de validación no tiene el formato clásico `flag{...}`: en este caso, **la propia bandera es el nombre del equipo**.

## 1. Preparación de la evidencia

Después de descargar y descomprimir el fichero del reto obtenemos el volcado:

```bash
ch2.dmp
```

## 2. Identificación del sistema operativo

Para analizar el dump podemos utilizar **Volatility**.

Con Volatility 2:

```bash
volatility -f ch2.dmp imageinfo
```

El análisis identifica la imagen como correspondiente a una máquina **Windows 7 de 32 bits**, proponiendo perfiles como:

```text
Win7SP1x86_23418
Win7SP0x86
Win7SP1x86
```

Los análisis públicos de esta misma imagen muestran además una fecha de adquisición cercana al `2013-01-12 16:59 UTC`.

Por ejemplo, podemos utilizar:

```bash
volatility -f ch2.dmp --profile=Win7SP0x86 pslist
```

Con Volatility 3 no es necesario seleccionar manualmente un perfil:

```bash
python3 vol.py -f ch2.dmp windows.info
```

---

## 3. ¿Dónde guarda Windows el hostname?

El objetivo es obtener el nombre de la estación de trabajo.

Una de las fuentes más fiables para encontrarlo es el **Registro de Windows**, concretamente el hive `SYSTEM`.

El nombre del equipo se encuentra normalmente en una ruta como:

```text
HKLM\SYSTEM\ControlSet001\Control\ComputerName\ComputerName
```

El valor que nos interesa se llama:

```text
ComputerName
```

Por lo tanto, el siguiente paso es recuperar el hive `SYSTEM` presente en memoria.

---

## 4. Enumeración de los Registry Hives

### Volatility 2

Ejecutamos:

```bash
volatility -f ch2.dmp \
  --profile=Win7SP0x86 \
  hivelist
```

Entre los resultados aparece:

```text
0x8b21c008  0x039ef008  \REGISTRY\MACHINE\SYSTEM
```

Ese hive contiene la configuración del sistema.

Los análisis disponibles públicamente de `ch2.dmp` muestran precisamente `0x8b21c008` asociado a `\REGISTRY\MACHINE\SYSTEM`.

### Volatility 3

El equivalente es:

```bash
python3 vol.py -f ch2.dmp windows.registry.hivelist
```

Buscamos una entrada semejante a:

```text
\REGISTRY\MACHINE\SYSTEM
```

Volatility 3 incorpora un plugin específico para enumerar hives y otro denominado `PrintKey` para listar claves y valores del Registro.

---

## 5. Recuperación del ComputerName

Una vez localizado el hive `SYSTEM`, podemos consultar directamente la clave.

### Volatility 2

```bash
volatility -f ch2.dmp \
  --profile=Win7SP0x86 \
  printkey \
  -K "ControlSet001\Control\ComputerName\ComputerName"
```

`printkey` permite mostrar las subclaves, valores y datos almacenados en una clave concreta del Registro.

La información relevante obtenida es equivalente a:

```text
Registry: \REGISTRY\MACHINE\SYSTEM

Key name: ComputerName

Values:

REG_SZ        ComputerName        WIN-ETSA91RKCFP
```

Por tanto, ya hemos recuperado el hostname.

```text
WIN-ETSA91RKCFP
```

---

## 6. Solución con Volatility 3

También podemos resolverlo utilizando únicamente Volatility 3.

Primero:

```bash
python3 vol.py -f ch2.dmp windows.registry.hivelist
```

Localizamos:

```text
\Registry\Machine\System
```

En esta imagen el hive se ha documentado con el offset:

```text
0x8b21c008
```

Después consultamos la clave:

```bash
python3 vol.py \
  -f ch2.dmp \
  windows.registry.printkey \
  --offset 0x8b21c008 \
  --key 'ControlSet001\Control\ComputerName\ComputerName'
```

El valor recuperado es:

```text
ComputerName    WIN-ETSA91RKCFP
```

Este procedimiento con Volatility 3 y el resultado `WIN-ETSA91RKCFP` han sido reproducidos sobre el dump del desafío.

---

## 7. Método alternativo: variables de entorno

Existe una segunda forma interesante de recuperar el hostname.

Windows establece normalmente la variable de entorno:

```text
COMPUTERNAME
```

Volatility permite inspeccionar las variables de entorno que estaban cargadas en los procesos.

Con Volatility 2:

```bash
volatility -f ch2.dmp \
  --profile=Win7SP0x86 \
  envars | grep COMPUTERNAME
```

También podríamos utilizar:

```bash
volatility -f ch2.dmp \
  --profile=Win7SP0x86 \
  envars | grep -i computer
```

Entre los resultados deberíamos encontrar algo similar a:

```text
COMPUTERNAME    WIN-ETSA91RKCFP
```

Esta técnica también ha sido documentada específicamente para este desafío.

En Volatility 3 existe igualmente el plugin `windows.envars`, cuyo propósito es mostrar las variables de entorno de los procesos.

Podemos ejecutar:

```bash
python3 vol.py -f ch2.dmp windows.envars
```

y filtrar:

```bash
python3 vol.py -f ch2.dmp windows.envars | grep COMPUTERNAME
```

---

## 8. Método rápido con strings

Para este reto concreto también es posible encontrar el hostname sin analizar formalmente el Registro.

Podemos buscar cadenas relacionadas con `COMPUTERNAME` directamente dentro de la memoria:

```bash
strings -a ch2.dmp | grep -i COMPUTERNAME
```

También:

```bash
strings -a ch2.dmp | grep -i "WIN-"
```

Como un dump de memoria conserva fragmentos de las variables de entorno y otras estructuras de Windows, es posible que aparezca directamente:

```text
COMPUTERNAME=WIN-ETSA91RKCFP
```

Este método puede resolver el desafío rápidamente, aunque desde el punto de vista forense es preferible utilizar el Registro o las variables de entorno porque aportan **contexto estructurado** sobre el origen de la información.

---

## 9. Automatización

La esencia del reto se puede automatizar buscando primero la variable `COMPUTERNAME`:

```bash
#!/bin/bash

DUMP="$1"

if [ -z "$DUMP" ]; then
    echo "Uso: $0 <memory.dmp>"
    exit 1
fi

echo "[+] Buscando COMPUTERNAME..."

strings -a "$DUMP" |
grep -i -A2 -B2 "COMPUTERNAME" |
head -30
```

Uso:

```bash
chmod +x hostname_memory.sh
./hostname_memory.sh ch2.dmp
```

Para un análisis más riguroso, el script de forense debería intentar posteriormente:

```text
1. Identificar si la evidencia es un dump de Windows
2. Ejecutar windows.registry.hivelist
3. Localizar \REGISTRY\MACHINE\SYSTEM
4. Consultar ControlSet001\Control\ComputerName\ComputerName
5. Consultar windows.envars
6. Buscar COMPUTERNAME
7. Comparar ambos resultados
```

Si Registro y variables de entorno devuelven el mismo hostname, la confianza en el resultado es muy alta.

---

## 10. Bandera

El hostname recuperado es:

```text
WIN-ETSA91RKCFP
```

Por lo tanto, la bandera que debe enviarse en Root-Me es:

```text
WIN-ETSA91RKCFP
```

## Conclusión

Este desafío introduce un concepto fundamental del análisis forense de memoria: **un volcado RAM no contiene solamente procesos**, sino también estructuras del sistema operativo, variables de entorno, fragmentos del Registro, credenciales, conexiones, archivos y numerosos artefactos que estaban presentes cuando se adquirió la memoria.

En este caso, Root-Me nos solicita recuperar el hostname de la máquina. El método forense más apropiado consiste en localizar el hive `SYSTEM` y consultar:

```text
ControlSet001
└── Control
    └── ComputerName
        └── ComputerName
```

De ahí se obtiene:

```text
ComputerName = WIN-ETSA91RKCFP
```

También es posible corroborar el hallazgo mediante la variable de entorno `COMPUTERNAME`, convirtiendo el resultado en una evidencia consistente obtenida por dos vías distintas.

## Qué debo recordar para otro CTF

- Un volcado de memoria contiene estructuras del sistema, no solamente procesos.
- Corroborar un dato por dos artefactos diferentes aumenta la confianza.
