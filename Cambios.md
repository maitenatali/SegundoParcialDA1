### Jornadas del Sur · Guardia de mantenimiento ### 

### Agenda personal**

El problema era que las favoritas podían perderse o quedar incorrectas al actualizar el catálogo. Además, no se contemplaba correctamente el caso de una charla retirada que posteriormente vuelve a ingresar.

- En `Programa.kt`, modifiqué `CarteleraDao` para que `reemplazar()` recupere las favoritas directamente desde Room justo antes de reemplazar el catálogo, dentro de una transacción.
- En `AgendaRepository.kt`, modifiqué `sincronizar()` para utilizar `dao.reemplazar(fichas)` en lugar de guardar directamente las charlas recibidas.

Así, las charlas que mantienen su identificador conservan su marca aunque cambien de título o sala. Las charlas retiradas desaparecen del catálogo y de Mi agenda; si vuelven a ingresar, aparecen sin marcar. Además, las marcas agregadas o quitadas durante una petición se respetan cuando llega la respuesta, porque se consulta el estado actualizado de las favoritas al momento de reemplazar los datos.

### Programa sin señal

La aplicación no diferenciaba correctamente entre **no tener conexión, no haber descargado nunca el programa y haber descargado exitosamente una edición sin charlas**.

- En `Programa.kt`, agregué consultas para recuperar las charlas guardadas y una operación transaccional para reemplazar la edición completa, incluso cuando viene vacía.
- En `AgendaRepository`, modifiqué `sincronizar()` para conservar las favoritas utilizando los identificadores estables de las charlas. Las charlas nuevas comienzan sin marcar y las que ya no existen se eliminan.
- En `ProgramaModel`, agregué una preferencia persistente (`hay_edicion_guardada`) para recordar si se descargó correctamente alguna edición, incluso si no contenía charlas. Ante un error, se informa si se están utilizando datos guardados o si todavía no hay datos disponibles.
- En `Catalogo.kt`, modifiqué la visualización para que la lista y los detalles sigan disponibles ante una falla, se pueda reintentar la descarga y se muestre «Sin charlas programadas» cuando la última edición descargada esté vacía.

El programa ahora conserva la última edición descargada correctamente cuando falla una actualización y permite distinguir entre una descarga fallida sin datos previos y una edición válida vacía.

### Volver al punto de lectura

El filtro de tema estaba en un `remember` dentro de `Catalogo`, así que se perdía al abrir una ficha o rotar. El filtro de día estaba en el ViewModel, que sobrevive a la rotación pero no a la muerte del proceso.

- En `ProgramaModel`, `dia` y `tema` pasaron a guardarse en un `SavedStateHandle` (con `elegirDia` y `elegirTema` para escribirlos).
- En `Catalogo`, elimine el `remember` del tema y los chips ahora leen y escriben esos valores del ViewModel.
- La ficha abierta ya se restauraba sola con `rememberSaveable` en `Portada`, no modifique nada.

Ahora los filtros sobreviven a abrir una ficha, rotar y a que Android mate el proceso. Una tarea nueva no tiene estado guardado, así que arranca sin filtros.

### Control de descargas

`onStart()` llamaba a `recargar()` cada vez que la actividad arrancaba (rotación, volver del fondo), y `recargar()` no impedía peticiones simultáneas.

- En `ProgramaModel`, agregue `cargarAlIniciar()`, que usa un flag en un `companion object` para hacer una sola carga automática por sesión.
- `recargar()` ahora sale si ya hay una petición en curso (`if (cargando.value) return`), y `cargando` se pone en `true` antes del `launch`.
- En `MainActivity`, `onStart()` llama a `cargarAlIniciar()` en lugar de `recargar()`.

Ahora se realiza una carga automática por proceso y una por cada “Actualizar” o “Reintentar”. Rotar, entrar a una ficha o cambiar de pestaña no agregan peticiones. Dos toques durante una petición cuentan como un solo intento. Al terminar, con éxito o falla, se puede volver a pedir. Los filtros siguen editables durante la espera porque no dependen de `cargando`.