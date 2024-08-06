#Pasos para la ejecución de los playbooks

#Contar con un servidor linux que tenga un usuario (no ROOT) con permisos de administrador

#Instalar ansible en dicho servidor: Sudo dnf install pipx; pipx install --include-deps ansible; pipx ensurepath; pipx inject ansible argcomplete; pipx inject ansible ansible-lint; activate-global-python-argcomplete3 --user

#Copiar la clave pública a los servidores remotos: ssh-copy-id usuario@servidor_remoto

#Descargar el repositorio desde: https://github.com/frankfrot30/obligatoriotallerlinux

#Editar el archivo hosts que se encuentra dentro de inventory (inventory/host) agregando los hosts sobre los que se van a ejecutar los playbooks

#Ejecutar playbook playbook1.yml estando ubicados dentro de la carpeta del repositorio: ansible-playbook -i ./inventory/hosts.yml --ask-become-pass ./playbooks/playbook1.yml 

#Ejecutar playbook playbook2.yml estando ubicados dentro de la carpeta del repositorio: ansible-playbook -i ./inventory/hosts.yml --ask-become-pass ./playbooks/playbook2.yml 