# Tareas: Eliminación de propiedades

**Entrada**: documentos de diseño en `specs/017-delete-property/`

**Prerequisitos**: `plan.md` y `spec.md`; se consultaron también `research.md`, `data-model.md`, `contracts/`, `quickstart.md` y `.specify/memory/constitution.md`.

**Pruebas**: incluir únicamente pruebas unitarias, solicitadas por el plan y permitidas por la constitución. No crear pruebas de integración o de contrato automatizadas sin autorización expresa.

**Rutas y bloqueo del checkout**: el checkout actual contiene los proyectos de backend y frontend, pero no todas las superficies previstas por esta feature. Las rutas `app/backend/...` y `app/frontend/...` son ubicaciones candidatas; cotejarlas con la solución real antes de crear o editar código. No crear proyectos sustitutos ni asumir integraciones inexistentes.

## Fase 1: Setup

**Propósito**: comprobar que está disponible el checkout de la solución existente; no inicializar una aplicación sustituta.

- [X] T001 Verificar la presencia del código y proyectos existentes bajo `app/backend/` y `app/frontend/` y de sus proyectos de pruebas unitarias bajo `app/backend/tests/` y `app/frontend/test/`; si faltan, detener la implementación y solicitar el checkout que contiene la aplicación. Verificado: existen `RealtorApi`, `RealtorApiTests`, `RealtorWeb` y `RealtorWebTests`.

## Fase 2: Fundacional

**Propósito**: cotejar rutas y patrones reales del checkout antes de empezar las historias; este prerrequisito bloquea todo el trabajo de implementación.

- [X] T002 Cotejar en `app/backend/` y `app/frontend/` las rutas candidatas de API, slices, Refit, listado, detalle y proyectos de pruebas citadas en estas tareas con la solución real; identificar sus ubicaciones equivalentes y no crear archivos hasta confirmar el mapeo. **BLOQUEADA**: el frontend solo contiene las páginas iniciales `Home`, `Error` y `NotFound`, sin listado/detalle de propiedades ni cliente Refit para propiedades; el backend contiene el slice de listado, pero no se encontró una integración de almacenamiento de imágenes con capacidad autorizada de borrado y restauración. Se requiere revisar el plan y las tareas antes de crear superficies nuevas, y resolver la integración de imágenes externa sin inventar un proveedor ni enviar solicitudes arbitrarias.

**Punto de control**: no comenzar las historias mientras T002 siga bloqueada por falta de superficies frontend existentes o por falta de una integración de imagen autorizada para borrado y restauración.

## Fase 3: Historia de usuario 1 — Eliminar una propiedad desde el catálogo (Prioridad: P1)

**Objetivo**: permitir que la persona operadora inicie y confirme la eliminación desde listado o detalle; cancelar no cambia datos, la interfaz evita duplicados y comunica el éxito confirmado.

**Prueba independiente**: con el cliente Refit simulado en pruebas unitarias, comprobar que cancelar/cerrar no envía solicitud, confirmar produce como máximo una solicitud en curso y el éxito retira el elemento de la vista y muestra confirmación.

### Pruebas unitarias de la historia 1

- [X] T003 [US1] Añadir pruebas unitarias de cancelar/cerrar, confirmación única, bloqueo de solicitudes duplicadas y resultado satisfactorio en `app/frontend/test/RealtorWeb/PropertyDeletionTests.cs` (cotejar el proyecto y la convención reales en T002).

### Implementación de la historia 1

- [X] T004 [P] [US1] Añadir a la interfaz Refit existente la operación `DELETE /api/properties/{id}`, sin cuerpo de éxito y con los errores del contrato frontend, en `app/frontend/src/RealtorWeb/Services/Api/Properties/IPropertiesApi.cs` (ruta candidata por cotejar en T002).
- [X] T005 [P] [US1] Crear el componente Razor reutilizable de confirmación, que identifica la propiedad, advierte que el borrado es permanente y permite confirmar, cancelar o cerrar, en `app/frontend/src/RealtorWeb/Features/Properties/Shared/PropertyDeleteConfirmation.razor` (ruta candidata por cotejar en T002).
- [X] T006 [P] [US1] Integrar la acción y confirmación de eliminación en el componente existente de listado bajo `app/frontend/src/RealtorWeb/Features/Properties/List/`; evitar navegación accidental de la tarjeta y retirar el elemento tras `204`, manteniendo el comportamiento existente de lista vacía o paginación (cotejar la ubicación real en T002).
- [X] T007 [P] [US1] Integrar la acción y confirmación de eliminación en el componente existente de detalle bajo `app/frontend/src/RealtorWeb/Features/Properties/Detail/`; tras `204`, salir del detalle o retirar la propiedad para que no siga disponible para interacción (cotejar la ubicación real en T002).

**Punto de control**: US1 se puede validar con un cliente simulado; el flujo extremo a extremo requiere también el endpoint de US2.

## Fase 4: Historia de usuario 2 — Mantener el contrato de eliminación consistente (Prioridad: P1)

**Objetivo**: distinguir UUID inválido, propiedad ausente, eliminación completada y errores; borrar la imagen solo mediante una integración existente autorizada y compensar el fallo de persistencia sin afirmar rollback remoto.

**Prueba independiente**: comprobar unitariamente `400` ante UUID inválido, `404` ante propiedad ausente, `204` sin cuerpo ante éxito, continuidad sin `imageUrl`, error al fallar el borrado de imagen y preservación de la propiedad; ante fallo de persistencia, verificar la comprobación de propiedad antes de restaurar, los resultados de restauración exitosa/fallida y la ausencia de compensación insegura cuando el resultado persistido es indeterminado.

### Pruebas unitarias de la historia 2

- [X] T008 [US2] Añadir pruebas unitarias para UUID inválido, propiedad ausente, éxito `204`, imagen nula y fallo al borrar imagen que conserva la propiedad; cubrir también fallo de persistencia con restauración exitosa y fallida, verificar que solo se intenta restaurar tras confirmar que la propiedad sigue persistida, que la restauración fallida deja la propiedad sin imagen e informa estado parcial sin éxito, y que un resultado persistido indeterminado no provoca una afirmación de estado ni restauración insegura, en `app/backend/tests/RealtorApiTests/Features/Properties/DeleteProperty/DeletePropertyTests.cs` (cotejar la ruta y el patrón reales en T002).

### Implementación de la historia 2

- [X] T009 [US2] Inspeccionar la integración de imágenes existente en `app/backend/src/RealtorApi/` y verificar, para las referencias admitidas por `imageUrl`, que está autorizada y permite tanto borrado como restauración; si falta autoridad o capacidad de restauración, detener y escalar antes de implementar operaciones destructivas, sin emitir un `DELETE` arbitrario ni crear/asumir un proveedor.
- [X] T010 [US2] Implementar el caso de uso en el vertical slice candidato `app/backend/src/RealtorApi/Features/Properties/DeleteProperty/`: localizar la propiedad, omitir operaciones de imagen si `imageUrl` es nulo, borrar primero la imagen con la integración verificada y conservar la propiedad si falla; si falla la persistencia, comprobar que la propiedad sigue existiendo antes de intentar restaurar la imagen con esa integración, informar error sin éxito si la restauración funciona, y si falla dejar la propiedad sin imagen e informar explícitamente el estado parcial sin éxito. Si no puede determinarse el resultado persistido, no afirmar si la propiedad permanece o fue eliminada ni restaurar la imagen; no asumir atomicidad ni rollback remoto. Añadir logging estructurado conforme al patrón real (ruta candidata por cotejar en T002).
- [X] T011 [US2] Mapear `DELETE /api/properties/{id}` como Minimal API/`ISlice` en `app/backend/src/RealtorApi/Features/Properties/DeleteProperty/`; validar UUID y traducir éxito a `204` sin cuerpo, ausencia a `404` y errores técnicos a `ProblemDetails`, sin controllers ni lógica de dominio en el endpoint (ruta candidata por cotejar en T002).

**Punto de control**: no declarar implementado el borrado de referencias para las que no se haya verificado la autoridad y capacidad necesarias. No se garantiza rollback del recurso externo.

## Fase 5: Historia de usuario 3 — Mantener la vista coherente ante resultados y errores (Prioridad: P2)

**Objetivo**: sincronizar listado y detalle ante `404` o comunicación incierta, informar errores sin éxito falso y permitir reintentar tras sincronizar cuando el resultado sea incierto.

**Prueba independiente**: con respuestas simuladas en pruebas unitarias, verificar `400`, `404`, `500` y fallo de comunicación, la retirada del estado obsoleto, la sincronización antes de reintentar ante resultado incierto y la ausencia de confirmación de éxito no demostrada.

### Pruebas unitarias de la historia 3

- [X] T012 [US3] Añadir pruebas unitarias de manejo de `400`, `404`, error del servidor y fallo de comunicación; comprobar mensajes, sincronización de estado obsoleto o incierto y ausencia de éxito no confirmado en `app/frontend/test/RealtorWeb/PropertyDeletionRecoveryTests.cs` (cotejar la ruta y el patrón reales en T002).

### Implementación de la historia 3

- [X] T013 [P] [US3] Actualizar el componente existente de listado bajo `app/frontend/src/RealtorWeb/Features/Properties/List/` para informar `404` y sincronizar/retirar la tarjeta obsoleta; mostrar errores recuperables sin éxito falso y refrescar el catálogo antes de habilitar un reintento tras comunicación incierta (cotejar la ubicación real en T002).
- [X] T014 [P] [US3] Actualizar el componente existente de detalle bajo `app/frontend/src/RealtorWeb/Features/Properties/Detail/` para informar `404` y sincronizar el detalle; mostrar errores recuperables sin éxito falso y consultar el estado antes de habilitar un reintento tras comunicación incierta (cotejar la ubicación real en T002).

## Fase 6: Pulido y validación transversal

**Propósito**: completar el artefacto OpenAPI requerido y validar las historias con el checkout ejecutable y escenarios documentados.

- [X] T015 Regenerar OpenAPI v1 ejecutando `support/scripts/generate-openapi-v1.ps1` y comprobar el contrato generado en `app/backend/src/RealtorApi/wwwroot/openapi/v1.json` (cotejar proyecto y rutas reales en T002).
- [X] T016 Ejecutar únicamente las pruebas unitarias de los proyectos backend y frontend identificados en T002, incluyendo las pruebas de eliminación y `OpenApiDriftTests` si forman parte del proyecto unitario existente; resolver los fallos antes de completar la iniciativa en `app/backend/tests/` y `app/frontend/test/`.
- [X] T017 Ejecutar los escenarios HTTP/UI aplicables de `specs/017-delete-property/quickstart.md` en el checkout ejecutable, usar solo recursos de prueba e integraciones de almacenamiento previamente autorizadas, detener y escalar si no se acredita esa autorización, y registrar allí la evidencia de validación.

## Dependencias y orden de ejecución

### Dependencias entre fases

- **Setup (Fase 1)**: T001 no depende de otras tareas y bloquea todo trabajo posterior si falta el checkout de aplicación.
- **Fundacional (Fase 2)**: T002 depende de T001 y bloquea las historias hasta cotejar las rutas y proyectos reales.
- **US1 (Fase 3) y US2 (Fase 4)**: ambas dependen de T002 y son independientes entre sí; US1 se valida aislada con Refit simulado. El flujo completo de US1 necesita el backend de US2.
- **US3 (Fase 5)**: depende de US1 y US2 porque maneja los resultados de la operación en ambas superficies.
- **Pulido (Fase 6)**: T015 depende de US2; T016 depende de las historias cuyas pruebas ejecuta; T017 depende de las historias aplicables y de un checkout ejecutable.

### Dependencias dentro de las historias

- **US1**: ejecutar T003 antes de implementar; después T004 y T005 son independientes. T006 y T007 requieren cliente y confirmación, y afectan superficies distintas.
- **US2**: T008 define los casos unitarios antes de implementar; T009 debe verificar autorización y capacidad de restauración antes de cualquier operación destructiva; T010 depende de T008/T009 y T011 depende de T010.
- **US3**: ejecutar T012 antes de implementar; T013 y T014 requieren US1/US2 y modifican superficies distintas.

### Oportunidades de paralelismo

- Tras T003, T004 (cliente Refit) y T005 (confirmación reutilizable) pueden avanzar en paralelo; luego T006 (listado) y T007 (detalle) pueden avanzar en paralelo.
- US1 y US2 pueden avanzar en paralelo una vez que T002 confirme el checkout; dentro de US2 la verificación T009 es una puerta previa al caso de uso y al borrado.
- Tras T012 y la disponibilidad de US1/US2, T013 (listado) y T014 (detalle) pueden avanzar en paralelo.
- No se identifican otras tareas paralelas con rutas o dependencias suficientemente independientes.

## Estrategia de implementación

### MVP

La **US1 (P1)** es el alcance MVP sugerido y puede probarse de forma independiente con un cliente Refit simulado. Para una entrega funcional de extremo a extremo hacen falta también la **US2 (P1)** y su mecanismo de almacenamiento previamente autorizado. Completar primero T001/T002; detenerse si falta el checkout, y detener/escalar si no se verifica la integración necesaria para borrar y restaurar.

### Entrega incremental

1. Completar Setup y Fundacional; no crear proyectos sustitutos si falta el código existente.
2. Implementar US1 y validar cancelación, confirmación y duplicados con pruebas unitarias.
3. Implementar US2 tras verificar la integración existente; validar errores, persistencia, compensación y estado indeterminado con pruebas unitarias.
4. Implementar US3 para reconciliar los resultados en listado y detalle.
5. Regenerar OpenAPI y ejecutar pruebas unitarias/escenarios de quickstart disponibles.

**Formato**: todas las tareas utilizan casilla, ID secuencial, etiqueta `[P]` solo cuando aplica, `[USn]` solo en fases de historias y una ruta de archivo o ubicación de proyecto concreta, marcada para cotejo cuando no está verificada por los artefactos.
