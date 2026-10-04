# Hardening Basics — Part 1

#hardening #linux #defensive #sudo #firewall #tryhackme

|URL|https://tryhackme.com/room/hardeningbasicspart1|
|---|---|
|Nivel|Info / Fácil|
|Plataforma|TryHackMe|
|Tipo|Walkthrough teórica (hardening de sistemas)|
|Estado|Completa — Cap. 1 (Cuentas) y Cap. 2 (Firewall)|

---

## [RESUMEN]

Sala defensiva sobre cómo endurecer un servidor Ubuntu 18.04. No hay explotación ni flags; termina con un quiz de comprensión. La serie cubre cuatro bloques; esta Part 1 incluye los dos primeros:

1. Cuentas de usuario (este documento)
2. Seguridad del firewall (este documento)
3. SSH y cifrado (Part 2)
4. Control de acceso obligatorio / MAC (Part 2)

Eje de todo el material: principio de mínimo privilegio (cada usuario solo el acceso justo para su trabajo). Lo aplicable a 18.04 vale en general para 20.04 y 22.04. Fuente de los temas: _Mastering Linux Security and Hardening_ (O'Reilly).

---

## [ACCESO A LA MÁQUINA]

Entorno Ubuntu 18.04 para practicar; ninguna pregunta exige tareas en la máquina. Credenciales globales con acceso total:

```
spooky:tryhackme
```

Conexión por SSH tras desplegar y conectar la VPN:

```bash
ssh spooky@<MACHINE_IP>
```

- `spooky` usuario, `<MACHINE_IP>` la IP del despliegue, contraseña `tryhackme`.
- Si SSH rechaza por algoritmos antiguos: `-o HostKeyAlgorithms=+ssh-rsa`.

---

## [1. CUENTAS DE USUARIO]

### 1.1 — sudo

`sudo` ("super-user do") permite a un usuario no-root ejecutar programas como root con su propia contraseña, solo durante esa orden. Ventajas: ralentiza al atacante (sin root no sabe qué cuenta atacar), evita repartir la contraseña de root y encaja con el mínimo privilegio.

### 1.2 — Añadir usuarios a un grupo de administración

Método 1 — grupo `sudo` (lo habitual en Ubuntu). En 18.04 el usuario de instalación entra automáticamente.

Lista los grupos del usuario:

```bash
groups nick
```

Añade un usuario existente al grupo sudo:

```bash
usermod -aG sudo nick
```

- `-G sudo` grupo suplementario. `-a` (append) conserva los grupos previos; sin `-a`, `-G` los reemplaza.

Crea el usuario ya dentro de sudo:

```bash
useradd -G sudo james
```

Lista lo que el usuario actual puede ejecutar con sudo:

```bash
sudo -l
```

Edita `/etc/sudoers` validando la sintaxis (solo root):

```bash
sudo visudo
```

Línea de grupo (el `%` indica grupo):

```
%sudo   ALL=(ALL:ALL) ALL
```

Sin pedir contraseña (desaconsejado):

```
%sudo   ALL=(ALL:ALL) NOPASSWD: ALL
```

Método 2 — `User_Alias` en la política (portable entre distros):

```
User_Alias ADMINS = nick, james, dark
ADMINS  ALL=(ALL:ALL) ALL
```

### 1.3 — Delegar privilegios (`Cmnd_Alias`)

```
Cmnd_Alias SYSTEM = /usr/bin/systemctl restart, /usr/bin/systemctl restart ssh, /usr/bin/chmod
User_Alias SYSTEMADMINS = nick, james
SYSTEMADMINS  ALL = SYSTEM
```

- Control literal: `systemctl restart ssh` se permite; `systemctl restart apache2` falla al no estar en la lista.
- Para todos los servicios, comodín: `/usr/bin/systemctl restart *`.

Otras asignaciones:

```
dark      ALL = WEBDEV          # usuario -> Cmnd_Alias entero
paradox   ALL = /usr/bin/cd     # usuario -> una sola orden
HR        ALL = HR              # User_Alias -> Cmnd_Alias
```

`Host_Alias`: propaga políticas por varios servidores (`Host_Alias MAILSERVERS = mail1, mail2`). Útil solo en redes grandes.

### 1.4 — Peligros de root

root puede todo, incluidos ficheros de sistema y arranque. En SO ofensivos (Kali, Parrot) usarlo es normal; en producción es un riesgo. La práctica correcta: cuenta estándar y elevar puntualmente con sudo.

### 1.5 — Desactivar el acceso de root

Tres capas combinables.

a) Shell de root en `/etc/passwd`:

```
root:x:0:0:root:/root:/usr/sbin/nologin
```

Bloquea `su`, `su -`, `sudo -i` y el login SSH de root. No afecta a `sudo -s` ni `sudo bash`, que usan la shell del usuario que invoca; por eso no basta sola y debe acompañarse de un `sudoers` restrictivo.

b) Login SSH de root en `/etc/ssh/sshd_config`:

```
PermitRootLogin no
```

Recargar: `systemctl restart ssh`.

c) Vía PAM (Pluggable Authentication Modules): librerías que autentican usuarios frente a servicios; config en `/etc/pam.d/` o `/etc/pam.conf`. Editarlo mal puede dejarte fuera del sistema. En `/etc/pam.d/sshd`:

```
auth required pam_listfile.so onerr=succeed item=user sense=deny file=/etc/ssh/deniedusers
```

Crear el fichero de denegados con root dentro:

```bash
echo "root" > /etc/ssh/deniedusers
```

- `auth` fase de autenticación. `required` debe pasar o falla. `pam_listfile.so` deniega/permite según un fichero. `onerr=succeed` acción ante error del módulo. `item=user` compara el usuario. `sense=deny` deniega si aparece. `file=` fichero con un usuario por línea.

### 1.6 — Shell escapes

Permitir editores con sudo abre escalada a root (GTFOBins). Con vim:

```bash
sudo vim -c ':!/bin/sh'
```

```bash
sudo vim -c ':set shell=/bin/sh' -c ':shell'
```

Ambos abren una shell con privilegios heredados. Mitigación: usar `sudoedit` (sin shell escapes) en la política:

```
operator ALL = sudoedit /etc/fstab
```

El usuario edita una copia temporal con su editor sin privilegios elevados; al cerrar, `sudoedit` escribe como root.

### 1.7 — Permisos de los directorios home

Ubuntu crea los home a 755 (UMASK 022): otros usuarios pueden leerlos. El UMASK se fija en `/etc/login.defs`; 077 los hace privados.

```bash
grep -i "^UMASK" /etc/login.defs
```

El UMASK es una máscara de bits (permisos = base AND NOT UMASK), con base 777 para directorios y 666 para ficheros:

- UMASK 022: directorios 755, ficheros 644
- UMASK 077: directorios 700, ficheros 600

Afecta solo a usuarios nuevos; a los existentes hay que ajustarles el home a mano (`chmod 750 /home/usuario`).

### 1.8 — Complejidad de contraseñas (pwquality)

Módulo PAM para exigir complejidad:

```bash
sudo apt-get install libpam-pwquality
```

Al instalarse añade su entrada en `/etc/pam.d/common-password`:

```
password requisite pam_pwquality.so retry=3
```

- `requisite` si falla, termina de inmediato con error. `pam_pwquality.so` aplica los requisitos de `pwquality.conf`. `retry=3` tres intentos.

Opciones en `/etc/security/pwquality.conf` (descomentar y modificar):

- `minlen` longitud mínima (por defecto 8, no baja de 6).
- `dcredit` `ucredit` `lcredit` `ocredit` crédito por dígitos, mayúsculas, minúsculas y otros; en negativo, pasa a ser el mínimo exigido de ese tipo.
- `minclass` clases mínimas de caracteres.
- `maxrepeat` `maxclassrepeat` máximo de caracteres consecutivos iguales o de la misma clase.
- `gecoscheck` impide usar datos del GECOS del usuario.

### 1.9 — Caducidad e historial

Cuatro conceptos de contraseñas: complejidad, longitud, caducidad e historial.

Caducidad en `/etc/login.defs`:

- `PASS_MAX_DAYS` máximo de días de uso (por defecto 99999; recomendado 90).
- `PASS_MIN_DAYS` mínimo antes de poder cambiarla (por defecto 0; recomendado 1).
- `PASS_WARN_AGE` días de aviso previos (por defecto 7).

Historial en `/etc/pam.d/common-password`:

```
password required pam_pwhistory.so remember=10 use_authtok
```

- `pam_pwhistory.so` gestiona el historial. `remember=n` recuerda las últimas n (recomendado 10), guardadas en `/etc/security/opasswd`. `use_authtok` y `shadow` en la línea de `pam_unix.so` generan shadow passwords al actualizar.

Con `PASS_MIN_DAYS=1` y `remember=10`, volver a la contraseña original exige al menos 11 días.

### 1.10 — Grupo lxd

Ubuntu mete usuarios en el grupo `lxd`, que es un vector de escalada conocido y lo detectan herramientas como linux-smart-enumeration (LSE). Quítalo de cualquier usuario que lo tenga:

```bash
sudo deluser <usuario> lxd
```

`adduser` no mete al usuario en grupos predefinidos, por lo que es preferible para crear cuentas nuevas.

---

## [2. FIREWALL]

### 2.1 — iptables y netfilter

iptables es una interfaz para `netfilter`, el firewall real de Linux. Ubuntu incluye además `ufw`, un frontend más sencillo. Las cuatro tablas de iptables:

- Filter: protección básica de firewall (la que se usa).
- NAT: traduce direcciones entre red pública y privadas.
- Mangle: modifica paquetes al pasar.
- Security: solo la usa SELinux.

Ver reglas (requiere root):

```bash
sudo iptables -L
```

Vacío significa que todo el tráfico está permitido. Cadenas de la tabla Filter: `INPUT` (entrante), `FORWARD` (reenviado a otra NIC), `OUTPUT` (saliente). Las reglas (ACL) se leen de arriba abajo.

### 2.2 — Añadir reglas

Aceptar conexiones ya iniciadas por el host:

```bash
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

- `-A INPUT` añade a la cadena INPUT. `-m conntrack` carga el módulo de seguimiento de conexiones. `--ctstate ESTABLISHED,RELATED` conexiones establecidas y relacionadas (RELATED: nuevas pero parte de una ya establecida). `-j ACCEPT` acepta y detiene el procesado (`-j` = jump).

Permitir puertos:

```bash
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4380 -j ACCEPT
```

- `-p` protocolo (tcp o udp). `--dport` puerto de destino. Admite nombres que resuelva `/etc/services` (`ssh`=22, `ftp`=21); ante la duda, usar el número.

### 2.3 — Bloquear tráfico y denegación implícita

Bloquear un servicio (ejemplo, SMB en 445):

```bash
sudo iptables -A INPUT -p tcp --dport 445 -j DROP
```

- `-j DROP` descarta los paquetes sin responder.

Regla de denegación implícita, al final de la lista:

```bash
sudo iptables -A INPUT -j DROP
```

Catch-all: todo lo no aceptado antes se descarta. El tráfico de salida se gestiona igual con la cadena `OUTPUT`.

### 2.4 — Persistencia

iptables no persiste tras reiniciar. Para guardar y restaurar:

```bash
sudo iptables-save > /etc/iptables/rules.v4
sudo apt install iptables-persistent
sudo iptables-restore < /etc/iptables/rules.v4
```

- `iptables-save` vuelca las reglas a un fichero; `iptables-persistent` las recarga al arrancar; `iptables-restore` las aplica desde el fichero.

### 2.5 — UFW

Frontend amigable de iptables (IPv4/IPv6), desactivado por defecto. Requiere root.

```bash
sudo ufw status
sudo ufw enable
sudo ufw disable
```

Permitir o denegar por puerto y por servicio:

```bash
sudo ufw allow 9000/tcp
sudo ufw deny 23
sudo ufw allow ssh
```

- Formato: `sudo ufw <allow|deny> <puerto>/<protocolo>` o `<servicio>`. UFW no requiere guardar manualmente. Sintaxis avanzada (IPs, rangos, subredes): https://help.ubuntu.com/community/UFW

---

## [RESPUESTAS CLAVE — quiz]

|Pregunta|Respuesta|
|---|---|
|Siglas de "sudo"|super-user do|
|Añadir a grupo conservando los demás|`usermod -aG`|
|Fichero de política de sudo|`/etc/sudoers` (editar con `visudo`)|
|Listar lo permitido con sudo|`sudo -l`|
|Shell para deshabilitar login de root|`/usr/sbin/nologin`|
|Config del servidor SSH|`/etc/ssh/sshd_config`|
|Directiva para prohibir root por SSH|`PermitRootLogin no`|
|Significado de PAM|Pluggable Authentication Modules|
|Alternativa a editores contra shell escapes|`sudoedit`|
|Web de referencia de escapes|GTFOBins|
|UMASK por defecto en Ubuntu|022|
|Fichero del UMASK|`/etc/login.defs`|
|UMASK más seguro sugerido|077|
|Permisos de home con UMASK 022|755|
|Paquete de complejidad|`libpam-pwquality`|
|Config de pwquality|`/etc/security/pwquality.conf`|
|Fichero PAM donde pwquality se añade|`/etc/pam.d/common-password`|
|`minlen` por defecto|8|
|Cuatro conceptos de contraseñas|complejidad, longitud, caducidad, historial|
|`PASS_MAX_DAYS` defecto / recomendado|99999 / 90|
|`PASS_MIN_DAYS` defecto / recomendado|0 / 1|
|`PASS_WARN_AGE` por defecto|7|
|Módulo PAM de historial|`pam_pwhistory.so`|
|Dónde se guardan las contraseñas antiguas|`/etc/security/opasswd`|
|Historial recomendado|recordar 10|
|Grupo que es riesgo de privesc|`lxd`|
|Herramienta que detecta lxd|linux-smart-enumeration (LSE)|
|Comando que no mete en grupos predefinidos|`adduser`|
|iptables es un frontend de|netfilter|
|Firewall simplificado de Ubuntu|ufw|
|Cuatro tablas de iptables|Filter, NAT, Mangle, Security|
|Tabla usada solo por SELinux|Security|
|Tres cadenas de la tabla Filter|INPUT, FORWARD, OUTPUT|
|Listar reglas|`sudo iptables -L`|
|Orden de lectura de las ACL|de arriba abajo|
|Estados de conntrack del ejemplo|ESTABLISHED, RELATED|
|Significado de `-j`|jump|
|Regla de denegación implícita|`sudo iptables -A INPUT -j DROP`|
|Estado por defecto de UFW|desactivado|
|Activar UFW|`sudo ufw enable`|
|Formato allow en UFW|`sudo ufw allow <puerto>/<protocolo>`|

---

## [LABORATORIO — monta y endurece tu VM]

Ejercicio para aplicar todo el Part 1 en una VM propia.

Paso 0 — Preparación. Crea una VM Ubuntu Server (18.04 para calcar la sala, o 22.04 LTS si la quieres soportada), crea tu usuario y marca OpenSSH. Snapshot nada más instalar. Antes de tocar `sudoers`, `/etc/pam.d/*` o `sshd_config`, haz snapshot y mantén una segunda terminal con sesión root abierta (`sudo -i`) por si te bloqueas.

Paso 1 — Delegación con sudo:

```bash
sudo visudo
```

```
Cmnd_Alias SYSTEM = /usr/bin/systemctl restart *, /usr/bin/chmod
User_Alias SYSTEMADMINS = testadmin
SYSTEMADMINS ALL = SYSTEM
```

```bash
sudo -l -U testadmin
```

Paso 2 — Endurecer root:

```bash
sudo usermod -s /usr/sbin/nologin root
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

Verificar: `sudo grep -i permitrootlogin /etc/ssh/sshd_config` y `su -` debe fallar.

Paso 3 — Permisos de home:

```bash
sudo sed -i 's/^UMASK.*/UMASK 077/' /etc/login.defs
sudo adduser pruebaumask
ls -ld /home/pruebaumask
```

Debe mostrar `drwx------` (700).

Paso 4 — Complejidad:

```bash
sudo apt-get install -y libpam-pwquality
sudo nano /etc/security/pwquality.conf
```

Ajusta `minlen=12`, `dcredit=-1`, `ucredit=-1`, `ocredit=-1`, `lcredit=-1`. Verifica con `passwd pruebaumask` usando una contraseña débil (debe rechazarla).

Paso 5 — Caducidad e historial:

```bash
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS 90/' /etc/login.defs
sudo sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS 1/'  /etc/login.defs
```

En `/etc/pam.d/common-password` añade antes de la línea de `pam_unix.so`:

```
password required pam_pwhistory.so remember=10 use_authtok
```

Verifica con `sudo chage -l pruebaumask`.

Paso 6 — Grupo lxd:

```bash
groups rene
sudo deluser rene lxd
```

Paso 7 — Firewall (iptables). Permite SSH antes del DROP o perderás el acceso:

```bash
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -j DROP
sudo iptables -L -n --line-numbers
sudo apt install -y iptables-persistent
```

Paso 8 — Alternativa UFW:

```bash
sudo ufw allow ssh
sudo ufw enable
sudo ufw status verbose
```

Verificación final:

- `sudo -l -U testadmin` muestra solo su Cmnd_Alias.
- `PermitRootLogin no` y `su -` falla.
- home nuevos a 700.
- contraseña débil rechazada.
- `chage -l` con caducidad 90 y mínimo 1.
- ningún usuario real en el grupo lxd.
- firewall con SSH permitido, denegación implícita y persistencia.

Versiones: 18.04 está fuera de soporte estándar, úsala solo en laboratorio aislado. En 22.04 todo sigue válido; el `PermitRootLogin` por defecto ya es `prohibit-password`, pero ponerlo a `no` sigue siendo lo correcto.

---

## [CHULETA]

```bash
# Cuentas / sudo
groups <usuario>
usermod -aG sudo <usuario>
adduser <usuario>
sudo deluser <usuario> lxd
sudo -l
sudo visudo

# Root
usermod -s /usr/sbin/nologin root
# /etc/ssh/sshd_config -> PermitRootLogin no
systemctl restart ssh

# Contraseñas / home
# /etc/login.defs -> UMASK 077 | PASS_MAX_DAYS 90 | PASS_MIN_DAYS 1
sudo apt-get install libpam-pwquality
# common-password -> pam_pwhistory.so remember=10 use_authtok
chage -l <usuario>

# Firewall iptables
sudo iptables -L -n --line-numbers
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -j DROP
sudo iptables-save > /etc/iptables/rules.v4

# Firewall ufw
sudo ufw status | enable | disable
sudo ufw allow 9000/tcp
sudo ufw deny 23
sudo ufw allow ssh
```

Plantillas de `/etc/sudoers`:

```
%sudo            ALL=(ALL:ALL) ALL
%sudo            ALL=(ALL:ALL) NOPASSWD: ALL
User_Alias  ADMINS   = nick, james
Cmnd_Alias  SYSTEM   = /usr/bin/systemctl restart *, /usr/bin/chmod
SYSTEMADMINS ALL = SYSTEM
operator    ALL = sudoedit /etc/fstab
```

---

## Notas rápidas

- Part 1 cubre Cap. 1 (cuentas) y Cap. 2 (firewall); SSH/cifrado y MAC están en Part 2.
- Eje: mínimo privilegio.
- `-a` en `usermod -aG` conserva los grupos; sin él se reemplazan.
- Endurecer root = tres capas (shell nologin, `PermitRootLogin no`, PAM); nologin no afecta a `sudo -s`.
- Editores con sudo = privesc; usar `sudoedit`.
- UMASK 077 para home privados.
- pwquality para complejidad, pam_pwhistory para historial (`/etc/security/opasswd`), login.defs para caducidad.
- Grupo lxd = privesc; quitarlo. `adduser` no mete en grupos predefinidos.
- iptables es frontend de netfilter; tablas Filter/NAT/Mangle/Security; cadenas INPUT/FORWARD/OUTPUT; ACL de arriba abajo; cerrar con denegación implícita; persistir con iptables-persistent.
- En el laboratorio: snapshot antes de tocar sudoers/PAM/sshd y permitir SSH antes del DROP.