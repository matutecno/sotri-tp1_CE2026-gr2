# sotri-tp1_02-application

## ¿Cómo FreeRTOS asigna tiempo de procesamiento a cada Tarea en una aplicación?

FreeRTOS asigna tiempo de procesamiento mediante su planificador o scheduler, el cual ejecuta siempre la tarea de mayor prioridad que se encuentre en estado Ready. En sistemas de un solo núcleo, solo una tarea puede usar la CPU a la vez, por lo que el planificador realiza cambios de contexto cuando una tarea se bloquea, cede la CPU o aparece una tarea de mayor prioridad. Además, FreeRTOS utiliza una interrupción periódica llamada tick interrupt, configurada mediante configTICK_RATE_HZ, para actualizar su base de tiempo y gestionar tareas bloqueadas por retardo. Si varias tareas tienen la misma prioridad y configUSE_TIME_SLICING está habilitado, el sistema puede alternar su ejecución mediante time slicing.

## ¿Cómo FreeRTOS elige qué Tarea debe ejecutarse en un momento dado?
El planificador (scheduler) de FreeRTOS es estricto y determinista. Sigue una regla inquebrantable: siempre ejecutará la tarea de mayor prioridad que se encuentre lista para ejecutarse.

Si hay una tarea de alta prioridad lista, se le otorga inmediatamente el control del procesador. Si hay dos o más tareas listas que comparten la misma prioridad más alta, FreeRTOS utiliza un algoritmo de planificación Round-Robin. Esto significa que alternará el control de la CPU entre esas tareas de igual prioridad de manera secuencial en cada tick del sistema.
## ¿Cómo la prioridad relativa de cada Tarea afecta el comportamiento del sistema?
En FreeRTOS, la prioridad relativa determina qué tarea tiene preferencia para usar la CPU: cuanto mayor es el número de prioridad, mayor es la urgencia de ejecución. 

En un sistema preemptivo, si una tarea de mayor prioridad pasa al estado Ready, puede interrumpir a una tarea de menor prioridad que esté ejecutándose y tomar la CPU. Sin embargo, si las tareas de alta prioridad permanecen siempre activas y no se bloquean ni esperan eventos, pueden provocar inanición (starvation) en las tareas de menor prioridad, impidiendo que estas se ejecuten.
## ¿Cuáles son los estados en los que puede encontrarse una Tarea?
En cualquier momento dado de la ejecución de la aplicación, una tarea se encuentra en uno de estos cuatro estados fundamentales:

Running (Ejecutándose): La tarea tiene el control total del procesador. En un microcontrolador de un solo núcleo, solo puede haber una tarea en este estado a la vez.

Ready (Lista / Preparada): La tarea está lista para procesar código, pero está pausada porque otra tarea de igual o mayor prioridad está usando actualmente el procesador.

Blocked (Bloqueada): La tarea está inactiva esperando que ocurra un evento temporal o externo. Esto puede ser esperar que pase un tiempo establecido (como vTaskDelay()) o esperar datos en una cola, un semáforo o un mutex. Mientras está bloqueada, no consume tiempo de procesamiento.

Suspended (Suspendida): La tarea está completamente detenida y el planificador la ignora. La única forma de entrar o salir de este estado es llamando explícitamente a las funciones vTaskSuspend() y vTaskResume().
## ¿Cómo implementar Tareas?
A nivel de código, una tarea se implementa como una función estándar del lenguaje C que nunca debe retornar ni finalizar. Se construyen dentro de un bucle infinito (for(;;) o while(1)). Es vital incluir una función de la API de FreeRTOS que provoque un estado de bloqueo para ceder el control del procesador a otras tareas.

```c
void vMiTarea(void *pvParameters) {
    /* Código de inicialización de la tarea (se ejecuta una sola vez) */
    
    for(;;) {
        /* Código principal de ejecución periódica */
        
        /* vTaskDelay bloquea la tarea y cede la CPU por 100 ticks */
        vTaskDelay(pdMS_TO_TICKS(100)); 
    }
}
```

## ¿Cómo crear una o más instancias de una Tarea?

Las tareas no existen para el planificador hasta que son creadas usando la API, específicamente la función xTaskCreate() (para asignación dinámica de memoria) o xTaskCreateStatic() (para memoria estática).

Puedes crear múltiples instancias a partir de la misma función base de C. Simplemente llamas a la API de creación varias veces, utilizando el argumento pvParameters para pasarle un contexto diferente a cada instancia (por ejemplo, para que una instancia controle un motor izquierdo y la otra un motor derecho, ejecutando el mismo código subyacente).

```c
/* Crear la Instancia 1 */
xTaskCreate(vMiTarea, "Motor_Izq", 1000, (void*) ParametroIzq, 1, NULL);

/* Crear la Instancia 2 utilizando la MISMA función (vMiTarea) */
xTaskCreate(vMiTarea, "Motor_Der", 1000, (void*) ParametroDer, 1, NULL);
```

## ¿Cómo eliminar una Tarea?
Para eliminar una tarea y removerla del planificador se utiliza la función vTaskDelete().

* **Autodestrucción:** Si una tarea ya no es necesaria y desea eliminarse a sí misma, simplemente debe llamar a vTaskDelete(NULL);.

* **Eliminar otra tarea:** Si una tarea necesita eliminar a otra, debe pasar como argumento el manejador (TaskHandle) de la tarea objetivo, el cual se obtuvo al momento de su creación.

* **Al eliminar una tarea,** FreeRTOS se encarga de liberar automáticamente la memoria RAM dinámica que se le había asignado para su funcionamiento interno y su pila de llamadas (Stack).


## Paso 03: Modificar las prioridades relativas asignadas a task_btn y task_led, compilar/depurar/observar comportamiento, asentar lo observado en el archivo sotri-tp1_02-application.md y reestablecer las prioridades relativas asignadas originales.

Al analizar el código fuente de `task_btn` y `task_led`, se observa que ambas tareas están implementadas como bucles infinitos que sondean continuamente los estados y el tiempo (`xTaskGetTickCount()`) sin hacer uso de llamadas bloqueantes de la API de FreeRTOS (como `vTaskDelay`). 

Debido a esta arquitectura de "polling continuo", las tareas consumen el 100% del tiempo de CPU que el planificador les otorga, lo que hace que el sistema sea extremadamente sensible a la asignación de prioridades.

## Casos de Prueba

### Caso 1: Ambas tareas tienen la MISMA prioridad (Comportamiento Original)
* **Observación:** El sistema funciona correctamente. El botón responde y el LED parpadea cuando corresponde.
* **Justificación:** Al tener la misma prioridad y no bloquearse nunca, el planificador (*scheduler*) de FreeRTOS utiliza el algoritmo *Round-Robin* (Time Slicing). El sistema operativo interrumpe periódicamente a cada tarea (en cada *Tick*) para darle tiempo de procesamiento a la otra, permitiendo que ambas avancen simultáneamente.

### Caso 2: `task_btn` tiene MAYOR prioridad que `task_led`
* **Observación:** Se pueden detectar las pulsaciones del botón, pero el LED nunca reacciona (nunca se enciende ni parpadea).
* **Justificación:** Inanición (*Starvation*). Como `task_btn` tiene mayor prioridad y nunca entra en estado *Blocked* (Bloqueado), el planificador de FreeRTOS le asigna el 100% del tiempo de CPU de forma ininterrumpida. `task_led` nunca pasa al estado *Running* para procesar los eventos que `task_btn` le envía.

### Caso 3: `task_led` tiene MAYOR prioridad que `task_btn`
* **Observación:** El sistema parece congelado y el botón deja de responder por completo.
* **Justificación:** Inanición (*Starvation*). De forma inversa al Caso 2, `task_led` acapara el 100% del tiempo del microcontrolador verificando continuamente sus banderas y el tiempo transcurrido. `task_btn` nunca obtiene tiempo de procesador para leer el estado físico del pin GPIO, por lo que nunca se detectan las pulsaciones.

## Conclusión final
Para que un sistema operativo en tiempo real (RTOS) funcione correctamente con tareas de distinta prioridad, es un requisito estricto que las tareas de mayor prioridad estén controladas por eventos (utilizando colas, semáforos o retrasos bloqueantes) para que cedan la CPU a las tareas de menor prioridad cuando no tienen trabajo útil que realizar.


## Paso 04: Crear tres instancias de task_btn para gestionar el mismo botón y que task_led elimine una de las instancias de task_btn, compilar/depurar/observar comportamiento y asentar lo observado en el archivo sotri-tp1_02-application.md.

### Fase A: Comportamiento antes de la eliminación (3 Instancias Activas)
* **Observación:** El comportamiento del botón se volvió errático y altamente inconsistente. Al presionar el botón una única vez, el LED a veces reaccionaba inmediatamente, a veces ignoraba la pulsación, o bien se imprimían logs duplicados de `"BTN PRESSED"` de forma caótica. El contador `g_task_btn_cnt` incrementaba a una velocidad aproximadamente tres veces mayor de la habitual.
* **Justificación:** Corrupción de Datos por Falta de Reentrada. Las tres tareas comparten la misma estructura global `task_btn_dta`. Si la `Instancia 1` detecta el botón presionado y cambia el estado a `ST_BTN_FALLING` guardando su *tick* de tiempo, en el siguiente *Time Slice* la `Instancia 2` o `Instancia 3` lee el estado modificado, corrompe el valor de `task_btn_dta.tick` con su propio tiempo actual o altera el estado interno de la máquina. Hay una condición de carrera (*Race Condition*) destructiva sobre las variables de estado y el contador global.

### Fase B: Comportamiento tras la eliminación por `task_led` (2 Instancias Activas)
* **Observación:** Al presionar el botón por primera vez, el log del sistema confirma la acción: `task_led - ELIMINANDO BTN_Inst 1`. Tras este punto, el sistema continúa funcionando de forma anómala, pero el contador `g_task_btn_cnt` reduce su velocidad de incremento (ahora avanza al doble de la velocidad normal en lugar del triple). Las pulsaciones siguen mostrando inconsistencias y rebotes de software erráticos.
* **Justificación:** Al invocar `vTaskDelete(xTaskBtnHandle1)`, FreeRTOS elimina correctamente a `BTN_Inst 1` de la lista de tareas listas (*Ready List*) y libera su Stack asignado. Sin embargo, dado que `BTN_Inst 2` y `BTN_Inst 3` siguen existiendo y continúan compartiendo la estructura global `task_btn_dta`, el conflicto de concurrencia y la corrupción de datos persisten entre las dos instancias supervivientes.

### Conclusión final
Para instanciar múltiples veces una misma función de tarea en FreeRTOS de forma segura, el código debe ser estrictamente **reentrante**. No debe utilizar variables globales compartidas de forma directa. En su lugar, cada instancia debe recibir un puntero a su propia estructura de datos privada encapsulada a través del parámetro de creación `void *parameters`.