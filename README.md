# IBM Concert

Repositorio de procedimientos técnicos para la preparación, descarga e instalación de IBM Concert en ambientes conectados y air-gapped.

El objetivo de este repositorio es centralizar guías reutilizables para escenarios de instalación de IBM Concert sobre máquinas virtuales Linux, arquitecturas x86 y Power, así como despliegues sobre OpenShift.

---

## 1. Objetivo del repositorio

Este repositorio busca documentar procedimientos operativos para:

- Descargar imágenes y binarios requeridos por IBM Concert desde una máquina con acceso a internet.
- Generar paquetes comprimidos para traslado a ambientes air-gapped.
- Validar el contenido descargado antes de moverlo al servidor destino.
- Documentar procedimientos de instalación según plataforma objetivo.
- Separar claramente las actividades de descarga, transferencia e instalación.

---

## 2. Alcance

El repositorio está orientado a los siguientes escenarios:

| Escenario | Plataforma | Arquitectura | Estado |
|---|---|---|---|
| Descarga airgap para VM Linux | Máquina virtual o servidor Linux con internet | x86_64 | Disponible |
| Descarga airgap para VM Linux | Máquina virtual o servidor Linux con internet | Power / ppc64le | Pendiente |
| Descarga airgap para OpenShift | Bastion o servidor con acceso a internet | x86_64 / ppc64le | Pendiente |
| Instalación Concert en VM Linux | Servidor destino air-gapped | x86_64 | Pendiente |
| Instalación Concert en OpenShift | Cluster OpenShift air-gapped | x86_64 / ppc64le | Pendiente |

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
├── integraciones/
│   └── watsonx-ai/
└── troubleshooting/
