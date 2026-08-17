# Bitácora de incidentes

Registro de los problemas reales encontrados durante la construcción del lab, con el diagnóstico y la causa raíz de cada uno. El valor de este archivo no está en la lista de comandos que funcionaron, sino en el razonamiento que llevó a encontrar los que no.

Formato de cada entrada: síntoma → diagnóstico → causa raíz → solución → aprendizaje.

---

## #1 — Imposible instalar QEMU: 404 en todos los mirrors y ruptura de dependencias

**Fecha:** 2026-08-17 · **Módulo:** 0 (prerequisitos) · **Estado:** resuelto

### Síntoma

Al intentar instalar el stack de virtualización, `pacman` falló en dos etapas distintas:

1. Primero, error 404 al descargar `qemu-system-x86`, `qemu-common`, `edk2-ovmf`, `rdma-core` y otros, contra **todos** los mirrors de la lista (más de 40).
2. Después de refrescar la base de datos con `pacman -Syy`, el 404 desapareció y apareció un error distinto:

```
:: instalando nettle (4.0-1) se rompe la dependencia con «nettle=3.10.2», necesaria para el paquete lib32-nettle
:: instalando nettle (4.0-1) se rompe la dependencia con «libnettle.so=8-64», necesaria para el paquete wget
```

### Diagnóstico

Los dos errores son síntomas del mismo problema de fondo, en dos fases:

- Los **404** venían de una base de datos local desactualizada: pacman pedía versiones de paquetes que ya no existían en los mirrors. Que fallaran *todos* los mirrors y no uno era la pista: el problema estaba del lado del cliente, no del servidor.
- La **ruptura de dependencias** apareció recién con la base de datos al día. QEMU exigía `nettle 4.0`, pero `wget` y `lib32-nettle` instalados estaban enlazados contra `nettle 3.10.2`. Instalar solo QEMU habría dejado esos paquetes con un `.so` que ya no existe.

`pacman -Qu` reveló la magnitud real: **775 paquetes pendientes**, unos cuatro meses de atraso.

### Causa raíz

Arch Linux es rolling release y **no soporta actualizaciones parciales**. No existe forma de instalar un paquete nuevo sin actualizar el sistema entero, porque los paquetes de los repos se compilan siempre contra las versiones actuales de sus dependencias.

### Solución

Antes de tocar nada, se revisaron las news de Arch de abril a agosto de 2026 buscando intervenciones manuales obligatorias. Cuatro candidatas, ninguna aplicable: `virtualbox-ext-vnc` y `varnish` no estaban instalados, e `iptables` ya usaba el backend nft.

Procedimiento ejecutado:

```bash
# 1. Red de seguridad: snapshot antes de tocar el sistema
sudo snapper -c root create --description "pre-upgrade lab-k3s 775 pkgs" --cleanup-algorithm number

# 2. Keyring primero, y solo el keyring
sudo pacman -Sy archlinux-keyring cachyos-keyring

# 3. Actualización completa
sudo pacman -Syu

# 4. Reboot obligatorio
```

El keyring va primero y aparte por una razón concreta: el instalado era del 2026-04-20, y con claves de firma rotadas la verificación de los otros 774 paquetes falla a mitad de la transacción, dejando el sistema a medio actualizar. Es la **única excepción legítima** a la regla de no hacer actualizaciones parciales.

El reboot no es opcional: cambiaron kernel (7.0.1 → 7.1.8), systemd (260 → 261), glibc (2.43 → 2.44) y el driver NVIDIA (595.58.03 → 610.57.04). Tras el upgrade, los módulos del kernel en ejecución ya no existen en disco.

Resultado: sin incidentes. Kernel 7.1.8 arrancó limpio, `kvm_intel` cargado, NVIDIA 610 respondiendo y Plasma Wayland sin problemas.

### Aprendizaje

`pacman -Sy <paquete>` es un antipatrón: deja la base de datos adelantada respecto de los paquetes instalados y es la vía más directa a un sistema roto. **Siempre `-Syu`.** Y en un sistema con meses de atraso, el orden correcto es snapshot → keyring → upgrade completo → reboot.

### Nota de seguridad relacionada

Las news de Arch del 2026-06-12 reportan un incidente de adopciones maliciosas de paquetes en el AUR, sin listar paquetes concretos. Los 10 paquetes AUR de esta máquina son de fines de abril, **anteriores al incidente**, así que no hubo exposición. La exposición aparecería al actualizarlos: `paru -Syu` queda pendiente y requiere revisar cada PKGBUILD antes de ejecutarlo.

---

## #2 — `virsh net-list` devuelve una tabla vacía con la red funcionando

**Fecha:** 2026-08-17 · **Módulo:** 0 (prerequisitos) · **Estado:** resuelto

### Síntoma

Dos comandos consecutivos con resultados contradictorios:

```
virsh -c qemu:///system version   → responde bien (libvirt 12.6.0, QEMU 11.1.0)
virsh net-list --all              → tabla completamente vacía
ip -br addr show virbr0           → virbr0  DOWN  192.168.122.1/24
```

Una red inexistente no puede tener un bridge con IP asignada.

### Diagnóstico

La diferencia entre los dos comandos era el flag `-c`. Evidencia de que la red sí estaba operativa:

- `/etc/libvirt/qemu/networks/default.xml` existía, con timestamp del `net-start`.
- El symlink de autostart estaba puesto: `autostart/default.xml → ../default.xml`.
- Y lo definitivo: **dnsmasq corriendo** con `--conf-file=/var/lib/libvirt/dnsmasq/default.conf`. Ese proceso solo existe si la red `default` está activa.

### Causa raíz

Sin `-c`, `virsh` ejecutado por un usuario sin privilegios no se conecta al daemon del sistema: cae por defecto en **`qemu:///session`**, un daemon por usuario que es un espacio completamente separado. En `session` no existen redes virtuales, y no por un bug: un usuario común no puede crear bridges ni reglas de NAT. La tabla vacía era la respuesta correcta a la pregunta equivocada.

El `virbr0` en estado `DOWN` era una pista falsa aparte. El flag real era **`NO-CARRIER`**: un bridge sin ningún puerto conectado, como un switch encendido sin cables. Pasa a `UP` cuando la primera VM conecta su interfaz virtual.

### Solución

Fijar el URI por defecto de forma persistente (shell: fish):

```fish
set -Ux LIBVIRT_DEFAULT_URI qemu:///system
```

`-U` la hace universal (fish la persiste entre sesiones sin editar archivos de configuración) y `-x` la exporta al entorno de los procesos hijos.

### Aprendizaje

`qemu:///system` y `qemu:///session` son dos mundos distintos, no dos formas de ver el mismo. Un lab que necesita bridges y NAT vive obligatoriamente en `system`. Conviene fijar `LIBVIRT_DEFAULT_URI` desde el primer día para no diagnosticar fantasmas.

**Confirmación colateral de la teoría de socket activation:** `libvirtd.service` reportaba `enabled=disabled` pero `active=active`. Nadie lo habilitó al arranque; systemd lo despertó cuando el primer `virsh` tocó el socket, que es exactamente el comportamiento buscado al habilitar `libvirtd.socket` en lugar del `.service`.

---

## #3 — Corrección de diseño: el BTF que importa es el del guest

**Fecha:** 2026-08-17 · **Módulo:** 1 (VMs) · **Estado:** corregido antes de impactar

### Síntoma

No hubo fallo. Fue un error de razonamiento detectado antes de construir.

### Diagnóstico

Durante el relevamiento inicial del host se verificó la presencia de `/sys/kernel/btf/vmlinux` y se concluyó que Falco podría usar modern eBPF sin compilar módulos. La verificación era correcta, pero **aplicaba a la arquitectura descartada** (k3s directo sobre el host).

En la arquitectura elegida, Falco corre como DaemonSet **dentro de las VMs**. El kernel que necesita BTF es el del guest. El BTF del host es irrelevante para este camino.

### Solución

Mover el chequeo de BTF al interior de cada VM, y elegir la distro invitada en función de eso: **Debian 13 (trixie)** en lugar de Debian 12, por traer kernel 6.12 con mejor soporte de eBPF. Los kernels de Debian se compilan con `CONFIG_DEBUG_INFO_BTF=y`, así que el plan no cambia; sí cambia dónde se verifica.

### Aprendizaje

Al cambiar de arquitectura hay que reauditar los supuestos técnicos heredados, no solo los pasos. Una verificación correcta sobre el componente equivocado es indistinguible de una verificación fallida hasta que algo se rompe tres módulos más adelante.

---

## #4 — `virt-install --cloud-init` aborta el primer boot y descarta la semilla

**Fecha:** 2026-08-17 · **Módulo:** 1 (VMs) · **Estado:** resuelto

### Síntoma

Tras ejecutar `virt-install` para ambos nodos, los dominios quedaron definidos pero **apagados**, sin ninguna concesión de DHCP:

```
sudo virsh list --all           → k3s-server: apagado / k3s-agent: apagado
sudo virsh net-dhcp-leases default → tabla vacía
sudo virsh console k3s-agent    → error: El dominio no está ejecutándose
```

### Diagnóstico

Se cruzaron tres señales independientes, y las tres apuntaron al mismo lugar:

1. **El disco nunca se escribió.** `qemu-img info` reportó `disk size: 196 KiB` sobre un overlay de 20 GiB virtuales. Eso es metadata pura de qcow2: cero bytes escritos por el guest. Si cloud-init hubiera corrido, el `growpart` y el `apt update` habrían dejado decenas de megabytes.
2. **La ISO de semilla había desaparecido.** El XML del dominio mostraba una entrada de cdrom **sin `<source file>`**: el dispositivo existía, vacío.
3. **La VM sí se había iniciado al menos una vez.** La imagen base había cambiado de dueño a `libvirt-qemu:libvirt-qemu`, resultado del *dynamic ownership* de libvirt, que transfiere la propiedad al usuario de QEMU al arrancar un dominio.

Un guest que arranca, no escribe un byte, y se apaga dejando el cdrom vacío, no es un guest que falló al bootear: es un guest al que le cortaron el arranque desde afuera.

### Causa raíz

La combinación de flags **`--cloud-init` junto con `--noautoconsole`**. El flag `--cloud-init` hace que `virt-install` trate el arranque como una instalación en dos fases: fase 1 con la ISO de semilla montada, y al considerarla terminada desmonta la ISO, redefine el dominio y lo deja apagado. Con `--noautoconsole` no queda nada esperando el fin del boot, así que la fase 1 se da por concluida de inmediato: la VM muere antes de que Debian levante y la semilla se descarta.

Arrancar los dominios sin más no habría alcanzado: sin la ISO de semilla, cloud-init no tiene de dónde leer el usuario ni la clave SSH, y el resultado es un sistema al que no se puede entrar.

### Solución

Dejar de depender del comportamiento implícito de `virt-install` y construir la semilla explícitamente.

**El contrato de NoCloud:** al bootear, cloud-init busca un dispositivo con etiqueta de volumen **`cidata`** y lee de ahí dos archivos con nombres exactos, sin extensión: `user-data` y `meta-data`.

```bash
sudo virsh undefine k3s-server && sudo virsh undefine k3s-agent   # sin --remove-all-storage

mkdir -p seed/server seed/agent
cp user-data-server.yaml seed/server/user-data
cp user-data-agent.yaml  seed/agent/user-data
printf 'instance-id: k3s-server-01\nlocal-hostname: k3s-server\n' > seed/server/meta-data
printf 'instance-id: k3s-agent-01\nlocal-hostname: k3s-agent\n'  > seed/agent/meta-data

xorrisofs -output /tmp/seed-server.iso -volid cidata -joliet -rock seed/server
xorrisofs -output /tmp/seed-agent.iso  -volid cidata -joliet -rock seed/agent
sudo mv /tmp/seed-*.iso /var/lib/libvirt/images/
```

Y en `virt-install`, reemplazar `--cloud-init` por la ISO montada como un cdrom común:

```
--disk /var/lib/libvirt/images/seed-server.iso,device=cdrom
```

Los overlays se reutilizaron sin recrearlos: al estar en 196 KiB estaba probado que no tenían ni un byte de guest.

### Aprendizaje

Dos lecciones. La primera, técnica: los flags de conveniencia que ocultan una máquina de estados (`--cloud-init` y su instalación en dos fases) fallan de formas difíciles de leer cuando se combinan con flags que alteran ese flujo. Construir la semilla NoCloud a mano son cuatro comandos más y elimina toda la ambigüedad.

La segunda, de método: **ninguna de las tres señales era concluyente por separado.** El tamaño del disco podía ser un guest que arrancó y no escribió; el cdrom vacío, una VM que nunca lo tuvo; el cambio de ownership, un arranque exitoso. Las tres juntas dejaban una sola explicación posible. Cuando el síntoma es ambiguo, la salida es sumar señales independientes, no mirar la misma más de cerca.

### Beneficio colateral

La semilla explícita es reproducible y versionable: quedó como infraestructura declarada en el repositorio en lugar de ser el efecto secundario de un flag. Mejor para el proyecto que la solución que falló.

---

## #5 — Los guests nunca salen del firmware: SeaBIOS no arranca la imagen

**Fecha:** 2026-08-17 · **Módulo:** 1 (VMs) · **Estado:** solución en prueba

### Síntoma

Con la semilla NoCloud construida a mano, ambos dominios arrancaron y se mantuvieron en ejecución. El bridge pasó a `UP`, confirmando interfaces virtuales conectadas. Pero no hubo concesiones de DHCP:

```
sudo virsh list --all              → k3s-server: ejecutando / k3s-agent: ejecutando
ip -br addr show virbr0            → virbr0  UP  192.168.122.1/24
sudo virsh net-dhcp-leases default → tabla vacía
```

### Diagnóstico

El diagnóstico avanzó descartando capas de abajo hacia arriba, y **tres hipótesis intermedias resultaron falsas**. Vale registrarlas porque el camino equivocado también informa.

**1. La red del host: descartada.** Las interfaces `vnet2` y `vnet3` estaban conectadas al bridge en estado `UP,LOWER_UP`. Pero la tabla ARP de `virbr0` estaba vacía y `virbr0.status` pesaba 0 bytes: los guests no habían emitido **ni un paquete**. El problema no era que el DHCP fallara, era que del otro lado no había nadie hablando.

**2. Los contadores del hipervisor reorientaron todo.** En lugar de pelear con el lock de escritura de `qemu-img` sobre una VM en ejecución, se consultaron los contadores de libvirt:

```
virsh domblkstat k3s-server vda   → rd_bytes 3372751872 (3,37 GB) / wr_req 0
virsh domifstat  k3s-server vnet2 → tx_packets 0 / rx_packets 1
virsh cpu-stats  k3s-server       → cpu_time 554 s
```

Esto reveló un guest que **sí ejecutaba código y sí leía disco intensamente**, pero sin escribir un byte ni transmitir un paquete. No estaba colgado sin hacer nada: estaba trabajando mucho y avanzando nada.

**3. Emulación por software (TCG): descartada.** La combinación de CPU alto con progreso nulo sugería falta de aceleración por hardware. Falso: `<domain type='kvm'>`, `kvm support: enabled` en el monitor de QEMU, y `/dev/kvm` con permisos `666`.

**4. La evidencia decisiva: el monitor de QEMU.** Consultando los registros del procesador virtual:

```
virsh qemu-monitor-command k3s-server --hmp 'info registers'

CS = f000    base 000f0000     ← segmento de la ROM del BIOS
EIP = 0000f81a                 ← modo real de 16 bits
HLT = 0                        ← ejecutando activamente, no detenido
```

Diez minutos después de arrancar, el CPU seguía ejecutando código de BIOS en modo real de 16 bits. Nunca hubo transferencia de control al kernel. `HLT=0` descarta la hipótesis de "firmware detenido mostrando un cartel de error": está reintentando en loop, y ese loop explica tanto los 3,37 GB leídos como el CPU consumido.

Matiz importante para no sobreinterpretar: `CS=f000` indica ejecución de código del BIOS, que puede ser **SeaBIOS mismo o una llamada `INT 13h` emitida por GRUB** para leer disco. En ambos casos se está antes del kernel, en modo real.

**5. La imagen base no es la culpable.** Se inspeccionó la tabla de particiones extrayendo los primeros 4 MB con `qemu-img dd` y leyendo las entradas GPT crudas con `od`:

```
MBR offset 0:         eb 63 90            → boot.img de GRUB presente
Firma de partición:   55 aa               → válida
BIOS boot partition:  PRESENTE            → arranque BIOS soportado
EFI System Partition: PRESENTE            → arranque UEFI soportado
Partición 0:          4f68bce3-e8cd-...   → root Linux x86-64
```

La imagen `genericcloud` de Debian 13 es de arranque híbrido y soporta las dos vías.

### Causa raíz

SeaBIOS no logra completar el chainload de GRUB desde esta imagen, y entra en un ciclo de reintentos. El mecanismo exacto del loop quedó sin determinar: se tomó la decisión de no seguir con la arqueología, porque existía un camino alternativo soportado por la propia imagen y estrictamente mejor.

### Solución

Migrar el arranque de BIOS legacy a **UEFI con OVMF** (`edk2-ovmf`, ya instalado). Tres razones, en orden de peso:

1. La partición EFI está presente en la imagen, así que la vía está soportada de fábrica.
2. **OVMF escribe al puerto serie por defecto.** Con `--graphics none` no hay dispositivo de video, y SeaBIOS manda sus mensajes al framebuffer VGA — razón por la cual `virsh console` devolvía una pantalla en blanco y el fallo era invisible. UEFI resuelve el problema de observabilidad, no solo el de arranque.
3. UEFI es lo que usan los proveedores cloud reales para estas imágenes.

```bash
sudo virsh destroy k3s-server && sudo virsh undefine k3s-server --nvram
sudo virsh destroy k3s-agent  && sudo virsh undefine k3s-agent  --nvram
# y recrear agregando --boot uefi al virt-install
```

El `--nvram` en el `undefine` importa: con UEFI cada VM tiene su propio archivo de variables NVRAM, y dejar uno viejo colgado puede contaminar el arranque siguiente.

### Aprendizaje

**Cuando el guest no habla, hay que preguntarle al hipervisor.** El instinto de mirar el archivo de disco choca con el lock de escritura de la VM en ejecución; `domblkstat`, `domifstat`, `cpu-stats` y el monitor de QEMU dan la misma información sin tocar el archivo, y con más precisión. Los registros del vCPU convirtieron diez minutos de especulación en una respuesta inequívoca.

**La observabilidad es parte de la configuración, no un extra.** `--graphics none` sin considerar a dónde escribe el firmware creó un punto ciego exactamente donde ocurrió el fallo. La lección aplica directo al resto del proyecto: si Falco detecta algo y nadie recibe el evento, es el mismo error de diseño con otro disfraz.

**Descartar sirve aunque no encuentre la causa.** Tres hipótesis cayeron antes de la correcta — red del host, emulación TCG, imagen sin soporte BIOS — y cada una redujo el espacio de búsqueda. Y en el punto donde determinar el mecanismo exacto del loop de SeaBIOS habría costado más que rodearlo, corresponde rodearlo: el objetivo es un cluster funcionando, no un informe forense sobre firmware.

### Confirmación

Tras agregar `--boot uefi`, los contadores del hipervisor cambiaron de forma inequívoca:

| Métrica | SeaBIOS | UEFI/OVMF |
|---|---|---|
| Registros del vCPU | `CS=f000`, modo real 16 bits | 64 bits, direcciones `ffffffffa30...` (espacio de kernel) |
| `wr_req` | 0 | 1106 (317 MB escritos) |
| `tx_packets` | 0 | 26 |

Los 317 MB escritos corresponden al `growpart` + `resize2fs` expandiendo la partición raíz de 3 a 20 GB, lo que además **prueba que cloud-init se ejecutó** y que la semilla NoCloud fue leída correctamente.

---

## #6 — `ufw` descarta el DHCP de la red virtual

**Fecha:** 2026-08-17 · **Módulo:** 1 (VMs) · **Estado:** resuelto

### Síntoma

Con los guests arrancando correctamente por UEFI, seguía sin haber concesiones de DHCP. Los nodos quedaban booteados, en reposo y sin dirección IP.

### Diagnóstico

La secuencia que acotó el problema, descartando de abajo hacia arriba:

**1. Los guests no tienen IP, confirmado.** Un barrido de ping sobre `192.168.122.2-30` dejó todas las entradas ARP en `INCOMPLETE`: nadie respondió. No era un problema de registro de leases, era ausencia real de dirección.

**2. Un falso positivo propio, que vale registrar.** Un primer chequeo de sockets concluyó que no había servidor DHCP escuchando. Era un error del comando: el `grep` llevaba un `head -5` que cortó justo la línea relevante. Repetido sin truncar, `dnsmasq` sí estaba escuchando en `0.0.0.0%virbr0:67`. **Lección: al truncar la salida de un diagnóstico se puede fabricar la conclusión opuesta.**

**3. La pieza decisiva: dnsmasq no logueó nada.** Cuando dnsmasq recibe una solicitud escribe `DHCPDISCOVER(virbr0) <mac>` en el syslog. Cero entradas desde el arranque de las VMs. No es que rechazara los paquetes: **nunca los recibió**.

**4. Se acotó dónde se pierden.** El guest recibía los BPDUs de STP del bridge, lo que prueba que la capa 2 funciona en ambos sentidos (los BPDUs no atraviesan las reglas IP). Con el guest transmitiendo, el bridge operativo y dnsmasq escuchando, el paquete solo podía perderse en netfilter, entre `virbr0` y el socket.

**5. La causa.** Revisando los firewalls del host:

```
ufw    enabled=enabled    active=active
```

### Causa raíz

`ufw` estaba activo en el host con su política por defecto de **denegar todo el tráfico entrante**. Los DHCP DISCOVER llegaban a `virbr0`, entraban a netfilter y eran descartados antes de alcanzar a dnsmasq. Es una incompatibilidad conocida entre ufw y las redes virtuales de libvirt: libvirt instala sus propias reglas en la tabla `libvirt_network` de nftables (verificada como presente), pero eso no exime al tráfico de las reglas de ufw.

Efecto secundario que habría aparecido después: ufw también bloquea el forwarding, así que incluso con direcciones IP estáticas los guests no habrían tenido salida a internet, y no habrían podido descargar k3s ni las imágenes de los contenedores.

### Solución

Reglas explícitas y mínimas, en lugar de desactivar el firewall:

```bash
sudo ufw allow in on virbr0 to any port 67 proto udp comment 'libvirt DHCP'
sudo ufw allow in on virbr0 to any port 53 comment 'libvirt DNS'
sudo ufw route allow in on virbr0 out on wlan0 comment 'k3s lab egress'
sudo ufw reload
```

Las dos primeras habilitan los servicios que el host le presta a la red virtual (DHCP y DNS vía dnsmasq). La tercera habilita el forwarding del lab hacia la interfaz de salida, necesario para descargar paquetes e imágenes.

Los clientes DHCP de los guests ya habían agotado sus reintentos, así que hubo que reiniciar las VMs para que volvieran a solicitar dirección.

### Aprendizaje

**El firewall del host es parte de la topología del lab, no un detalle del entorno.** libvirt instala sus reglas de nftables correctamente y aun así el tráfico muere: dos sistemas de firewall coexistiendo, cada uno correcto por separado, incompatibles en conjunto. Vale anotarlo porque es exactamente la clase de interacción que después hay que razonar con las NetworkPolicies de Kubernetes.

**"Nadie logueó nada" es una señal, no una ausencia de señal.** Que dnsmasq no tuviera una sola entrada fue más informativo que cualquier mensaje de error: descartó de un golpe todo el procesamiento del servidor y movió la búsqueda aguas arriba, a la capa de filtrado.

**Se mantuvo el firewall activo.** Desactivar ufw habría resuelto el síntoma en un comando, y habría sido la decisión incorrecta en un proyecto de seguridad. Las reglas quedaron mínimas, con comentarios y documentadas.
