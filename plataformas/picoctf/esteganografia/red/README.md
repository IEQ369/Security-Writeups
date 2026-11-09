# Write-up: RED

## Reto

- Plataforma: PicoCTF
- Categoría: Forense / Esteganografía
- Estado: `resuelto`

### Información general
- Categoría: Forense (Easy)
- Archivo: `red.png`
- Autor: Shuailin Pan (LeConjuror)
- Descripción oficial:\

  "RED, RED, RED, RED. Download the image: red.png"

## Herramientas utilizadas
- `zsteg`: para detectar esteganografía en imágenes PNG.
- `base64`: para decodificar cadenas codificadas en base64.
- `CyberChef`: herramienta web para decodificar base64 y visualizar datos binarios.

## Proceso de resolución
1. **Análisis de la imagen con `zsteg`:**

```bash
zsteg -a red.png
```
* Esto analiza la imagen en busca de datos ocultos en los bits menos significativos (LSB) de los canales de color.
```bash
meta Poem           .. text: "Crimson heart, vibrant and bold,\nHearts flutter at your sight.\nEvenings glow softly red,\nCherries burst with sweet life.\nKisses linger with your warmth.\nLove deep as merlot.\nScarlet leaves falling softly,\nBold in every stroke."
b1,rgba,lsb,xy      .. text: "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ=="
b1,rgba,msb,xy      .. file: OpenPGP Public Key
b2,g,lsb,xy         .. text: "ET@UETPETUUT@TUUTD@PDUDDDPE"
b2,rgb,lsb,xy       .. file: OpenPGP Secret Key
b2,bgr,msb,xy       .. file: OpenPGP Public Key
b2,rgba,lsb,xy      .. file: OpenPGP Secret Key
b2,rgba,msb,xy      .. text: "CIkiiiII"
b2,abgr,lsb,xy      .. file: OpenPGP Secret Key
b2,abgr,msb,xy      .. text: "iiiaakikk"
b3,rgba,msb,xy      .. text: "#wb#wp#7p"
b3,abgr,msb,xy      .. text: "7r'wb#7p"
b4,b,lsb,xy         .. file: 0421 Alliant compact executable not stripped
```
- Resultado detectado (cadena en base64):
```bash
cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==
```
* Decodificación de la cadena base64:
```bash
echo "cGljb0NURntyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==" | base64 --decode
```
* Cadena decodificada:
```bash
picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}
```

## Flag

```text
picoCTF{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}
```

## Cadena de resolución

`PNG → zsteg → plano b1 RGBA LSB → Base64 repetido → flag`

## Qué debo recordar para otro CTF

- `zsteg -a` genera falsos positivos; se debe priorizar texto estructurado y verificable.
- La misma cadena repetida cuatro veces no son cuatro flags distintas.
- Después de extraer un plano de bits, hay que identificar nuevamente la codificación obtenida.

## Aporte para el arsenal

- Agrupar resultados duplicados de `zsteg` sin perder su procedencia.
- Puntuar más alto cadenas con formato de flag o codificaciones completas.
