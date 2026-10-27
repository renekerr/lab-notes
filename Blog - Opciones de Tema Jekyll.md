# Blog — Opciones de tema (Jekyll)

> Nota de planificación (no es un writeup de sala). Resumen para decidir el cambio de look de `renekerr.github.io`. Todas las demos son públicas y se pueden abrir desde el navegador.

---

## Punto de partida (estado actual del blog)

`renekerr.github.io` usa el tema **Lanyon** (de @mdo, construido sobre **Poole**), **vendorizado** — es decir, copiado dentro del propio repo, no instalado como gema:

- Estructura en el repo: `_layouts/`, `_includes/`, `public/css/{lanyon,poole,syntax}.css`
- Fuentes: PT Serif / PT Sans
- Comentarios: **giscus** ya integrado

Es sólido y ligero, pero de ~2014. Objetivo: modernizar el look, sobre todo el resaltado de código y la legibilidad para writeups.

---

## Mejores opciones (ordenadas por recomendación)

### 1. Chirpy — recomendado

El que más encaja para un blog técnico / de CTF. Ventajas clave:

- Barra lateral fija con navegación.
- **Modo claro/oscuro**.
- Buscador integrado.
- Resaltado de código **con número de línea y botón de copiar**.
- Buen soporte para tags, categorías y tabla de contenidos.

Enlaces:
- Demo en vivo: https://chirpy.cotes.page/
- Repositorio: https://github.com/cotes2020/jekyll-theme-chirpy
- Plantilla de arranque (lo más cómodo para empezar): https://github.com/cotes2020/chirpy-starter
- Ficha con preview: https://jamstackthemes.dev/demo/theme/jekyll-theme-chirpy/

### 2. Minimal Mistakes — el todoterreno

El tema con mejor documentación de todos. Muy flexible, muchísimas opciones de layout, y páginas de ejemplo de syntax highlighting, galerías y archivos. Buena elección si se quiere algo muy configurable.

- Demo en vivo: https://mmistakes.github.io/minimal-mistakes/

### 3. Just the Docs — estilo wiki/documentación

Si en algún momento se quisiera orientar el sitio más como **documentación/wiki** que como blog cronológico. Navegación lateral jerárquica y buscador.

- Demo en vivo: https://just-the-docs.github.io/just-the-docs/

### 4. hacker-plus-theme — nicho CTF

Tema hecho a medida para **writeups de CTF**. Más de nicho, menos mantenido que los anteriores, pero temáticamente alineado.

- Repositorio: https://github.com/CaptainIRS/hacker-plus-theme

### 5. Opción mínima — seguir en Lanyon y modernizar

El menor esfuerzo: quedarse en Lanyon y solo retocar:
- `public/css/syntax.css` (modernizar el resaltado de código).
- La fuente en `_includes/head.html`.

Sirve si no se quiere migrar, solo refrescar.

---

## Ejemplos reales de blogs CTF/infosec con Chirpy

Para ver el mismo tipo de contenido (writeups con mucho bloque de código) ya montado sobre el tema, no solo la demo oficial:

- https://tr3m0x.github.io — writeups, soluciones de CTF y cheatsheets ("Built with Jekyll and Chirpy").
- https://github.com/dat09398/dat09398.github.io — blog de ciberseguridad + writeups sobre Chirpy starter.
- https://github.com/Jerit3787/ctf — blog de writeups (Chirpy starter).
- Topic de GitHub para explorar más: https://github.com/topics/chirpy-theme

---

## Recomendación

Por el tipo de contenido (writeups con muchos bloques de comandos y salidas), **Chirpy** es el que mejor encaja: el resaltado con número de línea y el botón de copiar vienen de lujo, y el modo oscuro es un plus para lectura técnica.

---

## Pendiente de decidir / hacer

1. **Elegir tema** (apuesta: Chirpy).
2. **Plan de migración** una vez elegido:
   - Portar los posts y su *front matter*.
   - **Reconectar giscus** (comentarios).
   - Ajustar `_config.yml`.
   - Si es Chirpy: montar el **workflow de GitHub Actions**, ya que usa plugins fuera de la whitelist de GitHub Pages.
3. **Prueba previa recomendada:** levantar una copia local con Chirpy (sin tocar el blog actual) y cargar un par de posts reales para verlo antes de decidir.
