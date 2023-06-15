# Notas sobre LoRa

## Parâmetros

* Pinos de operação: M0 (GPIO22), M1 (GPIO27) na nomenclatura BCM

Se o hat estiver ligado ao RPi, os pinos devem ser deixados a flutuar e devem ser definidos programaticamente
Modos de operação ([ref](https://www.waveshare.com/wiki/SX1262_868M_LoRa_HAT#Features))

* [M0=LO, M1=LO] : transmission mode 
* [M0=LO, M1=HI] : configuration mode
* [M0=HI, M1=LO] : WOR mode
* [M0=HI, M1=HI] : deep sleep mode (standby)

### Transmission mode
Dados são enviados imediatamente. Quando em idle pode receber



## Registros

Configuração é efetuada em 12 registros. A função de cada registro depende da configuração em geral. 
NOTA: Alguns registros são usados para diferentes parâmetros simultaneamente

### Parâmetros de configuração

* address     : 2 bytes 
* net_id      : 1 byte
* freq        : split into freq_start + freq_offset
  * start-freq = 850
  * offset_freq = freq - start_freq
* air_speed   : 1 byte (from a dictionary)
* buffer_size : 1 byte (from a dict)
* power       : 1 byte (from dict)
* baud_rate   : 1 byte (from dict)
* rssi        : 1 byte
  * 0x80 : print rssi
  * 0x00 : disable print
* crypt  : 2 bytes 

### Dicionários 

#### air speed

```
    lora_air_speed_dic = {
        1200:0x01,
        2400:0x02,
        4800:0x03,
        9600:0x04,
        19200:0x05,
        38400:0x06,
        62500:0x07
    }
```

#### power (in dBm)

```
    lora_power_dic = {
        22:0x00,
        17:0x01,
        13:0x02,
        10:0x03
    }
```

#### buffer size (in bytes)

```
    lora_buffer_size_dic = {
        240:0x00,
        128:0x40,
        64:0x80,
        32:0xC0
    }
```

#### Baud rate

```
    SX126X_UART_BAUDRATE_1200 = 0x00
    SX126X_UART_BAUDRATE_2400 = 0x20
    SX126X_UART_BAUDRATE_4800 = 0x40
    SX126X_UART_BAUDRATE_9600 = 0x60
    SX126X_UART_BAUDRATE_19200 = 0x80
    SX126X_UART_BAUDRATE_38400 = 0xA0
    SX126X_UART_BAUDRATE_57600 = 0xC0
    SX126X_UART_BAUDRATE_115200 = 0xE0
```

### Definição dos registros

* 0 : header (0xC0 escreve configuração permanentemente; 0xC2 configuração temporária)
* 1 : 0x0 (padding)
* 2 : 0x09 (__???__)
* 3 : high address : ((addr >> 8) & 0xFF)   **in relay mode this parameter does not matter (set to 0x01)**
* 4 : low address : (addr & 0xFF)           **in relay mode this parameter does not matter (set to 0x02)**
* 5 : net id : only relevant in LoRaWAN     **in relay mode this parameter does not matter (set to 0x03)**
* 6 : baud_rate + air_speed
* 7 : buffer_size + power + 0x20
  * add 0x20 enables reading rssi. don't add to get nothing
* 8 : offset_freq (the start_freq is chip dependent)
* 9 : 0x43 + rssi
  * If rssi is 0x80, an additional byte with the rssi value is read out at the end of the receiving message
  * If in relay mode, replace 0x43 by 0x03
* 10: crypt_high : (crypt >> 8) & 0xFF
* 11: crypt_low  : (crypt & 0xFF)


### Configuração

1. Enviar os 12 bytes numa única mensagem
  1. Se os registros foram escritos corretamente, resposta com 0xC1 no primeiro byte
2. Baixar M0 e M1 para estado LOW (0). --> chip em transmission mode

Para ler a configuração em vigor, enviar a seguinte palavra

```
[0xC1,0x00,0x09]
```

A resposta conterá 12 bytes, onde os 3 primeiros são os mesmos que acima.

__NOTA:__ Não esquecer de baixar o pino M1 para LOW após a query

## Transmissão

Os pinos M0 e M1 devem ambos estar em modo LOW

Os dados devem ser escritos na porta serial no formato "node address,freq, payload". por exemplo, "100,868,Test message"

## Recepção

Os pinos M0 e M1 devem ambos estar em modo LOW

1. Primeiro testar se há dados no buffer do canal. 
2. Ler enquanto houver dados

Formato dos dados lidos no buffer
buffer[0] << 8 + buffer[1] --> origin node address
buffer[2] + start_freq --> channel frequency
buffer[3:] --> message

## Let ruido do canal

Enviar commando `[0xC0,0xC1,0xC2,0xC3,0x00,0x02]`

Resposta terá 4 bytes na forma `[0xC1,0x00,0x02,val]`
O valor de rssi = -(256-val) 
