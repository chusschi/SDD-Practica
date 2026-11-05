# Especificación de funcionalidad: Eliminación de propiedades del catálogo

**Rama de funcionalidad**: No creada; no hay un hook `before_specify` habilitado  
**Directorio de funcionalidad**: `specs/017-delete-property`  
**Creado**: 2026-11-05  
**Estado**: Implementada
**Entrada**: Descripción del usuario: "Implementar eliminación de propiedad en el catálogo, `DELETE /api/properties/{id}`, `204/404/400`, consistencia de persistencia, y acción de eliminación confirmada desde frontend Blazor con Refit, actualización de vista, éxito/error. Incluir y resolver en la especificación las decisiones planteadas: tipo de eliminación, manejo del `imageUrl`, estrategia de confirmación y ubicación de acción."

## Clarifications

### Session 2026-11-05

- Q: Si la imagen se elimina correctamente pero falla después la eliminación de la propiedad, ¿qué resultado debe prevalecer? → A: Conservar la propiedad e intentar restaurar la imagen; si falla la restauración, informar explícitamente el estado parcial y no reportar éxito.

## Ciclo de estado del spec *(obligatorio)*

- Estado inicial: **Borrador**
- Transiciones permitidas: **Borrador -> Aprobada -> En implementación -> Implementada**
- Está prohibido cerrar en **Implementada** sin todas las tareas completadas en `tasks.md`.
- Está prohibido cerrar en **Implementada** sin evidencia de validación en `quickstart.md`.

## Trazabilidad de estado *(obligatorio)*

- 2026-11-05 Borrador - Spec creada por `/speckit.specify`.
- 2026-11-05 Borrador - Aclaraciones interactivas: eliminación física; eliminar también el recurso de imagen aunque sea externo; cancelar la eliminación de la propiedad si no se puede eliminar la imagen; confirmación mediante componente reutilizable; acción disponible en listado y detalle.
- 2026-11-05 Aprobada → En implementación - Transición obligatoria al inicio de la implementación; prerequisitos de tareas verificados.

## Escenarios de usuario y pruebas *(obligatorio)*

### Historia de usuario 1 - Eliminar una propiedad desde el catálogo (Prioridad: P1)

Como persona operadora del catálogo, quiero eliminar una propiedad desde el listado o su detalle para retirarla del catálogo de forma deliberada y recibir confirmación visible del resultado.

**Por qué esta prioridad**: Es el flujo principal que permite mantener el catálogo actualizado y evita que una eliminación accidental retire una propiedad.

**Prueba independiente**: Desde el listado y el detalle, iniciar la acción, cancelar y comprobar que la propiedad no cambia; repetir y confirmar, y comprobar que la propiedad desaparece y se informa el éxito.

**Escenarios de aceptación**:

1. **Dado** que una propiedad aparece en el listado o en su detalle, **cuando** la persona activa la acción de eliminar, **entonces** se presenta el componente reutilizable de confirmación, que identifica la propiedad e indica que la eliminación es permanente.
2. **Dado** que se presenta la confirmación, **cuando** la persona elige cancelar o cerrar el componente, **entonces** no se solicita la eliminación y la propiedad permanece en el catálogo.
3. **Dado** que se presenta la confirmación, **cuando** la persona confirma, **entonces** se solicita la eliminación y, al completarse correctamente, se muestra un mensaje de éxito y la propiedad deja de aparecer en la vista del catálogo.
4. **Dado** que una eliminación está en curso, **cuando** la persona intenta confirmarla de nuevo, **entonces** la interfaz impide solicitudes duplicadas.

### Historia de usuario 2 - Mantener el contrato de eliminación consistente (Prioridad: P1)

Como consumidor del catálogo, quiero que la operación de eliminación distinga entre una eliminación exitosa, un identificador inválido y una propiedad inexistente para manejar cada resultado de forma clara.

**Por qué esta prioridad**: El contrato inequívoco permite que el catálogo y otros consumidores reconozcan el estado real de la propiedad.

**Prueba independiente**: Solicitar la eliminación con un identificador existente, uno inexistente y uno con formato inválido; verificar el resultado correspondiente y el estado posterior de los datos.

**Escenarios de aceptación**:

1. **Dado** un identificador válido que corresponde a una propiedad existente, **cuando** se solicita su eliminación, **entonces** la operación devuelve `204 No Content`, sin cuerpo, y la propiedad ya no está disponible en el catálogo.
2. **Dado** un identificador válido que no corresponde a una propiedad, **cuando** se solicita su eliminación, **entonces** la operación devuelve `404 Not Found` con `ProblemDetails` y no altera otros datos.
3. **Dado** un identificador con formato inválido, **cuando** se solicita su eliminación, **entonces** la operación devuelve `400 Bad Request` con `ProblemDetails` y no altera datos.
4. **Dado** que falla la eliminación de la imagen o la persistencia de la propiedad, **cuando** la operación termina, **entonces** no informa éxito; ante fallo de imagen, conserva la propiedad, y ante fallo de persistencia posterior al borrado de imagen, intenta restaurarla e informa si la propiedad queda sin imagen.

### Historia de usuario 3 - Mantener la vista coherente ante resultados y errores (Prioridad: P2)

Como persona operadora del catálogo, quiero que la vista refleje el resultado real de la eliminación y me permita recuperarme de errores sin perder el contexto.

**Por qué esta prioridad**: Evita mostrar propiedades que ya no existen o retirar de la vista una propiedad cuya eliminación no se completó.

**Prueba independiente**: Simular respuestas exitosas, `400`, `404`, `500` y un fallo de comunicación; comprobar mensajes, actualización de la lista y posibilidad de reintento.

**Escenarios de aceptación**:

1. **Dado** que el servicio confirma la eliminación, **cuando** la interfaz recibe `204`, **entonces** actualiza el listado o sale del detalle para que la propiedad eliminada ya no esté disponible para interacción, y presenta una confirmación de éxito.
2. **Dado** que el servicio responde `404`, **cuando** la interfaz procesa el resultado, **entonces** informa que la propiedad ya no está disponible y actualiza la lista para retirar la tarjeta obsoleta.
3. **Dado** que el servicio responde `400`, `500` o no se puede contactar, **cuando** la eliminación no queda confirmada, **entonces** la interfaz informa el error, no presenta éxito y permite recuperarse o reintentar después de sincronizar la vista cuando el resultado sea incierto.
4. **Dado** que la propiedad no tiene `imageUrl`, **cuando** se confirma su eliminación, **entonces** el flujo se completa sin exigir una imagen.

### Casos límite

- Si la última propiedad de una página se elimina, el catálogo actualiza su paginación o presenta el estado vacío existente sin conservar una tarjeta obsoleta.
- Si dos personas intentan eliminar la misma propiedad, la primera eliminación completada devuelve éxito y la siguiente encuentra la propiedad inexistente (`404`).
- Si `imageUrl` es nulo, la eliminación no requiere borrar una imagen.
- Si `imageUrl` apunta a un recurso local o externo, el sistema intenta eliminar también ese recurso.
- Si no se puede eliminar el recurso de imagen, la operación no elimina la propiedad y comunica el error.
- Si se elimina la imagen pero falla la eliminación de la propiedad, el sistema intenta restaurar la imagen; si no puede hacerlo, informa el error y que la propiedad permanece sin imagen.
- Si se pierde la comunicación después de confirmar y no se conoce el resultado, la interfaz vuelve a sincronizar el catálogo antes de ofrecer un nuevo intento que pueda causar confusión.
- Una confirmación cancelada o cerrada no debe producir cambios en la propiedad, su imagen ni la lista.

## Requisitos *(obligatorio)*

### Requisitos funcionales

- **FR-001**: El sistema DEBE ofrecer la operación pública `DELETE /api/properties/{id}` para eliminar una propiedad identificada por un UUID (GUID), conforme al formato canónico existente.
- **FR-002**: Cuando la propiedad exista y la eliminación se complete, la operación DEBE devolver `204 No Content` sin cuerpo.
- **FR-003**: Cuando el identificador tenga formato inválido, la operación DEBE devolver `400 Bad Request` con `ProblemDetails` y sin eliminar datos.
- **FR-004**: Cuando el identificador sea válido pero la propiedad no exista, la operación DEBE devolver `404 Not Found` con `ProblemDetails` y sin afectar otras propiedades.
- **FR-005**: La eliminación DEBE ser permanente: la propiedad retirada no debe seguir disponible en consultas o listados y no se incorpora una capacidad de archivo, restauración de la propiedad o eliminación lógica.
- **FR-006**: Si la propiedad referencia un recurso mediante `imageUrl`, una eliminación exitosa DEBE retirar tanto la propiedad como dicho recurso, incluso si es externo. Si `imageUrl` es nulo, la eliminación DEBE continuar sin imagen.
- **FR-007**: Si no se puede eliminar la propiedad o el recurso de imagen asociado, la operación NO DEBE responder con éxito y DEBE comunicar un error mediante el formato de error establecido por la API. Si no se puede eliminar la imagen, la propiedad DEBE permanecer disponible. Si la imagen ya fue eliminada y falla después la eliminación de la propiedad, el sistema DEBE intentar restaurar la imagen; si la restauración también falla, DEBE informar el error y que la propiedad permanece disponible sin su imagen.
- **FR-008**: El frontend DEBE ofrecer la acción de eliminar desde el listado y el detalle de cada propiedad, sin activar accidentalmente la navegación de la tarjeta.
- **FR-009**: La acción DEBE solicitar confirmación mediante un componente reutilizable que identifique la propiedad, advierta que la eliminación es permanente y permita confirmar o cancelar. Cancelar o cerrar la confirmación NO DEBE iniciar la eliminación.
- **FR-010**: Solo después de la confirmación, el frontend DEBE solicitar la eliminación mediante la integración Refit existente y evitar solicitudes duplicadas mientras una solicitud esté en curso.
- **FR-011**: Al recibir `204`, el frontend DEBE mostrar confirmación de éxito y actualizar el listado o salir del detalle para que la propiedad eliminada ya no esté disponible para interacción.
- **FR-012**: Ante `400`, `404`, un error del servidor o un fallo de comunicación, el frontend DEBE mostrar un mensaje comprensible y no presentar una eliminación no confirmada como exitosa. Ante `404`, DEBE sincronizar la lista o el detalle para retirar el estado obsoleto; ante un resultado de comunicación incierto, DEBE sincronizar la vista antes de permitir un nuevo intento.
- **FR-013**: La acción de frontend DEBE funcionar también para propiedades sin imagen asociada.

### Entidades clave *(si la iniciativa involucra datos)*

- **Propiedad del catálogo**: Registro identificado por UUID que deja de estar disponible de forma permanente al completarse su eliminación.
- **Recurso de imagen de propiedad**: Recurso local o externo señalado mediante `imageUrl`, cuya eliminación acompaña a la de la propiedad.
- **Resultado de eliminación**: Resultado observable de la operación: éxito sin contenido, identificador inválido, propiedad inexistente o fallo de persistencia/comunicación.
- **Confirmación de eliminación**: Decisión explícita de la persona operadora para continuar con la retirada permanente de una propiedad.

## Decisiones funcionales resueltas

1. **Tipo de eliminación**: eliminación física permanente del registro, no eliminación lógica ni archivado. No se contempla restaurar una propiedad eliminada.
2. **Manejo de `imageUrl`**: se elimina también el recurso señalado, incluso si es externo. Si no se puede eliminar la imagen, la eliminación de la propiedad se cancela y se comunica el error. Si `imageUrl` es nulo, no se requiere acción adicional.
3. **Estrategia de confirmación**: componente reutilizable que identifica la propiedad y advierte que la eliminación es permanente; permite confirmar o cancelar. Cancelar no envía la solicitud.
4. **Ubicación de la acción**: disponible desde el listado y desde el detalle de cada propiedad. No se agrega eliminación masiva.
5. **Errores**: un fallo al eliminar la imagen conserva la propiedad. Si falla la eliminación de la propiedad después de borrar la imagen, se intenta restaurar la imagen. Si también falla la restauración, se comunica el estado parcial; en todos los casos el fallo no se presenta como éxito.

## Alcance

**Incluido**:

- Operación de eliminación de una propiedad individual con respuestas `204`, `404` y `400`.
- Eliminación de la propiedad y del recurso señalado por `imageUrl`, cuando exista.
- Confirmación, solicitud, mensajes de resultado y actualización del listado o detalle de la propiedad.

**Fuera de alcance**:

- Eliminación lógica, papelera, archivado, restauración de la propiedad o eliminación masiva.
- Cambios a permisos, autenticación o datos de otras propiedades.

## Criterios de éxito *(obligatorio)*

### Resultados medibles

- **SC-001**: En el 100 % de las pruebas con una propiedad existente, confirmar la eliminación produce `204` sin cuerpo y la propiedad deja de aparecer en consultas y listados.
- **SC-002**: En el 100 % de las pruebas con identificador inválido o propiedad inexistente, el resultado es respectivamente `400` o `404`, sin modificación de propiedades ajenas.
- **SC-003**: En toda eliminación exitosa, la propiedad y el recurso señalado por `imageUrl` (si existe) dejan de estar disponibles. Si no se puede eliminar la imagen, la propiedad permanece disponible. Si falla la persistencia después de borrar la imagen, se intenta restaurarla; si la restauración falla, se informa que la propiedad permanece sin imagen y no se reporta éxito.
- **SC-004**: En toda cancelación, no se envía la solicitud de eliminación; mientras una solicitud está en curso, no se envía otra para la misma propiedad desde la interfaz.
- **SC-005**: Tras un resultado exitoso o un `404` por eliminación concurrente, la interfaz deja de mostrar la propiedad como disponible; ante un error no confirmado, no la presenta como eliminada.

## Supuestos

- El identificador de propiedad canónico es UUID (GUID), de acuerdo con la especificación vigente de consulta por identificador.
- El sistema dispone de acceso operativo para intentar eliminar los recursos referenciados por `imageUrl`; si la eliminación de imagen falla, la propiedad debe permanecer disponible.
- El mecanismo que administra cada imagen permite intentar restaurarla si falla la persistencia de la propiedad tras borrar la imagen; si la restauración no es posible, el sistema comunica el estado parcial sin reportar éxito.
- La interfaz mantiene los permisos existentes y opera mediante la integración Refit solicitada para el frontend.
- La interfaz de listado ya dispone de acciones por tarjeta y de un estado para una lista vacía o paginada.
- El nombre de la operación, la ruta, los estados indicados y la integración Blazor con Refit se incluyen porque forman parte explícita del requerimiento; el diseño técnico interno se definirá en las fases posteriores.
