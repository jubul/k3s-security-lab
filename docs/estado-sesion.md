# Estado de la sesión — punto de retomada

> Documento de handoff. Se actualiza al cerrar cada módulo para poder retomar sin recontextualizar.

**Última actualización:** 2026-08-18, cierre del Módulo 5. Módulos 0 a 5 completos.
**Siguiente:** Módulo 6 — Falcosidekick → Elasticsearch → Kibana.

---

## Objetivo del proyecto

Lab de seguridad en Kubernetes, documentado para publicar en GitHub. Doble propósito: pieza de portfolio y cerrar el gap de conocimiento en k8s (nivel de partida declarado: cero).

**Stack:** k3s + Kyverno (admission control) + Falco (runtime security vía eBPF) + Falcosidekick + Elasticsearch/Kibana.

**Modo de trabajo acordado:** guiado paso a paso. Se explica el concepto y el objetivo del ejercicio; Juan tipea los comandos y los manifests. La documentación (bitácora, README, este archivo) la redacta Claude. La shell es **fish**: no sirven los heredocs de bash.

---

## Infraestructura operativa

```
        HOST CachyOS "valkyrie" (15 GB RAM, 16 cores)
        kubectl · helm · ufw ACTIVO con reglas para el lab
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
 │ Kyverno          │   │                  │
 │ Falco (DaemonSet)│   │ Falco (DaemonSet)│
 └──────────────────┘   └──────────────────┘
     Debian 13.6 · kernel 6.12.101 · UEFI · BTF presente
```

**Acceso:** `ssh jubul@192.168.122.6` y `ssh jubul@192.168.122.128` (clave ed25519, sudo NOPASSWD).

**Levantar el lab después de un reboot del host** (las VMs no tienen autostart, decisión deliberada para no consumir 9 GB en cada arranque del desktop):

```bash
sudo virsh start k3s-server && sudo virsh start k3s-agent
# ~40s y verificar
sudo virsh net-dhcp-leases default
kubectl get nodes
```

Las IPs se mantuvieron tras el primer reboot (dnsmasq respeta las concesiones), pero conviene verificar antes de asumirlo. Pendiente: script `infra/lab-up.sh` / `lab-down.sh`.

### Versiones

| Componente | Versión |
|---|---|
| Host | CachyOS, kernel 7.1.8-1-cachyos |
| libvirt / QEMU | 12.6.0 / 11.1.0 (`<domain type='kvm'>`) |
| Firmware VMs | edk2-ovmf 202605-1 (`OVMF_CODE.secboot.4m.fd`) |
| Guests | Debian 13.6, kernel 6.12.101+deb13-cloud-amd64 |
| k3s | v1.36.3+k3s1, containerd 2.3.2-k3s2 |
| Kyverno | v1.18.2 (chart, 4 controladores con 1 réplica) |
| Falco | 0.44.1 (chart falco-9.1.0), driver **modern_ebpf** |

### Decisiones de diseño y por qué

- **VMs con libvirt/KVM**, no k3d ni k3s en el host: Falco necesita kernels reales.
- **Arranque UEFI, obligatorio.** Con SeaBIOS los guests no arrancan (bitácora #5).
- **RAM 3 + 6 GB.** El ELK va en el agent por ser el componente pesado.
- **Debian 13 genericcloud**, checksum SHA512 verificado, kernel 6.12 con BTF.
- **Discos copy-on-write.** La imagen base **no se modifica ni se borra**.
- **Semilla cloud-init a mano** (NoCloud, `xorrisofs -volid cidata`), no `virt-install --cloud-init` (bitácora #4).
- **`ufw` activo** con reglas mínimas y comentadas (bitácora #6).
- **kubeconfig traído por SSH con `sudo cat`**, no con `--write-kubeconfig-mode 644`. Queda en modo 600.

---

## Roadmap

| # | Módulo | Estado |
|---|--------|--------|
| 0 | Prerequisitos KVM/libvirt | **Completo** |
| 1 | Dos VMs Debian 13 con cloud-init | **Completo** |
| 2 | k3s: server + agent join, kubeconfig al host | **Completo** |
| 3 | Fundamentos k8s (namespace, deployment, service, labels) | **Completo** |
| 4 | Kyverno: admission control y políticas | **Completo** |
| 5 | Falco: modern eBPF y reglas custom | **Completo** |
| 6 | Falcosidekick → Elasticsearch → Kibana | **Siguiente** |
| 7 | Documentación final y diagramas | Pendiente |
| 8 | *(candidato)* GitOps con Argo CD o Flux | Idea |

---

## Notas del Módulo 3

`manifests/`: Namespace `demo`, Deployment `web` (2 réplicas de `nginx:1.29-alpine` con requests/limits), Service `web` ClusterIP.

Conceptos verificados con las manos:
- **Reconciliación:** al borrar un pod, el ReplicaSet crea el reemplazo. Nadie reinicia nada.
- **Labels como cableado:** cambiar `app=web` por `app=roto` en un pod lo saca del EndpointSlice y el ReplicaSet crea otro. Las relaciones son consultas sobre etiquetas, no punteros.
- Flannel asigna una subred por nodo: `10.42.0.0/24` al server, `10.42.1.0/24` al agent.
- `pod-template-hash` la agrega el ReplicaSet; es el mecanismo de las actualizaciones graduales.

## Notas del Módulo 4 (Kyverno)

`policies/01-disallow-latest-tag.yaml`: ClusterPolicy con dos reglas (`require-image-tag` y `validate-image-tag`) cubriendo `containers`, `initContainers` y `ephemeralContainers`. Suite de casos negativos en `tests/`, **5 de 5 en verde**.

Aprendizajes de la API:
- `failureAction` va **a nivel de regla** en 1.18.2, no en el spec. El campo legacy `spec.validationFailureAction` sigue existiendo y muestra su default `Audit`, lo que resulta contradictorio al inspeccionar el objeto: **manda el de la regla**.
- El *conditional anchor* `=(campo)` significa "si existe, validalo". Sin él, un Pod sin `initContainers` fallaría por no declarar algo que no le corresponde.
- **Autogen:** Kyverno genera solo las variantes `autogen-` para controladores de pods. Sin eso el rechazo llegaría al crear el pod y el error quedaría en eventos.

## Notas del Módulo 5 (Falco)

`falco/values.yaml` es la configuración completa (driver, tty, json_output y `customRules`). Se instala/actualiza con:

```bash
helm upgrade falco falcosecurity/falco -n falco -f falco/values.yaml
```

**Sin `--reuse-values`**, porque el archivo ya contiene todo. Actualmente en REVISION 4.

Estado: driver `modern_ebpf` cargado (`Opening 'syscall' source with modern BPF probe`), 25 reglas del set estable más una propia, `Detect Service Account Token Read`, afinada de **804 alertas a 1** conservando la detección en dos imágenes distintas.

Aprendizajes que conviene no reaprender:
- **El `output` define el esquema.** Falco solo emite en el JSON los campos referenciados en la plantilla. No se puede filtrar en Kibana por un campo que nunca se emitió. Primero visibilidad, después afinado.
- **`proc.name` se trunca a 15 caracteres** (viene del `comm` del kernel). Para nombres largos, `proc.exepath`.
- **Pero `proc.exepath` colapsa los binarios multi-llamada:** en Alpine `cat` reporta `exepath=/bin/busybox`. Ninguno de los dos campos sirve para todo.
- **`fd.name` reporta la ruta tal como el proceso la abrió**, no el punto de montaje resuelto. El token es alcanzable por `/var/run/secrets/...` y por `/run/secrets/...`: la detección por path tiene que enumerar los alias. Se resuelve con `pmatch`.
- Las prioridades de Falco van `EMERGENCY` → `DEBUG`, y `Terminal shell in container` es **NOTICE**. Filtrar por WARNING esconde el reconocimiento.

Comando útil para ver alertas sin filtrar por severidad:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=200 \
  | grep '"rule":' | jq -r '.priority + " | " + .rule + " | " + (.output_fields["k8s.pod.name"] // "-")'
```

---

## Pendientes abiertos

- **Sin commitear**: las entradas #9 y #10 de la bitácora, este archivo y `falco/values.yaml`.
- El pod `intruso` (busybox) sigue en el namespace `demo`; sirve para pruebas, borrarlo cuando estorbe.
- Revisar los archivos `.pacnew` que dejó el upgrade del host.
- `paru -Syu` sin ejecutar. **Revisar cada PKGBUILD antes**, por el incidente de paquetes maliciosos en el AUR del 2026-06-12.
- Migrar la suite de `tests/` a `kyverno test` para poder correrla en CI (con `kubectl apply --dry-run` la lógica de exit code queda invertida para casos negativos).
- Más políticas de Pod Security: `runAsNonRoot`, `readOnlyRootFilesystem`, prohibir `privileged` y `hostPath`.
- Evaluar IPs estáticas vía `network-config` en la semilla NoCloud.
- Decidir si el README principal pasa a inglés (mejor alcance para búsquedas internacionales).

## Próximo paso: Módulo 6

Falcosidekick como router de alertas → Elasticsearch → Kibana, todo en el nodo agent (6 GB). El `json_output: true` de Falco ya está puesto pensando en esto.

Dos cosas a tener en cuenta al arrancar: Elasticsearch pide ~2 GB de heap y hay que fijarlo explícitamente o intenta tomar la mitad de la RAM del nodo; y el `output` de las reglas propias determina qué campos se van a poder graficar en Kibana, así que conviene revisarlo antes de indexar.

Los diez incidentes resueltos están en `bitacora.md` con causa raíz y evidencia.
