# Project
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <math.h>
#include <SoftwareSerial.h>
#include "DHT.h"

#define DHTPIN 7       // Digital pin connected to the DHT sensor
#define DHTTYPE DHT11  // DHT 11

// I2C LCD declaration
LiquidCrystal_I2C lcd(0x27, 2, 1, 0, 4, 5, 6, 7, 3, POSITIVE);

SoftwareSerial mySerial(2, 3);
DHT dht(DHTPIN, DHTTYPE);

#define TIMEOUT 2000

float calibration_value = 21.34;
float phval = 0.0;

unsigned long int avgval;
int buffer_arr[10], temp;

char tbuf[16];
int cntr = 0;

uint32_t timeout = millis();

char webpage[500];

float t, h;


// ---------------------------------------------------------
// Function to read pH value
// ---------------------------------------------------------
void read_ph(void)
{
    for (int i = 0; i < 10; i++)
    {
        buffer_arr[i] = analogRead(A0);
        delay(30);
    }

    // Sort the readings
    for (int i = 0; i < 9; i++)
    {
        for (int j = i + 1; j < 10; j++)
        {
            if (buffer_arr[i] > buffer_arr[j])
            {
                temp = buffer_arr[i];
                buffer_arr[i] = buffer_arr[j];
                buffer_arr[j] = temp;
            }
        }
    }

    // Calculate average value
    avgval = 0;

    for (int i = 2; i < 8; i++)
    {
        avgval += buffer_arr[i];
    }

    float volt = (float)avgval * 5.0 / 1024 / 6;

    phval = -5.70 * volt + calibration_value;
    phval += 7;

    Serial.print("pH Val: ");
    Serial.println(phval);
}


// ---------------------------------------------------------
// Function to display values on LCD
// ---------------------------------------------------------
void title(void)
{
    int val = 0;

    // Display pH value
    lcd.setCursor(0, 0);

    val = phval * 100;

    sprintf(
        tbuf,
        "PH: %02d.%02d",
        val / 100,
        abs(val % 100)
    );

    lcd.print(tbuf);

    // Display temperature
    lcd.setCursor(0, 1);

    val = t * 100;

    sprintf(
        tbuf,
        "TMP: %02d.%02d C ",
        val / 100,
        abs(val % 100)
    );

    lcd.print(tbuf);
}


// ---------------------------------------------------------
// Global variables
// ---------------------------------------------------------
boolean update_lcd;


// ---------------------------------------------------------
// Setup
// ---------------------------------------------------------
void setup()
{
    int l;

    Serial.begin(115200);
    mySerial.begin(115200);

    update_lcd = 0;

    lcd.begin(16, 2);
    lcd.backlight();

    title();

    lcd.clear();
    lcd.print("Initialize Wifi.");
    lcd.setCursor(0, 1);

    // Reset ESP8266
    SendCommand("AT+RST", "Ready");
    lcd.print("*");

    // Set ESP8266 as Access Point
    SendCommand("AT+CWMODE=2", "OK");
    lcd.print("*");

    // Configure Wi-Fi Access Point
    SendCommand(
        "AT+CWSAP=\"WATER_PH\",\"12345678\",5,3",
        "OK"
    );

    lcd.print("*");

    // Get IP address
    SendCommand("AT+CIFSR", "OK");
    lcd.print("*");

    // Enable multiple connections
    SendCommand("AT+CIPMUX=1", "OK");
    lcd.print("*");

    // Start server on port 80
    SendCommand("AT+CIPSERVER=1,80", "OK");
    lcd.print("*");

    delay(1000);

    lcd.clear();

    Serial.println("Sensor Initing...");
    lcd.print("Sensor Initing...");

    title();

    // Initialize DHT sensor
    dht.begin();
}


// ---------------------------------------------------------
// Main Loop
// ---------------------------------------------------------
void loop()
{
    // Read sensors every 1 second
    if (timeout + 1000 < millis())
    {
        // Read humidity and temperature
        h = dht.readHumidity();
        t = dht.readTemperature();

        if (isnan(h) || isnan(t))
        {
            Serial.println(
                F("Failed to read from DHT sensor!")
            );
        }
        else
        {
            Serial.print("T: ");
            Serial.println(t);

            Serial.print("H: ");
            Serial.println(h);
        }

        timeout = millis();

        // Read pH sensor
        read_ph();

        // Update LCD
        title();
    }


    // Check if ESP8266 is sending data
    if (mySerial.available())
    {
        char tbuf[50] = {0};
        char ch;

        int cnt = 100;
        int comma_cnt = 0;
        int k = 0;

        Serial.print(".");

        while (mySerial.available() > 0)
        {
            cnt = 100;

            while (mySerial.available() == 0 && cnt--)
            {
                delay(1);
            }

            ch = mySerial.read();

            if (ch == 0x00 || ch == 0x0A || ch == 0x0D)
            {
                break;
            }

            if (ch == '+' || tbuf[0] == '+')
            {
                if (ch == ',')
                {
                    comma_cnt++;
                }

                if (comma_cnt > 1)
                {
                    break;
                }

                tbuf[k++] = ch;
            }
        }


        if (strlen(tbuf) > 0)
        {
            // Clear remaining serial data
            while (mySerial.available() > 0)
            {
                mySerial.read();
            }

            Serial.print("[");
            Serial.print(tbuf);
            Serial.print("]");


            // Check for incoming IPD request
            if (strncmp(tbuf, "+IPD,", 5) == 0)
            {
                delay(1000);
                delay(100);

                // Get connection ID
                int connectionId = atoi(tbuf + 5);

                int indx = 0;


                // Create HTML webpage
                strcpy(
                    webpage + indx,
                    "<h1><center>WATER QUALITY TEST</center></h1><html>"
                );

                indx = strlen(webpage);

                strcpy(
                    webpage + indx,
                    "<meta http-equiv=\"refresh\" content=\"10\">"
                );

                indx = strlen(webpage);

                strcpy(
                    webpage + indx,
                    "<body style=\"background-color:"
                );

                indx = strlen(webpage);

                strcpy(
                    webpage + indx,
                    "green;\"><center>"
                );

                indx = strlen(webpage);

                strcpy(
                    webpage + indx,
                    "<br><h2>PH: "
                );

                indx = strlen(webpage);


                // Add pH value
                memset(tbuf, 0x00, 20);

                int pval = phval * 100;

                sprintf(
                    tbuf,
                    "%d.%02d",
                    pval / 100,
                    abs(pval % 100)
                );

                Serial.println(isnan(phval));
                Serial.println(phval);
                Serial.println(tbuf);

                strcpy(webpage + indx, tbuf);

                indx = strlen(webpage);


                // Add temperature
                strcpy(
                    webpage + indx,
                    "<br>Temp: "
                );

                indx = strlen(webpage);

                memset(tbuf, 0x00, 20);

                pval = t * 100;

                sprintf(
                    tbuf,
                    "%d.%02d C",
                    pval / 100,
                    abs(pval % 100)
                );

                Serial.println(isnan(t));
                Serial.println(t);
                Serial.println(tbuf);

                strcpy(webpage + indx, tbuf);

                indx = strlen(webpage);


                // Close HTML
                strcpy(
                    webpage + indx,
                    "</h2></body></html>"
                );

                indx = strlen(webpage);


                // Send webpage to ESP8266
                Serial.print("AT+CIPSEND=");
                Serial.print(connectionId);
                Serial.print(",");
                Serial.println(indx);

                mySerial.print("AT+CIPSEND=");
                mySerial.print(connectionId);
                mySerial.print(",");
                mySerial.println(indx);


                // Wait for '>' prompt
                int cnt = 1000;

                while (mySerial.read() != '>' && cnt--)
                {
                    delay(2);
                }

                delay(100);


                // Send webpage
                Serial.print(webpage);
                mySerial.print(webpage);


                // Wait for OK response
                cnt = 2000;

                while (mySerial.read() != 'K' && cnt--)
                {
                    delay(2);
                }


                // Close connection
                String closeCommand = "AT+CIPCLOSE=";

                closeCommand += connectionId;
                closeCommand += "\r\n";

                Serial.print("AT+CIPCLOSE=");
                Serial.println(connectionId);

                mySerial.print("AT+CIPCLOSE=");
                mySerial.println(connectionId);

                delay(1000);
            }
        }
    }
}


// ---------------------------------------------------------
// Read character from Serial
// ---------------------------------------------------------
char serial_rx(void)
{
    while (Serial.available() == 0);

    return Serial.read();
}


// ---------------------------------------------------------
// Read character from Serial 1
// ---------------------------------------------------------
char serial1_rx(void)
{
    while (Serial.available() == 0);

    return Serial.read();
}


// ---------------------------------------------------------
// Transmit character
// ---------------------------------------------------------
void serial_tx(char ch)
{
    Serial.print(ch);
}


// ---------------------------------------------------------
// Transmit string
// ---------------------------------------------------------
void serial_str(char *str)
{
    Serial.print(str);
}


// ---------------------------------------------------------
// Display string on LCD
// ---------------------------------------------------------
void lcd_str(char l, char p, char *str)
{
    if (l == 1)
    {
        lcd.setCursor(p - 1, 0);
    }
    else if (l == 2)
    {
        lcd.setCursor(p - 1, 1);
    }

    while (*str != '\0')
    {
        lcd.print(*str);
        str++;
    }
}


// ---------------------------------------------------------
// Send AT Command to ESP8266
// ---------------------------------------------------------
boolean SendCommand(String cmd, String ack)
{
    mySerial.println(cmd);

    // Send "AT+" command to module
    if (!echoFind(ack))
    {
        // Timed out waiting for acknowledgement
        return true;
    }
}


// ---------------------------------------------------------
// Wait for response from ESP8266
// ---------------------------------------------------------
boolean echoFind(String keyword)
{
    byte current_char = 0;
    byte keyword_length = keyword.length();

    long deadline = millis() + TIMEOUT;

    while (millis() < deadline)
    {
        if (mySerial.available())
        {
            char ch = mySerial.read();

            if (ch == keyword[current_char])
            {
                if (++current_char == keyword_length)
                {
                    return true;
                }
            }
        }
    }

    return false;
}
