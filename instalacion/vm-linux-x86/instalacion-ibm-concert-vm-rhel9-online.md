# Instalación de IBM Concert 3.0.x en VM Red Hat Enterprise Linux 9 con salida a Internet

## 1. Objetivo

Documentar el procedimiento de instalación de **IBM Concert 3.0.x** en una máquina virtual con **Red Hat Enterprise Linux 9** y salida a Internet, utilizando el modo **quick-start VM** como referencia inicial y un archivo `params.ini` controlado para definir el alcance base de instalación, desactivar capacidades opcionales no requeridas y limitar la instalación inicial de integraciones de IBM Concert Workflows.

El procedimiento considera la instalación de:

- IBM Concert
- IBM Concert Workflows
- IBM Concert Data Apps
- Hub integrado de IBM Concert

La instalación se realiza inicialmente con usuario `root` para preparar el sistema operativo y luego con un usuario operativo dedicado llamado `concertadmin`.

---

## 2. Alcance

Este procedimiento aplica para una instalación de IBM Concert en una VM Linux con acceso a Internet.

Incluye:

- Validación inicial del sistema operativo y arquitectura.
- Creación del usuario operativo `concertadmin`.
- Configuración de permisos requeridos.
- Validación de conectividad hacia GitHub e IBM Container Registry.
- Descarga del paquete `ibm-concert-x86.tar.gz`.
- Instalación de prerrequisitos VM con `manage-prereqs-vm`.
- Instalación de IBM Concert en modo quick-start con ajuste controlado de `params.ini`.
- Configuración controlada de `params.ini` para definir el alcance base de instalación y evitar que Workflows instale todas las integraciones desde el despliegue inicial.
- Validación final de K3s, pods, servicios y acceso web.
- Troubleshooting de los errores encontrados durante la instalación.

No incluye:

- Integración con watsonx.ai.
- Configuración de certificados propios.
- Configuración de bases de datos externas.
- Integraciones posteriores con herramientas externas.
- Definición personalizada avanzada del archivo `integrations.cfg`.
- Procedimientos de instalación en Kubernetes u OpenShift.

---

## 3. Referencias oficiales

- IBM Concert 3.0.x - Deploying on a virtual machine:  
  <https://www.ibm.com/docs/en/concert/3.0.x?topic=deployment-deploying-virtual-machine-vm>

- IBM Concert 3.0.x - Installing using quick-start mode:  
  <https://www.ibm.com/docs/en/concert/3.0.x?topic=vm-installing-using-quick-start-mode>

- IBM Concert GitHub releases:  
  <https://github.com/IBM/Concert/releases>

- IBM Concert 3.0.x - Configuring the `params.ini` file:  
  <https://www.ibm.com/docs/en/concert/3.0.x?topic=deployment-configuring-paramsini-file>

- IBM Concert 3.0.x - Configuring the installation of workflow integrations:  
  <https://www.ibm.com/docs/en/concert/3.0.x?topic=backend-configuring-installation-workflow-integrations>

---

## 4. Prerrequisitos

### 4.1 Sistema operativo

Para este procedimiento se validó una VM con:

```text
Red Hat Enterprise Linux 9.8 (Plow)
Arquitectura: x86_64
Compatibilidad requerida: x86_64-v3
```

Comandos de validación:

```bash
cat /etc/redhat-release
uname -m
hostnamectl
```

Salida esperada:

```text
Red Hat Enterprise Linux release 9.x
x86_64
```

Validación de compatibilidad `x86_64-v3`:

```bash
/lib64/ld-linux-x86-64.so.2 --help 2>/dev/null | grep x86-64-v3
```

Salida esperada:

```text
x86-64-v3
```

En la instalación validada se confirmó:

```text
x86_64-v3: SOPORTADO
```

---

### 4.2 Recursos de la VM

Validar CPU, memoria y disco antes de iniciar.

```bash
nproc
lscpu
free -h
lsblk -f
df -hT
```

Salida esperada referencial:

```text
CPU: 16 cores o superior para ambientes de prueba/referencia
Memoria: 32 GB o superior
Disco: espacio suficiente para instalación, imágenes y almacenamiento local
```

> Nota: si el filesystem raíz tiene poco espacio, se recomienda revisar previamente la estrategia de almacenamiento para runtime/contenedores antes de iniciar la instalación.

---

### 4.3 Herramientas base

Validar que existan herramientas básicas del sistema:

```bash
for cmd in curl wget tar gzip podman git jq openssl; do
  command -v "$cmd" >/dev/null 2>&1 && echo "$cmd: OK" || echo "$cmd: NO INSTALADO"
done
```

Salida esperada:

```text
curl: OK
wget: OK
tar: OK
gzip: OK
podman: OK
git: OK
jq: OK
openssl: OK
```

---

## 5. Preparación inicial como root

Los siguientes pasos se ejecutan con usuario `root`.

### 5.1 Crear usuario operativo

Crear un usuario dedicado para la instalación y operación inicial de Concert.

```bash
id concertadmin >/dev/null 2>&1 || useradd -m -s /bin/bash concertadmin
usermod -aG wheel concertadmin
id concertadmin
getent passwd concertadmin
```

Salida esperada:

```text
uid=1002(concertadmin) gid=1002(concertadmin) groups=1002(concertadmin),10(wheel)
concertadmin:x:1002:1002::/home/concertadmin:/bin/bash
```

---

### 5.2 Habilitar linger para el usuario

IBM Concert usa servicios y componentes que deben mantenerse activos aunque se cierre la sesión SSH del usuario.

```bash
loginctl enable-linger concertadmin
loginctl show-user concertadmin -p Linger
```

Salida esperada:

```text
Linger=yes
```

---

### 5.3 Configurar sudo sin password para la instalación

Durante la instalación se requieren privilegios para configurar prerrequisitos, K3s, permisos y servicios del sistema.

```bash
echo 'concertadmin ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/concertadmin
chmod 440 /etc/sudoers.d/concertadmin
visudo -cf /etc/sudoers.d/concertadmin
```

Salida esperada:

```text
/etc/sudoers.d/concertadmin: parsed OK
```

> Recomendación: este permiso puede mantenerse durante la instalación. Luego de completar y validar el despliegue, evaluar si se requiere restringirlo de acuerdo con las políticas internas del ambiente.

---

## 6. Preparación del usuario `concertadmin`

Cambiar al usuario operativo:

```bash
su - concertadmin
```

Validar usuario:

```bash
whoami
id
sudo -n true && echo "sudo sin password: OK" || echo "sudo requiere password"
sudo whoami
loginctl show-user "$(whoami)" -p Linger
```

Salida esperada:

```text
concertadmin
uid=1002(concertadmin) gid=1002(concertadmin) groups=1002(concertadmin),10(wheel)
sudo sin password: OK
root
Linger=yes
```

---

### 6.1 Configurar variables de entorno del usuario

Agregar variables útiles para K3s y Kubernetes:

```bash
cat >> ~/.bashrc <<'EOF'

# IBM Concert VM environment
export XDG_RUNTIME_DIR="/run/user/$UID"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"
EOF

source ~/.bashrc
```

Validar:

```bash
echo "XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR"
echo "DBUS_SESSION_BUS_ADDRESS=$DBUS_SESSION_BUS_ADDRESS"
echo "KUBECONFIG=$KUBECONFIG"
```

Salida esperada:

```text
XDG_RUNTIME_DIR=/run/user/1002
DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1002/bus
KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

---

### 6.2 Validar NM Cloud Setup en RHEL 9

En RHEL 9 se recomienda validar que `nm-cloud-setup` no quede interfiriendo con la configuración de red utilizada por K3s.

```bash
systemctl is-enabled nm-cloud-setup.service 2>/dev/null || true
systemctl is-enabled nm-cloud-setup.timer 2>/dev/null || true
systemctl is-active nm-cloud-setup.service 2>/dev/null || true
systemctl is-active nm-cloud-setup.timer 2>/dev/null || true
```

Salida obtenida en esta instalación:

```text
inactive
inactive
```

Si el servicio o timer aparece habilitado, deshabilitarlo:

```bash
sudo systemctl disable nm-cloud-setup.service nm-cloud-setup.timer 2>/dev/null || true
```

Validar nuevamente:

```bash
systemctl is-active nm-cloud-setup.service 2>/dev/null || true
systemctl is-active nm-cloud-setup.timer 2>/dev/null || true
```

Salida esperada:

```text
inactive
inactive
```

---

## 7. Validación de conectividad con Internet

### 7.1 Validar resolución DNS

```bash
getent hosts github.com
getent hosts ibm.com
getent hosts cp.icr.io
getent hosts icr.io
```

Salida esperada referencial:

```text
140.82.x.x      github.com
2600:1404:...   ibm.com
169.62.x.x      cp.icr.io
169.63.x.x      icr.io
```

---

### 7.2 Validar salida HTTPS

```bash
curl -k -I --connect-timeout 10 https://www.ibm.com | head -n 5
curl -k -I --connect-timeout 10 https://github.com | head -n 5
```

Salida obtenida en esta instalación:

```text
HTTP/2 303
server: AkamaiGHost
location: https://www.ibm.com/us-en

HTTP/2 200
content-type: text/html; charset=utf-8
```

Interpretación:

- `HTTP/2 303` desde IBM confirma conectividad y redirección válida.
- `HTTP/2 200` desde GitHub confirma acceso HTTPS correcto.

---

### 7.3 Validar GitHub Release e IBM Container Registry

```bash
curl -L -o /dev/null -s -w "HTTP: %{http_code}\nTiempo: %{time_total}s\n" \
  --connect-timeout 15 \
  https://github.com/IBM/Concert/releases/download/v3.0.0/ibm-concert-x86.tar.gz

curl -k -I --connect-timeout 15 https://cp.icr.io | head -n 10
curl -k -I --connect-timeout 15 https://icr.io | head -n 10
```

Salida obtenida en esta instalación:

```text
HTTP: 200
Tiempo: 0.68s

HTTP/2 405
...

HTTP/2 405
...
```

Interpretación:

- `HTTP 200` en GitHub confirma que el paquete puede descargarse.
- `HTTP/2 405` en `cp.icr.io` e `icr.io` no representa falla de conectividad. Confirma que el endpoint responde por HTTPS, aunque el método HTTP usado por `curl -I` no sea aceptado por el registry.

---

## 8. Descarga del paquete IBM Concert

Crear directorio de trabajo:

```bash
mkdir -p ~/concert-install
cd ~/concert-install
```

Descargar el paquete de IBM Concert 3.0.0:

```bash
wget -O ibm-concert-x86.tar.gz \
  https://github.com/IBM/Concert/releases/download/v3.0.0/ibm-concert-x86.tar.gz
```

Salida obtenida en esta instalación:

```text
Saving to: 'ibm-concert-x86.tar.gz'
2026-06-14 12:56:03 (115 MB/s) - 'ibm-concert-x86.tar.gz' saved [22661918/22661918]
-rw-r--r-- 1 concertadmin concertadmin 22M Jun 12 20:40 ibm-concert-x86.tar.gz
```

Validar hash y extraer:

```bash
sha256sum ibm-concert-x86.tar.gz
tar -tzf ibm-concert-x86.tar.gz | head -n 20
tar xfz ibm-concert-x86.tar.gz
ls -lah
```

Salida obtenida en esta instalación:

```text
15d99884ed84c223ef94be5b98b5df3d6ea1fae0cf7b6c9f5b8d66480e53fc73  ibm-concert-x86.tar.gz

ibm-concert/bin/setup
ibm-concert/bin/setup-cdw-on-k8s
ibm-concert/bin/setup-cdw-on-vm
ibm-concert/bin/setup-sc-on-k8s

ibm-concert
ibm-concert-x86.tar.gz
```

---

## 9. Revisión del instalador

Validar que el script `setup` exista y sea ejecutable:

```bash
ls -lah ~/concert-install/ibm-concert/bin/setup
file ~/concert-install/ibm-concert/bin/setup
```

Salida obtenida en esta instalación:

```text
-rwx------ 1 concertadmin concertadmin 35K Jun 12 06:22 ibm-concert/bin/setup
ibm-concert/bin/setup: Bourne-Again shell script, UTF-8 Unicode text executable
```

Revisar ayuda del instalador:

```bash
cd ~/concert-install
./ibm-concert/bin/setup --help
```

Salida relevante:

```text
--quickstart-vm
    Create a default params.ini file for VM quickstart installation
    This will generate etc/params.ini with configuration to install:
      - IBM Concert
      - IBM Data Apps
      - IBM Concert Workflows
    Note: This option is only available for VM installations
    Note: You still need to provide registry credentials and accept license
```

---

## 10. Primer intento de instalación quick-start

Antes de ejecutar la instalación, cargar la IBM Entitlement Key en una variable temporal.

> No colocar la entitlement key en el documento, scripts versionados, capturas públicas ni historial compartido.

```bash
set +o history
read -rsp "Ingrese IBM Entitlement Key: " REG_PASS
echo
export REG_PASS
set -o history
```

Ejecutar instalación:

```bash
cd ~/concert-install/ibm-concert
./bin/setup --quickstart-vm --license_acceptance=y
```

En esta instalación, el primer intento se detuvo por prerrequisitos faltantes:

```text
VALIDATION FAILED - Summary of Issues:

ERRORS:
  1. k3s installation not detected
  2. KUBECONFIG environment variable is not set
  3. helm not detected

RECOMMENDATIONS:
  1. Ensure KUBECONFIG=/etc/rancher/k3s/k3s.yaml env variable is exported

ERROR VM prerequisites validation failed
ERROR Run the following command as root to install prerequisites:
ERROR   /home/concertadmin/concert-install/ibm-concert/bin/manage-prereqs-vm --install --user=concertadmin
ERROR Then re-run the setup script as concertadmin
```

Interpretación:

El instalador validó correctamente sistema operativo y utilitarios base, pero detectó que todavía no existía un runtime K3s funcional ni Helm. La acción correcta es ejecutar el instalador de prerrequisitos VM como `root`.

---

## 11. Instalación de prerrequisitos VM

Cambiar temporalmente a `root`:

```bash
exit
```

Ejecutar el instalador de prerrequisitos recomendado por el propio instalador:

```bash
/home/concertadmin/concert-install/ibm-concert/bin/manage-prereqs-vm --install --user=concertadmin
```

Salida obtenida en esta instalación:

```text
System architecture detected: x86_64 (normalized: amd64)
Platform architecture check passed: x86_64 (amd64)
Configuring RHEL-specific prerequisites...
RHEL-specific prerequisites configured successfully
k3s installation completed successfully
```

Reiniciar la VM:

```bash
systemctl reboot
```

Ingresar nuevamente como `concertadmin`:

```bash
su - concertadmin
```

---

## 12. Validación de K3s, Helm y Kubectl

Validar versiones y estado del nodo:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

k3s --version
helm version
kubectl get nodes
```

En esta instalación, inicialmente apareció el siguiente error para `k3s` y `kubectl`:

```text
-bash: /usr/local/bin/k3s: Permission denied
version.BuildInfo{Version:"v3.20.0", GitCommit:"b2e4314fa0f229a1de7b4c981273f61d69ee5a59", GitTreeState:"clean", GoVersion:"go1.25.6"}
-bash: /usr/local/bin/kubectl: Permission denied
```

Interpretación:

Helm estaba instalado correctamente, pero el usuario `concertadmin` no tenía permiso de ejecución sobre los binarios `k3s` y `kubectl`, o no tenía permisos suficientes sobre el kubeconfig.

Corregir permisos:

```bash
sudo chmod 755 /usr/local /usr/local/bin
sudo chmod 755 /usr/local/bin/k3s /usr/local/bin/kubectl

if command -v setfacl >/dev/null 2>&1; then
  sudo setfacl -m u:concertadmin:r /etc/rancher/k3s/k3s.yaml
  sudo setfacl -m u:concertadmin:x /etc/rancher /etc/rancher/k3s
else
  sudo chmod 644 /etc/rancher/k3s/k3s.yaml
fi
```

Validar nuevamente:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

k3s --version
helm version
kubectl get nodes -o wide
kubectl get ns
```

Salida obtenida en esta instalación:

```text
k3s version v1.33.4+k3s1 (148243c4)
go version go1.24.5

version.BuildInfo{Version:"v3.20.0", GitCommit:"b2e4314fa0f229a1de7b4c981273f61d69ee5a59", GitTreeState:"clean", GoVersion:"go1.25.6"}

NAME   STATUS   ROLES                  AGE     VERSION
vm-1   Ready    control-plane,master   5m26s   v1.33.4+k3s1
```

---

## 13. Reintento de instalación IBM Concert

Cargar nuevamente la IBM Entitlement Key en variable temporal, ya que no debe mantenerse expuesta en sesión:

```bash
set +o history
read -rsp "Ingrese IBM Entitlement Key: " REG_PASS
echo
export REG_PASS
set -o history
```

Validar runtime antes de reinstalar:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

whoami
id
echo "$KUBECONFIG"
ls -lah "$KUBECONFIG"
k3s --version
helm version
kubectl get nodes -o wide
```

Salida esperada:

```text
concertadmin
uid=1002(concertadmin) gid=1002(concertadmin) groups=1002(concertadmin),10(wheel)
/etc/rancher/k3s/k3s.yaml
k3s version v1.33.4+k3s1
vm-1 Ready control-plane,master
```

### 13.1 Preparar `params.ini` base controlado

Para esta instalación se utilizará como base el archivo oficial:

```text
concert-dataapps-workflows-vm-quickstart-params.ini
```

Este archivo permite instalar en una misma VM:

- Hub local de IBM Concert
- IBM Concert
- IBM Concert Data Apps
- IBM Concert Workflows

La recomendación para una instalación inicial, POC o ambiente de laboratorio es mantener los valores predeterminados del sample y modificar únicamente los parámetros necesarios para:

- Definir el FQDN de la VM para Concert Workflows.
- Desactivar opciones adicionales que no se utilizarán en la primera instalación.
- Evitar que Workflows instale todas las integraciones disponibles desde el despliegue inicial.

> Nota: no se recomienda agregar variables adicionales si no serán modificadas. Esto permite mantener el archivo más limpio, fácil de revisar y alineado al comportamiento predeterminado del instalador.

Ubicarse en el directorio de instalación:

```bash
cd ~/concert-install/ibm-concert
```

Crear una copia del `params.ini` actual si ya existe:

```bash
if [ -f etc/params.ini ]; then
  cp etc/params.ini etc/params.ini.bak.$(date +%Y%m%d_%H%M%S)
fi
```

Crear el nuevo `params.ini` desde el sample para instalar Concert, Concert Data Apps y Concert Workflows:

```bash
cp etc/sample-params/concert-dataapps-workflows-vm-quickstart-params.ini etc/params.ini
```

Validar el contenido base del archivo:

```bash
grep -Ev '^\s*(#|$)' etc/params.ini
```

Salida esperada referencial:

```text
INSTALL_VM=true
REG_USER=
IMAGE_REGISTRY_PREFIX=cp.icr.io/cp
ENABLE_CROSS_PRODUCT_INTEGRATION=
HUB_IMAGE_REGISTRY_SUFFIX=/platform-hub
INSTALL_CONCERT=true
CONCERT_IMAGE_REGISTRY_SUFFIX=/concert
INSTALL_DATAAPPS=true
DATAAPPS_IMAGE_REGISTRY_SUFFIX=/concert
INSTALL_WORKFLOWS=true
WORKFLOWS_INSTANCE_ADDRESS=
```

Interpretación:

- `INSTALL_CONCERT=true` instala IBM Concert.
- `INSTALL_DATAAPPS=true` instala IBM Concert Data Apps.
- `INSTALL_WORKFLOWS=true` instala IBM Concert Workflows.
- Si `HUB_URL` y `HUB_ACCESS_KEY` no se definen, el instalador desplegará un Hub local.
- `WORKFLOWS_INSTANCE_ADDRESS` debe completarse con el FQDN de la VM.

---

### 13.2 Configurar FQDN de Concert Workflows

Concert Workflows requiere un FQDN para identificar la instancia de Workflows sobre la VM.

Obtener el FQDN de la VM:

```bash
hostname -f
```

Ejemplo de salida:

```text
vm-1.itz-hacnty.local
```

Validar que el FQDN resuelva correctamente:

```bash
getent hosts $(hostname -f)
ping -c 3 $(hostname -f)
```

Configurar `WORKFLOWS_INSTANCE_ADDRESS` con el FQDN obtenido:

```bash
WORKFLOWS_FQDN=$(hostname -f)

sed -i "s/^WORKFLOWS_INSTANCE_ADDRESS=.*/WORKFLOWS_INSTANCE_ADDRESS=${WORKFLOWS_FQDN}/" etc/params.ini
```

Validar el cambio:

```bash
grep '^WORKFLOWS_INSTANCE_ADDRESS=' etc/params.ini
```

Salida esperada referencial:

```text
WORKFLOWS_INSTANCE_ADDRESS=vm-1.itz-hacnty.local
```

> Importante: el valor de `WORKFLOWS_INSTANCE_ADDRESS` no debe incluir protocolo ni puerto. No colocar `https://` ni `:443`.

---

### 13.3 Desactivar opciones no requeridas para la instalación base

Para una primera instalación se recomienda mantener la configuración base y desactivar opciones adicionales que no serán utilizadas en esta etapa.

En este procedimiento se desactivan explícitamente:

- `ENABLE_CROSS_PRODUCT_INTEGRATION`, porque no se configurará integración inicial con productos como Instana o Turbonomic.
- `WORKFLOWS_ENABLE_FAAS`, porque no se requiere FaaS para validar inicialmente Concert Workflows.
- `WORKFLOWS_ENABLE_ISTIO`, porque no se configurará service mesh/mTLS en esta instalación base.
- `INSTALL_SECURECODER`, porque Secure Coder no forma parte del alcance inicial de esta instalación.

También se define:

- `WORKFLOWS_INSTALL_ALL_INTEGRATIONS=false`, para evitar que Workflows instale todas las integraciones disponibles desde el despliegue inicial.

Aplicar los ajustes:

```bash
set_param() {
  local key="$1"
  local value="$2"

  if grep -q "^${key}=" etc/params.ini; then
    sed -i "s|^${key}=.*|${key}=${value}|" etc/params.ini
  else
    echo "${key}=${value}" >> etc/params.ini
  fi
}

set_param ENABLE_CROSS_PRODUCT_INTEGRATION false
set_param WORKFLOWS_ENABLE_FAAS false
set_param WORKFLOWS_ENABLE_ISTIO false
set_param INSTALL_SECURECODER false
set_param WORKFLOWS_INSTALL_ALL_INTEGRATIONS false
```

Validar los parámetros configurados:

```bash
grep -E 'ENABLE_CROSS_PRODUCT_INTEGRATION|WORKFLOWS_ENABLE_FAAS|WORKFLOWS_ENABLE_ISTIO|INSTALL_SECURECODER|WORKFLOWS_INSTALL_ALL_INTEGRATIONS' etc/params.ini
```

Salida esperada:

```text
ENABLE_CROSS_PRODUCT_INTEGRATION=false
WORKFLOWS_ENABLE_FAAS=false
WORKFLOWS_ENABLE_ISTIO=false
INSTALL_SECURECODER=false
WORKFLOWS_INSTALL_ALL_INTEGRATIONS=false
```

Interpretación:

- Si `WORKFLOWS_INSTALL_ALL_INTEGRATIONS` queda vacío o en `true`, Workflows intentará instalar todas las integraciones disponibles.
- Si `WORKFLOWS_INSTALL_ALL_INTEGRATIONS=false`, Workflows instalará un conjunto reducido de integraciones comunes.
- FaaS, Istio, Secure Coder e integración cross-product pueden evaluarse posteriormente, cuando la instalación base ya se encuentre estable.

> Importante: después de personalizar `etc/params.ini`, no volver a ejecutar el instalador con `--quickstart-vm`, ya que ese modo puede regenerar el archivo con valores predeterminados. Para usar el archivo ajustado, ejecutar `setup` sin `--quickstart-vm`.

---

### 13.4 Validar archivo `params.ini` final

Antes de ejecutar nuevamente el instalador, revisar el archivo final sin comentarios ni líneas vacías:

```bash
grep -Ev '^\s*(#|$)' etc/params.ini
```

Salida esperada referencial:

```text
INSTALL_VM=true
REG_USER=
IMAGE_REGISTRY_PREFIX=cp.icr.io/cp
ENABLE_CROSS_PRODUCT_INTEGRATION=false
HUB_IMAGE_REGISTRY_SUFFIX=/platform-hub
INSTALL_CONCERT=true
CONCERT_IMAGE_REGISTRY_SUFFIX=/concert
INSTALL_DATAAPPS=true
DATAAPPS_IMAGE_REGISTRY_SUFFIX=/concert
INSTALL_WORKFLOWS=true
WORKFLOWS_INSTANCE_ADDRESS=vm-1.itz-hacnty.local
WORKFLOWS_ENABLE_FAAS=false
WORKFLOWS_ENABLE_ISTIO=false
INSTALL_SECURECODER=false
WORKFLOWS_INSTALL_ALL_INTEGRATIONS=false
```

> Nota: el archivo final debe mantener una sola línea por variable. Si se realizan ajustes manuales, validar que no existan valores duplicados.

---

### 13.5 Ejecutar instalación usando el `params.ini` ajustado

Ejecutar nuevamente la instalación utilizando el archivo `params.ini` personalizado:

```bash
cd ~/concert-install/ibm-concert
./bin/setup --license_acceptance=y --registry_password="$REG_PASS"
```

Salida relevante esperada:

```text
Platform architecture check passed: x86_64 (amd64)
RHEL-specific prerequisites validated successfully
k3s installation detected (version: v1.33.4+k3s1)
KUBECONFIG environment variable is set: /etc/rancher/k3s/k3s.yaml
KUBECONFIG file exists and is accessible

[SUCCESS] Setup completed
[SUCCESS] Deployment completed successfully
COMPLETED: Hub - SUCCESS
COMPLETED: Concert - SUCCESS
COMPLETED: Concert Data Apps - SUCCESS
COMPLETED: Concert Workflows - SUCCESS

Products successfully installed: 4

You can access IBM Concert at: https://vm-1.itz-7g0v3f.local:12443

Admin username:   ibmconcert
Initial password: ********

DEPLOYMENT SUCCESSFUL
```

> El password inicial es generado durante la instalación. No debe registrarse en el repositorio ni compartirse en texto plano.

Eliminar la variable sensible de la sesión:

```bash
unset REG_PASS
```

---

## 14. Acceso inicial a IBM Concert

URL generada por el instalador:

```text
https://vm-1.itz-7g0v3f.local:12443
```

Usuario administrador inicial:

```text
ibmconcert
```

Password inicial:

```text
Generado durante la instalación. No documentar en el repositorio.
```

Si la estación desde donde se accede no resuelve el FQDN de la VM, registrar temporalmente el hostname en el archivo `hosts` local.

### 14.1 Ejemplo en Linux/macOS

```bash
sudo vi /etc/hosts
```

Agregar:

```text
<IP_VM> vm-1.itz-7g0v3f.local
```

### 14.2 Ejemplo en Windows

Editar como administrador:

```text
C:\Windows\System32\drivers\etc\hosts
```

Agregar:

```text
<IP_VM> vm-1.itz-7g0v3f.local
```

Validar resolución desde la estación local:

```bash
ping vm-1.itz-7g0v3f.local
```

Acceder desde navegador:

```text
https://vm-1.itz-7g0v3f.local:12443
```

Durante el primer acceso se debe usar:

```text
Usuario: ibmconcert
Password: password inicial generado por el instalador
```

Recomendación:

- Cambiar el password inicial luego del primer acceso.
- Usar un registro DNS formal para accesos recurrentes.
- Evitar depender de entradas locales en `hosts` para ambientes compartidos.

---

## 15. Validación final de instalación

Ejecutar como `concertadmin`:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

k3s --version
helm version
kubectl version --client=true
kubectl get nodes -o wide
kubectl get ns
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get deploy -A
```

Salida relevante obtenida en esta instalación:

```text
k3s version v1.33.4+k3s1 (148243c4)

NAME   STATUS   ROLES                  AGE   VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                              KERNEL-VERSION                 CONTAINER-RUNTIME
vm-1   Ready    control-plane,master   47m   v1.33.4+k3s1   10.0.2.2      <none>        Red Hat Enterprise Linux 9.8 (Plow)   5.14.0-687.15.1.el9_8.x86_64   containerd://2.0.5-k3s2
```

Validar endpoint local:

```bash
curl -k -I --connect-timeout 15 https://localhost:12443 | head -n 20
curl -k -I --connect-timeout 15 https://vm-1.itz-7g0v3f.local:12443 | head -n 20
```

Salida obtenida en esta instalación:

```text
HTTP/1.1 303 See Other
HTTP/1.1 303 See Other
```

Interpretación:

El código `303 See Other` indica que el endpoint HTTPS de IBM Concert está respondiendo y redirigiendo correctamente al flujo de autenticación/aplicación.

---

## 16. Troubleshooting

### 16.1 Error: `k3s installation not detected`, `KUBECONFIG environment variable is not set`, `helm not detected`

Error observado:

```text
VALIDATION FAILED - Summary of Issues:

ERRORS:
  1. k3s installation not detected
  2. KUBECONFIG environment variable is not set
  3. helm not detected
```

Causa probable:

Los prerrequisitos VM no fueron instalados antes de ejecutar `setup`.

Acción correctiva:

```bash
# Ejecutar como root
/home/concertadmin/concert-install/ibm-concert/bin/manage-prereqs-vm --install --user=concertadmin
systemctl reboot
```

Luego ingresar como `concertadmin`, validar K3s/Helm y reintentar el setup.

---

### 16.2 Error: `Permission denied` al ejecutar `k3s` o `kubectl`

Error observado:

```text
-bash: /usr/local/bin/k3s: Permission denied
-bash: /usr/local/bin/kubectl: Permission denied
```

Causa probable:

El usuario `concertadmin` no cuenta con permisos de ejecución sobre los binarios o no tiene acceso al kubeconfig.

Acción correctiva:

```bash
sudo chmod 755 /usr/local /usr/local/bin
sudo chmod 755 /usr/local/bin/k3s /usr/local/bin/kubectl
sudo setfacl -m u:concertadmin:r /etc/rancher/k3s/k3s.yaml
sudo setfacl -m u:concertadmin:x /etc/rancher /etc/rancher/k3s
```

Validar:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
k3s --version
kubectl get nodes
```

Salida esperada:

```text
k3s version v1.33.4+k3s1
vm-1 Ready control-plane,master
```

---

### 16.3 El navegador no resuelve el FQDN de la VM

Síntoma:

```text
No se puede resolver vm-1.itz-7g0v3f.local
```

Acción correctiva:

Registrar temporalmente el FQDN en el archivo `hosts` de la estación local:

```text
<IP_VM> vm-1.itz-7g0v3f.local
```

Luego acceder por navegador:

```text
https://vm-1.itz-7g0v3f.local:12443
```

---

### 16.4 El puerto 12443 no responde desde la estación local

Validar desde la VM:

```bash
curl -k -I https://localhost:12443
```

Validar firewall:

```bash
sudo firewall-cmd --list-all
```

Si se requiere abrir el puerto:

```bash
sudo firewall-cmd --add-port=12443/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

---

### 16.5 Respuesta `HTTP/2 405` al validar `cp.icr.io` o `icr.io`

Salida posible:

```text
HTTP/2 405
```

Interpretación:

No necesariamente representa error. En la validación con `curl -I`, el registry puede responder `405 Method Not Allowed`. Lo importante es que exista respuesta HTTPS del endpoint.

---

### 16.6 Error: `Stage "Install integrations job" timeout` en Concert Workflows

Error observado:

```text
ERROR: Stage "Install integrations job" timeout
ERROR Installation of Concert Workflows failed with exit code: 1
```

Causa probable:

Concert Workflows quedó esperando la finalización del job de instalación de integraciones. En instalaciones VM, si `WORKFLOWS_INSTALL_ALL_INTEGRATIONS` queda vacío o en `true`, el instalador puede intentar cargar todas las integraciones disponibles, lo cual incrementa el tiempo de instalación y el consumo de recursos.

Validaciones recomendadas:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

kubectl get pods -n concert-workflows -o wide
kubectl get jobs -n concert-workflows
kubectl get events -n concert-workflows --sort-by=.lastTimestamp | tail -80
helm status rna-core -n concert-workflows
```

Acción correctiva recomendada para una POC, validación inicial o ambiente de laboratorio:

```bash
cd ~/concert-install/ibm-concert

cp etc/params.ini etc/params.ini.bak.$(date +%Y%m%d_%H%M%S)

set_param() {
  local key="$1"
  local value="$2"

  if grep -q "^${key}=" etc/params.ini; then
    sed -i "s|^${key}=.*|${key}=${value}|" etc/params.ini
  else
    echo "${key}=${value}" >> etc/params.ini
  fi
}

set_param ENABLE_CROSS_PRODUCT_INTEGRATION false
set_param WORKFLOWS_ENABLE_FAAS false
set_param WORKFLOWS_ENABLE_ISTIO false
set_param INSTALL_SECURECODER false
set_param WORKFLOWS_INSTALL_ALL_INTEGRATIONS false

grep -E 'ENABLE_CROSS_PRODUCT_INTEGRATION|WORKFLOWS_ENABLE_FAAS|WORKFLOWS_ENABLE_ISTIO|INSTALL_SECURECODER|WORKFLOWS_INSTALL_ALL_INTEGRATIONS' etc/params.ini
```

Luego reintentar la instalación sin usar `--quickstart-vm`:

```bash
./bin/setup --license_acceptance=y --registry_password="$REG_PASS"
```

Interpretación:

Este ajuste evita instalar todas las integraciones desde el despliegue inicial. Una vez que Workflows quede operativo, se puede evaluar la carga de integraciones adicionales según el caso de uso requerido.

---

## 17. Recomendaciones finales

- No documentar passwords, entitlement keys, tokens ni secretos en el repositorio.
- Cambiar el password inicial del usuario `ibmconcert` luego del primer acceso.
- Usar DNS formal para resolver el FQDN de Concert en lugar de depender del archivo `hosts`.
- Validar apertura del puerto `12443/tcp` desde la estación de administración.
- Mantener la IBM Entitlement Key solo como variable temporal durante la instalación.
- Para una POC, validación inicial o ambiente de laboratorio, utilizar `params.ini` controlado y definir `WORKFLOWS_INSTALL_ALL_INTEGRATIONS=false` antes del reintento final de instalación.
- Mantener los valores predeterminados del sample cuando no se requiera modificarlos y desactivar explícitamente solo las capacidades opcionales que no formen parte del alcance inicial, como FaaS, Istio, Secure Coder o integración cross-product.
- Validar periódicamente el estado del nodo K3s y de los pods:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes
kubectl get pods -A
kubectl get svc -A
```

- Conservar una copia del paquete descargado y el hash validado si el entorno requiere trazabilidad de instalación.
- Para configuración de certificados propios, integraciones externas o watsonx.ai, documentar procedimientos separados.

---

## 18. Resumen de instalación validada

```text
Producto: IBM Concert
Versión validada: v3.0.0
Sistema operativo: Red Hat Enterprise Linux 9.8
Arquitectura: x86_64 / x86_64-v3 soportado
Runtime Kubernetes: K3s v1.33.4+k3s1
Helm: v3.20.0
Usuario operativo: concertadmin
URL de acceso: https://vm-1.itz-7g0v3f.local:12443
Usuario administrador inicial: ibmconcert
Estado final: DEPLOYMENT SUCCESSFUL
```
