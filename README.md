# Reto 3 — Despliegue en Kubernetes con Deployment y Service

Despliegue del microservicio `demo-micro` —la misma imagen publicada en los retos 1 y 2— sobre un clúster de Kubernetes, expuesto mediante un Service de tipo NodePort.

---

## 0. Sobre la distribución de Kubernetes

El enunciado propone **k3s**, que se instala con `curl -sfL https://get.k3s.io | sh -`. Ese instalador es exclusivo de Linux: k3s es un binario que se registra como servicio systemd y no existe para macOS.

Este reto se resolvió sobre el **Kubernetes integrado en Docker Desktop** (v1.32.2), que expone la misma API estándar. Los tres manifiestos —`Namespace`, `Deployment` y `Service`— pertenecen a las APIs `v1` y `apps/v1`, comunes a todas las distribuciones, por lo que se aplican sin modificación en k3s, k3d, minikube, EKS, AKS o GKE.

> Alternativa fiel al enunciado en macOS: **k3d**, que ejecuta k3s real dentro de contenedores Docker.
> ```bash
> brew install k3d
> k3d cluster create demo -p "30080:30080@server:0"
> ```

---

## 1. Verificar el clúster

```bash
kubectl config current-context     # -> docker-desktop
kubectl get nodes                  # -> Ready
kubectl version
```

---

## 2. Desplegar

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

O de una vez, aplicando el directorio completo:

```bash
kubectl apply -f k8s/
```

---

## 3. Verificar el despliegue

```bash
kubectl get all -n demo
kubectl get deploy,po,svc -n demo -o wide
kubectl rollout status deployment/demo-micro -n demo
```

Debe mostrar **2 pods** en estado `Running` y `READY 1/1`, y el Service con el puerto `80:30080/TCP`.

Detalle de un pod y sus eventos:

```bash
kubectl describe pod -n demo -l app=demo-micro | head -60
kubectl logs -n demo -l app=demo-micro --tail=30
```

---

## 4. Acceder al servicio

**Vía NodePort** (Docker Desktop expone el nodo en localhost):

```bash
curl http://localhost:30080/ ; echo
curl http://localhost:30080/version ; echo
curl http://localhost:30080/actuator/health ; echo
```

**Vía port-forward** (alternativa, útil cuando el NodePort no es accesible):

```bash
kubectl -n demo port-forward svc/demo-micro 8080:80
# en otra terminal:
curl http://localhost:8080/
```

---

## 5. Comprobaciones adicionales

**Balanceo entre réplicas** — el Service reparte las peticiones entre los dos pods:

```bash
kubectl get po -n demo -o wide
for i in $(seq 1 6); do curl -s http://localhost:30080/ | head -c 60; echo; done
```

**Auto-recuperación** — al eliminar un pod, el Deployment crea otro para mantener las réplicas:

```bash
kubectl get po -n demo
kubectl delete pod -n demo -l app=demo-micro --field-selector status.phase=Running --wait=false | head -1
kubectl get po -n demo -w      # Ctrl+C para salir
```

**Escalado**:

```bash
kubectl scale deployment/demo-micro -n demo --replicas=4
kubectl get po -n demo
kubectl scale deployment/demo-micro -n demo --replicas=2
```

**Usuario no privilegiado** dentro del pod:

```bash
kubectl exec -n demo deploy/demo-micro -- whoami     # -> appuser
```

---

## 6. Limpieza

```bash
kubectl delete -f k8s/
# o, eliminando el namespace completo:
kubectl delete namespace demo
```

---

## 7. Diferencias respecto a los manifiestos del enunciado

| Cambio | Motivo |
|---|---|
| Imagen con etiqueta `sha-955b92c` en lugar de `:latest` | Con una etiqueta móvil e `imagePullPolicy: IfNotPresent`, el nodo reutiliza indefinidamente la primera imagen que descargó: el despliegue deja de ser reproducible y no se sabe qué código está corriendo. El SHA identifica el commit exacto. |
| `strategy: RollingUpdate` con `maxUnavailable: 0` | Garantiza que siempre haya una réplica atendiendo durante una actualización. Sin declararlo, el valor por defecto permite dejar pods fuera de servicio. |
| `startupProbe` en lugar de `initialDelaySeconds` largos | Cubre el arranque de Spring Boot sin fijar una espera arbitraria: si arranca antes, las otras probes se activan antes; si tarda más, no se reinicia el pod por error. |
| Probes separadas a `/actuator/health/liveness` y `/readiness` | Spring Boot expone grupos distintos: readiness indica si puede recibir tráfico, liveness si sigue vivo. Apuntar ambas al mismo endpoint hace que un fallo de dependencias reinicie el pod en vez de sólo sacarlo del balanceo. |
| `securityContext` con `runAsNonRoot` y `readOnlyRootFilesystem` | La imagen ya define `appuser`, pero declararlo en el manifiesto hace que el clúster lo verifique y rechace el pod si la imagen intentara ejecutarse como root. |
| `targetPort` referenciado por nombre (`http`) | Si cambia el puerto del contenedor basta actualizarlo en un sitio; el Service sigue apuntando correctamente. |
| Volumen `emptyDir` montado en `/tmp` | Consecuencia de `readOnlyRootFilesystem`: la JVM necesita un directorio temporal con permiso de escritura. |
| Etiquetas `app.kubernetes.io/*` | Convención estándar de Kubernetes; facilita el filtrado y la integración con herramientas de observabilidad. |

---

## 8. Errores frecuentes

| Síntoma | Causa / solución |
|---|---|
| `ImagePullBackOff` | La imagen no existe o el repositorio es privado. Verificar con `docker pull` y, si es privado, crear el `imagePullSecret`. |
| `CrashLoopBackOff` | El contenedor arranca y muere. Revisar con `kubectl logs -n demo <pod> --previous`. |
| Pods en `Running` pero `READY 0/1` | Las probes fallan. Comprobar que el endpoint de salud responde: `kubectl exec -n demo <pod> -- wget -qO- localhost:8080/actuator/health`. |
| `curl localhost:30080` no responde | El NodePort no está expuesto en el host. Usar `kubectl port-forward` como alternativa. |
| `OOMKilled` | El límite de memoria es insuficiente para la JVM. Revisar que `JAVA_TOOL_OPTIONS` esté aplicado y subir el límite si hace falta. |
| `Pending` sin asignar nodo | Recursos insuficientes en el clúster. Verificar con `kubectl describe pod` la sección Events. |
