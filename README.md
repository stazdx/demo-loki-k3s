# Grafana Loki en k3s

Para la siguiente demo usaremos lo siguiente:

- [Multipass](https://multipass.run/)
- [k3s](https://k3s.io/)
- [Helm](https://helm.sh/)
- [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/)
- [Loki](https://grafana.com/oss/loki/)
- [Grafana](https://grafana.com/)

### Instalamos Multipass

Podemos instalarlo tanto en Windows, Mac como Linux.

| OS | Referencia |
| ------ | ------ |
| Windows | https://multipass.run/docs/installing-on-windows |
| Mac | https://multipass.run/docs/installing-on-macos |
| Linux | https://multipass.run/docs/installing-on-linux |

Para esta Demo, utilizaremos la instalación en mac utilizando Homebrew

```sh
# Instalamos Multipass con brew cask
$ brew cask install multipass

# Verificamos que lo tenemos instalado
$ multipass
```

### Creamos nuestro nodo Linux (Ubuntu)

```sh
# Crearemos una instancia Ubuntu pasando como parámetros nombre del nodo, ram y disco que le asignaremos
$ multipass launch --name demok3s --mem 2G --disk 5G

# Ingresaremos dentro de nuestra instancia ya creada
$ multipass shell demok3s

# Verificamos que estamos dentro de la instancia y escalaremos a root
$ sudo su
```

### Instalamos k3s

```sh
# Instalamos k3s
$ curl -sfL https://get.k3s.io | sh -

# Agregaremos variable de entorno KUBECONFIG
$ export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Verificamos que tenemos kubernetes instalado
$ kubectl cluster-info
```

### Instalamos Helm

```sh
# Instalamos Helm
$ snap install helm --classic

# Verificamos que tengamos instalado Helm
$ helm
```

### Instalamos Loki & Promtail

```sh
# Agregamos Loki al repo de Helm
$ helm repo add loki https://grafana.github.io/loki/charts

# Actualizamos Helm para que agregue Loki
$ helm repo update

# Instalamos Loki-stack
$ helm upgrade --install loki loki/loki-stack
```

### Instalamos Grafana

```sh
# Agregamos Grafana al repo de Helm
$ helm repo add grafana https://grafana.github.io/helm-charts

# Actualizamos Helm para que agregue Grafana
$ helm repo update

# Instalamos Grafana
$ helm upgrade --install grafana grafana/grafana

# Obtenemos nuestro password del usuario admin
$ kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# Obtenemos el ip de nuestro nodo en otra ventana del terminal (fuera de nuestra instancia Ubuntu)
$ multipass ls | sed -e '1d' | awk '{print $3}'

# Redireccionamos el tráfico que sale de nuestro servicio grafana:80 hacia $NODE_IP:8084, donde $NODE_IP es el IP que obtuvimos en el comando anterior (ejecutamos dentro de nuestra instancia Ubuntu)
$ kubectl port-forward --address $NODE_IP service/grafana 8084:80

# Vamos a nuestro navegador e ingresamos -> $NODE_IP:8084 y autenticamos con  el usuario admin y el password que obtuvimos antes
```

### Configuramos Loki como Data Source

1. Desde el menu lateral de Grafana vamos a `Configuration` -> `Data Sources` -> `Add Data Source`
2. Buscamos loki y lo seleccionamos
3. En el campo `URL` colocamos `http://loki:3100`
4. Luego click en `Save & Test`

### Observamos logs

1. Desde el menu lateral de Grafana vamos a `Explore`
2. En `log labels` seleccionamos un filtro y podremos observar logs :blush:

### Desplegamos NginX balanceando carga

```sh
# Vemos nuestro manifiesto a ejecutar con 2 réplicas y el label app=nginx
$ kubectl create deploy nginx --image stazdx/nginx --replicas 2 --dry-run -o yaml

# Ejecutamos nuestro manifiesto
$ kubectl create deploy nginx --image stazdx/nginx --replicas 2

# Exponemos un servicio que balancee nuestros 2 pods creados
$ kubectl create service loadbalancer nginx --tcp=9090:80

# Vamos a nuestro navegador e ingresamos -> $NODE_IP:9090 veremos la pantalla de bienvenida de NginX
```

### Visualizamos los logs 

1. Regresamos a nuestro Grafana en el menu lateral vamos a `Explore`
2. En nuestro filtro `log labels` seleccionamos el label `app` -> `nginx`
3. Ahora realizamos unas 100 peticiones para generar más tráfico y veamos en acción el balanceo de carga, para esto ejecutamos:

```sh
$ for i in {1..100}; do curl http://$NODE_IP:9090/; done
```

4. Regresamos a nuestro Grafana y verificamos que Loki está centralizando los logs generados :blush:

### Levantando nuestro stack desde un manifiesto de configuración para Helm

```sh

# Primero borramos nuestros charts instalados 
$ helm del grafana loki

# Instalamos grafana apuntando al manifest loki.yaml que se encuentra en la carpeta manifest, para esto copiamos su contenido dentro de un archivo en nuestra instancia
$ helm upgrade --install loki loki/loki-stack -f loki.yaml

# Instalamos grafana apuntando al manifest grafana.yaml que se encuentra en la carpeta manifest, para esto copiamos su contenido dentro de un archivo en nuestra instancia
$ helm upgrade --install grafana grafana/grafana -f grafana.yaml

# Vamos a nuestro navegador e ingresamos -> $NODE_IP:8084 y autenticamos con  el usuario admin y el password que pusimos en nuestro manifiesto de grafana

# Podremos observar que ya tenemos el datasource de loki preconfigurado y ya podremos ver logs :)

```

### Borrar nuestra instancia de Multipass

```sh

# Primero borramos nuestros charts instalados 
$ multipass delete demok3s

# Borra todas las instancias de multipass
$ multipass purge

```

### Vista de Grafana Loki

![Alt text](loki.png?raw=true "Grafana Loki")


Happy coding :smile: !!


