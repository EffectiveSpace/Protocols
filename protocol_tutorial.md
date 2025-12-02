# Туториал по работе с протоколами: Python ↔ Arduino

## Оглавление
1. [Введение](#введение)
2. [Что такое протоколы?](#что-такое-протоколы)
3. [Структура протокола](#структура-протокола)
4. [Примеры использования](#примеры-использования)
5. [Шаг за шагом: Python → Arduino](#шаг-за-шагом-python--arduino)
6. [Задачи для практики](#задачи-для-практики)
7. [Ресурсы](#ресурсы)

---

## Введение

Добро пожаловать в увлекательный мир протоколов! В этом туториале вы научитесь создавать собственные протоколы общения между компьютером (Python) и микроконтроллером (Arduino). Это как изобрести свой собственный способ "разговора" между устройствами!

---

## Что такое протоколы?

Представьте, что вы хотите договориться с другом о тайном сигнале. Например:
- Подмигивание левым глазом = "всё хорошо"
- Подмигивание правым глазом = "опасность"

Это и есть **протокол** — правила общения. В электронике протоколы помогают устройствам обмениваться информацией.

**Примеры из жизни:**
- QR-код — содержит зашифрованную информацию
- Радиоволны — передают музыку по эфиру
- Светофор — сигнализирует остановку/движение

---

## Что такое enum и FSM?

### Enum (перечисление)

**Enum** (сокращение от enumeration - перечисление) — это способ задать набор констант. Это удобно, когда у вас есть фиксированный набор значений. Вместо чисел `0`, `1`, `2` вы можете использовать понятные имена.

**Пример:**
```cpp
#define RED 12
#define YELLOW 11
#define GREEN 10

enum LightState {
  RED_STATE,      // = 0
  YELLOW_STATE,   // = 1
  GREEN_STATE     // = 2
};
```

Теперь вместо `digitalWrite(12, 1);` можно писать `digitalWrite(RED, 1);`, а вместо `data[0] = 1;` можно писать `data[0] = YELLOW_STATE;` — понятнее и удобнее!

### FSM (Finite State Machine - конечный автомат)

**FSM** — это модель поведения, которая в каждый момент времени находится *в одном* из конечного числа состояний. FSM переходит из одного состояния в другое при определённых условиях.

**Пример: Светофор**
- В состоянии `RED_STATE` включён красный свет
- Через 5 секунд переходит в `YELLOW_STATE`
- Через 2 секунды переходит в `GREEN_STATE`
- Через 5 секунд снова в `RED_STATE`

**Код светофора:**
```cpp
enum LightState {
  RED_STATE,
  YELLOW_STATE,
  GREEN_STATE
};

LightState currentState = RED_STATE;

void setup() {
  pinMode(RED, OUTPUT);
  pinMode(YELLOW, OUTPUT);
  pinMode(GREEN, OUTPUT);
}

void loop() {
  switch (currentState) {
    case RED_STATE:
      digitalWrite(RED, 1);
      digitalWrite(YELLOW, 0);
      digitalWrite(GREEN, 0);
      delay(5000);
      currentState = YELLOW_STATE;
      break;

    case YELLOW_STATE:
      digitalWrite(RED, 0);
      digitalWrite(YELLOW, 1);
      digitalWrite(GREEN, 0);
      delay(2000);
      currentState = GREEN_STATE;
      break;

    case GREEN_STATE:
      digitalWrite(RED, 0);
      digitalWrite(YELLOW, 0);
      digitalWrite(GREEN, 1);
      delay(5000);
      currentState = RED_STATE;
      break;
  }
}
```

В протоколах FSM идеально подходит для обработки пакетов: сначала ждём `SYNC1`, потом `SYNC2`, затем длину данных, и т.д.

---

## Структура протокола

Мы будем использовать такую структуру пакета:

```
[SYNC1, SYNC2, LEN_DATA, DATA..., CRC]
```

### Объяснение:
- `SYNC1` (синхронизация 1) — начало пакета
- `SYNC2` (синхронизация 2) — ещё одна проверка начала
- `LEN_DATA` — сколько байт данных в пакете
- `DATA` — сами полезные данные
- `CRC` — контрольная сумма (защита от ошибок)

### Пример пакета:
```
[0xAA, 0x55, 0x03, 0x01, 0x02, 0x03, 0xFF]
```
- `0xAA` — начало
- `0x55` — подтверждение
- `0x03` — 3 байта данных
- `0x01, 0x02, 0x03` — данные
- `0xFF` — контрольная сумма

---

## Примеры использования

### Пример 1: Светофор
Python отправляет команду: "включить светодиод"

### Пример 2: Сенсор температуры
Arduino отправляет температуру в Python

### Пример 3: Игра
Python генерирует случайное число, Arduino отображает его на светодиодной матрице

---

## Шаг за шагом: Python → Arduino

### Python-код (отправка)
```python
import serial
import time

# Открываем порт (проверьте правильный порт в Arduino IDE)
ser = serial.Serial('COM3', 9600)  # Windows: COM3, Linux: /dev/ttyUSB0

# Формируем данные
data = [0x01, 0x02, 0x03]
crc = sum(data) & 0xFF  # простая контрольная сумма

# Собираем пакет
packet = [0xAA, 0x55, len(data)] + data + [crc]

# Отправляем
ser.write(packet)
time.sleep(1)
ser.close()

### Arduino-код (приём) - улучшенный с enum и FSM
```cpp
// Определяем возможные команды через enum
enum CommandType {
  CMD_LED_ON = 0x01,
  CMD_LED_OFF = 0x02,
  CMD_LED_BLINK = 0x03,
  CMD_GET_SENSOR = 0x04
};

// FSM состояния приёма пакета
enum ReceiveState {
  WAIT_SYNC1,
  WAIT_SYNC2,
  WAIT_LEN,
  WAIT_DATA,
  WAIT_CRC
};

const int SYNC1 = 0xAA;
const int SYNC2 = 0x55;

// Переменные для FSM
ReceiveState currentState = WAIT_SYNC1;
byte buffer[256]; // буфер для данных
int dataIndex = 0;
int expectedDataLen = 0;

void setup() {
  Serial.begin(9600);
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  // FSM для обработки входящих байтов
  while (Serial.available()) {
    byte incomingByte = Serial.read();

    switch (currentState) {
      case WAIT_SYNC1:
        if (incomingByte == SYNC1) {
          currentState = WAIT_SYNC2;
        }
        break;

      case WAIT_SYNC2:
        if (incomingByte == SYNC2) {
          currentState = WAIT_LEN;
        } else {
          currentState = WAIT_SYNC1; // ошибка синхронизации
        }
        break;

      case WAIT_LEN:
        expectedDataLen = incomingByte;
        dataIndex = 0;
        currentState = (expectedDataLen > 0) ? WAIT_DATA : WAIT_CRC;
        break;

      case WAIT_DATA:
        buffer[dataIndex] = incomingByte;
        dataIndex++;
        if (dataIndex >= expectedDataLen) {
          currentState = WAIT_CRC;
        }
        break;

      case WAIT_CRC:
        if (check_crc(buffer, expectedDataLen, incomingByte)) {
          // Пакет принят успешно - обрабатываем команду
          processCommand(buffer[0]);
        }
        currentState = WAIT_SYNC1; // возвращаемся к ожиданию следующего пакета
        break;
    }
  }
}

void processCommand(byte command) {
  switch(command) {
    case CMD_LED_ON:
      digitalWrite(LED_BUILTIN, HIGH);
      break;
    case CMD_LED_OFF:
      digitalWrite(LED_BUILTIN, LOW);
      break;
    case CMD_LED_BLINK:
      if (expectedDataLen > 1) {
        blinkLED(buffer[1]); // количество миганий
      }
      break;
    case CMD_GET_SENSOR:
      sendSensorData();
      break;
    default:
      // неизвестная команда
      break;
  }
}

void blinkLED(int times) {
  for(int i = 0; i < times; i++) {
    digitalWrite(LED_BUILTIN, HIGH);
    delay(200);
    digitalWrite(LED_BUILTIN, LOW);
    delay(200);
  }
}

void sendSensorData() {
  // отправляем фиктивные данные сенсора
  byte sensor_data[] = {0xAA, 0x55, 0x01, 25, 0xFF}; // температура 25°C
  Serial.write(sensor_data, 5);
}

bool check_crc(byte data[], int len, byte crc) {
  byte calc_crc = 0;
  for (int i = 0; i < len; i++) {
    calc_crc += data[i];
  }
  return (calc_crc & 0xFF) == crc;
}
```

---

## Задачи для практики

### Задача 1: Управление сервоприводом
**Цель:** Python отправляет угол поворота, Arduino поворачивает сервопривод.

**Шаги:**
1. Добавить сервопривод к Arduino
2. Добавить новую команду в enum в Arduino: `CMD_SERVO_ANGLE`
3. Изменить код: `data[0]` = команда, `data[1]` = угол поворота (0-180)
4. Дополнить Python-код для отправки случайных углов
5. Использовать FSM и switch-case в Arduino для обработки команды

### Задача 2: Датчик температуры
**Цель:** Arduino отправляет температуру в Python, Python отображает её.

**Шаги:**
1. Подключить датчик температуры к Arduino (например, DS18B20)
2. Добавить команду запроса сенсора в enum в Arduino: `CMD_GET_SENSOR`
3. Arduino читает температуру и отправляет через протокол
4. Python принимает и выводит значение
5. Использовать FSM и switch-case в Arduino для обработки команд

### Задача 3: Игра "угадай число"
**Цель:** Python генерирует число, Arduino отображает его на 7-сегментном индикаторе.

**Шаги:**
1. Подключить 7-сегментный индикатор
2. Добавить команду отображения числа в enum в Arduino: `CMD_DISPLAY_NUMBER`
3. Python отправляет случайное число (0-9)
4. Arduino отображает его
5. Использовать FSM и switch-case в Arduino для обработки команд

### Задача 4: Улучшенная защита
**Цель:** Заменить простую CRC на более сложную (например, XOR всех байт).

**Шаги:**
1. Изменить формулу CRC в Python
2. Обновить проверку CRC в Arduino
3. Использовать FSM для корректной обработки состояний приёма в Arduino

### Задача 5: Множественные команды
**Цель:** Добавить команды: "включить светодиод", "выключить светодиод", "мигать" с использованием enum и FSM в Arduino.

**Шаги:**
1. `data[0]` = команда (используя enum в Arduino)
2. `data[1]` = параметр (например, количество миганий)
3. Использовать FSM для обработки состояний приёма пакета в Arduino
4. Использовать switch-case для обработки команд в Arduino

---

## Ресурсы

### Библиотеки:
- [pyserial](https://pypi.org/project/pyserial/) — работа с последовательным портом
- [SoftwareSerial](https://arduinogetstarted.com/tutorials/arduino-softwareserial) — программная реализация UART

### Полезные статьи:
- [Протоколы связи в Arduino](https://alexgyver.ru/lessons/serial/)
- [Примеры протоколов](https://en.wikipedia.org/wiki/Communication_protocol)
- [Протокол UART] (https://repka-pi.ru/docs/94)

### Видео:
- [Объяснение UART](https://rutube.ru/video/74acfe8ba586911aa8af012cd53baf2b/)
- [Работа с протоколами](https://yandex.ru/video/preview/4834464431570622392)

---

Создано для любителей программирования и электроники!  
(c) 2025, Подростковая школа программирования