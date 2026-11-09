# Write-up: Analista de tráfico
## Reto
- Plataforma: CTF SCESI
- Nombre: Analista de tráfico
- Categoría: Forensics
- Puntos: 120
- Archivo/objetivo: `4847a62673c33ec1bb86fe9e9a910e89`
- Estado: `resuelto`
## Reconocimiento
```bash
file 4847a62673c33ec1bb86fe9e9a910e89
capinfos 4847a62673c33ec1bb86fe9e9a910e89
tshark -r 4847a62673c33ec1bb86fe9e9a910e89 -q -z io,phs
```
**Hallazgo:** Aunque el reto se distribuía comprimido, el archivo recibido ya es un PCAPNG válido, no un GZIP. Contiene 109.180 paquetes capturados durante 54,919 segundos sobre el loopback de Windows. Hay 7.279 solicitudes `POST /flag` generadas por `python-requests/2.34.2` contra `172.19.16.1:80`.
## Hipótesis
- [x] Las respuestas HTTP repetidas son ruido y no contienen fragmentos diferentes.
- [x] Cada solicitud transporta un carácter candidato en el parámetro `caracter` y relleno aleatorio en `cuerpo`.
- [x] El formato conocido `umssctf{...}` proporciona `{` y `}` como referencias para localizar paquetes que necesariamente pertenecen a la flag.
- [x] Al comparar esos paquetes de referencia se puede buscar una propiedad estructural común que separe los caracteres reales del tráfico señuelo.
## Descubrimiento manual en Wireshark
1. Abrir la captura y entrar en `Statistics → Protocol Hierarchy`. HTTP destaca por el gran número de paquetes, así que se empieza con el filtro `http.request`.
2. Seleccionar cualquiera de los POST y expandir `Hypertext Transfer Protocol` y los datos del cuerpo. Todos tienen la forma `caracter=6&cuerpo=KikTGz...`: `caracter` guarda un solo símbolo y `cuerpo` parece relleno alfanumérico aleatorio.
3. No conviene leer 7.279 cuerpos. Ya se conoce el formato `umssctf{...}`, así que la llave `{` sirve como referencia segura: ese carácter necesariamente pertenece a la flag. Se busca su representación URL-encoded con el filtro `http.file_data contains "%7B"`.
4. Solo aparece el frame `12244`. Al abrirlo y revisar su cabecera HTTP se observa por primera vez `Content-Length: 1102`. El número no fue calculado ni adivinado: se leyó directamente del paquete que transportaba `{`. La captura no explica por qué el autor eligió específicamente `1102`; parece ser una constante arbitraria.
5. Para comprobar que `1102` no es una coincidencia, se buscan el cierre y los separadores con `http.file_data contains "%7D"` y `http.file_data contains "caracter=_"`. El cierre y los cuatro guiones bajos también tienen longitud `1102`.
6. En cualquiera de esos paquetes se puede hacer clic derecho sobre `Content-Length: 1102` y elegir `Apply as Column`. Después se aplica `http.request.method == "POST" && http.content_length == 1102`: quedan solamente 35 solicitudes, una cantidad coherente con una flag corta.
7. Los paquetes ya están en orden temporal. Leyendo únicamente el valor que sigue a `caracter=` se puede reconstruir a mano `umssctf%7B3l_l3ngth_3s_1mp0rt4nt3_27%7D`; al interpretar `%7B` como `{` y `%7D` como `}` aparece la flag.
**Comprobación visual del patrón:**
| Búsqueda | Frames | Content-Length |
| --- | --- | --- |
| `caracter=%7B` | `12244` | `1102` |
| `caracter=_` | `15231`, `37053`, `37953`, `86349` | `1102` |
| `caracter=%7D` | `104644` | `1102` |
## Comandos y resultados
Los siguientes comandos no descubren mágicamente el valor `1102`: automatizan el conteo y la copia después de encontrar manualmente el patrón anterior.
```bash
tshark -r 4847a62673c33ec1bb86fe9e9a910e89 \
  -Y 'http.request' -T fields -e http.request.method | sort | uniq -c
tshark -r 4847a62673c33ec1bb86fe9e9a910e89 \
  -Y 'http.response' -T fields -e http.response.code | sort | uniq -c
```
**Resultado útil:** Existen 7.279 POST y las 7.279 respuestas son `501 Unsupported method ('POST')`. El servidor rechaza el método después de recibir el cuerpo, por lo que el código de respuesta no elimina la información enviada por el cliente.
```bash
tshark -r 4847a62673c33ec1bb86fe9e9a910e89 \
  -Y 'http.request.method == "POST"' \
  -T fields -E separator='|' \
  -e frame.number -e http.content_length -e http.file_data \
  | rg 'caracter=(%7B|%7D|_)'
```
**Resultado útil:** En todo el tráfico solo aparecen una llave de apertura, una de cierre y cuatro `_`; las seis solicitudes tienen `Content-Length` igual a `1102`. Las llaves aparecen codificadas como `%7B` y `%7D`.
```bash
tshark -r 4847a62673c33ec1bb86fe9e9a910e89 \
  -Y 'http.request.method == "POST" && http.content_length == 1102' \
  -T fields -e http.file_data \
  | sed -E 's/^caracter=([^&]*)&cuerpo=.*/\1/' \
  | tr -d '\n' \
  | sed -e 's/%7B/{/g' -e 's/%7D/}/g'
```
**Resultado útil:** Hay 35 solicitudes con longitud `1102`. Sus caracteres, concatenados en el orden de captura y decodificados como URL, producen `umssctf{3l_l3ngth_3s_1mp0rt4nt3_27}`.
## Cadena de resolución
`PCAPNG → inventario de protocolos → 7279 POST /flag → separar caracter/cuerpo → detectar caracteres estructurales raros → longitud común 1102 → filtrar sin reordenar → URL-decode → flag`
## Flag
`umssctf{3l_l3ngth_3s_1mp0rt4nt3_27}`
## Qué debo recordar para otro CTF
- Indicador que me hizo reconocer la técnica: muchas solicitudes casi idénticas, cuerpos aleatorios de tamaños variables y un parámetro que contiene exactamente un carácter.
- Herramienta que funcionó: TShark para extraer campos HTTP masivamente; revisar miles de paquetes a mano en Wireshark no era necesario.
- Comando mínimo: filtrar `http.content_length == 1102`, extraer `caracter`, conservar el orden y decodificar `%7B/%7D`.
- Error/trampa: concentrarse en los `501`, seguir un único stream o intentar leer manualmente los cuerpos largos. Cada solicitud usa una conexión distinta y la respuesta repetida no contiene la señal.
- `Content-Length` mide los bytes codificados del formulario completo. Por eso las llaves ocupan tres caracteres (`%7B`, `%7D`) y el relleno `cuerpo` se acorta dos bytes para mantener el total en `1102`.
- Nunca ordenar ni deduplicar antes de reconstruir: el orden cronológico de las solicitudes forma el mensaje.
- Las llaves fueron la referencia disponible en este reto, no una regla universal. Si el formato esperado no contiene caracteres especiales, se deben buscar otras anclas conocidas —prefijo de la competencia, magic bytes, texto esperado o diferencias de cabeceras— o agrupar todas las longitudes y comparar las reconstrucciones sin asumir previamente cuál es correcta.
## Candidato para el arsenal forense
- Extraer solicitudes HTTP a una tabla con número de frame, tiempo, stream, método, URI, código de respuesta, Content-Length, claves y valores del formulario.
- Agrupar el campo candidato por propiedades estables como longitud, URI, encabezados, intervalo temporal o respuesta, manteniendo siempre el orden original dentro de cada grupo.
- Aplicar URL decoding y puntuar cada reconstrucción por imprimibilidad, llaves balanceadas y patrones de flag, sin asumir un prefijo concreto.
- Mostrar por qué se seleccionó un grupo y cuántos paquetes lo forman, evitando un solucionador mágico sin evidencia.
