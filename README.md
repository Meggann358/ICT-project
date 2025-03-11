#include <U8g2lib.h>      // OLED display library
#include <TimeLib.h>      // Time manipulation


// OLED Display (SSD1306 128x64 I2C)
U8G2_SSD1306_128X64_NONAME_1_HW_I2C u8g2(U8G2_R0);


// Pin Definitions
const uint8_t BTN_MODE = 2;    // Cycle modes
const uint8_t BTN_UP = 3;      // Increment/Start
const uint8_t BTN_DOWN = 4;    // Decrement/Reset
const uint8_t BTN_TOGGLE = 5;  // Toggle display


// Feature Modes
enum Mode { CLOCK, STOPWATCH };
Mode currentMode = CLOCK;


// Clock Settings
enum TimeField { HOUR, MINUTE, SECOND, DAY, MONTH, YEAR };
TimeField selectedField = HOUR;
bool analogDisplay = false;


// Stopwatch
bool isRunning = false;
uint32_t startTime = 0;
uint32_t pausedTime = 0;


// Button States
uint32_t lastDebounce = 0;
const uint16_t DEBOUNCE_DELAY = 50;


void setup() {
  pinMode(BTN_MODE, INPUT_PULLUP);
  pinMode(BTN_UP, INPUT_PULLUP);
  pinMode(BTN_DOWN, INPUT_PULLUP);
  pinMode(BTN_TOGGLE, INPUT_PULLUP);
  
  u8g2.begin();
  setTime(15, 11, 24, 27, 9, 2023); // Initial time
}


void loop() {
  handleInput();
  updateDisplay();
  delay(50);
}


void handleInput() {
  if (millis() - lastDebounce < DEBOUNCE_DELAY) return;
  
  if (!digitalRead(BTN_MODE)) {
    currentMode = (currentMode == CLOCK) ? STOPWATCH : CLOCK;
    lastDebounce = millis();
    return;
  }


  if (currentMode == CLOCK) {
    handleClockInput();
  } else {
    handleStopwatchInput();
  }
}


void handleClockInput() {
  static uint32_t holdStart = 0;
  
  if (!digitalRead(BTN_TOGGLE)) {
    analogDisplay = !analogDisplay;
    lastDebounce = millis();
  }
  
  if (!digitalRead(BTN_UP)) {
    if (millis() - holdStart > 1000) {
      cycleField();
      holdStart = millis();
    } else {
      adjustTime(true);
      lastDebounce = millis();
    }
  } else {
    holdStart = millis();
  }


  if (!digitalRead(BTN_DOWN)) {
    adjustTime(false);
    lastDebounce = millis();
  }
}


void handleStopwatchInput() {
  if (!digitalRead(BTN_UP)) {
    isRunning = !isRunning;
    if (isRunning) startTime = millis();
    else pausedTime += millis() - startTime;
    lastDebounce = millis();
  }


  if (!digitalRead(BTN_DOWN) && !isRunning) {
    pausedTime = 0;
    lastDebounce = millis();
  }
}


void cycleField() {
  selectedField = static_cast<TimeField>((selectedField + 1) % 6);
}


void adjustTime(bool increment) {
  tmElements_t tm;
  breakTime(now(), tm);


  switch (selectedField) {
    case HOUR:
      tm.Hour = constrain(tm.Hour + (increment ? 1 : -1), 0, 23);
      break;
    case MINUTE:
      tm.Minute = constrain(tm.Minute + (increment ? 1 : -1), 0, 59);
      break;
    case SECOND:
      tm.Second = constrain(tm.Second + (increment ? 1 : -1), 0, 59);
      break;
    case DAY: {
      uint8_t maxDay = daysInMonth(tm.Month, tm.Year + 1970);
      tm.Day = constrain(tm.Day + (increment ? 1 : -1), 1, maxDay);
      break;
    }
    case MONTH:
      tm.Month = constrain(tm.Month + (increment ? 1 : -1), 1, 12);
      break;
    case YEAR: {
      int year = tm.Year + 1970 + (increment ? 1 : -1);
      year = constrain(year, 2000, 2099);
      tm.Year = year - 1970;
      break;
    }
  }
  setTime(makeTime(tm));
}


void updateDisplay() {
  u8g2.firstPage();
  do {
    if (currentMode == CLOCK) {
      analogDisplay ? drawAnalogClock() : drawDigitalClock();
    } else {
      drawStopwatch();
    }
  } while (u8g2.nextPage());
}


void drawDigitalClock() {
  char buf[9];
  sprintf(buf, "%02d:%02d:%02d", hour(), minute(), second());
  
  u8g2.setFont(u8g2_font_profont29_tf);
  u8g2.drawStr(0, 30, buf);
  
  sprintf(buf, "%02d/%02d/%04d", day(), month(), year());
  u8g2.setFont(u8g2_font_5x8_tr);
  u8g2.drawStr(0, 50, buf);
}


void drawAnalogClock() {
  const uint8_t cx = 64, cy = 32, r = 30;
  u8g2.drawCircle(cx, cy, r);


  // Draw hands
  drawHand(cx, cy, hour() * 30 + minute() * 0.5, r * 0.6, 3);   // Hour
  drawHand(cx, cy, minute() * 6 + second() * 0.1, r * 0.8, 2);  // Minute
  drawHand(cx, cy, second() * 6, r * 0.9, 1);                   // Second


  // Date display
  char buf[11];
  sprintf(buf, "%02d/%02d/%04d", day(), month(), year());
  u8g2.setFont(u8g2_font_5x8_tr);
  u8g2.drawStr(0, 63, buf);
}


void drawHand(uint8_t x, uint8_t y, float angle, uint8_t length, uint8_t width) {
  angle = radians(angle - 90);
  float x2 = x + cos(angle) * length;
  float y2 = y + sin(angle) * length;
  u8g2.drawLine(x, y, x2, y2);
}


void drawStopwatch() {
  uint32_t elapsed = pausedTime + (isRunning ? millis() - startTime : 0);
  uint16_t mins = elapsed / 60000;
  uint8_t secs = (elapsed % 60000) / 1000;
  uint16_t ms = elapsed % 1000;


  char buf[16];
  sprintf(buf, "%02d:%02d.%03d", mins, secs, ms);
  
  u8g2.setFont(u8g2_font_profont29_tf);
  u8g2.drawStr(0, 30, buf);
  
  u8g2.setFont(u8g2_font_5x8_tr);
  u8g2.drawStr(0, 50, isRunning ? "RUNNING" : "PAUSED");
}
