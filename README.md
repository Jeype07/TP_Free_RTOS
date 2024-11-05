# TP_Free_RTOS

L’objectif de ce TP est de mettre en place quelques applications sous FreeRTOS
en utilisant la carte NUCLEO-G431RB conçue autour du STM32G431RBT6.

Rajouter les lignes suivantes
```C
int __io_putchar(int ch) {
  HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
  return ch;
}
```
Cela permet d'utiliser le printf.

# 1 FreeRTOS, tâches et sémaphores
## 1.1 Tâche simple
1) Total

2)

## 1.2 Sémaphores pour la synchronisation
3)
4)
5)
6)
## 1.3 Notification
7)
## 1.4 Queues
8)
## 1.5 Réentrance et exclusion mutuelle
9)
```C
#define STACK_SIZE 256
#define TASK1_PRIORITY 1
#define TASK2_PRIORITY 2
#define TASK1_DELAY 1
#define TASK2_DELAY 2
```
```C
ret = xTaskCreate(task_bug, "Tache 1", STACK_SIZE, \
  (void *) TASK1_DELAY, TASK1_PRIORITY, NULL);
configASSERT(pdPASS == ret);
ret = xTaskCreate(task_bug, "Tache 2", STACK_SIZE, \
  (void *) TASK2_DELAY, TASK2_PRIORITY, NULL);
configASSERT(pdPASS == ret);
```
```c
void task_bug(void * pvParameters)
{
  int delay = (int) pvParameters;
  for(;;)
  {
    printf("Je suis %s et je m'endors pour \
      %d ticks\r\n", pcTaskGetName(NULL), delay);
    vTaskDelay(delay);
  }
}
```










