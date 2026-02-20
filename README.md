<p align="center"><img width="180" alt="portfolio_view" src="badge.png"></p>
<h4 align="center"><a href="https://www.cncf.io/certification/ckad/">https://www.cncf.io/certification/ckad/</a></h3>
<h1 align="center">Certified Kubernetes Application Developer (CKAD) Notes</h1>

# CKAD Notes

## 1. Core Concepts
## 2. Configuration
## 3. Multi-Container Pods
## 4. Observability
## 5. Pod Design
## 6. Services & Networking
## 7. State Persistence
## 8. Security
## 9. Exam Tips & Speed Tricks

### Aliases
```bashrc
# --- Core ---
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias krm='kubectl delete'

# --- Apply / Create ---
alias ka='kubectl apply -f'
alias kcf='kubectl create -f'
alias kc='kubectl create'
alias kr='kubectl run'

# --- Logs / Exec ---
alias kl='kubectl logs'
alias ke='kubectl exec -it'

# --- Namespaces ---
alias kn='kubectl config set-context --current --namespace'

# --- YAML generation ---
export do='--dry-run=client -o yaml'
```

### Tips



