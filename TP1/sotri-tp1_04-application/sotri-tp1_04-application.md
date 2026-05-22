# sotri-tp1_04-application

## Comportamiento observado en el paso 03

Se modificó la tarea `task_led` para que su procesamiento sea periódico mediante `vTaskDelayUntil()`. Antes de la modificación, la tarea ejecutaba su máquina de estados de forma continua dentro de un bucle infinito, sin liberar explícitamente el procesador. Luego del cambio, la tarea se ejecuta cada `DEL_LED_MED`, equivalente a 250 ms.

Al presionar el botón, la tarea del botón envía el evento `EV_LED_BLINK` hacia la tarea del LED. La tarea `task_led` recibe el evento, cambia al estado `ST_LED_BLINK` y enciende el LED. Luego, mientras permanece en ese estado, conmuta el LED cada `DEL_LED_MAX`, equivalente a 500 ms. Se observó que el LED parpadea de forma periódica y que la tarea ya no queda ejecutándose continuamente, sino que se bloquea hasta su siguiente activación.

## Comportamiento observado en el paso 04

Se modificó la tarea `task_btn` para que su procesamiento sea periódico mediante `vTaskDelayUntil()`. Antes de la modificación, la tarea leía el estado del pulsador de forma continua dentro de un bucle infinito. Luego del cambio, la tarea se ejecuta cada `DEL_BTN_MED`, equivalente a 25 ms.

La máquina de estados del botón continúa realizando el antirrebote mediante los estados `ST_BTN_FALLING` y `ST_BTN_RISING`, usando un tiempo de validación de `DEL_BTN_MAX`, equivalente a 50 ms. Al presionar el botón de forma estable, se detecta el evento `BTN PRESSED` y se envía `EV_LED_BLINK` a la tarea del LED. Al soltar el botón de forma estable, se detecta `BTN HOVER` y se envía `EV_LED_OFF`. Se observó que el sistema mantiene el comportamiento funcional esperado, pero con tareas periódicas que liberan CPU entre ejecuciones.