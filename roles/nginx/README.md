# Rol nginx

Instala nginx (familia RedHat), publica una página generada por plantilla,
aplica `server_tokens off`, asegura firewalld activo con HTTP abierto y
verifica que la página sirva el nombre del host.

| Variable | Por defecto | Descripción |
|---|---|---|
| `nginx_server_tokens` | `"off"` | Valor de `server_tokens` (entre comillas: sin ellas YAML lo lee como booleano) |

Usa la colección `ansible.posix` (módulo `firewalld`). Ver el [README principal](../../README.md).
