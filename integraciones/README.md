# Integraciones

Esta carpeta contiene procedimientos de integración de IBM Concert con servicios externos.

El objetivo es mantener las integraciones separadas de la instalación base, para evitar mezclar dependencias opcionales con el despliegue principal.

---

## Contenido

```text
integraciones/
├── README.md
└── watsonx-ai/
    └── configuracion-watsonxai-concert.md
```

---

## Procedimientos disponibles

| Integración | Plataforma | Estado | Documento |
|---|---|---|---|
| watsonx.ai | VM / OpenShift | Disponible | [Ver procedimiento](watsonx-ai/configuracion-watsonxai-concert.md) |

---

## Alcance

Los procedimientos de esta carpeta cubren:

- Configuración de conectividad.
- Parámetros requeridos para la integración.
- Validación de credenciales.
- Pruebas funcionales posteriores.
- Troubleshooting específico de integración.

---

## Recomendaciones

- Mantener la instalación base separada de las integraciones.
- No incluir API keys, tokens, passwords ni secretos reales.
- Usar variables referenciales para valores sensibles.
- Validar conectividad antes de configurar la integración.
- Documentar salidas esperadas de las validaciones principales.
