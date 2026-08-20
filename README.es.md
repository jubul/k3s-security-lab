# k3s Security Lab

*[English](README.md) · **Español***

Laboratorio de seguridad en Kubernetes construido desde cero sobre VMs KVM: cluster **k3s** multi-nodo con **admission control** (Kyverno), **detección en runtime** (Falco) y **centralización de alertas** en Elasticsearch/Kibana.

El objetivo del repositorio no es demostrar que el stack levanta, sino documentar **cómo se razonó cada decisión y cada fallo**. Todo el diagnóstico de los problemas encontrados está en [`docs/bitacora.md`](docs/bitacora.md), con causa raíz y evidencia — incluidos los errores propios.

---

## Estado actual

| # | Módulo | Estado |
|---|--------|--------|
| 0 | Prerequisitos KVM/libvirt en el host | ✅ Completo |
| 1 | Dos VMs Debian 13 provisionadas con cloud-init | ✅ Completo |
| 2 | Cluster k3s: control plane + nodo de trabajo | ✅ Completo |
| 3 | Fundamentos: Namespace, Deployment, Service, labels | ✅ Completo |
| 4 | Kyverno: admission control y políticas | ✅ Completo |
| 5 | Falco: detección en runtime con eBPF | ✅ Completo |
| 6 | Falcosidekick → Elasticsearch → Kibana | ✅ Completo |
| 7 | NetworkPolicies y Pod Security | 🔨 2 de 4 |
| 8 | Documentación final y diagramas | ⬜ Pendiente |

**Verificado:** la suite de casos negativos de las políticas pasa 5 de 5 ([`tests/`](tests/)); la regla propia de Falco detecta el robo de token de Service Account en dos imágenes con rutas de montaje distintas, con el ruido afinado de 804 alertas a 1; el pipeline indexa en Elasticsearch de punta a punta; y las políticas de Pod Security rechazan pods privilegiados y con root **sin dejar de permitir que el DaemonSet de Falco recree sus propios pods** — que es la prueba que valida el diseño de las excepciones.

---

## Arquitectura

```
        HOST (CachyOS · 16 cores · 15 GB RAM)
        kubectl · helm · navegador → Kibana
        ufw activo, con reglas explícitas para el lab
                        │
              virbr0 — NAT libvirt — 192.168.122.0/24
                        │
        ┌───────────────┴────────────────┐
        │                                │
┌───────────────────┐          ┌───────────────────┐
│    k3s-server     │          │     k3s-agent     │
│  192.168.122.6    │◄────────►│  192.168.122.128  │
│   3 GB · 2 vCPU   │  flannel │   6 GB · 4 vCPU   │
│                   │  VXLAN   │                   │
│  control plane    │          │  Elasticsearch    │
│  Kyverno          │          │  Kibana           │
│  Falco (DaemonSet)│          │  Falco (DaemonSet)│
└───────────────────┘          └───────────────────┘
   Debian 13.6 · kernel 6.12 · arranque UEFI · BTF presente
```

Falco corre como **DaemonSet**: una instancia por nodo, cada una observando las syscalls de su propio kernel.

---

## Stack y por qué

| Componente | Elección | Motivo |
|---|---|---|
| Cluster | **k3s** | Un binario con todo el control plane. Certificado por CNCF y usado en producción en edge, no un juguete. |
| Infraestructura | **libvirt/KVM** | Kernels reales por nodo. Falco observa syscalls: en contenedores compartiendo el kernel del host las reglas no se comportan como en producción. |
| Guest | **Debian 13 genericcloud** | Imagen mínima solo con virtio. Kernel 6.12 con `CONFIG_DEBUG_INFO_BTF=y`, requisito para eBPF sin compilar módulos. |
| Provisioning | **cloud-init (NoCloud)** | Nodos reproducibles desde YAML versionado, sin instalación manual. |
| Admission control | **Kyverno** | Las políticas son objetos de Kubernetes en YAML: se versionan en git y se explican sin traducir un lenguaje aparte. OPA/Gatekeeper tiene más poder y bastante más curva. |
| Runtime security | **Falco** | Estándar CNCF para detección basada en syscalls. |
| Alertas | **Falcosidekick + ELK** | Falco detecta; el pipeline convierte la detección en algo consultable e histórico. |

### Prevención y detección son capas distintas

El proyecto combina deliberadamente dos enfoques complementarios:

- **Kyverno actúa en la puerta.** Un pod rechazado en admission nunca ejecutó una línea de código. Es prevención, y solo alcanza para lo que se puede decidir mirando el manifest.
- **Falco actúa adentro.** Un contenedor legítimo que a las tres semanas abre una shell inesperada o lee `/etc/shadow` pasó todos los controles de admisión. Eso solo se ve observando comportamiento.

Ninguna de las dos capas reemplaza a la otra, y esa es la tesis del lab.

---

## Estructura del repositorio

```
├── docs/
│   ├── journal.md          Incidentes con diagnóstico y causa raíz (inglés)
│   ├── bitacora.md         Los mismos incidentes en español
│   └── estado-sesion.md    Estado de avance y punto de retomada
├── infra/
│   └── cloud-init/         user-data y meta-data de cada nodo (NoCloud)
├── manifests/              App de ejemplo y pod de simulación de ataques
├── policies/               ClusterPolicies de Kyverno
├── falco/
│   └── values.yaml         Configuración completa de Falco: driver, salida y reglas propias
├── elk/                    Elasticsearch (StatefulSet) y Kibana (Deployment)
├── netpol/                 NetworkPolicies de aislamiento
└── tests/                  Casos negativos de las políticas, con resultados esperados
```

---

## Reproducirlo

Requiere un host Linux con virtualización por hardware (`vmx` o `svm`), ~10 GB de RAM libre y 45 GB de disco.

### 1. Host: virtualización

```bash
sudo pacman -S --needed qemu-desktop libvirt virt-install dnsmasq openbsd-netcat libisoburn
sudo systemctl enable --now libvirtd.socket
sudo usermod -aG libvirt $USER      # requiere volver a iniciar sesión
sudo virsh net-start default && sudo virsh net-autostart default
```

En fish, para no repetir `-c qemu:///system` en cada comando:

```fish
set -Ux LIBVIRT_DEFAULT_URI qemu:///system
```

### 2. Firewall del host

Si usás `ufw`, sin estas reglas los guests no obtienen IP ni salida a internet ([bitácora #6](docs/bitacora.md)):

```bash
sudo ufw allow in on virbr0 to any port 67 proto udp comment 'libvirt DHCP'
sudo ufw allow in on virbr0 to any port 53 comment 'libvirt DNS'
sudo ufw route allow in on virbr0 out on <interfaz-de-salida> comment 'k3s lab egress'
sudo ufw reload
```

### 3. Imagen base y discos

```bash
cd /tmp
curl -LO https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2
curl -LO https://cloud.debian.org/images/cloud/trixie/latest/SHA512SUMS
sha512sum --ignore-missing -c SHA512SUMS      # verificar antes de ejecutar
sudo mv debian-13-genericcloud-amd64.qcow2 /var/lib/libvirt/images/

cd /var/lib/libvirt/images
sudo qemu-img create -f qcow2 -F qcow2 -b debian-13-genericcloud-amd64.qcow2 k3s-server.qcow2 20G
sudo qemu-img create -f qcow2 -F qcow2 -b debian-13-genericcloud-amd64.qcow2 k3s-agent.qcow2 20G
```

Los discos son overlays copy-on-write: arrancan en ~200 KB. **La imagen base no debe modificarse ni borrarse**, o se corrompen los dos nodos.

### 4. Semillas de cloud-init

```bash
cd infra/cloud-init
mkdir -p seed/server seed/agent
cp user-data-server.yaml seed/server/user-data
cp user-data-agent.yaml  seed/agent/user-data
printf 'instance-id: k3s-server-01\nlocal-hostname: k3s-server\n' > seed/server/meta-data
printf 'instance-id: k3s-agent-01\nlocal-hostname: k3s-agent\n'  > seed/agent/meta-data

xorrisofs -output /tmp/seed-server.iso -volid cidata -joliet -rock seed/server
xorrisofs -output /tmp/seed-agent.iso  -volid cidata -joliet -rock seed/agent
sudo mv /tmp/seed-*.iso /var/lib/libvirt/images/
```

Reemplazá la clave pública en los `user-data-*.yaml` por la tuya. El contrato de NoCloud es estricto: etiqueta de volumen `cidata` y archivos llamados exactamente `user-data` y `meta-data`, sin extensión.

### 5. Las VMs

```bash
sudo virt-install \
  --name k3s-server --memory 3072 --vcpus 2 --cpu host-passthrough \
  --boot uefi \
  --disk /var/lib/libvirt/images/k3s-server.qcow2,format=qcow2,bus=virtio \
  --disk /var/lib/libvirt/images/seed-server.iso,device=cdrom \
  --osinfo debian13 --network network=default,model=virtio \
  --graphics none --console pty,target_type=serial --import
```

Ídem para el agent con 6144 MB, 4 vCPU y sus propios archivos.

⚠️ **`--boot uefi` es obligatorio.** Con arranque BIOS (SeaBIOS) los guests no arrancan: el firmware entra en un ciclo de reintentos y la VM queda ejecutándose sin llegar nunca al kernel ([bitácora #5](docs/bitacora.md)).

⚠️ **No usar `--cloud-init` de `virt-install`.** Combinado con `--noautoconsole` aborta el primer arranque y descarta la ISO de semilla ([bitácora #4](docs/bitacora.md)). De ahí que la semilla se construya a mano en el paso 4.

### 6. k3s

En el server:

```bash
curl -sfL https://get.k3s.io | sudo sh -s - server \
  --node-ip 192.168.122.6 --tls-san 192.168.122.6
sudo cat /var/lib/rancher/k3s/server/node-token
```

En el agent:

```bash
curl -sfL https://get.k3s.io | sudo \
  K3S_URL=https://192.168.122.6:6443 K3S_TOKEN='<token>' \
  sh -s - agent --node-ip 192.168.122.128
```

En el host:

```bash
sudo pacman -S --needed kubectl helm
mkdir -p ~/.kube
ssh jubul@192.168.122.6 'sudo cat /etc/rancher/k3s/k3s.yaml' > ~/.kube/config
chmod 600 ~/.kube/config
sed -i 's|127.0.0.1|192.168.122.6|' ~/.kube/config
kubectl get nodes -o wide
```

### 7. Kyverno

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/ && helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace \
  --set admissionController.replicas=1 --set backgroundController.replicas=1 \
  --set cleanupController.replicas=1 --set reportsController.replicas=1

kubectl apply -f policies/
kubectl get clusterpolicy
```

Verificar la suite de casos negativos — los cuatro primeros deben ser rechazados y el quinto aceptado:

```bash
kubectl apply -f tests/ -n demo --dry-run=server
```

### 8. Falco

Toda la configuración vive en [`falco/values.yaml`](falco/values.yaml): el driver, el formato de salida y las reglas propias.

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts && helm repo update
helm install falco falcosecurity/falco -n falco --create-namespace -f falco/values.yaml
kubectl rollout status daemonset/falco -n falco
```

Para actualizar tras editar el values file, **sin `--reuse-values`** (el archivo ya contiene la configuración completa):

```bash
helm upgrade falco falcosecurity/falco -n falco -f falco/values.yaml
helm list -n falco      # verificar que la REVISION subió
```

Confirmar que cargó el driver correcto — tiene que aparecer `Opening 'syscall' source with modern BPF probe`:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=40 | grep -i "BPF probe"
```

`driver.kind: modern_ebpf` usa eBPF con **CO-RE** (*Compile Once, Run Everywhere*): el programa viene precompilado y se adapta al kernel local leyendo el BTF en `/sys/kernel/btf/vmlinux`. Sin BTF habría que compilar un módulo contra los headers de cada nodo, e instalar toolchains de compilación en nodos de producción es justamente lo que no se quiere.

**Probarlo con una intrusión simulada:**

```bash
kubectl apply -f manifests/04-intruso.yaml
kubectl exec -it intruso -n demo -- sh
# adentro:  cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

Y ver las alertas **sin filtrar por severidad** — filtrar por `warning` esconde las reglas `NOTICE`, que son la mayoría de la actividad de reconocimiento:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=200 \
  | grep '"rule":' \
  | jq -r '.priority + " | " + .rule + " | " + (.output_fields["k8s.pod.name"] // "-")'
```

### 9. Elasticsearch y Kibana

```bash
kubectl apply -f elk/
kubectl rollout status statefulset/elasticsearch -n elk
kubectl rollout status deployment/kibana -n elk
```

Falcosidekick se despliega como subchart desde `falco/values.yaml` (`falcosidekick.enabled: true`), lo que además cablea automáticamente el `http_output` de Falco. Verificar que el pipeline indexa:

```bash
kubectl exec -n elk elasticsearch-0 -- curl -s 'localhost:9200/_cat/indices/falco-*?v'
```

Kibana queda en **http://192.168.122.128:30601** vía NodePort. El data view debe usar el patrón **`falco-*`** con comodín: los índices son diarios (`falco-2026.08.20`), así que un patrón sin comodín funciona hoy y falla mañana.

### 10. NetworkPolicies y Pod Security

```bash
kubectl apply -f netpol/
kubectl apply -f policies/
```

Las políticas de Pod Security están en `Enforce`. La prueba que valida el diseño de las excepciones **no** es que rechacen un pod privilegiado, sino que Falco pueda seguir recreando los suyos:

```bash
kubectl delete pod -n falco -l app.kubernetes.io/name=falco
kubectl get daemonset falco -n falco    # debe volver a 2/2
```

Si la excepción está mal escrita, nada falla de forma visible: el DaemonSet simplemente no alcanza sus réplicas y el cluster queda sin detección.

---

## Decisiones de seguridad

Decisiones tomadas deliberadamente, con su justificación:

- **El kubeconfig se copia por SSH con `sudo cat`, no con `--write-kubeconfig-mode 644`.** Ese archivo es la credencial de administrador del cluster; aflojarle los permisos para ahorrar un `sudo` sería incoherente con el objetivo del proyecto. Queda en modo `600`.
- **`ufw` se mantiene activo.** Desactivarlo resolvía el problema de DHCP en un comando. En su lugar se agregaron reglas mínimas, comentadas y documentadas.
- **La imagen base se verifica con SHA512 antes de ejecutarla.** Verificar el artefacto que vas a correr es el punto de partida, no un trámite.
- **No se crea el usuario `debian` por defecto de la imagen.** Al declarar `users:` en cloud-init se reemplaza la lista por defecto: menos cuentas, menos superficie.
- **Imágenes con tag explícito y fijo, nunca `latest`.** Enforzado por política, no por convención.
- **La excepción de Falco en las políticas va por namespace Y label.** Exceptuar el namespace completo también habilitaría a `falcosidekick`, que no necesita ningún privilegio.
- **Las cargas propias se arreglaron en lugar de exceptuarse.** `kube-system` se exceptúa porque lo gestiona k3s; `demo` y `elk` se corrigieron. Si se exceptúa todo lo que falla, la política no protege nada.
- **El `node-token` de k3s es una credencial.** Quien lo tenga puede sumar nodos al cluster, es decir ejecutar cargas en él. No debe llegar al repositorio.

### Lo que este lab **no** es

Es un entorno de aprendizaje, no una referencia de producción. Limitaciones conscientes:

- Control plane de un solo nodo con SQLite, sin alta disponibilidad.
- Claves SSH sin passphrase y `sudo` sin contraseña en los nodos.
- Todos los controladores de Kyverno con una réplica, por restricción de RAM.
- **Elasticsearch sin autenticación ni TLS.** Hoy está protegido solo por aislamiento de red: quien comprometa un pod del namespace `falco` o el de Kibana mantiene acceso total de lectura y borrado. Pendiente habilitar `xpack.security`.
- **El SIEM vive dentro del cluster que monitorea.** Es cómodo para el lab y es un antipatrón en producción: la recolección debería ocurrir fuera de la frontera de confianza del sistema vigilado.
- Solo se controla el ingress de red; el egress queda abierto.
- Los nodos toman IP por DHCP: pueden cambiar si se recrean las VMs.

---

## Lo más interesante del proyecto

Si vas a leer una sola cosa, leé la [bitácora](docs/bitacora.md). Doce incidentes con su diagnóstico completo. Algunos ejemplos:

**Un guest que ejecutaba código sin arrancar nunca.** Las VMs figuraban en ejecución, consumían CPU y leían 3,37 GB de disco, pero no escribían un byte ni transmitían un paquete. Con el archivo de disco bloqueado por la VM en ejecución, el diagnóstico salió de los contadores del hipervisor y del monitor de QEMU: los registros del vCPU mostraban `CS=f000` en modo real de 16 bits, o sea código de BIOS. Diez minutos después del arranque el control nunca había pasado al kernel.

**Una política de admisión que pasaba su propia prueba y se podía evadir de dos formas.** Bloqueaba `nginx:latest` correctamente. Pero solo validaba `spec.containers`, dejando libres `initContainers` y `ephemeralContainers`; y como buscaba la cadena `:latest`, una imagen sin tag —que el runtime resuelve a `latest` igual— pasaba limpia. Al reescribirla, un error de capitalización (`initcontainers` en lugar de `initContainers`) hizo que la validación se **salteara en silencio**, en una política que reportaba `Ready`.

**Una regla de detección correcta y, tal como estaba, inservible.** La regla propia de robo de token de Service Account funcionaba: detectaba el ataque. También generaba **804 alertas con 2 verdaderos positivos** — un 0,25% de señal, porque todo componente de Kubernetes lee su propio token para autenticarse. Afinarla exigió antes arreglar la visibilidad (Falco solo emite en el JSON los campos que la plantilla `output` referencia, así que los campos de proceso venían vacíos), y trajo dos hallazgos sobre los campos disponibles: `proc.name` se trunca a 15 caracteres porque viene del `comm` del kernel, y `proc.exepath` colapsa los binarios multi-llamada — en Alpine `cat` reporta `/bin/busybox`. Ninguno de los dos sirve para todo.

**Un SIEM que el atacante podía borrar.** Auditando el cluster antes de escribir políticas apareció que cualquier pod alcanzaba Elasticsearch sin autenticar: se verificó creando y borrando un índice de prueba **desde el pod comprometido**, armando el request HTTP a mano con `nc` porque el `wget` de BusyBox no soporta `--method`. Es decir, un atacante podía eliminar los índices que registraron su propia intrusión, con las herramientas que ya venían en una imagen mínima. Se mitigó con NetworkPolicies; la lección de diseño es que el SIEM no debería vivir dentro de la frontera de confianza del sistema que vigila.

**Endurecer también apaga la telemetría.** Al pasar el pod de simulación de root a uid 1000, el mismo ataque dejó de generar dos alertas y generó una: `cat /etc/shadow` ahora da *permission denied*, y el macro `open_read` de Falco exige un descriptor abierto de verdad (`fd.num >= 0`), así que **un open fallido no matchea ninguna regla**. El ataque falló, pero el intento tampoco quedó registrado. Es una interacción entre la capa de prevención y la de detección que no aparece en los tutoriales.

**Un patrón que se repitió siete veces.** El instrumento de medición produciendo la conclusión: un `head -5` que llevó a afirmar que no había servidor DHCP cuando sí lo había; un `grep -i warning` que ocultó una detección de prioridad `NOTICE`; un `ignore_above: 256` que hace que una agregación de Kibana devuelva vacío sin error; un `jq` accediendo a un campo ausente que dejó un inventario de violaciones en blanco; un `tail -4` que escondió una de las dos reglas que sí habían rechazado un pod; un caso de prueba que no cubría la intersección de dos condiciones; y un error de capitalización que hacía que una validación se salteara sin fallar. **Ninguno de los siete avisó de que estaba recortando** — y varios ocurrieron después de haber identificado el patrón, que es justamente lo que lo hace interesante.

---

## Roadmap

- [x] Políticas de Kyverno con casos negativos versionados en `tests/`
- [x] Falco con modern eBPF, regla propia y validación con eventos reales
- [x] Pipeline Falcosidekick → Elasticsearch → Kibana
- [x] NetworkPolicies aislando la evidencia de las cargas del cluster
- [x] Pod Security: prohibir `privileged` y exigir `runAsNonRoot`, con excepciones quirúrgicas
- [ ] `automountServiceAccountToken: false` donde no se necesita: prevención que complementa la regla de robo de token
- [ ] Demostrar `Drop and execute new binary in container` — la detección que cubre al atacante en contenedores distroless, que debe traer su propio binario
- [ ] Completar Pod Security: `readOnlyRootFilesystem`, prohibir `hostPath`, `hostNetwork` y `hostPID`, exigir `seccompProfile: RuntimeDefault`
- [ ] NetworkPolicies de egress (hoy solo se controla el ingress)
- [ ] Habilitar `xpack.security` con TLS en Elasticsearch: hoy el aislamiento es solo de red
- [ ] Migrar `tests/` a `kyverno test` para correrlo en CI: con `kubectl apply --dry-run` el exit code queda invertido para casos negativos
- [ ] RBAC propio del namespace `falco`, que es la vía privilegiada hacia la evidencia
- [ ] Diagrama de arquitectura y modelo de amenazas
- [ ] Scripts `infra/lab-up.sh` y `lab-down.sh` para levantar y bajar el lab
- [ ] *(candidato)* GitOps con Argo CD o Flux, que elimina por diseño el desfase entre lo declarado en el repo y lo aplicado en el cluster
