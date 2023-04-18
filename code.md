// Inclusion de librerias
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/adc.h"
#include "lcd_i2c.h"
#include <math.h>

// GPIO del ADC
#define ANALOG_GPIO 26
// Canal del ADC
#define ANALOG_CH   0

// Tiempo de refresco para el siete segmentos
#define SLEEP_MS 10
// Tiempo entre conversiones
#define ADC_DELAY_MS  50

#define NUM_SAMPLES 10

int num_conversiones = 0; //numero de conversiones 
float temp_sum = 0.0; //suma de temperatura
float temp_avg = 0.0; //promedio de temperatura
// Variable para almacenar el resultado del ADC
uint16_t adc_value = 0;
// Variable para guardar el valor de temperatura
float temperatura = 0.0;
// Constante de temperatura para el termistor
const uint16_t beta = 3950;

/*
 * @brief Callback para la interrupcion de timer
 * @param t: puntero a repeating_timer
 */
bool muestreo_periodico(struct repeating_timer *t) {
   // Lectura analogica (variable adc_value)
    adc_value = adc_read();
    // Calcular valor de temperatura (variable temperatura)
    temperatura = 1 / (log(1 / (4065. / adc_value - 1)) / beta + 1.0 / 298.15) - 273.15;
    // Sumar la temperatura a la suma acumulada
    temp_sum += temperatura;
    // Incrementar el contador de conversiones
    num_conversiones++;
    // Si se han realizado NUM_SAMPLES conversiones
    if (num_conversiones == NUM_SAMPLES) {
        // Calcular el promedio de las temperaturas
        temp_avg = temp_sum / NUM_SAMPLES;
        // Resetear el contador y la suma acumulada
        num_conversiones = 0;
        temp_sum = 0.0;

        printf("Temperatura promedio: %.2f\n", temp_avg); //muestra el promedio de temperatura 
}
}
/*
 * @brief Display temperature value in 7 segment display
 * @param temperature: value to display
 */
void display_temp(float temperatura) {
  // Variable para armar el string
  char str[16];
  // Armo string con temperatura
  sprintf(str, "Temp=%.2f C", temperatura);
  // Limpio display
  lcd_clear();
  // Muestro
  lcd_string(str);
}

/*
 * @brief Programa principal
 */
int main() {
  // Inicializacion de UART
  stdio_init_all();
  // Creo un repeating timer
  struct repeating_timer timer;
  // Creo un callback para la interrupcion del timer
  add_repeating_timer_ms(ADC_DELAY_MS, muestreo_periodico, NULL, &timer);
  // Inicializo ADC
  adc_init();
  // Inicializo GPIO26 como entrada analogica
  adc_gpio_init(ANALOG_GPIO);
  // Selecciono canal analogico
  adc_select_input(ANALOG_CH);
  // Configuro el I2C0 a 100 KHz de clock
  i2c_init(i2c0, 100 * 1000);
  // Elijo GPIO4 como linea de SDA
  gpio_set_function(4, GPIO_FUNC_I2C);
  // Elijo GPIO5 como linea de SCL
  gpio_set_function(5, GPIO_FUNC_I2C);
  // Activo pull-up en ambos GPIO, son debiles por lo que
  // es recomendable usar pull-ups externas
  gpio_pull_up(4);
  gpio_pull_up(5);
  // Inicializo display
  lcd_init();

  // Bucle infinito
  while (true) {
    // Muestro temperatura
    display_temp(temperatura);
    // Espero
    sleep_ms(SLEEP_MS);
  }
}
