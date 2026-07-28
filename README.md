# Simulacro - Práctica de Sistemas Distribuidos

App Express con 3 retos clásicos de Docker, Kubernetes y CI/CD.

---

## Reto 1 — El contenedor que "corre" pero no responde

**Síntoma:** `docker build` OK, `docker run` OK, contenedor activo, pero `curl http://localhost:3000/` no responde.

**Causa:** El servidor Express escucha solo en `127.0.0.1` (loopback interno). Docker mapea el puerto a la IP del contenedor (`172.17.0.x`), pero nadie escucha ahí.

### Diagnosticar

```bash
# Construir y ejecutar
docker build -t simulacro-reto1 .
docker run -d --name reto1 -p 3000:3000 simulacro-reto1

# Verificar que no responde
curl http://localhost:3000/               # ❌ connection refused

# Revisar logs
docker logs reto1                         # "Servidor escuchando en puerto 3000"

# Dentro del contenedor — localhost funciona
docker exec reto1 node -e "require('http').get('http://localhost:3000/',(r)=>{r.on('end',()=>console.log('OK'))}).on('error',(e)=>console.log('FALLA:',e.message))"

# Pero la IP del contenedor NO
docker exec reto1 node -e "const h=require('http'),os=require('os');const ni=os.networkInterfaces();Object.keys(ni).forEach(k=>{ni[k].forEach(i=>{if(!i.internal&&i.family==='IPv4'){h.get('http://'+i.address+':3000/',(r)=>{let d='';r.on('data',c=>d+=c);r.on('end',()=>console.log('IP '+i.address+':3000 -> OK'))}).on('error',(e)=>console.log('IP '+i.address+':3000 ->',e.message))}})})"
# IP 172.17.0.2:3000 -> connect ECONNREFUSED   ← el bug
```

### Solución

En `server.js` — agregar `'0.0.0.0'` al `listen`:

```js
app.listen(PORT, '0.0.0.0', () => {
  console.log('...');
});
```

`0.0.0.0` = todas las interfaces de red. Sin esto, Node.js escucha solo en `127.0.0.1`.

### Verificar

```bash
docker rm -f reto1
docker build -t simulacro-reto1 .
docker run -d --name reto1 -p 3000:3000 simulacro-reto1
curl http://localhost:3000/               # ✅ 200 OK
```

### Otros casos posibles (mismo síntoma)

| Causa | Dónde | Síntoma adicional |
|-------|-------|-------------------|
| Puerto mismatch | `Dockerfile` | `EXPOSE 3000` pero app escucha en `8080` |
| WORKDIR incorrecto | `Dockerfile` | `CMD ["node","server.js"]` pero server.js no está en el WORKDIR |
| Puerto bloqueado | Host (firewall) | Funciona dentro del contenedor, desde afuera no |
| App crashea silenciosamente | `server.js` | Contenedor activo pero proceso muerto (revisar con `docker top`) |

---

## Reto 2 — Pods en Running, servicio sin respuesta

**Síntoma:** `kubectl get pods` muestra todos Running, pero `kubectl get endpoints` está vacío y el Service no entrega tráfico.

**Causa:** Las labels del pod NO coinciden con el selector del Service.

### Diagnosticar

```bash
kubectl apply -f k8s/

kubectl get pods --show-labels
# app=cicd-practica-sd     ← los pods tienen esta label

kubectl describe service cicd-practica-sd
# Selector: app=nonexistent                 ← el Service busca otra label
# Endpoints: <none>                         ← sin endpoints

kubectl get endpoints cicd-practica-sd
# <none>                                     ← vacío
```

### Solución

En `k8s/service.yaml` — alinear selector con las labels del pod:

```yaml
selector:
  app: cicd-practica-sd
```

Luego:
```bash
kubectl apply -f k8s/service.yaml
```

### Si los pods quedan 0/1 Ready

Las readiness/liveness probes fallan. Revisar:

```bash
kubectl describe pod <nombre-pod> | grep -A5 Readiness
kubectl logs <nombre-pod>
```

Causa común: la app responde `500` en `/health`. En `k8s/deployment.yaml`, fijar:

```yaml
env:
  - name: SIMULATE_FAILURE
    value: "false"
```

### Verificar

```bash
kubectl get pods
# Todos 1/1 Running

kubectl get endpoints cicd-practica-sd
# 10.244.0.x:3000,...    ← IPs de los pods listadas
```

### Otros casos posibles

| Causa | Síntoma |
|-------|---------|
| Mismatch de puertos | Service `targetPort: 8080` pero pod expone `3000` |
| Pod no Ready (probe fallando) | Pod Running pero 0/1 Ready |
| Selector con label incorrecta | Service selector `app: frontend` pero pod tiene `app: backend` |
| Namespace diferente | Pod en `ns-app`, Service en `default` |

---

## Reto 3 — Pipeline que despliega aunque las pruebas fallen

**Síntoma:** El workflow de GitHub Actions siempre termina en verde. Incluso con tests fallando, el job `deploy` se ejecuta.

**Causa:** El job `deploy` no tiene la dependencia `needs: build-test`, por lo que corre en paralelo sin esperar el resultado de las pruebas.

### El bug en el workflow (`.github/workflows/ci-cd.yml`)

```yaml
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:              # ← sin "needs: build-test"
    runs-on: ubuntu-latest
    steps:
      - run: docker build ...
```

### Diagnosticar

1. Romper una prueba en `server.test.js`:

```js
assert.strictEqual(res.status, 404);  // espera 404 pero recibe 200
```

2. Hacer push a `main`:

```bash
git add .
git commit -m "romper test a proposito"
git push origin main
```

3. Ir a GitHub Actions → el `build-test` falla (❌) pero `deploy` igual corre (✅).

### Solución

Agregar `needs: build-test` al job `deploy`:

```yaml
deploy:
  needs: build-test   # ← esto lo detiene si build-test falla
  runs-on: ubuntu-latest
```

GitHub Actions ejecuta jobs en paralelo por defecto. `needs` crea una dependencia explícita: si el job anterior falla, este no se ejecuta (skipped).

### Verificar

1. Romper otra prueba (ej: cambiar `200` por `500` en otro `assert`)
2. Push a `main`
3. GitHub Actions: `build-test` falla ❌, `deploy` aparece como **skipped** (no se ejecutó)

### Otros casos posibles

| Causa | Síntoma |
|-------|---------|
| `if: always()` en deploy | Se ejecuta incluso si falla, pisando `needs` |
| Jobs sin dependencias | Todos corren en paralelo, fallos ignorados |
| `continue-on-error: true` | El paso falla pero el job sigue como exitoso |
| Tests que pasan pero no cubren | Pipeline verde pero funcionalidad rota |

---

## Comandos rápidos

### Docker
```bash
docker build -t simulacro-reto1 .
docker run -d --name reto1 -p 3000:3000 simulacro-reto1
docker logs reto1
docker exec -it reto1 sh
docker rm -f reto1
```

### Kubernetes
```bash
kubectl apply -f k8s/
kubectl get pods -w
kubectl get pods --show-labels
kubectl get endpoints cicd-practica-sd
kubectl describe service cicd-practica-sd
kubectl delete deployment cicd-practica-sd
kubectl delete service cicd-practica-sd
```

### Tests y CI/CD
```bash
npm install         # instalar dependencias localmente
npm test            # ejecutar tests
git push origin main    # disparar workflow
```

---

## Giro final — Escalabilidad y despliegue sin corte

**Contexto:** Marketing lanzó una campaña sin avisar. El tráfico se triplica. El próximo despliegue no debe causar corte perceptible.

### 1. Ajustar réplicas para el pico de tráfico

Opción manual — aumentar réplicas en `k8s/deployment.yaml`:

```yaml
spec:
  replicas: 12
```

```bash
kubectl apply -f k8s/deployment.yaml
kubectl get pods -w
# Aparecen 12 pods gradualmente
```

Opción automática — **Horizontal Pod Autoscaler (HPA)**:

```bash
# Crear HPA basado en CPU (ajustar según el recurso que se sature)
kubectl autoscale deployment cicd-practica-sd --min=3 --max=15 --cpu-percent=60

# Ver estado
kubectl get hpa -w
# NAME                REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS
# cicd-practica-sd    Deployment/cicd-practica-sd    35%/60%   3         15        3
```

HPA escala automáticamente entre 3 y 15 réplicas cuando el CPU supera el 60%.

### 2. Estrategia de despliegue sin corte

El `deployment.yaml` ya usa `RollingUpdate`:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1     # máximo 1 pod fuera de servicio a la vez
    maxSurge: 1           # máximo 1 pod extra durante la actualización
```

`RollingUpdate` reemplaza pods de a uno: el viejo se drena, el nuevo se levanta, solo cuando el nuevo está Ready (pasa readiness probe) sigue con el siguiente. Así **nunca se interrumpe el servicio** porque siempre hay réplicas disponibles.

Alternativas que podrían usarse:

| Estrategia | Cómo | Ventaja |
|------------|------|---------|
| **RollingUpdate** (usada) | Reemplaza pods gradualmente | Sin corte, controlable |
| **Blue-Green** | Dos entornos completos, se cambia el Service de un golpe | Rollback instantáneo |
| **Canary** | Versión nueva recibe X% del tráfico, se aumenta gradualmente | Validación con tráfico real |

### 3. Simular tráfico y verificar cero cortes

```bash
# Terminal 1 — generar tráfico continuo
while ($true) { curl -s http://localhost:3000/ > $null; Start-Sleep -Milliseconds 200 }
# o con un CLI: kubectl run -it --rm load-generator --image=busybox -- sh
# while true; do wget -qO- http://cicd-practica-sd; done

# Terminal 2 — aplicar cambio (ej: cambiar APP_VERSION)
kubectl set image deployment/cicd-practica-sd app=ghcr.io/whosstixi19/practica-sd:v2
kubectl rollout status deployment/cicd-practica-sd
# rolling update complete!

# Terminal 1 — nunca hubo error, todas las respuestas 200
```

### 4. Verificar el Service con endpoints poblados

```bash
kubectl get endpoints cicd-practica-sd
# 10.244.0.x:3000,10.244.0.y:3000,...   ← todas las réplicas aparecen
```

---

## Justificación para cada reto (para sustentar con el profesor)

### Reto 1 — Contenedor que corre pero no responde

**Justificación:** El defecto demuestra la diferencia entre un contenedor "vivo" (el proceso corre) y uno "accesible" (el proceso escucha en la interfaz correcta). En Node.js, `app.listen(PORT)` sin host se vincula a `0.0.0.0` (todas las interfaces). El error se provoca al pasar explícitamente `'127.0.0.1'`, que ata el socket solo a loopback. El diagnóstico con `docker exec` revela que el servidor responde en `localhost` pero no en la IP del contenedor (`172.17.0.x`). Docker port forwarding (`-p`) mapea al tráfico a la IP del contenedor, no a `127.0.0.1`, por eso desde el host no hay respuesta. La corrección (`0.0.0.0`) restaura la escucha en todas las interfaces. Conceptos clave: binding de sockets, redes Docker (bridge/overlay), diferencia entre `127.0.0.1` y `0.0.0.0`.

### Reto 2 — Pods Running, servicio sin respuesta

**Justificación:** El defecto muestra cómo Kubernetes empareja Pods y Services mediante **labels y selectores**, un mecanismo declarativo fundamental. El Service usa un selector (`app: nonexistent`) que no coincide con las labels del pod (`app: cicd-practica-sd`). El controlador de endpoints no encuentra pods que cumplan el selector, por lo que `kubectl get endpoints` queda vacío. El tráfico entrante al Service no tiene backend al cual reenviarse. Adicionalmente, si las probes (readiness/liveness) fallan — como cuando `SIMULATE_FAILURE=true` hace que `/health` devuelva 500 — el pod nunca se marca como Ready y el Service lo excluye aunque el selector coincida. Conceptos clave: labels y selectores, endpoints, readiness probes, Service-Pod vinculación.

### Reto 3 — Pipeline que despliega aunque las pruebas fallen

**Justificación:** El defecto demuestra la necesidad de **dependencias explícitas entre jobs** en GitHub Actions. Por defecto, los jobs de un workflow se ejecutan en paralelo. Si `deploy` no declara `needs: build-test`, arranca al mismo tiempo que las pruebas, sin importar su resultado. Incluso si `npm test` falla (assertion error, test rojo), el job de deploy continúa y publica la imagen. Esto puede llevar a desplegar código defectuoso a producción. La corrección (`needs: build-test`) establece una dependencia secuencial: GitHub Actions espera a que `build-test` termine exitosamente antes de iniciar `deploy`; si falla, el deploy se marca como "skipped" y no se ejecuta. Conceptos clave: dependencias entre jobs, pipeline stages, fail-fast, integridad del pipeline, GitOps.

---

**Autor:** Jose Tixi — Sistemas Distribuidos
