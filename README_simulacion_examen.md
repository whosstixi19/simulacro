# Simulación de examen: Docker, CI/CD, GHCR, Kubernetes y Canary

Este repositorio utiliza la aplicación `inventario-app` como punto de partida. La aplicación contiene una interfaz web, una API REST en Node.js y almacenamiento local en `data/products.json`.

## Flujo general

```text
Código → GitHub → GitHub Actions → npm test → Docker build
      → Trivy → GHCR → Kubernetes → Deployment → Pods → Service
```

Imagen empleada:

```text
ghcr.io/carlosmoyanog/practicaexamen-inventario-app
```

> Sustituya nombres, hashes, puertos y URL cuando corresponda.

---

## 1. Requisitos

```powershell
node --version
npm --version
git --version
docker --version
kubectl version --client
minikube version
```

Se requiere Node.js, Docker Desktop, Git, kubectl, Minikube y una cuenta de GitHub.

---

## 2. Aplicación local

Instalar dependencias y ejecutar pruebas:

```powershell
npm ci
npm test
```

Ejecutar:

```powershell
npm start
```

Verificar:

```powershell
curl.exe http://localhost:3000/
curl.exe http://localhost:3000/health
curl.exe http://localhost:3000/version
curl.exe http://localhost:3000/api/products
```

---

## 3. Docker

Construir:

```powershell
docker build -t inventario-app:v1 .
```

Sin caché:

```powershell
docker build --no-cache -t inventario-app:v1 .
```

Ejecutar:

```powershell
docker run --name inventario-app -p 3000:3000 -e APP_VERSION=v1 -e APP_COLOR=blue inventario-app:v1
```

En segundo plano:

```powershell
docker run -d --name inventario-app -p 3000:3000 -e APP_VERSION=v1 -e APP_COLOR=blue inventario-app:v1
```

Verificar:

```powershell
curl.exe http://localhost:3000/health
curl.exe http://localhost:3000/version
curl.exe http://localhost:3000/api/products
```

Diagnóstico:

```powershell
docker ps
docker ps -a
docker logs inventario-app
docker logs -f inventario-app
docker inspect inventario-app
docker port inventario-app
docker exec -it inventario-app sh
```

Dentro del contenedor:

```sh
env
pwd
ls -la
ps
wget -qO- http://localhost:3000/health
exit
```

Detener y eliminar:

```powershell
docker stop inventario-app
docker rm inventario-app
docker rm -f inventario-app
```

### Reto: contenedor activo que no responde

Dockerfile defectuoso típico:

```dockerfile
FROM node20-alpine
WORKDIR app
COPY package.json .
RUN npm install
COPY . .
EXPOSE 3000
CMD [node, server.js]
```

Correcciones principales:

- `node20-alpine` → `node:20-alpine`.
- `WORKDIR app` → `WORKDIR /app`.
- Usar `package-lock.json` y `npm ci`.
- Usar sintaxis JSON válida: `CMD ["node", "server.js"]` o `CMD ["npm", "start"]`.
- Publicar el puerto real con `docker run -p PUERTO_HOST:PUERTO_CONTENEDOR`.

---

## 4. Git y GitHub

Inicializar:

```powershell
git init
git status
git add .
git commit -m "Initial inventory app with Docker support"
git branch -M main
```

Conectar con GitHub:

```powershell
git remote add origin https://github.com/USUARIO/REPOSITORIO.git
git remote -v
git push -u origin main
```

Cambios posteriores:

```powershell
git status
git diff
git add .
git commit -m "Descripción del cambio"
git push
```

Historial:

```powershell
git log --oneline
git log --oneline --graph --decorate --all
```

---

## 5. GitHub Actions y GHCR

El pipeline debe tener dos jobs encadenados:

```text
build-test → build-push
```

La línea esencial es:

```yaml
needs: build-test
```

Esto impide publicar o desplegar cuando las pruebas fallan.

### Probar fail-fast

Romper temporalmente una prueba y subirla:

```powershell
npm test
git add server.test.js
git commit -m "Break test to verify fail-fast"
git push
```

Resultado esperado:

- `build-test` falla.
- `build-push` queda omitido.
- No se publica una imagen.

Restaurar:

```powershell
git add server.test.js
git commit -m "Restore passing test"
git push
```

### Descargar la imagen desde GHCR

```powershell
docker pull ghcr.io/carlosmoyanog/practicaexamen-inventario-app:latest
```

Con hash:

```powershell
docker pull ghcr.io/carlosmoyanog/practicaexamen-inventario-app:HASH_DEL_COMMIT
```

Ejecutar:

```powershell
docker run --rm -p 3000:3000 ghcr.io/carlosmoyanog/practicaexamen-inventario-app:latest
```

---

## 6. Minikube

Iniciar Docker Desktop y ejecutar:

```powershell
minikube status
minikube start --driver=docker
kubectl config current-context
kubectl config use-context minikube
kubectl cluster-info
kubectl get nodes
```

El nodo debe aparecer en estado `Ready`.

---

## 7. Validar y aplicar Kubernetes

Validar sin crear recursos:

```powershell
kubectl apply --dry-run=client -f k8s/deployment.yaml
kubectl apply --dry-run=client -f k8s/service.yaml
kubectl apply --dry-run=client -f k8s/
```

Mostrar YAML procesado:

```powershell
kubectl apply --dry-run=client -f k8s/deployment.yaml -o yaml
```

Aplicar:

```powershell
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

O:

```powershell
kubectl apply -f k8s/
```

Esperar:

```powershell
kubectl rollout status deployment/inventario-app
```

Consultar:

```powershell
kubectl get deployments
kubectl get replicasets
kubectl get pods
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl get services
kubectl get all
```

Filtrar:

```powershell
kubectl get pods -l app=inventario-app
```

---

## 8. Diagnóstico de pods

```powershell
kubectl describe pod NOMBRE_DEL_POD
kubectl logs NOMBRE_DEL_POD
kubectl logs -f NOMBRE_DEL_POD
kubectl exec -it NOMBRE_DEL_POD -- sh
kubectl exec NOMBRE_DEL_POD -- wget -qO- http://localhost:3000/health
kubectl exec NOMBRE_DEL_POD -- cat /app/data/products.json
```

Eventos:

```powershell
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## 9. Service sin respuesta

Consultar:

```powershell
kubectl get service inventario-app-service
kubectl describe service inventario-app-service
kubectl get endpoints inventario-app-service
kubectl get endpointslices -l kubernetes.io/service-name=inventario-app-service
```

Comparar etiquetas y selector:

```powershell
kubectl get pods --show-labels
kubectl describe service inventario-app-service
```

Si aparece `<none>` en endpoints, el selector del Service no coincide con las etiquetas de los pods o los pods no están Ready.

Ejemplo correcto:

```yaml
# Deployment
template:
  metadata:
    labels:
      app: webapp
```

```yaml
# Service
selector:
  app: webapp
```

---

## 10. Acceder al Service

Windows con Minikube y driver Docker:

```powershell
minikube service inventario-app-service --url
```

Mantener esa terminal abierta.

En otra terminal:

```powershell
curl.exe http://127.0.0.1:PUERTO/
curl.exe http://127.0.0.1:PUERTO/health
curl.exe http://127.0.0.1:PUERTO/version
curl.exe http://127.0.0.1:PUERTO/api/products
```

Alternativa:

```powershell
kubectl port-forward service/inventario-app-service 8080:80
```

```powershell
curl.exe http://127.0.0.1:8080/health
```

---

## 11. Recreación de pods y pérdida de datos

Listar:

```powershell
kubectl get pods -l app=inventario-app
```

Eliminar:

```powershell
kubectl delete pod NOMBRE_DEL_POD
```

Observar recreación:

```powershell
kubectl get pods -l app=inventario-app -w
```

Salir con `Ctrl + C`.

Consultar la base local de cada pod:

```powershell
kubectl exec POD_1 -- cat /app/data/products.json
kubectl exec POD_2 -- cat /app/data/products.json
```

Conclusión:

```text
Más réplicas no implica persistencia.
Cada pod posee su propio sistema de archivos efímero.
```

---

## 12. Rolling Update

Configuración:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

Actualizar con una imagen concreta:

```powershell
kubectl set image deployment/inventario-app inventario-app=ghcr.io/carlosmoyanog/practicaexamen-inventario-app:HASH
```

Observar:

```powershell
kubectl rollout status deployment/inventario-app
kubectl get pods -w
kubectl rollout history deployment/inventario-app
```

Rollback:

```powershell
kubectl rollout undo deployment/inventario-app
kubectl rollout undo deployment/inventario-app --to-revision=NUMERO
```

---

## 13. Canary 80/20

Arquitectura:

```text
4 pods stable → v1 / blue
1 pod canary  → v2-canary / green
```

Validar:

```powershell
kubectl apply --dry-run=client -f k8s/canary/deployment-stable.yaml
kubectl apply --dry-run=client -f k8s/canary/deployment-canary.yaml
kubectl apply --dry-run=client -f k8s/canary/service.yaml
```

Aplicar:

```powershell
kubectl apply -f k8s/canary/deployment-stable.yaml
kubectl apply -f k8s/canary/deployment-canary.yaml
kubectl apply -f k8s/canary/service.yaml
```

Esperar:

```powershell
kubectl rollout status deployment/inventario-app-stable-practica-examen
kubectl rollout status deployment/inventario-app-canary-practica-examen
```

Consultar:

```powershell
kubectl get pods -l environment=practica-examen --show-labels
kubectl get pods -l environment=practica-examen,track=stable
kubectl get pods -l environment=practica-examen,track=canary
```

Service y endpoints:

```powershell
kubectl get service inventario-app-canary-service
kubectl describe service inventario-app-canary-service
kubectl get endpoints inventario-app-canary-service
kubectl get endpointslices -l kubernetes.io/service-name=inventario-app-canary-service
```

Obtener URL:

```powershell
minikube service inventario-app-canary-service --url
```

### Distribución de tráfico

Forzar una conexión nueva por petición:

```powershell
$results = 1..100 | ForEach-Object {
    $json = curl.exe -s `
        -H "Connection: close" `
        http://127.0.0.1:PUERTO/version

    $json | ConvertFrom-Json
}

$results |
    Group-Object version |
    Select-Object Name, Count
```

Mostrar cada backend:

```powershell
1..30 | ForEach-Object {
    $json = curl.exe -s `
        -H "Connection: close" `
        http://127.0.0.1:PUERTO/version

    $response = $json | ConvertFrom-Json
    Write-Host "$($response.version) | $($response.color) | $($response.hostname)"
}
```

Agrupar por hostname:

```powershell
$results |
    Group-Object hostname |
    Sort-Object Count -Descending |
    Select-Object Name, Count
```

Escalar o retirar Canary:

```powershell
kubectl scale deployment/inventario-app-canary-practica-examen --replicas=2
kubectl scale deployment/inventario-app-canary-practica-examen --replicas=0
```

Promover una imagen nueva:

```powershell
kubectl set image deployment/inventario-app-stable-practica-examen inventario-app=ghcr.io/carlosmoyanog/practicaexamen-inventario-app:HASH_NUEVO
```

---

## 14. Secret de Kubernetes

Crear sin versionar el valor:

```powershell
kubectl create secret generic inventario-app-secret-practica-examen `
  --from-literal=API_KEY="practica-api-key-2026"
```

Consultar:

```powershell
kubectl get secrets
kubectl get secret inventario-app-secret-practica-examen
kubectl describe secret inventario-app-secret-practica-examen
kubectl get secret inventario-app-secret-practica-examen -o yaml
```

Referencia en Deployment:

```yaml
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: inventario-app-secret-practica-examen
      key: API_KEY
```

Aplicar:

```powershell
kubectl apply -f k8s/canary/deployment-stable.yaml
kubectl apply -f k8s/canary/deployment-canary.yaml
```

Verificar sin revelar el valor:

```powershell
$pod = kubectl get pods `
  -l environment=practica-examen,track=stable `
  -o jsonpath="{.items[0].metadata.name}"

kubectl exec $pod -- sh -c 'env | grep -q "^API_KEY=" && echo API_KEY_cargada_correctamente'
```

```powershell
$canaryPod = kubectl get pods `
  -l environment=practica-examen,track=canary `
  -o jsonpath="{.items[0].metadata.name}"

kubectl exec $canaryPod -- sh -c 'env | grep -q "^API_KEY=" && echo API_KEY_cargada_correctamente_en_Canary'
```

---

## 15. Trivy

Flujo:

```text
construir imagen → escanear → si pasa, publicar en GHCR
```

Configuración principal:

```yaml
- name: Escanear imagen con Trivy
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    image-ref: ${{ env.REGISTRY }}/${{ steps.image.outputs.name }}:${{ github.sha }}
    format: table
    exit-code: "1"
    ignore-unfixed: true
    vuln-type: os,library
    severity: CRITICAL
```

Si encuentra una vulnerabilidad crítica:

- el paso termina con código 1;
- el login y la publicación quedan omitidos;
- debe corregirse el componente vulnerable;
- no debe cambiarse `exit-code` a `0`.

Comprobar npm dentro de una imagen corregida:

```powershell
docker run --rm inventario-app:trivy-fix npm --version
```

Comprobar el paquete `tar` interno de npm:

```powershell
docker run --rm inventario-app:trivy-fix node -p "require('/usr/local/lib/node_modules/npm/node_modules/tar/package.json').version"
```

---

## 16. Giro final: tráfico triplicado

Escalado manual:

```powershell
kubectl scale deployment/inventario-app --replicas=4
kubectl get pods -w
```

Para Stable:

```powershell
kubectl scale deployment/inventario-app-stable-practica-examen --replicas=6
```

### HPA opcional

```powershell
minikube addons enable metrics-server
kubectl top nodes
kubectl top pods
kubectl autoscale deployment inventario-app --cpu-percent=60 --min=2 --max=10
kubectl get hpa
kubectl describe hpa inventario-app
```

Eliminar:

```powershell
kubectl delete hpa inventario-app
```

Generar tráfico:

```powershell
1..500 | ForEach-Object {
    curl.exe -s -H "Connection: close" http://127.0.0.1:PUERTO/health | Out-Null
}
```

Bucle continuo:

```powershell
while ($true) {
    curl.exe -s http://127.0.0.1:PUERTO/health | Out-Null
}
```

Mientras hay tráfico:

```powershell
kubectl set image deployment/inventario-app inventario-app=ghcr.io/carlosmoyanog/practicaexamen-inventario-app:HASH_NUEVO
kubectl rollout status deployment/inventario-app
```

---

## 17. Limpieza

Canary:

```powershell
kubectl delete -f k8s/canary/
```

Base:

```powershell
kubectl delete -f k8s/service.yaml
kubectl delete -f k8s/deployment.yaml
```

Minikube:

```powershell
minikube stop
minikube delete
```

---

## 18. Checklist del simulacro

### Reto 1

- [ ] El contenedor aparece activo.
- [ ] Se revisaron logs y puerto real.
- [ ] Se corrigió el Dockerfile.
- [ ] La aplicación responde desde fuera.

### Reto 2

- [ ] Los pods están Running/Ready.
- [ ] Se revisó `describe service`.
- [ ] Se revisaron endpoints.
- [ ] Selector y etiquetas coinciden.
- [ ] Existe al menos una IP detrás del Service.

### Reto 3

- [ ] Se rompió una prueba.
- [ ] Se comprobó el defecto.
- [ ] Se agregó `needs: build-test`.
- [ ] El despliegue/publicación quedó bloqueado.
- [ ] El pipeline pasó después de corregir la prueba.

### Giro final

- [ ] Se aumentaron réplicas o se configuró HPA.
- [ ] Se aplicó Rolling Update, Blue-Green o Canary.
- [ ] Se generó tráfico durante el despliegue.
- [ ] No hubo una interrupción perceptible.
- [ ] Se guardó evidencia.

---

## 19. Commits de referencia

```powershell
git add .github/workflows/ci-cd.yml
git commit -m "Add build and test GitHub Actions job"
git push
```

```powershell
git add .github/workflows/ci-cd.yml
git commit -m "Add Docker image publishing to GHCR"
git push
```

```powershell
git add k8s/
git commit -m "Add Kubernetes base and Canary deployment manifests"
git push
```

```powershell
git add k8s/canary/deployment-stable.yaml k8s/canary/deployment-canary.yaml
git commit -m "Consume Kubernetes Secret from Canary deployments"
git push
```

```powershell
git add .github/workflows/ci-cd.yml
git commit -m "Add Trivy image security scan to CI pipeline"
git push
```

```powershell
git add Dockerfile
git commit -m "Update npm to remediate critical Trivy finding"
git push
```

---

## 20. Conceptos clave para el examen

- GitHub guarda código y manifiestos.
- GitHub Actions automatiza pruebas, build, escaneo y publicación.
- Docker empaqueta la aplicación.
- GHCR almacena imágenes.
- Deployment mantiene réplicas y actualizaciones.
- Pod ejecuta el contenedor.
- Service encuentra pods mediante etiquetas.
- Canary reparte tráfico indirectamente según el número de pods.
- `Running` no garantiza que el Service tenga endpoints.
- Más réplicas no garantizan persistencia.
- Para trazabilidad se recomienda usar el hash de la imagen, no solo `latest`.

---

# GUÍA PASO A PASO PARA RESOLVER CADA RETO

Esta sección sigue directamente la estructura del simulacro: tres retos iniciales y el giro final. Cada reto debe resolverse siguiendo el ciclo:

```text
Observar el síntoma
    ↓
Recoger evidencia
    ↓
Identificar la causa
    ↓
Corregir el artefacto
    ↓
Aplicar el cambio
    ↓
Verificar el resultado
    ↓
Guardar evidencia
```

---

## RETO 1 — El contenedor corre, pero la aplicación no responde

### Situación inicial

El build termina correctamente y el contenedor aparece activo:

```powershell
docker ps
```

Sin embargo, al probar:

```powershell
curl.exe http://localhost:3000/
```

no se obtiene respuesta.

Dockerfile defectuoso de referencia:

```dockerfile
FROM node20-alpine
WORKDIR app
COPY package.json .
RUN npm install
COPY . .
EXPOSE 3000
CMD [node, server.js]
```

### Objetivo

Determinar por qué el contenedor está activo pero la aplicación no puede alcanzarse desde el host.

### Paso 1. Construir la imagen defectuosa

```powershell
docker build -t inventario-app-reto1 .
```

### Paso 2. Ejecutar el contenedor

```powershell
docker run -d --name reto1-app -p 3000:3000 inventario-app-reto1
```

### Paso 3. Confirmar que está activo

```powershell
docker ps
docker ps -a
```

Un contenedor puede aparecer como `Up` aunque la aplicación:

- escuche en otro puerto;
- esté ligada solamente a `127.0.0.1`;
- tenga un comando de inicio incorrecto;
- no contenga archivos necesarios;
- esté reiniciándose;
- no haya arrancado correctamente.

### Paso 4. Revisar los logs

```powershell
docker logs reto1-app
```

Seguirlos en tiempo real:

```powershell
docker logs -f reto1-app
```

Buscar mensajes como:

```text
Server listening on port 3000
```

o:

```text
Server listening on port 8080
```

El puerto informado por la aplicación es el dato más importante.

### Paso 5. Entrar al contenedor

```powershell
docker exec -it reto1-app sh
```

Dentro del contenedor:

```sh
pwd
ls -la
env
ps
```

Probar la aplicación desde dentro:

```sh
wget -qO- http://localhost:3000/health
```

Si no responde, probar el puerto observado en los logs:

```sh
wget -qO- http://localhost:8080/health
```

Salir:

```sh
exit
```

### Paso 6. Revisar el mapeo de puertos

```powershell
docker port reto1-app
docker inspect reto1-app
```

La opción:

```powershell
-p 3000:8080
```

significa:

```text
puerto del host 3000 → puerto del contenedor 8080
```

Si la aplicación escucha en `8080`, publicar `3000:3000` no funciona.

### Paso 7. Corregir el Dockerfile

Ejemplo correcto para `inventario-app`:

```dockerfile
FROM node:20-alpine AS test

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci

COPY server.js db.js server.test.js ./
COPY public ./public
COPY data ./data

RUN npm test


FROM node:20-alpine AS runtime

WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3000

COPY package.json package-lock.json ./

RUN npm ci --omit=dev \
    && npm cache clean --force

COPY server.js db.js ./
COPY public ./public

RUN mkdir -p data \
    && chown -R node:node /app

USER node

EXPOSE 3000

CMD ["npm", "start"]
```

### Explicación de los errores

#### Error 1

```dockerfile
FROM node20-alpine
```

Corrección:

```dockerfile
FROM node:20-alpine
```

El nombre oficial incluye dos puntos entre la imagen y la etiqueta.

#### Error 2

```dockerfile
WORKDIR app
```

Corrección:

```dockerfile
WORKDIR /app
```

Se recomienda una ruta absoluta.

#### Error 3

```dockerfile
COPY package.json .
RUN npm install
```

Mejor:

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci
```

`npm ci` utiliza exactamente las versiones de `package-lock.json`.

#### Error 4

```dockerfile
CMD [node, server.js]
```

Corrección:

```dockerfile
CMD ["node", "server.js"]
```

o:

```dockerfile
CMD ["npm", "start"]
```

La forma JSON exige cadenas entre comillas.

#### Error 5

`EXPOSE 3000` no publica el puerto automáticamente.

Todavía se requiere:

```powershell
docker run -p 3000:3000 ...
```

### Paso 8. Reconstruir

```powershell
docker rm -f reto1-app
docker build --no-cache -t inventario-app-reto1-corregida .
```

### Paso 9. Ejecutar la versión corregida

```powershell
docker run -d `
  --name reto1-app `
  -p 3000:3000 `
  -e APP_VERSION=v1 `
  -e APP_COLOR=blue `
  inventario-app-reto1-corregida
```

### Paso 10. Verificar desde fuera

```powershell
curl.exe http://localhost:3000/
curl.exe http://localhost:3000/health
curl.exe http://localhost:3000/version
curl.exe http://localhost:3000/api/products
```

Resultado mínimo esperado:

```json
{"status":"ok"}
```

### Evidencia que debe guardarse

- `docker ps`.
- `docker logs reto1-app`.
- prueba interna con `docker exec`.
- Dockerfile antes y después.
- `curl` exitoso desde el host.

---

## RETO 2 — Los pods están Running, pero el Service no responde

### Situación inicial

Los pods aparecen:

```text
READY   STATUS
1/1     Running
```

pero el Service no responde.

Manifiesto defectuoso:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: tu-imagen:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 8080
```

### Objetivo

Comprobar por qué el Service no encuentra ningún pod aunque los pods estén funcionando.

### Paso 1. Aplicar el manifiesto defectuoso

```powershell
kubectl apply -f reto2-defectuoso.yaml
```

### Paso 2. Revisar los pods

```powershell
kubectl get pods
kubectl get pods --show-labels
```

Los pods pueden estar correctamente iniciados y aun así no pertenecer al Service.

### Paso 3. Revisar el Service

```powershell
kubectl get service web-service
kubectl describe service web-service
```

Prestar atención a:

```text
Selector:
Endpoints:
```

### Paso 4. Consultar endpoints

```powershell
kubectl get endpoints web-service
```

Resultado defectuoso esperado:

```text
ENDPOINTS   <none>
```

También:

```powershell
kubectl get endpointslices -l kubernetes.io/service-name=web-service
```

### Paso 5. Comparar selector y etiquetas

El Service busca:

```yaml
selector:
  app: webapp
```

Pero los pods tienen:

```yaml
labels:
  app: web
```

No existe coincidencia.

```text
Service busca app=webapp
Pods poseen app=web
Resultado: cero endpoints
```

### Paso 6. Corregir el manifiesto

Opción recomendada: usar `app: webapp` en todo el Deployment.

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: web-deployment

spec:
  replicas: 2

  selector:
    matchLabels:
      app: webapp

  template:
    metadata:
      labels:
        app: webapp

    spec:
      containers:
        - name: web
          image: ghcr.io/carlosmoyanog/practicaexamen-inventario-app:latest

          ports:
            - name: http
              containerPort: 3000

          readinessProbe:
            httpGet:
              path: /health
              port: http

---
apiVersion: v1
kind: Service

metadata:
  name: web-service

spec:
  type: NodePort

  selector:
    app: webapp

  ports:
    - name: http
      port: 80
      targetPort: http
      nodePort: 30080
```

Además del selector, comprobar que:

- `containerPort` coincide con el puerto real;
- `targetPort` coincide con el puerto del contenedor;
- los pods superan la readiness probe;
- la imagen es accesible.

### Paso 7. Aplicar la corrección

```powershell
kubectl apply -f reto2-corregido.yaml
```

### Paso 8. Esperar el rollout

```powershell
kubectl rollout status deployment/web-deployment
```

### Paso 9. Verificar etiquetas

```powershell
kubectl get pods --show-labels
```

Debe aparecer:

```text
app=webapp
```

### Paso 10. Verificar endpoints

```powershell
kubectl get endpoints web-service
```

Resultado esperado:

```text
10.244.x.x:3000,10.244.x.x:3000
```

O revisar:

```powershell
kubectl describe service web-service
```

### Paso 11. Acceder

```powershell
minikube service web-service --url
```

Mantener esa terminal abierta.

En otra terminal:

```powershell
curl.exe http://127.0.0.1:PUERTO/health
curl.exe http://127.0.0.1:PUERTO/version
```

Alternativa:

```powershell
kubectl port-forward service/web-service 8080:80
```

```powershell
curl.exe http://127.0.0.1:8080/health
```

### Otros diagnósticos útiles

```powershell
kubectl describe pod NOMBRE_DEL_POD
kubectl logs NOMBRE_DEL_POD
kubectl get pods -o wide
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Evidencia que debe guardarse

- pods `Running`.
- Service con `Endpoints: <none>`.
- etiquetas antes de corregir.
- manifiesto corregido.
- Service con endpoints poblados.
- `curl` exitoso.

---

## RETO 3 — El pipeline continúa aunque las pruebas fallen

### Situación inicial

Workflow defectuoso:

```yaml
name: ci-cd

on:
  push:
    branches:
      - main

jobs:
  build-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  deploy:
    runs-on: ubuntu-latest

    steps:
      - run: docker build -t app:${{ github.sha }} .
      - run: docker push registry/app:${{ github.sha }}
      - run: kubectl set image deployment/web-deployment web=registry/app:${{ github.sha }}
```

### Defecto

Los jobs son independientes. GitHub Actions puede ejecutarlos en paralelo.

```text
build-test ──────────────►
deploy     ──────────────►
```

Aunque `npm test` falle, `deploy` puede continuar.

### Objetivo

Agregar una dependencia explícita para que el segundo job solo se ejecute cuando el primero haya terminado exitosamente.

### Paso 1. Confirmar el defecto

Romper una prueba temporalmente.

Ejemplo conceptual:

```javascript
assert.equal(response.statusCode, 500);
```

cuando la aplicación realmente devuelve `200`.

Ejecutar:

```powershell
npm test
```

Debe fallar.

Subir:

```powershell
git add server.test.js
git commit -m "Break test to reproduce pipeline defect"
git push
```

En Actions, comprobar que:

- `build-test` falla;
- `deploy` o `build-push` se ejecuta igualmente.

### Paso 2. Corregir el workflow

Agregar:

```yaml
needs: build-test
```

Ejemplo:

```yaml
jobs:
  build-test:
    name: Instalar dependencias y ejecutar pruebas
    runs-on: ubuntu-latest

    steps:
      - name: Descargar repositorio
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: npm

      - name: Instalar dependencias
        run: npm ci

      - name: Ejecutar pruebas
        run: npm test

  build-push:
    name: Construir, escanear y publicar imagen

    needs: build-test

    if: github.event_name == 'push'

    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Descargar repositorio
        uses: actions/checkout@v4

      # construcción, Trivy y publicación
```

La relación se vuelve:

```text
build-test
    ↓ solo si pasa
build-push
```

### Paso 3. Subir la corrección sin arreglar todavía la prueba

```powershell
git add .github/workflows/ci-cd.yml
git commit -m "Make image publication depend on tests"
git push
```

### Paso 4. Comprobar el fail-fast

Resultado esperado:

```text
build-test: failed
build-push: skipped
```

La imagen no debe publicarse.

### Paso 5. Restaurar la prueba

```powershell
git add server.test.js
git commit -m "Restore passing tests"
git push
```

### Paso 6. Confirmar el pipeline completo

Resultado esperado:

```text
build-test ✅
build-push ✅
Trivy ✅
GHCR ✅
```

### Paso 7. Comprobar la imagen

```powershell
docker pull ghcr.io/carlosmoyanog/practicaexamen-inventario-app:latest
```

O con el hash del commit:

```powershell
docker pull ghcr.io/carlosmoyanog/practicaexamen-inventario-app:HASH
```

### Evidencia que debe guardarse

- prueba rota.
- job de pruebas fallido.
- segundo job ejecutándose antes de la corrección.
- commit con `needs: build-test`.
- segundo job omitido después de la corrección.
- pipeline completamente verde.

---

## GIRO FINAL — El tráfico se triplicará y no debe haber corte

### Situación

Marketing anuncia que el tráfico se triplicará y que el siguiente despliegue no debe causar una interrupción perceptible.

### Objetivos

1. Aumentar capacidad.
2. Aplicar una estrategia de despliegue sin interrupción.
3. Generar tráfico durante la actualización.
4. Confirmar que la aplicación continúa respondiendo.

---

### Solución A — Aumentar réplicas manualmente

Consultar:

```powershell
kubectl get deployment inventario-app
```

Escalar:

```powershell
kubectl scale deployment/inventario-app --replicas=4
```

Comprobar:

```powershell
kubectl get pods -w
kubectl get deployment inventario-app
```

Para Canary Stable:

```powershell
kubectl scale deployment/inventario-app-stable-practica-examen --replicas=6
```

### ¿Qué resuelve?

Aumenta el número de procesos disponibles para atender tráfico.

No garantiza por sí solo:

- persistencia;
- balanceo externo avanzado;
- capacidad infinita;
- que no exista un cuello de botella en la base de datos.

---

### Solución B — HPA opcional

Habilitar métricas:

```powershell
minikube addons enable metrics-server
```

Esperar y comprobar:

```powershell
kubectl top nodes
kubectl top pods
```

El Deployment debe tener `resources.requests.cpu`.

Crear HPA:

```powershell
kubectl autoscale deployment inventario-app `
  --cpu-percent=60 `
  --min=2 `
  --max=10
```

Consultar:

```powershell
kubectl get hpa
kubectl describe hpa inventario-app
```

Eliminar:

```powershell
kubectl delete hpa inventario-app
```

---

### Estrategia 1 — Rolling Update

Configuración:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

Actualizar imagen:

```powershell
kubectl set image `
  deployment/inventario-app `
  inventario-app=ghcr.io/carlosmoyanog/practicaexamen-inventario-app:HASH_NUEVO
```

Observar:

```powershell
kubectl rollout status deployment/inventario-app
kubectl get pods -w
```

Rollback:

```powershell
kubectl rollout undo deployment/inventario-app
```

### Explicación

Con dos réplicas, `maxUnavailable: 1` permite que como máximo una esté no disponible. `maxSurge: 1` permite crear una réplica adicional temporal.

---

### Estrategia 2 — Canary usada en esta práctica

Recursos:

```text
deployment-stable.yaml → 4 pods v1
deployment-canary.yaml → 1 pod v2-canary
service.yaml           → selecciona ambos
```

Aplicar:

```powershell
kubectl apply -f k8s/canary/deployment-stable.yaml
kubectl apply -f k8s/canary/deployment-canary.yaml
kubectl apply -f k8s/canary/service.yaml
```

Esperar:

```powershell
kubectl rollout status deployment/inventario-app-stable-practica-examen
kubectl rollout status deployment/inventario-app-canary-practica-examen
```

Verificar:

```powershell
kubectl get pods -l environment=practica-examen --show-labels
kubectl get endpoints inventario-app-canary-service
```

Deben aparecer cinco pods/endpoints.

Obtener URL:

```powershell
minikube service inventario-app-canary-service --url
```

### Probar reparto 80/20

```powershell
$results = 1..100 | ForEach-Object {
    $json = curl.exe -s `
        -H "Connection: close" `
        http://127.0.0.1:PUERTO/version

    $json | ConvertFrom-Json
}

$results |
    Group-Object version |
    Select-Object Name, Count
```

Resultado aproximado:

```text
v1          80
v2-canary   20
```

No tiene que ser exacto.

### ¿Por qué se usa `Connection: close`?

El Service distribuye conexiones. Una conexión HTTP persistente puede quedarse ligada al mismo pod. Al cerrar cada conexión se observa mejor el reparto.

### Promover Canary

Actualizar Stable con la nueva imagen:

```powershell
kubectl set image `
  deployment/inventario-app-stable-practica-examen `
  inventario-app=ghcr.io/carlosmoyanog/practicaexamen-inventario-app:HASH_NUEVO
```

Esperar:

```powershell
kubectl rollout status deployment/inventario-app-stable-practica-examen
```

Luego retirar Canary:

```powershell
kubectl scale deployment/inventario-app-canary-practica-examen --replicas=0
```

### Rollback Canary

```powershell
kubectl scale deployment/inventario-app-canary-practica-examen --replicas=0
```

La versión estable sigue recibiendo tráfico.

---

### Generar tráfico durante el despliegue

Bucle:

```powershell
while ($true) {
    $timestamp = Get-Date -Format "HH:mm:ss"
    try {
        $response = Invoke-RestMethod http://127.0.0.1:PUERTO/health
        Write-Host "$timestamp OK $($response.status)"
    }
    catch {
        Write-Host "$timestamp ERROR"
    }

    Start-Sleep -Milliseconds 300
}
```

En otra terminal ejecutar el rollout:

```powershell
kubectl set image `
  deployment/inventario-app `
  inventario-app=ghcr.io/carlosmoyanog/practicaexamen-inventario-app:HASH_NUEVO
```

Comprobar que no aparecen errores continuos en el generador de tráfico.

### Evidencia que debe guardarse

- réplicas antes y después;
- HPA, cuando se use;
- pods durante el rollout;
- tráfico continuo;
- `rollout status` exitoso;
- distribución Canary;
- rollback o promoción.

---

## ORDEN RECOMENDADO DURANTE LOS 90 MINUTOS

### Primeros 15 minutos — Reto 1

```text
docker build
docker run
docker ps
docker logs
docker exec
corregir Dockerfile
curl exitoso
```

### Minutos 15–30 — Reto 2

```text
kubectl apply
kubectl get pods
kubectl describe service
kubectl get endpoints
corregir labels/selector
curl exitoso
```

### Minutos 30–45 — Reto 3

```text
romper prueba
push
observar defecto
agregar needs
push
comprobar skipped
restaurar prueba
```

### Últimos 45 minutos — Giro

```text
aumentar réplicas
aplicar Rolling o Canary
generar tráfico
realizar despliegue
verificar ausencia de corte
guardar evidencia
```

---

## CHECKLIST FINAL POR RETO

### Reto 1

- [ ] El contenedor está activo.
- [ ] Se revisaron logs.
- [ ] Se confirmó el puerto real.
- [ ] Se entró al contenedor.
- [ ] Se corrigió el Dockerfile.
- [ ] La aplicación responde desde el host.

### Reto 2

- [ ] Los pods están Running/Ready.
- [ ] El Service inicialmente no tiene endpoints.
- [ ] Se compararon selector y labels.
- [ ] Se corrigió la coincidencia.
- [ ] El Service muestra IPs.
- [ ] La aplicación responde mediante el Service.

### Reto 3

- [ ] Se rompió una prueba.
- [ ] Se confirmó que el segundo job se ejecutaba.
- [ ] Se agregó `needs: build-test`.
- [ ] El segundo job quedó omitido.
- [ ] Se restauró la prueba.
- [ ] El pipeline terminó completamente en verde.

### Giro

- [ ] Se aumentó capacidad.
- [ ] Se usó Rolling Update o Canary.
- [ ] Se generó tráfico.
- [ ] El despliegue terminó exitosamente.
- [ ] No se observó interrupción perceptible.
- [ ] Se guardó evidencia.
