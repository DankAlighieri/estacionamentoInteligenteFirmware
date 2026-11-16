# Firmware do Estacionamento Inteligente (protótipo)

Documentação do firmware em [`main/main.c`](../main/main.c), escrito com a API do ESP-IDF (v6.1, alvo ESP32).

## 1. O que o firmware faz

Monitora **2 vagas** de estacionamento com **um sensor magnético** e indica o estado de cada vaga com LEDs:

| Estado da vaga | LED verde | LED vermelho |
|----------------|-----------|--------------|
| Livre          | aceso     | apagado      |
| Ocupada        | apagado   | aceso        |

O estado também é impresso no monitor serial a cada 500 ms.

## 2. Hardware

Esquemático: [`assets/esquematico_esp32_gy82_vagas.svg`](../assets/esquematico_esp32_gy82_vagas.svg)

| Função            | Pino ESP32 | Observação                          |
|-------------------|------------|-------------------------------------|
| I2C SDA (GY-82)   | GPIO21     | Pull-up interno habilitado          |
| I2C SCL (GY-82)   | GPIO22     | Pull-up interno habilitado          |
| Vaga 1 – verde    | GPIO13     | Resistor de 220 Ω em série          |
| Vaga 1 – vermelho | GPIO14     | Resistor de 220 Ω em série          |
| Vaga 2 – verde    | GPIO25     | Resistor de 220 Ω em série          |
| Vaga 2 – vermelho | GPIO26     | Resistor de 220 Ω em série          |

O GY-82 usa o chip **HMC5883L** (magnetômetro de 3 eixos), endereço I2C `0x1E`, alimentado em 3V3.

## 3. Como a detecção funciona

Um carro é uma grande massa de metal e **distorce o campo magnético** ao redor dele. O sensor mede esse campo; se ele mudar o suficiente em relação ao normal, consideramos que há um carro.

1. **Referência:** logo após ligar, o firmware lê o campo uma vez e guarda como "vaga livre".
2. **Comparação:** a cada 500 ms lê de novo e calcula a diferença para a referência.
3. **Decisão:** se a diferença passar de `LIMITE` (0,08 Gauss), a vaga é considerada ocupada.

## 4. Estrutura do código

`main.c` tem 3 funções auxiliares e o `app_main`:

| Função | O que faz |
|--------|-----------|
| `sensor_escreve(reg, valor)` | Escreve um byte em um registrador do HMC5883L via I2C. |
| `sensor_le(&x, &y)` | Lê 6 bytes a partir do registrador `0x03` e converte X e Y para Gauss (divide por 1090, o ganho configurado). |
| `mostra_vaga(verde, vermelho, ocupada)` | Acende o LED certo conforme o estado da vaga. |
| `app_main()` | Configura GPIOs, I2C e sensor; tira a referência; entra no laço principal. |

Fluxo do `app_main`:

```
configura LEDs (saída)
cria barramento I2C  ->  adiciona o sensor  ->  configura o sensor
lê referência (vagas livres)
loop a cada 500 ms:
    lê X e Y
    ocupada1 = |X - ref_X| > LIMITE
    ocupada2 = |Y - ref_Y| > LIMITE
    atualiza LEDs e imprime no log
```

### Funções do ESP-IDF utilizadas

| API | Uso |
|-----|-----|
| `gpio_reset_pin`, `gpio_set_direction`, `gpio_set_level` | Controle dos LEDs |
| `i2c_new_master_bus` | Cria o barramento I2C |
| `i2c_master_bus_add_device` | Registra o sensor no barramento |
| `i2c_master_transmit` | Escreve nos registradores |
| `i2c_master_transmit_receive` | Lê os dados (envia o endereço do registrador e recebe 6 bytes) |
| `ESP_ERROR_CHECK` | Aborta com mensagem se alguma chamada falhar |
| `ESP_LOGI` | Log no monitor serial |
| `vTaskDelay` | Espera entre leituras |

### Registradores do HMC5883L usados

| Registrador | Valor | Significado |
|-------------|-------|-------------|
| `0x00` (Config A) | `0x70` | Média de 8 amostras, 15 Hz |
| `0x01` (Config B) | `0x20` | Ganho ±1,3 Ga (1090 LSB/Ga) |
| `0x02` (Modo)     | `0x00` | Medição contínua |
| `0x03`–`0x08`     | —      | Dados: X(H,L), Z(H,L), Y(H,L) |

Atenção: a ordem dos dados no sensor é **X, Z, Y** (não X, Y, Z).

## 5. Como compilar e gravar

No terminal do ESP-IDF, na raiz do projeto:

```bash
idf.py build
idf.py -p COMx flash monitor
```

(troque `COMx` pela porta serial da placa). Para sair do monitor: `Ctrl+]`.

## 6. Ajustes e testes

- **Ligue com as vagas livres.** A referência é tirada no boot; se houver carro nesse momento, o resultado fica invertido.
- **`LIMITE`** (`main.c`): se as vagas acendem vermelho sem carro, aumente; se não detectam o carro, diminua. Use os valores do log como guia.
- Para trocar o eixo de uma vaga, altere qual variável (`x` ou `y`) é comparada em `app_main`.

## 7. Limitações conhecidas (propositais, por ser protótipo)

- Sem filtro/debounce: leituras ruidosas podem fazer o LED piscar.
- Sem histerese: perto do limite, o estado pode oscilar.
- Sem tratamento de erro além de abortar (`ESP_ERROR_CHECK`): se o sensor não responder, o ESP32 reinicia.
- A referência não é salva; é recalculada a cada boot.
- Sem comunicação (Wi-Fi/MQTT): o estado só aparece nos LEDs e no log.

## 8. Próximos passos sugeridos

1. Média de várias leituras / debounce para estabilizar a detecção.
2. Histerese (limites diferentes para ocupar e liberar).
3. Calibração salva na NVS.
4. Publicar o estado das vagas via MQTT.
