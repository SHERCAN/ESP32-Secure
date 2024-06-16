# Proceso de Arranque Seguro y Encriptación de Flash en ESP32
### Introducción
El proceso de arranque seguro y encriptación de flash en ESP32 es crucial para garantizar la seguridad de los dispositivos IoT. Esta guía proporciona instrucciones detalladas sobre cómo implementar estos procesos de manera efectiva.

<img src="https://cdn-icons-png.flaticon.com/512/6360/6360061.png" alt="Descripción de la imagen" style="width:50px;"> 

# Proceso Irreversible: Tómese con Precaución

## 1. Función para Arranque Seguro (Secure Boot)
### 1.1 Instalación de Dependencias
- Instale OpenSSL según su sistema operativo.
  - Ejemplo para Windows: `choco install openssl`
- Instale la herramienta esptool en Python.
  - Ejecute el comando: `pip install esptool`
  
### 1.2 Generación de Llave RSA-3072 
- Genere una llave RSA-3072 con OpenSSL.
  - Comando: `openssl genrsa -out rsa_key.pem 3072`
- O bien, genere una llave RSA-3072 con la herramienta espsecure, que la generará en formato .bin.
  - Comando: `espsecure generate_signing_key --version 2 --scheme rsa3072 secure_v2.pem`
- Verifique la llave generada.
  - Comando: `openssl rsa -in rsa_key.pem -check`
- Verifique la llave generada en formato de texto.
  - Comando: `openssl rsa -in rsa_key.pem -text -noout`
  
### 1.3 Firmado de Binarios para Cargar en la ESP 
- Firme los binarios de su aplicación, firmware, particiones, bootloader, etc., utilizando la llave generada, ya sea en formato .bin o .pem.
  - Comando: `espsecure sign_data firmware.bin --version 2 --keyfile secure_v2.bin --output firmware_signed.bin`
  
### 1.4 Creación de la Llave Pública y Flasheo 
- Cree la llave pública para quemar en la ESP32.
  - Comando: `espsecure extract_public_key -k secure_v2.bin --version 2 public_key_efuse.bin`
- Extraer la llave de encriptación en hexadecimal para poder quemarla en los EFUSES
  - `espsecure digest_rsa_public_key -k public_key_efuse.bin -o hex_public_key.bin`
- Flashee el bootloader en la ESP32. (Revise la partición de la ESP; si es necesario, cambie la dirección de escritura).
  - Comando: `espefuse --chip esp32 --port COM3 burn_key secure_boot hex_public_key.bin`
  
### 1.5 Flasheo de los Binarios Firmados
- Utilice este código para flashear todos los binarios.
  - Comando: `esptool -c esp32 -p COM3 write_flash 0x1000 bootloader.bin 0x8000 partitions.bin 0x11000 firmware.bin`
### 1.6 Verificación de Efuses 
- Verifique los efuses utilizando la herramienta espefuse. Vea [Utilidades](#utilidades).
  - Comando: `espefuse -p COM3 summary`

## 2. Función para Encriptar el Flash 
### 2.1 Generación de la Clave de Encriptación del Flash
- Genere la clave de encriptación del flash mediante la herramienta espsecure.
  - Comando: `espsecure generate_flash_encryption_key flash_enc_key.bin`

### 2.2 Adición de la Clave Binaria a los Efuses 
- Agregue la llave de encriptación a los efuses de **BLOCK1**. Vea [Utilidades](#utilidades).
  - Comando: `espefuse -p COM3 burn_key  flash_encryption flash_enc_key.bin`
  - Tambien puede utilizar este: `espefuse burn_key BLOCK1 flash_encryption_key.bin`
- Active el mecanismo de cifrado flash.
  - Comando: `espefuse -p COM3 burn_efuse FLASH_CRYPT_CNT`
- Configure el cifrado para utilizar la clave de encripción del flash.
  - Comando: `espefuse -p COM3 burn_efuse FLASH_CRYPT_CONFIG 0x0F`

### 2.3 Encriptación de Partición, Firmware y Bootloader
- ### Herramienta: espsecure 
  - Ejemplo de comando partición: `espsecure encrypt_flash_data -k flash_enc_key.bin -a 0x8000 -o partitions_signed_flash.bin partitions.bin`

  - Ejemplo de comando firmware: `espsecure encrypt_flash_data -k flash_enc_key.bin -a 0x11000 -o firmware_signed_flash.bin firmware.bin`

  - Ejemplo de Comando Bootloader: `espsecure encrypt_flash_data -k flash_enc_key.bin -a 0x1000 -o bootloader_signed_flash.bin bootloader.bin`

### 2.4 Flasheo de los Binarios en la ESP32
- ### Herramienta: esptool
  - Comando: `esptool -c esp32 -p COM3 write_flash 0x8000 partitions_signed_flash.bin 0x11000 firmware_signed_flash.bin 0x1000 bootloader_signed_flash.bin`

### 3. Verificación de Efuses 
- Verifique los efuses utilizando la herramienta espefuse.
  - Comando: `espefuse -p COM3 summary`

## Utilidades 
### Verificación de Efuses
- Utilice la herramienta espefuse para verificar los efuses.
  - Comando: `espefuse -p COM3 summary`
- El ítem **a** indica que el flash no está encriptado, el ítem **b** indica que no hay código encriptado en el flash, y el ítem **c** indica que no hay encriptación para el arranque seguro.
  - Los datos que deben aparecer son los siguientes:
  a. `FLASH_CRYPT_CNT = 0`, `FLASH_CRYPT_CONFIG = 0`
  b. `BLOCK1 (BLOCK1): Flash encryption key = 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 R/W`
  c. `BLOCK2 (BLOCK2): Secure boot key = 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 R/W`


### Borrado de Toda la Memoria Flash
- Comando: `esptool.py --port com3 erase_flash`

### Escritura del Bootloader
- Comando: `esptool.py --port /dev/ttyUSB0 --baud 921600 write_flash 0x1000 bootloader.bin`

### Escritura del Esquema de Particiones
- Comando: `esptool.py --port /dev/ttyUSB0 --baud 921600 write_flash 0x8000 partitions.bin`

### Escritura del Firmware
- Comando: `esptool.py --port /dev/ttyUSB0 --baud 921600 write_flash 0x10000 firmware.bin`

### Comprobación del Cifrado 
- Puede utilizar la librería "esp_flash_encrypt.h" para tomar decisiones lógicas adicionales.

# Bibliografía
- [Foro Pycom: Secure Boot and Encryption](https://forum.pycom.io/topic/4949/secureboot-and-encryption/3)
- [Motius: How to Build a Secure IoT Prototype with Arduino and ESP32](https://www.motius.com/post/how-to-build-a-secure-iot-prototype-with-arduino-and-esp32)
- [Medium: ESP32 Firmware Encryption](https://medium.com/@kattelroshan1/esp32-firmware-encryption-a53eb1c9bf9c)
