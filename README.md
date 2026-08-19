# Obligatorio Taller Linux

Automatiza el despliegue de una aplicación ToDo con Ansible:

- **rh1** (familia Red Hat): Java, Apache Tomcat y el `.war` de la aplicación (`playbook1.yml`)
- **ub1** (Ubuntu): MariaDB con la base de datos `todo` (`playbook2.yml`), accesible solo desde `rh1`

La app queda accesible en `http://<ip_rh1>:8080/todo`. No hay ningún usuario precreado: la primera vez hay que ir a "Signup" y registrarse antes de poder loguearse (con un usuario que no existe, el login no tira error, simplemente no pasa nada).

## Estructura del repositorio

```
.
├── Vagrantfile                    # Provisiona ub1, rh1 y una VM controller (opcional, ver más abajo)
├── ansible.cfg                    # Config de Ansible (host_key_checking off, útil para VMs efímeras)
├── requirements.yml                # Collections de Ansible necesarias
├── inventory/
│   ├── hosts.yml.example          # Template trackeado, IPs vacías (grupos [ubuntu] y [redhat])
│   └── hosts.yml                  # Copia real que lee Ansible, con las IPs (gitignored)
├── playbooks/
│   ├── playbook1.yml               # Java + Tomcat + app (host: redhat)
│   ├── playbook2.yml               # MariaDB + base de datos (host: ubuntu)
│   ├── vault_defaults.yml          # Template de contraseñas por defecto, texto plano (trackeado por git)
│   ├── group_vars/
│   │   └── all/
│   │       ├── vars.yml            # Variables parametrizables (usuario, puertos, versión de Tomcat, nombres de BD)
│   │       └── vault.yml           # Copia real que lee Ansible, generada y encriptada localmente (gitignored)
│   ├── files/
│   │   ├── todo.war                # Aplicación empaquetada, se copia tal cual a rh1
│   │   └── tomcat.service          # Unit de systemd para Tomcat
│   └── templates/
│       └── app.properties.j2       # Config de conexión a la BD (IP/puerto/credenciales desde las variables)
└── documentation/
    └── obligatorio.pdf             # Enunciado del obligatorio
```

## Variables parametrizables

Todo lo configurable vive en `playbooks/group_vars/`, no hay valores hardcodeados en los playbooks:

| Variable | Archivo | Default | Qué es |
|---|---|---|---|
| `ansible_user` | `vars.yml` | `sysadmin` | Usuario SSH/sudo en los servidores destino |
| `tomcat_version` | `vars.yml` | `9.0.121` | Versión de Tomcat a instalar |
| `tomcat_port` | `vars.yml` | `8080` | Puerto donde escucha Tomcat (y se abre en el firewall) |
| `app_dir` / `config_dir` | `vars.yml` | `/opt/tomcat/webapps` / `/opt/config` | Rutas de instalación en `rh1` |
| `db_name` | `vars.yml` | `todo` | Nombre de la base de datos |
| `db_user` | `vars.yml` | `prueba` | Usuario de la app en MariaDB |
| `mysql_port` | `vars.yml` | `3306` | Puerto de MariaDB (y se abre en el firewall solo para `app_server_ip`) |
| `mysql_root_password` | `vars.yml` → `vault.yml` | `AppM2023.` | Contraseña root de MariaDB |
| `db_password` | `vars.yml` → `vault.yml` | `prueba2024` | Contraseña del usuario de la app |

`vars.yml` y `vault.yml` están dentro de `playbooks/group_vars/all/`: Ansible carga automáticamente todo lo que hay en esa carpeta para el grupo especial `all` (por eso `vault.yml` no necesita estar referenciado en ningún lado — con solo existir ahí, sus variables quedan disponibles).

**`vault.yml` no es el archivo que trackea git** — ese es `playbooks/vault_defaults.yml` (texto plano, editable, ahí se cambian los valores por defecto). `vault.yml` se genera y se encripta *localmente* a partir de ese template:

- Con la Opción A (Vagrant): `vagrant up` genera una vault-password aleatoria propia en `.vault_pass.txt`, copia `vault_defaults.yml` a `group_vars/all/vault.yml` y lo encripta con esa password (solo la primera vez — no lo vuelve a tocar en corridas siguientes, para no pisar cambios manuales). Como `group_vars/all/vault.yml` está en `.gitignore`, nunca se commitea: cada clon del repo genera y encripta su propia copia, con su propia password, sin depender de nadie más.
- Con la Opción B (servidores propios): hacé lo mismo a mano antes de correr los playbooks — ver esa sección.

Si preferís no encriptarlo (por ejemplo para debuggear rápido), copiá `vault_defaults.yml` a `group_vars/all/vault.yml` sin encriptar y listo — Ansible lo lee igual.

`app_server_ip` se calcula solo a partir del inventario (IP de `rh1`): se usa para que MariaDB solo permita conectarse al usuario de la app desde ese servidor (`mysql_user` con `host` restringido + regla de `ufw` con `src`), en vez de aceptar conexiones desde cualquier IP.

## Prerrequisitos

En la máquina desde donde se ejecuta Ansible (el "control node"):

```bash
sudo dnf install pipx
pipx install --include-deps ansible
pipx ensurepath
pipx inject ansible argcomplete
pipx inject ansible ansible-lint
activate-global-python-argcomplete3 --user
```

Instalar las collections que usan los playbooks:

```bash
ansible-galaxy collection install -r requirements.yml
```

Se necesitan dos servidores destino (uno Ubuntu y uno de familia Red Hat), cada uno con un usuario `sysadmin` no-root con permisos de sudo. Si no tenés esos servidores a mano, podés levantarlos localmente con Vagrant (siguiente sección), que además te da un control node listo para usar.

## Opción A: todo automático con Vagrant (entorno local)

Requiere [Vagrant](https://developer.hashicorp.com/vagrant/downloads) y [VirtualBox](https://www.virtualbox.org/) instalados. No hace falta instalar Ansible en tu máquina ni usar WSL, ni pedirle nada a nadie.

```bash
vagrant up
```

Con un solo comando:

1. Levanta `ub1` y `rh1`, crea en ambas el usuario `sysadmin` con sudo sin contraseña.
2. Genera un par de claves SSH propio del proyecto (`.vagrant_ssh/`, no toca tu `~/.ssh` personal) y lo instala en `ub1`/`rh1`.
3. Regenera `inventory/hosts.yml` (gitignored) con las IPs fijas de las VMs.
4. Genera una vault-password aleatoria propia en `.vault_pass.txt`.
5. Levanta una tercera VM, `controller`, con Ansible instalado, le copia la clave privada y la vault-password, **genera `playbooks/group_vars/all/vault.yml` a partir de `vault_defaults.yml` y lo encripta** con esa password (ver "Variables parametrizables"), instala las collections de `requirements.yml`, y **corre `playbook2.yml` y `playbook1.yml` automáticamente** (la base primero, la app después) contra `ub1`/`rh1`.

Para reaplicar los playbooks después de editarlos: `vagrant provision`. Para entrar a alguna VM: `vagrant ssh ub1` / `vagrant ssh rh1` / `vagrant ssh controller`. Para destruir todo: `vagrant destroy`.

## Opción B: servidores propios (VMs existentes, cloud, on-prem, etc.)

1. Contar con los dos servidores (uno Ubuntu, uno familia Red Hat), cada uno con un usuario `sysadmin` no-root con permisos de administrador.
2. Copiar tu clave pública a cada servidor: `ssh-copy-id sysadmin@servidor_remoto`
3. Copiar `inventory/hosts.yml.example` a `inventory/hosts.yml` (no existe en el repo, se genera) y completar las IPs:

   ```bash
   cp inventory/hosts.yml.example inventory/hosts.yml
   ```

   ```ini
   [ubuntu]
   ub1 ansible_host=<ip_del_servidor_ubuntu>

   [redhat]
   rh1 ansible_host=<ip_del_servidor_redhat>
   ```

4. Generar `playbooks/group_vars/all/vault.yml` (no existe en el repo, se genera; ver "Variables parametrizables"):

   ```bash
   cp playbooks/vault_defaults.yml playbooks/group_vars/all/vault.yml
   ```

   Opcionalmente encriptalo (`ansible-vault encrypt playbooks/group_vars/all/vault.yml`) — en ese caso agregá `--vault-password-file <archivo>` o `--ask-vault-pass` a los comandos de abajo.

## Ejecutar los playbooks manualmente

Parados en la raíz del repositorio:

```bash
ansible-playbook -i ./inventory/hosts.yml --ask-become-pass ./playbooks/playbook2.yml
ansible-playbook -i ./inventory/hosts.yml --ask-become-pass ./playbooks/playbook1.yml
```

> El orden importa: `playbook2` (base de datos) va primero porque `playbook1` despliega una app que depende de que esa base ya exista.

> Con la Opción A (Vagrant) esto ya se corre solo — ver arriba. Con la Opción A, `sysadmin` tiene sudo sin contraseña, así que se puede omitir `--ask-become-pass`.

Al terminar, la aplicación queda disponible en `http://<ip_rh1>:8080/todo`.
