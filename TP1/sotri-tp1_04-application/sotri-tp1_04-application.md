# sotri-tp1_04-application

## ¿Cómo implementar el procesamiento periódico mediante una Tarea?

La forma correcta de implementar procesamiento periódico en FreeRTOS es mediante `vTaskDelayUntil()`. A diferencia de `vTaskDelay()`, que agrega el retardo *después* de que la tarea termina su trabajo (acumulando drift con el tiempo), `vTaskDelayUntil()` bloquea la tarea hasta un instante absoluto, compensando automáticamente el tiempo de ejecución de cada iteración.

El patrón de uso es el siguiente:

```c
void vMiTareaPeriodica(void *parameters)
{
    TickType_t xLastWakeTime = xTaskGetTickCount(); // guardar tick inicial
    const TickType_t xPeriod = pdMS_TO_TICKS(25);  // período deseado

    for (;;)
    {
        /* trabajo de la tarea */

        vTaskDelayUntil(&xLastWakeTime, xPeriod);   // bloquear hasta el próximo ciclo
    }
}
```

`xLastWakeTime` se actualiza automáticamente en cada llamada. El resultado es una tarea que se despierta con período constante y libera la CPU entre ejecuciones, permitiendo que otras tareas y la tarea IDLE obtengan tiempo de procesamiento.

En este proyecto se aplica en `task_btn` (período 25 ms) y en `task_led` (período 250 ms).

---

## ¿Cuándo se ejecutará la Tarea IDLE y cómo se puede utilizar?

La tarea IDLE se ejecuta cuando **ninguna otra tarea está en estado Ready**, es decir, cuando todas las tareas de la aplicación están Blocked o Suspended esperando algún evento o retardo. Tiene la prioridad más baja del sistema (prioridad 0) y es creada automáticamente por FreeRTOS al iniciar el scheduler.

Antes de incorporar `vTaskDelayUntil()`, las tareas `task_btn` y `task_led` corrían en polling continuo y nunca liberaban la CPU, por lo que la tarea IDLE nunca se ejecutaba. Con el procesamiento periódico, entre cada activación las tareas quedan Blocked y la tarea IDLE obtiene tiempo de CPU.

Se puede aprovechar mediante el hook `vApplicationIdleHook()` (habilitado con `configUSE_IDLE_HOOK 1` en `FreeRTOSConfig.h`), ya implementado en `app/src/freertos.c`. Sus usos típicos son:

- **Bajo consumo energético:** ejecutar la instrucción `__WFI` (Wait For Interrupt) para detener el clock del núcleo hasta la próxima interrupción.
- **Diagnóstico:** incrementar un contador (`g_task_idle_cnt`) que indica cuánto tiempo libre tiene el sistema — un valor alto indica CPU subutilizada.
- **Limpieza de memoria:** FreeRTOS usa la tarea IDLE para liberar la RAM de tareas que fueron eliminadas con `vTaskDelete()`.

La restricción fundamental es que el código del idle hook **nunca debe bloquearse** (no puede llamar a `vTaskDelay()`, esperar semáforos, etc.) y debe retornar para que la tarea IDLE pueda cumplir su función de limpieza interna.

---

## Comportamiento observado en el paso 03

Se modificó la tarea `task_led` para que su procesamiento sea periódico mediante `vTaskDelayUntil()`. Antes de la modificación, la tarea ejecutaba su máquina de estados de forma continua dentro de un bucle infinito, sin liberar explícitamente el procesador. Luego del cambio, la tarea se ejecuta cada `DEL_LED_MED`, equivalente a 250 ms.

Al presionar el botón, la tarea del botón envía el evento `EV_LED_BLINK` hacia la tarea del LED. La tarea `task_led` recibe el evento, cambia al estado `ST_LED_BLINK` y enciende el LED. Luego, mientras permanece en ese estado, conmuta el LED cada `DEL_LED_MAX`, equivalente a 500 ms. Se observó que el LED parpadea de forma periódica y que la tarea ya no queda ejecutándose continuamente, sino que se bloquea hasta su siguiente activación.

## Comportamiento observado en el paso 04

Se modificó la tarea `task_btn` para que su procesamiento sea periódico mediante `vTaskDelayUntil()`. Antes de la modificación, la tarea leía el estado del pulsador de forma continua dentro de un bucle infinito. Luego del cambio, la tarea se ejecuta cada `DEL_BTN_MED`, equivalente a 25 ms.

La máquina de estados del botón continúa realizando el antirrebote mediante los estados `ST_BTN_FALLING` y `ST_BTN_RISING`, usando un tiempo de validación de `DEL_BTN_MAX`, equivalente a 50 ms. Al presionar el botón de forma estable, se detecta el evento `BTN PRESSED` y se envía `EV_LED_BLINK` a la tarea del LED. Al soltar el botón de forma estable, se detecta `BTN HOVER` y se envía `EV_LED_OFF`. Se observó que el sistema mantiene el comportamiento funcional esperado, pero con tareas periódicas que liberan CPU entre ejecuciones.