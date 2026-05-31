# Configuración de conexión de IBM Concert con watsonx.ai SaaS

## 1. Objetivo

Documentar el procedimiento para configurar la conexión entre **IBM Concert on-premises** y una instancia existente de **IBM watsonx.ai SaaS**.

Esta configuración permite que Concert utilice capacidades de IA generativa soportadas por watsonx.ai para funciones como análisis de vulnerabilidades, evaluación de evidencia de cumplimiento, generación de recomendaciones y asistencia dentro de la interfaz de Concert.

---

## 2. Alcance

Este documento cubre:

- Validación de prerrequisitos en IBM Cloud y watsonx.ai SaaS.
- Creación o validación de una API Key de IBM Cloud.
- Creación o validación de un proyecto en watsonx.ai.
- Asociación de Watson Machine Learning / watsonx.ai Runtime al proyecto.
- Identificación del `Project ID` y del endpoint `WATSONX_API_URL`.
- Configuración de variables para IBM Concert desplegado sobre OpenShift.
- Configuración de variables para IBM Concert desplegado sobre VM Linux.
- Reinicio del servicio requerido en Concert.
- Validaciones posteriores y comandos de troubleshooting.

Este documento no cubre:

- Instalación de IBM watsonx.ai on-premises.
- Instalación de IBM Cloud Pak for Data.
- Alta, compra o administración comercial de la suscripción de watsonx.ai SaaS.
- Configuración avanzada de modelos, tuning o deployment de modelos personalizados.
- Instalación base de IBM Concert.
- Descarga airgap de imágenes.

> Nota: esta configuración corresponde a una actividad posterior o complementaria a la instalación base de IBM Concert.

---

## 3. Consideraciones importantes

IBM Concert on-premises puede conectarse a una instancia existente de **watsonx.ai SaaS** o a una instancia de watsonx.ai on-premises.

Sin embargo, la licencia on-premises de Concert no habilita automáticamente el uso de una instancia SaaS compartida de watsonx.ai provista por IBM. Para usar watsonx.ai SaaS, se debe contar con el acceso, suscripción y permisos correspondientes en IBM Cloud.

Antes de continuar, validar con el responsable de IBM Cloud o del cliente:

- Cuenta IBM Cloud correcta.
- Región donde se encuentra watsonx.ai.
- Proyecto watsonx.ai a utilizar.
- Servicio Watson Machine Learning / watsonx.ai Runtime asociado.
- API Key válida.
- Permisos requeridos.

---

## 4. Flujo general de configuración

```text
IBM Cloud / watsonx.ai SaaS
        │
        ├── Crear o validar API Key
        ├── Crear o validar proyecto watsonx.ai
        ├── Asociar Watson Machine Learning / watsonx.ai Runtime
        ├── Obtener Project ID
        └── Identificar endpoint regional de watsonx.ai
                │
                ▼
IBM Concert on-premises
        │
        ├── Exportar variables WATSONX
        ├── Actualizar secret o archivo de configuración
        ├── Reiniciar servicio py-utils
        └── Validar funcionamiento de capacidades IA
```

---

## 5. Prerrequisitos

### 5.1 Prerrequisitos en IBM Cloud

Se requiere contar con:

- Cuenta IBM Cloud activa.
- Acceso a una instancia de **watsonx.ai SaaS**.
- Permisos para crear o usar una **IBM Cloud API Key**.
- Permisos sobre el proyecto watsonx.ai.
- Permisos para asociar o utilizar **Watson Machine Learning / watsonx.ai Runtime**.
- Región de IBM Cloud definida para watsonx.ai.

Roles sugeridos para la preparación:

- Administrador o editor sobre el proyecto watsonx.ai.
- Permisos suficientes para asociar servicios al proyecto.
- Permisos para generar API Key o uso de una API Key gestionada por el equipo responsable.

---

### 5.2 Prerrequisitos en IBM Concert sobre OpenShift

El cluster o bastion desde donde se realizará la configuración debe contar con:

- Acceso al cluster OpenShift.
- Cliente `oc` instalado.
- Permisos para modificar secrets en el namespace de Concert.
- IBM Concert instalado y operativo.
- Namespace de Concert identificado.
- Deployment `roja-py-utils` disponible en el namespace de Concert.

Validar acceso al cluster:

```bash
oc whoami
```

Validar permisos sobre el namespace de Concert:

```bash
oc auth can-i patch secret -n concert
oc auth can-i rollout restart deployment -n concert
```

Validar que exista el secret de configuración:

```bash
oc get secret app-cfg-secret -n concert
```

Validar que exista el deployment requerido:

```bash
oc get deployment roja-py-utils -n concert
```

> Nota: en este documento se utiliza `concert` como namespace de ejemplo. Ajustar el valor si el despliegue utiliza otro namespace.

---

### 5.3 Prerrequisitos en IBM Concert sobre VM Linux

El servidor VM debe contar con:

- IBM Concert instalado y operativo.
- Acceso administrativo al sistema operativo.
- Ruta de instalación identificada.
- Acceso al archivo `local_config.env`.
- Permisos para reiniciar el servicio `ibm-roja-py-utils`.

Ejemplo de ruta de instalación:

```bash
export INSTALL_DIR=/opt/ibm-concert/ibm-concert
```

Validar estructura esperada:

```bash
ls -la $INSTALL_DIR
ls -la $INSTALL_DIR/ibm-concert-std/etc/
```

---

## 6. Datos requeridos para la configuración

Antes de ejecutar comandos en Concert, se deben tener identificados los siguientes valores:

| Variable | Descripción | Ejemplo |
|---|---|---|
| `WATSONX_API_KEY` | API Key de IBM Cloud con acceso a watsonx.ai | No colocar en archivos del repo |
| `WATSONX_API_PROJECT_ID` | ID del proyecto watsonx.ai | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `WATSONX_API_URL` | Endpoint regional de watsonx.ai | `https://us-south.ml.cloud.ibm.com` |
| `CONCERT_NAMESPACE` | Namespace donde está instalado Concert en OpenShift | `concert` |
| `INSTALL_DIR` | Ruta de instalación de Concert en VM Linux | `/opt/ibm-concert/ibm-concert` |

---

## 7. Crear o validar IBM Cloud API Key

Desde IBM Cloud:

1. Ingresar a IBM Cloud.
2. Ir a **Manage**.
3. Ingresar a **Access (IAM)**.
4. Seleccionar **API keys**.
5. Crear una nueva API Key o validar una existente.
6. Guardar el valor de la API Key en un mecanismo seguro.

Recomendaciones:

- No almacenar la API Key en archivos planos.
- No subir la API Key a GitHub.
- No pegar la API Key en tickets públicos o documentos compartidos.
- Usar una API Key controlada y gestionada por el equipo responsable.
- Rotar la API Key si fue expuesta por error.

---

## 8. Crear o validar proyecto en watsonx.ai

Desde watsonx.ai SaaS:

1. Ingresar a la instancia de watsonx.ai.
2. Validar que se esté usando la cuenta IBM Cloud correcta.
3. Ir a **Projects**.
4. Crear un nuevo proyecto o seleccionar uno existente.
5. Ingresar al proyecto.
6. Ir a la pestaña **Manage** o **General**.
7. Copiar el valor de **Project ID**.

El valor obtenido se utilizará como:

```bash
WATSONX_API_PROJECT_ID=<PROJECT_ID>
```

---

## 9. Asociar Watson Machine Learning / watsonx.ai Runtime

Dentro del proyecto watsonx.ai:

1. Ingresar al proyecto.
2. Ir a **Manage**.
3. Seleccionar **Services & integrations**.
4. Asociar un servicio existente o crear uno nuevo.
5. Seleccionar **Watson Machine Learning** o **watsonx.ai Runtime**, según aparezca en la consola.
6. Validar que el servicio quede asociado al proyecto.

Consideraciones:

- El servicio debe estar disponible en la región correspondiente.
- El proyecto y el servicio asociado deben ser consistentes con la región seleccionada.
- Si no existe un servicio asociado, algunas funciones de watsonx.ai no estarán disponibles para Concert.

---

## 10. Identificar endpoint de watsonx.ai SaaS

El valor `WATSONX_API_URL` depende de la región donde se encuentra el servicio watsonx.ai.

Ejemplo para Dallas / `us-south`:

```bash
export WATSONX_API_URL=https://us-south.ml.cloud.ibm.com
```

Ejemplos comunes de referencia:

```bash
# Dallas
export WATSONX_API_URL=https://us-south.ml.cloud.ibm.com

# Frankfurt
export WATSONX_API_URL=https://eu-de.ml.cloud.ibm.com

# Tokyo
export WATSONX_API_URL=https://jp-tok.ml.cloud.ibm.com

# Sydney
export WATSONX_API_URL=https://au-syd.ml.cloud.ibm.com
```

> Nota: validar siempre la región y disponibilidad de modelos en la documentación oficial de watsonx.ai, porque la disponibilidad puede variar por región y por configuración del servicio.

---

## 11. Configuración en IBM Concert sobre OpenShift

### 11.1 Definir variables

Definir namespace de Concert:

```bash
export CONCERT_NAMESPACE=concert
```

Solicitar API Key sin dejarla visible en pantalla:

```bash
read -s -p "IBM Cloud API Key para watsonx.ai: " WATSONX_API_KEY
echo
```

Definir Project ID:

```bash
export WATSONX_API_PROJECT_ID=<PROJECT_ID>
```

Definir endpoint regional de watsonx.ai:

```bash
export WATSONX_API_URL=https://us-south.ml.cloud.ibm.com
```

Validar variables no sensibles:

```bash
echo $CONCERT_NAMESPACE
echo $WATSONX_API_PROJECT_ID
echo $WATSONX_API_URL
```

No imprimir el valor de `WATSONX_API_KEY`.

---

### 11.2 Validar recursos de Concert

Validar secret:

```bash
oc get secret app-cfg-secret -n $CONCERT_NAMESPACE
```

Validar deployment:

```bash
oc get deployment roja-py-utils -n $CONCERT_NAMESPACE
```

Crear backup del secret antes de modificarlo:

```bash
oc get secret app-cfg-secret -n $CONCERT_NAMESPACE -o yaml \
  > app-cfg-secret-backup-$(date +%Y%m%d%H%M%S).yaml
```

> Importante: el backup del secret contiene valores sensibles en base64. Guardarlo en una ubicación segura y no subirlo al repositorio.

---

### 11.3 Actualizar secret de Concert

Actualizar el secret `app-cfg-secret` con los valores de watsonx.ai SaaS:

```bash
oc patch secret app-cfg-secret -n "$CONCERT_NAMESPACE" --type=merge -p "$(cat <<'EOF'
{
  "data": {
    "WATSONX_API_KEY": "$(printf "%s" "$WATSONX_API_KEY" | base64 | tr -d '\n')",
    "WATSONX_API_PROJECT_ID": "$(printf "%s" "$WATSONX_API_PROJECT_ID" | base64 | tr -d '\n')",
    "WATSONX_API_URL": "$(printf "%s" "$WATSONX_API_URL" | base64 | tr -d '\n')"
  }
}
EOF
)"
```

> Nota: si el bloque anterior se copia tal cual y no expande las variables por el uso de comillas simples en `EOF`, usar la alternativa siguiente.

Alternativa directa:

```bash
WX_KEY_B64=$(printf "%s" "$WATSONX_API_KEY" | base64 | tr -d '\n')
WX_PROJECT_B64=$(printf "%s" "$WATSONX_API_PROJECT_ID" | base64 | tr -d '\n')
WX_URL_B64=$(printf "%s" "$WATSONX_API_URL" | base64 | tr -d '\n')

oc patch secret app-cfg-secret -n "$CONCERT_NAMESPACE" --type=merge -p "{
  \"data\": {
    \"WATSONX_API_KEY\": \"$WX_KEY_B64\",
    \"WATSONX_API_PROJECT_ID\": \"$WX_PROJECT_B64\",
    \"WATSONX_API_URL\": \"$WX_URL_B64\"
  }
}"
```

Validar que las claves fueron agregadas al secret:

```bash
oc get secret app-cfg-secret -n $CONCERT_NAMESPACE -o jsonpath='{.data.WATSONX_API_PROJECT_ID}' | base64 -d; echo
oc get secret app-cfg-secret -n $CONCERT_NAMESPACE -o jsonpath='{.data.WATSONX_API_URL}' | base64 -d; echo
```

No imprimir el valor de `WATSONX_API_KEY`.

---

### 11.4 Reiniciar servicio requerido

Reiniciar el deployment `roja-py-utils`:

```bash
oc rollout restart deployment/roja-py-utils -n $CONCERT_NAMESPACE
```

Validar estado del rollout:

```bash
oc rollout status deployment/roja-py-utils -n $CONCERT_NAMESPACE
```

Validar pods:

```bash
oc get pods -n $CONCERT_NAMESPACE | grep roja-py-utils
```

---

## 12. Configuración en IBM Concert sobre VM Linux

> Esta sección aplica solo si Concert fue instalado sobre una VM Linux. Si el despliegue está sobre OpenShift, usar la sección anterior.

### 12.1 Definir variables

Definir ruta de instalación:

```bash
export INSTALL_DIR=/opt/ibm-concert/ibm-concert
```

Solicitar API Key:

```bash
read -s -p "IBM Cloud API Key para watsonx.ai: " WATSONX_API_KEY
echo
```

Definir Project ID:

```bash
export WATSONX_API_PROJECT_ID=<PROJECT_ID>
```

Definir endpoint de watsonx.ai:

```bash
export WATSONX_API_URL=https://us-south.ml.cloud.ibm.com
```

---

### 12.2 Respaldar archivo de configuración local

Validar archivo:

```bash
ls -la $INSTALL_DIR/ibm-concert-std/etc/local_config.env
```

Crear backup:

```bash
cp -p $INSTALL_DIR/ibm-concert-std/etc/local_config.env \
  $INSTALL_DIR/ibm-concert-std/etc/local_config.env.bak.$(date +%Y%m%d%H%M%S)
```

---

### 12.3 Agregar variables de watsonx.ai

Agregar configuración:

```bash
cat <<EOF >> $INSTALL_DIR/ibm-concert-std/etc/local_config.env
WATSONX_API_KEY=$WATSONX_API_KEY
WATSONX_API_PROJECT_ID=$WATSONX_API_PROJECT_ID
WATSONX_API_URL=$WATSONX_API_URL
EOF
```

Validar variables no sensibles:

```bash
grep WATSONX_API_PROJECT_ID $INSTALL_DIR/ibm-concert-std/etc/local_config.env
grep WATSONX_API_URL $INSTALL_DIR/ibm-concert-std/etc/local_config.env
```

No imprimir el valor de `WATSONX_API_KEY`.

---

### 12.4 Reiniciar servicio

Reiniciar servicio `py-utils`:

```bash
$INSTALL_DIR/ibm-concert-std/bin/start_service ibm-roja-py-utils
```

Validar estado de los servicios según el mecanismo operativo de la instalación:

```bash
$INSTALL_DIR/ibm-concert-std/bin/status
```

> Nota: si el comando `status` no se encuentra disponible en la versión instalada, validar los scripts existentes dentro de `$INSTALL_DIR/ibm-concert-std/bin/`.

---

## 13. Validaciones posteriores

### 13.1 Validar conectividad hacia watsonx.ai desde el bastion

```bash
curl -I $WATSONX_API_URL
```

Resultado esperado:

```text
HTTP/2 200
```

También puede devolver redirecciones o respuestas controladas por el servicio. Lo importante es validar resolución DNS y conectividad HTTPS.

---

### 13.2 Validar conectividad desde OpenShift

Si el cluster permite ejecutar una imagen de prueba:

```bash
oc run wx-connectivity-test \
  -n $CONCERT_NAMESPACE \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl \
  --command -- curl -I $WATSONX_API_URL
```

Si el ambiente no permite descargar imágenes externas, usar un pod existente que tenga herramientas de red o solicitar la validación al equipo de plataforma.

---

### 13.3 Validar logs del servicio en OpenShift

```bash
oc logs deployment/roja-py-utils -n $CONCERT_NAMESPACE --tail=200
```

Filtrar errores relacionados:

```bash
oc logs deployment/roja-py-utils -n $CONCERT_NAMESPACE --tail=300 \
  | grep -iE "watson|watsonx|wx|api|unauthorized|forbidden|error"
```

---

### 13.4 Validar desde la interfaz de Concert

Desde la consola web de Concert, validar las funciones que utilizan IA generativa, por ejemplo:

- AI Assistant.
- Análisis de vulnerabilidades.
- Recomendaciones de remediación.
- Evaluación de evidencia de cumplimiento.
- Funciones de Workflows que utilicen generación o asistencia con IA.

El alcance exacto de la validación depende de los módulos instalados y de los permisos del usuario dentro de Concert.

---

## 14. Troubleshooting

### 14.1 Error de autenticación contra watsonx.ai

Posibles causas:

- API Key inválida.
- API Key expirada o revocada.
- API Key sin permisos suficientes.
- Cuenta IBM Cloud incorrecta.
- Proyecto watsonx.ai asociado a otra cuenta o región.

Validar:

```bash
echo $WATSONX_API_PROJECT_ID
echo $WATSONX_API_URL
```

No imprimir la API Key.

Si se sospecha exposición o error de clave, generar una nueva API Key y actualizar nuevamente el secret o archivo de configuración.

---

### 14.2 Error por Project ID incorrecto

Validar en watsonx.ai:

- Que el proyecto exista.
- Que el Project ID corresponda al proyecto correcto.
- Que el proyecto tenga asociado Watson Machine Learning / watsonx.ai Runtime.
- Que el usuario o API Key tenga acceso al proyecto.

En OpenShift, validar el valor cargado:

```bash
oc get secret app-cfg-secret -n $CONCERT_NAMESPACE \
  -o jsonpath='{.data.WATSONX_API_PROJECT_ID}' | base64 -d; echo
```

---

### 14.3 Error por endpoint regional incorrecto

Validar que el endpoint corresponda a la región donde está el proyecto y el servicio de watsonx.ai.

Ejemplo:

```bash
curl -I $WATSONX_API_URL
```

Si el proyecto está en otra región, actualizar:

```bash
export WATSONX_API_URL=<ENDPOINT_REGIONAL_CORRECTO>
```

Luego aplicar nuevamente el cambio en Concert.

---

### 14.4 El deployment no toma la nueva configuración

Validar que el secret fue actualizado:

```bash
oc get secret app-cfg-secret -n $CONCERT_NAMESPACE -o yaml | grep WATSONX
```

Reiniciar nuevamente:

```bash
oc rollout restart deployment/roja-py-utils -n $CONCERT_NAMESPACE
oc rollout status deployment/roja-py-utils -n $CONCERT_NAMESPACE
```

Validar que se generó un pod nuevo:

```bash
oc get pods -n $CONCERT_NAMESPACE | grep roja-py-utils
```

---

### 14.5 El cluster no tiene salida a internet hacia watsonx.ai

Validar con el equipo de red o plataforma:

- Salida HTTPS hacia el endpoint regional de watsonx.ai.
- Resolución DNS.
- Proxy corporativo, si aplica.
- Reglas de firewall.
- NetworkPolicy o EgressFirewall en OpenShift.

Comandos de referencia:

```bash
oc get networkpolicy -n $CONCERT_NAMESPACE
oc get egressfirewall -A 2>/dev/null
```

> Nota: `EgressFirewall` depende de la configuración y versión de OpenShift. Si el recurso no existe, validar con el equipo administrador del cluster.

---

## 15. Seguridad y manejo de credenciales

No almacenar valores sensibles en el repositorio.

Evitar subir archivos que contengan:

- `WATSONX_API_KEY`
- Backups del secret `app-cfg-secret`
- Archivos `.env` con credenciales
- Capturas de pantalla con claves
- Salidas completas de `oc get secret -o yaml`

Limpiar variables sensibles al finalizar:

```bash
unset WATSONX_API_KEY
unset WATSONX_API_PROJECT_ID
unset WATSONX_API_URL
```

Si se ejecutaron comandos con valores sensibles en la línea de comandos, revisar y limpiar el historial según las políticas del ambiente.

---

## 16. Resumen del procedimiento OpenShift

```text
1. Validar acceso a IBM Cloud y watsonx.ai SaaS.
2. Crear o validar IBM Cloud API Key.
3. Crear o validar proyecto watsonx.ai.
4. Asociar Watson Machine Learning / watsonx.ai Runtime.
5. Obtener Project ID.
6. Identificar endpoint regional WATSONX_API_URL.
7. Validar namespace y secret de Concert en OpenShift.
8. Actualizar secret app-cfg-secret.
9. Reiniciar deployment roja-py-utils.
10. Validar logs y funciones IA desde Concert.
```

---

## 17. Ubicación sugerida en el repositorio

```text
integraciones/watsonx-ai/Configuracion-WatsonxAI-SaaS-Concert.md
```

---

## 18. Referencias oficiales

- IBM Concert 2.3.x - Implementing IBM watsonx.ai:  
  https://www.ibm.com/docs/en/concert/2.3.x?topic=deployment-implementing-watsonxai-premises

- IBM Concert 2.3.x - Role of generative AI in Concert:  
  https://www.ibm.com/docs/en/concert/2.3.x?topic=overview-role-generative-ai-in-concert

- IBM Cloud IAM - Managing user API keys:  
  https://cloud.ibm.com/docs/iam?interface=ui&topic=iam-userapikey

- IBM watsonx.ai - Creating a project:  
  https://dataplatform.cloud.ibm.com/docs/content/wsj/getting-started/projects.html?context=wx

- IBM watsonx.ai - Adding associated services to a project:  
  https://dataplatform.cloud.ibm.com/docs/content/wsj/getting-started/assoc-services.html

- IBM watsonx.ai - Regional availability of services and features:  
  https://dataplatform.cloud.ibm.com/docs/content/wsj/getting-started/regional-datactr.html?context=wx
