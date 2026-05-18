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

<!-- Pegar aquí la respuesta de Gemini sobre: app.c, task_btn.c, task_led.c, task_led_interface.c, freertos.c -->
