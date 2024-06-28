# Receiving-the-RPM-and-Temperature-Data-from-Google-FireBase

### Table of Contents

1. [Library Inclusions and Definitions](#1-library-inclusions-and-definitions)
2. [CAN Bus and Display Setup](#2-can-bus-and-display-setup)
3. [Variables for Data Storage](#3-variables-for-data-storage)
4. [Functions for Displaying Data](#4-functions-for-displaying-data)
   - [Drawing Temperatures](#drawing-temperatures)
   - [Drawing RPM](#drawing-rpm)
5. [Setup Function](#5-setup-function)
6. [Main Loop](#6-main-loop)

### 1. Library Inclusions and Definitions

```cpp
#include <mcp2515.h>
#include <SPI.h>
#include <Wire.h>
#include <U8g2lib.h>
```

### 2. CAN Bus and Display Setup

```cpp
#define SENDER_CAN_ID 0x124
const int spiCS = D8;

MCP2515 mcp2515Receiver(spiCS);
struct can_frame canMsg;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define MAX_TEMP_VALUES 10

U8G2_SSD1306_128X64_NONAME_F_HW_I2C display(U8G2_R0, /* reset=*/ U8X8_PIN_NONE);
```

### 3. Variables for Data Storage

```cpp
float lastReceivedRPM = 0.0;
float tempValues[MAX_TEMP_VALUES];
int tempCount = 0;
```

### 4. Functions for Displaying Data

#### Drawing Temperatures

```cpp
void drawTemps() {
  display.setFont(u8g2_font_profont12_mf);
  int xPosition = 2;
  display.setCursor(xPosition, 60);

  for (int i = 0; i < tempCount && i < 2; ++i) { // Adjusted to limit to two temperatures
    display.print(tempValues[i], 2);
    if (i < tempCount - 1 && i < 1) { // Adjusted to limit to one separator
      display.print("  ");
    }
  }
}
```

#### Drawing RPM

```cpp
void draw() {
  display.setFont(u8g2_font_profont29_mf);
  char rpmCharArray[10];
  dtostrf(lastReceivedRPM, 1, 0, rpmCharArray);
  int rpmTextWidth = display.getStrWidth(rpmCharArray);
  int xPosition = (SCREEN_WIDTH - rpmTextWidth) / 2;
  int yPosition = 30;
  display.setCursor(xPosition, yPosition);
  display.print(rpmCharArray);
}
```

### 5. Setup Function

```cpp
void setup() {
  Serial.begin(9600);
  SPI.begin();

  display.begin();
  
  Serial.println("Initializing MCP2515 Receiver...");
  mcp2515Receiver.reset();
  mcp2515Receiver.setBitrate(CAN_500KBPS, MCP_8MHZ);
  mcp2515Receiver.setNormalMode();
  Serial.println("MCP2515 Receiver Initialized Successfully!");
}
```

### 6. Main Loop

```cpp
void loop() {
  if (mcp2515Receiver.readMessage(&canMsg) == MCP2515::ERROR_OK) {
    if (canMsg.can_id == SENDER_CAN_ID) {
      if (canMsg.can_dlc == 8) {
        // Extract RPM and speed from CAN message
        float receivedRPM, receivedSpeed;
        memcpy(&receivedRPM, &canMsg.data[0], sizeof(receivedRPM));
        memcpy(&receivedSpeed, &canMsg.data[4], sizeof(receivedSpeed));

        // Print received RPM and speed
        Serial.print("Received RPM: ");
        Serial.print(receivedRPM);
        Serial.print(" RPM, Speed: ");
        Serial.print(receivedSpeed);
        Serial.println(" m/s");

        // Update the last received RPM value
        lastReceivedRPM = receivedRPM;

        // Display on OLED
        display.clearBuffer();
        draw();
        drawTemps();
        display.sendBuffer();
      } else if (canMsg.can_dlc == 4) {
        // Extract temperature from CAN message
        float temp;
        memcpy(&temp, &canMsg.data[0], sizeof(temp));

        // Print received temperature
        Serial.print("Received Temperature: ");
        Serial.print(temp);
        Serial.println(" °C");

        // Store temperature
        if (tempCount < MAX_TEMP_VALUES) {
          tempValues[tempCount] = temp;
          tempCount++;
        } else {
          for (int i = 0; i < MAX_TEMP_VALUES - 1; ++i) {
            tempValues[i] = tempValues[i + 1];
          }
          tempValues[MAX_TEMP_VALUES - 1] = temp;
        }

        // Display on OLED
        display.clearBuffer();
        draw();
        drawTemps();
        
        // Calculate average temperature
        float avgTemp = (tempValues[0] + tempValues[1]) / 2.0;
        
        // Display average temperature in the bottom middle
        char avgTempCharArray[10];
        dtostrf(avgTemp, 1, 2, avgTempCharArray);
        int avgTempWidth = display.getStrWidth(avgTempCharArray);
        int avgTempX = (SCREEN_WIDTH - avgTempWidth) / 2;
        display.setCursor(avgTempX, 60);
        display.print(avgTempCharArray);
        
        display.sendBuffer();
      }
    }
  }
}
```

### Summary

This README provides a structured overview of how the code integrates CAN bus communication via MCP2515, data processing for RPM and temperature readings, and OLED display using U8g2. Each section is linked in the table of contents for easy navigation. Adjust the headings and content as needed to fit your specific documentation style and project requirements.
