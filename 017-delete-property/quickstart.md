# Guía rápida de validación: Eliminación de propiedades

## Objetivo

Validar el contrato `DELETE /api/properties/{id}`, la regla de borrado de la imagen antes de la propiedad y las interacciones de confirmación desde el listado y el detalle. Para campos del contrato y reglas de estado, consultar [contracts/backend-delete-property.openapi.yaml](contracts/backend-delete-property.openapi.yaml), [contracts/frontend-property-deletion.contract.yaml](contracts/frontend-property-deletion.contract.yaml) y [data-model.md](data-model.md).

## Prerrequisitos

- Checkout que contenga la solución de aplicación; el checkout inspeccionado para esta planificación no contiene el código de aplicación.
- SDK indicado en `global.json` (versión `11.0.100-preview.5.26302.115`).
- Backend y PostgreSQL configurados según la guía operativa vigente del proyecto.
- Un registro de prueba con UUID conocido; preparar variantes con imagen nula y referencias que el mecanismo existente admita.
- Antes de probar almacenamiento, identificar y verificar la integración existente, su autoridad para cada referencia y sus capacidades de borrado/restauración. No usar una URL externa real ni emitir solicitudes arbitrarias.
- El checkout con aplicación y los nombres reales de proyectos deben verificarse antes de ejecutar comandos; esta guía no presume rutas de código fuente concretas.

## Escenario 1: Validación automatizada

Desde la raíz del checkout que contenga la aplicación, localizar los proyectos de prueba reales (no presentes en este checkout), y ejecutar cada proyecto unitario que cubra backend y frontend:

```powershell
Get-ChildItem -Path . -Recurse -Filter *.csproj
dotnet test <ruta-real-del-proyecto-unitario-backend.csproj>
dotnet test <ruta-real-del-proyecto-unitario-frontend.csproj>
```

Sustituir los marcadores por las rutas identificadas; no crear proyectos ni asumir nombres. Si las pruebas están organizadas de otra forma, validar los proyectos unitarios existentes equivalentes de acuerdo con la constitución.

Resultado esperado:

- Pruebas unitarias backend: UUID inválido, propiedad ausente, éxito `204`, imagen nula, fallo al borrar imagen sin eliminación de propiedad, fallo de persistencia con intento de restauración y restauración exitosa o fallida. Confirmar que al fallar restauración la propiedad permanece sin imagen y el resultado no es éxito.
- Pruebas unitarias frontend: cancelar/cerrar sin solicitud, una solicitud durante la confirmación, éxito, `404`, error general y sincronización antes de reintentar tras resultado incierto.
- Sin pruebas de integración nuevas salvo que una spec aprobada las autorice expresamente.

## Escenario 2: Contrato HTTP

Con la API y una base de datos de validación disponibles, enviar solicitudes usando el contrato OpenAPI:

1. `DELETE /api/properties/{uuid-existente}`.
   - Esperado: `204 No Content`, sin cuerpo; la propiedad deja de aparecer en consultas/listados.
2. `DELETE /api/properties/{uuid-inexistente}`.
   - Esperado: `404` y `ProblemDetails`; no cambia ninguna otra propiedad.
3. `DELETE /api/properties/no-es-uuid`.
   - Esperado: `400` y `ProblemDetails`; no cambia datos.
4. Repetir la solicitud sobre un ID ya eliminado.
   - Esperado: `404`, consistente con la regla de concurrencia de la spec.

## Escenario 3: Referencia de imagen y fallos compensatorios

1. Eliminar una propiedad de prueba cuyo `imageUrl` sea nulo.
   - Esperado: el flujo no requiere operación de imagen y termina con `204`.
2. Usar pruebas unitarias con el mecanismo de almacenamiento existente sustituido por un doble de prueba que falle al borrar la imagen.
   - Esperado: error con `ProblemDetails`, sin éxito y con la propiedad todavía disponible.
3. Simular fallo al persistir la eliminación después de borrar la imagen y confirmar que la propiedad aún existe.
   - Esperado: se intenta restaurar la imagen con la integración autorizada existente. Si la restauración funciona, la propiedad y su imagen permanecen y se devuelve un error, no `204`.
4. Simular que la restauración también falla.
   - Esperado: la propiedad permanece disponible sin imagen; el error informa explícitamente ese estado parcial y no reporta éxito.
5. Para referencias locales o externas, verificar antes de cualquier prueba que el mecanismo ya existente está autorizado y opera sobre esa referencia. Si no se puede acreditar, detener esa validación y escalar; no enviar un `DELETE` arbitrario ni ejecutar borrados reales.

## Escenario 4: Confirmación desde el listado

1. Abrir la lista de propiedades y activar la acción Eliminar de una tarjeta.
2. Comprobar que la confirmación identifica la propiedad e indica que el borrado es permanente.
3. Cancelar o cerrar.
   - Esperado: ninguna solicitud enviada y tarjeta sin cambios.
4. Repetir y confirmar.
   - Esperado: una solicitud; al responder `204`, mensaje de éxito y tarjeta retirada. Si era la última tarjeta, se muestra el estado vacío o la paginación existente sin tarjeta obsoleta.
5. Intentar varios clics durante la solicitud.
   - Esperado: como máximo una solicitud en curso.

## Escenario 5: Confirmación desde el detalle y resultados de error

1. Abrir el detalle de una propiedad, activar Eliminar y cancelar.
   - Esperado: no se envía la solicitud.
2. Repetir y confirmar.
   - Esperado: con `204`, mensaje de éxito y salida del detalle o navegación a la vista existente donde la propiedad ya no esté disponible.
3. Forzar `404`.
   - Esperado: aviso de propiedad no disponible y sincronización del detalle/listado.
4. Forzar `400`, `500` y fallo de comunicación.
   - Esperado: mensaje comprensible y ausencia de mensaje de éxito. En resultado de comunicación incierto, sincronizar antes de habilitar un nuevo intento.

## Criterios de validación

- `SC-001` y `SC-003`: éxito solo después de quitar la propiedad y el recurso de imagen existente.
- `SC-002`: `400`/`404` diferenciados, con otras propiedades intactas.
- `FR-007`/`SC-003`: al fallar persistencia después del borrado de imagen, intentar restaurar; si falla, conservar la propiedad sin imagen e informar estado parcial sin éxito. No asumir rollback o atomicidad entre la persistencia y el almacenamiento.
- `SC-004`: cancelar no invoca la operación y la UI bloquea duplicados.
- `SC-005`: éxito y `404` dejan la vista coherente; error no confirmado nunca se presenta como eliminación exitosa.

## Evidencia de esta fase de planificación

- Se ejecutó `.specify/scripts/powershell/setup-plan.ps1 -Json` desde la raíz, con política de ejecución limitada al proceso tras el primer intento bloqueado por la política del entorno. Resultado: `FEATURE_SPEC=specs/017-delete-property/spec.md`, `IMPL_PLAN=specs/017-delete-property/plan.md`, `SPECS_DIR=specs/017-delete-property`, `BRANCH` vacío.
- Se inspeccionaron la spec, constitución, plantilla de plan, plan de referencia 016 e instrucciones de backend, frontend y persistencia. Se revisó que `imageUrl` puede ser nulo o referir un recurso local/externo; la autorización se deriva de la integración existente, no de la URL.
- No se ejecutaron `dotnet test`, build ni escenarios HTTP/UI: no hay proyectos de aplicación en este checkout. La validación queda descrita para el checkout que incluya la solución.
- No se pudo obtener rama ni diff de Git: el directorio de trabajo no es un repositorio Git.
- No se verificó la existencia de una integración de borrado/restauración ni se ejecutaron operaciones contra recursos de imagen; debe verificarse en el checkout con aplicación antes de implementar o validar esa integración.
- `tasks.md` se contrastó solo para detectar desalineaciones y no se modificó; sus tareas deben regenerarse o actualizarse para cubrir el intento de restauración y el caso de fallo de restauración/preservación de la propiedad.
