# MODELOS DE IA NO KUBERNETES COM O OLLAMA

Repositório criado com base nos vídeos da [@LinuxTips](https://www.youtube.com/@LinuxTips/videos). 
Esse repositório contém o passo-a-passo para subir uma IA localmente utilizando o Kubernetes. Incluíndo a instalação das ferramentas e configuração do minikube para utilizar a nvidia como gpu.

# **Instalando e customizando o Kubectl, Minikube e Helm**

## **Instalação do Kubectl no GNU/Linux**

Vamos instalar o `kubectl` com os seguintes comandos.

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

```

### **Customizando o kubectl**

**Auto-complete**

Execute o seguinte comando para configurar o alias e autocomplete para o `kubectl`.

No Bash:

```
source <(kubectl completion bash) # configura o autocomplete na sua sessão atual (antes, certifique-se de ter instalado o pacote bash-completion).

echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanentemente ao seu shell.
```

### No ZSH:

```
source <(kubectl completion zsh)echo "[[ $commands[kubectl] ]] && source <(kubectl completion zsh)"
```

### **Criando um alias para o kubectl**

Crie o alias `k` para `kubectl`:

```
alias k=kubectl

complete -F __start_kubectl k

```

## Instalando o Minikube

Minikube é o Kubernetes local, com foco em facilitar o aprendizado e o desenvolvimento para o Kubernetes.

Tudo que você precisa é de um contêiner Docker (ou compatível de forma semelhante) ou de um ambiente de máquina virtual, e o Kubernetes está a um único comando de distância:`minikube start`

```jsx
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

## **Installing the NVIDIA Container Toolkit**

1. Configure o repositório de produção:
    
    **`$** curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg **\**&& curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | **\**
        sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | **\**
        sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list`
    
2. Atualize a lista de pacotes do repositório:
    
    **`$** sudo apt-get update`
    
3. Instale os pacotes NVIDIA Container Toolkit:
    
    **`$** sudo apt-get install -y nvidia-container-toolkit`
    
    ### Configurando o Docker
    
    1. Configure o ambiente de execução do contêiner usando o `nvidia-ctk`comando:
        
        **`$** sudo nvidia-ctk runtime configure --runtime=docker`
        
        O `nvidia-ctk`comando modifica o `/etc/docker/daemon.json`arquivo no host. O arquivo é atualizado para que o Docker possa usar o NVIDIA Container Runtime.
        
    2. Reinicie o daemon do Docker:
        
        **`$** sudo systemctl restart docker`
        
        3. Start minikube:
        
        `minikube start --driver docker --container-runtime docker --gpus all`
        
        Comandos básicos do minikube:
        
        ```yaml
            minikube delete
            minikube start
            minikube dashboard
            k get nodes
            k describe nodes | grep -i gpu
        ```
        
        ## **Instalando o Helm**
        
        O Helm agora conta com um *script* que automaticamente baixará a última versão disponível do Helm e [instalará localmente](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3).
        
        ```
        $ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
        $ chmod 700 get_helm.sh
        $ ./get_helm.sh
        ```
        
        Adicionar o Ingress Controller, no caso o Nginx Ingress Controller. 
        
        ```jsx
        minikube addons enable ingress
        ```
        
        Listas o Nginx Ingress Controller:
        
        ```jsx
        k get pods -n ingress-nginx
        ```
        
        # Criando o Ollama
        
        Criar o namespace
        
        ```jsx
        k create ns ollama
        k get ns
        ```
        
        Para adicionar o repositório do Ollama, você pode utilizar o seguinte comando:
        
        ```bash
        helm repo add ollama-helm https://otwld.github.io/ollama-helm/
        helm repo update
        ```
        
        Vamos criar o nosso arquivo `values.yaml` para configurar o Ollama
        
        ```yaml
        ollama:
          gpu:
            # -- Enable GPU integration
            enabled: true
        
            # -- GPU type: 'nvidia' or 'amd'
            type: 'nvidia'
        
            # -- Specify the number of GPU to 2
            number: 1
        
          # -- List of models to pull at container startup
          models:
            - codellama
        
        ingress:
          enabled: true
          hosts:
            - host: ollama.cascao.io
              paths:
                - path: /
                  pathType: Prefix
        ```
        
        instalar o Ollama no Kubernetes com o Helm. Para isso, você pode utilizar o seguinte comando:
        
        ```bash
        helm install ollama ollama-helm/ollama -n ollama -f ollama/values.yaml
        ```
        
        A saída será algo como:
        
        ```bash
        NAME: ollama
        LAST DEPLOYED: Wed Mar 27 20:13:29 2024
        NAMESPACE: ollama
        STATUS: deployed
        REVISION: 1
        NOTES:
        1. Get the application URL by running these commands:
          http://ollama.cascao.io
        ```
        
        Para acompanhar a criação:
        
        ```jsx
        k get pods -n ollama -w
        ```
        
        Agora vamos setar a variável de ambiente `OLLAMA_HOST` para que o CLI do Ollama possa interagir com o Ollama que está no Kubernetes.
        
        ```bash
        export OLLAMA_HOST=http://ollama.cascao.io
        ```
        
        Inclusive, caso você esteja executando o Kubernetes em um ambiente local, você pode adicionar o endereço no seu arquivo /etc/hosts para que você possa acessar o Ollama pelo endereço configurado no Ingress.
        
        No caso do Minikube, basta digitar o seguinte comando:
        
        ```jsx
        echo "$(minikube ip) [ollama.cascao.io](http://ollama.badtux.io/)" | sudo tee -a /etc/hosts
        ```
        
        Quando os pods estiverem prontos, você poderá acessar o Ollama pelo endereço que você configurou no Ingress. No meu caso, eu posso acessar o Ollama pelo endereço [http://ollama.cascao.io/](http://ollama.badtux.io/).
        
        Vamos fazer um curl para verificar se o Ollama está funcionando corretamente:
        
        ```jsx
        curl [http://ollama.cascao.io/](http://ollama.badtux.io/)
        ```
        
        A saída será algo como:
        
        ```jsx
        Ollama is running%
        ```
        
        Agora você já pode usar o CLI do Ollama para interagir com o Ollama que está em execução no Kubernetes.
        
        ```jsx
        ollama run codellama
        ```
        
        # Criando o Open-WebUI
        
        Criar o namespace
        
        ```jsx
        k create ns open-webui
        k get ns
        ```
        
        Vamos criar o nosso arquivo `open-webui.yaml` para criar o Namespace para o Open WebUI:
        
        ```yaml
        apiVersion: v1
        kind: Namespace
        metadata:
          name: open-webui
        
        ```
        
        Criar o arquivo webui-deployment.yaml
        
        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: open-webui-deployment
          namespace: open-webui
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: open-webui
          template:
            metadata:
              labels:
                app: open-webui
            spec:
              containers:
              - name: open-webui
                image: ghcr.io/open-webui/open-webui:main
                ports:
                - containerPort: 8080
                resources:
                  requests:
                    cpu: "500m"
                    memory: "500Mi"
                  limits:
                    cpu: "1000m"
                    memory: "1Gi"
                env:
                - name: OLLAMA_BASE_URL
                  value: "http://ollama.cascao.io"
                tty: true
                volumeMounts:
                - name: webui-volume
                  mountPath: /app/backend/data
              volumes:
              - name: webui-volume
                persistentVolumeClaim:
                  claimName: ollama-webui-pvc
        ```
        
        Criar o arquivo webui-service.yaml
        
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: open-webui-service
          namespace: open-webui
        spec:
          type: NodePort  # Use LoadBalancer if you're on a cloud that supports it
          selector:
            app: open-webui
          ports:
            - protocol: TCP
              port: 8080
              targetPort: 8080
              # If using NodePort, you can optionally specify the nodePort:
              # nodePort: 30000
        
        ```
        
        E para o Ingress, vamos criar o arquivo `webui-ingress.yaml`:
        
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: open-webui-ingress
          namespace: open-webui
          #annotations:
            # Use appropriate annotations for your Ingress controller, e.g., for NGINX:
            # nginx.ingress.kubernetes.io/rewrite-target: /
        spec:
          rules:
          - host: open-webui.cascao.io
            http:
              paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: open-webui-service
                    port:
                      number: 8080
        
        ```
        
        Agora, para finalizar, vamos criar o arquivo webui-pvc.yaml para criar o PersistentVolumeClaim para o Open WebUI:
        
        ```yaml
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          labels:
            app: ollama-webui
          name: ollama-webui-pvc
          namespace: open-webui
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 2Gi
        ```
        
        Agora vamos aplicar esses arquivos e torcer para que não tenhamos digitados nada errado.
        
        ```jsx
        kubectl apply -f open-webui/open-webui.yaml
        kubectl apply -f open-webui/webui-deployment.yaml
        kubectl apply -f open-webui/webui-service.yaml
        kubectl apply -f open-webui/webui-ingress.yaml
        kubectl apply -f open-webui/webui-pvc.yaml
        
        ```
        
        Vamos esperar os pods ficarem prontos para acessar o Open WebUI. Para verificar o status dos pods, você pode utilizar o seguinte comando:
        
        ```jsx
        
        kubectl get pods -n open-webui
        
        ```
        
        Acompanhar o deploy
        
        ```bash
        # Descrição do pod
        k describe pods -n open-webui open-webui-deployment-b9d5596-f68cv
        
        # PVC
        k get pvc -A
        ```
        
        Inclusive, caso você esteja executando o Kubernetes em um ambiente local, você pode adicionar o endereço no seu arquivo `/etc/hosts` para que você possa acessar o Ollama pelo endereço configurado no Ingress.
        
        No caso do Minikube, basta digitar o seguinte comando:
        
        ```bash
        echo "$(minikube ip) open-webui.cascao.io" | sudo tee -a /etc/hosts
        ```
        
        Quando tudo estiver ok, já podemos acessar o Open WebUI através do endereço que configuramos no Ingress. No meu caso, eu posso acessar o Open WebUI pelo endereço `http://open-webui.cascao.io`.
        
        ![https://lwfiles.mycourse.app/633c72fac8c963ec854a3950-public/20108db9040d1cd20613105ef4006729.png](https://lwfiles.mycourse.app/633c72fac8c963ec854a3950-public/20108db9040d1cd20613105ef4006729.png)
        
        Como ainda não temos um usuário criado, você precisa criar um usuário para acessar o Open WebUI. Depois de criar o usuário, você poderá acessar a interface gráfica e interagir com a sua IA de uma forma muito mais amigável e visual. Os dados são armazenados no PersistentVolumeClaim que criamos, e não é compartilhado com ningúem, então fique tranquilo.
        
        Depois de criado a sua conta, você poderá acessar o Open WebUI e enviar mensagens para a sua IA, ela irá responder com base no que ela aprendeu durante o treinamento e de acordo com o modelo que você está utilizando.
        
        ![https://lwfiles.mycourse.app/633c72fac8c963ec854a3950-public/ee4565c91dbd1900345e9c3f0495c19c.png](https://lwfiles.mycourse.app/633c72fac8c963ec854a3950-public/ee4565c91dbd1900345e9c3f0495c19c.png)
        
        Na parte superior da tela, você pode ver o nome do modelo que você está utilizando, nós estamos utilizando o modelo Llama2, que definimos lá no arquivo `values.yaml` do Ollama. Se precisar de outros, você pode adicionar na lista de modelos no chart Helm do Ollama e depois fazer o `helm upgrade` para atualizar o Ollama no Kubernetes. É bem simples!