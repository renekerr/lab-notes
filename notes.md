# Linux Modules (THM): guía de estudio

Oct 8, 2026 · @rene

## Sobre la sala

Sala de TryHackMe: [Linux Modules](https://tryhackme.com/room/linuxmodules) ("All Linux related"). Es una sala de repaso para familiarizarse con la terminal. Se ha planteado para no tener que leer las páginas `man` completas de cada herramienta sin saber qué significa cada flag. Se recomienda haber completado antes las salas de "Linux Fundamentals", y para la parte de `grep -E` la sala "Regular Expressions" de concatenate.

Se puede hacer en una VM Linux propia o con la AttackBox (los usuarios gratuitos pueden desplegarla una hora, tiempo suficiente según la autora).

**Herramientas que cubre:** `du`, `grep` / `egrep` / `fgrep`, `tr`, `awk`, `sed`, `xargs`, `curl`, `wget`, `xxd` y alguna más.

### Qué significa "first blood"

Es un término de los CTF (y de los videojuegos): **la primera persona o equipo que resuelve un reto o consigue una flag**. Se usa en TryHackMe, Hack The Box y competiciones, y suele dar reconocimiento (a veces una insignia o mención en el ranking) más que puntos extra. Variantes: *user blood* (primer acceso inicial) y *root blood* (primera escalada a root) en máquinas nuevas.

En esta sala **no hay first bloods, solo puntos**, así que no hay prisa. Las pistas de las tareas pueden contener respuestas directas, por lo que conviene mirarlas solo si hace falta.

### Cómo aprovecharla

- Prueba variantes de cada comando en tu VM, rompe cosas a propósito y mira qué errores salen.
- Entiende cada flag antes de pasar a la siguiente tarea.
- Mantén esta guía como referencia: cada sección recoge el contenido de la sala más correcciones y matices.

## du (disk usage)

`du` muestra qué ficheros y directorios consumen espacio. Sin flags, lista solo los directorios con su tamaño en KB; los ficheros de dentro no aparecen.

### Flags de la sala

| Flag | Descripción |
| --- | --- |
| `-a` | Lista también los ficheros, no solo los directorios |
| `-h` | Tamaños legibles (B, KB, MB, GB) |
| `-c` | Imprime el total al final |
| `-d <número>` | Profundidad máxima a mostrar (por ejemplo `-d 2`) |
| `--time` | Añade la fecha de última modificación |

### Ejemplos de la sala

```bash
du -a /home/                     # cada fichero de /home con su tamaño en KB
du -a /home/ | grep user         # solo lo que contiene "user" en el nombre
du --time -d 1 .                 # alternativa a ls con tamaños y fechas
stat <fichero>                   # propietario, permisos, tamaños y fechas
```

`du` no muestra el propietario; para eso se usa `stat`.

### Matices y combinaciones útiles

- `du -sh <dir>`: tamaño total de un directorio en una línea (`-s` = summarize).
- `-d 1` equivale a `--max-depth=1`.
- `du -ah . | sort -rh | head -10`: los 10 elementos que más ocupan. `sort -rh` ordena de mayor a menor entendiendo K, M y G.
- `du` mide espacio en disco usado, que puede diferir del tamaño aparente. Para bytes reales: `--apparent-size`.
- `--time` es una opción larga (dos guiones) y solo existe en `du` de GNU; en macOS da error.
- `du -shc dir1 dir2` añade una línea `total` al pasar varios directorios.
- `stat -c '%U' fichero` muestra solo el propietario en Linux.
- `2>/dev/null` descarta los errores de permiso denegado y deja la salida limpia.

### Práctica

1. `du -h -d 1 /var 2>/dev/null`: ¿qué subdirectorio pesa más?
2. Compara `du -sh /etc` con `du -sh --apparent-size /etc`.
3. `du -a /etc 2>/dev/null | grep ssh`: filtra la salida con grep.

## grep, egrep y fgrep

grep busca un patrón en un fichero y muestra las líneas que lo contienen. **Sintaxis:** `grep "PATRÓN" file.txt`. Para usar `egrep`, la sala recomienda haber hecho la sala "Regular Expressions".

### La familia

- `grep -E` equivale a `egrep` (expresiones regulares extendidas).
- `grep -F` equivale a `fgrep` (cadena fija, sin interpretar regex).
- `egrep` y `fgrep` están obsoletos; hoy se usa `grep -E` y `grep -F`.

### Flags de la sala

| Flag | Descripción |
| --- | --- |
| `-R` | Búsqueda recursiva dentro de directorios |
| `-h` | En búsquedas recursivas, no antepone el nombre del fichero |
| `-c` | Muestra solo cuántas veces se encontró el patrón |
| `-i` | Ignora mayúsculas y minúsculas |
| `-l` | Lista solo los nombres de fichero con coincidencias |
| `-n` | Muestra el número de línea |
| `-v` | Muestra las líneas que NO contienen el patrón |
| `-E` | Interpreta el patrón como regex extendida |
| `-e` | Permite especificar varios patrones (`grep -e P1 -e P2 file.txt`) |

### Corrección: `-e` frente a `-E`

La sala presenta esto de forma confusa. Lo correcto es:

- `-e PATRÓN` solo indica "este es un patrón". Sirve para pasar varios (`-e a -e b`) o patrones que empiezan por `-`. No cambia el tipo de regex.
- `-E` cambia el dialecto de regex a extendido (ERE). Por defecto grep usa BRE.
- No es cierto que BRE case un patrón y ERE dos o más: la diferencia es de sintaxis. En ERE los metacaracteres `+ ? | ( ) { }` funcionan directamente; en BRE hay que escaparlos (`\+`, `\|`, `\(`).
- Sí se pueden combinar: `grep -E -e 'a+' -e 'b|c' fichero`.
- El ejemplo `^s.*n$` funciona igual en BRE y ERE, así que no ilustra la diferencia.

```bash
grep 'sun\|moon' file.txt                 # BRE: la | se escapa (extensión GNU)
grep -E 'sun|moon' file.txt               # ERE: alternancia directa
grep -E '[0-9]{3}-[0-9]{4}' file.txt      # 3 dígitos, guion, 4 dígitos
```

### Otros matices

- `-R` y `-r` son recursivos; `-R` además sigue enlaces simbólicos.
- `-c` cuenta líneas que coinciden, no apariciones. Para contar apariciones: `grep -o patrón fichero | wc -l`.
- `-L` lista los ficheros sin coincidencias.
- `-o` imprime solo la parte que coincide; muy útil para extraer IPs, emails o hashes.
- `-w` casa palabra completa; `-A n`, `-B n` y `-C n` muestran líneas de contexto.

### Práctica

```bash
printf 'sun\nmoon\nSunny day\n192.168.1.10\nuser: admin\n' > test.txt
grep -i 'sun' test.txt
grep -E '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$' test.txt
grep -o -E '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' test.txt
grep -v 'sun' test.txt
```

## Strops (string operations)

La sala defiende que dominar la manipulación de cadenas es lo que da su potencia a la terminal: encadenando comandos con pipes se puede seleccionar un solo byte de un texto enorme. Indexar arrays, hacer slicing en Python o usar grep son formas de manipular cadenas. En bash hay muchas herramientas para lo mismo; se elige la de sintaxis más corta.

**Herramientas principales:** `tr`, `awk`, `sed`, `xargs`. **Complementarias:** `sort`, `uniq`.

| Herramienta | Qué hace |
| --- | --- |
| `tr` | Sustituye, borra o comprime caracteres (solo lee de stdin) |
| `awk` | Divide cada línea en campos y opera sobre columnas |
| `sed` | Edita el flujo de texto línea a línea, sobre todo con sustituciones (`s/viejo/nuevo/`) |
| `xargs` | Convierte líneas de entrada en argumentos de otro comando |
| `sort` | Ordena líneas |
| `uniq` | Elimina o cuenta líneas consecutivas repetidas |

`uniq` solo detecta duplicados contiguos, por lo que casi siempre va después de `sort`. Patrón clásico para contar y ordenar por frecuencia:

```bash
... | sort | uniq -c | sort -rn
```

Adelanto de cómo se complementan, con una sola entrada:

```bash
echo "Hola Mundo desde Linux" | tr 'a-z' 'A-Z'          # HOLA MUNDO DESDE LINUX
echo "Hola Mundo desde Linux" | awk '{print $2}'        # Mundo
echo "Hola Mundo desde Linux" | sed 's/Mundo/Kali/'     # Hola Kali desde Linux
printf 'a\nb\nc\n' | xargs echo                         # a b c
```

Según el nivel de trabajo: `tr` actúa sobre caracteres, `awk` sobre campos y `sed` sobre patrones.

Pendiente de la sala: `xargs`, `sort`, `uniq`, `curl`, `wget` y `xxd`. Cuando pegues esas secciones se añaden como apartados nuevos.

## awk

awk es un lenguaje de scripting para manipular datos y generar informes: no necesita compilarse y admite variables, funciones numéricas y de cadenas, y operadores lógicos. La sala lo llama la herramienta "todo en uno" (el AWP del CSGO), aunque advierte de que sus comandos pueden ser más largos que los de `sed` o `xargs`. El nombre viene de sus creadores: Aho, Weinberger y Kernighan.

**Sintaxis:** `awk [flags] 'patrón/comandos' [fichero]`. También acepta entrada por pipe. Si los comandos están en un script, se ejecutan con `-f`: `awk -f script.awk input.txt`.

### Fichero de ejemplo (`file.txt`)

Todas las capturas usan este fichero. Primero `cat` y luego `awk '{print}'`, con la misma salida:

```
pjrox@kali:~/Desktop$ cat file.txt
ippsec youtube hackthebox 34024
john youtube ctf 50024
thecybermentor tcmsec courses 25923
liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365
stok hackerone bughunting 1234
pjrox@kali:~/Desktop$ awk '{print}' file.txt
ippsec youtube hackthebox 34024
john youtube ctf 50024
thecybermentor tcmsec courses 25923
liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365
stok hackerone bughunting 1234
```

### Buscar un patrón

El patrón va entre barras `/patrón/`. Para saber quién juega CTF:

```
pjrox@kali:~/Desktop$ awk '/ctf/' file.txt
john youtube ctf 50024
liveoverflow youtube ctf 45345
```

### Campos: `$1`, `$2`, … `$n`

Los campos se separan por espacio por defecto. `$0` es la línea completa. Las llaves `{}` delimitan la acción. Usa comillas simples; con comillas dobles el shell interpreta `$`, así que habría que escaparlo con `\$`.

Sin coma, los campos se pegan:

```
pjrox@kali:~/Desktop$ awk '{print $1 $3}' file.txt
ippsechackthebox
johnctf
thecybermentortcourses
liveoverflowctf
nahamsecbughunting
stokbughunting
```

Con coma se separan con el separador de salida (un espacio por defecto):

```
pjrox@kali:~/Desktop$ awk '{print $1,$3}' file.txt
ippsec hackthebox
john ctf
thecybermentor courses
liveoverflow ctf
nahamsec bughunting
stok bughunting
```

### Variables integradas

| Variable | Significado | Uso |
| --- | --- | --- |
| `NR` | Número de registro (línea) | `awk '{print NR,$0}' file.txt` |
| `FS` | Separador de campos de entrada | `awk 'BEGIN{FS="o"} ...'` |
| `RS` | Separador de registros de entrada (por defecto `\n`) | `awk 'BEGIN{RS="o"} ...'` |
| `OFS` | Separador de campos de salida | `awk 'BEGIN{OFS=":"} ...'` |
| `ORS` | Separador de registros de salida | `awk 'BEGIN{ORS="\n\n"} ...'` |

**NR** numera las líneas:

```
pjrox@kali:~/Desktop$ awk '{print NR,$0}' file.txt
1 ippsec youtube hackthebox 34024
2 john youtube ctf 50024
3 thecybermentor tcmsec courses 25923
4 liveoverflow youtube ctf 45345
5 nahamsec hackerone bughunting 12365
6 stok hackerone bughunting 1234
```

**BEGIN y END** son patrones especiales: `BEGIN` se ejecuta antes de leer la entrada y `END` después de procesarla toda. Sirven para definir `FS`, `RS`, etc. y para imprimir totales.

**FS** (aquí la letra `o` como separador, y se imprime el segundo campo):

```
pjrox@kali:~/Desktop$ awk 'BEGIN{FS="o"} {print $2}' file.txt
utube hacktheb
hn y
r tcmsec c
verfl
ne bughunting 12365
k hacker
```

Combinado con `END` para contar filas:

```
pjrox@kali:~/Desktop$ awk 'BEGIN{FS="o"} {print $1,$3} END{print "Total Rows= " NR}' file.txt
ippsec y x 34024
j utube ctf 50024
thecyberment urses 25923
live w y
nahamsec hacker
st ne bughunting 1234
Total Rows= 6
```

**RS** (cada `o` genera un salto de línea nuevo, además de los `\n` originales):

```
pjrox@kali:~/Desktop$ awk 'BEGIN{RS="o"} {print $0}' file.txt
ippsec y
utube hacktheb
x 34024
j
hn y
utube ctf 50024
thecyberment
r tcmsec c
urses 25923
live
verfl
w y
utube ctf 45345
nahamsec hacker
ne bughunting 12365
st
k hacker
ne bughunting 1234
```

**OFS** solo se aplica cuando los campos se imprimen por separado. Con `$0` no cambia nada, porque se imprime la línea entera:

```
pjrox@kali:~/Desktop$ awk 'BEGIN{OFS=":"} {print $0}' file.txt
ippsec youtube hackthebox 34024
john youtube ctf 50024
thecybermentor tcmsec courses 25923
liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365
stok hackerone bughunting 1234
pjrox@kali:~/Desktop$ awk 'BEGIN{OFS=":"} {print $1,$2,$3.$4}' file.txt
ippsec:youtube:hackthebox34024
john:youtube:ctf50024
thecybermentor:tcmsec:courses25923
liveoverflow:youtube:ctf45345
nahamsec:hackerone:bughunting12365
stok:hackerone:bughunting1234
```

En el segundo comando se escribió `$3.$4` (con punto) en lugar de `$3,$4`, por eso esos dos campos salen pegados.

Con `$0,$0` la línea se imprime dos veces unida por `:`:

```
pjrox@kali:~/Desktop$ awk 'BEGIN{OFS=":"} {print $0,$0}' file.txt
ippsec youtube hackthebox 34024:ippsec youtube hackthebox 34024
john youtube ctf 50024:john youtube ctf 50024
thecybermentor tcmsec courses 25923:thecybermentor tcmsec courses 25923
liveoverflow youtube ctf 45345:liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365:nahamsec hackerone bughunting 12365
stok hackerone bughunting 1234:stok hackerone bughunting 1234
```

**ORS** con doble salto de línea (una línea en blanco entre registros):

```
pjrox@kali:~/Desktop$ awk 'BEGIN{ORS="\n\n"} {print $0}' file.txt
ippsec youtube hackthebox 34024

john youtube ctf 50024

thecybermentor tcmsec courses 25923

liveoverflow youtube ctf 45345

nahamsec hackerone bughunting 12365

stok hackerone bughunting 1234

```

### Flags importantes (según la sala)

| Flag | Descripción |
| --- | --- |
| `-F` | Define el separador de campos (`FS`) sin usar `BEGIN` |
| `-v` | Define variables (equivale a lo hecho con `BEGIN{OFS=":"}`) |
| `-D` | Depurar scripts `.awk` |
| `-o` | Fichero de salida (por defecto `awkprof.out`) |

### Recursos que recomienda la sala

- Tutorialspoint: AWK Workflow (aprender scripting awk rápido)
- La sentencia `printf` en awk (formato de cadenas)
- GeeksforGeeks: AWK command in Unix/Linux with examples
- `man gawk`, para profundizar

## tr (translate)

tr sirve para cambiar mayúsculas y minúsculas, sustituir o borrar caracteres de una cadena. Según la sala es el comando más sencillo y imprescindible para operaciones rápidas con texto. Trabaja con **conjuntos** de caracteres.

**Sintaxis:** `tr [flags] [origen] [destino]`

### Flags (según la sala)

| Flag | Descripción de la sala |
| --- | --- |
| `-d` | Borra un conjunto de caracteres |
| `-t` | Concatena el conjunto origen con el destino (truncate) |
| `-s` | Sustituye el conjunto origen por el destino (squeeze) |
| `-c` | Complemento: invierte la selección. Con `-d`, borra todo excepto el conjunto indicado |

### Ejemplos

Pasar todos los caracteres alfabéticos a mayúsculas, con rangos o con clases de caracteres (el resultado es el mismo):

```
cat file.txt | tr 'a-z' 'A-Z'
cat file.txt | tr -s '[:lower:]' '[:upper:]'
```

Las clases tipo `[:lower:]` funcionan como los emojis de Discord con `:palabra:`. Hay más; se ven con `tr --help`.

Ver las credenciales de un usuario que están en dígitos: la sala borra con `-d` las letras minúsculas y mayúsculas, el símbolo `:` y el espacio, y deja solo los números.

> Nota: las capturas de la sección de tr no estaban adjuntas, así que aquí no hay transcripción de su salida. Si las pegas, las añado.

### Recursos que recomienda la sala

- GeeksforGeeks: tr command in Unix/Linux with examples
- Linuxize: Tr Command in Linux with Examples

## sed (stream editor)

sed es un editor de flujo: aplica operaciones de texto (buscar y reemplazar, buscar, insertar, borrar) sobre una entrada que le llega por pipe o redirigida con `<`. La sala lo llama la segunda herramienta más potente después de awk, y lo compara con un `vi` orientado a flujos: no abres el fichero, le pasas la orden y procesa la entrada.

**Sintaxis:** `sed [flags] 'script' [fichero]`

### Flags de la sala

| Flag | Descripción |
| --- | --- |
| `-e` | Añade un script o comando a ejecutar |
| `-f` | Lee el script desde un fichero |
| `-E` | Usa expresiones regulares extendidas |
| `-n` | Suprime la impresión automática |

### Anatomía del script

La sala la divide así (entre comillas simples):

```
'[condición opcional] [comando]/[patrón a buscar]/[reemplazo]/[flags]'
```

Ejemplo desglosado: `sed -e '1,3 s/john/JOHN/g' file.txt`

- `-e`: lo siguiente es un script. Con un solo comando no hace falta.
- `1,3`: condición, el rango de líneas 1 a 3 (empiezan en 1).
- `s`: comando de sustitución.
- `/`: delimitador entre las partes.
- `john`: patrón a buscar (puede ser regex).
- `JOHN`: texto de reemplazo.
- `g`: aplica el cambio a todas las coincidencias de la línea.
- `file.txt`: fichero de entrada.

### Comandos y flags según la sala

| Elemento | Descripción de la sala |
| --- | --- |
| `s` | Sustitución (el más usado) |
| `y` | Como sustitución pero carácter a carácter; no admite argumentos ni condiciones |
| `g` | Aplica el cambio globalmente en la línea |
| `i` | Búsqueda sin distinguir mayúsculas |
| `d` | Borra la línea completa |
| `p` | Imprime la coincidencia (sale duplicada salvo que se use `-n`) |
| `1`, `2`, `3`… `n` | Actúa sobre la ocurrencia número n de la línea (con `s`) |

### Ejemplos de las capturas

Todas usan `file.txt`, el mismo de la sección de awk.

**Sustituir en un rango de líneas** (solo la línea 2 contiene `john`):

```
pjrox@kali:~/Desktop$ sed -e '1,3 s/john/JOHN/g' file.txt
ippsec youtube hackthebox 34024
JOHN youtube ctf 50024
thecybermentor tcmsec courses 25923
liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365
stok hackerone bughunting 1234
```

**Ver un rango de líneas** (`-n` evita los duplicados que genera `p`):

```
pjrox@kali:~/Desktop$ sed -n '3,5p' file.txt
thecybermentor tcmsec courses 25923
liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365
```

**Ver todo el fichero menos un rango** (borrando con `d`):

```
pjrox@kali:~/Desktop$ sed '3,5d' file.txt
ippsec youtube hackthebox 34024
john youtube ctf 50024
stok hackerone bughunting 1234
```

**Ver varios rangos**, usando `-e` una vez por rango:

```
pjrox@kali:~/Desktop$ sed -n -e '1,2p' -e '4,5p' file.txt
ippsec youtube hackthebox 34024
john youtube ctf 50024
liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365
```

**Actuar sobre la ocurrencia n de una línea.** Con `1` cambia el primer `youtube` de cada línea; con `2` no cambia nada porque ninguna línea tiene dos:

```
pjrox@kali:~/Desktop$ sed 's/youtube/YOUTUBE/1' file.txt
ippsec YOUTUBE hackthebox 34024
john YOUTUBE ctf 50024
thecybermentor tcmsec courses 25923
liveoverflow YOUTUBE ctf 45345
nahamsec hackerone bughunting 12365
stok hackerone bughunting 1234
pjrox@kali:~/Desktop$ sed 's/youtube/YOUTUBE/2' file.txt
ippsec youtube hackthebox 34024
john youtube ctf 50024
thecybermentor tcmsec courses 25923
liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365
stok hackerone bughunting 1234
```

**Limpiar espacios sobrantes** con una regex. `file1.txt` tiene huecos irregulares (el espaciado de abajo es aproximado) y `s/  */ /g` los reduce a un espacio, porque `   * ` significa un espacio seguido de cero o más espacios:

```
pjrox@kali:~/Desktop$ cat file1.txt
ippsec      youtube hackthebox 34024
john          youtube         ctf         50024
thecybermentor    tcmsec            courses        25923
liveoverflow            youtube ctf          45345
nahamsec                hackerone bughunting            12365
stok       hackerone        bughunting 1234
pjrox@kali:~/Desktop$ sed 's/  */ /g' file1.txt
ippsec youtube hackthebox 34024
john youtube ctf 50024
thecybermentor tcmsec courses 25923
liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365
stok hackerone bughunting 1234
```

**Regex con grupos**: poner un `=>` al inicio de cada línea y encerrar los números entre corchetes:

```
pjrox@kali:~/Desktop$ cat file.txt
ippsec youtube hackthebox 34024
john youtube ctf 50024
thecybermentor tcmsec courses 25923
liveoverflow youtube ctf 45345
nahamsec hackerone bughunting 12365
stok hackerone bughunting 1234
pjrox@kali:~/Desktop$ sed 's/\(^\b[[:alpha:] ]*\)\([[:digit:]]*\)/\=\> \1\[\2\]/g' file.txt
=> ippsec youtube hackthebox [34024]
=> john youtube ctf [50024]
=> thecybermentor tcmsec courses [25923]
=> liveoverflow youtube ctf [45345]
=> nahamsec hackerone bughunting [12365]
=> stok hackerone bughunting [1234]
```

### Cómo leer la regex del último ejemplo

- El 70 % es regex, no sed. `\( ... \)` abre y cierra un **grupo**; sed los numera por orden como `\1`, `\2`…
- Grupo 1: `^` (inicio de línea), `\b` (inicio de palabra) y `[[:alpha:] ]*`, que toma letras **y espacios** todo lo que pueda. Hay un espacio dentro del corchete; por eso avanza varias palabras hasta que empiezan los dígitos.
- Grupo 2: `[[:digit:]]*`, los dígitos que siguen.
- Reemplazo: ` =>  `, el grupo 1, `[`, el grupo 2 y `]`.
- `[:alpha:]` y `[:digit:]` son *bracket expressions*, equivalentes a `a-zA-Z` y `0-9`, y solo valen dentro de corchetes.

### Correcciones y matices

- **`d` y `p` son comandos, no flags de `s`.** Se usan solos con una dirección (`3,5d`, `3,5p`), como en los ejemplos. Las capturas lo muestran así.
- **`\1` no es lo mismo que `/1`.** En el reemplazo, `\1` y `\2` son referencias a los grupos (con barra invertida). El número al final de `s/a/b/2` es la ocurrencia a sustituir. La sala los confunde en el texto.
- **`2g`** sustituye la segunda ocurrencia y todas las siguientes.
- **`y` es el equivalente de `tr`** en sed: `sed 'y/abc/xyz/'`; los dos conjuntos deben tener la misma longitud.
- **El ejemplo de limpieza colapsa espacios intermedios.** Para quitar solo los espacios del final de la línea: `sed 's/[[:space:]]*$//'`.
- **Con `-E` la regex de los grupos es mucho más legible**, sin barras de escape ni `\b`, y con la `g` sobrando por el `^`:

```bash
sed -E 's/^([[:alpha:] ]*)([[:digit:]]*)/=> \1[\2]/' file.txt
```

- **`-i` edita el fichero en su sitio** (no aparece en la sala). `sed -i.bak 's/a/b/' fichero` hace antes una copia `.bak`. En macOS la sintaxis es distinta (`-i ''`).
- Si el reemplazo contiene `/`, se puede usar otro delimitador: `sed 's|/usr|/opt|'`.

### Recursos que recomienda la sala

- GeeksforGeeks: Sed Command in Linux/Unix with examples
- gnu.org: sed, a stream editor (documentación oficial)
- Tecmint: 15 Useful 'sed' Command Tips and Tricks for Daily Linux System Administration Tasks

## Notas de repaso: correcciones y matices

Varias descripciones de la sala son imprecisas; esto es lo que conviene tener claro.

### awk

- **`-D` y `-o`:** en gawk, `-D` (`--debug`) lanza el depurador interactivo y `-o` (`--pretty-print`) escribe el programa reformateado en `awkprof.out`. No sirve para guardar el resultado: para eso se redirige con `>`.
- **Versión de awk:** la sala dice que gawk viene en todas las distribuciones. En Debian, Kali y Ubuntu el `awk` por defecto suele ser mawk, y en macOS es el awk de BSD, así que `-D` y `-o` pueden no existir.
- **Comillas:** los ejemplos de las capturas usan comillas simples por fuera y dobles por dentro (`'BEGIN{FS="o"} ...'`). El ejemplo del texto con comillas dobles por fuera no funcionaría tal cual por la expansión de `$1` en el shell.
- **Typo en una captura:** `$3.$4` concatena los campos; para separarlos hay que escribir `$3,$4`.
- **Más útiles en la práctica:**

```bash
awk -F: '{print $1}' /etc/passwd     # -F fija el separador sin BEGIN
awk '{print $NF}' file.txt           # último campo de cada línea
awk '$4 > 40000' file.txt            # filtrar por condición numérica
awk '{print $1}' auth.log | sort | uniq -c | sort -rn
```

### tr

- **`-s` no sustituye:** comprime caracteres repetidos consecutivos en uno solo (`tr -s ' '` reduce varios espacios a uno). La sustitución es lo que hace `tr` por defecto, sin flags.
- **`-t` no concatena:** trunca el conjunto origen a la longitud del destino. Es útil solo en casos concretos.
- **`-c` + `-d`:** `tr -cd '0-9'` conserva solo los dígitos y borra todo lo demás. Es una forma más corta de lo que hace el ejemplo de las credenciales.
- **Solo lee de stdin:** no acepta ficheros como argumento, y `cat` no hace falta: `tr 'a-z' 'A-Z' < file.txt`.

```bash
echo "user: admin 1234" | tr -cd '0-9'          # 1234
echo "a   b    c" | tr -s ' '                  # a b c
echo "Hola Mundo" | tr '[:lower:]' '[:upper:]'  # HOLA MUNDO
```

## Laboratorio de hardening

Plan: montar una máquina nueva (Ubuntu), aplicar hardening por bloques y usar el proceso para practicar las herramientas de esta guía con datos reales.

### Base de la máquina

- Ubuntu Server 24.04 LTS, sin escritorio: menos superficie de ataque y más cercano a un servidor real. Con Mac Apple Silicon hace falta la ISO ARM64.
- Un snapshot nada más instalar y otro antes de cada bloque de hardening, para poder volver atrás si algo se rompe (por ejemplo, quedarse sin SSH).

### Pasos de hardening

1. Actualizar el sistema y activar `unattended-upgrades`.
2. Crear un usuario sin privilegios con `sudo` y dejar de usar root.
3. SSH: acceso con clave, `PermitRootLogin no`, `PasswordAuthentication no`.
4. Firewall con `ufw`: denegar todo por defecto y abrir solo lo necesario.
5. `fail2ban` contra fuerza bruta.
6. Auditoría con `lynis audit system` y repaso de sus sugerencias.
7. Revisar servicios y puertos expuestos con `ss -tulpn`.

### Dónde encaja cada herramienta

| Herramienta | Uso en el laboratorio |
| --- | --- |
| `du` | Ver qué llena `/var/log`: `du -ah /var \| sort -rh \| head` |
| `grep` | Intentos fallidos: `grep -i 'failed password' /var/log/auth.log` |
| `grep -o -E` + `sort` + `uniq -c` | Extraer IPs atacantes y contarlas |
| `awk` | Sacar columnas de los logs (usuario, IP, fecha) |
| `sed` y `tr` | Limpiar y transformar la salida |
| `xargs` | Aplicar un comando a una lista, por ejemplo bloquear con `ufw` cada IP |
| `curl` y `wget` | Probar tus servicios y cabeceras; descargar scripts como el de Lynis |
| `xxd` | Inspeccionar ficheros binarios y hexadecimal |

### Ejercicio propuesto

Expón SSH un tiempo, revisa en los logs los intentos reales de login y monta un one-liner que dé las 10 IPs más insistentes. Practica `grep`, `awk`, `sort` y `uniq` a la vez. Un punto de partida:

```bash
grep -i 'failed password' /var/log/auth.log | grep -o -E '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | sort | uniq -c | sort -rn | head -10
```

## Elección de distribución

Para practicar hardening y configuración, la recomendación es **Ubuntu Server LTS (24.04)** y, como segundo paso, una máquina con **Rocky Linux o AlmaLinux**.

| Distribución | Cuándo elegirla | Nota |
| --- | --- | --- |
| Ubuntu Server LTS | Punto de partida | Es lo más habitual en servidores reales; soporte de 5 años; AppArmor, `ufw` y `unattended-upgrades`; Lynis y OpenSCAP en sus repositorios; guías CIS y de Canonical |
| Rocky / AlmaLinux | Segundo paso | Familia RHEL, muy presente en entornos corporativos; enseña SELinux, `firewalld` y `dnf` |
| Debian estable | Entender el sistema desde la base | Parecida a Ubuntu pero más austera |
| Kali | Laboratorio ofensivo | Pensada para ataque, no para endurecer |
| Alpine | Contenedores | Usa musl y BusyBox, así que algunas herramientas se comportan distinto; no es ideal para empezar |

Con un Mac Apple Silicon hacen falta las imágenes ARM64; Ubuntu Server y Rocky las ofrecen.

### Plan de estudio sugerido

1. Instalar Ubuntu Server y hacer un snapshot.
2. Aplicar el hardening por bloques: SSH, firewall, auditoría con Lynis y fail2ban.
3. Atacar la máquina desde una Kali para comprobar que las medidas funcionan.
4. Repetir el proceso en Rocky con SELinux.
