# Modelo de datos y estados: Eliminación de propiedades

## Alcance del modelo

La feature no agrega entidades persistentes ni cambia el esquema de PostgreSQL. Describe los datos que ya intervienen, la decisión de borrado y los estados observables del caso de uso y la interfaz.

## Entidades y conceptos

### Propiedad del catálogo

- **Identificador**: `id` — UUID/GUID canónico.
- **Datos relevantes**: `title` para identificarla en la confirmación y `imageUrl` nullable para localizar su recurso de imagen. El resto de campos existentes no cambian.
- **Relación**: puede tener cero o un recurso de imagen referenciado por `imageUrl`.
- **Regla de eliminación**: eliminación física permanente; al completar la operación, la propiedad deja de estar disponible en las consultas y listados.
- **Validaciones**:
  - Un UUID malformado produce `400` sin cambios.
  - Un UUID válido sin propiedad asociada produce `404` sin afectar otras propiedades.

### Recurso de imagen asociado

- **Referencia**: `imageUrl` nullable de la propiedad.
- **Ubicación**: recurso local o URL externa, según la referencia asociada.
- **Relación**: pertenece al flujo de eliminación de una propiedad; no se borra una imagen cuando `imageUrl` es nulo.
- **Regla de eliminación**: cuando hay referencia, el sistema intenta eliminar el recurso mediante el mecanismo autorizado que administra esa ubicación antes de persistir la eliminación de la propiedad.
- **Fallo**: si el recurso no puede eliminarse, la propiedad permanece disponible y el caso de uso informa un error.
- **Fallo posterior de persistencia**: si la persistencia de la eliminación falla y se confirma que el registro permanece, se intenta restaurar el recurso de imagen mediante la integración autorizada existente. Si la restauración falla, el registro permanece disponible sin imagen y el resultado es error, nunca éxito.
- **Límite técnico**: `imageUrl` es una referencia, no una autorización ni prueba de que exista una operación remota `DELETE`. Se debe verificar la disponibilidad y autoridad de una integración existente para borrar y restaurar esa referencia. No se permite asumir un proveedor, introducir uno nuevo ni ejecutar borrado remoto arbitrario. La operación de persistencia y el almacenamiento no son atómicos entre sí; no se promete rollback remoto.

### Solicitud de eliminación

- **Entrada**: identificador UUID tomado de `/api/properties/{id}`.
- **Propósito**: solicitar el borrado de una sola propiedad después de la confirmación explícita de la persona operadora.
- **Duplicados**: la UI impide enviar otra solicitud mientras la primera siga en curso. Solicitudes independientes concurrentes se resuelven por la existencia persistida: una puede completar con `204`; una posterior a la eliminación recibe `404`.

### Resultado de eliminación

| Estado | Significado | Efecto permitido |
|---|---|---|
| `204 No Content` | Imagen (si existe) y propiedad eliminadas según la operación | UI confirma éxito y retira el elemento |
| `400 Bad Request` | Identificador con formato inválido | No se modifican datos |
| `404 Not Found` | Identificador válido, propiedad ausente | No se modifican otras propiedades; UI sincroniza estado obsoleto |
| Error técnico con `ProblemDetails` | Fallo al borrar imagen o persistir propiedad | No se informa éxito; fallo al borrar imagen conserva la propiedad |
| Persistencia fallida; restauración exitosa | La eliminación del registro no se confirma y el registro permanece | Se intenta restaurar la imagen y se informa error sin éxito |
| Persistencia fallida; restauración fallida | El registro permanece sin imagen | Se informa estado parcial y no se reporta éxito |
| Resultado de persistencia indeterminado | No se ha verificado si el registro permanece | No se afirma un estado ni se informa éxito; se trata como error técnico |
| Resultado de comunicación incierto | No se conoce si el backend completó | UI sincroniza antes de permitir reintento |

No se incorpora un tipo de respuesta con cuerpo para éxito.

### Confirmación de eliminación

- **Datos presentados**: identidad legible de la propiedad (incluido título) e indicación de que la eliminación es permanente.
- **Acciones**: confirmar o cancelar/cerrar.
- **Transiciones**: cerrar/cancelar devuelve a la vista sin solicitud ni cambios; confirmar inicia una única solicitud.
- **Reutilización**: el mismo componente se usa desde listado y detalle.

## Estados de interfaz

1. **Disponible**: acción de eliminar visible en listado y detalle.
2. **Confirmación abierta**: el componente identifica la propiedad e informa que el borrado es permanente.
3. **Cancelada**: confirmación cerrada sin llamada ni modificación.
4. **Eliminando**: solicitud en curso, confirmación y acción bloqueadas frente a duplicados.
5. **Eliminación confirmada**: respuesta `204`; mostrar éxito y retirar la tarjeta o abandonar el detalle.
6. **No encontrada**: respuesta `404`; mensaje correspondiente y sincronización de lista/detalle.
7. **Error recuperable**: `400`, error del servidor o fallo de comunicación; mensaje sin afirmar éxito.
8. **Sincronizando resultado incierto**: volver a consultar el estado antes de habilitar otro intento.

La confirmación reutilizable se ofrece en las acciones del listado y del detalle. Identifica la propiedad y avisa que la eliminación es permanente; cancelar o cerrar no envía solicitud ni modifica datos.

## Invariantes

- Cancelar o cerrar nunca ejecuta la API.
- La propiedad se elimina solo después de confirmar.
- Un fallo de eliminación del recurso de imagen no elimina la propiedad.
- Si falla la persistencia después de eliminar la imagen, se intenta restaurarla cuando se verifica que la propiedad sigue existiendo; un fallo de restauración deja la propiedad sin imagen y se comunica como estado parcial, sin éxito.
- Solo `204` se trata como éxito confirmado.
- Tras éxito o `404`, no se conserva un elemento obsoleto como accionable.
- La operación no cambia ni elimina otras propiedades.
- No se restaura una propiedad ya eliminada ni se incorpora archivado o eliminación masiva; la restauración compensatoria de imagen ante un fallo de persistencia no restaura la propiedad.
