## Paso a Paso para desplegar Nextcloud en Kubernetes con Secrets y ConfigMap

**Objetivo:** Desplegar Nextcloud en Kubernetes utilizando Secrets para almacenar las credenciales de la base de datos de forma segura y configuración más flexible con. Se incluye la configuración de una base de datos MariaDB para la persistencia de datos.

**Requisitos previos:**

- Tener un clúster de Kubernetes en funcionamiento.
- Conocimiento básico de Kubernetes (pods, deployments, services, secrets, configmaps, ingress).
- Credenciales de base de datos (usuario y contraseña).

**Pasos:**

**1. Crear Secrets para las credenciales de la base de datos:**

Comando para pasar a Base64:
```shell
echo -n "usuario" | base64 -w 0
```
Archivo con usuario y clave codificado:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nextcloud-db-credentials
type: Opaque
data:
  username: dXN1YXJpbw==  # usuario en Base64
  password: Y2xhdmU=  # clave en Base64
```

**2. Crear ConfigMap para la configuración de la DB:**
Esto es lo mínimo, podrían agregarse en este archivo más detalles
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nextcloud-config
data:
  DB_NAME: "nextcloud"  # Replace if your database name is different

```
**3. Crear Deployment para la base de datos MariaDB:**

Este es el ejemplo más básico y sin redundancia. Existe la posibilidad de hacer despliegues más complejos de MariaDB.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:11.3.2
        ports:
        - containerPort: 3306
        envFrom:  # Variables de entorno
        - secretRef:  # Secreto para las credenciales
            name: nextcloud-db-credentials
        - configMapRef:  # ConfigMap para DB_NAME
            name: nextcloud-config
        env:  # También es necesaria la clave de root
        - name: MARIADB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: nextcloud-db-credentials
              key: password
```

**4. Crear PV y PVC**

Almacenamiento local persistente para Nextcloud. Es posible utilizar otro tipo de almacenamiento pero por simplicidad se usa este.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nextcloud-storage

spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/nextcloud  # directorio en la máquina

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-pvc

spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi  # No superar la capacidad del PV
  storageClassName: ""  # Opcional de existir
```

**5. Crear Deployment para Nextcloud:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextcloud
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      containers:
      - name: nextcloud
        image: nextcloud:29.0-apache
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nextcloud-data
          mountPath: /var/www/html
        envFrom:
        - secretRef:
            name: nextcloud-db-credentials
        - configMapRef:
            name: nextcloud-config
      volumes:
      - name: nextcloud-data
        persistentVolumeClaim:
          claimName: nextcloud-pvc
```

**6. Crear Service e Ingress para Nextcloud:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nextcloud-service
spec:
  selector:
    app: nextcloud
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

---

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud-ingress
spec:
  rules:
  - host: microk.local
    http:
      paths:
      - pathType: Prefix 
        path: "/"
        backend:
          service:
            name: nextcloud-service
            port:
              number: 80 
```