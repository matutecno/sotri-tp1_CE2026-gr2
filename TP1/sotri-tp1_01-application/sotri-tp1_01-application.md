# sotri-tp1_01-application

## Análisis del código fuente — Consulta Gemini (Paso 06)

<!-- Pegar aquí la respuesta de Gemini sobre: startup_stm32f103rbtx.s, main.c, stm32f1xx_it.c, FreeRTOSConfig.h, freertos.c -->
## 1. Análisis y Explicación del Funcionamiento del Código Fuente

### A. `startup_stm32f446retx.s`
Es el archivo de arranque (startup) escrito en ensamblador ARM Cortex-M4 (sintaxis unificada). Sus funciones principales son:
* **Definición de la Tabla de Vectores (`g_pfnVectors`):** Modela la distribución de la memoria al inicio del mapa de direcciones (0x0000_0000 o mapeado mediante alias). Coloca en la primera posición el puntero inicial de la pila (`_estack`) y en la segunda la dirección del manejador de reset (`Reset_Handler`), seguido por las excepciones del sistema y las interrupciones externas (IRQ).
* **`Reset_Handler`:** Es el punto de entrada físico del procesador tras un reset. Realiza las siguientes tareas secuenciales:
  1. Inicializa el puntero de pila (`sp`) con la dirección `_estack`.
  2. Llama a la función `SystemInit` (proporcionada por CMSIS) para realizar configuraciones básicas del sistema de reloj y memoria.
  3. Copia los valores iniciales de las variables globales/estáticas inicializadas desde la memoria Flash (`_sidata`) hacia la SRAM (`_sdata` hasta `_edata`).
  4. Inicializa con ceros la sección `.bss` en la SRAM (`_sbss` hasta `_ebss`), asegurando que las variables no inicializadas comiencen en cero.
  5. Llama a las funciones de inicialización de arreglos de la biblioteca estándar de C (`__libc_init_array`).
  6. Bifurca (salta) a la función `main()` de la aplicación.

### B. `main.c`
Constituye la pieza central de la inicialización de la aplicación y la orquestación del hardware. Su flujo se resume en:
* **Inicialización de la HAL:** Invoca a `HAL_Init()`, que configura el flash prefetch, las prioridades de interrupción y establece una base de tiempo de ticks.
* **Configuración del Reloj del Sistema (`SystemClock_Config`):** Modifica los registros de la unidad de control de reloj (RCC) para conmutar la fuente de reloj principal desde el oscilador interno (HSI) de 16 MHz hacia el lazo de seguimiento de fase (PLL), elevando la frecuencia del bus del sistema.
* **Inicialización de Periféricos:** Llama a funciones autogeneradas (`MX_GPIO_Init`, `MX_USART2_UART_Init` y `MX_TIM2_Init`) que configuran las estructuras de control (`HandleTypeDef`) y registros de los periféricos asociados.
* **Arranque de la Aplicación y RTOS:** * Inicia el Timer 2 en modo interrupción (`HAL_TIM_Base_Start_IT(&htim2)`).
  * Invoca `app_init()` para preparar la lógica de la aplicación.
  * Define y crea estáticamente un hilo por defecto (`defaultTask`) utilizando las macros de CMSIS-RTOS (`osThreadDef`, `osThreadCreate`).
  * Inicia el planificador multitarea con `osKernelStart()`.

### C. `stm32f4xx_it.c`
Mapea los vectores físicos de interrupción definidos en el archivo `.s` a funciones ejecutables en C.
* **Excepciones del Núcleo:** Define manejadores críticos como `NMI_Handler`, `HardFault_Handler`, `MemManage_Handler`, `BusFault_Handler` y `UsageFault_Handler`. Contienen bucles infinitos para atrapar fallos catastróficos de hardware y permitir la inspección mediante un depurador.
* **Interrupciones de Periféricos:** Aloja las funciones `TIM1_UP_TIM10_IRQHandler` y `TIM2_IRQHandler`. Su única responsabilidad es redirigir el flujo hacia la función genérica de la HAL (`HAL_TIM_IRQHandler`), pasando como argumento el manejador correspondiente (`&htim1` o `&htim2`).

### D. `FreeRTOSConfig.h`
Es el archivo de cabecera de configuración que adapta el núcleo de FreeRTOS a los requisitos del hardware y de la aplicación:
* `configUSE_PREEMPTION 1`: Activa el planificador preentivo, permitiendo que las tareas de mayor prioridad desalojen inmediatamente a las de menor prioridad.
* `configTICK_RATE_HZ 1000`: Establece que el latido (*tick*) del sistema operativo ocurra cada 1.000 ms (1 kHz).
* `configSUPPORT_STATIC_ALLOCATION 1`: Habilita la creación de objetos del RTOS (tareas, colas) utilizando memoria asignada estáticamente en tiempo de compilación.
* `configGENERATE_RUN_TIME_STATS 1`: Activa la recolección de estadísticas de uso de CPU de las tareas. Requiere enlazar un temporizador externo de mayor velocidad a través de las macros `portCONFIGURE_TIMER_FOR_RUN_TIME_STATS` y `portGET_RUN_TIME_COUNTER_VALUE`.
* Mapeo de vectores: Traduce los nombres internos del puerto FreeRTOS (`vPortSVCHandler`, `xPortPendSVHandler`, `xPortSysTickHandler`) hacia los nombres estándar de CMSIS (`SVC_Handler`, `PendSV_Handler`, `SysTick_Handler`).

### E. `freertos.c`
Proporciona la infraestructura complementaria requerida por la configuración de FreeRTOS:
* Implementa `vApplicationGetIdleTaskMemory` para suministrar de forma estática los buffers de estructuras de control (`StaticTask_t`) y la pila (`StackType_t`) destinados a la tarea *Idle* del sistema.
* Contiene definiciones débiles (`__weak`) para las funciones de gancho (*hooks*) del sistema: `vApplicationIdleHook`, `vApplicationTickHook` y `vApplicationStackOverflowHook`, permitiendo al desarrollador instrumentar diagnósticos o lógicas de bajo consumo de energía.

---

## 2. Evolución de las Variables `SysTick` y `SystemCoreClock`

El comportamiento de estas variables clave se describe en la siguiente línea de tiempo de ejecución:

| Fase de Ejecución | Estado de `SystemCoreClock` | Estado de la Base de Tiempo `SysTick` |
| :--- | :--- | :--- |
| **`Reset_Handler`** | Inicializada implícitamente al valor por defecto del oscilador interno HSI (16 MHz en la familia STM32F4). | **Inactivo / Desactivado.** El periférico SysTick en el núcleo Cortex-M4 no se inicializa ni cuenta. |
| **Inicio de `main()` hasta antes de `SystemClock_Config()`** | Se mantiene en 16 MHz (HSI). | **Inactivo.** No hay modificaciones en los registros de configuración del SysTick (`STK_CTRL`). |
| **Posterior a `SystemClock_Config()`** | Refleja la nueva frecuencia calculada a partir del PLL multiplicador. Basado en los registros HSI=16MHz, M=16, N=336, P=4, el reloj del sistema (`SYSCLK`) sube a **84 MHz**. | **Inactivo.** Aunque el reloj del sistema cambió, el temporizador interno del núcleo aún no se ha encendido. |
| **Durante `HAL_Init()`** | Mantiene el valor configurado del bus (84 MHz). | **Inactivo.** Cabe notar que la HAL típicamente utiliza otro recurso (en este proyecto `TIM1`) como base de tiempo interna para evitar conflictos, dejando SysTick libre. |
| **Llamada a `osKernelStart()`** | Se mantiene constante a la frecuencia máxima configurada (84 MHz). | **Activado y en Funcionamiento.** FreeRTOS configura el periférico cargando el registro de recarga (`STK_LOAD`) para que desborde exactamente a 1 kHz (`configTICK_RATE_HZ`). Comienzan a dispararse de forma periódica las excepciones `SysTick_Handler`. |

---

## 3. Comportamiento del Programa desde `Reset_Handler` hasta antes del `while (1)`

El flujo de control del firmware sigue una transición estricta desde un entorno monohilo determinista por hardware hasta un entorno de ejecución concurrente gestionado por software:

1. **Hardware Boot:** Al arrancar, el procesador se encuentra en Modo Hilo (*Thread mode*), con nivel de acceso Privilegiado y utilizando el Stack Principal (MSP).
2. **Inicialización Física:** Se ejecuta la limpieza y el copiado de memoria en el startup y se da paso al código C.
3. **Establecimiento de Abstracciones:** Se configuran los relojes del sistema y la base de tiempo interna para las funciones de retardo de ST. Los periféricos físicos se inicializan con sus llamadas correspondientes de la HAL.
4. **Instanciación del RTOS:** Se llama a `osThreadCreate()`, la cual llena el Bloque de Control de Tareas (TCB) de la tarea predeterminada y prepara su espacio de pila simulación de contexto inicial.
5. **Punto de No Retorno (`osKernelStart`):** Al invocar esta función, FreeRTOS toma el control absoluto de las excepciones de hardware (`SVC_Handler`), configura las prioridades de los registros del NVIC, inicializa el temporizador de conteo por hardware `SysTick` y realiza el primer cambio de contexto cargando los registros de la tarea con mayor prioridad en el procesador.
6. **Bucle `while (1)` Inalcanzable:** Debido a que el planificador (*scheduler*) reemplaza de forma permanente el flujo secuencial del microcontrolador por un mecanismo de interrupciones y conmutación de hilos, el flujo de ejecución nunca retorna a la función `main()`. El bucle `while(1)` al final de `main()` queda aislado como código muerto, accesible únicamente si existiera un fallo catastrófico por falta de memoria dinámica/estática al arrancar el kernel.

---

## 4. Interacción de SysTick y Timer 2 (`TIM2`) con FreeRTOS

Ambos periféricos de hardware son utilizados por el kernel de FreeRTOS para propósitos completamente diferenciados:

### A. SysTick (Gestión de Tiempo y Conmutación)
* **Cómo interactúa:** A través del mapeo de interrupciones en `FreeRTOSConfig.h`: `#define xPortSysTickHandler SysTick_Handler`. El vector de interrupción de SysTick apunta directamente al manejador interno del puerto de FreeRTOS.
* **Para qué sirve:** Es el latido de tiempo operativo (*Tick Source*). Cada 1 ms, la interrupción interrumpe la tarea actual para:
  1. Incrementar el contador de tiempo del sistema (`xTickCount`).
  2. Despertar a las tareas que se encontraban bloqueadas temporalmente (ej. por llamadas a `osDelay`).
  3. Evaluar si la tarea en ejecución ha agotado su rodaja de tiempo (en modo *Time-slicing*) o si una tarea de mayor prioridad ha salido de su estado de bloqueo, forzando la solicitud de una excepción `PendSV` para realizar un cambio de contexto inmediato.

### B. Timer 2 (`TIM2`) (Métricas y Perfilado de Rendimiento)
* **Cómo interactúa:** Funciona de manera independiente al reloj de *Tick* del sistema. Incrementa una variable global volátil llamada `ulHighFrequencyTimerTicks` dentro de su función de callback de interrupción periódica. FreeRTOS se enlaza a esta variable por medio de las siguientes directivas macro:
  ```c
  #define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS configureTimerForRunTimeStats
  #define portGET_RUN_TIME_COUNTER_VALUE getRunTimeCounterValue

---

## 5. Interacción del Timer 2 (TIM2) con la HAL del Proyecto STM32

La interacción de TIM2 con la capa de abstracción de hardware de STMicroelectronics sigue el patrón de diseño estructurado basado en interrupciones y funciones de retrollamada (callbacks):

* **Configuración de Estructuras:** En main.c, la variable global TIM_HandleTypeDef htim2 encapsula la instancia física del periférico y los parámetros de inicialización de su base de tiempo.

* **Disparo de la Interrupción por Hardware:** Cuando el contador interno del periférico alcanza el valor límite configurado en el registro de periodo (TIM2->ARR = 4199), el hardware del módulo temporizador genera una señal de interrupción por desbordamiento que es procesada por el Controlador de Interrupciones Vectorizado Anidado (NVIC).

* **Manejador Físico (ISR):** El procesador detiene la ejecución actual y salta a la función TIM2_IRQHandler() ubicada en stm32f4xx_it.c. Esta función actúa como el punto de entrada directo de nivel de silicio.

* **Abstracción y Limpieza de Flags de la HAL:** La ISR invoca inmediatamente a la función de la librería HAL_TIM_IRQHandler(&htim2). Esta función interna de la HAL realiza inspecciones críticas de bajo nivel: valida qué registro causó el disparo (ej. actualización, captura, comparación) y limpia automáticamente los bits de banderas de interrupción (UIF) en el registro de estado (TIM2->SR) para prevenir que la interrupción se vuelva a disparar inmediatamente de forma infinita.

* **Llamada de Retorno (Callback):** Una vez procesados los registros, la función de la HAL enruta el flujo hacia la función común de notificación de eventos: HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim).

* **Ejecución de la Lógica del Usuario:** Dentro de este Callback (implementado al final de main.c), el código evalúa la condición de la instancia: if (htim->Instance == TIM2). Al confirmarse que el evento proviene de dicho temporizador, incrementa la variable ulHighFrequencyTimerTicks, completando la sincronización de estados entre el hardware físico, la abstracción de ST y los requisitos métricos de la aplicación.

---

## Análisis del código fuente — Consulta Gemini (Paso 08)

El código fuente proporcionado implementa una arquitectura basada en eventos (Event-Triggered System) utilizando el sistema operativo en tiempo real FreeRTOS. Su propósito principal es gestionar la concurrencia entre dos tareas: una que monitoriza un pulsador físico (con lógica de antirrebote) y otra que controla el estado y parpadeo de un LED. El código hace uso intensivo de máquinas de estados (statecharts) y de la capa de abstracción de hardware (HAL), típica del ecosistema de microcontroladores STM32.

A continuación, se detalla el funcionamiento de cada uno de los archivos:

## 1. Análisis Detallado por Archivo

### 1.1. `app.c` — Inicialización y Orquestación de Tareas
Este archivo actúa como el módulo de inicialización de la capa de aplicación antes de que comience la ejecución del planificador de FreeRTOS.

* **Definición de Variables Globales de Monitoreo:**
    Se declaran contadores de 32 bits (`g_app_cnt`, `g_app_task_cnt`, `g_app_tick_cnt`, `g_task_idle_cnt`, `g_app_stack_overflow_cnt`) orientados a la auditoría del rendimiento y diagnóstico de fallos en el sistema.
* **Función `app_init()`:**
    1.  **Inicialización de Variables:** Resetea los contadores globales al valor inicial `0ul`.
    2.  **Registro de Logger:** Imprime mensajes informativos en la consola de depuración indicando que la aplicación ha iniciado y el Tick actual del sistema mediante `xTaskGetTickCount()`.
    3.  **Creación de Tareas con `xTaskCreate`:**
        * **`Task BTN`:** Asocia la función `task_btn`. Se le asigna un tamaño de pila equivalente a `2 * configMINIMAL_STACK_SIZE` y una prioridad de `tskIDLE_PRIORITY + 1ul` (Prioridad 1). Su manejador de referencia es `h_task_btn`.
        * **`Task LED`:** Asocia la función `task_led`. Comparte los mismos atributos de tamaño de pila y prioridad que la tarea del botón, facilitando un esquema de tiempo compartido por sustitución (*Round-Robin*) si ambas están listas. Su manejador es `h_task_led`.
    4.  **Manejo de Errores Críticos:** Cada creación de tarea es validada mediante `configASSERT(pdPASS == ret)`. Si falla la asignación de memoria en el *heap*, la ejecución se detiene para diagnóstico.
    5.  **Diagnóstico de Memoria:** Llama a `xPortGetFreeHeapSize()` para verificar la cantidad de memoria dinámica restante en el *Heap*.
    6.  **Inicialización de Periféricos de Bajo Nivel:** Invoca a `app_it_init()` para configurar las interrupciones específicas de la aplicación y a `cycle_counter_init()` para habilitar el contador de ciclos de reloj (DWT), útil para métricas de tiempo de ejecución precisas.

---

### 1.2. `task_btn.c` — Monitoreo de Entrada y Filtro Antirrebote
Este módulo gestiona la lectura del pulsador mecánico conectado a un pin GPIO mediante una máquina de estados orientada al tiempo.

* **Estructura de Datos (`task_btn_dta`):**
    Almacena de forma encapsulada el evento actual (`EV_BTN_UP`), el estado interno de la FSM (`ST_BTN_UP`), el contador de tiempo de referencia (`tick`), y las referencias de hardware GPIO (`B1_GPIO_Port`, `B1_Pin`).
* **Ciclo de Ejecución Infinito (`task_btn`):**
    La tarea corre dentro de un bucle `for(;;)` donde incrementa el contador de diagnóstico `g_task_btn_cnt` e invoca continuamente a la función `task_btn_statechart()`. Dado que esta tarea no se bloquea explícitamente (no utiliza funciones como `vTaskDelay`), confía en que el planificador realice un cambio de contexto por tiempo compartido (*time-slicing*) o que la FSM resuelva eficientemente.
* **Máquina de Estados Finita (`task_btn_statechart`):**
    1.  **Lectura del Hardware:** Evalúa el pin mediante `HAL_GPIO_ReadPin()`. Si coincide con `BTN_PRESSED`, actualiza el evento interno a `EV_BTN_DOWN`; de lo contrario, se define como `EV_BTN_UP`.
    2.  **Lógica de Transición (Switch-Case):**
        * `ST_BTN_UP` (Reposo, botón suelto): Si el evento cambia a `EV_BTN_DOWN`, guarda el tiempo actual (`xTaskGetTickCount()`) y transiciona a `ST_BTN_FALLING`.
        * `ST_BTN_FALLING` (Validación de pulsación): Espera a que transcurra un tiempo mayor o igual a `DEL_BTN_MAX` (50 ms/ticks). Pasado este tiempo, vuelve a verificar el hardware: si sigue presionado, confirma una pulsación legítima, registra el log `"BTN PRESSED"`, envía el comando `EV_LED_BLINK` a la tarea del LED a través de la interfaz y pasa a `ST_BTN_DOWN`. Si fue un ruido o rebote corto, regresa a `ST_BTN_UP`.
        * `ST_BTN_DOWN` (Botón retenido): Permanece en este estado hasta que el evento cambia a `EV_BTN_UP`. En ese instante, registra el tick actual y pasa a `ST_BTN_RISING`.
        * `ST_BTN_RISING` (Validación de liberación): Aplica un retardo de filtrado de 50 ms (`DEL_BTN_MAX`). Si al concluir el tiempo el botón se mantiene suelto, confirma la liberación, registra el log `"BTN HOVER"`, envía el comando `EV_LED_OFF` al LED y regresa al estado de reposo `ST_BTN_UP`.

---

### 1.3. `task_led.c` — Controlador y Actuador del LED
Este componente se encarga de modificar el estado físico del LED de la placa de desarrollo (`LD2`) respondiendo de manera reactiva a las señales lógicas enviadas por la tarea del botón.

* **Estructura de Datos (`task_led_dta`):**
    Contiene la bandera de actualización (`flag`), el evento recibido (`event`), el estado actual de la FSM (`state`), el registro de tiempo (`tick`), y la configuración física del pin GPIO (`LD2_GPIO_Port`, `LD2_Pin`).
* **Función de Tarea (`task_led`):**
    Apaga inicialmente el LED usando `HAL_GPIO_WritePin` y entra en su bucle infinito incrementando `g_task_led_cnt` y evaluando la FSM mediante `task_led_statechart()`.
* **Máquina de Estados Finita (`task_led_statechart`):**
    * `ST_LED_OFF` (Estado Apagado): Monitorea si `flag == true` y si el evento es `EV_LED_BLINK`. Al cumplirse, limpia la bandera (`flag = false`), guarda el tiempo de referencia en `tick`, cambia de estado a `ST_LED_BLINK` y enciende el pin físico del LED (`LED_ON`).
    * `ST_LED_BLINK` (Estado de Parpadeo): 
        * *Prioridad de Interrupción:* Primero evalúa si ha llegado un nuevo evento de apagado (`EV_LED_OFF` con `flag == true`). Si es así, consume el evento, apaga físicamente el LED (`LED_OFF`) y regresa inmediatamente a `ST_LED_OFF`.
        * *Temporización Asíncrona:* Si no hay órdenes de apagado, evalúa el tiempo transcurrido desde el último cambio. Si la diferencia de ticks es mayor o igual a `DEL_LED_MAX` (500 ms), actualiza la variable `tick` e invierte el estado del pin físico utilizando `HAL_GPIO_TogglePin()`. Esto produce un parpadeo constante a una frecuencia de 1 Hz (500 ms encendido, 500 ms apagado) sin bloquear el hilo de ejecución.

---

### 1.4. `task_led_interface.c` — Interfaz de Comunicación entre Tareas (IPC)
Este archivo implementa el patrón de diseño de desacoplamiento de software mediante una interfaz de funciones.

* **Función `put_event_task_led(task_led_ev_t event)`:**
    Permite que cualquier tarea externa (en este caso, `task_btn`) envíe un evento de forma segura a la estructura interna del LED. Modifica la variable compartida `task_led_dta.event` con el nuevo comando y establece `task_led_dta.flag = true` para indicarle a la FSM del LED que tiene una nueva instrucción pendiente por procesar.
    
    *Nota Arquitectural:* Aunque funcional para propósitos educativos, al no utilizar primitivas nativas de FreeRTOS (como colas de mensajes o grupos de eventos) ni secciones críticas, este mecanismo carece de protección intrínseca contra condiciones de carrera (*race conditions*), dependiendo enteramente de la naturaleza atómica de las escrituras de punteros/variables en arquitecturas ARM Cortex-M.

---

### 1.5. `freertos.c` — Funciones de Gancho (Hooks) y Diagnóstico del Kernel
Este archivo define las funciones de callback globales que el núcleo de FreeRTOS invoca de manera automática cuando experimenta eventos operativos específicos.

* **`vApplicationIdleHook(void)`:**
    Es invocada repetidamente por la tarea *Idle* de FreeRTOS, la cual tiene la prioridad más baja del sistema y solo se ejecuta cuando ninguna otra tarea de la aplicación está lista para ejecutarse. En este código, incrementa el contador `g_task_idle_cnt`. En aplicaciones comerciales, este espacio es vital para ejecutar instrucciones de bajo consumo (`WFI` - *Wait For Interrupt*) para apagar módulos del microcontrolador y ahorrar energía.
* **`vApplicationTickHook(void)`:**
    Se ejecuta de forma síncrona dentro del contexto de la Interrupción de Servicio (ISR) del reloj del sistema (*System Tick Interrupt*), comúnmente configurado a intervalos de 1 ms. Incrementa el contador general `g_app_tick_cnt`. Debido a que corre dentro de una ISR, su código es extremadamente corto y carece de llamadas a APIs bloqueantes de FreeRTOS.
* **`vApplicationStackOverflowHook(xTaskHandle xTask, signed char *pcTaskName)`:**
    Es una rutina de protección crítica frente a fallos de memoria de tiempo de ejecución. Si el *kernel* detecta que el puntero de pila (*Stack Pointer*) de alguna tarea ha sobrepasado los límites asignados durante su creación (mediante el análisis de patrones en la memoria), invoca esta función. El sistema suspende temporalmente las interrupciones mutuas entrando en una sección crítica (`taskENTER_CRITICAL()`) y cuelga intencionalmente la ejecución mediante un `configASSERT( 0 )`. Esto detiene el firmware, evitando comportamientos erráticos o escrituras corruptas en memoria, permitiendo al desarrollador conectar un depurador JTAG/SWD para identificar qué tarea (`pcTaskName`) falló.

---

## 2. Resumen de Flujo de Eventos y Señales

El ciclo dinámico de interacción del sistema puede resumirse en la siguiente secuencia temporal:

```
[ Pulsador Físico ]
        │  (Cambio eléctrico en GPIO)
        ▼
[ task_btn_statechart ] ──(Filtra rebotes por 50ms)──► [ Confirma Presionado ]
                                                              │
                                                   (Llama put_event_task_led)
                                                              │
                                                              ▼
[ task_led_statechart ] ◄──(Detecta flag e inicia 1Hz)─── [ Modifica Estructura ]
        │
        ▼
[ Pin Físico LD2 (LED) ]
```

1.  **Estado de Reposo:** `Task BTN` evalúa el botón en alto (`ST_BTN_UP`). `Task LED` mantiene el LED apagado (`ST_LED_OFF`). El sistema pasa la mayor parte del tiempo incrementando `g_task_idle_cnt` a través del *Idle Hook*.
2.  **Transición de Presionado:** El usuario presiona el botón. `Task BTN` detecta el flanco de bajada, espera 50 ms, reevalúa y confirma la pulsación. Envía el evento `EV_LED_BLINK` mediante la interfaz.
3.  **Activación del Actuador:** `Task LED` lee la bandera activa, pasa al estado `ST_LED_BLINK`, enciende el LED y comienza a conmutar el pin de salida cada 500 ms de forma autónoma.
4.  **Transición de Liberación:** El usuario suelta el botón. `Task BTN` detecta el flanco de subida, filtra el rebote por 50 ms y envía el evento `EV_LED_OFF`. `Task LED` intercepta inmediatamente la bandera, aborta el parpadeo temporizado, apaga el LED y regresa al estado de reposo inicial.