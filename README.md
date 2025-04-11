# FreeRTOS Lab1

## Requirement
LED-task will have two states (S1, S2)
* S1: First, the Red LED lights up for 1 second, followed by the Orange LED 
lighting up for 1 second (with the Red LED turned off), then the Green LED 
lights up for 1 second (with both the Red and Orange LEDs turned off). This 
sequence repeats, cycling through the Red, Orange, and Green LEDs.
* S2: Only Orange LED is blinking (2 second ON, 2 second OFF, …).

Button-task: If the button is pressed, the LED-task will switch to another state 
(S1→S2).

## Port

* User LD3: orange LED is a user LED connected to the I/O PD13 of the `STM32F407VGT6`
* User LD4: green LED is a user LED connected to the I/O PD12 of the `STM32F407VGT6`
* User LD5: red LED is a user LED connected to the I/O PD14 of the `STM32F407VGT6`
* B1 USER: User and Wake-up buttons are connnected to the I/O PA0 of the `STM32F407VGT6`

### Pin Assignment
* PD12 is set as Green LED, PD13 is set as Orange LED, PD14 is set as Red LED
* PA0 is set as Blue Button

## Task

### How you create task in FreeRTOS

```C
BaseType_t xTaskCreate(TaskFunction_t pxTaskCode, const char * const pcName, const configSTACK_DEPTH_TYPE usStackDepth, void * const pvParameters, UbaseType_t uxPriority, TaskHandle_t * const pxCreatedTask);
```

| argument | Description |
|----------|-------------|
| pvTaskCode | Pointer to the task entry function. |
| pcName | A descriptive name for the task. |
| usStackDepth | The number of words(not bytes) to allocate for use as the task's stack. |
| pvParameters | The value that will passed into the created task as the task's parameter. |
| uxPriority | The priority at which the created task will execute. |
| pxCreatedTask | Used to pass a handle to the created task out of the xTaskCreate() function, pxCreatedTask is optional and can be set to `NULL`. |

e.g.
```C
xTaskCreate(LED_Handler, "LEDTask_APP", 128, NULL, 1, NULL);
```

### How you schedule task in FreeRTOS

`vTaskStartScheduler()`
* Start the RTOS scheduler, giving the kernel control over task execution
* The idle task and optionally the timer daemon task are created automatically when the scheduler starts
* It only returns if there is not enough heap memory to create these tasks
* RTOS demo projects typically call it in the main function of main.c

e.g.
```C
int main()
{
    xTaskCreate(LED_Handler, "LEDTask_APP", 128, NULL, 1, NULL);

    vTaskStartScheduler();

    return 0;
}
```

## Queue

### How you create queue in FreeRTOS

```C
QueueHandle_t xQueueCreate(UBaseType_t uxQueueLength, UBaseType_t uxItemSize);
```

| argument | Description |
|----------|-------------|
| uxQueueLength | The maximum number of items the queue can hold at any one time. |
| uxItemSize | The size, in bytes, required to hold each item in the queue. |

e.g.
```C
MsgQueue = xQueueCreate(10, sizeof(uint32_t));
```

### How you perform send & receive in FreeRTOS

```C
BaseType_t xQueueSend(QueueHandle_t xQueue, const void * pvItemQueue, TickType_t xTicksToWait);
```

| argument | Description |
|----------|-------------|
| xQueue | The handle to the queue on which the items is to be posted. |
| pvItemToQueue | The size, in bytes, required to hold each item in the queue. |
| xTicksToWait | The maximum amount of time the task should block waiting for space to become available on the queue, should it already be full. |

e.g.
```C
void SenderTask(void *pvParameters)
{
    uint32_t msg = 42;
    while(1) {
        xQueueSend(MsgQueue, &msg, portMAX_DELAY);
        taskYIELD();  // Immediately give CPU time to another task
    }
}
```

```C
BaseType_t xQueueReceive(QueueHandle_t xQueue, void * pvBuffer, TickType_t xTicksToWait);
```
| argument | Description |
|----------|-------------|
| xQueue | The handle to the queue on which the item is to be received. |
| pvBuffer | Pointer to the buffer into which the received item will be copied. |
| xTicksToWait | The maximum amount of time the task should block waiting for an item to receive should the queue be empty at the time of the call. |

e.g.
```C
void LEDTask(void *pvParameters)
{
    uint32_t receiveValue;
    while(1) {
        if (xQueueReceive(MsgQueue, &receivedValue, portMAX_DELAY) == pdPASS) {
            HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0); // Toggle LED when button press is received
        }
    }
}
```

## Button Issues

Mechanical switches bounce when pressed, causing multiple rapid detections instead of a single press. This can lead to unintended multiple task executions.

### Solution
Use software debounce by adding a delay after detecting a button press  
e.g.  
```C
void Button_Handler(void *pvParameters)
{
    uint32_t buttonState = 0;
    uint32_t LEDState = 0;
    while(1) {
        if(HAL_GPIO_ReadPin(BLUE_BUTTON_GPIO_Port, GPIO_PIN_0)) {
            vTaskDelay(10); // Delays the task, allowing other tasks to run
            while(HAL_GPIO_ReadPin(BLUE_BUTTON_GPIO_Port, GPIO_PIN_0)) {;}

            /* ... */
        }
    }
}
```
