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

---

## #7 — La política de Kyverno que funcionaba y se podía evadir de dos formas

**Fecha:** 2026-08-17 · **Módulo:** 4 (Kyverno) · **Estado:** resuelto parcialmente (ver #8)

### Síntoma

No hubo síntoma. Ese es el punto de esta entrada.

La primera versión de la política que prohíbe el tag `latest` se aplicó sin errores, quedó `Ready`, y bloqueó correctamente el caso de prueba:

```
kubectl run malo --image=nginx:latest -n demo
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request
```

Todo indicaba que la política estaba lista.

### Diagnóstico

Se probaron casos negativos que la prueba original no cubría. Dos pasaron:

```
initContainers con nginx:latest  →  pod created   ← evadida
image: nginx  (sin tag)          →  pod created   ← evadida
containers con nginx:latest      →  BLOQUEADO     ← control OK
```

**Evasión 1 — contenedores no cubiertos.** El `pattern` validaba únicamente `spec.containers`. Un Pod tiene tres listas de contenedores: `containers`, `initContainers` (corren antes que los principales, con acceso a los mismos volúmenes) y `ephemeralContainers` (los inyecta `kubectl debug` en un pod ya en ejecución). Cualquiera con permiso de crear pods podía introducir la imagen que quisiera por las dos listas no validadas.

**Evasión 2 — el tag implícito.** `image: nginx` no contiene la cadena `:latest`, así que el glob `!*:latest` no la matchea. Pero el runtime la resuelve a `nginx:latest` de todos modos. La política cubría el error explícito y dejaba pasar el implícito, que es justamente el que se comete sin darse cuenta.

### Causa raíz

La política se validó contra el caso que se esperaba que fallara, no contra el conjunto de casos que debía cubrir. Una regla de admisión no se prueba demostrando que bloquea lo que quiere bloquear, sino verificando que **no exista camino** para lo que quiere prohibir.

### Solución

Reescritura con **dos reglas** en lugar de una:

1. `require-image-tag` — patrón `"*:*"`: exige tag explícito, cierra la evasión 2.
2. `validate-image-tag` — patrón `"!*:latest"`: prohíbe `latest`, la validación original.

Y cada regla cubre las tres listas de contenedores, usando el *conditional anchor* de Kyverno:

```yaml
pattern:
  spec:
    containers:
      - image: "*:*"
    =(initContainers):
      - image: "*:*"
    =(ephemeralContainers):
      - image: "*:*"
```

La sintaxis `=(campo)` significa "si este campo existe, validalo". Sin el anchor, un Pod sin `initContainers` fallaría la validación por no declarar un campo que no le corresponde tener, y la política rompería todos los deploys legítimos del cluster.

Hicieron falta dos reglas y no una: las dos validaciones aplican sobre la misma clave (`image`), y en un solo `pattern` YAML no puede haber dos veces la misma clave (ver #8).

### Aprendizaje

**Una política de seguridad se prueba con los casos que deberían fallar, no con los que deberían pasar.** El caso feliz confirma que la regla está instalada; los casos negativos confirman que sirve.

**Enumerar la superficie completa antes de escribir la regla.** Las dos evasiones venían del mismo error: asumir que "el contenedor de un pod" es un solo lugar. Son tres, y dos son las interesantes para un atacante justamente porque se olvidan.

**Observación colateral: autogen.** Kyverno generó automáticamente `autogen-resource-rule`, una variante de la regla para controladores de pods (Deployment, DaemonSet, StatefulSet, Job, CronJob). Sin eso el rechazo ocurriría cuando el ReplicaSet intentara crear el pod, y el error quedaría en eventos en lugar de volver a la terminal de quien hizo el `apply`.

---

## #8 — Tres bugs silenciosos en la política endurecida

**Fecha:** 2026-08-17 · **Módulo:** 4 (Kyverno) · **Estado:** dos resueltos, uno abierto

### Síntoma

La segunda versión de la política, escrita para cerrar las evasiones de #7, tenía tres defectos. Solo uno se manifestó como error visible.

### Diagnóstico

**Bug 1 — nombre inválido (falló ruidosamente).** El objeto se llamaba `ExecutionPolicy`:

```
The ClusterPolicy "ExecutionPolicy" is invalid: metadata.name: Invalid value:
"ExecutionPolicy": a lowercase RFC 1123 subdomain must consist of lower case
alphanumeric characters, '-' or '.'
```

Los nombres de objetos de Kubernetes son subdominios DNS RFC 1123 y no admiten mayúsculas. No es una convención de estilo: esos nombres terminan formando parte de nombres DNS reales dentro del cluster. Este bug fue el benigno, porque impidió que el archivo se aplicara.

**Bug 2 — clave YAML duplicada (falló en silencio).** Dentro del mismo `pattern` la clave `containers` aparecía dos veces, una con `"*:*"` y otra con `"!*:latest"`. En un mapping YAML una clave repetida no produce error: **la última sobrescribe a la primera**. El chequeo de tag explícito sobre `containers` se descartaba sin aviso.

**Bug 3 — error de capitalización (falló en silencio).** Se escribió `=(initcontainers)` en minúsculas; el campo real de la API es `initContainers`. Combinado con la semántica del anchor, el efecto es particularmente engañoso: `=(campo)` valida *si el campo existe*, y un campo llamado `initcontainers` no existe nunca en un Pod. La regla **no fallaba: se salteaba**.

El escenario peligroso era la suma de 1, 2 y 3. Corrigiendo solo el nombre, la política se habría aplicado, habría reportado `Ready`, habría bloqueado `nginx:latest` en `containers` — y habría dejado los dos agujeros de #7 intactos, con toda la apariencia de estar funcionando.

### Bug residual (abierto)

Tras corregir el nombre y separar las dos reglas, quedó una divergencia entre ellas:

```yaml
- name: require-image-tag        # regla 1
      =(initcontainers):         # minúscula  ← sin corregir
- name: validate-image-tag       # regla 2
      =(initContainers):         # camelCase  ← corregida
```

El typo se corrigió en la regla 2 y no en la regla 1, con un efecto muy preciso:

```
initContainer con latest    →  BLOQUEADO   (lo atrapa la regla 2)
initContainer SIN tag       →  created     ← agujero abierto
container con latest        →  BLOQUEADO
container sin tag           →  BLOQUEADO
pod correcto                →  created     (sin falsos positivos)
```

El agujero sobrevive exactamente en la intersección de las dos condiciones: initContainer **y** tag omitido. Pendiente: corregir la línea 19 de `policies/01-disallow-latest-tag.yaml`.

### Aprendizaje

**Los bugs que fallan ruidosamente son los baratos.** De los tres, el único que se manifestó fue el nombre inválido. Los otros dos habrían quedado en el repositorio, en una política declarada `Ready`, con las validaciones desactivadas en silencio.

**Una divergencia por copy-paste entre dos reglas es invisible en la revisión del código.** Las dos reglas *parecen* iguales al leerlas. La única forma de encontrar la diferencia fue el caso de prueba que combinaba las dos condiciones.

**De ahí la decisión de versionar los casos negativos** en `tests/`, un archivo por caso, ejecutables en conjunto con `kubectl apply -f tests/ --dry-run=server`. Y el paso siguiente natural es el CLI `kyverno test`, que declara el resultado esperado de cada caso y corre sin cluster: eso permite meter las políticas en CI y evitar que un pull request mergee una regla que abre un agujero.

**Nota de método:** en esta sesión se instruyó a usar heredocs de bash (`<<'EOF'`) para las pruebas, sintaxis que la shell del entorno (fish) no soporta. El incidente reorientó la solución hacia algo mejor: los casos de prueba como archivos versionados en el repositorio en lugar de comandos escritos en la terminal y perdidos.

---

## #9 — Una detección que sí ocurrió y se dio por ausente: el filtro por prioridad

**Fecha:** 2026-08-18 · **Módulo:** 5 (Falco) · **Estado:** resuelto

### Síntoma

Tras instalar Falco y simular una intrusión (`kubectl exec` a un pod, `cat /etc/shadow`), se observó **una sola alerta**: la lectura del archivo sensible. La apertura de la shell no aparecía, pese a ser el evento más evidente de los dos.

La primera hipótesis fue que la regla no estaba en el set cargado: en versiones recientes de Falco el ruleset por defecto se redujo y varias reglas ruidosas se movieron a los sets `incubating` y `sandbox`, que no se cargan solos.

### Diagnóstico

La hipótesis era falsa. Inspeccionando el set efectivamente cargado dentro del pod:

```
reglas totales cargadas: 25
- rule: Run shell untrusted
- rule: Terminal shell in container    ← presente
```

La regla estaba cargada. Extrayéndola completa apareció la causa:

```yaml
- rule: Terminal shell in container
  condition: spawned_process and container and shell_procs and proc.tty != 0 and container_entrypoint ...
  priority: NOTICE          ← acá
```

El comando usado para revisar las alertas era `kubectl logs ... | grep -i warning`. La regla tiene prioridad **NOTICE**, así que el filtro la descartaba. Listando sin filtrar:

```
Notice   | Terminal shell in container      | intruso | sh
Warning  | Read sensitive file untrusted    | intruso | cat /etc/shadow
```

Las dos alertas habían disparado desde el primer momento.

### Causa raíz

Un filtro aplicado sobre la propia salida de diagnóstico, sin verificar que el rango de severidades filtrado incluyera lo que se estaba buscando.

Escala de prioridades de Falco, de mayor a menor: `EMERGENCY`, `ALERT`, `CRITICAL`, `ERROR`, `WARNING`, `NOTICE`, `INFORMATIONAL`, `DEBUG`.

### Aprendizaje

**Es el tercer incidente del proyecto con la misma forma:** el `head -5` que llevó a concluir que no había servidor DHCP (#6), el caso de prueba que no cubría la intersección de dos condiciones (#8), y este filtro por severidad. En los tres casos **el instrumento de medición produjo la conclusión**, y en ninguno avisó de que estaba recortando.

La versión aplicada a este dominio es directa y grave: un SOC que filtra su dashboard por `severity >= WARNING` se pierde toda la actividad de reconocimiento, que es en su mayoría NOTICE — y el reconocimiento es lo que precede al resto. Un pipeline de detección que descarta señal por defecto es indistinguible de uno que no detecta.

---

## #10 — La regla de robo de token: alias de rutas, campos truncados y 804 falsos positivos

**Fecha:** 2026-08-18 · **Módulo:** 5 (Falco) · **Estado:** resuelto

### Contexto

Regla propia para detectar lectura del token de Service Account (`/var/run/secrets/kubernetes.io/serviceaccount/token`), el vector clásico para pasar de "ejecución de código en un contenedor" a "acceso al API server con la identidad del pod". Es un caso que **el admission control no puede cubrir**: el token está montado legítimamente por el kubelet y leerlo es una operación normal para cualquier SDK. Solo se distingue por comportamiento.

Aparecieron tres problemas distintos, en tres capas distintas.

### Problema 1 — Alias de rutas: dos caminos al mismo archivo

La condición original usaba `fd.name startswith "/var/run/secrets/kubernetes.io/serviceaccount/"`.

Se detectó que la ruta del montaje difiere según la imagen:

```
busybox:1.37        /var/run es directorio real   →  /var/run/secrets/kubernetes.io/serviceaccount
nginx:1.29-alpine   /var/run → ../run (symlink)   →  /run/secrets/kubernetes.io/serviceaccount
```

Mismo `mountPath` en la spec del pod, mismo kubelet, y el kernel los monta en rutas distintas porque el runtime resuelve el symlink de la imagen antes de crear el punto de montaje.

**Corrección de un diagnóstico intermedio erróneo.** Se concluyó de ahí que la regla detectaría el robo en el pod busybox y no en el pod nginx. Verificado después, eso era falso: `fd.name` reporta **la ruta tal como el proceso la abrió**, no el punto de montaje resuelto, así que `cat /var/run/...` en el pod alpine también matchea.

Pero la corrección seguía siendo necesaria, por un motivo más serio: **el atacante elige la ruta**. Comprobado directamente:

```
cat /var/run/secrets/.../token  →  fd.name = /var/run/secrets/.../token
cat /run/secrets/.../token      →  fd.name = /run/secrets/.../token      ← mismo pod
```

Con la regla original, el segundo comando pasaba invisible. No es un problema de qué imagen se usa, es que **dos rutas distintas llevan al mismo archivo y una detección basada en path tiene que enumerarlas todas**: la clase de evasión por aliasing de rutas.

Solución, con el operador `pmatch` de Falco, que entiende jerarquía de directorios en lugar de comparar cadenas:

```yaml
and fd.name pmatch (/var/run/secrets/kubernetes.io/serviceaccount, /run/secrets/kubernetes.io/serviceaccount)
```

### Problema 2 — El output template define el esquema de datos

Al intentar afinar la regla, los campos de proceso venían vacíos:

```
proc.name → null    proc.pname → null    proc.tty → null
proc.cmdline → "cat /var/run/secrets/..."    ← este sí
```

**Falco solo incluye en el JSON los campos referenciados en la plantilla `output`.** `proc.cmdline` estaba porque figuraba en el template; los demás no existían en el evento.

La consecuencia va más allá del debugging: **no se puede filtrar en un SIEM por un campo que nunca se emitió**. El `output` no es cosmética, es la definición del esquema. Si falta un campo, hay que modificar la regla y esperar a que el evento vuelva a ocurrir.

De ahí la regla de orden: **primero la visibilidad, después el afinado.** No se afina lo que no se ve.

### Problema 3 — 804 alertas, 2 reales

Con los campos visibles, el volumen quedó a la vista:

```
804 alertas totales
  579  namespace kyverno
  222  namespace kube-system
    3  namespace demo    ← 2 son los ataques simulados
```

Señal/ruido: **0,25%**. Una regla técnicamente correcta y operativamente inútil: nadie encuentra esas dos líneas.

El discriminador salió de medir, no de suponer:

```
8 reports-control     ← binarios de las apps leyendo su propio token para autenticarse
6 kyverno
5 background-cont
4 cleanup-control
2 metrics-server
1 traefik
1 cat                 ← el ataque
```

**Hallazgo colateral: `proc.name` se trunca a 15 caracteres.** Viene del campo `comm` del kernel, que tiene ese límite duro — de ahí `reports-control` y `background-cont`. Un filtro por `proc.name = "background-controller"` no habría matcheado nunca, produciendo el cuarto fallo silencioso del proyecto. Para nombres largos corresponde `proc.exepath`.

**Y el contraejemplo, en la otra dirección:** en Alpine `cat` es un symlink a busybox, así que el mismo ataque reporta `proc.name=cat` pero `proc.exepath=/bin/busybox`. Un filtro por `proc.exepath in (/bin/cat)` se habría perdido ese pod. Ninguno de los dos campos es universalmente correcto: `proc.name` se trunca, `proc.exepath` colapsa los binarios multi-llamada. Hay que saber de qué fallo se está cuidando cada regla.

Solución aplicada — filtrar por quién lee, no por dónde:

```yaml
- list: token_reader_tools
  items: [cat, head, tail, more, less, base64, xxd, od, strings, curl, wget, nc, socat, tar, cp, dd]

- macro: suspicious_token_reader
  condition: (proc.name in (token_reader_tools) or proc.name in (shell_binaries))
```

Resultado: **de 804 alertas a 1**, conservando la detección en las dos imágenes.

Se descartó explícitamente la alternativa de excluir por namespace (`not k8s.ns.name in (kube-system, kyverno)`): es más simple y peor, porque vuelve invisible a un atacante que consiga ejecución en `kube-system`, que es justamente donde más costaría.

### Limitación declarada

La regla filtra por nombre de proceso, así que **no detecta a un atacante que lea el token desde su propio binario** (un implant en Go, Python o similar). Cubre el acceso oportunista con herramientas de línea de comandos, no al adversario preparado. Queda escrito en el `desc` de la regla: un `desc` honesto sobre los límites vale más que uno que promete cobertura total.

### Aprendizaje

**Afinar es la mitad del trabajo, y se hace con datos.** El discriminador correcto no se adivina: se emite el campo, se mide la distribución y se decide sobre la evidencia. Las dos hipótesis que se plantearon antes de medir (que el ruido venía de los binarios de las apps, y que el problema del symlink afectaba por imagen) resultaron una correcta y una equivocada.

**Una detección sin afinar no es una detección a medias, es ninguna.** 804 eventos con 2 verdaderos positivos no se triagean: se ignoran. Y una regla que se ignora es peor que no tenerla, porque genera la sensación de estar cubierto.

---

## #11 — La evidencia expuesta: cualquier pod podía borrar los índices de Falco

**Fecha:** 2026-08-20 · **Módulo:** 7 (seguridad en k8s) · **Estado:** mitigado

### Síntoma

Ninguno. Se encontró auditando el estado del cluster antes de escribir políticas, no por un fallo.

### Diagnóstico

Elasticsearch se despliega con `xpack.security.enabled=false` (decisión consciente del módulo 6, documentada). Sin NetworkPolicies, eso significa que el Service `elasticsearch.elk.svc.cluster.local:9200` era alcanzable desde **cualquier pod del cluster**, sin autenticar.

Verificado desde el pod de simulación de ataque, en tres pasos:

```
GET  /_cat/indices    → lista todos los índices, incluido falco-2026.08.20
PUT  /prueba-netpol   → índice creado
DELETE /prueba-netpol → índice borrado
```

Se usó un índice de prueba descartable; no se tocó `falco-*`.

**Detalle relevante de la técnica:** el `wget` de BusyBox no soporta `--method`, así que las escrituras se hicieron con `nc`, armando el request HTTP a mano:

```sh
printf 'DELETE /prueba-netpol HTTP/1.1\r\nHost: es\r\nConnection: close\r\n\r\n' | nc elasticsearch.elk.svc.cluster.local 9200
```

*Living off the land* con lo que trae una imagen mínima. "La imagen es chica" no es un control de seguridad.

### Causa raíz

El SIEM vivía dentro de la frontera de confianza del sistema que vigila, sin autenticación ni aislamiento de red. Un atacante que compromete cualquier contenedor podía **borrar los índices que registraron su propia intrusión**.

### Solución

Tres NetworkPolicies en el namespace `elk`:

1. `default-deny-ingress` con `podSelector: {}` — cierra el namespace entero.
2. `allow-elasticsearch` — puerto 9200 solo desde el namespace `falco` (vía `namespaceSelector` sobre la label automática `kubernetes.io/metadata.name`) **o** desde los pods con label `app: kibana`.
3. `allow-kibana` — puerto 5601 desde `ipBlock: 192.168.122.0/24`.

El `ipBlock` es necesario porque el tráfico que entra por el NodePort **no viene de un pod**: kube-proxy lo hace SNAT a la IP del nodo, así que ningún `podSelector` lo matchea.

**Verificación, con control positivo:**

```
DNS desde el pod atacante   resuelve a 10.42.1.29     ← el bloqueo no es un fallo de DNS
pod atacante → 10.42.1.29   BLOQUEADO
falcosidekick → ES          ALCANZABLE                ← control positivo
Kibana desde el host        HTTP 200
logs de Kibana              sin errores contra ES
```

El control positivo es la parte que valida la prueba: si `falcosidekick` también hubiera quedado bloqueado, no se sabría si la política discrimina o si simplemente rompió todo.

### Aprendizaje

**La semántica de `from` en una NetworkPolicy depende de un guión.** Dos ítems separados en la lista son **OR**; dos selectores bajo el mismo ítem son **AND**:

```yaml
from:
  - namespaceSelector: {...}     # OR
  - podSelector: {...}
from:
  - namespaceSelector: {...}     # AND — aquí sería el conjunto vacío
    podSelector: {...}
```

**Posible fallo silencioso a tener en cuenta:** k3s trae su controlador de NetworkPolicy embebido, pero si se arranca con `--disable-network-policy` el API server **acepta** los objetos y nadie los aplica. Política creada, cero efecto, ningún error. Solo la prueba de conectividad lo detecta.

### Lo que esto no resuelve

- **Sin control de egress.** Se cerró quién entra a `elk`; los pods siguen pudiendo salir a cualquier parte.
- **Elasticsearch sigue sin autenticación.** El control es solo de red: quien comprometa un pod del namespace `falco` o el de Kibana mantiene acceso total de lectura y borrado.
- **El namespace `falco` es ahora una vía privilegiada** hacia la evidencia, además de correr el pod más privilegiado del cluster. Necesita su propio RBAC.

---

## #12 — Endurecer sin matar la herramienta de seguridad

**Fecha:** 2026-08-20 · **Módulo:** 7 (seguridad en k8s) · **Estado:** resuelto

### Contexto

Aplicar Pod Security con Kyverno: prohibir `privileged` y exigir `runAsNonRoot`. El problema de fondo es que **Falco es el componente más privilegiado del cluster**: su DaemonSet corre `privileged: true` y monta del host `/boot`, `/lib/modules`, `/usr`, `/etc`, `/sys/kernel`, `/proc` y todos los sockets de runtime de contenedores. Una política restrictiva mal diseñada impide que el DaemonSet recree sus pods, y **el resultado es quedarse sin detección sin que nada falle de forma visible**: el DaemonSet simplemente no alcanza las réplicas deseadas.

### El inventario primero

La política se aplicó primero en modo **`Audit`**, que no rechaza nada y escribe los resultados en los `policyreport`. Resultado sobre el cluster completo:

```
29 violaciones en 4 namespaces
  require-run-as-nonroot   kube-system 9 · falco 7 · elk 5 · demo 5
  disallow-privileged      falco 3
```

Dos conclusiones: **nada** en el cluster declaraba `runAsNonRoot`, y el privilegio estaba concentrado en un único componente identificado y justificado.

Ese orden —auditar, medir, decidir excepciones, después enforce— es el mismo bucle que se usó para afinar la regla de Falco en #10: primero visibilidad, después restricción. Enforce a ciegas sobre un cluster con cargas es cómo se causan incidentes con la excusa de prevenirlos.

### Criterio de las excepciones

**Para `disallow-privileged`, excepción quirúrgica.** Lo obvio sería exceptuar el namespace `falco` completo; es incorrecto, porque ahí también vive `falcosidekick`, que no necesita ningún privilegio. Se excluye por namespace **y** label:

```yaml
exclude:
  any:
    - resources:
        namespaces: [falco]
        selector:
          matchLabels:
            app.kubernetes.io/name: falco
```

**Para `require-run-as-nonroot`, criterio de propiedad.** `kube-system` no es propio: CoreDNS, local-path-provisioner y svclb los gestiona k3s y el próximo upgrade sobrescribe cualquier cambio. Se exceptúan y se documenta. Pero `demo` y `elk` sí son propios, y ahí la respuesta correcta **no es la excepción sino arreglar las cargas**: si se exceptúa todo lo que falla, la política no protege nada y solo produce sensación de cumplimiento.

### Los cinco defectos encontrados en la revisión

1. **Los bloques `exclude` estaban ausentes por completo, con las dos reglas ya en `Enforce`.** El defecto grave: aplicarla habría impedido que Falco recreara sus pods.
2. **La imagen de `web` decía `nginx-unprivileged:1.29-alpine`, sin el prefijo `nginxinc/`.** Verificado contra el registro: `docker.io/library/nginx-unprivileged` devuelve `object not found`. `ImagePullBackOff` garantizado.
3. **El Deployment `web` no tenía `securityContext`.**
4. **Ese `securityContext` había terminado como campo raíz de `03-demo-service.yaml`**, donde no hace absolutamente nada. Se fue de archivo.
5. **Kibana tampoco tenía `securityContext`**, así que habría sido rechazado en su próximo reinicio.

Y otra vez el desfase entre archivo editado y cluster: la política no estaba aplicada. Cuarta aparición en el proyecto; esta vez evitó el daño.

### Un hallazgo sobre `runAsNonRoot`

Elasticsearch y Kibana **ya corrían como uid 1000**: los cambios en ellos fueron puramente declarativos. Eso no los vuelve superfluos. `runAsNonRoot: true` hace que el **kubelet se niegue a arrancar** el contenedor si el usuario efectivo de la imagen resuelve a root. Sin declararlo, hoy corre como 1000 porque la imagen lo dice, y un pull de una versión futura que cambie el `USER` devuelve el proceso a root sin que nadie se entere. Declararlo convierte una costumbre en una garantía verificada.

El corolario práctico apareció al recrear el pod de simulación: `runAsNonRoot: true` **por sí solo no alcanza** para una imagen que declara `USER root`, como busybox. El kubelet rechaza el arranque con `CreateContainerConfigError` ("container has runAsNonRoot and image will run as root"). Hay que indicar explícitamente `runAsUser`.

### Verificación

```
pod privilegiado          RECHAZADO por disallow-privileged Y require-run-as-nonroot
pod sin runAsNonRoot      RECHAZADO
pod que cumple            ACEPTADO
Falco tras borrar pods    2/2 listos, modern BPF probe, custom-rules.yaml cargado
web                       HTTP 200 como uid=101(nginx)
namespace demo            sin incumplimientos
```

La prueba de borrar los pods de Falco es la que valida el diseño de la excepción. Sin ella, el fallo aparecería semanas después, en el próximo reinicio de un nodo.

### Efecto de segundo orden: endurecer también reduce la telemetría

Con el pod de simulación corriendo como uid 1000 en lugar de root, el mismo ataque produce un resultado distinto:

```
cat /var/run/secrets/.../token   → leído       → ALERTA de Falco
cat /etc/shadow                  → Permission denied → SIN alerta
```

Antes generaba dos alertas; ahora genera una. La razón es que el macro `open_read` de Falco exige `fd.num >= 0`, es decir un descriptor abierto de verdad: **un open fallido no matchea**. El endurecimiento bloqueó el ataque *y* eliminó el registro del intento.

No es un problema —el ataque no funcionó— pero es una interacción entre las capas de prevención y detección que conviene tener presente: **al endurecer las cargas, el dashboard ve menos, y parte de lo que deja de ver son intentos fallidos que sí tienen valor forense.** Detectar intentos rechazados requiere reglas construidas con otras condiciones.

### Pendiente documentado

`falcosidekick` puede correr non-root: corresponde configurarlo vía `falco/values.yaml` y estrechar la exclusión de `require-run-as-nonroot` al label del DaemonSet, en lugar de exceptuar el namespace completo.
