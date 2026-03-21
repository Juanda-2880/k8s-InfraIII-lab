# DESPLIEGUE DE UN CLÚSTER DE KUBERNETES

## Implementación con kubeadm en Rocky Linux 9.7

**Documentación Técnica Completa - Guía paso a paso para la instalación y configuración**

---

## Tabla de Contenidos

1. [Introducción y Objetivos](#introducción-y-objetivos)
2. [Contexto del Laboratorio](#contexto-del-laboratorio)
3. [Fase 1: Configuración del Servidor Bastión](#fase-1-configuración-del-servidor-bastión)
4. [Fase 2: Preparación de los Nodos Kubernetes](#fase-2-preparación-de-los-nodos-kubernetes)
5. [Fase 3: Inicialización del Clúster](#fase-3-inicialización-del-clúster)
6. [Fase 4: Validación y Pruebas](#fase-4-validación-y-pruebas)
7. [Resultados y Conclusiones](#resultados-y-conclusiones)

---

## Introducción y Objetivos

Este documento presenta una guía completa para la implementación de un clúster de Kubernetes funcional utilizando kubeadm en un entorno virtualizado. La solución propuesta se construye sobre Rocky Linux 9.7 ejecutándose en máquinas virtuales bajo VirtualBox, replicando así un escenario similar al que se encontraría en un entorno de producción.

Kubernetes es una plataforma de orquestación de contenedores que automatiza el despliegue, escalado y gestión de aplicaciones containerizadas. Para este laboratorio, hemos seleccionado kubeadm como herramienta de inicialización, que simplifica significativamente el proceso de configuración del clúster manteniendo la flexibilidad necesaria para realizar ajustes personalizados.

### Objetivos del Laboratorio

- Implementar un clúster de Kubernetes completamente funcional con un nodo control plane y múltiples nodos worker
- Configurar la infraestructura de red necesaria incluyendo servicios DHCP y DNS para la gestión automática de direcciones IP
- Establecer un servidor bastión que actúe como punto central de administración del clúster
- Validar el correcto funcionamiento del clúster mediante despliegues de prueba y verificaciones de conectividad de red
- Documentar cada paso del proceso para facilitar futuras implementaciones y mantenimiento

---

## Contexto del Laboratorio

### Arquitectura de Infraestructura

La solución requiere la creación de tres máquinas virtuales en VirtualBox, cada una con un rol específico en la infraestructura:

- **Servidor Bastión**: Actúa como servidor DHCP y DNS, proporcionando servicios de red centralizados. Además, sirve como punto de administración remota del clúster mediante kubectl

- **Nodo Master (Control Plane)**: Ejecuta los componentes de control de Kubernetes. Requiere mínimo 2 cores de CPU y 2 GB de RAM para garantizar un funcionamiento estable

- **Nodo Worker**: Ejecuta las cargas de trabajo containerizadas. Requiere mínimo 1 core de CPU y 2 GB de RAM

### Configuración de Red

Cada máquina virtual está configurada con múltiples interfaces de red para diferentes propósitos: una interfaz NAT para salida a internet, una interfaz Bridge para acceso SSH desde la máquina host, una interfaz Host-Only para la red privada, y una red interna dedicada para la comunicación del clúster.

---

## Fase 1: Configuración del Servidor Bastión

El servidor bastión proporciona la infraestructura de red fundamental para el clúster. Su configuración incluye servicios DNS y DHCP que automatizan la gestión de direcciones IP y la resolución de nombres.

### 1.1 Configuración de Red e IP Estática

Primero se configura una IP estática en la red interna. El comando utiliza NetworkManager (nmcli) para establecer una dirección fija en la subred 192.168.100.0/24:

```bash
# Reemplaza enp0s10 con tu interfaz real de red interna
sudo nmcli con mod enp0s10 ipv4.addresses 192.168.100.10/24 ipv4.method manual
sudo nmcli con up enp0s10
```

Este comando configura la interfaz enp0s10 con la dirección IP 192.168.100.10 utilizando notación CIDR, donde /24 indica que los primeros 24 bits son la red (máscara 255.255.255.0).

### 1.2 Instalación del Servidor DNS (BIND)

BIND es el servidor DNS más utilizado en sistemas Linux. Se instala con el siguiente comando:

```bash
sudo dnf install bind bind-utils -y
```

Luego, se edita el archivo de configuración `/etc/named.conf` para permitir consultas desde la red interna:

```
options {
    listen-on port 53 { 127.0.0.1; 192.168.100.10; };
    allow-query     { localhost; 192.168.100.0/24; };
    recursion yes;
    # ... mantén el resto de las opciones por defecto ...
};
```

Se activa el servicio y se abre el puerto en firewalld:

```bash
sudo systemctl enable --now named
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
```

### 1.3 Instalación del Servidor DHCP

El servidor DHCP automatiza la asignación de direcciones IP a los nodos del clúster. Se instala mediante:

```bash
sudo dnf install dhcp-server -y
```

La configuración del servidor DHCP se realiza en el archivo `/etc/dhcp/dhcpd.conf`. Esta configuración incluye reservaciones de IP fijas basadas en las direcciones MAC de los nodos:

```
authoritative;
subnet 192.168.100.0 netmask 255.255.255.0 {
  range 192.168.100.50 192.168.100.100;
  option routers 192.168.100.10;
  option domain-name-servers 192.168.100.10;

  host master {
    hardware ethernet 08:00:27:AA:BB:CC;
    fixed-address 192.168.100.20;
  }

  host worker {
    hardware ethernet 08:00:27:XX:YY:ZZ;
    fixed-address 192.168.100.21;
  }
}
```

**Nota**: Las direcciones MAC (hardware ethernet) deben reemplazarse con las direcciones reales de las máquinas virtuales. Se inicia el servicio:

```bash
sudo systemctl enable --now dhcpd
sudo firewall-cmd --permanent --add-service=dhcp
sudo firewall-cmd --reload
```

---

## Fase 2: Preparación de los Nodos Kubernetes

Esta fase prepara los nodos Master y Worker con los ajustes necesarios del sistema operativo para ejecutar Kubernetes correctamente. Los pasos incluyen la configuración de hostnames, desactivación de swap, ajustes de kernel y la instalación del runtime de contenedores.

### 2.1 Configuración de Hostnames

Cada nodo requiere un hostname único para facilitar la identificación y la resolución de nombres.

En el nodo Master:

```bash
sudo hostnamectl set-hostname master
```

En el nodo Worker:

```bash
sudo hostnamectl set-hostname worker
```

### 2.2 Desactivación de Swap y Configuración de SELinux

Kubernetes requiere que la memoria swap esté desactivada para garantizar un rendimiento predecible. SELinux debe configurarse en modo permisivo para evitar conflictos de permisos. Se ejecutan en ambos nodos:

```bash
# Desactivar Swap temporal y permanentemente
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Configurar SELinux a modo permisivo
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 2.3 Configuración de Módulos del Kernel y Parámetros sysctl

Kubernetes requiere módulos específicos del kernel y parámetros de red para funcionar correctamente. El módulo overlay permite la superposición de sistemas de archivos, y br_netfilter permite que iptables procese el tráfico de bridges:

```bash
# Cargar módulos del kernel
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Configurar sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### 2.4 Configuración del Firewall

El firewall debe configurarse para permitir el tráfico específico de Kubernetes.

En el nodo Master se abren los puertos del Control Plane:

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10259/tcp
sudo firewall-cmd --permanent --add-port=10257/tcp
sudo firewall-cmd --reload
```

En el nodo Worker se abren los puertos para servicios y NodePorts:

```bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --reload
```

Los puertos listados corresponden a:
- **6443**: API Server
- **2379-2380**: etcd
- **10250**: kubelet
- **10259**: scheduler
- **10257**: controller-manager
- **30000-32767**: NodePorts para servicios

### 2.5 Instalación de containerd, kubeadm, kubelet y kubectl

containerd es el runtime de contenedores seleccionado. kubeadm, kubelet y kubectl son los componentes base de Kubernetes. Se instalan en ambos nodos:

```bash
# Agregar repositorio de Docker e instalar containerd
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install containerd.io -y

# Configurar containerd para usar SystemdCgroup
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl enable --now containerd

# Agregar repositorio de Kubernetes (versión 1.28)
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
EOF

# Instalar kubelet, kubeadm y kubectl
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## Fase 3: Inicialización del Clúster

Esta fase cubre la inicialización del clúster Kubernetes, incluyendo la configuración del nodo Master, la instalación del plugin de red y la incorporación del nodo Worker.

### 3.1 Inicialización del Nodo Master

El nodo Master se inicializa utilizando kubeadm init. Este comando crea el control plane de Kubernetes y genera los certificados necesarios:

```bash
sudo kubeadm init --apiserver-advertise-address=192.168.100.20 --pod-network-cidr=10.244.0.0/16
```

**Parámetros utilizados**:
- `--apiserver-advertise-address`: Establece la IP del API Server (192.168.100.20)
- `--pod-network-cidr`: Define la red de pods (10.244.0.0/16), compatible con Flannel

Al completarse la inicialización, kubeadm muestra un comando `kubeadm join` que debe guardarse, ya que se utilizará para unir los nodos Worker al clúster.

Luego se configura el acceso a kubectl en el nodo Master:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3.2 Instalación del Plugin de Red CNI (Flannel)

Flannel es un plugin de red que proporciona una red virtual para la comunicación entre pods en diferentes nodos. Se instala aplicando un manifest de Kubernetes:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Esta operación descarga e instala automáticamente los componentes de Flannel en el clúster. Después de ejecutar este comando, los pods del sistema comenzarán a inicializarse.

### 3.3 Unión del Nodo Worker al Clúster

Para incorporar el nodo Worker al clúster, se ejecuta el comando `kubeadm join` que fue generado en el Master. Este comando contiene un token de autenticación temporal y la dirección del Master:

```bash
kubeadm join 192.168.100.20:6443 --token XXXXX --discovery-token-ca-cert-hash sha256:XXXXX
```

**Nota**: Los valores XXXXX deben reemplazarse con el token y hash específicos generados por `kubeadm init` en el Master.

### 3.4 Configuración de Acceso Remoto desde el Bastión

Para administrar el clúster de forma remota desde el servidor bastión, se copia el archivo de configuración admin.conf del Master:

```bash
mkdir -p ~/.kube
scp tu_usuario@192.168.100.20:/etc/kubernetes/admin.conf ~/.kube/config
```

Esto permite ejecutar comandos kubectl desde el bastión sin necesidad de conectarse directamente al Master, centralizando la administración del clúster.

---

## Fase 4: Validación y Pruebas

Esta fase verifica que el clúster está completamente operativo y puede gestionar cargas de trabajo. Se realizan pruebas de conectividad, despliegues y resolución de nombres.

### 4.1 Verificación del Estado de los Nodos

El primer paso es verificar que todos los nodos están en estado Ready:

```bash
kubectl get nodes
```

Este comando debe mostrar el nodo master y el nodo worker con estado Ready. La salida es similar a:

```
NAME     STATUS   ROLES           AGE     VERSION
master   Ready    control-plane   5m      v1.28.x
worker   Ready    <none>          3m      v1.28.x
```

### 4.2 Verificación de Pods del Sistema

Los pods del sistema deben estar en estado Running. Se verifica con:

```bash
kubectl get pods -n kube-system
```

Este comando lista todos los pods en el namespace kube-system, incluyendo los componentes del control plane, Flannel y CoreDNS.

### 4.3 Despliegue de Prueba (Nginx)

Se crea un despliegue de prueba para validar que el clúster puede programar y ejecutar pods:

```bash
kubectl create deployment nginx-test --image=nginx
```

Se verifica el estado de los pods del despliegue:

```bash
kubectl get pods -o wide
```

La salida muestra el nombre del pod, su estado, el nodo donde se ejecuta, y su dirección IP. El pod debe mostrar estado Running y estar asignado al nodo worker.

### 4.4 Exposición del Despliegue mediante NodePort

El despliegue se expone mediante un servicio de tipo NodePort para acceder a la aplicación desde fuera del clúster:

```bash
kubectl expose deployment nginx-test --port=80 --type=NodePort
```

Se obtiene información del servicio creado:

```bash
kubectl get svc
```

La salida muestra el servicio nginx-test con un puerto asignado en el rango 30000-32767 (por ejemplo, 30123). Este puerto puede usarse para acceder al servicio desde cualquier nodo del clúster.

### 4.5 Prueba de Conectividad

Se verifica que la aplicación es accesible desde el bastión:

```bash
curl http://<IP_DEL_WORKER>:<PUERTO_ASIGNADO>
```

Ejemplo:
```bash
curl http://192.168.100.21:30123
```

Este comando debe devolver la página HTML de Nginx, demostrando que el tráfico fluye correctamente a través del clúster.

### 4.6 Verificación de Resolución de Nombres (CoreDNS)

CoreDNS proporciona resolución de nombres interna en Kubernetes. Se verifica su estado:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Se pueden revisar los logs de CoreDNS para diagnosticar problemas de resolución:

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---

## Resultados y Conclusiones

### Resumen de la Implementación

La implementación exitosa de este laboratorio demuestra la capacidad de desplegar un clúster de Kubernetes funcional utilizando herramientas estándar en la industria. Los pasos ejecutados incluyen:

- Configuración de infraestructura de red con servicios DNS y DHCP centralizados
- Preparación de nodos con ajustes de sistema operativo específicos para Kubernetes
- Inicialización del control plane utilizando kubeadm
- Instalación de plugin de red CNI para comunicación inter-pod
- Unión de nodos worker al clúster
- Validación mediante despliegues de aplicaciones y pruebas de conectividad
- Administración remota centralizada desde un servidor bastión

### Validaciones Realizadas

El clúster ha sido validado completamente mediante múltiples pruebas que confirman su operatividad:

- **Todos los nodos se encuentran en estado Ready**: Tanto el Master como el Worker están completamente operativos y disponibles para recibir cargas de trabajo

- **Pods del sistema en estado Running**: Los componentes de Kubernetes (apiserver, etcd, scheduler, controller-manager) y plugins de red (Flannel) están funcionando correctamente

- **Despliegues funcionales**: Los pods de aplicaciones se crean y ejecutan correctamente en los nodos designados

- **Conectividad de red**: Los servicios son accesibles desde fuera del clúster a través de NodePorts, demostrando que el networking funciona correctamente

- **Resolución de nombres**: CoreDNS proporciona resolución de servicios dentro del clúster

### Conclusiones

Este laboratorio ha proporcionado una experiencia práctica completa en la configuración y administración de un clúster de Kubernetes. Los conceptos aprendidos incluyen:

- La importancia de preparar correctamente la infraestructura de red y del sistema operativo antes de desplegar Kubernetes
- Cómo utilizar kubeadm para inicializar clústeres de forma reproducible
- La arquitectura de Kubernetes con componentes del control plane y nodos worker
- El papel crítico de los plugins de red en la comunicación entre pods
- Cómo desplegar y exponer aplicaciones en Kubernetes
- Herramientas y prácticas para la administración y monitoreo de clústeres

La configuración resultante proporciona una plataforma sólida para ejecutar aplicaciones containerizadas en un entorno altamente disponible y escalable. El conocimiento adquirido puede aplicarse tanto en escenarios de laboratorio como en despliegues de producción, adaptando los principios a diferentes necesidades y restricciones de infraestructura.

