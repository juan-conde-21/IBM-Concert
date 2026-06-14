# IBM Concert

Repositorio de procedimientos técnicos para la preparación, descarga, instalación, validación, integración y desinstalación de IBM Concert en ambientes conectados y air-gapped.

El objetivo de este repositorio es centralizar guías reutilizables para escenarios de IBM Concert sobre máquinas virtuales Linux, arquitecturas x86 y Power, así como despliegues sobre OpenShift.

---

## 1. Objetivo del repositorio

Este repositorio busca documentar procedimientos operativos para:

- Descargar imágenes y binarios requeridos por IBM Concert desde una máquina con acceso a internet.
- Generar paquetes comprimidos para traslado a ambientes air-gapped.
- Validar el contenido descargado antes de moverlo al servidor destino.
- Documentar procedimientos de instalación según plataforma objetivo.
- Documentar validaciones posteriores a la instalación.
- Documentar procedimientos de desinstalación controlada, considerando respaldo previo y validación posterior.
- Separar claramente las actividades de descarga, transferencia, instalación, validación, integración y desinstalación.

---

## 2. Alcance

El repositorio está orientado a los siguientes escenarios:

| Escenario | Plataforma | Arquitectura | Estado |
|---|---|---|---|
| Descarga airgap para VM Linux | Máquina virtual o servidor Linux con internet | x86_64 | Disponible |
| Descarga airgap para VM Linux | Máquina virtual o servidor Linux con internet | Power / ppc64le | Pendiente |
| Descarga airgap para OpenShift | Bastion o servidor con acceso a internet | x86_64 / ppc64le | Disponible |
| Instalación Concert en VM Linux con internet | Máquina virtual Red Hat Enterprise Linux | x86_64 | Disponible |
| Instalación Concert en VM Linux air-gapped | Servidor destino air-gapped | x86_64 | Pendiente |
| Instalación Concert en VM Linux | Servidor destino Linux | Power / ppc64le | Pendiente |
| Instalación Concert en OpenShift | Cluster OpenShift conectado o air-gapped | x86_64 / ppc64le | Disponible |
| Desinstalación Concert en VM Linux | Máquina virtual Red Hat Enterprise Linux | x86_64 | Disponible |
| Integraciones | watsonx.ai | VM / OpenShift | Disponible |
| Troubleshooting | VM / OpenShift | x86_64 / ppc64le | En construcción |

---

## 3. Estructura del repositorio

```text
IBM-Concert/
├── README.md
├── airgap/
│   ├── descarga/
│   │   ├── vm-linux-x86/
│   │   ├── vm-linux-power/
│   │   └── openshift/
│   └── validacion/
├── instalacion/
│   ├── vm-linux-x86/
│   ├── vm-linux-power/
│   └── openshift/
├── desinstalacion/
│   ├── vm-linux-x86/
│   ├── vm-linux-power/
│   └── openshift/
├── integraciones/
│   └── watsonx-ai/
└── troubleshooting/
```

---

## 4. Descripción de carpetas

### `airgap/`

Contiene los procedimientos relacionados con la preparación de paquetes para ambientes sin salida a internet.

Incluye:

- Descarga de binarios.
- Descarga de imágenes.
- Validación de archivos descargados.
- Preparación de paquetes comprimidos.
- Transferencia hacia servidores destino.
- Carga de imágenes al registry privado.

Esta carpeta no debe mezclar pasos de instalación. Su objetivo es cubrir únicamente la preparación del contenido requerido para ambientes air-gapped.

---

### `instalacion/`

Contiene los procedimientos de instalación de IBM Concert según plataforma objetivo.

Incluye:

- Instalación sobre VM Linux.
- Instalación sobre OpenShift.
- Configuración de prerrequisitos.
- Ejecución del instalador.
- Validación posterior a la instalación.
- Troubleshooting específico del despliegue.

Cuando el procedimiento corresponda a una VM con salida a internet, se debe documentar como instalación conectada y no como procedimiento airgap.

---

### `desinstalacion/`

Contiene los procedimientos para retirar IBM Concert de forma controlada.

Incluye:

- Validación del estado previo.
- Identificación de namespaces, pods, PVC y PV.
- Respaldo previo de datos persistentes.
- Ejecución del desinstalador oficial.
- Validación posterior de limpieza.
- Recomendaciones para reinstalación o limpieza manual.

Esta sección es útil para laboratorios, POC, reinstalaciones y limpieza de ambientes de prueba.

---

### `integraciones/`

Contiene procedimientos de integración de IBM Concert con servicios externos.

Actualmente considera:

- Integración con watsonx.ai.
- Configuración de conectividad.
- Validación de credenciales.
- Pruebas funcionales posteriores.

Las integraciones deben mantenerse separadas de la instalación base, para evitar mezclar dependencias opcionales con el despliegue principal.

---

### `troubleshooting/`

Contiene errores conocidos, validaciones operativas y acciones correctivas.

Puede incluir:

- Errores durante descarga.
- Errores durante instalación.
- Problemas con registry privado.
- Problemas con K3s, Helm, Podman u OpenShift.
- Problemas de conectividad.
- Validaciones posteriores al despliegue.
- Errores durante desinstalación.

---

## 5. Convenciones de documentación

Cada procedimiento debe mantener un estilo técnico, claro y operativo.

La estructura recomendada es:

```text
# Título del procedimiento

## Objetivo
## Alcance
## Prerrequisitos
## Procedimiento
## Validaciones
## Troubleshooting
## Recomendaciones
```

Cuando corresponda, se debe separar claramente:

- Descarga.
- Transferencia.
- Carga de imágenes al registry privado.
- Instalación.
- Validación.
- Integraciones.
- Desinstalación.

---

## 6. Criterios de redacción

Para mantener claridad en los procedimientos:

- Usar el término `registry privado`.
- Usar el término `cargar imágenes al registry privado`.
- Evitar términos ambiguos o poco claros.
- Mantener comandos listos para copiar y ejecutar.
- Explicar la interpretación de las salidas principales.
- Incluir salidas esperadas debajo de los comandos más relevantes.
- No incluir credenciales reales.
- No incluir tokens.
- No incluir passwords.
- No incluir IBM Entitlement Keys.
- No incluir secretos de clientes o ambientes productivos.

---

## 7. Seguridad

Los documentos no deben contener información sensible.

No se debe publicar:

- IBM Entitlement Keys.
- Passwords de usuarios administradores.
- Tokens de acceso.
- Secrets de Kubernetes u OpenShift.
- Credenciales de registry.
- Certificados privados.
- Información interna del cliente.
- IPs privadas sensibles, salvo que sean ejemplos genéricos o ambientes de laboratorio.

Cuando sea necesario documentar una variable sensible, usar valores referenciales:

```bash
export REG_PASS="<IBM_ENTITLEMENT_KEY>"
```

o:

```text
Initial password: generado durante la instalación y no documentado por seguridad.
```

---

## 8. Recomendaciones generales

- Mantener la descarga airgap separada de la instalación.
- Mantener la instalación separada de las integraciones.
- Mantener la desinstalación como un procedimiento independiente.
- Validar siempre los prerrequisitos antes de ejecutar instaladores.
- Validar la salida de cada comando antes de continuar con el siguiente paso.
- Documentar errores encontrados y la acción correctiva aplicada.
- No publicar evidencias completas si contienen datos sensibles.
- Mantener los procedimientos actualizados por versión de IBM Concert.

---

## 9. Estado general

Este repositorio se encuentra en construcción continua.

Los procedimientos se irán actualizando conforme se validen nuevos escenarios de IBM Concert sobre VM Linux, OpenShift, arquitecturas Power y ambientes air-gapped.
