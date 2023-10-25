    1  cat /proc/meminfo 
    2  cat /proc/cpuinfo 
    3  cat /proc/cpuinfo | less
    4  ls -l /var/
    5  ls -l /var/log
    6  cat /var/log/README 
    7  su -
    8  ip a
    9  vi .bash_rc
   10  vi .bashrc 
   11  su -
   12  mkdir initialsetup
   13  su -
   14  vi initialsetup/notes
   15  exit
   16  ssh-add -l
   17  sudo apt update
   18  sudo apt install curl

## install minikube

   19  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   20  sudo install minikube-linux-amd64 /usr/local/bin/minikube

## start minikube

   21  minikube start

### no docker!

   22  apt search docker | less
   23  sudo apt install docker.io

### not in docker group!

   24  sudo vi /etc/group
   25  minikube start

### must relog for group :/

   26  exit

### woot

   28  minikube start
   29  v1.28.2
   30  sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   31  kubectl get pods -A
   32  mkdir dgraph
   33  cd dgraph/
   34  vi oldnotes
   35  kubectl create deployment dgraph-testo --image=dgraph/standalone:v23.1.0
   36  kubectl get pods
   37  kubectl logs dgraph-testo-c7b96c796-844dv
   38  kubectl logs dgraph-testo-c7b96c796-844dv | grep HTTP
   39  kubectl get deployments
   40  kubectl expose deployment/dgraph-testo --type="NodePort" --port 8080
   41  kubectl get services
   42  minikube ip
   43  curl 192.168.49.2:32428
   44  curl 192.168.49.2:32428/admin
   45  kubectl describe nodes
   46  curl 192.168.49.2:32428/admin
   47  sudo apt install nginx
   48  vi oldnotes 
   49  sudo vi /etc/nginx/sites-available/default 
   50  sudo nginx -s reload
   51  curl localhost/dgraph/admin
   52  history > minikube-dgraph.md
