[← Volver al resumen](../README.es.md) · 🇬🇧 [Read in English](case-study-02-reversal-guardrail.md)

# Deshacer un documento sin corromper el inventario

**Revertir una transacción aprobada es fácil hasta que dos documentos no se ponen de acuerdo sobre dónde está una pieza. Esto es lo que costó hacer segura la cancelación, y lo que pasó cuando el dato necesario para revertir nunca se capturó.**

---

## La operación

Un lector falla en un carril de peaje. Un técnico lo cambia. El documento que registra el cambio —aprobado, con folio final— mueve inventario real: la pieza que entra queda *instalada* en ese carril, la que sale queda *dada de baja*.

A veces ese documento está mal. Alguien eligió el número de serie equivocado, o el reemplazo nunca ocurrió, o el lote entero se capturó dos veces. Hay que deshacerlo.

## El principio: contraponer, nunca borrar

Un documento cancelado no se elimina. Se **contrapone**, como funciona una nota de crédito en contabilidad:

- El folio nunca se libera, nunca se reutiliza, nunca se pone en nulo. Un hueco en la serie se explica, no se tapa.
- Las partidas y la evidencia fotográfica se conservan.
- Las observaciones de cada pieza no se reescriben: se le antepone una marca de cancelación al texto original.
- Todo lo restaurado queda escrito en un asiento de reversa con su motivo, su autor y su evidencia.

Un inventario que leen auditores no puede tener historia que desaparece. En el momento en que permites un borrado, la pregunta "¿cómo se veía esto en agosto?" deja de tener respuesta.

## Para deshacer hay que saber hacia *qué* deshacer

Suena obvio y es lo que casi todos los sistemas se equivocan. La aprobación muta cinco campos por pieza: estatus, estado de justificación, almacén, carril y observaciones. Para revertirla hacen falta los valores de antes.

Por eso la aprobación **fotografía el estado previo de cada pieza antes de tocar nada.** Dos detalles deciden si esa fotografía sirve de algo:

**Va primero en el ciclo, sin excepción.** Capturada después de la mutación, guardaría el estado nuevo, y la reversión devolvería la pieza a donde ya estaba: un no-op silencioso. El peor tipo de error, porque nada falla, simplemente miente en voz baja.

**Se escribe dentro de la misma transacción y del mismo candado de fila que la mutación.** O se guardan la fotografía y el cambio, o no se guarda ninguno. Una pieza nunca puede terminar instalada sin su foto previa.

Hay un campo que deliberadamente **no** se lee de la fotografía: el estado de justificación. Ese lo cambió antes el asistente que bloqueó la pieza, no la aprobación, así que la fotografía guarda un valor intermedio y el valor previo verdadero vive en otro lado. Restaurarlo desde la fotografía habría liberado piezas que estaban legítimamente amparadas por un documento anterior, y habrían reaparecido como pendientes. Nada habría fallado; simplemente los números habrían dejado de cuadrar. Saber *cuál* campo tiene mal la fuente obvia costó leer el asistente, no la aprobación.

## El guardarraíl

Tener la fotografía no basta. Considera esto:

```
01 sep   Documento A   instala el lector #1 en el carril 3
10 sep   Documento B   da de baja el lector #1, instala el lector #2
hoy      alguien intenta cancelar el documento A
```

Revertir A dejaría el lector #1 de vuelta en *instalado*, mientras B —todavía vigente— dice que se dio de baja. Dos documentos válidos contradiciéndose, y un inventario que no corresponde a ninguno.

Por eso la reversión se bloquea cuando una pieza siguió su camino después de la aprobación. Tres cosas pueden bloquearla: un documento de reemplazo posterior, una salida posterior, o una exención vigente sobre esa pieza.

## La parte que era correcta e inútil

El bloqueo decía: *"estos equipos ya siguieron su camino. Hay que deshacer en orden inverso: cancela primero el documento más reciente."*

Cierto, y de nada sirve. No decía **cuál**. Quien veía el mensaje tenía que salir a buscar a mano qué había tocado esa pieza después, en qué orden, y desde dónde se cancela cada cosa.

Así que ahora el bloqueo construye la ruta: cada documento que bloquea, del más reciente al más antiguo —que es el orden en que hay que deshacerlos— con su fecha, su estatus, las piezas en disputa y un enlace a donde se cancela cada uno. El último paso cierra el círculo: *vuelve aquí y cancela este*.

Dos decisiones de diseño dentro de eso:

**Agrupa por documento, no por pieza.** Si un folio posterior se quedó con tres de tus piezas, eso es **un** paso con tres series listadas, no tres pasos. Se cancela una sola vez.

**Un cuarto caso se reporta aparte.** A veces el estado de una pieza se movió sin ningún documento detrás. Esos no se arreglan cancelando nada, así que se muestran por separado, en otro color, diciendo exactamente eso. Mezclarlos con los pasos mandaría a alguien a cazar un documento que no existe, que es peor que no decir nada, porque parece una instrucción.

La ruta se muestra solo a superusuarios. Nombra documentos de otras plazas, de otros periodos y de gente que no le reporta a quien está mirando la pantalla.

## Cuando el dato para revertir nunca se capturó

La foto previa se agregó en cierto momento de la vida del sistema. Los documentos aprobados antes de eso no la tienen, y el bloqueo para esos dice, con razón, que no hay a qué estado regresar la pieza.

Eso era inofensivo mientras la ventana de cancelación fuera de 30 días: esos documentos quedaban fuera de alcance al mes y el caso se extinguía solo. Después la regla cambió —se permitió que un superusuario cancelara sin importar la fecha— y el caso se volvió alcanzable a cualquier antigüedad. **Un cambio correcto en sí mismo invalidó una suposición escrita en un comentario de otro lado.**

Una auditoría lo midió antes de decidir nada:

| | documentos | piezas |
|---|---|---|
| Aprobados sin foto previa | 11,345 | 17,296 |
| → reconstruibles desde el historial | | 4,064 (23%) |
| → el historial arrancó después de la aprobación | | 10 |
| → nada registrado antes de la aprobación | | 13,222 (76%) |

Tres de cada cuatro no se pueden recuperar. **La reconstrucción masiva quedó descartada**, y la respuesta honesta para esos es que no se pueden cancelar: la corrección se hace levantando un documento nuevo.

## La reparación, para los casos que sí importaban

Un lote de diez piezas se había capturado dos veces. Cinco de los documentos tenían que cancelarse, y los cinco eran anteriores a la foto previa.

La tentación es editar a mano las filas del inventario. Es el movimiento equivocado, y no por purismo: la cancelación hace mucho más que mover campos. Escribe el asiento de reversa, marca el historial de cada pieza con el motivo, recalcula el formato mensual, anota el cruce de periodo, libera la evidencia fotográfica para que se pueda reusar y avisa a los involucrados. Editar a mano deja inventario movido **sin ningún documento que lo explique**: justo la falla que el sistema existe para evitar.

Así que en vez de eso se repuso el *único dato que faltaba* y se dejó correr la cancelación normal. Una herramienta genera un plan: por cada pieza afectada prellena el estado previo desde el historial de filas donde existe, y deja los campos en **nulo** donde no — en nulo a propósito, porque un valor plausible prellenado se aprueba de un vistazo y un nulo obliga a decidir. Una persona llena esos, un ensayo en seco valida el plan completo sin escribir nada, y solo entonces se aplica: todo o nada, nunca pisando una fotografía real, y sellado con su fuente, su fecha y quién lo autorizó. Una fotografía reconstruida nunca se hace pasar por original, y la pantalla de confirmación lo avisa antes de ejecutar.

Al final: un campo escrito por pieza, y la reversión completa corriendo sola, con todo su rastro documental.

## El orden no era cosmético

Leyendo el plan, un número de serie aparecía dos veces: como la pieza que entra en un documento y como la que sale en otro.

```
como pieza que entra del doc A   →  vuelve a: disponible, sin carril
como pieza que sale del doc D    →  vuelve a: instalado, carril 3
```

Cancela D primero y la pieza pasa a *instalado*, luego A la baja a *disponible*. Correcto. Cancela A primero y termina *instalada*, que es exactamente lo que no quieres de un duplicado.

El guardarraíl obliga al orden bueno. No es una recomendación impresa en una pantalla: A está genuinamente bloqueada hasta que D desaparezca.

---

## Qué me llevo de esto

**El guardarraíl debe nombrar la salida.** Un bloqueo correcto pero mudo cuesta más de lo que ahorra, porque la persona que está enfrente va a encontrar otro camino, normalmente una edición directa a la base de datos. La ruta convirtió un muro en una instrucción, y esa es la diferencia entre un control que la gente respeta y uno que rodea.

**Medir antes de decidir.** La auditoría de 11,345 documentos tomó una tarde y mató una idea que me gustaba. Reconstruir todo desde el historial era elegante, y en tres cuartas partes de los casos habría escrito el estado posterior a la aprobación como si fuera el anterior: una cancelación que no revierte nada mientras reporta éxito. El número es lo que la detuvo.

**Arreglar la entrada, no la salida.** Reponer el único campo faltante y dejar correr la operación real conservó todas las garantías que esa operación trae consigo. Meter mano directo al inventario habría sido más rápido y habría costado el asiento, el historial, el recálculo y los avisos: todo lo que hace que el número sea confiable después.

[← Volver al resumen](../README.es.md)
