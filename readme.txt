k create ns mynamespace --dry-run=client -o yaml > basic/create-ns.yaml
k run nginx --image=nginx --dry-run=client -o yaml > basic/nginx.yaml
k create deploy nginx --image=nginx --dry-run=client -o yaml > basic/deploy.yaml
