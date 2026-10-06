[← Volver al resumen](../README.es.md) · 🇬🇧 [Read in English](case-study-01-data-integrity.md)

# Corrupción silenciosa en un formato mensual congelado

**El 87% de un formato oficial se había desincronizado y nadie lo sabía. El candado no podía vivir en la aplicación, porque la aplicación era justamente lo que lo esquivaba.**

---

## Qué es el formato

Cada almacén lleva un formato mensual de máximos y mínimos: por cada concepto que controla, cuál debe ser el mínimo y el máximo, cuánto se consumió y cuánto sobró. Es un formato oficial. Una vez que un mes cierra, ese mes es una **fotografía congelada**: una cifra ya reportada no puede cambiar después, o los meses dejan de cuadrar con lo que se entregó.

La columna del sobrante es la que importa aquí. Se calcula a partir del **stock actual**: no tiene filtro de fecha, porque el stock es algo que existe ahora, no algo que ocurrió en agosto.

Esa es toda la vulnerabilidad, dicha en una frase: una columna que siempre significa "hoy", guardada en una fila que significa "agosto".

Lo único que la convertía en fotografía mensual era una condición que se negaba a escribir fuera de la ventana del propio mes.

## El síntoma

El formato apareció con la misma cifra de sobrante en todos los meses del año. Después, tras un cambio, octubre apareció en ceros.

Dos respuestas equivocadas distintas del mismo origen, lo cual es típico: la condición que debía fijar cada mes en su lugar tenía agujeros, y según por cuál te colaras, o embarrabas un mes sobre todos los demás, o no escribías nada.

## Medir antes de arreglar

Lo primero no fue arreglar. Fue averiguar hasta dónde llegaba el daño.

Escribí un comando de conciliación: para cada concepto de cada almacén, recalcular la cifra desde los datos de origen y compararla con la guardada. El punto fino es que **repite las fórmulas en vez de llamar al método que las escribe**: el método escribe, y una verificación que escribe siempre dice que coincide.

La respuesta: **1,303 de 1,505 conceptos estaban desincronizados.** Ochenta y siete por ciento.

Ese número cambió el problema. No era un caso raro que parchar, era el estado normal de los datos. Todo lo que se apoyaba en ese formato se apoyaba en arena, y el primer entregable dejó de ser un arreglo: era reparar 1,303 filas y poder demostrar después que estaban bien.

## Las dos causas raíz

**Un periodo cerrado todavía se podía escribir.** El candado existía pero dependía de que el llamador pasara la bandera correcta, y había seis llamadores. Lo moví a la primera línea del método: un periodo cerrado regresa de inmediato salvo que quien llame sea el cierre mismo. Una regla que seis llamadores tienen que recordar no es una regla.

**El sobrante se podía escribir en un mes pasado.** La comprobación de ventana tenía dos puertas abiertas —una para filas que se creaban por primera vez, otra para un camino de cierre forzado— y ambas escribían el stock de hoy en el mes al que perteneciera la fila. Hoy el sobrante solo se escribe en el mes en curso; una fila creada fuera de su propio mes recibe una nota explícita diciendo que no tiene dato de sobrante, en lugar de un número que parece real.

La columna de consumo no necesitaba ese candado: filtra por fecha, así que recalcular agosto da agosto de verdad. Es reconstruible. El sobrante no. Esa asimetría es la razón de que las dos columnas se traten distinto, y conviene notarla antes de escribir el candado, no después.

## El problema de fondo

Con los dos candados puestos las cifras quedaban correctas, siempre y cuando cada cambio en el inventario se anunciara. La aplicación usaba signals de Django para eso.

**Los signals de Django se esquivan por diseño.** No se disparan con un `.update()` de queryset, ni con `bulk_create`, ni con SQL directo. Y el proyecto tenía alrededor de diez lugares haciendo una actualización masiva sobre justamente el campo que alimenta la fórmula del sobrante, por buenas razones: eso es lo que se usa cuando hay que cambiar mil filas sin cargar mil objetos.

Así que ninguna cantidad de trabajo sobre los signals iba a cerrar el hueco. El hueco no era un signal faltante: era una categoría de escritura que los signals no pueden ver.

## Las opciones

**Quitar las actualizaciones masivas.** Honesto, y equivocado. Existen porque son la herramienta correcta para esas operaciones. Cambiar diez de ellas por ciclos fila por fila habría canjeado un problema de correctitud por uno de rendimiento, y el siguiente ingeniero habría reintroducido una en menos de un año.

**Recalcular todo por horario.** Simple, y demasiado lento para ser seguro. El sobrante solo se puede calcular mientras el mes está abierto; si la tarea nocturna corre después del cierre, la cifra mala queda congelada para siempre.

**Poner la regla donde no se pueda esquivar.** Un trigger de base de datos se dispara en cada escritura, venga del camino de código que venga, incluidos los que todavía nadie ha escrito. Eso fue lo que se construyó.

## Lo que se construyó

Un trigger vigila las seis columnas que alimentan la fórmula del sobrante. Cuando alguna cambia, escribe un par `(almacén, concepto)` en una tabla de cola. Un trabajador drena la cola cada cinco minutos y recalcula solo esos pares.

El trigger es **deliberadamente tonto**: anota y regresa. No lleva nada de lógica de negocio. Toda la lógica se queda en Python, donde se puede leer, probar y cambiar sin ser especialista en bases de datos. El único trabajo del trigger es ser imposible de saltar.

Cuatro detalles decidieron si esto funcionaba:

**`IS DISTINCT FROM`, no `<>`.** Comparar un valor contra `NULL` con `<>` da `NULL`, no `true` ni `false`, y la condición falla en silencio. La mitad de esas columnas son nullables. Es el tipo de error que produce un sistema que funciona perfecto con los datos con los que lo probaste.

**Se anotan el par nuevo y el viejo.** Si una pieza se mueve de una plaza a otra, hay dos formatos afectados, no uno. Un trigger que solo anota el estado nuevo deja la plaza vieja silenciosamente mal.

**Una restricción de unicidad más "no hagas nada en caso de conflicto" mantiene la cola acotada.** Mil movimientos del mismo par dejan una fila. Sin eso, una operación masiva sobre diez mil filas encolaría diez mil trabajos para el mismo recálculo.

**Del lado de Python se lee el diccionario de la instancia, no los atributos.** El signal que fotografía los valores previos de una fila tiene que leerlos sin tocar los descriptores de atributo del modelo. En un queryset traído con `.only()` o `.defer()`, tocar un campo omitido dispara una consulta *por objeto*: un listado de cinco mil entradas se habría convertido en cinco mil y una consultas, con el culpable escondido dentro de un signal de otro módulo. Como quedó, el costo es de cero consultas cuando no cambió nada relevante.

## Demostrar que funcionaba

Un trigger que no se dispara se ve exactamente igual que un sistema sin nada que hacer. Así que se probó por el camino que los signals no ven: una actualización masiva, dentro de una transacción que después se revirtió, verificando que apareciera la fila en la cola. Apareció.

El comando de conciliación corrió entonces sobre producción y reportó **cero diferencias**, después de reparar las 1,303 que había encontrado.

## Dos decisiones de horario que vale la pena explicar

**La conciliación corre a las 2:45 AM, quince minutos antes del cierre de las 3:00.** No semanal, y no después. El sobrante solo se puede calcular mientras el mes sigue abierto; el cierre lo congela de forma permanente. Si la cifra está mal el último día del mes, queda mal para siempre. Una auditoría semanal podría caer después del cierre, y entonces solo serviría para documentar el error.

**El aviso va a superusuarios, por correo, y nunca al sistema de notificaciones.** Esas notificaciones llegan a supervisores de plaza y administradores, y "se repararon 40 conceptos" no es algo sobre lo que puedan actuar. Ponerlo ahí entrenaría a la gente que recibe alertas reales a ignorar su bandeja. El código del aviso además nunca lanza excepción: si el correo falla, los datos ya se repararon, y tumbar la reparación porque rebotó un correo sería absurdo.

---

## Qué me llevo de esto

Lo que volvería a hacer igual es **medir antes de arreglar**. El instinto, al ver una cifra mal, es encontrar el error y parcharlo. Correr la conciliación primero convirtió un reporte de bug en un número —87%— y ese número es lo que justificó construir un trigger en vez de agregar otra cláusula de guarda. Sin él habría parchado el síntoma y entregado, y los caminos de actualización masiva seguirían abiertos hoy.

Lo que hay que vigilar es el **diseño en dos capas**: los signals de Python ahora se traslapan con el trigger. Los signals son instantáneos, el trigger es garantizado, y los dos terminan llamando al mismo recálculo. Esa redundancia es deliberada, pero es del tipo de cosas que confunden a quien lea el código después, así que está documentada en los dos extremos.

[← Volver al resumen](../README.es.md)
