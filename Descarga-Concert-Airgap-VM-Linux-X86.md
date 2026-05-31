# Descarga de imágenes IBM Concert para instalación air-gapped en VM Linux X86

## 1. Objetivo

El presente documento describe el procedimiento para descargar las imágenes requeridas por IBM Concert, Concert Workflows y Concert Data Apps desde un servidor Linux con acceso a internet, generar un paquete comprimido en formato `.tar.gz` y trasladarlo posteriormente hacia el servidor destino donde se realizará la instalación en modalidad air-gapped.

Este procedimiento está orientado únicamente a la etapa de descarga, empaquetado y transferencia del instalador. No cubre la instalación final de IBM Concert en el servidor destino.

## 2. Alcance

El procedimiento cubre las siguientes actividades:

1. Validación de prerrequisitos en el servidor con acceso a internet.
2. Instalación de paquetes base según sistema operativo Linux.
3. Configuración del runtime de contenedores requerido para la descarga de imágenes.
4. Descarga del paquete de instalación de IBM Concert.
5. Descarga de prerrequisitos de K3s y Helm para instalación air-gapped.
6. Definición de productos a descargar.
7. Descarga de imágenes desde IBM Container Registry.
8. Validación de archivos generados.
9. Compresión del paquete airgap en formato `.tar.gz`.
10. Transferencia del paquete hacia el servidor destino.

## 3. Consideraciones generales

Para la descarga de imágenes se requiere un servidor Linux con acceso a internet. Este servidor no necesariamente será el servidor definitivo donde se instalará IBM Concert; puede ser una máquina temporal utilizada únicamente para preparar el paquete air-gapped.

El servidor de descarga debe contar con conectividad hacia los repositorios públicos requeridos, IBM Container Registry y GitHub. Asimismo, debe disponer de espacio suficiente para almacenar temporalmente el paquete de instalación, las imágenes descargadas y el archivo comprimido final.

Se recomienda considerar como mínimo **100 GB de espacio libre** para la actividad de descarga y compresión. Este valor puede variar según los productos seleccionados:

- IBM Concert.
- IBM Concert Workflows.
- IBM Concert Data Apps.

La arquitectura del servidor de descarga y del servidor destino debe ser compatible, preferentemente `x86_64`.

## 4. Prerrequisitos del servidor de descarga

El servidor desde donde se realizará la descarga debe contar con los siguientes requisitos:

| Requisito | Descripción |
|---|---|
| Sistema operativo | Linux x86_64. Puede utilizarse RHEL, Ubuntu, SUSE u otro sistema Linux compatible. |
| Espacio libre | Mínimo recomendado: 100 GB para descarga y compresión. |
| Usuario | Usuario con permisos `sudo` o acceso `root`. |
| Runtime de contenedores | Docker o Podman instalado. |
| Conectividad a internet | Acceso a GitHub, IBM Container Registry y repositorios del sistema operativo. |
| IBM Entitlement Key | Requerido para descargar imágenes desde IBM Container Registry. |
| Herramientas base | `curl`, `wget`, `tar`, `gzip`, `sha256sum`, `sed`, `findutils`. |

## 5. Validación inicial del servidor

Ejecutar los siguientes comandos para validar arquitectura, sistema operativo y espacio disponible.

Ejecutar comando:

```bash
uname -m
cat /etc/os-release
```

Ejemplo de salida esperada:

```bash
x86_64
```

Validar espacio disponible:

```bash
df -h
```

Ejemplo:

```bash
df -h /opt
```

El filesystem que se utilizará para la descarga debe contar con al menos 100 GB libres.

Validar conectividad:

```bash
curl -I https://github.com
curl -I https://cp.icr.io
```

## 6. Creación de rutas de trabajo

Para este procedimiento se utilizarán las siguientes rutas de referencia:

| Variable | Ruta | Descripción |
|---|---|---|
| `WORKDIR` | `/opt/ibm-concert-airgap` | Ruta de trabajo para descargar y preparar el paquete. |
| `OUTDIR` | `/opt/ibm-concert-output` | Ruta donde se generará el `.tar.gz` final. |

Ejecutar comando:

```bash
export WORKDIR=/opt/ibm-concert-airgap
export OUTDIR=/opt/ibm-concert-output

sudo mkdir -p ${WORKDIR}
sudo mkdir -p ${OUTDIR}
sudo chown -R $(id -u):$(id -g) ${WORKDIR} ${OUTDIR}
```

Validar rutas:

```bash
ls -ld ${WORKDIR} ${OUTDIR}
df -h ${WORKDIR} ${OUTDIR}
```

## 7. Instalación de paquetes base

### 7.1 Ubuntu

Ejecutar comando:

```bash
sudo apt update
sudo apt install -y \
  curl \
  wget \
  tar \
  gzip \
  ca-certificates \
  gnupg \
  lsb-release \
  coreutils \
  findutils \
  sed
```

Instalar Docker:

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

Validar Docker:

```bash
docker --version
sudo docker info | head
```

Asignar permisos al usuario actual, si se requiere ejecutar Docker sin `sudo`:

```bash
sudo usermod -aG docker $USER
```

Nota: cerrar sesión y volver a ingresar para que el cambio de grupo tome efecto.

### 7.2 Red Hat Enterprise Linux / Rocky Linux / AlmaLinux

Ejecutar comando:

```bash
sudo dnf install -y \
  curl \
  wget \
  tar \
  gzip \
  ca-certificates \
  coreutils \
  findutils \
  sed
```

Instalar Podman:

```bash
sudo dnf install -y podman
```

Validar Podman:

```bash
podman --version
podman info | head
```

### 7.3 SUSE Linux Enterprise Server / openSUSE

Ejecutar comando:

```bash
sudo zypper refresh
sudo zypper install -y \
  curl \
  wget \
  tar \
  gzip \
  ca-certificates \
  coreutils \
  findutils \
  sed
```

Instalar Docker:

```bash
sudo zypper install -y docker
sudo systemctl enable --now docker
```

Validar Docker:

```bash
docker --version
sudo docker info | head
```

## 8. Validación de storage del runtime de contenedores

Antes de descargar las imágenes, validar que el runtime de contenedores tenga espacio suficiente. Las rutas más comunes son:

| Runtime | Ruta por defecto |
|---|---|
| Docker | `/var/lib/docker` |
| Podman root | `/var/lib/containers` |
| Podman rootless | `~/.local/share/containers` |

Validar espacio en Docker:

```bash
docker info | grep "Docker Root Dir"
df -h /var/lib/docker
```

Validar espacio en Podman:

```bash
podman info | grep -i graphroot -A 2
df -h /var/lib/containers
```

Si el filesystem raíz `/` no tiene espacio suficiente, se recomienda configurar una ruta alternativa para el runtime antes de ejecutar la descarga.

### 8.1 Configuración opcional de Docker con ruta alternativa

Ejemplo para utilizar `/opt/container-storage/docker` como ruta de Docker:

```bash
sudo mkdir -p /opt/container-storage/docker/data
sudo mkdir -p /opt/container-storage/docker/run
```

Editar el archivo de configuración:

```bash
sudo vi /etc/docker/daemon.json
```

Agregar el siguiente contenido:

```json
{
  "data-root": "/opt/container-storage/docker/data",
  "exec-root": "/opt/container-storage/docker/run"
}
```

Reiniciar Docker:

```bash
sudo systemctl restart docker
```

Validar la nueva ruta:

```bash
docker info | grep "Docker Root Dir"
```

### 8.2 Configuración opcional de Podman con ruta alternativa

Ejemplo para usuario root:

```bash
sudo mkdir -p /opt/container-storage/containers
sudo vi /etc/containers/storage.conf
```

Validar o modificar los parámetros:

```ini
[storage]
driver = "overlay"
graphroot = "/opt/container-storage/containers"
```

Validar configuración:

```bash
podman info | grep -i graphroot -A 2
```

## 9. Descarga del paquete de instalación de IBM Concert

Ingresar a la ruta de trabajo:

```bash
cd ${WORKDIR}
```

Definir la versión de IBM Concert a descargar:

```bash
export CONCERT_VERSION=2.3.1
```

Descargar el paquete:

```bash
wget https://github.com/IBM/Concert/releases/download/v${CONCERT_VERSION}/ibm-concert-x86.tar.gz
```

Configurar permisos por defecto:

```bash
umask 0022
```

Extraer el paquete:

```bash
tar -xzf ibm-concert-x86.tar.gz
cd ibm-concert
```

Validar contenido:

```bash
ls -lh
```

## 10. Descarga de prerrequisitos para instalación air-gapped

Este paso descarga los componentes requeridos de K3s y Helm para que puedan ser incluidos dentro del paquete airgap.

Ejecutar comando:

```bash
export K3S_RELEASE_VER=v1.33.4+k3s1
export HELM_VERSION=${HELM_VERSION:-v3.20.0}

mkdir -p prerequisites

K3S_RELEASE_VER_ENCODE=$(echo $K3S_RELEASE_VER | sed 's/+/%2B/g')
curl -Lo ./prerequisites/k3s https://github.com/k3s-io/k3s/releases/download/${K3S_RELEASE_VER_ENCODE}/k3s
chmod a+x ./prerequisites/k3s

K3S_RELEASE_VER_SHORT=$(echo $K3S_RELEASE_VER | sed 's/+.*//g')
curl -Lo ./prerequisites/k3s-airgap-images-amd64.tar.zst https://github.com/k3s-io/k3s/releases/download/${K3S_RELEASE_VER_SHORT}%2Bk3s1/k3s-airgap-images-amd64.tar.zst

curl -Lo ./prerequisites/install.sh https://get.k3s.io/
chmod a+x ./prerequisites/install.sh

helm_download_name=helm-${HELM_VERSION}-linux-amd64.tar.gz
curl -Lo ./prerequisites/${helm_download_name} https://get.helm.sh/${helm_download_name}
```

Validar descarga de prerrequisitos:

```bash
ls -lh prerequisites/
```

## 11. Definición de productos a descargar

Crear o editar el archivo `etc/params.ini` para definir qué componentes se descargarán.

Ejecutar comando:

```bash
vi etc/params.ini
```

Ejemplo para descargar IBM Concert, Concert Workflows y Concert Data Apps:

```ini
INSTALL_CONCERT=TRUE
INSTALL_DATAAPPS=TRUE
INSTALL_WORKFLOWS=TRUE
```

Ejemplo para descargar únicamente IBM Concert:

```ini
INSTALL_CONCERT=TRUE
INSTALL_DATAAPPS=FALSE
INSTALL_WORKFLOWS=FALSE
```

Nota: Si el archivo `etc/params.ini` no existe, el script puede descargar las imágenes de todos los productos soportados. Por ello, se recomienda definir explícitamente los componentes requeridos.

## 12. Exportar IBM Entitlement Key

Exportar la llave de IBM Container Registry en la variable `REG_PASS`.

Ejecutar comando recomendado:

```bash
read -s -p "IBM Entitlement Key: " REG_PASS
export REG_PASS
echo
```

Validar que la variable existe sin exponer su contenido:

```bash
[ -n "$REG_PASS" ] && echo "REG_PASS configurado" || echo "REG_PASS no configurado"
```

Importante: no guardar el IBM Entitlement Key en archivos del repositorio, scripts versionados o historial compartido.

## 13. Descarga de imágenes de IBM Concert

Desde la ruta del instalador:

```bash
cd ${WORKDIR}/ibm-concert
```

Ejecutar la descarga de imágenes:

```bash
./bin/manage-images --save
```

El script descargará las imágenes desde IBM Container Registry y generará los archivos comprimidos dentro de la ruta `images/<version>/`.

Validar archivos generados:

```bash
ls -lh images/*/
du -sh images
```

Ejemplo de archivos esperados:

```bash
images-hub.tar.gz
images-concert.tar.gz
images-dataapps.tar.gz
images-workflows.tar.gz
```

Los archivos pueden variar según los productos seleccionados en `etc/params.ini`.

## 14. Validación previa a la compresión final

Antes de generar el paquete final, validar que existan las carpetas y archivos requeridos.

Ejecutar comando:

```bash
cd ${WORKDIR}
ls -ld ibm-concert
ls -lh ibm-concert/images/*/
ls -lh ibm-concert/prerequisites/
```

Validar tamaño total:

```bash
du -sh ibm-concert
```

Validar espacio disponible para el archivo final:

```bash
df -h ${OUTDIR}
```

## 15. Generación del paquete airgap comprimido

Crear el archivo `.tar.gz` final en la ruta de salida definida.

Ejecutar comando:

```bash
cd ${WORKDIR}
tar -czf ${OUTDIR}/ibm-concert-x86-airgap.tar.gz ibm-concert/
```

Validar archivo generado:

```bash
ls -lh ${OUTDIR}/ibm-concert-x86-airgap.tar.gz
```

Validar que el archivo comprimido pueda leerse correctamente:

```bash
tar -tzf ${OUTDIR}/ibm-concert-x86-airgap.tar.gz | head
```

## 16. Generación de checksum

Generar checksum para validar integridad del archivo al copiarlo al servidor destino.

Ejecutar comando:

```bash
cd ${OUTDIR}
sha256sum ibm-concert-x86-airgap.tar.gz | tee ibm-concert-x86-airgap.tar.gz.sha256
```

Validar archivos finales:

```bash
ls -lh ${OUTDIR}
```

## 17. Transferencia del paquete al servidor destino

El paquete generado debe ser copiado al servidor destino donde se realizará la instalación air-gapped.

### 17.1 Transferencia por SCP

Ejemplo:

```bash
scp ${OUTDIR}/ibm-concert-x86-airgap.tar.gz usuario@servidor-destino:/tmp/
scp ${OUTDIR}/ibm-concert-x86-airgap.tar.gz.sha256 usuario@servidor-destino:/tmp/
```

Ejemplo con ruta destino específica:

```bash
scp ${OUTDIR}/ibm-concert-x86-airgap.tar.gz usuario@servidor-destino:/opt/ibm-concert-airgap/
scp ${OUTDIR}/ibm-concert-x86-airgap.tar.gz.sha256 usuario@servidor-destino:/opt/ibm-concert-airgap/
```

### 17.2 Transferencia por disco externo o medio removible

Copiar el archivo generado hacia el disco externo:

```bash
cp ${OUTDIR}/ibm-concert-x86-airgap.tar.gz /ruta/disco-externo/
cp ${OUTDIR}/ibm-concert-x86-airgap.tar.gz.sha256 /ruta/disco-externo/
sync
```

Luego conectar el disco externo al servidor destino y copiar el contenido:

```bash
sudo mkdir -p /opt/ibm-concert-airgap
sudo cp /ruta/disco-externo/ibm-concert-x86-airgap.tar.gz /opt/ibm-concert-airgap/
sudo cp /ruta/disco-externo/ibm-concert-x86-airgap.tar.gz.sha256 /opt/ibm-concert-airgap/
```

### 17.3 Transferencia por storage compartido

Si existe un filesystem compartido entre el servidor de descarga y el servidor destino, copiar el paquete hacia la ruta acordada:

```bash
cp ${OUTDIR}/ibm-concert-x86-airgap.tar.gz /mnt/storage-compartido/
cp ${OUTDIR}/ibm-concert-x86-airgap.tar.gz.sha256 /mnt/storage-compartido/
sync
```

## 18. Validación en el servidor destino

En el servidor destino, ingresar a la ruta donde se copió el paquete:

```bash
cd /opt/ibm-concert-airgap
```

Validar integridad:

```bash
sha256sum -c ibm-concert-x86-airgap.tar.gz.sha256
```

La salida esperada debe ser similar a:

```bash
ibm-concert-x86-airgap.tar.gz: OK
```

Extraer el paquete:

```bash
tar -xzf ibm-concert-x86-airgap.tar.gz
cd ibm-concert
```

Validar contenido del paquete:

```bash
ls -lh
ls -lh images/*/
ls -lh prerequisites/
```

En este punto, el paquete airgap ya se encuentra disponible en el servidor destino para continuar con la instalación de IBM Concert conforme a la documentación oficial.

## 19. Comandos de limpieza opcional en el servidor de descarga

Una vez validado que el archivo fue copiado correctamente al servidor destino, se puede liberar espacio en el servidor de descarga.

Ejecutar comando:

```bash
rm -rf ${WORKDIR}/ibm-concert
rm -f ${WORKDIR}/ibm-concert-x86.tar.gz
```

Si se utilizó Docker y se desea liberar capas descargadas:

```bash
docker system df
sudo docker system prune -a
```

Si se utilizó Podman:

```bash
podman system df
podman system prune -a
```

Nota: ejecutar comandos de limpieza solo si el servidor de descarga no requiere conservar las imágenes localmente.

## 20. Resumen del flujo

```text
Servidor con internet
    |
    |-- Instala paquetes base y Docker/Podman
    |-- Descarga ibm-concert-x86.tar.gz
    |-- Descarga prerrequisitos K3s y Helm
    |-- Define productos en etc/params.ini
    |-- Ejecuta ./bin/manage-images --save
    |-- Genera ibm-concert-x86-airgap.tar.gz
    |-- Genera checksum SHA256
    |
    v
Transferencia por SCP, disco externo o storage compartido
    |
    v
Servidor destino air-gapped
    |
    |-- Recibe paquete .tar.gz
    |-- Valida checksum
    |-- Extrae paquete
    |-- Queda listo para continuar instalación
```

## 21. Referencias

- IBM Concert - Installing Concert, Concert Workflows, and Concert Data Apps in an air-gapped environment VM: https://www.ibm.com/docs/en/concert/2.3.x?topic=dvmv-installing-concert-concert-workflows-concert-data-apps-in-air-gapped-environment-vm
- IBM Concert - System requirements VM: https://www.ibm.com/docs/en/concert/2.3.x?topic=planning-system-requirements-vm
- IBM Concert releases: https://github.com/IBM/Concert/releases
