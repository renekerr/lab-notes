Sí, tu lógica es correcta. Solo hay un detalle: no necesitas tratar el diccionario vacío como caso especial. La pregunta "¿existe ya la clave?" ya cubre ese primer caso, porque con el diccionario vacío la respuesta simplemente es "no".

Tampoco necesitas el contador a 0: el propio valor de cada clave es el contador.

**Tu versión tal cual:**

```python
conteo = {}

for nombre in nombres:
    if nombre in conteo:
        conteo[nombre] += 1
    else:
        conteo[nombre] = 1
```

**Versión más compacta con `.get()`:**

```python
conteo = {}

for nombre in nombres:
    conteo[nombre] = conteo.get(nombre, 0) + 1
```

**Con `collections.Counter`**, que hace exactamente esto ya hecho:

```python
from collections import Counter

conteo = Counter()
for nombre in nombres:
    conteo[nombre] += 1
```

Si en tu loop ya tienes los nombres a mano, incluso puedes hacer `Counter(lista_de_nombres)`. Además, `conteo.most_common()` te devuelve los tipos ordenados de más a menos frecuentes.

Un apunte práctico: si el nombre del log incluye la fecha/hora, tendrás que quitarla antes de contar (con `split`, una regex o `Path.stem`), para que todos los del mismo tipo generen la misma clave. Si me pasas un ejemplo de nombre de archivo, te ayudo a extraer la parte común.