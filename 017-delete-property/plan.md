# Plan de Implementación: Eliminación de propiedades

**Rama**: no disponible (el setup devolvió `BRANCH` vacío; este checkout no es un repositorio Git) | **Fecha**: 2026-11-05 | **Spec**: [spec.md](spec.md)

**Entrada**: Especificación funcional desde `specs/017-delete-property/spec.md`

## Resumen

Añadir la eliminación permanente de una propiedad mediante `DELETE /api/properties/{id}` y exponer la acción desde el listado y el detalle de Blazor. La API valida el UUID y localiza la propiedad. Si `imageUrl` tiene un valor, el caso de uso solicita eliminar el recurso mediante la integración autorizada que realmente lo administra; si esa operación falla o no se puede verificar que existe una integración autorizada compatible, la propiedad permanece y no se devuelve éxito. Tras eliminar la imagen, se persiste la eliminación de la propiedad. Si la persistencia falla y se confirma que la propiedad sigue existiendo, se intenta restaurar la imagen con la misma integración. Si la restauración falla, se informa el estado parcial: la propiedad permanece disponible sin imagen y no se informa éxito. Esta restauración es compensatoria, no una transacción distribuida ni una garantía de rollback del recurso remoto.

El frontend reutiliza `IPropertiesApi` y un componente reutilizable de confirmación. Solo al confirmar envía la solicitud; mientras está en curso bloquea solicitudes duplicadas. Tras éxito retira el elemento de la vista y muestra confirmación; ante `404` sincroniza la vista; ante un resultado de comunicación incierto vuelve a consultar antes de permitir otro intento.

Las decisiones funcionales resueltas en la spec quedan fijadas aquí: eliminación física de la propiedad (sin papelera ni restauración de la propiedad); intento de eliminar el recurso apuntado por `imageUrl`, incluso si es externo, solo mediante una integración autorizada; si falla el borrado de la imagen, se conserva la propiedad; si falla la persistencia de la propiedad después del borrado de imagen, se intenta restaurar la imagen y, si no se puede, se comunica el estado parcial sin éxito; confirmación mediante componente reutilizable; acción tanto en listado como en detalle. No se incorpora eliminación masiva ni otra funcionalidad.

## Contexto técnico

**Lenguaje/Versión**: C# con .NET 11 (`net11.0`, SDK declarado en `global.json`: `11.0.100-preview.5.26302.115`).

**Dependencias primarias**: ASP.NET Core Minimal APIs, `ISlice`/vertical slices, EF Core, Npgsql, PostgreSQL, FluentValidation, `ProblemDetails`, Blazor Web App con Razor Components y Refit mediante `IHttpClientFactory`.

**Almacenamiento**: PostgreSQL para la propiedad; el recurso de imagen se resuelve mediante el mecanismo ya asociado a `imageUrl`, que debe identificarse en el checkout de aplicación. Documentos previos mencionan almacenamiento local bajo `wwwroot/assets/properties` y referencias `/assets/properties/{fileName}`; son evidencia documental, no confirmación del código desplegado ni de un proveedor externo. Antes de implementar hay que verificar autoridad, soporte de borrado y posibilidad de restauración para cada clase de referencia. No se inventará proveedor, se inferirá una operación remota desde la URL ni se afirmará atomicidad remota. No se prevé cambio de esquema ni migración.

**Pruebas**: Pruebas unitarias según la constitución; validación HTTP/UI según `quickstart.md` cuando haya un checkout ejecutable. El checkout actual no contiene la aplicación; no se fijan rutas ni nombres de proyectos no verificados. Las pruebas deben cubrir también el fallo de persistencia, la restauración exitosa y fallida, y la preservación confirmada de la propiedad.

**Plataforma objetivo**: API ASP.NET Core y frontend Blazor Interactive Server en navegadores compatibles, según el plan de referencia 016.

**Tipo de proyecto**: Solución web existente con backend Minimal API y frontend Blazor.

**Objetivos de rendimiento**: No se fija un umbral de latencia no presente en la spec. Las operaciones de I/O serán asíncronas; la UI impedirá solicitudes duplicadas durante una operación.

**Restricciones**:

- Respetar `204` sin cuerpo, `400` y `404` con `ProblemDetails`; los fallos de almacenamiento/persistencia no se presentan como éxito y usan el formato de error establecido.
- No usar controllers; mantener el patrón `ISlice`, lógica de caso de uso fuera del endpoint/DbContext y logging estructurado.
- La eliminación de la imagen precede a la eliminación de la propiedad para que un fallo al borrar la imagen no retire el registro. Si falla la persistencia, solo intentar restaurar la imagen cuando se confirme que la propiedad sigue persistida; comunicar sin éxito cualquier fallo de restauración y no afirmar rollback remoto.
- No emitir una petición HTTP `DELETE` arbitraria a una URL externa controlada por datos. Para cualquier referencia debe verificarse que existe una integración autorizada y que su destino y operaciones aplican a ese recurso. La falta de soporte o un fallo de borrado conserva la propiedad y se comunica como error; la capacidad existente de restauración debe comprobarse antes de dar por implementable la compensación.
- No añadir paquetes, persistencia, rutas de UI ni comportamiento fuera de la spec.
- Solo unit tests, salvo modificación expresa de la spec que justifique otro tipo.
- Los documentos generados se redactan en español.

**Escala/alcance**: Un caso de uso backend, integración Refit, confirmación reutilizable y acciones en las superficies existentes de listado y detalle, con sus pruebas unitarias. No cambia el modelo persistente.

## Revisión de Constitución

*Puerta: evaluar antes de la Fase 0 y volver a evaluar tras el diseño.*

### Antes de Fase 0

- **Principio I — Solución única y compartida: PASS.** La feature amplía la solución web existente.
- **Principio II — Spec-Driven Development: PASS.** Existe `spec.md`; el plan y sus contratos se derivan de esa fuente. No se crea `tasks.md` en esta fase.
- **Principio III — Arquitectura canónica: PASS.** Backend Minimal API en vertical slice y frontend Razor Components.
- **Principio IV — Stack obligatorio: PASS.** Se mantiene .NET 11, EF Core/Npgsql, Blazor, Refit y CSS propio.
- **Principio V — Dominio y contratos: PASS CON RESTRICCIÓN.** La regla de borrar imagen antes de propiedad queda en el caso de uso, no en UI, endpoint ni DbContext. Se requiere `ProblemDetails`, logging estructurado y pruebas unitarias. La eliminación remota debe ser mediante integración autorizada, nunca un `DELETE` genérico a destinos arbitrarios.
- **Principio VI — Idioma: PASS.** Los artefactos Markdown de esta feature están en español.
- **Principio VII — Gobernanza: PASS CON LIMITACIÓN DE INSPECCIÓN.** Se usa la raíz canónica `specs/` y se respetan los artefactos previos. El checkout actual no es Git y no contiene la aplicación, por lo que no se puede verificar estructura ni integración concreta; no se derivan archivos fuente.
- **Principio VIII — Ingeniería inversa y documentación retrospectiva: NO APLICA.** La feature no se presenta como documentación retrospectiva de un cambio ya implementado.
- **Principio IX — Especificación verificable y alcance: PASS.** Las respuestas HTTP, efectos UI, fallos y criterios de éxito son comprobables; las decisiones funcionales relevantes ya están contestadas.

**Resultado previo**: aprobado para diseño documental. La ausencia del código de aplicación limita la verificación de patrones concretos y se registra como bloqueo técnico de ejecución, no como aclaración funcional pendiente.

## Estructura del proyecto

### Documentación de esta feature

```text
specs/017-delete-property/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── contracts/
    ├── backend-delete-property.openapi.yaml
    └── frontend-property-deletion.contract.yaml
```

`tasks.md` no se genera en `/speckit.plan`; corresponde a la fase posterior `/speckit.tasks`.

**Decisión de estructura**: En implementación, seguir la arquitectura ya prescrita: caso de uso backend en vertical slice `ISlice` con handler y endpoint Minimal API delgado; reutilizar la integración Refit existente y el patrón Razor ya presente para las acciones de listado y detalle, compartiendo el componente de confirmación. No se fijan directorios, nombres de archivo, clases ni proyectos porque el código de aplicación no está en este checkout. La inspección del checkout que contenga la aplicación es un prerrequisito de implementación.

## Fase 0: Investigación

Las decisiones y su evidencia se registran en [research.md](research.md). La investigación documental encontró:

1. La convención backend prescribe Minimal API, `ISlice`, vertical slices, handler y errores centralizados con `Result<T>`/`ProblemDetails`.
2. La convención frontend prescribe Refit tipado con `IHttpClientFactory` y prohíbe `HttpClient` directo en componentes.
3. Las features previas modelan `imageUrl` como relativo a `wwwroot/assets/properties` en almacenamiento local; la spec 017 extiende explícitamente el alcance a referencias externas.
4. No hay código de aplicación en este checkout para verificar el almacenamiento real de imagen, la operación de restauración, los servicios Refit ni los estados de las pantallas.
5. Un `DELETE` externo genérico a partir de `imageUrl` no es una integración autorizada. En implementación se debe verificar, para las referencias realmente admitidas, la disponibilidad y autoridad del mecanismo existente para borrar y restaurar. Si no está disponible, no se inventa proveedor ni se opera sobre destinos arbitrarios.
6. La persistencia de PostgreSQL y el borrado de un recurso no forman una transacción atómica común. Ante fallo de persistencia, la compensación de imagen se intenta solo tras confirmar que la propiedad sigue existiendo. Si restaurar falla, se informa el estado parcial especificado; nunca se promete rollback remoto. Si el resultado persistente no puede determinarse, no se informa éxito ni se afirma un estado que no se haya verificado.

No quedan decisiones funcionales pendientes ni marcadores `NEEDS CLARIFICATION`. La disponibilidad de las integraciones autorizadas de borrado y restauración es un prerrequisito técnico que debe verificarse en el checkout con aplicación, no una invitación a crear un proveedor nuevo.

## Fase 1: Diseño y contratos

Artefactos generados:

1. [data-model.md](data-model.md): actores, propiedad, referencia de imagen, comando/resultados y estados de interacción.
2. [contracts/backend-delete-property.openapi.yaml](contracts/backend-delete-property.openapi.yaml): contrato HTTP público de API, incluyendo errores y ausencia de cuerpo en `204`.
3. [contracts/frontend-property-deletion.contract.yaml](contracts/frontend-property-deletion.contract.yaml): contrato frontend/API y comportamiento de confirmación/sincronización.
4. [quickstart.md](quickstart.md): pasos de validación automatizada y manual, incluidos fallo de persistencia y restauración compensatoria. No se ejecutaron: no hay proyectos de aplicación en este checkout.
5. El bloque SPECKIT de `.github/copilot-instructions.md` ya apunta a `specs/017-delete-property/plan.md`; se dejó sin cambios. No se ejecutó el hook opcional.

## Revalidación de Constitución (post-diseño)

- **Principios I, II, III, IV y VI: PASS.** Los artefactos corresponden a una sola solución, respetan el flujo documental, el stack declarado y el idioma.
- **Principio V: PASS CON PRERREQUISITOS.** Se mantienen Minimal API/vertical slice, lógica fuera del endpoint/UI/DbContext, `ProblemDetails`, logging estructurado y pruebas unitarias. El borrado/restauración de imagen requiere integración existente y autorizada; no se presume capacidad ni atomicidad con PostgreSQL.
- **Principio VII: PASS CON LIMITACIÓN DOCUMENTADA.** No se modifica infraestructura upstream, spec ni agente context. No se coteja contra fuentes de aplicación ausentes.
- **Principio VIII — Ingeniería inversa y documentación retrospectiva: NO APLICA.** La feature no documenta retrospectivamente código existente.
- **Principio IX: PASS.** Los requisitos de `imageUrl`, error de persistencia, estado parcial, confirmación y ubicación de acción quedan trazados a contratos y validaciones observables. La especificación revisada los formula coherentemente.

**Resultado de puerta posterior**: aprobado para la siguiente fase documental. La implementación queda condicionada a inspeccionar el checkout real y verificar la disponibilidad, autoridad y capacidades de borrado/restauración de las integraciones aplicables. No debe prometerse rollback de almacenamiento remoto ni crear integraciones no autorizadas. Ante fallo de persistencia, se verifica que la propiedad permanezca antes de intentar restaurar; si la restauración falla se comunica el estado parcial sin éxito. Si el resultado de persistencia es indeterminado, no se inventa un resultado compensatorio.

## Contraste con `tasks.md` (sin cambios en esta fase)

Las tareas existentes se revisaron únicamente para detectar desalineaciones. T007 menciona fallo de persistencia, pero no distingue restauración exitosa y fallida; T009 describe el borrado y persistencia sin exigir el intento compensatorio ni el estado parcial; T008 verifica autoridad para borrar, pero no la capacidad autorizada de restaurar. No se modificó `tasks.md`: requiere regeneración o actualización posterior antes de implementar, incluyendo pruebas de ambos resultados de restauración y verificación de que la propiedad permanece sin imagen cuando la restauración falla.

## Complejidad y excepciones

No se introducen nuevos proyectos, frameworks, entidades persistentes, migraciones ni servicios paralelos. La integración autorizada para imágenes externas y la capacidad de restauración son verificaciones de disponibilidad sobre infraestructura existente, no elecciones de proveedor ni autorización para crear un cliente HTTP genérico.
