# Desinstalación de IBM Concert en VM

## Objetivo

Desinstalar IBM Concert, Concert Workflows y Concert Data Apps desde una máquina virtual Red Hat Enterprise Linux 9 utilizando el script oficial incluido en el paquete de instalación.

El procedimiento considera una desinstalación controlada, con validación previa del ambiente, respaldo de la información persistente y validación posterior de limpieza.

## Alcance

Este procedimiento aplica para una instalación de IBM Concert 3.0.x desplegada en una máquina virtual con K3s y almacenamiento local mediante `local-path`.

El procedimiento considera:

- Validación del estado actual de la instalación.
- Identificación de los volúmenes persistentes utilizados por Concert.
- Respaldo de los datos persistentes antes de desinstalar.
- Ejecución del desinstalador oficial.
- Validación posterior de namespaces, pods, PVC, PV y acceso HTTP/HTTPS.

## Prerrequisitos

Antes de iniciar, validar lo siguiente:

- Acceso SSH a la máquina virtual.
- Usuario operativo utilizado durante la instalación, por ejemplo `concertadmin`.
- Permisos `sudo` para detener e iniciar K3s.
- Variable `KUBECONFIG` apuntando al archivo de K3s.
- Directorio de instalación disponible.
- Espacio suficiente para generar el respaldo de los volúmenes persistentes.

Definir las variables base:

```bash
export INSTALL_DIR="$HOME/concert-install/ibm-concert"
export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"
export BACKUP_DIR="$HOME/concert-backup-before-uninstall-v3.0.0-$(date +%Y%m%d-%H%M%S)"

mkdir -p "$BACKUP_DIR"

echo "INSTALL_DIR=$INSTALL_DIR"
echo "KUBECONFIG=$KUBECONFIG"
echo "BACKUP_DIR=$BACKUP_DIR"
```

Salida esperada:

```text
INSTALL_DIR=/home/concertadmin/concert-install/ibm-concert
KUBECONFIG=/etc/rancher/k3s/k3s.yaml
BACKUP_DIR=/home/concertadmin/concert-backup-before-uninstall-v3.0.0-YYYYMMDD-HHMMSS
```

Validar que el script oficial de desinstalación exista:

```bash
ls -lah "$INSTALL_DIR/bin/uninstall-products-on-vm"
"$INSTALL_DIR/bin/uninstall-products-on-vm" --help
```

Salida esperada:

```text
-rwxr-xr-x ... /home/concertadmin/concert-install/ibm-concert/bin/uninstall-products-on-vm
```

El script debe mostrar opciones para desinstalar todos los productos o productos específicos como `concert`, `dataapps`, `workflows` o `hub`.

---

## 1. Validar el estado previo de la instalación

Antes de desinstalar, validar que K3s se encuentre activo y que los componentes de Concert estén desplegados.

```bash
export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"

systemctl is-active k3s
kubectl get nodes -o wide
kubectl get ns
kubectl get pods -A
kubectl get pvc -A
kubectl get pv
```

Salida esperada:

```text
active

NAME   STATUS   ROLES                  VERSION
vm-1   Ready    control-plane,master   v1.33.4+k3s1
```

En una instalación activa se deben observar namespaces relacionados con Concert, por ejemplo:

```text
concert
concert-dataapps
concert-workflows
platform-hub
faas
```

También se deben observar PVC asociados a los productos instalados:

```text
NAMESPACE           NAME                                      STATUS   CAPACITY   STORAGECLASS
concert             roja-appdb-pvc                            Bound    500Gi      local-path
concert             roja-configdb-pvc                         Bound    50Gi       local-path
concert             data-roja-minio-0                         Bound    512Gi      local-path
concert-dataapps    dataapps-pvc                              Bound    10Gi       local-path
concert-workflows   rna-core-mysqldb-volume-...               Bound    4Gi        local-path
platform-hub        ibm-solis-embedded-db-pvc                 Bound    50Gi       local-path
```

Interpretación:

Los PVC en estado `Bound` confirman que existen datos persistentes asociados a la instalación. Antes de eliminar los productos, se debe respaldar la ruta real donde K3s almacena los volúmenes.

---

## 2. Identificar la ruta real de almacenamiento persistente

Revisar primero el directorio `localstorage` del paquete de instalación:

```bash
ls -lah "$INSTALL_DIR/localstorage"
du -sh "$INSTALL_DIR/localstorage"
find "$INSTALL_DIR/localstorage" -maxdepth 2 -mindepth 1 | sort
```

Salida obtenida en la instalación validada:

```text
### Contenido de localstorage
logs

### Tamaño de localstorage
124K    /home/concertadmin/concert-install/ibm-concert/localstorage
```

Interpretación:

En esta instalación, `localstorage` solo contenía logs. No se encontró una ruta `localstorage/volumes`, por lo que el respaldo no debe realizarse sobre esa ruta.

Identificar los volúmenes persistentes reales usados por K3s:

```bash
kubectl get storageclass -o wide
kubectl get pvc -A -o wide
kubectl get pv -o wide
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.local.path}{"\n"}{end}'
```

Salida esperada:

```text
NAME                   PROVISIONER             RECLAIMPOLICY
local-path (default)   rancher.io/local-path   Delete
```

Ejemplo de rutas detectadas:

```text
pvc-193a5712-... -> /var/lib/rancher/k3s/storage/pvc-193a5712-..._concert-dataapps_dataapps-pvc
pvc-9261a069-... -> /var/lib/rancher/k3s/storage/pvc-9261a069-..._concert_roja-appdb-pvc
pvc-e4986491-... -> /var/lib/rancher/k3s/storage/pvc-e4986491-..._concert_data-roja-minio-0
```

Validar tamaño de la ruta persistente real:

```bash
sudo du -sh /var/lib/rancher/k3s/storage
sudo du -sh /var/lib/rancher/k3s/server/db
```

Salida obtenida en la instalación validada:

```text
4.9G    /var/lib/rancher/k3s/storage
40M     /var/lib/rancher/k3s/server/db
```

Interpretación:

La ruta `/var/lib/rancher/k3s/storage` contiene los volúmenes persistentes reales creados por el provisionador `local-path`. La ruta `/var/lib/rancher/k3s/server/db` contiene la base interna de K3s. Ambas rutas se deben respaldar antes de ejecutar la desinstalación.

---

## 3. Respaldar datos persistentes antes de desinstalar

Para obtener un respaldo consistente, detener temporalmente K3s.

```bash
sudo systemctl stop k3s
systemctl is-active k3s || true
```

Salida esperada:

```text
inactive
```

Generar respaldo de los volúmenes persistentes de K3s:

```bash
sudo tar czf "$BACKUP_DIR/k3s-local-path-storage-before-uninstall-clean.tar.gz" \
  -C /var/lib/rancher/k3s storage

echo "RC_STORAGE=$?"
```

Salida esperada:

```text
RC_STORAGE=0
```

Generar respaldo de la base interna de K3s:

```bash
sudo tar czf "$BACKUP_DIR/k3s-server-db-before-uninstall-clean.tar.gz" \
  -C /var/lib/rancher/k3s/server db

echo "RC_DB=$?"
```

Salida esperada:

```text
RC_DB=0
```

Ajustar propietario y generar hash de validación:

```bash
sudo chown concertadmin:concertadmin "$BACKUP_DIR"/*.tar.gz

sha256sum "$BACKUP_DIR/k3s-local-path-storage-before-uninstall-clean.tar.gz" \
  > "$BACKUP_DIR/k3s-local-path-storage-before-uninstall-clean.tar.gz.sha256"

sha256sum "$BACKUP_DIR/k3s-server-db-before-uninstall-clean.tar.gz" \
  > "$BACKUP_DIR/k3s-server-db-before-uninstall-clean.tar.gz.sha256"

ls -lh "$BACKUP_DIR"
```

Salida esperada:

```text
k3s-local-path-storage-before-uninstall-clean.tar.gz
k3s-local-path-storage-before-uninstall-clean.tar.gz.sha256
k3s-server-db-before-uninstall-clean.tar.gz
k3s-server-db-before-uninstall-clean.tar.gz.sha256
```

En la instalación validada, el backup limpio generó una salida similar:

```text
RC_STORAGE=0
RC_DB=0

Backup limpio generado correctamente.

k3s-local-path-storage-before-uninstall-clean.tar.gz   1.1G
k3s-server-db-before-uninstall-clean.tar.gz            3.1M
```

Iniciar nuevamente K3s:

```bash
sudo systemctl start k3s
```

Validar que el nodo vuelva a estar disponible:

```bash
export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"

for i in {1..40}; do
  kubectl get nodes -o wide && break
  echo "API Server aún no disponible. Reintentando..."
  sleep 5
done
```

Salida esperada:

```text
NAME   STATUS   ROLES                  VERSION
vm-1   Ready    control-plane,master   v1.33.4+k3s1
```

---

## 4. Ejecutar el desinstalador oficial

Ingresar al directorio de instalación:

```bash
cd "$INSTALL_DIR"
```

Ejecutar el desinstalador oficial en modo interactivo:

```bash
./bin/uninstall-products-on-vm
```

Durante la ejecución, el script solicitará confirmación para eliminar los productos instalados. Responder `yes` cuando se requiera confirmar la desinstalación.

Ejemplo de confirmación:

```text
Are you sure you want to uninstall Concert? (yes/no):
yes
```

En caso de que el instalador solicite namespaces para Concert Workflows o FaaS, usar los namespaces desplegados durante la instalación. En una instalación quick-start estándar pueden ser:

```text
Concert Workflows namespace: concert-workflows
FaaS namespace: faas
```

El script también puede ejecutarse sin prompts usando:

```bash
./bin/uninstall-products-on-vm --force
```

No obstante, para documentación operativa se recomienda el modo interactivo, ya que permite validar cada producto antes de eliminarlo.

---

## 5. Validar estado posterior a la desinstalación

Finalizada la desinstalación, validar el estado del ambiente:

```bash
export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"

systemctl is-active k3s || true
kubectl get nodes -o wide || true
kubectl get ns || true
kubectl get pods -A || true
kubectl get pvc -A || true
kubectl get pv || true
```

Validar que no existan recursos relacionados a Concert:

```bash
kubectl get ns | egrep "concert|platform-hub|faas" || echo "No se encontraron namespaces relacionados a Concert"

kubectl get pods -A | egrep "concert|platform-hub|faas" || echo "No se encontraron pods relacionados a Concert"

kubectl get pvc -A | egrep "concert|platform-hub|faas" || echo "No se encontraron PVC relacionados a Concert"
```

Salida esperada:

```text
No se encontraron namespaces relacionados a Concert
No se encontraron pods relacionados a Concert
No se encontraron PVC relacionados a Concert
```

Validar que la URL de IBM Concert ya no responda:

```bash
curl -k -I --connect-timeout 15 https://localhost:12443 || true
```

Salida esperada:

```text
Connection refused
```

o una respuesta equivalente indicando que el servicio ya no se encuentra disponible.

Validar el directorio de instalación:

```bash
ls -ld "$INSTALL_DIR" 2>/dev/null || echo "Directorio de instalación no encontrado: $INSTALL_DIR"
```

Salida esperada:

```text
Directorio de instalación no encontrado: /home/concertadmin/concert-install/ibm-concert
```

Si el directorio aún existe, revisar su contenido antes de eliminarlo manualmente, especialmente si se requiere conservar logs o archivos de configuración.

---

## Troubleshooting

### Error: `tar: file changed as we read it`

Durante el backup puede aparecer una advertencia similar:

```text
tar: storage/.../undo_001: file changed as we read it
```

Causa probable:

Algún proceso seguía escribiendo en los volúmenes persistentes mientras se generaba el backup.

Acción recomendada:

Detener K3s, repetir el backup y validar que los códigos de retorno sean `0`.

```bash
sudo systemctl stop k3s
systemctl is-active k3s || true

sudo tar czf "$BACKUP_DIR/k3s-local-path-storage-before-uninstall-clean.tar.gz" \
  -C /var/lib/rancher/k3s storage

echo "RC_STORAGE=$?"

sudo tar czf "$BACKUP_DIR/k3s-server-db-before-uninstall-clean.tar.gz" \
  -C /var/lib/rancher/k3s/server db

echo "RC_DB=$?"
```

Salida esperada:

```text
RC_STORAGE=0
RC_DB=0
```

Luego iniciar K3s nuevamente:

```bash
sudo systemctl start k3s
kubectl get nodes -o wide
```

### Error: `The connection to the server 127.0.0.1:6443 was refused`

Este error puede aparecer si K3s quedó detenido luego del backup:

```text
The connection to the server 127.0.0.1:6443 was refused
```

Acción recomendada:

```bash
sudo systemctl start k3s
systemctl is-active k3s
kubectl get nodes -o wide
```

Salida esperada:

```text
active

NAME   STATUS   ROLES
vm-1   Ready    control-plane,master
```

### Ruta `localstorage/volumes` no encontrada

En esta instalación se validó que `localstorage` solo contenía logs:

```text
/home/concertadmin/concert-install/ibm-concert/localstorage/logs
```

Por ello, el backup se realizó sobre la ruta persistente real de K3s:

```text
/var/lib/rancher/k3s/storage
```

Antes de cada desinstalación, validar siempre los PV con:

```bash
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.local.path}{"\n"}{end}'
```

---

## Recomendaciones

- No ejecutar la desinstalación sin antes respaldar los datos persistentes.
- No asumir que los volúmenes se encuentran en `localstorage/volumes`; validar siempre los PV y sus rutas reales.
- Mantener los backups fuera del directorio de instalación de IBM Concert.
- Validar que el backup tenga hash SHA256 antes de continuar.
- Usar modo interactivo para desinstalaciones documentadas.
- Usar `--force` solo cuando se tenga certeza de que se desea eliminar todos los productos sin confirmaciones.
- No documentar credenciales, tokens, passwords ni entitlement keys.
- Después de desinstalar, validar namespaces, pods, PVC, PV y acceso HTTPS.
