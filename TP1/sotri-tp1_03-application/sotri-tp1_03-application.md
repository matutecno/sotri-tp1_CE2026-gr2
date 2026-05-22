# sotri-tp1_03-application

## ¿Cómo usar el parámetro de Tarea?
En FreeRTOS, el parámetro de una tarea se usa para enviar información a la función de la tarea en el momento en que esta se crea. Este parámetro se pasa mediante el argumento `pvParameters` de la función `xTaskCreate()`, y dentro de la tarea se recibe como un puntero de tipo `void *`.

## ¿Cómo cambiar la prioridad de una Tarea ya creada?
Para cambiar la prioridad de una tarea ya creada en FreeRTOS se utiliza la función `vTaskPrioritySet()`. Para modificar una tarea específica, primero se debe guardar su identificador o `handle` al momento de crearla con `xTaskCreate()`. Luego, ese `handle` se pasa como argumento a `vTaskPrioritySet()` junto con la nueva prioridad deseada.

## Comportamiento observado en el paso 03

Al compilar y depurar, se observa que ambas instancias ejecutan el mismo código fuente, pero interactúan con hardware distinto sin interferirse. El RTOS asigna un bloque de control (TCB) y un espacio de Stack independiente para cada instancia. El parámetro pvParameters permite inyectar el contexto del hardware (puerto y pin) en tiempo de creación, demostrando que en un RTOS las tareas son reusables y la separación de memoria local está garantizada por el planificador.

## Comportamiento observado en el paso 04

Al arrancar el Scheduler, las dos tareas task_led toman el control inmediato de la CPU debido a que se crearon con prioridad configMAX_PRIORITIES - 1, preemocionando (desplazando) a cualquier otra tarea (como task_btn). Ejecutan su bloque de inicialización de forma determinista y, al invocar vTaskPrioritySet(NULL, config->prioridad_normal), el planificador reevalúa la lista de tareas Ready. Como sus prioridades acaban de bajar, la CPU realiza un cambio de contexto inmediato (Context Switch) hacia tareas de mayor o igual prioridad, recuperando el flujo relativo asignado originalmente para el resto de la ejecución.