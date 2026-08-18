# Estado de la sesión — punto de retomada

> Documento de handoff. Se actualiza al cerrar cada módulo para poder retomar sin recontextualizar.

**Última actualización:** 2026-08-17, cierre de la primera sesión (~16:30 a 20:00). Módulos 0 a 3 completos, módulo 4 en curso.
**Módulo en curso:** 4 (Kyverno) — con un agujero de política pendiente de cerrar, ver más abajo.

**Cluster:** k3s v1.36.3+k3s1, containerd 2.3.2-k3s2, 2 nodos `Ready`. Componentes en `kube-system`: CoreDNS, Traefik (+ svclb como DaemonSet), metrics-server, local-path-provisioner. `kubectl` y `helm` instalados en el host, kubeconfig en `~/.kube/config` (modo 600, apuntando a `192.168.122.6`).

---

## Objetivo del proyecto

Lab de seguridad en Kubernetes, documentado para publicar en GitHub. Doble propósito: pieza de portfolio y cerrar el gap de conocimiento en k8s (nivel de partida declarado: cero).

**Stack objetivo:** k3s (cluster) + Kyverno (admission control / policy as code) + Falco (runtime security vía eBPF) + Falcosidekick (router de alertas) + Elasticsearch/Kibana (almacenamiento y visualización).

**Modo de trabajo acordado:** guiado paso a paso. Se explica el concepto y el objetivo del ejercicio; Juan tipea todos los comandos y YAML. La bitácora la redacta Claude (pedido explícito del 2026-08-17).

---

## Infraestructura operativa

```
        HOST CachyOS "valkyrie" (15 GB RAM, 16 cores)
        kubectl / helm / navegador → Kibana
        ufw ACTIVO con reglas explícitas para el lab
                    │
              virbr0 (NAT libvirt, 192.168.122.0/24)
                    │
        ┌───────────┴────────────┐
        │                        │
 ┌──────────────────┐   ┌──────────────────┐
 │   k3s-server     │   │    k3s-agent     │
 │ 192.168.122.6    │◄─►│ 192.168.122.128  │
 │ 3 GB / 2 vcpu    │   │ 6 GB / 4 vcpu    │
 │ control-plane    │   │ ELK irá acá      │
 └──────────────────┘   └──────────────────┘
     Debian 13.6 · kernel 6.12.101 · UEFI · BTF presente
```

**Acceso:** `ssh jubul@192.168.122.6` y `ssh jubul@192.168.122.128` (clave ed25519 `~/.ssh/id_ed25519`, sin passphrase, sudo NOPASSWD).

⚠️ **Las IPs son por DHCP.** Si las VMs se recrean pueden cambiar. Consultar con `sudo virsh net-dhcp-leases default`. Pendiente evaluar pasarlas a estáticas vía `network-config` en la semilla NoCloud.

### Decisiones de diseño y por qué

- **VMs con libvirt/KVM**, no k3d ni k3s en el host: Falco necesita kernels reales para que las reglas se comporten como en producción, y el multi-nodo real es lo que le da peso al proyecto.
- **Arranque UEFI, no BIOS.** Obligatorio: con SeaBIOS los guests no arrancan (ver bitácora #5).
- **Reparto de RAM 3 + 6 GB.** El ELK es el componente pesado (Elasticsearch ~2 GB de heap, Kibana ~1 GB) y va en el agent. Se dejan ~6 GB al host, que es un desktop con navegador.
- **Debian 13 (trixie) genericcloud**, checksum SHA512 verificado. Kernel 6.12 con BTF, mejor soporte de eBPF que Debian 12.
- **Discos copy-on-write** (`qemu-img -b`) de 20 GB virtuales sobre una base de 328 MB. La imagen base **no se debe modificar ni borrar**: corrompería las dos VMs.
- **Semilla cloud-init construida a mano** (NoCloud, `xorrisofs -volid cidata`) en lugar de `virt-install --cloud-init`, que falla (bitácora #4). Queda versionada en `infra/cloud-init/`.
- **`ufw` se mantiene activo** con reglas mínimas y comentadas, en lugar de desactivarlo (bitácora #6).

## Estado del host

| Recurso | Valor |
|---|---|
| Distro | CachyOS (Arch), kernel 7.1.8-1-cachyos |
| CPU / RAM / Disco | 16 cores con `vmx` / 15 GB / 65 GB libres en btrfs |
| Virtualización | libvirt 12.6.0, QEMU 11.1.0, `<domain type='kvm'>` |
| Firmware VMs | edk2-ovmf 202605-1 (`OVMF_CODE.secboot.4m.fd`) |
| Snapshots | snapper + limine-snapper-sync (rollback desde el menú de Limine) |
| Firewall | ufw activo, reglas para `virbr0` (DHCP, DNS, egress vía `wlan0`) |
| `LIBVIRT_DEFAULT_URI` | `qemu:///system` (variable universal de fish) |

---

## Roadmap

| # | Módulo | Estado |
|---|--------|--------|
| 0 | Prerequisitos KVM/libvirt | **Completo** |
| 1 | Dos VMs Debian 13 con cloud-init | **Completo** |
| 2 | k3s: server + agent join, kubeconfig al host | **Completo** |
| 3 | Fundamentos k8s (namespace, deployment, service, labels) | **Completo** |
| 4 | Kyverno: admission control y políticas | **En curso** |
| 5 | Falco: modern eBPF y reglas custom | Pendiente |
| 6 | ELK + Falcosidekick: pipeline de alertas y dashboards | Pendiente |
| 7 | Repo y documentación final | Pendiente |

---

## Verificación de cierre del Módulo 1

Ejecutada por SSH en ambos nodos, todo OK:

| Chequeo | Resultado |
|---|---|
| hostname | `k3s-server` / `k3s-agent` (cloud-init aplicó correctamente) |
| kernel | 6.12.101+deb13-cloud-amd64 |
| **BTF en el guest** | **presente en ambos** → Falco con modern eBPF viable |
| cgroups | v2 (requisito de k3s) |
| cloud-init | `status: done` |
| swap | 0 entradas (requisito de k3s) |
| disco | 20 GB con 19 GB libres (growpart funcionó) |
| sudo | NOPASSWD operativo |
| salida a internet | HTTP 200 contra `get.k3s.io` |

---

## Pendientes abiertos

- Revisar los archivos `.pacnew` que dejó el upgrade del host.
- `paru -Syu` sin ejecutar: los paquetes AUR pueden necesitar rebuild tras el salto de glibc 2.43 → 2.44. **Revisar cada PKGBUILD antes**, por el incidente de paquetes maliciosos en el AUR del 2026-06-12.
- Evaluar IPs estáticas para los nodos vía `network-config` en la semilla NoCloud.
- `tcpdump` no está instalado en el host; sería útil para diagnóstico de red.
- Documentar como decisión deliberada que el usuario `debian` por defecto no se crea (menos superficie de ataque).

## Notas del Módulo 2

- El kubeconfig se trajo con `ssh ... 'sudo cat /etc/rancher/k3s/k3s.yaml'` en lugar de usar `--write-kubeconfig-mode 644`. Decisión deliberada: ese archivo es la credencial de administrador del cluster y aflojarle los permisos contradice el objetivo del proyecto. Queda en modo `600`.
- El `node-token` de `/var/lib/rancher/k3s/server/node-token` es una credencial: quien lo tenga puede sumar nodos al cluster. **No debe llegar al repositorio**; censurarlo si aparece en documentación.
- `br_netfilter` no está cargado en el host, por lo que el tráfico entre VMs del mismo bridge no pasa por netfilter. Por eso el join al 6443 y el VXLAN de flannel (UDP 8472) funcionan sin reglas de ufw.
- Se agregó `--tls-san 192.168.122.6` al server: sin esa SAN en el certificado, el acceso al API desde el host por IP falla la validación TLS.

## Notas del Módulo 3 (completo)

Escritos y aplicados en `manifests/`: Namespace `demo`, Deployment `web` (2 réplicas de `nginx:1.29-alpine` con requests/limits) y Service `web` de tipo ClusterIP. Revisados sin observaciones.

Conceptos verificados con las manos:
- **Reconciliación:** al borrar un pod a mano, el ReplicaSet crea el reemplazo. Nadie "reinicia" nada; un controlador nota la diferencia entre estado deseado y real.
- **Labels como cableado:** al cambiar `app=web` por `app=roto` en un pod, este desaparece del EndpointSlice y el ReplicaSet crea otro. Las relaciones en Kubernetes son consultas sobre etiquetas, no punteros.
- El scheduler repartió un pod por nodo. Flannel asigna una subred por nodo: `10.42.0.0/24` al server, `10.42.1.0/24` al agent.
- La label `pod-template-hash` la agrega el ReplicaSet; es el hash que aparece en el nombre de los pods y el mecanismo de las actualizaciones graduales.

## Notas del Módulo 4 (en curso)

**Instalado:** Kyverno v1.18.2 vía Helm, cuatro controladores (admission, background, reports, cleanup) con una réplica cada uno por restricción de RAM.

**Escrito:** `policies/01-disallow-latest-tag.yaml`, una ClusterPolicy con dos reglas (`require-image-tag` y `validate-image-tag`) cubriendo `containers`, `initContainers` y `ephemeralContainers`.

Aprendizajes de la API que conviene no olvidar:
- En Kyverno 1.18.2 `failureAction` va **a nivel de regla** (`spec.rules[].validate.failureAction`), no a nivel de spec. El campo legacy `spec.validationFailureAction` sigue existiendo y muestra su default `Audit`, lo que resulta contradictorio al inspeccionar el objeto: **manda el de la regla**.
- El *conditional anchor* `=(campo)` significa "si este campo existe, validalo". Sin él, un Pod sin `initContainers` fallaría por no declarar un campo que no le corresponde.
- **Autogen:** Kyverno genera solo las variantes de cada regla para controladores de pods (Deployment, DaemonSet, StatefulSet, Job, CronJob), con el prefijo `autogen-`. Sin eso el rechazo llegaría al crear el pod y el error quedaría en eventos en lugar de volver al `apply`.

### ⚠️ Agujero abierto — primera tarea al retomar

`policies/01-disallow-latest-tag.yaml` **línea 19**: dice `=(initcontainers)` (minúscula) y debe decir `=(initContainers)` (camelCase). Está corregido en la regla 2 y no en la regla 1.

Efecto verificado: un initContainer **sin tag** pasa la validación. Los demás casos bloquean bien.

```
container con latest        →  BLOQUEADO
container sin tag           →  BLOQUEADO
initContainer con latest    →  BLOQUEADO
initContainer sin tag       →  PASA        ← el agujero
pod correcto                →  PASA        (sin falsos positivos)
```

Corrección secundaria: los dos `message` dicen `nginx-1.29-alpine` con guión; va con dos puntos, `nginx:1.29-alpine`. En un mensaje cuyo objetivo es enseñar a poner el tag, el ejemplo mal escrito desorienta.

## Próximos pasos al retomar

1. **Corregir la línea 19** de la política y reverificar los cinco casos.
2. **Crear `tests/`** con un archivo por caso negativo (los cinco de la tabla de arriba), ejecutables juntos con `kubectl apply -f tests/ --dry-run=server`. Nota: la shell del entorno es **fish**, que no soporta heredocs de bash — los casos van como archivos, no como comandos.
3. **Commitear**, que el repositorio todavía no tiene ningún commit.
4. Seguir con más políticas de Pod Security (`runAsNonRoot`, `readOnlyRootFilesystem`, prohibir `privileged` y `hostPath`) y después el **Módulo 5: Falco**.

Los ocho incidentes resueltos están documentados en `bitacora.md` con causa raíz y evidencia.
