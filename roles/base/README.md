# Rol base

Paquetes base según la familia del sistema (`dnf` en RedHat, `apt` en Debian)
y directorios de trabajo en `/opt/lab` (`root:root 0750`).

| Variable | Dónde se define | Descripción |
|---|---|---|
| `paquetes_base` | `group_vars` de cada grupo | Lista de paquetes: cambia según la distribución |
| `base_directorios_lab` | `defaults/main.yml` | Directorios a crear dentro de `/opt/lab` |

Ver el [README principal](../../README.md).
