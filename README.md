# Proyecto - PRC - Sistemas Operativos - Vagrant - 2021
<!-- ABOUT THE PROJECT -->
## Descripción del Proyecto
El objetivo de este proyecto es configurar un laboratorio con vagrant en el cual se tengan múltiples máquinas virtuales en Docker configuradas en modo Swarm con un manager y varios nodos workers.

### Requisitos
Para este proyecto se requiere:
* Windows/MAC OS/ Linux.
* Powershell (en caso sea windows).
* Vagrant.

### Archivo de Vagrantfile
En el archivo ```Vagrantfile``` se utilizará el siguiente código para este proyecto:  
1. Variables a utilizar en el archivo:
   ```sh
   BOX_NAME = "bento/ubuntu-20.04"
   RAM = "512"
   CPUS = 2
   MANAGERS = 2
   MANAGER_IP = "172.1.1.1"
   WORKERS = 3
   WORKER_IP = "172.1.1.10"
   ```
   donde ```BOX_NAME``` indica la versión de sistema operativo a utilizar, ```RAM``` y ```CPU``` indican la cantidad de memoria y cpus asiganda a cada máquina virtual, ```MANAGERS``` y ```WORKERS``` indican la cantidad de manager y workers a utilizar en la red.
2. Scripts que van a ser necesarios para la instalación de Docker y luego la unión de los máquinas virtuales a manager y a workers usando Swarm.
   ```sh
   $install_docker_script = <<SCRIPT
      echo "------ VERIFICANDO ACTUALIZACIONES ------"
      sudo apt-get update
      echo "------ INSTALANDO DOCKER  ------"
      curl -sSL https://get.docker.com/ | sh
      sudo usermod -aG docker vagrant
   SCRIPT

   $manager_script = <<SCRIPT
   if [ "$#" -eq 1 ]; then
      echo "------ INICIANDO DOCKER SWARM ------"
      echo Arguments: $*
      docker swarm init --listen-addr $1:2377 --advertise-addr $1:2377
      docker swarm join-token -q manager > /vagrant/swarm-token-manager
      docker swarm join-token -q worker > /vagrant/swarm-token-worker
      echo "------ INSTALANDO PORTAINER ------"
      curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml
      docker stack deploy --compose-file=portainer-agent-stack.yml portainer  
   else
      echo "OCURRIO UN ERROR EN LOS PARAMETROS"
   fi
   SCRIPT

   $node_script = <<SCRIPT
   echo "Arguments: $*"
   if [ "$#" -eq 2 ]; then
      echo "------ HACIENDO JOIN CON DOCKER SWARM ------"
      docker swarm join --token `cat /vagrant/swarm-token-$1` $2:2377
   else
      echo "OCURRIO UN ERROR EN LOS PARAMETROS"
   fi
   SCRIPT
   ```
3. Configuración básica de Vagrant:
   ```sh
   Vagrant.configure("2") do |config|
      config.vm.box = BOX_NAME
      config.vm.synced_folder ".", "/vagrant"
      config.vm.provision "shell",inline: $install_docker_script, privileged: true
      config.vm.provider "virtualbox" do |vb|
      vb.memory = RAM
      vb.cpus = CPUS
    end
   ```
4. Configuración para las máquinas que son manager:
   ```sh
   (1..MANAGERS).each do |i|
        config.vm.define "manager0#{i}" do |manager|
          manager.vm.network :private_network, ip: "#{MANAGER_IP}#{i}"
          manager.vm.hostname = "manager0#{i}"
    ```      
    en el caso del primer manager, se le configura los puertos a usar y tambén se usa el script de manager para que pueda ser levantada la red desde esa máquina virtual. En el caso de que no sea el primer manager eso quiere decir que ya fue levantada la red y solo debe de unirse a la misma, se utiliza el parámetro "manager" para indicar que sigue siendo otro manager el que se va a agregar.
    ```sh
          if first_manager_ip.to_s.empty?
            manager.vm.network :forwarded_port, guest: 8080, host: 9501
            manager.vm.network :forwarded_port, guest: 5000, host: 9502
            manager.vm.network :forwarded_port, guest: 9000, host: 9503
            first_manager_ip = "#{MANAGER_IP}#{i}"
            manager.vm.provision "shell", inline: $manager_script, :args => ["#{first_manager_ip}"], privileged: true
          else
            manager.vm.provision "shell", inline: $node_script,:args => ["manager", "#{first_manager_ip}"], privileged: true
          end
        end
    end
    ```
5. Configuración para las máquinas que son workers:
   ```sh
      (1..WORKERS).each do |i|
        config.vm.define "worker0#{i}" do |worker|
            worker.vm.network :private_network, ip: "#{WORKER_IP}#{i}"
            worker.vm.hostname = "worker0#{i}"
            worker.vm.provision "shell", inline: $node_script, :args => ["worker", "#{first_manager_ip}"], privileged: true
        end
      end
   end
   ```
   En este caso se utiliza el parámetro "worker" para indicar que la máquina se agregará como worker en el script de node.

## Ejecución y Verificación
En este punto ya está terminado el archivo de vagrantfile por lo que para que comience a ejecutarse, en powershell, debe correse el siguiente comando:
   ```sh
   vagrant up
   ```
este comando, si no hay errores en el archivo de vagrantfile, levantará la cantidad de managers y de workers indicadas en el archivo. Puede ser algo tardado en lo que se hace las configuraciones para cada una de las máquinas. Cuando el comando termine de ejecutarse, se puede verificar el estado de las máquinas levatndas con el comando:
   ```sh
   vagrant status
   ```
En dado caso se necesite entrar por medio de SSH a una máquina virtual se puede con el siguiente comando:
   ```sh
   vagrant ssh manager01
   ```
Dentro de cualquier máquina virtual se pueden ejecutar los comandos de docker para ver el estado en el cual estan los conetenedores. La verificación via web se logra en la siguiente dirección: http://localhost:9503

![Cluster](https://github.com/Angeles-ortiz/proyecto_prc_sistemas_operativos_2021/blob/main/images/cluster.png)

Si aparece la siguiente imagen, entonces todo se configuró correctamente. La imagen anterior puede ser visualizada en el url http://localhost:9503/#!/1/docker/swarm/visualizer

## Pruebas adicionales
Para la ejecución de servicios y de que nuestra red con Swarm es funcional, debemos ingresar a cualquiera de las máquinas virtuales marcadas como managers:
   ```sh
   vagrant ssh manager01
   ```
Luego utilizar el siguiente comando para crear los servicios:
 ```sh
 docker service create --name prueba_nginx --replicas 15 --publish published=8080,target=80 nginx
 ```
Este comando crea 15 replicas del servicio de nginx. Estas 15 replicas son distribuidas entre la red de manager y workers que nosotros tengamos. De la aplicación descrita en los pasos anteriores, la imagen en Portainer del cluster seria como la siguiente:

![Cluster](https://github.com/Angeles-ortiz/proyecto_prc_sistemas_operativos_2021/blob/main/images/cluster2.png)
