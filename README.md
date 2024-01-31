# Create and-Manage Cloud-Resources Course

## Introducción

En este repositorio encontraréis paso por paso de la realización del Challenge Lab de Google Cloud Platform en el que hemos creado un proyecto de Google Cloud Platform con una instancia jumphost, un cluster de Kubernetes y un balanceador de carga HTTP.

## ¡Comenzamos!

Comenzamos abriendo nuestra consola de Google Cloud Platform y vamos a activar la shell de Google Cloud. Cloud Shell es una máquina virtual que cuenta con herramientas para desarrolladores. Ofrece un directorio principal persistente de 5 GB y se ejecuta en Google Cloud. Cloud Shell proporciona acceso de línea de comandos a tus recursos de Google Cloud.

Cuando nos conectemos, se nos habrá completado la autenticació, y el proyecto estará configurado de forma predeterminada para la sesión actual. 

```powershell
gcloud auth list
```

Nos saldrá una pequeña ventana en la que deberemos clicar en autorizar.

![1](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/7852feb7-c575-4382-b84a-f7f9ef50830f)

## Tarea 1: Creamos una instancia jumphost para este proyecto

Nos dirigimos al menú de navegación y clicamos en `Compute Engine` y seleccionamos `VM Instance`. Configuraremos nuestra máquina de la siguiente manera:

1. Nombre: nucleus-jumphost-671.
2. Región: us-central1.
3. Zona: us-central1-b.
4. Tipo de Máquina: e2-micro.
5. Boot Disk: dejamos el que viene por defecto (Debian/Linux).
6. El resto de configuraciones las dejamos por defecto.

Clicamos en crear y esperamos un par de minutos hasta que se cree nuestra máquina virtual.

![2](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/d41dffba-7219-47f2-b797-55838d1a1234)

Para construir el cluster de Kubernetes, vamos a usar el comando `gcloud container clusters create`. Este comando nos permite crear un clúster de Kubernetes en Google Kubernetes Engine.

Los requisitos son los siguientes:

- Creamos un cruster en la zona us-central1-b.
- Uso de contendero Docker hello-app (gcr.io/google-samples/hello-app:2.0) como marcador de posición; el equipo reemplazará el contenedor con su propio trabajo más adelante.
- Exponemos la aplicación en el puerto 8080.

Para ello, primero configuramos la zona:

```powershell
gcloud config set compute/zone us-central1-b
```

Y ahora creamos el cluster:

```powershell
gcloud container clusters create nucleus-webserver1
```
![3](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/84298312-c7c8-43d7-bc61-9f689dc85340)

Le asignamos credenciales de administrador:

```powershell
gcloud container clusters get-credentials nucleus-webserver1
```

Y desplegamos el cluster:

```powershell
kubectl create deployment hello-app --image=gcr.io/google-samples/hello-app:2.

kubectl expose deployment hello-app --type=LoadBalancer --port 8081

kubectl get service
```

![4](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/c431abff-ce1f-4042-ac84-05a5bf8b88c2)

## Tarea 3: Configuramos una balanceador de carga HTTP

Podemor servir el sitio a través de servidores web nginx, pero queremos asegurarnos de que el entorno sea tolerante a fallos. Para ello, crearemos un balanceador de carga HTTP con un grupo de instancias administradas de 2 servidores web nginx. Usaremos el siguiente código para configurar los servidores web; el equipo lo reemplazará con su propia configuración más adelante.

```powershell
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```
![5](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/d91b3db5-fa86-4561-b436-af3e6db6b6df)

Vamos a necesitar:

- Una template para la instance. No usaremos el tipo de máquina por defecto. En su lugar, usaremos `e2-medium`.

```powershell
gcloud compute instance-templates create nginx-template --metadata-from-file startup-script=startup.sh
```

![6](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/2cad6ea6-7482-4fbb-9201-c7bbe197bdda)

- Un target pool.

```powershell
gcloud compute target-pools create nginx-pool
```

(Si la localización no es la misma, podemos presionar N para cambiarla, ya que normalmente toma la localización predeterminada) y elegimos el nombre de la localización.

![7](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/8f2b1b1d-3a30-4146-8e5b-9e23533d5057)

```powershell	
gcloud compute instance-groups managed create nginx-group --base-instance-name nginx --size 2 --template nginx-template --target-pool nginx-pool

gcloud compute instances list
```

![8](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/5976a2f3-4409-4c9d-876a-0416951ecac1)

![9](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/229fc821-e8d2-4c69-9602-abf277580375)

- Una regla de firewall que permita el tráfico (80/tcp).

```powershell
gcloud compute firewall-rules create permit-tcp-rule-636 --allow tcp:80

gcloud compute forwarding-rules create nginx-lb --region us-central1 --ports=80 --target-pool nginx-pool

gcloud compute forwarding-rules list
```

![10](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/61f6a3a7-16b7-4b10-b3c1-17897ac22d3d)


- Una regla de firewall que permita el tráfico (80/tcp).

```powershell
gcloud compute firewall-rules create permit-tcp-rule-636 --allow tcp:80

gcloud compute forwarding-rules create nginx-lb --region us-central1 --ports=80 --target-pool nginx-pool

gcloud compute forwarding-rules list
```

![11](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/53d0c24d-abf8-4c15-89f6-91ee4fcd49f7)

- Un servicio de backend y un unirlo al grupo de instancias administradas (http:80).

```powershell	
gcloud compute backend-services create nginx-backend --protocol HTTP --http-health-checks http-basic-check --global

gcloud compute backend-services add-backend nginx-backend --instance-group nginx-group --instance-group-zone us-central1-b --global
```

![12](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/f586f31e-c62c-4981-944b-d5700d241628)

- Una URL map y un target HTTP proxy para enrutar las solicitudes al servicio de backend.

```powershell
gcloud compute url-maps create web-map --default-service nginx-backend

gcloud compute target-http-proxies create http-lb-proxy --url-map web-map
```

![13](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/92874a23-e229-439f-bda0-034ee64fa388)

- Una regla de reenvío que use el proxy como destino.(forwarding rule).

```powershell
gcloud compute forwarding-rules create http-content-rule --global --target-http-proxy http-lb-proxy --ports 80

gcloud compute forwarding-rules list
```

![14](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/3a6d8fb7-9192-4018-9bee-0d838c778631)

Tras toda esta configuración, deberemos esperar entre 5 y 7 minutos para que se configuren todos los recursos.

![15](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/d50c44e4-e1a2-4b51-988c-82878d756126)

![16](https://github.com/Legnakra/Create-and-Manage-Cloud-Resources-Course/assets/98739593/418dfaca-8881-4eee-8296-c3a6da9f5e06)

## Resumen

Tras esto, podemos decir que hemos creado un proyecto de Google Cloud Platform con una instancia jumphost, un cluster de Kubernetes y un balanceador de carga HTTP.

## Autora :computer:
* María Jesús Alloza Rodríguez
