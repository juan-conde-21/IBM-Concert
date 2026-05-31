# Descarga y carga de imágenes de IBM Concert para instalación air-gapped sobre OpenShift

## 1. Objetivo

Documentar el procedimiento para descargar y copiar las imágenes de IBM Concert, Concert Data Apps y Concert Workflows hacia un registry privado, como paso previo a una instalación air-gapped sobre Red Hat OpenShift.

Este procedimiento está orientado al escenario donde el cluster OpenShift no tiene acceso directo a IBM Container Registry (`cp.icr.io`) y requiere consumir las imágenes desde un registry privado accesible dentro de la red restringida.

> Nota de terminología: en este documento se utiliza el término **cargar imágenes al registry privado** para describir el proceso de llevar las imágenes desde IBM Container Registry hacia el registry destino. La opción técnica del comando IBM sigue siendo `--mirror`, porque así está definida en la herramienta `manage-images`.

---

## 2. Alcance

Este documento cubre:

- Validación de prerrequisitos para la máquina de descarga o bastion.
- Descarga del paquete base de IBM Concert.
- Preparación del entorno de trabajo.
- Autenticación contra IBM Container Registry.
- Autenticación contra el registry privado destino.
- Copia y publicación de imágenes hacia el registry privado.
- Validación básica de las imágenes cargadas.
- Recomendaciones para continuar con la instalación en OpenShift.

Este documento no cubre:

- Instalación de IBM Concert sobre OpenShift.
- Configuración final del archivo `params.ini` para despliegue.
- Exposición de rutas.
- Configuración de certificados.
- Configuración de watsonx.ai.
- Instalación o administración del registry privado.
- Procedimientos de backup, restore o upgrade.

---

## 3. Arquitectura del proceso

El flujo recomendado para OpenShift air-gapped es el siguiente:

```text
Máquina de descarga / Bastion
        │
        ├── Acceso a internet
        ├── Acceso a IBM Container Registry
        ├── Acceso al registry privado
        │
        ▼
Registry privado
        │
        ├── Recibe imágenes copiadas
        ├── Almacena imágenes de IBM Concert
        └── Queda disponible para OpenShift
                │
                ▼
Cluster OpenShift air-gapped
        │
        └── Consume imágenes desde el registry privado
```

---

## 4. Prerrequisitos

### 4.1 Máquina de descarga o bastion

La máquina desde donde se ejecutará la descarga y el copia de imágenes debe contar con:

- Sistema operativo Linux.
- Acceso a internet.
- Acceso a IBM Container Registry: `cp.icr.io`.
- Acceso al registry privado destino.
- Docker o Podman instalado.
- IBM Entitlement Key vigente.
- Credenciales del registry privado destino.
- Mínimo 100 GB libres para descarga, capas temporales y ejecución del proceso.
- Usuario con privilegios `sudo`.
- Herramientas base: `curl`, `wget`, `tar`, `gzip`, `jq`, `sha256sum`.

Sistemas operativos sugeridos:

- Red Hat Enterprise Linux 8 o superior.
- Red Hat Enterprise Linux 9 o superior.
- Ubuntu Server 20.04 o superior.
- Ubuntu Server 22.04 o superior.
- Rocky Linux / AlmaLinux.
- SUSE Linux Enterprise Server, si se dispone de paquetes equivalentes.

---

### 4.2 Registry privado destino

El registry privado debe cumplir con lo siguiente:

- Estar desplegado y accesible desde la máquina de descarga o bastion.
- Estar accesible desde el cluster OpenShift destino.
- Soportar Docker Registry HTTP API V2.
- Contar con usuario y contraseña de escritura.
- Tener espacio suficiente para almacenar las imágenes.
- Contar con una ruta o namespace definido para IBM Concert.

Ejemplos de registries privados:

- Red Hat Quay.
- Harbor.
- JFrog Artifactory.
- Docker Registry.
- Otros registros OCI compatibles.

Ejemplo de ruta destino:

```text
registry.midominio.local:5000/concert
```

> Nota: Para la carga de imágenes, IBM requiere indicar el registry privado con la ruta o namespace destino. Por ejemplo: `registry.midominio.local:5000/concert`.

---

### 4.3 Cluster OpenShift destino

Aunque este documento se enfoca en la descarga y copia de imágenes, es recomendable validar que el cluster destino tenga:

- Red Hat OpenShift Container Platform 4.12 o superior.
- Acceso al registry privado.
- Storage Class configurada.
- Permisos de administrador para la instalación posterior.
- Cliente `oc` disponible en el host desde donde se instalará.
- Helm 3.x disponible para la instalación posterior.

Validar acceso al registry privado desde OpenShift será parte del proceso de instalación, pero se recomienda confirmarlo antes de iniciar el despliegue final.

---

## 5. Instalación de paquetes base

### 5.1 Red Hat Enterprise Linux / Rocky Linux / AlmaLinux

```bash
sudo dnf install -y curl wget tar gzip jq podman
```

Validar Podman:

```bash
podman --version
```

---

### 5.2 Ubuntu Server

```bash
sudo apt-get update
sudo apt-get install -y curl wget tar gzip jq ca-certificates gnupg
```

Instalar Docker:

```bash
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
```

Validar Docker:

```bash
docker --version
```

Agregar el usuario actual al grupo Docker si corresponde:

```bash
sudo usermod -aG docker $USER
```

Cerrar sesión y volver a ingresar para aplicar el cambio de grupo.

---

### 5.3 SUSE Linux Enterprise Server

```bash
sudo zypper refresh
sudo zypper install -y curl wget tar gzip jq podman
```

Validar Podman:

```bash
podman --version
```

---

## 6. Variables de trabajo

Definir las variables principales del procedimiento.

```bash
export WORKDIR=$HOME/CONCERT-AIRGAP-OCP
export CONCERT_VERSION=2.3.1
export CONCERT_PACKAGE=ibm-concert-x86.tar.gz
export CONCERT_PACKAGE_URL=https://github.com/IBM/Concert/releases/download/v${CONCERT_VERSION}/${CONCERT_PACKAGE}
```

Crear directorio de trabajo:

```bash
mkdir -p ${WORKDIR}
cd ${WORKDIR}
```

Validar espacio disponible:

```bash
df -h ${WORKDIR}
```

Validar arquitectura del servidor:

```bash
uname -m
```

Para este procedimiento se considera arquitectura:

```text
x86_64
```

---

## 7. Configuración del runtime de contenedores

El proceso de descarga y carga de imágenes utiliza Docker o Podman. Se recomienda validar que el runtime tenga suficiente espacio disponible.

### 7.1 Validar Docker

```bash
docker info | grep -E "Docker Root Dir|Storage Driver"
```

Validar espacio del filesystem usado por Docker:

```bash
df -h /var/lib/docker
```

---

### 7.2 Validar Podman

```bash
podman info | grep -E "graphRoot|runRoot"
```

Validar espacio del filesystem usado por Podman:

```bash
df -h /var/lib/containers
```

---

## 8. Descargar paquete base de IBM Concert

Descargar el paquete de IBM Concert:

```bash
cd ${WORKDIR}

wget ${CONCERT_PACKAGE_URL}
```

Validar descarga:

```bash
ls -lh ${CONCERT_PACKAGE}
```

Extraer el paquete:

```bash
umask 0022
tar -xzf ${CONCERT_PACKAGE}
```

Ingresar al directorio extraído:

```bash
cd ${WORKDIR}/ibm-concert
```

Validar contenido:

```bash
ls -la
```

---

## 9. Preparar credenciales

### 9.1 IBM Entitlement Key

Solicitar la IBM Entitlement Key sin dejarla visible en pantalla:

```bash
read -s -p "IBM Entitlement Key: " REG_PASS
export REG_PASS
echo
```

Validar que la variable fue cargada:

```bash
test -n "$REG_PASS" && echo "REG_PASS configurado"
```

---

### 9.2 Registry privado destino

Definir el registry privado destino.

Ejemplo:

```bash
export TGT_REGISTRY=registry.midominio.local:5000/concert
export TGT_REGISTRY_USER=regadmin
```

Solicitar password del registry privado sin dejarlo visible en pantalla:

```bash
read -s -p "Password del registry privado: " TGT_REGISTRY_PASSWORD
export TGT_REGISTRY_PASSWORD
echo
```

Validar variables:

```bash
echo "Registro destino: ${TGT_REGISTRY}"
echo "Usuario destino : ${TGT_REGISTRY_USER}"
```

> Nota: No almacenar claves en archivos planos, scripts versionados ni repositorios Git.

---

## 10. Login manual a registries

Antes de ejecutar el copia de imágenes, se recomienda validar autenticación contra ambos registries.

### 10.1 Login a IBM Container Registry

Con Docker:

```bash
echo "${REG_PASS}" | docker login cp.icr.io -u cp --password-stdin
```

Con Podman:

```bash
echo "${REG_PASS}" | podman login cp.icr.io -u cp --password-stdin
```

---

### 10.2 Login al registry privado

Con Docker:

```bash
echo "${TGT_REGISTRY_PASSWORD}" | docker login $(echo ${TGT_REGISTRY} | cut -d '/' -f 1) -u "${TGT_REGISTRY_USER}" --password-stdin
```

Con Podman:

```bash
echo "${TGT_REGISTRY_PASSWORD}" | podman login $(echo ${TGT_REGISTRY} | cut -d '/' -f 1) -u "${TGT_REGISTRY_USER}" --password-stdin
```

---

## 11. Ejecutar copia de imágenes de imágenes

Ejecutar el script `manage-images` con la opción `--mirror`:

```bash
cd ${WORKDIR}/ibm-concert

./bin/manage-images --mirror \
  --tgt-registry=${TGT_REGISTRY} \
  --tgt-registry-user=${TGT_REGISTRY_USER} \
  --tgt-registry-password=${TGT_REGISTRY_PASSWORD}
```

Este proceso descarga las imágenes requeridas desde IBM Container Registry y las carga en el registry privado definido en `TGT_REGISTRY`.

El tiempo de ejecución dependerá de:

- Velocidad de internet.
- Latencia hacia IBM Container Registry.
- Velocidad de escritura del registry privado.
- Cantidad de productos seleccionados para cargar.
- Capacidad del runtime local de contenedores.

---

## 12. Validar resultado del copia de imágenes

Al finalizar el proceso, validar que no existan errores en la salida del comando.

También se recomienda revisar el registry privado desde su consola web o mediante sus herramientas de administración.

Ejemplo de validación con Docker o Podman:

```bash
docker search ${TGT_REGISTRY} || true
```

```bash
podman search ${TGT_REGISTRY} || true
```

> Nota: Algunos registries privados no permiten `search` o requieren configuración adicional. En ese caso, validar desde la interfaz del registro o mediante su API.

---

## 13. Relación entre carga de imágenes y params.ini de instalación

Durante la instalación en OpenShift air-gapped, los parámetros del archivo `params.ini` deben apuntar al registry privado donde fueron cargadas las imágenes.

Si el copia de imágenes se realizó hacia:

```text
registry.midominio.local:5000/concert
```

En la instalación se debe usar una configuración similar:

```ini
REG_USER=regadmin
IMAGE_REGISTRY_PREFIX=registry.midominio.local:5000

HUB_IMAGE_REGISTRY_SUFFIX=/concert
CONCERT_IMAGE_REGISTRY_SUFFIX=/concert
WORKFLOWS_IMAGE_REGISTRY_SUFFIX=/concert
DATAAPPS_IMAGE_REGISTRY_SUFFIX=/concert
```

La lógica de rutas es la siguiente:

| Elemento | Ejemplo |
|---|---|
| Registry privado con namespace usado en la carga de imágenes | `registry.midominio.local:5000/concert` |
| `IMAGE_REGISTRY_PREFIX` | `registry.midominio.local:5000` |
| `*_IMAGE_REGISTRY_SUFFIX` | `/concert` |

---

## 14. Validaciones recomendadas antes de instalar

Antes de iniciar la instalación en OpenShift, validar:

### 14.1 Registry privado accesible desde bastion

```bash
curl -k https://$(echo ${TGT_REGISTRY} | cut -d '/' -f 1)/v2/
```

Dependiendo del certificado y autenticación, la respuesta puede ser `200`, `401` o similar. Lo importante es que exista conectividad con el endpoint `/v2/`.

---

### 14.2 Registry privado accesible desde OpenShift

Desde un nodo o pod de prueba, validar conectividad hacia el registry privado.

Ejemplo desde una máquina con `oc`:

```bash
oc run registry-test \
  --image=registry.access.redhat.com/ubi9/ubi \
  --restart=Never \
  --command -- sleep 3600
```

Ingresar al pod:

```bash
oc exec -it registry-test -- bash
```

Validar resolución y conectividad:

```bash
curl -k https://registry.midominio.local:5000/v2/
```

Eliminar pod de prueba:

```bash
oc delete pod registry-test
```

> Nota: Si el cluster no puede resolver o alcanzar el registry privado, la instalación fallará con errores como `ImagePullBackOff`.

---

## 15. Consideraciones para certificados del registry privado

Si el registry privado usa certificado autofirmado o una CA interna, validar que OpenShift confíe en dicho certificado antes de instalar.

En caso contrario, los pods pueden fallar al descargar imágenes desde el registry privado.

Validar el certificado desde el bastion:

```bash
openssl s_client -connect registry.midominio.local:5000 -showcerts </dev/null
```

Si corresponde, coordinar la incorporación de la CA al cluster OpenShift antes del despliegue final.

---

## 16. Continuidad con el procedimiento de instalación

Una vez que las imágenes estén cargadas en el registry privado, continuar con el procedimiento de instalación air-gapped sobre OpenShift.

La instalación se debe documentar en un archivo separado, por ejemplo:

```text
instalacion/openshift/Instalacion-Concert-OpenShift-Airgap.md
```

En dicha instalación se deberá:

1. Extraer el paquete de IBM Concert en el host de instalación.
2. Configurar `etc/params.ini`.
3. Definir namespaces.
4. Definir Storage Class.
5. Configurar `IMAGE_REGISTRY_PREFIX` hacia el registry privado.
6. Configurar los sufijos `*_IMAGE_REGISTRY_SUFFIX`.
7. Ejecutar `bin/setup`.
8. Exponer rutas.
9. Validar pods, PVC y acceso web.

---

## 17. Troubleshooting

### 17.1 Error de autenticación contra IBM Container Registry

Validar:

- IBM Entitlement Key vigente.
- Usuario `cp`.
- Acceso a `cp.icr.io`.
- Conectividad HTTPS.

Comando de prueba:

```bash
echo "${REG_PASS}" | docker login cp.icr.io -u cp --password-stdin
```

---

### 17.2 Error de autenticación contra registry privado

Validar:

- Usuario del registry privado.
- Password vigente.
- Permisos de escritura.
- Ruta o namespace existente.
- Certificado del registro.

Comando de prueba:

```bash
echo "${TGT_REGISTRY_PASSWORD}" | docker login $(echo ${TGT_REGISTRY} | cut -d '/' -f 1) -u "${TGT_REGISTRY_USER}" --password-stdin
```

---

### 17.3 Error por falta de espacio

Validar espacio del directorio de trabajo:

```bash
df -h ${WORKDIR}
```

Validar espacio de Docker:

```bash
df -h /var/lib/docker
```

Validar espacio de Podman:

```bash
df -h /var/lib/containers
```

---

### 17.4 Error por certificado no confiable

Validar certificado del registry privado:

```bash
openssl s_client -connect registry.midominio.local:5000 -showcerts </dev/null
```

Si el registro utiliza CA interna, registrar la CA en el host de descarga y en OpenShift según el estándar de la organización.

---

### 17.5 Error ImagePullBackOff durante instalación posterior

Validar desde OpenShift:

```bash
oc get events -A --sort-by=.lastTimestamp | grep -i image
```

Describir el pod afectado:

```bash
oc describe pod <pod_name> -n <namespace>
```

Causas frecuentes:

- `IMAGE_REGISTRY_PREFIX` incorrecto.
- Sufijo `*_IMAGE_REGISTRY_SUFFIX` incorrecto.
- Credenciales del registry inválidas.
- OpenShift no confía en el certificado del registro.
- El cluster no tiene conectividad hacia el registry privado.
- Las imágenes no fueron cargadas completamente.

---

## 18. Limpieza de variables sensibles

Al finalizar el proceso, limpiar variables sensibles:

```bash
unset REG_PASS
unset TGT_REGISTRY_PASSWORD
```

También se recomienda limpiar el historial si se ingresó alguna credencial manualmente:

```bash
history | tail
```

Si corresponde:

```bash
history -d <numero_de_linea>
```

---

## 19. Resumen del procedimiento

```text
1. Preparar máquina de descarga o bastion.
2. Validar conectividad hacia cp.icr.io.
3. Validar conectividad hacia el registry privado.
4. Descargar paquete base de IBM Concert.
5. Extraer paquete.
6. Exportar IBM Entitlement Key.
7. Exportar credenciales del registry privado.
8. Ejecutar manage-images --mirror.
9. Validar imágenes en el registry privado.
10. Continuar con la instalación air-gapped sobre OpenShift.
```

---

## 20. Ubicación sugerida en el repositorio

Ubicación sugerida del documento:

```text
airgap/descarga/openshift/Descarga-Concert-Airgap-OpenShift.md
```

---

## 21. Referencia

Documentación oficial de IBM Concert 2.3.x:

```text
https://www.ibm.com/docs/en/concert/2.3.x?topic=dk-installing-concert-concert-workflows-concert-data-apps-in-air-gapped-environment-kubernetes
```
