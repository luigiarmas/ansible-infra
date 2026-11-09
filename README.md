# ansible-infra

**Configuración de un laboratorio mixto (Rocky Linux y Ubuntu) como código.**
Los servidores ya no se configuran a mano: su estado deseado (paquetes base,
directorios de trabajo y un servidor web nginx endurecido) está declarado en
roles de Ansible. Se aplica con un solo comando a toda la infraestructura, es
idempotente (repetirlo no cambia nada que ya esté bien) y las contraseñas de
sudo de cada servidor se guardan cifradas con ansible-vault.

## Qué hace

| Rol | Qué configura | Plataformas |
|---|---|---|
| `base` | Paquetes base por familia (`dnf` en RedHat, `apt` en Debian) y directorios de trabajo en `/opt/lab` (`root:root 0750`) | Rocky Linux 9, Ubuntu |
| `nginx` | nginx con una página generada por plantilla, `server_tokens off`, firewalld activo con HTTP abierto, y verificación del contenido servido | Rocky Linux 9 |

## Estructura

```
.
├── ansible.cfg              # configuración del proyecto (inventario, sin .retry)
├── inventory.example.ini    # plantilla del inventario (el real está en .gitignore)
├── base.yml                 # aplica el rol base a todos los servidores
├── nginx.yml                # aplica el rol nginx al grupo [rocky]
├── group_vars/
│   ├── all.yml              # variables comunes e intérprete Python de los servidores
│   ├── rocky/vars.yml       # variables del grupo [rocky]
│   └── ubuntu/vars.yml      # variables del grupo [ubuntu]
├── roles/
│   ├── base/
│   └── nginx/
├── requirements.txt         # toolchain del nodo de control (Python)
└── requirements.yml         # colecciones de Ansible
```

## Requisitos

- **Nodo de control:** Python 3.12 o superior. Versiones fijadas en `requirements.txt`:
  ansible-core 2.21.6 (necesario para sudo-rs en Ubuntu 25.10+) y ansible-lint 26.11.0.
- **Colecciones:** `ansible.posix`, fijada en `requirements.yml`.
- **Servidores:** acceso SSH por llave y un usuario con sudo.

## Puesta en marcha

```bash
# 1. Toolchain en un entorno virtual
python3.12 -m venv ~/.venvs/ansible
source ~/.venvs/ansible/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

# 2. Inventario real (no se versiona)
cp inventory.example.ini inventory.ini    # y editar con los datos reales

# 3. Contraseña de sudo de cada grupo, cifrada (usar la MISMA contraseña de vault en todos)
read -rs -p "Contraseña de sudo del grupo rocky: " SECRETO; echo
printf '%s' "$SECRETO" | ansible-vault encrypt_string --ask-vault-pass \
  --stdin-name vault_become_password > group_vars/rocky/vault.yml
unset SECRETO
# repetir para group_vars/ubuntu/vault.yml

# 4. Llave SSH en el agente
eval "$(ssh-agent -s)" && ssh-add ~/.ssh/<llave>
```

## Uso

```bash
ansible-playbook base.yml --ask-vault-pass     # todos los servidores
ansible-playbook nginx.yml --ask-vault-pass    # grupo [rocky]
ansible-lint                                   # revisión de calidad del código
```

Con ansible-core 2.21, cualquier comando contra estos grupos (incluidos los
ad-hoc) necesita `--ask-vault-pass`, porque la contraseña de become se resuelve
en cada tarea.

## Pruebas realizadas

| Prueba | Resultado |
|---|---|
| Segunda ejecución de los dos playbooks | `changed=0` (idempotente) |
| Refactor de playbooks a roles sobre servidores ya configurados | `changed=0`: mismo comportamiento |
| firewalld detenido a mano | El rol lo vuelve a arrancar (`changed=1`) y HTTP sigue abierto |
| Un espacio de más en una plantilla | `--diff` lo muestra; `changed=1` sin recargar nginx (esa tarea no tiene `notify`) |
| Actualización de ansible-core 2.14 → 2.21 | Prueba de regresión: detectó una falla del intérprete Python, ya corregida |
| sudo-rs sin `ansible_become_exe` | Funciona con 2.21.6 (verificado con `-e` y `sudo --version`) |
| `ansible-lint` | 0 fallas, perfil `production` |

## Decisiones de diseño (por prioridad)

1. **Secretos cifrados con ansible-vault.** La contraseña de sudo de cada grupo
   se guarda en el repo cifrada (AES-256) y solo se descifra en memoria al
   ejecutar. Así la ejecución puede ser automática sin dejar credenciales
   legibles. El nombre de la variable queda visible (`vault_become_password`)
   para saber qué secretos existen sin descifrarlos.
2. **ansible-lint en perfil `production`.** Es el control de calidad del código:
   estructura, nombres y metadatos. No prueba que la configuración funcione: eso
   lo demuestran las ejecuciones idempotentes (`changed=0`) y las fallas
   provocadas a propósito.
3. **Intérprete Python explícito** (`ansible_python_interpreter`). Con Ansible
   2.21, el nodo de control descubría el Python del venv, que no tiene las
   librerías del sistema (`firewall`). Declararlo evita que Ansible adivine.
4. **sudo-rs en lugar de `sudo.ws`.** El arreglo para Ubuntu 26.04 se retiró
   después de comprobar que ansible-core 2.21.6 ya soporta sudo-rs. sudo-rs
   evita una clase de fallos de memoria del sudo clásico.
5. **firewalld dentro del rol `nginx`, sin dependencias entre roles.** Con un
   solo rol que abre puertos, extraer un rol `firewall` no se justifica todavía.

## Seguridad

- El inventario real y los archivos de vault están en `.gitignore`. Se verificó
  con `git check-ignore` antes del primer commit.
- La contraseña de vault no se guarda en el repositorio.
- Las IP del inventario de ejemplo son del rango de documentación 192.0.2.0/24 (RFC 5737).
- nginx no expone su versión (`server_tokens off`), y la página del laboratorio
  no muestra IP ni versión del sistema.
- No se usa `--diff` en tareas que manejen secretos: imprime el contenido de los archivos.

## Lo que aprendí

- **Git guarda todo.** Un dato que entró en un commit sigue en el historial
  aunque se borre después. Por eso este repositorio es nuevo, sin historial, y
  el anterior quedó privado como respaldo.
- **Actualizar también es un cambio.** Pasar de ansible-core 2.14 a 2.21 rompió
  un playbook. Se detectó con una prueba de regresión antes de seguir.
- **Una prueba que pasa no siempre prueba algo.** Un `-e` mal escrito hizo que
  un experimento "funcionara" por la razón equivocada. Desde entonces, primero
  verifico que la condición de la prueba se aplicó.
- Ansible es nuevo para mí: esta es la base, y se irá puliendo con práctica.

## Licencia

MIT. Ver [LICENSE](LICENSE).
