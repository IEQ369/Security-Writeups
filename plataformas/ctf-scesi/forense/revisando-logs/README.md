# Write-up: Revisando logs
## Reto
- Plataforma: CTF SCESI
- Nombre: Revisando logs
- Categoría: Forensics
- Puntos: 100
- Archivo/objetivo: `e117d99987384f67f2789e9d7c8c7f91.zip`
- Estado: `resuelto`
## Reconocimiento
```bash
file e117d99987384f67f2789e9d7c8c7f91.zip
unzip -l e117d99987384f67f2789e9d7c8c7f91.zip
unzip -t e117d99987384f67f2789e9d7c8c7f91.zip
```
**Hallazgo:** Es un ZIP válido. Contiene un archivo `reto` de 94 bytes y un `.bash_history` de 25.284 bytes y 1.175 líneas. `reto` solamente repite la descripción; la evidencia útil está en el historial.
## Hipótesis
- [x] La bandera fue construida mediante comandos registrados en `.bash_history`.
- [x] Sus fragmentos se escribieron en un archivo que cambió varias veces de nombre.
- [x] El orden cronológico de los comandos permite reconstruir el contenido.
## Descubrimiento humano, paso a paso
1. Se abre `.bash_history` sin ordenarlo. `more` conserva su orden y sirve para recorrerlo, pero `less` resulta más cómodo porque permite buscar y retroceder.
2. Leer las 1.175 líneas una por una no es necesario. Como el formato conocido de la competencia empieza por `umssctf{`, se busca ese texto. Aparece en la línea 20:
   ```bash
   echo "umssctf{oculto" >> pass
   ```
   Esto proporciona el primer fragmento y, sobre todo, una nueva pista: el archivo se llama `pass`.
3. Se busca `pass` en el historial. En la línea 61 fue renombrado a `teto`:
   ```bash
   mv pass teto
   ```
4. Ahora se busca `teto`. En la línea 95 se añadió `_o_no_re`, y en la 208 cambió de nombre a `pepa`:
   ```bash
   sed -i '$a _o_no_re' teto
   mv teto pepa
   ```
5. Se repite el seguimiento con `pepa`. En la línea 291 se agregó `visando_`; después fue renombrado a `casa`:
   ```bash
   python3 -c 'open("pepa","a").write("visando_")'
   mv pepa casa
   ```
6. Al buscar `casa`, aparece el fragmento final y el último cambio de nombre:
   ```bash
   printf "comandos}" >> casa
   mv casa flag
   ```
7. Los fragmentos, conservados en el orden de ejecución, son:
   ```text
   umssctf{oculto
   _o_no_re
   visando_
   comandos}
   ```
   Al unirlos se obtiene una bandera con estructura y frase coherentes.
## Búsqueda eficiente
Una vez descubiertos progresivamente los nombres, se puede reducir todo el historial a la cadena de operaciones relevante:
```bash
rg -a -n '\b(pass|teto|pepa|casa|flag)\b' .bash_history
```
El resultado es:
```text
20:echo "umssctf{oculto" >> pass
61:mv pass teto
95:sed -i '$a _o_no_re' teto
208:mv teto pepa
291:python3 -c 'open("pepa","a").write("visando_")'
383:mv pepa casa
500:printf "comandos}" >> casa
572:mv casa flag
```
La opción `-a` obliga a tratar el historial como texto. Es necesaria aquí porque una línea contiene muchos bytes NUL y algunas herramientas clasifican por ello el archivo como datos binarios.
## Por qué `sort | uniq -u` es peligroso
- `cat archivo | sort` es equivalente a `sort archivo`; `cat` no aporta nada en ese caso.
- `sort` destruye el orden cronológico. En un historial, el orden permite saber qué ocurrió antes de cada escritura o cambio de nombre.
- Después de ordenar, `uniq -u` elimina todas las líneas repetidas. Un comando repetido puede ser una pista, una parte de una secuencia o una acción ejecutada varias veces.
- En este reto los ocho comandos relevantes son únicos, por lo que sobrevivieron por casualidad. El método no es seguro para otra evidencia.

Si se quiere quitar ruido para explorar, debe conservarse siempre intacto el historial original y mantener los números de línea. Cualquier vista ordenada o deduplicada debe ser solamente auxiliar, nunca la base de la reconstrucción.
## Cadena de resolución
`ZIP → .bash_history → buscar prefijo conocido → seguir el nombre del archivo → identificar escrituras y renombrados → ordenar por número de línea → unir fragmentos → flag`
## Flag
`umssctf{oculto_o_no_revisando_comandos}`
## Qué debo recordar para otro CTF
- Indicador que hizo reconocer la técnica: una escritura parcial del prefijo de la bandera hacia un archivo.
- El nombre del archivo es una pista de seguimiento. Cada `mv` crea una nueva identidad que debe rastrearse en las líneas posteriores.
- No basta con buscar `echo`: el reto mezcla `echo`, `sed`, Python y `printf` para modificar el mismo contenido.
- Los saltos de línea introducidos por `echo` y `sed` forman parte del archivo físico, pero los fragmentos de una bandera pueden requerir concatenación lógica.
- Nunca ordenar ni deduplicar la evidencia antes de reconstruir una secuencia.
- Un historial puede contener bytes NUL u otros caracteres de control; una herramienta que diga «archivo binario» no demuestra que no haya comandos legibles.
## Candidato para el arsenal forense
- Módulo de historiales de shell que conserve línea, orden y bytes originales.
- Detectar comandos que crean, anexan, editan, copian, mueven o renombran archivos.
- Construir la genealogía de cada archivo a través de `mv` y `cp`, y mostrarla como `pass → teto → pepa → casa → flag`.
- Reconocer escrituras comunes mediante `echo`, `printf`, `sed`, `tee`, redirecciones y pequeños comandos de Python o Perl, sin ejecutar comandos no confiables.
- Extraer candidatos en el orden original y explicar qué líneas aportaron cada fragmento.
- Buscar tanto el texto literal como representaciones *leet* puntuadas (`3/e`, `4/a`, `0/o`, `5/s`, `7/t`, y alternativas para `1/i/l`) sin alterar ni ocultar la evidencia original.
- Advertir si el usuario intenta ordenar o deduplicar antes de una reconstrucción.
