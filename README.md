# TP_Free_RTOS
`# 000000`

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
1)  le paramètre TOTAL_HEAP_SIZE represente la mémoire alloué au tas pour le fonctionnement de FreeRTOS.
```C
#define configENABLE_FPU                         0
#define configENABLE_MPU                         0

#define configUSE_PREEMPTION                     1
#define configSUPPORT_STATIC_ALLOCATION          0
#define configSUPPORT_DYNAMIC_ALLOCATION         1
#define configUSE_IDLE_HOOK                      0
#define configUSE_TICK_HOOK                      0
#define configCPU_CLOCK_HZ                       ( SystemCoreClock )
#define configTICK_RATE_HZ                       ((TickType_t)1000)
#define configMAX_PRIORITIES                     ( 7 )
#define configMINIMAL_STACK_SIZE                 ((uint16_t)128)
#define configTOTAL_HEAP_SIZE                    ((size_t)3072)
#define configMAX_TASK_NAME_LEN                  ( 16 )
#define configUSE_16_BIT_TICKS                   0
#define configUSE_MUTEXES                        1
#define configQUEUE_REGISTRY_SIZE                8
#define configUSE_PORT_OPTIMISED_TASK_SELECTION  1
/* USER CODE BEGIN MESSAGE_BUFFER_LENGTH_TYPE */
/* Defaults to size_t for backward compatibility, but can be changed
   if lengths will always be less than the number of bytes in a size_t. */
#define configMESSAGE_BUFFER_LENGTH_TYPE         size_t
/* USER CODE END MESSAGE_BUFFER_LENGTH_TYPE */

/* The following flag must be enabled only when using newlib */
#define configUSE_NEWLIB_REENTRANT          1

/* Set the following definitions to 1 to include the API function, or zero
to exclude the API function. */
#define INCLUDE_vTaskPrioritySet             1
#define INCLUDE_uxTaskPriorityGet            1
#define INCLUDE_vTaskDelete                  1
#define INCLUDE_vTaskCleanUpResources        0
#define INCLUDE_vTaskSuspend                 1
#define INCLUDE_vTaskDelayUntil              0
#define INCLUDE_vTaskDelay                   1
#define INCLUDE_xTaskGetSchedulerState       1



/* The lowest interrupt priority that can be used in a call to a "set priority"
function. */
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY   15

/* The highest interrupt priority that can be used by any interrupt service
routine that makes calls to interrupt safe FreeRTOS API functions.  DO NOT CALL
INTERRUPT SAFE FREERTOS API FUNCTIONS FROM ANY INTERRUPT THAT HAS A HIGHER
PRIORITY THAN THIS! (higher priorities are lower numeric values. */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5
```
2) Tache faisant clignoter la LED toutes les 100 ms
```C
#define STACK_SIZE 1000
#define DELAY_1000 1000
#define DELAY_100 100
```
```C
void LED_Init(){
	HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);
}

void task_switch_LED(void * pvParameters){
	int count = 0;
	/* Block for 100ms. */
	const TickType_t xDelay = (TickType_t) DELAY_100 / portTICK_PERIOD_MS;
	for(;;){
		HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
		printf("Count : %d\r\n", count);
		count++;
		vTaskDelay(xDelay);
	}
}
```

The macro portTICK_PERIOD_MS can be used to calculate real time from the tick rate - with the resolution of one tick period.  
In the main loop  
```C
xTaskCreate(task_switch_LED, "Toggle LED", STACK_SIZE, NULL, tskIDLE_PRIORITY, NULL);
vTaskStartScheduler();	// démarre le sheduler = boucle infinie
```

## 1.2 Sémaphores pour la synchronisation
3) taskGive et taskTake
```C
void taskGive(void * pvParameters){
	const TickType_t xDelay = (TickType_t) DELAY_100 / portTICK_PERIOD_MS; //100ms
	for(;;){
		printf("Waiting to give the semaphore\r\n");
		xSemaphoreGive(sem1);
		printf("Semaphore given\r\n");
		vTaskDelay(xDelay);
	}
}

void taskTake(void * pvParameters){
    for(;;){
        printf("Waiting to take the semaphore\r\n");
        xSemaphoreTake(sem1, ((TickType_t) DELAY_1));
        printf("Semaphore taken\r\n");
    }
}
```
In the main loop : 
```C
sem1 = xSemaphoreCreateBinary();
xTaskCreate(taskGive, "Give the semaphore each 100ms", STACK_SIZE, NULL, 1, NULL);
xTaskCreate(taskTake, "Take the semaphore", STACK_SIZE, NULL, 2, NULL);
xTaskCreate(task_switch_LED, "Toggle LED", STACK_SIZE, NULL, tskIDLE_PRIORITY, NULL);

vTaskStartScheduler();	// démarre le sheduler = boucle infinie
```
4) Mécanise de gestion d'erreur
```C
void taskTake(void * pvParameters){

    for(;;){
        printf("Waiting to take the semaphore\r\n");
        if (xSemaphoreTake(sem1, ((TickType_t) DELAY_1)) == pdTRUE){
        	printf("Semaphore taken\r\n");
    	}
    	else {
			printf("Failed to take semaphore, reset software\r\n");
			NVIC_SystemReset(); // Reset the uC
        }
    }
}
```
5) Ajout de 100 ms à chaque itération pour valider la gestion d'erreur
```C
void taskGive(void * pvParameters){
	TickType_t xDelay = ((TickType_t) DELAY_100 / portTICK_PERIOD_MS; //100ms
	for(;;){
		printf("Waiting to give the semaphore, Delay = %u\r\n", (unsigned int)xDelay);
		xSemaphoreGive(sem1);
		printf("Semaphore given\r\n");
		vTaskDelay(xDelay);
		xDelay += 100;
	}
}
```
6) Changment de priorités
- Affichage pour taskeTake prioritaire sur TaskGive : 
```C
==============START==============
Waiting to take the semaphore
Waiting to give the semaphore, Delay = 100
Semaphore taken
Waiting to take the semaphore
Semaphore given
Waiting to give the semaphore, Delay = 200
Semaphore taken
Waiting to take the semaphore
...
...
Waiting to take the semaphore
Semaphore given
Waiting to give the semaphore, Delay = 1000
Semaphore taken
Waiting to take the semaphore
Semaphore given
Failed to take semaphore, reset software
```

Ici, taskTake à la priorité donc on commence par attendre de prendre le sémaphore et on affiche "Waiting to take the semaphore".  
Celui-ci n'a pas encore été donné donc taskTake passe en attente et taskGive s'active et on attends de donner le sémaphore, on affiche "Waiting to give the sempahore".  
Le sémaphore est donné par taskGive, donc taskGive se bloque et taskTake est immédiatement débloquée pour prendre le sémaphore "Semaphore taken" (car taskTake est plus prioritaire). 
taskGive n'a donc pas le temps d'afficher "Semaphore given".  
La boucle while de taskTake recommence et on attends de nouveau de prendre le sémaphore "Waiting to take the semaphore", donc taskTake passe en attente.  
taskGive se débloque et peut enfin afficher "Sémaphore given".
La boucle while de task give recommence "Waiting to give the semaphore".  
Puis taskGive donne le sémaphore et débloque taskTake "Semaphore taken".  
Et ainsi de suite... 
  
- Affichage pour taskeGive prioritaire sur TaskTake :  
```C
Waiting to give the semaphore, Delay = 100
Semaphore given
Waiting to take the semaphore
Semaphore taken
Waiting to take the semaphore
Waiting to give the semaphore, Delay = 200
Semaphore given
Semaphore taken
Waiting to take the semaphore
Waiting to give the semaphore, Delay = 300
...
...
Waiting to give the semaphore, Delay = 1100
Semaphore given
Semaphore taken
Waiting to take the semaphore
Failed to take semaphore, reset software
```
Dans ce cas task give est prioritaire donc on commence par attendre de donner le semaphore "waiting to give the semaphore", puis on le donne.
La tache ne bloque pas tout de suite car taskGive est prioritaire, donc la boucle continue et on affiche "semaphore given".  
taskGive se bloque à cause du vTaskDelay, et on taskTake se lance, on attends de prendre le semaphore " Waiting to the take the semaphore", puis on le prend car il a déjà été donné auparavant, puis on affiche "semaphore taken". 
la boucle while de taskTake recommence et on essaye de prendre le semaphore "waiting to take the semaphore",  
mais celui-ci n'a pas encore été donné donc taskTake se bloque et on passe dans taskGive. On attends de donner le semaphore "Waiting to give the semaphore", puis on le donne mais on reste dans taskGive car prioritaire.  
Ensuite vTaskDelay bloque tasgive, et taskTake se lance.
et ainsi de suite... 
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










