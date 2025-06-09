# Práctica 4: Sistemas Operativos en Tiempo Real

En esta práctica se trabaja con **sistemas operativos en tiempo real** (RTOS) utilizando FreeRTOS en un ESP32. Se implementan tareas concurrentes, sincronización con semáforos, gestión de interrupciones, colas y desarrollo de aplicaciones interactivas tanto físicas como web.

---

## Ejercicio Práctico 1

```cpp
#include <Arduino.h>

void anotherTask(void * parameter) {
  for(;;) {
    Serial.println("this is another Task");
    delay(1000);
  }
  vTaskDelete(NULL);  // Esta línea nunca se ejecutará, pero es buena práctica incluirla
}

void setup() {
  Serial.begin(115200);

  xTaskCreate(
    anotherTask,    // Función de la tarea
    "another Task", // Nombre de la tarea
    10000,          // Tamaño de la pila en bytes
    NULL,           // Parámetro de la tarea
    1,              // Prioridad de la tarea
    NULL            // Manejador de la tarea
  );
}

void loop() {
  Serial.println("this is ESP32 Task");
  delay(1000);
}
```
Descripción

-La tarea principal imprime: this is ESP32 Task

-La tarea secundaria imprime: this is another Task

-Ambas tareas se ejecutan de forma concurrente cada segundo.

---

## Ejercicio Práctico 2

Este ejercicio controla el encendido y apagado de un LED mediante dos tareas sincronizadas con un semáforo binario.

```cpp
#include <Arduino.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"

const int ledPin = 2;
SemaphoreHandle_t xSemaphore;

void TaskTurnOn(void *parameter) {
  for (;;) {
    if (xSemaphoreTake(xSemaphore, portMAX_DELAY)) {
      digitalWrite(ledPin, HIGH);
      Serial.println("LED ENCENDIDO");
      delay(1000);
      xSemaphoreGive(xSemaphore);
    }
    vTaskDelay(100 / portTICK_PERIOD_MS);
  }
}

void TaskTurnOff(void *parameter) {
  for (;;) {
    if (xSemaphoreTake(xSemaphore, portMAX_DELAY)) {
      digitalWrite(ledPin, LOW);
      Serial.println("LED APAGADO");
      delay(1000);
      xSemaphoreGive(xSemaphore);
    }
    vTaskDelay(100 / portTICK_PERIOD_MS);
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);

  xSemaphore = xSemaphoreCreateBinary();

  xTaskCreate(TaskTurnOn, "Encender LED", 1000, NULL, 1, NULL);
  xTaskCreate(TaskTurnOff, "Apagar LED", 1000, NULL, 1, NULL);

  xSemaphoreGive(xSemaphore);
}

void loop() {}
```

Descripción

-Dos tareas controlan el encendido y apagado del LED usando un semáforo.

-Cada tarea alterna el control cada segundo.

-Ejemplo de salida:

```Serial Monitor
LED ENCENDIDO
LED APAGADO
LED ENCENDIDO

```

---

##Ejercicio de Mejora: Reloj Digital Multitarea

Este ejercicio implementa un reloj digital ajustable mediante botones, con visualización en serie y control de LEDs.

```cpp
#include <Arduino.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

QueueHandle_t colaHoras;
QueueHandle_t colaMinutos;
QueueHandle_t colaSegundos;

const int buttonPin1 = 4; // Botón para cambiar de modo
const int buttonPin2 = 5; // Botón para incrementar hora o minuto
const int ledPin = 2;     // LED indicador del modo

int modo = 0; // 0: reloj, 1: ajustar horas, 2: ajustar minutos

void taskActualizarReloj(void *pvParameters) {
  int horas = 0, minutos = 0, segundos = 0;

  for (;;) {
    if (modo == 0) {
      segundos++;
      if (segundos >= 60) {
        segundos = 0;
        minutos++;
        if (minutos >= 60) {
          minutos = 0;
          horas++;
          if (horas >= 24) {
            horas = 0;
          }
        }
      }
    }

    xQueueOverwrite(colaHoras, &horas);
    xQueueOverwrite(colaMinutos, &minutos);
    xQueueOverwrite(colaSegundos, &segundos);

    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

void taskLeerBotones(void *pvParameters) {
  bool ultimoEstadoBtn1 = HIGH;
  bool ultimoEstadoBtn2 = HIGH;

  for (;;) {
    bool estadoBtn1 = digitalRead(buttonPin1);
    bool estadoBtn2 = digitalRead(buttonPin2);

    if (estadoBtn1 == LOW && ultimoEstadoBtn1 == HIGH) {
      modo = (modo + 1) % 3; // Cambiar modo
      Serial.print("Modo cambiado a: ");
      Serial.println(modo);
      delay(300); // Anti-rebote
    }

    if (estadoBtn2 == LOW && ultimoEstadoBtn2 == HIGH) {
      if (modo == 1) { // Ajustar horas
        int horas;
        xQueuePeek(colaHoras, &horas, 0);
        horas = (horas + 1) % 24;
        xQueueOverwrite(colaHoras, &horas);
      } else if (modo == 2) { // Ajustar minutos
        int minutos;
        xQueuePeek(colaMinutos, &minutos, 0);
        minutos = (minutos + 1) % 60;
        xQueueOverwrite(colaMinutos, &minutos);
      }
      delay(300); // Anti-rebote
    }

    ultimoEstadoBtn1 = estadoBtn1;
    ultimoEstadoBtn2 = estadoBtn2;

    vTaskDelay(pdMS_TO_TICKS(50));
  }
}

void taskMostrarHora(void *pvParameters) {
  int horas, minutos, segundos;

  for (;;) {
    xQueuePeek(colaHoras, &horas, portMAX_DELAY);
    xQueuePeek(colaMinutos, &minutos, portMAX_DELAY);
    xQueuePeek(colaSegundos, &segundos, portMAX_DELAY);

    Serial.print("Hora: ");
    if (horas < 10) Serial.print("0");
    Serial.print(horas);
    Serial.print(":");
    if (minutos < 10) Serial.print("0");
    Serial.print(minutos);
    Serial.print(":");
    if (segundos < 10) Serial.print("0");
    Serial.print(segundos);
    Serial.print("  | Modo: ");
    Serial.println(modo);

    digitalWrite(ledPin, modo == 0 ? LOW : HIGH);

    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

void setup() {
  Serial.begin(115200);

  pinMode(buttonPin1, INPUT_PULLUP);
  pinMode(buttonPin2, INPUT_PULLUP);
  pinMode(ledPin, OUTPUT);

  colaHoras = xQueueCreate(1, sizeof(int));
  colaMinutos = xQueueCreate(1, sizeof(int));
  colaSegundos = xQueueCreate(1, sizeof(int));

  int cero = 0;
  xQueueOverwrite(colaHoras, &cero);
  xQueueOverwrite(colaMinutos, &cero);
  xQueueOverwrite(colaSegundos, &cero);

  xTaskCreate(taskActualizarReloj, "Actualizar Reloj", 2048, NULL, 1, NULL);
  xTaskCreate(taskLeerBotones, "Leer Botones", 2048, NULL, 1, NULL);
  xTaskCreate(taskMostrarHora, "Mostrar Hora", 2048, NULL, 1, NULL);
}

void loop() {
  // No se utiliza
}
```

###Descripción
-Tareas:

  -Avance del reloj

  -Lectura de botones

  -Actualización de display

  -Control de LEDs

-Botones:

  -Cambiar modo (reloj / ajuste horas / ajuste minutos)

  -Incrementar valor según el modo

-Salida en serie:

```Serial Monitor
Hora: 12:34:56  | Modo: 0
Hora: 12:34:57  | Modo: 0
Hora: 12:34:58  | Modo: 0
```
```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"

const char* ssid = "ESP32_AP";
const char* password = "12345678";

WebServer server(80);

const int ledPins[] = {2, 4, 5};
const int buttonPins[] = {15, 18, 19};

int currentLed = -1;
int score = 0;
int gameDuration = 30;
int gameTimeLeft = 0;
bool gameRunning = false;

SemaphoreHandle_t gameSemaphore;

void startGame() {
  if (!gameRunning) {
    gameRunning = true;
    score = 0;
    gameTimeLeft = gameDuration;

    xSemaphoreGive(gameSemaphore);

    Serial.println("Juego iniciado");
    server.send(200, "text/plain", "Juego iniciado");
  } else {
    server.send(200, "text/plain", "El juego ya está en curso");
  }
}

void stopGame() {
  if (gameRunning) {
    gameRunning = false;
    Serial.println("Juego detenido");
    server.send(200, "text/plain", "Juego detenido");
  } else {
    server.send(200, "text/plain", "El juego no está en curso");
  }
}

void setDifficulty() {
  if (server.hasArg("level")) {
    String level = server.arg("level");
    if (level == "easy") {
      gameDuration = 30;
    } else if (level == "medium") {
      gameDuration = 20;
    } else if (level == "hard") {
      gameDuration = 10;
    }
    server.send(200, "text/plain", "Dificultad cambiada a " + level);
  } else {
    server.send(400, "text/plain", "Nivel de dificultad no especificado");
  }
}

void IRAM_ATTR buttonPressed() {
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  if (gameRunning && currentLed != -1) {
    score++;
    Serial.println("¡Acierto!");
    Serial.print("Puntuación: ");
    Serial.println(score);
    xHigherPriorityTaskWoken = pdTRUE;
  }
  if (xHigherPriorityTaskWoken) {
    portYIELD_FROM_ISR();
  }
}

void taskLedController(void *pvParameters) {
  for (;;) {
    if (xSemaphoreTake(gameSemaphore, portMAX_DELAY)) {
      while (gameRunning && gameTimeLeft > 0) {
        int previousLed = currentLed;
        do {
          currentLed = random(0, 3);
        } while (currentLed == previousLed);

        for (int i = 0; i < 3; i++) {
          digitalWrite(ledPins[i], i == currentLed ? HIGH : LOW);
        }

        Serial.print("LED activo: ");
        Serial.println(currentLed);

        vTaskDelay(pdMS_TO_TICKS(1000));
      }

      for (int i = 0; i < 3; i++) {
        digitalWrite(ledPins[i], LOW);
      }

      Serial.println("Juego terminado");
      Serial.print("Puntuación final: ");
      Serial.println(score);

      gameRunning = false;
    }
  }
}

void taskGameTimer(void *pvParameters) {
  for (;;) {
    if (gameRunning && gameTimeLeft > 0) {
      vTaskDelay(pdMS_TO_TICKS(1000));
      gameTimeLeft--;

      Serial.print("Tiempo restante: ");
      Serial.println(gameTimeLeft);
    } else {
      vTaskDelay(pdMS_TO_TICKS(500));
    }
  }
}

void setup() {
  Serial.begin(115200);
  randomSeed(analogRead(0));

  for (int i = 0; i < 3; i++) {
    pinMode(ledPins[i], OUTPUT);
    pinMode(buttonPins[i], INPUT_PULLUP);
    attachInterrupt(digitalPinToInterrupt(buttonPins[i]), buttonPressed, FALLING);
  }

  WiFi.softAP(ssid, password);
  Serial.print("Dirección IP: ");
  Serial.println(WiFi.softAPIP());

  server.on("/start", startGame);
  server.on("/stop", stopGame);
  server.on("/difficulty", setDifficulty);
  server.begin();

  gameSemaphore = xSemaphoreCreateBinary();

  xTaskCreate(taskLedController, "Control LEDs", 4096, NULL, 1, NULL);
  xTaskCreate(taskGameTimer, "Temporizador", 2048, NULL, 1, NULL);
}

void loop() {
  server.handleClient();
}
```
###Descripción
-El ESP32 crea un punto de acceso WiFi.

-El servidor web permite iniciar/detener el juego y cambiar la dificultad.

-Tareas FreeRTOS:

  -Activación aleatoria de LEDs.

  -Lectura de botones (con interrupciones).

  -Control de tiempo del juego.

-El jugador debe presionar el botón correspondiente al LED encendido.

-La puntuación y el tiempo restante se muestran por el puerto serie.

-Ejemplo de salida:
```cpp
Dirección IP: 192.168.4.1
Juego iniciado
Puntuación: 6
Tiempo restante: 24
```
