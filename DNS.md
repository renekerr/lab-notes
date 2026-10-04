---
tags: [dns, dig, recon, enumeracion, reference]
plataforma: TryHackMe
fecha: 2026-10-01
estado: referencia
---

# DNS y `dig` — Fundamentos y referencia

> **Índice:** [[Networking MOC]] · **Fundamentos de red:** [[Networking]] · **Enum operativa:** [[Network Protocols Playbook#2. DNS (53 TCP/UDP)]]

> **Salas relacionadas:** [[Dig Dug]] · [[DNS Manipulation]]

## 1. Qué es DNS

DNS traduce nombres de dominio (`tryhackme.com`) a direcciones IP. Es un sistema jerárquico y distribuido: ningún servidor tiene toda la información; cada uno sabe a quién preguntar después.

### Jerarquía de un dominio (se lee de derecha a izquierda)

```
admin.tryhackme.com.
  │       │      │  └── Root (.)  → raíz, normalmente invisible
  │       │      └───── TLD (com)
  │       └──────────── Second-Level Domain (tryhackme)
  └──────────────────── Subdominio (admin)
```

|Nivel|Ejemplo|Notas|
|---|---|---|
|Root|`.`|Gestionado por los 13 grupos de root servers|
|TLD|`.com`, `.es`, `.co.uk`|gTLD (genéricos) y ccTLD (de país)|
|SLD|`tryhackme`|Máx. 63 caracteres, a-z 0-9 y guiones (no al inicio ni al final)|
|Subdominio|`admin`|Cada etiqueta máx. 63 caracteres; nombre completo máx. 253|

### Tipos de registro

|Registro|Función|
|---|---|
|**A**|Nombre → IPv4|
|**AAAA**|Nombre → IPv6|
|**CNAME**|Alias hacia otro nombre|
|**MX**|Servidores de correo (prioridad: número menor = preferido)|
|**TXT**|Texto libre: verificación de propiedad, SPF, DKIM, DMARC|
|**NS**|Servidores autoritativos de la zona|
|**SOA**|Datos de la zona: servidor primario, contacto, serial, tiempos|
|**PTR**|IP → nombre (DNS inverso)|

### Proceso de resolución

1. El equipo consulta su **caché local**.
2. Si no está, pregunta al **resolver recursivo** (ISP, 8.8.8.8, 1.1.1.1...).
3. El resolver pregunta a un **root server**, que lo remite al servidor del **TLD**.
4. El TLD lo remite al **servidor autoritativo** del dominio.
5. El autoritativo da la respuesta final, que se cachea durante el **TTL** (en segundos).

---

## 2. `dig` en profundidad

`dig` (Domain Information Groper) construye una consulta DNS, la envía a un servidor y muestra la respuesta **en bruto**, sin interpretarla. Es preferible a `nslookup` para depurar y hacer recon porque enseña el paquete completo.

### Sintaxis

```bash
dig [@servidor] [dominio] [tipo] [+opciones]
```

|Parte|Qué decide|Si se omite|
|---|---|---|
|`@servidor`|**A quién** preguntas|Resolver de `/etc/resolv.conf`|
|`dominio`|**Por qué nombre** preguntas|La raíz `.`|
|`tipo`|**Qué registro** quieres|`A`|
|`+opciones`|Cómo se envía o muestra|Salida completa|

> [!tip] Regla mental **`@` decide a quién, el dominio decide qué, el tipo decide qué registro.** Si una respuesta extraña, mira primero la línea `SERVER` y el `status`.

El orden de dominio y tipo es indiferente: dig reconoce los tipos por su nombre.

### Anatomía de la salida

Ejemplo real (respuesta del reto):

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53966
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;givemetheflag.com.             IN      A

;; ANSWER SECTION:
givemetheflag.com.      0       IN      TXT     "flag{...}"

;; Query time: 28 msec
;; SERVER: <TARGET_IP>#53(<TARGET_IP>) (UDP)
```

**HEADER**

- `status`: resultado de la consulta.

|Status|Significado|
|---|---|
|`NOERROR`|Correcto (ojo: puede venir con ANSWER: 0 si el nombre existe pero no tiene ese tipo)|
|`NXDOMAIN`|El nombre no existe|
|`SERVFAIL`|El servidor falló al resolver|
|`REFUSED`|El servidor se niega a responder|

- `id`: número aleatorio para emparejar pregunta y respuesta.
- `QUERY/ANSWER/AUTHORITY/ADDITIONAL`: número de registros en cada sección.

**Flags**

|Flag|Significado|
|---|---|
|`qr`|Es una respuesta|
|`aa`|Respuesta **autoritativa** (del dueño de la zona, no de caché)|
|`rd`|Se pidió recursión (dig la pide por defecto)|
|`ra`|El servidor ofrece recursión (es un resolver)|
|`ad`|Datos validados con DNSSEC|

**Secciones**

- **QUESTION**: lo que se preguntó. El `;` inicial indica comentario. `IN` = clase Internet.
- **ANSWER**: registros que responden, con formato `nombre TTL clase tipo valor`.
- **AUTHORITY**: servidores responsables de la zona. Con `NXDOMAIN` suele aparecer el `SOA`.
- **ADDITIONAL**: datos auxiliares (p. ej., IPs de los NS).
- **Pie**: tiempo de respuesta y **servidor que respondió realmente**.

### Comandos de uso diario

```bash
dig +short ejemplo.com
```

- `+short` → muestra solo el valor; útil en scripts.

```bash
dig +noall +answer ejemplo.com
```

- `+noall` → oculta todas las secciones
- `+answer` → vuelve a mostrar solo ANSWER

```bash
dig ejemplo.com MX
```

- `MX` → servidores de correo del dominio.

```bash
dig ejemplo.com NS
```

- `NS` → servidores autoritativos.

```bash
dig ejemplo.com ANY
```

- `ANY` → pide todos los registros (muchos servidores lo bloquean hoy, RFC 8482).

```bash
dig -x 8.8.8.8
```

- `-x` → consulta inversa: construye la petición PTR (`8.8.8.8.in-addr.arpa`).

```bash
dig +trace ejemplo.com
```

- `+trace` → sigue la resolución desde la raíz: root → TLD → autoritativo.

```bash
dig +tcp ejemplo.com
```

- `+tcp` → fuerza TCP en lugar de UDP.

### `dig` en pentesting y bug bounty

Localiza los servidores autoritativos del objetivo:

```bash
dig +short NS objetivo.com
```

```bash
dig @ns1.objetivo.com objetivo.com TXT
```

- `@ns1.objetivo.com` → pregunta directamente al autoritativo (sin caché intermedia)
- `TXT` → SPF, verificaciones de servicios de terceros, pistas de infraestructura

```bash
dig axfr @ns1.objetivo.com objetivo.com
```

- `axfr` → solicita una transferencia de zona completa
- Si el servidor está mal configurado, devuelve **todos** los registros de la zona

Detecta CNAMEs hacia servicios externos (si el destino ya no existe → candidato a **subdomain takeover**):

```bash
dig +short CNAME sub.objetivo.com
```

---
## 3. Troubleshooting de `dig`

|Síntoma|Causa probable|Comprobación|
|---|---|---|
|`NXDOMAIN` con `SERVER: 8.8.8.8`|Faltó `@servidor` o se puso una IP como dominio|Revisar sintaxis|
|`timed out`|Servidor inalcanzable o que ignora la consulta|Añadir dominio correcto; `ip route get <IP>`; `ping`|
|`REFUSED`|El servidor no atiende consultas para esa zona o desde tu IP|Probar otro servidor o dominio|
|`NOERROR` con ANSWER: 0|El nombre existe pero no tiene ese tipo de registro|Cambiar el tipo|

```bash
ip route get <TARGET_IP>
```

- Debe salir `dev tun0`; si no, el `.ovpn` no enruta esa red.

---

## Key Takeaways

- `dig @servidor dominio tipo`: **`@` decide a quién, el dominio decide qué, el tipo decide qué registro.**
- Sin `@`, la consulta va al resolver por defecto; sin dominio, se pregunta por la raíz.
- La línea `SERVER` del pie es la primera comprobación ante cualquier resultado extraño.
- Un timeout no siempre es un problema de red: algunos servidores simplemente no responden a consultas que no les interesan.
- El flag `aa` distingue una respuesta autoritativa de una cacheada.
- En recon: NS → consulta directa a los autoritativos → intento de AXFR → revisión de TXT y CNAME.
