# Actividad UNIR — AuraTech en Kubernetes (kit de referencia)

> **Kit de referencia, no la entrega final.** Tú haces cada paso en tu PC
> (primero minikube, luego EC2 con k3s). Este kit NO fue probado con
> Docker ni con minikube: es el mapa para que lo hagas tú, paso por paso.
> La app **no** fue desarrollada por nosotros: es el repo real
> https://github.com/waeijn/auratech (tienda de ventas, ya hecha).

## Qué es AuraTech

Una tienda con **2 proyectos separados, juntos en el mismo repo**, cada uno
con su propio Dockerfile, más MySQL. Por eso el Dockerfile es fácil:
cada uno se construye solo, en su carpeta.

| Parte | Qué es | Puerto |
|---|---|---|
| `auratech-frontend/` | Tienda React: home, productos, carrito, checkout, login, y `/admin` (Dashboard, pedidos, productos, usuarios) | 30080 |
| `auratech-backend/src/` | API Laravel 12 + Sanctum (esa carpeta `src/` ES la raíz de la app Laravel) | interna, 80 |
| MySQL 8.0 | Base de datos, StatefulSet, 1 réplica | interna, 3306 |

**Hay UN SOLO Deployment de frontend** (`auratech-frontend:1.0`), con 2 réplicas, en el NodePort 30080. La tienda y el admin son la misma app: la vista cambia por el rol con el que te logueas.
Lo que cambia la vista es el **rol con el que te logueas**: las rutas
`/admin`, `/admin/orders`, `/admin/products` y `/admin/users` son
`adminOnly` en el código. Un usuario normal no entra ahí aunque escriba la URL.

**Logins de prueba** (los crea el seed):

- **Cliente:** `user@auratech.com` / `password123` — ve la tienda, carrito y pedidos
- **Admin:** `admin@auratech.com` / `admin123` — además ve `/admin` (dashboard, pedidos, productos, usuarios)

## Paso 0 — minikube (si aún no arranca)

Si en PowerShell `minikube version` te dice que no reconoce el comando,
cierra y reabre PowerShell, y revisa que `C:\Program Files\Kubernetes\Minikube`
esté en el PATH. Luego:

```powershell
minikube start --driver=docker
kubectl get nodes
```

## Paso 1 — clonar el repo

```powershell
git clone https://github.com/waeijn/auratech.git
cd auratech
```

## Paso 2 — generar la APP_KEY (¡obligatoria!)

Sin una APP_KEY válida Laravel da error 500. Genera una y guárdala,
la vas a pegar en el Secret en el Paso 5.

Si tienes PHP en tu PC, desde la carpeta del backend:

```powershell
cd auratech-backend\src
php artisan key:generate --show
cd ..\..
```

Si no tienes PHP, en PowerShell genera una clave del mismo formato
(`base64:` + 44 caracteres):

```powershell
"base64:" + [Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }))
```

## Paso 3 — copiar nuestros Dockerfiles, el nginx y el parche

Descomprime este ZIP **en la misma carpeta donde clonaste el repo**,
para que queden lado a lado: `auratech` y `unir-actividad-k8s-auratech`.

```powershell
# Dockerfile del backend -> adentro de la app Laravel (auratech-backend/src)
Copy-Item unir-actividad-k8s-auratech\dockerfiles\Dockerfile.backend auratech-backend\src\Dockerfile

# Dockerfile del front + nginx -> adentro del front
Copy-Item unir-actividad-k8s-auratech\dockerfiles\Dockerfile.frontend auratech-frontend\Dockerfile
Copy-Item unir-actividad-k8s-auratech\dockerfiles\nginx.conf auratech-frontend\nginx.conf

# PARCHE OBLIGATORIO: el front original apunta a http://localhost:18082/api.
# Si no lo cambias, la tienda habla con un localhost que no existe en Kubernetes.
Copy-Item unir-actividad-k8s-auratech\patches\api.js auratech-frontend\src\config\api.js -Force
```

El parche cambia solo la URL base a `/api` (relativa): nginx la reenvía
a `api-svc` dentro del cluster. Los `API_ENDPOINTS` quedan igual.

## Paso 4 — construir y subir las 2 imágenes

Son solo 2 imágenes: la API y el front. La tienda y el admin comparten la del front.

```powershell
# API: el contexto es auratech-backend\src (ahi esta composer.json y artisan)
docker build -t luisraga12/auratech-api:1.0 auratech-backend\src
docker push luisraga12/auratech-api:1.0

# Front: el contexto es auratech-frontend (ahi estan package.json y nginx.conf)
docker build -t luisraga12/auratech-frontend:1.0 auratech-frontend
docker push luisraga12/auratech-frontend:1.0
```

> El front es multi-stage: etapa 1 construye React con Node (`npm run build`
> genera `build/`, no `dist/`), etapa 2 sirve ese `build/` con nginx.
> La imagen final es solo nginx + los archivos ya construidos, sin Node.

## Paso 5 — poner tu APP_KEY en el Secret

Abre `unir-actividad-k8s-auratech\k8s\01-secret.yaml` y cambia el valor
de `APP_KEY`. Ojo: en ese archivo los valores van en **base64**.
Genera el base64 de tu clave en PowerShell así:

```powershell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("base64:TU-CLAVE-AQUI"))
```

Y pega el resultado en la línea `APP_KEY:` del Secret. Los demás valores
son de ejemplo, cámbialos si quieres (si cambias la contraseña de MySQL,
cámbiala en el Secret, no en otro lado).

## Paso 6 — aplicar los YAML en orden

```powershell
kubectl apply -f unir-actividad-k8s-auratech\k8s\00-namespace.yaml
kubectl apply -f unir-actividad-k8s-auratech\k8s\01-secret.yaml
kubectl apply -f unir-actividad-k8s-auratech\k8s\02-statefulset-mysql.yaml
kubectl apply -f unir-actividad-k8s-auratech\k8s\03-deployment-api.yaml
kubectl apply -f unir-actividad-k8s-auratech\k8s\04-deployment-tienda.yaml
kubectl apply -f unir-actividad-k8s-auratech\k8s\05-services.yaml
```

Espera a que MySQL esté Ready antes del siguiente paso:

```powershell
kubectl get pods -n pruebas -w
```

## Paso 7 — migraciones + seed (UNA SOLA VEZ)

```powershell
kubectl apply -f unir-actividad-k8s-auratech\k8s\06-job-migrate-seed.yaml
kubectl logs job/auratech-migrate-seed -n pruebas
```

Esto crea las tablas (users, categories, products, carts, orders, reviews…)
y siembra el admin, el cliente y los productos. **No lo corras 2 veces:**
`migrate:fresh` borra todo y vuelve a sembrar.

## Paso 8 — verificar

```powershell
kubectl get pods -n pruebas
kubectl get svc -n pruebas
minikube ip
```

Abre en el navegador (cambia `<IP>` por la IP de minikube):

- Tienda: `http://<IP>:30080` — entra como `user@auratech.com` / `password123`
- Admin: `http://<IP>:30080/admin` — entra como `admin@auratech.com` / `admin123`
- API: `http://<IP>:30080/api/products` — te debe devolver el catálogo en JSON
  (la API en sí no tiene NodePort propio, se llega por el proxy `/api` del front)

## En EC2 con k3s

Lo mismo, pero con la IP pública de la instancia. En el Security Group
abre el puerto **30080**. No hay que reconstruir las
imágenes si cambia la IP, porque el front habla con la API por `/api`
adentro del cluster, no por IP.

## Qué NO funciona / qué tener en cuenta

- **Sin APP_KEY no arranca la API** (error 500). Es el error más común.
- **Sin el parche de `api.js` el front no habla con tu API.**
- El `.env.example` del repo trae `DB_CONNECTION=sqlite` y sesión/cache
  en base de datos: en Kubernetes los forzamos a MySQL y a `file`/`sync`
  con las variables del Secret, para no necesitar más servicios.
- Sanctum usa tokens Bearer, no cookies, así que no hay que configurar dominios.
