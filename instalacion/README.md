# Instalación

Esta carpeta contiene los procedimientos de instalación de IBM Concert según la plataforma objetivo.

El objetivo es mantener separados los escenarios de instalación sobre VM Linux, OpenShift y futuras arquitecturas Power.

---

## Contenido

```text
instalacion/
├── README.md
├── vm-linux-x86/
│   └── instalacion-ibm-concert-vm-rhel9-online.md
├── vm-linux-power/
└── openshift/
    └── instalacion-concert-openshift.md
```

---

## Procedimientos disponibles

| Procedimiento | Plataforma | Arquitectura | Estado | Documento |
|---|---|---|---|---|
| Instalación Concert en VM Linux con internet | Red Hat Enterprise Linux 9 | x86_64 | Disponible | [Ver procedimiento](vm-linux-x86/instalacion-ibm-concert-vm-rhel9-online.md) |
| Instalación Concert en VM Linux air-gapped | Red Hat Enterprise Linux | x86_64 | Pendiente | Pendiente |
| Instalación Concert en VM Linux | Linux | Power / ppc64le | Pendiente | Pendiente |
| Instalación Concert en OpenShift | Cluster OpenShift | x86_64 / ppc64le | Disponible | [Ver procedimiento](openshift/instalacion-concert-openshift.md) |

---

## Alcance

Los procedimientos de esta carpeta cubren:

- Configuración de prerrequisitos.
- Preparación del usuario de instalación.
- Descarga o uso del paquete de instalación.
- Ejecución del instalador.
- Validación posterior del despliegue.
- Troubleshooting específico del proceso de instalación.

---

## Recomendaciones

- Validar CPU, memoria, disco, sistema operativo y arquitectura antes de instalar.
- Ejecutar la instalación con un usuario operativo, no directamente como `root`, salvo que el procedimiento lo requiera.
- No documentar passwords, tokens ni IBM Entitlement Keys.
- Incluir salidas esperadas debajo de los comandos principales.
- Mantener las integraciones en la carpeta `integraciones/`.
- Mantener la desinstalación en la carpeta `desinstalacion/`.
