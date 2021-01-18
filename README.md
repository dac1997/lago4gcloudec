# lago4gcloudec

## Preparacion de cluster Kubernetes en Google cloud:

En google Cloud podemos crear un cluster Kubernetes accediendo a esta seccion y configurando con los parametros requeridos. Se recomienda hacer uso de autoscaling para mejor manejo de los recursos y de un numero de nodos comparable a la cantidad de trabajos que se vayan a realizar, en especial si estos son trabajos pesados como las simulaciones de una hora de fluencia.

Una vez que tenemos un cluster preparado, entramos a traves del Cloud Shell y vamos a aplicar los siguientes archivos de configuracion en nuestro cluster para manejar argo:

	kubectl apply -f https://raw.githubusercontent.com/argoproj/argo/stable/manifests/quick-start-postgres.yaml

El siguiente script es una serie de comandos que instalan argo. Es posible tener que volver a correrlo cada vez que se vuelve a acceder al cluster, entonces por conveniencia se los incluye en este script de bash

	bash Cluster_Setup/argo.sh

La siguiente linea define que el usuario de google es administrador del cluster y tiene los permisos necesarios

	kubectl create clusterrolebinding corsika admin --clusterrole=cluster-admin --user=youruser@gmail.com

Hacemos patch de la configuracion de argo

	kubectl patch configmap workflow-controller-configmap  --patch "$(cat patch-workflow-controller-configmap.yaml)"

## Creacion de un disco para guardar resultados

El siguiente comando crea un disco para almacenar datos. Ajuste el tamaño del disco segun lo que vaya a nacesitar, tomando en cuenta que la salida total de la simulacion de una hora de fluencia es de al rededor de 300 GB (Se puede disminuir de manera muy significativa si se cambia el script rain.pl para no producir archivos .long). Ademas, por conveniencia, vale la pena escoger la misma zona donde se coloco el cluster

	gcloud compute disks create --size=300GB --zone=us-central1-c gce-nfs-disk-1

Si se utiliza un nombre diferente para el disco es necesario modificar los archivos 001-nfs-server.yaml y 002-nfs-server-service.yaml antes de usarlos

	kubectl apply -f Cluster_Setup/001-nfs-server.yaml

	kubectl apply -f Cluster_Setup/002-nfs-server-service.yaml

Luuego necesitamos obtener la IP para nuestro servidor que se consigue a partir del siguiente comando:

	kubectl get svc nfs-server-1

Es necesario modificar el archivo 003-pv-pvc.yaml para incluir el IP obtenido ademas de cambiar el tamaño del disco para el tamaño que se necesite y lugo se aplica:

	kubectl apply -f Cluster_Setup/003-pv-pvc.yaml

Podemos Chequear que esta listo cuando la salida del siguiente comando incluye 'Bound'

	kubectl get pvc nfs-1

Creamos un pod que nos permita acceder a nuestros archivos, asegurandonos que el nombre del claim en la configuracion cincide con el nuestro:

	kubectl apply -f Cluster_Setup/pv-pod.yaml

Ahora creamos un servidor http para descargar los archivos, notar que el servidor es publico, lo que puede ser tanto una ventaja como una desventaja. Ademas, la descarga consume credito. Para reducir el tiempo y el consumo en el caso de CORSIKA, solo descargar los binarios, a menos que en realidad hagan falta los demás

	mkdir conf.d
	cd conf.d
	curl -sLO https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/conf.d/nginx-basic.conf
	cd ..
	kubectl create configmap basic-config --from-file=conf.d
	kubectl create  -f Cluster_Setup/deployment-http-fileserver.yaml
	kubectl expose deployment http-fileserver --type LoadBalancer --port 80 --target-port 80

Buscamos entre los pods el que corresponde a nuestro servidor http y reemplazamos la serie de numeros en el siguiente comando:

	kubectl exec http-fileserver-XXXXXXXX-YYYYY -n argo -- rm /usr/share/nginx/html/index.html


y el cluster esta listo para correr simulaciones usando argo y acceder a los resultados a traves de http

## Correr Simulaciones preparadas

Esta incluida el directorio 'Simulation_yaml_Launchers' que contiene los archivos yaml que llaman a realizar distintas simulaciones. Aqui vamos a utilizar el de prueba:

Primero vamos a entender que hace el codigo, por ejemplo 'testprotons.yaml':

el api de argo y el tipo de tarea que estamos realizando que es un Flujo de trabajo

	apiVersion: argoproj.io/v1alpha1
	kind: Workflow

Generamos un nombre para nuestro pod a partir de test-. Es decir el nombre del pod y el workflow de argo sera test-XXXX

	metadata:
 	 generateName: test-
 
Especificamos los volumenes que se van a utilizar
  
	spec:
  	entrypoint: test
  		volumes:
  		- name: task-pv-storage
   		 persistentVolumeClaim:
     		 claimName: nfs-1
    
El template señala los contenedores que se van a utilizar  y los compando que se van a correr, el nombre de este template es test y la imagen que usamos proviene de mi DockerHub

  	templates:
  	- name: test
    		script:
     		 image: docker.io/dacb1997/lagocloudtest:test
      		command: [sh]
      		source: |
      
Ejecuta el comando del primer grupo de simulaciones

        bash go-test-pr-1.sh 
	
Dado que utiliza screen, chequeamos si hay screens siendo utilizadas, si aun estan, esperamos una hora. Las simulaciones de una hora de fluencia suelen tardar una buena cantidad de horas, esta de prueba va a demorar menos, entonces podria modificarse el tiempo de espera por conveniencia.

        while screen -list | grep -q "There are screens on"
        do
        sleep 3600
        screen -list 
        done 
        echo "First Ready"
	
Cuando esta listo, movemos los archivos resultantes a /mnt/vol/ que es donde esta montado nuestro volumen persistente. Liberando espacio en nuestro contenedor, evitando que luego el orquestrador de kubernetes nos expulse el pod

        mv  test/DAT* /mnt/vol/
	
Mostramos los archivos que tenemos por el momento en el volumen

        echo ls -l /mnt/vol
        ls -l /mnt/vol
        
Indicamos donde se monta el volumen persistente, debe ser consistente
        
      volumeMounts:
      - name: task-pv-storage
        mountPath: /mnt/vol
        
Ahora que lo hemos entendido, podemos enviar el trabajo utilizando

	argo submit testprotons.yaml
	
## Crear tu propio Contenedor con CORSIKA y ARTI

Es posible crear tu propia imagen para crear un contenedor listo para preparar tus simulaciones utilizando [ARTI](https://github.com/lagoproject/arti)
