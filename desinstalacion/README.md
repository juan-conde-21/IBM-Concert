# Desinstalación

Esta carpeta contiene los procedimientos para retirar IBM Concert de forma controlada.

El objetivo es documentar un camino seguro para laboratorios, POC, reinstalaciones o limpieza de ambientes, considerando respaldo previo, ejecución del desinstalador oficial y validación posterior.

---

## Contenido

```text
desinstalacion/
├── README.md
├── vm-linux-x86/
│   └── desinstalacion-ibm-concert-vm-rhel9.md
├── vm-linux-power/
└── openshift/
```

---

## Procedimientos disponibles

| Procedimiento | Plataforma | Arquitectura | Estado | Documento |
|---|---|---|---|---|
| Desinstalación Concert en VM Linux | Red Hat Enterprise Linux 9 | x86_64 | Disponible | [Ver procedimiento](vm-linux-x86/desinstalacion-ibm-concert-vm-rhel9.md) |
| Desinstalación Concert en VM Linux | Linux | Power / ppc64le | Pendiente | Pendiente |
| Desinstalación Concert en OpenShift | Cluster OpenShift | x86_64 / ppc64le | Pendiente | Pendiente |

---

## Alcance

Los procedimientos de esta carpeta cubren:

- Validación del estado previo.
- Identificación de namespaces, pods, PVC y PV.
- Identificación de rutas reales de almacenamiento.
- Respaldo previo de datos persistentes.
- Ejecución del desinstalador oficial.
- Validación posterior de limpieza.
- Troubleshooting de errores durante la desinstalación.

---

## Recomendaciones

- No ejecutar la desinstalación sin validar previamente los recursos desplegados.
- Respaldar los datos persistentes antes de eliminar productos.
- No asumir rutas de almacenamiento; validar siempre los PV y PVC.
- Mantener los backups fuera del directorio de instalación.
- Validar namespaces, pods, PVC, PV y URL al finalizar.
- No documentar credenciales ni secretos.
