# Investigación: Eliminación de propiedades

## Contexto investigado

Se revisaron `specs/017-delete-property/spec.md`, la constitución, las instrucciones locales de backend, frontend y persistencia, `global.json`, y los planes/contratos de las features 003, 007, 012 y 016. El código de aplicación no está presente en el checkout inspeccionado; por ello las referencias a patrones se cotejaron con documentación del proyecto y no con implementaciones fuente.

## Decisión 1: Ubicación y límites del caso de uso backend

- **Decisión**: Incorporar el caso de uso como vertical slice de eliminación de propiedades, siguiendo Minimal API + `ISlice` + handler; el endpoint delega el caso de uso y la conversión de errores continúa por el mecanismo centralizado `Result<T>`/`ProblemDetails`. La ubicación concreta se determina al inspeccionar la solución real.
- **Justificación**: Es el patrón obligatorio de `.github/instructions/backend.instructions.md` y coincide con la arquitectura documentada en features previas. La entidad y reglas de negocio no deben residir en el endpoint o `DbContext`.
- **Alternativas consideradas**:
  - Controller MVC: descartado por la constitución y las instrucciones, que exigen Minimal APIs y prohíben controllers.
  - Lógica completa dentro de `Program.cs` o del endpoint: descartada por contravenir la separación de responsabilidades y el patrón de vertical slice.

## Decisión 2: Respuestas y validación del identificador

- **Decisión**: El contrato de `DELETE /api/properties/{id}` usa `204 No Content` sin cuerpo ante éxito, `400 Bad Request` con `ProblemDetails` para UUID inválido, `404 Not Found` con `ProblemDetails` para propiedad inexistente, y error `500` con `ProblemDetails` para fallo técnico de borrado/persistencia.
- **Justificación**: La spec exige resultados distintos para identificador inválido, inexistente y eliminado. Las instrucciones requieren errores esperados como `ProblemDetails`; las features 007/016 documentan el patrón de `400`/`404`.
- **Alternativas consideradas**:
  - `200` con objeto de respuesta: descartada porque FR-002 establece `204` sin contenido.
  - Devolver éxito idempotente para un ID inexistente: descartada porque FR-004 establece `404` y los casos concurrentes lo requieren.
  - Exponer excepciones internas al consumidor: descartado porque el formato público debe ser comprensible y centralizado.

## Decisión 3: Orden del borrado, persistencia y compensación de imagen

- **Decisión**: Cargar la propiedad y su referencia. Si `imageUrl` no es nulo, borrar primero el recurso por medio de la integración existente, autorizada y competente para esa referencia; si el borrado falla, no retirar la propiedad. Después persistir la eliminación física. Si la persistencia falla, comprobar que la propiedad sigue existiendo antes de intentar restaurar la imagen con el mecanismo autorizado. Si la restauración falla, devolver un error sin éxito e informar el estado parcial: propiedad conservada sin imagen. Si el resultado persistente no puede determinarse, no afirmar que la propiedad se conservó ni ejecutar una compensación que pueda dejar un recurso huérfano; registrar/informar el resultado no confirmado según el formato de error existente.
- **Justificación**: Esta secuencia incorpora la aclaración vigente en spec.md y distingue compensación de atomicidad. Un fallo de borrado de imagen conserva la propiedad. La restauración es un intento sobre el recurso, no una reversión transaccional, y solo procede cuando se sabe que la propiedad permanece. Sin `imageUrl` se omiten ambas operaciones de imagen. No se requiere cambio de esquema.
- **Alternativas consideradas**:
  - Eliminar primero el registro y después la imagen: descartada, porque un fallo de imagen dejaría la propiedad retirada, contra FR-007.
  - Suponer que una transacción de base de datos revierte un borrado de archivo/proveedor: descartado; la transacción PostgreSQL no puede revertir un efecto externo.
  - Prometer restauración siempre exitosa: descartado; el requisito contempla explícitamente que falle y exige comunicar el estado parcial.
  - Restaurar tras cualquier excepción de persistencia sin comprobar el estado del registro: descartado, porque el resultado puede ser incierto y podría restaurarse una imagen para una propiedad ya eliminada.
  - Eliminación lógica, papelera o restauración de la propiedad: descartadas expresamente por FR-005; la compensación de imagen no equivale a restaurar la propiedad.

## Decisión 4: Resolución de `imageUrl`, incluidos borrado y restauración

- **Decisión**: Tratar `imageUrl` como referencia, no como permiso para operar sobre su destino. En el checkout con aplicación, identificar la integración de almacenamiento vigente y verificar que su autoridad cubre la referencia concreta, tanto para borrado como para la restauración compensatoria requerida tras un fallo de persistencia. Aplicar las comprobaciones y restricciones de destino existentes. No enviar `DELETE` genérico a URL arbitraria ni introducir proveedor, credenciales, protocolo o flujo nuevo. Si no se acredita una integración autorizada para borrar, conservar la propiedad y comunicar error. Si el borrado se completa pero falta capacidad de restauración y después falla la persistencia, comunicar la condición parcial de acuerdo con la spec, sin afirmar una reversión garantizada.
- **Justificación**: FR-006/FR-007 requieren retirar el recurso local o externo y especifican la respuesta ante fallos; la aclaración precisa el intento de restauración. Documentación previa refiere almacenamiento local `wwwroot/assets/properties` y rutas `/assets/properties/{fileName}`, pero no prueba el estado de la implementación ni establece un protocolo externo universal. La autoridad para borrar/restaurar debe derivarse del mecanismo real administrado, no de la URL aportada por datos.
- **Alternativas consideradas**:
  - Borrar solo rutas locales: no cubre el alcance explícito de la spec.
  - Enviar `DELETE` a cualquier URL de `imageUrl`: descartado; una URL no acredita autorización ni es necesariamente una API de borrado.
  - Inventar o asumir proveedor externo: descartado por no estar evidenciado en el checkout.
  - Ignorar error de borrado y continuar con la propiedad: descartado por FR-007.
- **Riesgo/bloqueo a verificar al implementar**: El checkout actual no contiene la aplicación ni permite identificar la integración, sus permisos o su capacidad de restauración. Verificarlo antes de modificar código. Si no existe una integración autorizada, detener/escalar esa parte sin sustituirla por solicitudes arbitrarias. No prometer que el recurso remoto puede restaurarse o que el resultado de persistencia sea atómico.

## Decisión 5: Integración Refit y estados de interfaz

- **Decisión**: Extender `IPropertiesApi` usando el registro existente de Refit con `IHttpClientFactory`; exponer la acción en el listado y el detalle. Un componente de confirmación reutilizable identifica la propiedad y comunica la permanencia. Cancelar/cerrar no invoca la API. Deshabilitar confirmaciones repetidas durante la solicitud.
- **Justificación**: Cumple FR-008–FR-011 y las instrucciones frontend que prohíben llamadas HTTP directas desde componentes. Los planes 012 y 016 documentan acciones/listas y estados explícitos, sin que eso determine las rutas concretas del checkout ausente.
- **Alternativas consideradas**:
  - Borrar al activar el botón, sin confirmación: descartado por FR-009.
  - Dos diálogos independientes: descartados por la petición explícita de componente reutilizable.
  - Llamada directa a `HttpClient` desde Razor: prohibida por la instrucción frontend.

## Decisión 6: Reconciliación de vista y errores

- **Decisión**: En `204`, retirar la propiedad de la lista o navegar fuera del detalle y mostrar éxito. En `404`, informar que ya no existe y sincronizar la vista. En otros errores, no mostrar éxito y permitir recuperación. Si la comunicación deja resultado incierto, consultar nuevamente el recurso/listado antes de habilitar otro intento.
- **Justificación**: Sigue FR-011–FR-012 y el caso de concurrencia de la spec; mantiene la vista coherente sin presentar como eliminada una operación no confirmada.
- **Alternativas consideradas**:
  - Retirar visualmente el elemento antes de confirmar API: descartada porque puede ocultar un registro cuya eliminación falló.
  - Permitir reintento inmediato tras un timeout: descartada por el requisito de resincronizar antes de un nuevo intento.

## Decisión 7: Contratos y alcance de pruebas

- **Decisión**: Mantener un contrato OpenAPI del endpoint y un contrato YAML de interacción frontend; validar lógica con pruebas unitarias conforme a la constitución, incluyendo persistencia fallida y restauración exitosa/fallida, y seguir `quickstart.md` para comprobaciones HTTP/UI manuales.
- **Justificación**: API y frontend son interfaces externas de esta feature y necesitan contratos comprobables; la constitución permite por defecto pruebas unitarias y las guías previas documentan validación funcional manual.
- **Alternativas consideradas**:
  - Pruebas de integración obligatorias: no se agregan en el plan porque la constitución exige justificación expresa en spec, plan y tasks y la spec no las solicita.
  - Omitir contrato frontend: descartado porque la acción aparece en dos superficies y tiene reglas de confirmación, errores y sincronización.

## Fuentes del proyecto

- `specs/017-delete-property/spec.md`: comportamiento funcional y aclaración más reciente sobre fallo de persistencia, intento de restauración y estado parcial si esta falla.
- `.specify/memory/constitution.md`: arquitectura, stack, contratos, idioma y límites de pruebas.
- `.github/instructions/backend.instructions.md`: Minimal APIs, `ISlice`, vertical slices, errores y logging estructurado.
- `.github/instructions/frontend.instructions.md`: Razor Components y uso de Refit registrado con `IHttpClientFactory`.
- `.github/instructions/database.instructions.md`: límites para persistencia y migraciones; no se deriva cambio de esquema.
- `specs/003-properties-persistence-seeding/research.md`: convención conocida de imágenes bajo `wwwroot/assets/properties`.
- `specs/007-properties-get-by-id/plan.md` y `research.md`: API por UUID, `ProblemDetails` y nullabilidad de imagen.
- `specs/012-blazor-property-detail/contracts/frontend-property-detail.contract.yaml` y `specs/016-blazor-update-property/contracts/frontend-property-update.contract.yaml`: contratos frontend previos.
- `global.json`: versión .NET declarada.

## Estado de aclaraciones

No quedan decisiones funcionales sin resolver ni marcadores `NEEDS CLARIFICATION`. La existencia de un proveedor autorizado para borrar URLs externas y el patrón real de implementación son verificaciones técnicas pendientes porque las fuentes de aplicación no están presentes; no se sustituyen con una decisión funcional nueva.
