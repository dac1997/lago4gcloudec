# lago4gcloudec

kubectl apply -f https://raw.githubusercontent.com/argoproj/argo/stable/manifests/quick-start-postgres.yaml

kubectl create clusterrolebinding amamos-al-diego --clusterrole=cluster-admin --user=@gmail.com

kubectl patch configmap workflow-controller-configmap -n argo --patch "$(cat patch-workflow-controller-configmap.yaml)"

gcloud compute disks create --size=200GB --zone=us-central1-c gce-nfs-disk-1

kubectl apply -f 001-nfs-server.yaml

kubectl apply -f 002-nfs-server-service.yaml

kubectl get svc nfs-server-1

kubectl apply -f 003-pv-pvc.yaml

kubectl get pvc nfs-1

kubectl apply -f pv-pod.yaml

mkdir conf.d
cd conf.d
curl -sLO https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/conf.d/nginx-basic.conf
cd ..
kubectl create configmap basic-config --from-file=conf.d

kubectl create  -f deployment-http-fileserver.yaml
kubectl expose deployment http-fileserver --type LoadBalancer --port 80 --target-port 80

kubectl exec http-fileserver-XXXXXXXX-YYYYY -n argo -- rm /usr/share/nginx/html/index.html
