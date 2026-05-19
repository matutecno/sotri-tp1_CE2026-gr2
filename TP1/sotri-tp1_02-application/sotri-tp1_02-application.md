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