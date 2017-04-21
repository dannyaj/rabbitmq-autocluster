# rabbitmq-autocluster on Kubernetes (insecure kubernetes using 8080)
 RabbitMQ cluster with Kubernetes Backend
# Step 1 Build your own image
Clone kuberstack

git clone https://github.com/kuberstack/kubernetes-rabbitmq-autocluster.git

Modify kubernetes-rabbitmq-autocluster/Dockerfile

FROM rabbitmq:latest

ENV RABBITMQ_USE_LONGNAME=true \
    AUTOCLUSTER_LOG_LEVEL=debug \
    AUTOCLUSTER_CLEANUP=true \
    CLEANUP_INTERVAL=60 \
    CLEANUP_WARN_ONLY=false \
    AUTOCLUSTER_TYPE=k8s \
    LANG=en_US.UTF-8

ADD plugins/*.ez /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.9/plugins/

RUN rabbitmq-plugins enable --offline autocluster
RUN rabbitmq-plugins enable --offline rabbitmq_management

# Build Docker image
docker build -t dannyaj/rabbitmq-autocluster .
# push image
docker push dannyaj/rabbitmq-autocluster

# Step 2 Create a erlang.cookie in secret
echo $(openssl rand -base64 32) > erlang.cookie

kubectl -n $namespace create secret generic erlang.cookie --from-file=erlang.cookie
# Step 3 Modify kubernetes-rabbitmq-autocluster/manifestos/rabbitmq-svc.yaml 

apiVersion: v1 \
kind: Service \
metadata:\
  namespace: rabbitmq-autocluster\
  labels:\
    app: rabbitmq\
  name: rabbitmq\
spec:\
  type: NodePort\
  ports:
  - port: 5672
    name: port-5672
  - port: 4369
    name: port-4369
  - port: 5671
    name: port-5671
  - port: 15672
    name: port-15672
  - port: 25672
    name: port-25672 \
  selector:\
    app: rabbitmq\
  externalIPs:
    - 10.10.1.216
   
# Step 4 Modify kubernetes-rabbitmq-autocluster/manifestos/rabbitmq-deploy.yaml

apiVersion: apps/v1beta1 \
kind: StatefulSet\
metadata:\
  namespace: rabbitmq-autocluster\
  name: rabbitmq\
spec:\
  serviceName: rabbitmq-cluster\
  replicas: 3\
  template:\
    metadata:\
      labels:\
        app: rabbitmq\
    spec:\
      containers:\
      - name: rabbitmq \
        image: dannyaj/rabbitmq-autocluster\
        ports:\
          - containerPort: 5672 \
            name: port-5672\
          - containerPort: 4369\
            name: port-4369\
          - containerPort: 5671\
            name: port-5671\
          - containerPort: 15672\
            name: port-15672\
          - containerPort: 25672\
            name: port-25672\
        env:\
          - name: HOSTNAME\
            valueFrom:\
             fieldRef:\
              fieldPath: status.podIP\
          - name: MY_POD_IP\
            valueFrom:\
             fieldRef:\
              fieldPath: status.podIP\
          - name: AUTOCLUSTER_LOG_LEVE\
            value: "debug"\
          - name: AUTOCLUSTER_CLEANUP\
            value: "true"\
          - name: CLEANUP_INTERVAL\
            value: "30"\
          - name: CLEANUP_WARN_ONLY\
            value: "false"\
          - name: AUTOCLUSTER_TYPE\
            value: "k8s"\
          - name: K8S_SCHEME\
            value: "http"\
          - name: K8S_HOST\
            value: "10.10.1.10"\
          - name: K8S_PORT\
            value: "8080"\
          - name: RABBITMQ_ERLANG_COOKIE\
            valueFrom:\
             secretKeyRef:\
              name: erlang.cookie\
              key: erlang.cookie
              
 
 
# Step 5 Running
kubectl create -f kubernetes-rabbitmq-autocluster/rabbitmq-namespace.yaml

kubectl create -f kubernetes-rabbitmq-autocluster/manifestos/

# Step 6 Clustering test

FIRST_POD=$(kubectl get pods -n $namespace -l 'app=rabbitmq' -o jsonpath='{.items[0].metadata.name }')

kubectl -n $namespace exec -ti $FIRST_POD rabbitmqctl cluster_status

Cluster status of node 'rabbit@17.17.15.7' ...
[{nodes,[{disc,['rabbit@17.17.15.7','rabbit@17.17.15.8','rabbit@17.17.45.7',
                'rabbit@17.17.59.5','rabbit@17.17.94.4']}]},
 {running_nodes,['rabbit@17.17.15.8','rabbit@17.17.59.5','rabbit@17.17.45.7',
                 'rabbit@17.17.94.4','rabbit@17.17.15.7']},
 {cluster_name,<<"rabbit@rabbitmq-0.rabbitmq-cluster.rabbitmq-autocluster.svc.k8s.kidscrape">>},
 {partitions,[]},
 {alarms,[{'rabbit@17.17.15.8',[]},
          {'rabbit@17.17.59.5',[]},
          {'rabbit@17.17.45.7',[]},
          {'rabbit@17.17.94.4',[]},
          {'rabbit@17.17.15.7',[]}]}]
          
 # Step 7 Scaling test 
 
 kubectl scale StatefulSet rabbitmq --replicas=5
 
 # Reference
 
 kuberstack/kubernetes-rabbitmq-autocluster (yml files and dockerfile)\
https://github.com/kuberstack/kubernetes-rabbitmq-autocluster\
kuberstack/kubernetes-rabbitmq-autocluster (build rabbitmq image)\
https://hub.docker.com/r/kuberstack/kubernetes-rabbitmq-autocluster/\
rabbitmq-autocluster (config setting)\
https://github.com/rabbitmq/rabbitmq-autocluster

 
