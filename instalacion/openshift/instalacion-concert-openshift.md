# Instalación de IBM Concert sobre Red Hat OpenShift

## 1. Objetivo

Documentar el procedimiento base para instalar **IBM Concert**, **Concert Data Apps** y **Concert Workflows** sobre un cluster **Red Hat OpenShift Container Platform (OCP)** en modalidad conectada.

Este procedimiento considera que el host de instalación o bastion tiene acceso a internet y puede descargar las imágenes directamente desde **IBM Container Registry** (`cp.icr.io`).

---

## 2. Alcance

Este documento cubre:

- Validación de prerrequisitos del cluster OpenShift.
- Validación de herramientas requeridas en el host de instalación.
- Descarga del paquete de IBM Concert.
- Preparación del archivo `params.ini`.
- Instalación de IBM Concert, Concert Data Apps y Concert Workflows.
- Exposición de rutas en OpenShift.
- Validaciones posteriores a la instalación.
- Comandos básicos de troubleshooting.

Este documento no cubre:

- Instalación air-gapped sobre OpenShift.
- Mirroring de imágenes hacia un registry privado.
- Configuración avanzada de certificados.
- Configuración avanzada de Istio o Service Mesh.
- Integración con watsonx.ai.
- Backup, restore o upgrade.

> Nota: la integración con watsonx.ai debe documentarse en un procedimiento separado, porque corresponde a una configuración posterior o complementaria a la instalación base.

---

## 3. Flujo general de instalación

```text
Host de instalación / Bastion con internet
        │
        ├── Validación de acceso al cluster OpenShift
        ├── Validación de storage class y dominio de aplicaciones
        ├── Descarga del paquete IBM Concert
        ├── Configuración de params.ini
        ├── Ejecución de bin/setup
        │
        ▼
Cluster OpenShift
        │
        ├── Namespace Hub
        ├── Namespace Concert
        ├── Namespace Data Apps
        ├── Namespace Workflows
        └── Rutas de acceso
```

---

## 4. Prerrequisitos

### 4.1 Cluster OpenShift

El cluster OpenShift debe contar con:

- Red Hat OpenShift Container Platform 4.14.0 o superior para arquitectura x86.
- Cluster Linux x86 para la instalación estándar.
- Acceso administrativo al cluster.
- Nodos worker con capacidad suficiente.
- Storage class configurada.
- Dominio de aplicaciones configurado en OpenShift.
- Conectividad desde el cluster hacia IBM Container Registry (`cp.icr.io`) para una instalación conectada.

IBM recomienda como referencia para OCP:

| Rol | Arquitectura | Cantidad mínima | vCPU mínimo | Memoria recomendada | Storage mínimo |
|---|---|---:|---:|---:|---:|
| Worker / Compute | x86_64 | 4 nodos | 16 vCPU por nodo | 64 GB RAM por nodo | 500 GB por nodo |

Para IBM Power, validar la versión mínima soportada de OCP y la arquitectura correspondiente antes de ejecutar el procedimiento.

---

### 4.2 Host de instalación

El host desde donde se ejecutará la instalación debe contar con:

- Sistema operativo Linux.
- Acceso al cluster OpenShift.
- Acceso a internet.
- Acceso a `cp.icr.io`.
- Cliente `oc` instalado.
- Cliente `kubectl` disponible.
- Helm 3.x instalado, requerido si se instalará Concert Workflows.
- Herramientas base: `wget`, `curl`, `tar`, `vi` o `vim`.
- IBM Entitlement Key válida.

Sistemas operativos sugeridos para el host de instalación:

- Red Hat Enterprise Linux 8 o superior.
- Red Hat Enterprise Linux 9 o superior.
- Ubuntu Server 20.04 o superior.
- Ubuntu Server 22.04 o superior.
- Rocky Linux / AlmaLinux.
- SUSE Linux Enterprise Server, si se dispone de paquetes equivalentes.

---

### 4.3 Instalación de paquetes base en el host

#### Red Hat / Rocky / AlmaLinux

```bash
sudo dnf install -y wget curl tar gzip vim jq
```

#### Ubuntu / Debian

```bash
sudo apt-get update
sudo apt-get install -y wget curl tar gzip vim jq
```

#### SUSE Linux Enterprise Server

```bash
sudo zypper install -y wget curl tar gzip vim jq
```

---

### 4.4 Validar cliente OpenShift

```bash
oc version
```

Validar cliente `kubectl`:

```bash
kubectl version --client
```

Validar Helm:

```bash
helm version
```

Si Helm no está instalado, se puede instalar con:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

---

### 4.5 Validar conectividad hacia IBM Container Registry

```bash
curl -I https://cp.icr.io
```

También se puede validar resolución DNS:

```bash
nslookup cp.icr.io
```

---

### 4.6 Validar IPv6 para Concert Workflows

Cuando se instala Concert Workflows, validar si IPv6 se encuentra habilitado en el host o nodo donde aplique la validación:

```bash
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
```

Resultado esperado:

```text
0
```

Si el resultado es `1`, IPv6 se encuentra deshabilitado y debe revisarse antes de continuar.

> Nota: este requisito aplica para versiones específicas de Concert Workflows. Validar siempre con la documentación oficial de la versión que se instalará.

---

## 5. Acceso al cluster OpenShift

Iniciar sesión en OpenShift:

```bash
oc login --token=<TOKEN> --server=<API_SERVER>
```

Validar el usuario conectado:

```bash
oc whoami
```

Validar permisos administrativos:

```bash
oc auth can-i '*' '*' --all-namespaces
```

Resultado esperado:

```text
yes
```

Validar nodos del cluster:

```bash
oc get nodes -o wide
```

Validar versión del cluster:

```bash
oc get clusterversion
```

---

## 6. Validaciones previas en OpenShift

### 6.1 Validar storage classes

```bash
oc get sc
```

Validar si existe una storage class por defecto:

```bash
oc get sc | grep default
```

En este procedimiento se usa como ejemplo la siguiente storage class:

```text
ocs-external-storagecluster-ceph-rbd
```

Ajustar este valor según el ambiente destino.

---

### 6.2 Validar dominio de aplicaciones del cluster

Concert Workflows requiere definir el parámetro `WORKFLOWS_INSTANCE_ADDRESS`.

Obtener el dominio base de aplicaciones:

```bash
oc get ingresses.config cluster -o jsonpath='{.spec.domain}'; echo
```

Ejemplo de salida:

```text
apps.midominio.openshift.com
```

Con base en este dominio, definir el FQDN para Workflows:

```text
WORKFLOWS_INSTANCE_ADDRESS=solis-gw-concertworkflows.apps.midominio.openshift.com
```

---

## 7. Descargar IBM Concert

Crear directorio de trabajo:

```bash
mkdir -p $HOME/CONCERT
cd $HOME/CONCERT
```

Descargar paquete para arquitectura x86:

```bash
wget https://github.com/IBM/Concert/releases/download/v2.3.1/ibm-concert-x86.tar.gz
```

Para arquitectura IBM Power, validar el paquete correspondiente antes de descargar. Como referencia, IBM publica paquetes `ppc64le` para Power.

Extraer paquete:

```bash
tar xfz ibm-concert-x86.tar.gz
```

Definir variable de instalación:

```bash
export INSTALL_DIR=$HOME/CONCERT/ibm-concert
cd $INSTALL_DIR
```

Validar contenido:

```bash
ls -la
```

---

## 8. Preparar archivo params.ini

Listar archivos de ejemplo disponibles:

```bash
ls -la $INSTALL_DIR/etc/sample-params/
```

Para instalar Concert, Concert Data Apps y Concert Workflows sobre Kubernetes/OpenShift, copiar el archivo de parámetros correspondiente:

```bash
cp $INSTALL_DIR/etc/sample-params/concert-dataapps-workflows-k8s-params.ini \
   $INSTALL_DIR/etc/params.ini
```

En algunas versiones puede existir un archivo quickstart. Validar el nombre exacto con:

```bash
ls $INSTALL_DIR/etc/sample-params/ | grep k8s
```

Crear backup del archivo:

```bash
cp -p $INSTALL_DIR/etc/params.ini $INSTALL_DIR/etc/params.ini.bak
```

Editar el archivo:

```bash
vi $INSTALL_DIR/etc/params.ini
```

---

## 9. Ejemplo de params.ini

A continuación se muestra un ejemplo de configuración para una instalación conectada sobre OpenShift.

Ajustar namespaces, storage class y dominio según el ambiente destino.

```ini
##### PARAMS.INI #####

# Tipo de instalación
INSTALL_EKS=false

# Registry IBM
REG_USER=cp
IMAGE_REGISTRY_PREFIX=cp.icr.io/cp

# HUB
HUB_NS=concert-hub
HUB_IMAGE_REGISTRY_SUFFIX=/solis-hub
SCALE_CONFIG_HUB=level_1
STORAGE_CLASS_HUB=ocs-external-storagecluster-ceph-rbd

# CONCERT
INSTALL_CONCERT=true
CONCERT_NS=concert
CONCERT_IMAGE_REGISTRY_SUFFIX=/concert
SCALE_CONFIG_CONCERT=level_1
STORAGE_CLASS_CONCERT=ocs-external-storagecluster-ceph-rbd

# DATA APPS
INSTALL_DATAAPPS=true
DATAAPPS_NS=concert-dataapps
DATAAPPS_IMAGE_REGISTRY_SUFFIX=/concert
SCALE_CONFIG_DATAAPPS=level_1
STORAGE_CLASS_DATAAPPS=ocs-external-storagecluster-ceph-rbd

# WORKFLOWS
INSTALL_WORKFLOWS=true
WORKFLOWS_NS=concert-workflows
WORKFLOWS_IMAGE_REGISTRY_SUFFIX=/concert
WORKFLOWS_INSTANCE_ADDRESS=solis-gw-concertworkflows.apps.midominio.openshift.com
STORAGE_CLASS_WORKFLOWS=ocs-external-storagecluster-ceph-rbd
```

---

## 10. Descripción de parámetros principales

| Parámetro | Descripción |
|---|---|
| `INSTALL_EKS=false` | Indica que la instalación no corresponde a Amazon EKS. |
| `REG_USER=cp` | Usuario utilizado para autenticación contra IBM Container Registry. |
| `IMAGE_REGISTRY_PREFIX=cp.icr.io/cp` | Registry origen desde donde se descargarán las imágenes. |
| `HUB_NS` | Namespace donde se desplegará el Hub. |
| `HUB_IMAGE_REGISTRY_SUFFIX` | Sufijo de imágenes asociado al Hub. |
| `CONCERT_NS` | Namespace donde se desplegará IBM Concert. |
| `CONCERT_IMAGE_REGISTRY_SUFFIX` | Sufijo de imágenes asociado a IBM Concert. |
| `DATAAPPS_NS` | Namespace donde se desplegará Concert Data Apps. |
| `DATAAPPS_IMAGE_REGISTRY_SUFFIX` | Sufijo de imágenes asociado a Data Apps. |
| `WORKFLOWS_NS` | Namespace donde se desplegará Concert Workflows. |
| `WORKFLOWS_IMAGE_REGISTRY_SUFFIX` | Sufijo de imágenes asociado a Workflows. |
| `WORKFLOWS_INSTANCE_ADDRESS` | FQDN de acceso para Concert Workflows. |
| `STORAGE_CLASS_*` | Storage class que utilizará cada componente. |
| `SCALE_CONFIG_*` | Nivel de recursos definido para cada componente. |

---

## 11. Consideración sobre watsonx.ai

Si se requiere habilitar capacidades de IA para Concert Workflows, la configuración debe manejarse como un procedimiento adicional.

IBM permite crear un archivo de valores personalizados para habilitar AI en Workflows. Por ejemplo:

```yaml
rna:
  instance:
    ai:
      watsonx_auth_type: "iam"
      watsonx_api_key: "<WATSONX_API_KEY>"
      watsonx_project_id: "<WATSONX_PROJECT_ID>"
      watsonx_cluster_url: "https://us-south.ml.cloud.ibm.com"
```

No se recomienda incluir estos valores directamente en el procedimiento base ni subir credenciales al repositorio.

Ubicación sugerida para documentarlo de forma separada:

```text
integraciones/watsonx-ai/Configuracion-WatsonxAI-Concert.md
```

---

## 12. Ejecutar instalación

Por seguridad, no colocar passwords directamente en el comando ni dejarlos en el historial.

Solicitar password inicial del usuario administrador de Concert:

```bash
read -s -p "Password inicial de admin Concert: " CONCERT_ADMIN_PASSWORD
echo
```

Solicitar IBM Entitlement Key:

```bash
read -s -p "IBM Entitlement Key: " REGISTRY_PASSWORD
echo
```

Ejecutar instalación:

```bash
$INSTALL_DIR/bin/setup \
  --license_acceptance=y \
  --username=admin \
  --password="${CONCERT_ADMIN_PASSWORD}" \
  --registry_password="${REGISTRY_PASSWORD}"
```

La instalación puede tomar varios minutos, dependiendo del cluster, la conectividad y la descarga de imágenes.

Resultado esperado:

```text
DEPLOYMENT SUCCESSFUL
```

---

## 13. Validaciones posteriores a la instalación

### 13.1 Validar namespaces

```bash
oc get ns | grep concert
```

Ejemplo esperado:

```text
concert-hub
concert
concert-dataapps
concert-workflows
```

---

### 13.2 Validar pods

Validar pods del Hub:

```bash
oc get pods -n concert-hub
```

Validar pods de Concert:

```bash
oc get pods -n concert
```

Validar pods de Data Apps:

```bash
oc get pods -n concert-dataapps
```

Validar pods de Workflows:

```bash
oc get pods -n concert-workflows
```

Validar todos los pods asociados:

```bash
oc get pods -A | grep concert
```

---

### 13.3 Validar deployments

```bash
oc get deploy -n concert-hub
oc get deploy -n concert
oc get deploy -n concert-dataapps
oc get deploy -n concert-workflows
```

---

### 13.4 Validar statefulsets

```bash
oc get statefulset -n concert-hub
oc get statefulset -n concert
oc get statefulset -n concert-dataapps
oc get statefulset -n concert-workflows
```

---

### 13.5 Validar PVC

```bash
oc get pvc -n concert-hub
oc get pvc -n concert
oc get pvc -n concert-dataapps
oc get pvc -n concert-workflows
```

Los PVC deben encontrarse en estado:

```text
Bound
```

---

## 14. Exponer rutas en OpenShift

### 14.1 Exponer ruta de Concert

Ejecutar:

```bash
$INSTALL_DIR/ibm-concert-k8s/ocp-route.sh concert
```

Validar ruta:

```bash
oc get route concert -n concert
```

Extraer configuración generada:

```bash
oc extract -n concert secret/app-cfg-secret --to=-
```

> Importante: este comando puede mostrar información sensible. No copiar su salida en repositorios, tickets públicos, capturas o documentos compartidos.

---

### 14.2 Exponer ruta de Concert Data Apps

Ejecutar:

```bash
$INSTALL_DIR/ibm-dataapps-k8s/ocp-route.sh concert-dataapps
```

Validar ruta:

```bash
oc get route -n concert-dataapps
```

---

### 14.3 Validar ruta de Concert Workflows

Definir namespace:

```bash
export CW_NS=concert-workflows
```

Validar rutas:

```bash
oc get route -n ${CW_NS}
```

Identificar la ruta `solis-gw` o la ruta generada para Workflows.

Ejemplo:

```text
solis-gw-concertworkflows.apps.midominio.openshift.com
```

URL de acceso:

```text
https://solis-gw-concertworkflows.apps.midominio.openshift.com
```

---

## 15. Validación de acceso

Obtener host de Concert:

```bash
oc get route concert -n concert -o jsonpath='{.spec.host}'; echo
```

Acceder desde navegador:

```text
https://<ROUTE_HOST>
```

Credenciales iniciales:

```text
Usuario: admin
Password: definido durante la ejecución de bin/setup
```

Acceso a Workflows:

```text
https://<WORKFLOWS_ROUTE_HOST>/workflows/
```

Acceso a Data Apps:

```text
https://<DATAAPPS_ROUTE_HOST>
```

---

## 16. Comandos útiles de troubleshooting

### 16.1 Revisar eventos

```bash
oc get events -n concert-hub --sort-by=.lastTimestamp
oc get events -n concert --sort-by=.lastTimestamp
oc get events -n concert-dataapps --sort-by=.lastTimestamp
oc get events -n concert-workflows --sort-by=.lastTimestamp
```

---

### 16.2 Describir pod con error

```bash
oc describe pod <pod_name> -n <namespace>
```

---

### 16.3 Revisar logs

```bash
oc logs <pod_name> -n <namespace>
```

Si el pod tiene más de un contenedor:

```bash
oc logs <pod_name> -c <container_name> -n <namespace>
```

---

### 16.4 Validar imágenes usadas por los pods

```bash
oc get pods -n concert -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.spec.containers[*].image}{"\n"}{end}'
```

---

### 16.5 Validar rutas

```bash
oc get route -A | grep concert
```

---

### 16.6 Validar secrets relacionados

```bash
oc get secrets -n concert
oc get secrets -n concert-hub
oc get secrets -n concert-dataapps
oc get secrets -n concert-workflows
```

---

## 17. Errores comunes

### 17.1 Error de autenticación contra IBM Container Registry

Validar:

- IBM Entitlement Key vigente.
- Acceso a `cp.icr.io`.
- Parámetro `REG_USER=cp`.
- Parámetro `IMAGE_REGISTRY_PREFIX=cp.icr.io/cp`.
- Restricciones de proxy o firewall.

Comando de referencia:

```bash
curl -I https://cp.icr.io
```

---

### 17.2 Pods en ImagePullBackOff

Validar eventos:

```bash
oc get events -n <namespace> --sort-by=.lastTimestamp
```

Describir pod:

```bash
oc describe pod <pod_name> -n <namespace>
```

Revisar si el error corresponde a:

- Credenciales inválidas.
- Registry no alcanzable.
- Imagen no encontrada.
- Restricciones de red.
- Proxy no configurado.

---

### 17.3 PVC en Pending

Validar storage classes:

```bash
oc get sc
```

Describir PVC:

```bash
oc describe pvc <pvc_name> -n <namespace>
```

Revisar si:

- La storage class existe.
- La storage class permite aprovisionamiento dinámico.
- Hay capacidad disponible en el backend de almacenamiento.
- El nombre usado en `params.ini` coincide con el nombre real de la storage class.

---

### 17.4 Ruta no accesible

Validar dominio de aplicaciones:

```bash
oc get ingresses.config cluster -o jsonpath='{.spec.domain}'; echo
```

Validar rutas:

```bash
oc get route -A | grep concert
```

Describir ruta:

```bash
oc describe route concert -n concert
```

---

### 17.5 Error por ubicación incorrecta de params.ini

Validar que el archivo exista en la ruta correcta:

```bash
ls -la $INSTALL_DIR/etc/params.ini
```

Validar backup:

```bash
ls -la $INSTALL_DIR/etc/params.ini.bak
```

Validar parámetros principales:

```bash
grep -E "INSTALL_|NS=|STORAGE_CLASS|IMAGE_REGISTRY|WORKFLOWS_INSTANCE" $INSTALL_DIR/etc/params.ini
```

---

## 18. Limpieza de variables sensibles

Después de ejecutar la instalación, limpiar variables sensibles de la sesión:

```bash
unset REGISTRY_PASSWORD
unset CONCERT_ADMIN_PASSWORD
```

Evitar guardar credenciales en:

```text
.bash_history
params.ini
README.md
tickets
capturas de pantalla
repositorios Git
```

Si se ejecutó un comando con credenciales visibles, revisar el historial:

```bash
history | tail
```

Si corresponde, eliminar entradas sensibles del historial de acuerdo con las políticas de la organización.

---

## 19. Resumen operativo

```text
1. Validar acceso administrativo al cluster OpenShift.
2. Validar versión del cluster, nodos, storage class y dominio de aplicaciones.
3. Validar herramientas del host de instalación: oc, kubectl, helm, wget, curl y tar.
4. Descargar el paquete IBM Concert.
5. Extraer el paquete y definir INSTALL_DIR.
6. Copiar el archivo base params.ini.
7. Ajustar namespaces, storage class, registry y FQDN de Workflows.
8. Ejecutar bin/setup con usuario, password y entitlement key.
9. Validar namespaces, pods, deployments, statefulsets y PVC.
10. Exponer rutas de Concert y Data Apps.
11. Validar ruta de Workflows.
12. Acceder a la consola web.
13. Limpiar variables sensibles.
```

---

## 20. Ubicación sugerida en el repositorio

```text
instalacion/openshift/Instalacion-Concert-OpenShift.md
```

---

## 21. Referencias

- IBM Concert 2.3.x - Deploying on Kubernetes: https://www.ibm.com/docs/en/concert/2.3.x?topic=deployment-deploying-kubernetes
- IBM Concert 2.3.x - Installing Concert, Concert Workflows, and Concert Data Apps on OCP: https://www.ibm.com/docs/en/concert/2.3.x?topic=kubernetes-installing-ocp-cluster
- IBM Concert 2.3.x - Configuring params.ini file: https://www.ibm.com/docs/en/concert/2.3.x?topic=deployment-configuring-paramsini-file
- IBM Concert GitHub Releases: https://github.com/IBM/Concert/releases
