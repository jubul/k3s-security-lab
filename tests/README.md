# Casos de prueba de las políticas

Casos negativos de las ClusterPolicies de [`../policies/`](../policies/). Cada archivo es un Pod mínimo que ejercita **una** vía de evasión, con el resultado esperado declarado en el encabezado.

## Ejecutar

```bash
kubectl apply -f tests/ -n demo --dry-run=server
```

`--dry-run=server` envía el objeto al API server, que lo pasa por los admission webhooks y devuelve el veredicto **sin persistir nada**. Es la forma correcta de probar políticas: se ejercita la ruta real de admisión, no una simulación.

## Resultados esperados

| Archivo | Vía probada | Regla que debe atraparlo | Esperado |
|---|---|---|---|
| `01-latest-container.yaml` | `:latest` en `containers` | `validate-image-tag` | 🚫 bloqueado |
| `02-notag-container.yaml` | sin tag en `containers` | `require-image-tag` | 🚫 bloqueado |
| `03-latest-initcontainer.yaml` | `:latest` en `initContainers` | `validate-image-tag` | 🚫 bloqueado |
| `04-notag-initcontainer.yaml` | sin tag en `initContainers` | `require-image-tag` | 🚫 bloqueado |
| `05-valido.yaml` | todo con tags fijos | ninguna | ✅ permitido |

Los cuatro primeros deben devolver un error de admisión; el quinto debe reportar `created (server dry run)`.

## Criterio de diseño

**En los casos 03 y 04 el contenedor principal es deliberadamente válido.** Si el caso falla, la única causa posible es el initContainer. Un caso de prueba con dos cosas mal a la vez no distingue cuál de las dos regla lo atrapó, y deja de servir como diagnóstico.

**El caso positivo no es opcional.** Una política que bloquea todo es igual de inútil que una que no bloquea nada. El `05` verifica que los *conditional anchors* (`=(campo)`) funcionen: sin ellos, un Pod que no declara `initContainers` fallaría la validación por no tener un campo que no le corresponde tener.

**Un archivo, una vía.** El caso `04` es el que detectó el bug residual de [`bitacora.md` #8](../docs/bitacora.md): un error de capitalización (`initcontainers` en lugar de `initContainers`) presente en una sola de las dos reglas. Es el único caso que necesita esa regla **y** esa lista de contenedores simultáneamente — con casos más gruesos, el agujero pasaba desapercibido.

## Próximo paso

Migrar a **`kyverno test`**, el CLI de Kyverno, que declara el resultado esperado de cada caso en un archivo y corre **sin cluster**. Eso permite ejecutar la suite en CI y hacer que un pull request que abra un agujero en una política falle el pipeline.
