# sotri-tp1_03-application

## ¿Cómo usar el parámetro de Tarea?
En FreeRTOS, el parámetro de una tarea se usa para enviar información a la función de la tarea en el momento en que esta se crea. Este parámetro se pasa mediante el argumento `pvParameters` de la función `xTaskCreate()`, y dentro de la tarea se recibe como un puntero de tipo `void *`.

## ¿Cómo cambiar la prioridad de una Tarea ya creada?
Para cambiar la prioridad de una tarea ya creada en FreeRTOS se utiliza la función `vTaskPrioritySet()`. Para modificar una tarea específica, primero se debe guardar su identificador o `handle` al momento de crearla con `xTaskCreate()`. Luego, ese `handle` se pasa como argumento a `vTaskPrioritySet()` junto con la nueva prioridad deseada.