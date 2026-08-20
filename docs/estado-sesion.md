# Estado de la sesión — punto de retomada

> Documento de handoff. Se actualiza al cerrar cada módulo para poder retomar sin recontextualizar.

**Última actualización:** 2026-08-20. Módulos 0 a 6 completos; módulo 7 (seguridad en k8s) con dos de cuatro partes hechas.
**Siguiente:** cerrar 7.3 (`automountServiceAccountToken: false`) y 7.4 (demostrar la regla `Drop and execute new binary in container`).

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
| 6 | Falcosidekick → Elasticsearch → Kibana | **Completo** |
| 7 | Seguridad en k8s: NetworkPolicies y Pod Security | **En curso (2 de 4)** |
| 8 | Documentación final y diagramas | Pendiente |
| 9 | *(candidato)* GitOps con Argo CD o Flux | Idea |

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

## Notas del Módulo 6 (ELK)

Pipeline completo y funcionando: **Falco → Falcosidekick → Elasticsearch → Kibana**. Todo en el nodo agent vía `nodeSelector`.

- `elk/01-elasticsearch.yaml`: StatefulSet de 1 réplica (ES 9.5.1), heap fijado en `-Xms1g -Xmx1g`, `fsGroup: 1000`, PVC de 10Gi con `local-path`. El storage class es `WaitForFirstConsumer`, así que **el volumen se crea en el disco del nodo donde cae el pod**: el `nodeSelector` no es optimización, es condición de corrección.
- `elk/02-kibana.yaml`: Deployment (stateless, guarda su config dentro de ES) + Service `NodePort 30601`. Acceso: **http://192.168.122.128:30601**.
- Falcosidekick se despliega como subchart desde `falco/values.yaml` (`falcosidekick.enabled: true`), lo que además cablea el `http_output` de Falco automáticamente. `webui` desactivada a propósito (requiere Redis).
- Índices diarios `falco-YYYY.MM.DD`, así que el data view de Kibana debe usar el patrón **`falco-*`** con comodín.
- Hay un `_index_template/falco` aplicado con `number_of_replicas: 0`. Los templates solo actúan en la creación, por eso el índice del día en que se aplicó siguió en `rep 1` (cluster en `yellow`, que con un solo nodo es el estado correcto).

**Nota de contexto:** Juan tiene 4 años de experiencia con ELK en producción, HA y PCI-DSS. No corresponde explicarle Elastic; sí la capa de Kubernetes.

**Deuda técnica identificada (su terreno, decisión suya):** el mapeo dinámico generó 29 campos con solo 9 documentos, 20 bajo `output_fields.*`, incluidos nombres con corchetes (`output_fields.proc.aname[2]`). Cada regla nueva agrega mappings. La opción conocida es mapear `output_fields` como `flattened`, a cambio de perder tipos numéricos.

## Notas del Módulo 7 (seguridad en k8s)

**7.1 — NetworkPolicies (hecho).** `netpol/01-elk-aislamiento.yaml`: default-deny de ingress en `elk`, más allow de ES:9200 solo desde el namespace `falco` o pods `app: kibana`, y allow de Kibana:5601 desde `ipBlock: 192.168.122.0/24`. Cierra la vulnerabilidad de la bitácora #11. Sin control de egress todavía.

**7.2 — Pod Security con Kyverno (hecho).** `policies/02-pod-security.yaml`: `disallow-privileged` y `require-run-as-nonroot`, las dos en `Enforce`, con excepciones documentadas. Todas las cargas propias se arreglaron en lugar de exceptuarse: `web` pasó a `nginxinc/nginx-unprivileged:1.29-alpine` en el 8080 (uid 101), y ES, Kibana e `intruso` declaran su `securityContext`.

Cosas para no reaprender:
- **La excepción de Falco va por namespace Y label** (`app.kubernetes.io/name: falco`), no por namespace solo: falcosidekick vive ahí y no necesita privilegios.
- **`runAsNonRoot: true` solo no alcanza** si la imagen declara `USER root` (busybox): el kubelet rechaza con `CreateContainerConfigError` y hay que indicar `runAsUser`.
- **Siempre validar la excepción borrando los pods de Falco** y confirmando que el DaemonSet vuelve a 2/2. Si la excepción está mal, no falla nada visible: simplemente te quedás sin detección.
- **Endurecer reduce la telemetría.** Con `intruso` como uid 1000, `cat /etc/shadow` da permission denied y **no genera alerta**, porque el macro `open_read` exige `fd.num >= 0`. El mismo ataque pasó de 2 alertas a 1.

**7.3 y 7.4 pendientes:** `automountServiceAccountToken: false` como prevención que complementa la regla de robo de token, y demostrar `Drop and execute new binary in container` (ya existe en el set estable, usa `proc.is_exe_upper_layer`) — es la detección que cubre al atacante en un contenedor distroless, que tiene que traer su propio binario.

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
