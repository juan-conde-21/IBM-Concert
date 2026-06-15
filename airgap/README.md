# Airgap

Esta carpeta contiene los procedimientos relacionados con la preparación de paquetes para ambientes sin salida a internet.

El objetivo es separar la descarga, validación, empaquetado y transferencia de los componentes requeridos por IBM Concert, sin mezclar estos pasos con la instalación final.

---

## Contenido

```text
airgap/
├── README.md
├── descarga/
│   ├── vm-linux-x86/
│   │   └── descarga-concert-airgap-vm-linux-x86.md
│   ├── vm-linux-power/
│   └── openshift/
│       └── descarga-concert-airgap-openshift.md
└── validacion/
```

---

## Procedimientos disponibles

| Procedimiento | Plataforma | Arquitectura | Estado | Documento |
|---|---|---|---|---|
| Descarga airgap para VM Linux | Máquina virtual o servidor Linux con internet | x86_64 | Disponible | [Ver procedimiento](descarga/vm-linux-x86/descarga-concert-airgap-vm-linux-x86.md) |
| Descarga airgap para VM Linux | Máquina virtual o servidor Linux con internet | Power / ppc64le | Pendiente | Pendiente |
| Descarga airgap para OpenShift | Bastion o servidor con acceso a internet | x86_64 / ppc64le | Disponible | [Ver procedimiento](descarga/openshift/descarga-concert-airgap-openshift.md) |

---

## Alcance

Los procedimientos de esta carpeta cubren:

- Descarga de binarios.
- Descarga de imágenes.
- Validación de archivos descargados.
- Preparación de paquetes comprimidos.
- Transferencia hacia servidores destino.
- Carga de imágenes al registry privado.

---

## Recomendaciones

- No incluir credenciales reales, tokens, passwords ni IBM Entitlement Keys.
- Mantener la descarga separada de la instalación.
- Documentar siempre la versión de IBM Concert utilizada.
- Validar el tamaño y checksum de los archivos generados.
- Usar el término `registry privado`.
- Usar el término `cargar imágenes al registry privado`.
