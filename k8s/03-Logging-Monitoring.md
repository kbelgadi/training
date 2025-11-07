## Monitor cluster components

no embeded monitoring solution\
need to install monitoring solutions: metrics server (original deprecated: heapster), prometheus, elastic stack, datadog, dynatrace\
metrics server: in memory (no persistance storage)\
metrics are provided by cAdvisor included into kubelet: retrieve pods performance metrics

install metrics server:
git clone metrics server github repo
kubectl apply -f ...

To display metrics:
kubectl top node --> cpu/ram usage on nodes
kubectl top pods --> cpu/ram usage by each pod

## Application logs

for a container runnning standalone: docker logs\
for a pod: kubectl logs -f <pod name> <container name if multiple containers>