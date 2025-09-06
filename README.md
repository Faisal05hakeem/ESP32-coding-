# ESP32-coding-
ESP32 coding for the Obstacle detection 
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/ledc.h"
#include "esp_timer.h"

#define TRIG_PIN GPIO_NUM_5
#define ECHO_PIN GPIO_NUM_18
#define VIBRATION_PIN GPIO_NUM_19
#define BUZZER_PIN GPIO_NUM_21

#define THRESHOLD_DISTANCE_CM 20

// Function to measure distance using ultrasonic sensor
float measure_distance_cm() {
    // Send 10us pulse to TRIG
    gpio_set_level(TRIG_PIN, 0);
    ets_delay_us(2);
    gpio_set_level(TRIG_PIN, 1);
    ets_delay_us(10);
    gpio_set_level(TRIG_PIN, 0);

    // Wait for ECHO pin to go high
    int64_t start_time = esp_timer_get_time();
    while (gpio_get_level(ECHO_PIN) == 0) {
        if ((esp_timer_get_time() - start_time) > 1000000) { // 1 second timeout
            return -1; // timeout error
        }
    }
    int64_t echo_start = esp_timer_get_time();

    // Wait for ECHO pin to go low
    while (gpio_get_level(ECHO_PIN) == 1) {
        if ((esp_timer_get_time() - echo_start) > 30000) { // 30 ms max echo time (~5 meters)
            return -1; // timeout error
        }
    }
    int64_t echo_end = esp_timer_get_time();

    // Calculate pulse duration in microseconds
    int64_t pulse_duration = echo_end - echo_start;

    // Calculate distance in cm
    // Speed of sound = 34300 cm/s
    // Distance = (pulse_duration in us) * (speed of sound cm/us) / 2
    float distance_cm = (pulse_duration * 0.0343) / 2.0;

    return distance_cm;
}

void app_main(void) {
    // Configure pins
    gpio_config_t io_conf = {
        .intr_type = GPIO_INTR_DISABLE,
        .mode = GPIO_MODE_OUTPUT,
        .pin_bit_mask = (1ULL << TRIG_PIN) | (1ULL << VIBRATION_PIN) | (1ULL << BUZZER_PIN),
        .pull_down_en = 0,
        .pull_up_en = 0
    };
    gpio_config(&io_conf);

    gpio_config_t echo_conf = {
        .intr_type = GPIO_INTR_DISABLE,
        .mode = GPIO_MODE_INPUT,
        .pin_bit_mask = (1ULL << ECHO_PIN),
        .pull_down_en = 0,
        .pull_up_en = 0
    };
    gpio_config(&echo_conf);

    while (1) {
        float distance = measure_distance_cm();
        if (distance > 0 && distance <= THRESHOLD_DISTANCE_CM) {
            // Object is near: activate vibration and buzzer
            gpio_set_level(VIBRATION_PIN, 1);
            gpio_set_level(BUZZER_PIN, 1);
        } else {
            // Object is far or error: deactivate vibration and buzzer
            gpio_set_level(VIBRATION_PIN, 0);
            gpio_set_level(BUZZER_PIN, 0);
        }
        vTaskDelay(pdMS_TO_TICKS(200)); // Delay 200 ms before next measurement
    }
}
