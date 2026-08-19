# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# "vagrant up" deja todo el stack andando solo, sin pasos manuales:
#   - ub1:         destino Ubuntu (MariaDB, playbook2.yml)
#   - rh1:         destino Rocky Linux / familia Red Hat (Java+Tomcat+app, playbook1.yml)
#   - controller:  nodo de control con Ansible instalado, que corre ambos
#                  playbooks contra ub1/rh1 automáticamente (provisioner
#                  ansible_local) apenas terminan de levantar. Evita depender
#                  de WSL para tener un control node en Windows.
#
# Antes de levantar nada se genera (si no existe) un par de claves SSH propio
# del proyecto en .vagrant_ssh/ (no se toca tu ~/.ssh personal): la pública se
# instala en ub1/rh1 para el usuario "sysadmin", la privada se copia dentro de
# "controller". También se genera inventory/hosts.yml con las IPs fijas de
# abajo, y una vault-password aleatoria en .vault_pass.txt que "controller"
# usa para generar y encriptar playbooks/group_vars/all/vault.yml (gitignored,
# se genera a partir del template playbooks/vault_defaults.yml — ver README).
#
# Reaplicar los playbooks tras editarlos: vagrant provision
# Apagar todo: vagrant destroy

require "fileutils"
require "securerandom"

UBUNTU_IP     = "192.168.56.10"
REDHAT_IP     = "192.168.56.11"
CONTROLLER_IP = "192.168.56.9"
SSH_USER      = "sysadmin"

PROJECT_DIR     = __dir__
KEY_DIR         = File.join(PROJECT_DIR, ".vagrant_ssh")
PRIVATE_KEY     = File.join(KEY_DIR, "id_ed25519")
PUBLIC_KEY      = "#{PRIVATE_KEY}.pub"
VAULT_PASS_FILE = File.join(PROJECT_DIR, ".vault_pass.txt")

unless File.exist?(PRIVATE_KEY)
  FileUtils.mkdir_p(KEY_DIR)
  system("ssh-keygen", "-t", "ed25519", "-f", PRIVATE_KEY, "-N", "", "-C", "vagrant-ansible-controller")
end

unless File.exist?(VAULT_PASS_FILE)
  File.write(VAULT_PASS_FILE, SecureRandom.hex(32))
end

# Se genera de una (no hace falta esperar a "vagrant up"): las IPs son fijas,
# así el inventario ya existe cuando el provisioner de "controller" lo necesita.
File.write(File.join(PROJECT_DIR, "inventory", "hosts.yml"), <<~INV)
  # Generado automáticamente por el Vagrantfile. No editar a mano: se
  # sobreescribe en cada "vagrant up" / "vagrant provision" / "vagrant status".
  # Para apuntar a servidores propios en lugar de VMs locales, no uses Vagrant:
  # completá este archivo a mano (ver README, "Opción B").

  [ubuntu]
  ub1 ansible_host=#{UBUNTU_IP}


  [redhat]
  rh1 ansible_host=#{REDHAT_IP}
INV

public_key = File.exist?(PUBLIC_KEY) ? File.read(PUBLIC_KEY).strip : nil

Vagrant.configure("2") do |config|

  [["ub1", "ubuntu/jammy64", UBUNTU_IP, 1024, 1],
   ["rh1", "generic/rocky9", REDHAT_IP, 2048, 2]].each do |name, box, ip, memory, cpus|
    config.vm.define name do |node|
      node.vm.box = box
      node.vm.hostname = name
      node.vm.network "private_network", ip: ip
      node.vm.provider "virtualbox" do |vb|
        vb.memory = memory
        vb.cpus = cpus
      end

      # Crea el usuario "sysadmin" (con sudo sin contraseña) que esperan los
      # playbooks, e instala la clave pública del proyecto para que
      # "controller" pueda conectarse por SSH sin contraseña.
      node.vm.provision "shell", inline: <<-SHELL
        set -e
        id -u #{SSH_USER} >/dev/null 2>&1 || useradd -m -s /bin/bash #{SSH_USER}
        echo "#{SSH_USER} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/#{SSH_USER}
        chmod 440 /etc/sudoers.d/#{SSH_USER}
        mkdir -p /home/#{SSH_USER}/.ssh
        touch /home/#{SSH_USER}/.ssh/authorized_keys
      SHELL

      if public_key
        node.vm.provision "shell", inline: <<-SHELL
          echo "#{public_key}" >> /home/#{SSH_USER}/.ssh/authorized_keys
          sort -u -o /home/#{SSH_USER}/.ssh/authorized_keys /home/#{SSH_USER}/.ssh/authorized_keys
          chown -R #{SSH_USER}:#{SSH_USER} /home/#{SSH_USER}/.ssh
          chmod 700 /home/#{SSH_USER}/.ssh
          chmod 600 /home/#{SSH_USER}/.ssh/authorized_keys
        SHELL
      end
    end
  end

  config.vm.define "controller" do |controller|
    controller.vm.box = "ubuntu/jammy64"
    controller.vm.hostname = "controller"
    controller.vm.network "private_network", ip: CONTROLLER_IP
    controller.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
    end

    # Se instala vía pip (no apt) porque el paquete "ansible" de los repos de
    # Ubuntu 22.04 es 2.10, muy viejo: no trae módulos que usan los playbooks
    # como ansible.builtin.systemd_service (requiere ansible-core >= 2.13).
    controller.vm.provision "shell", inline: <<-SHELL
      set -e
      export DEBIAN_FRONTEND=noninteractive
      apt-get update
      apt-get install -y python3-pip
      pip3 install --upgrade ansible
    SHELL

    if File.exist?(PRIVATE_KEY)
      controller.vm.provision "file", source: PRIVATE_KEY, destination: "/home/vagrant/.ssh/id_ed25519"
      controller.vm.provision "shell", inline: <<-SHELL
        chmod 600 /home/vagrant/.ssh/id_ed25519
        chown vagrant:vagrant /home/vagrant/.ssh/id_ed25519
      SHELL
    end

    controller.vm.provision "file", source: VAULT_PASS_FILE, destination: "/home/vagrant/.vault_pass.txt"

    # playbooks/group_vars/all/vault.yml es gitignored: no es el archivo que
    # trackea git (eso es playbooks/vault_defaults.yml, siempre en texto
    # plano). Acá se genera la copia real que lee Ansible y se encripta con
    # la vault-password local recién generada. Como nunca se commitea, no
    # hay riesgo de que alguien más herede una copia encriptada con una
    # password que no tiene. Solo se hace si no existe todavía, para no
    # pisar cambios manuales (ansible-vault edit) en corridas posteriores.
    controller.vm.provision "shell", privileged: false, inline: <<-SHELL
      set -e
      DEST=/vagrant/playbooks/group_vars/all/vault.yml
      if [ ! -f "$DEST" ]; then
        cp /vagrant/playbooks/vault_defaults.yml "$DEST"
        ansible-vault encrypt "$DEST" --vault-password-file /home/vagrant/.vault_pass.txt
      fi
    SHELL

    controller.vm.provision "shell", privileged: false, inline: <<-SHELL
      set -e
      ansible-galaxy collection install -r /vagrant/requirements.yml
    SHELL

    # Corre los playbooks contra ub1/rh1 usando la clave y el inventario ya
    # provisionados arriba. "run: always" para que "vagrant provision" los
    # vuelva a aplicar después de editarlos (son idempotentes).
    # playbook2 (base de datos) va antes que playbook1 (app): la app depende
    # de que la base ya exista.
    ["playbook2.yml", "playbook1.yml"].each do |playbook|
      controller.vm.provision "ansible_local", run: "always" do |ansible|
        ansible.compatibility_mode = "2.0"
        ansible.install = false
        ansible.provisioning_path = "/vagrant"
        ansible.playbook = "playbooks/#{playbook}"
        ansible.inventory_path = "inventory/hosts.yml"
        ansible.limit = "all"
        ansible.vault_password_file = "/home/vagrant/.vault_pass.txt"
        # ansible.cfg vive en /vagrant (carpeta compartida "world writable"),
        # así que Ansible lo ignora por seguridad: el host_key_checking hay
        # que desactivarlo acá en vez de confiar en ese archivo.
        ansible.raw_arguments = [
          "--private-key=/home/vagrant/.ssh/id_ed25519",
          "--ssh-extra-args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'"
        ]
      end
    end
  end

end
