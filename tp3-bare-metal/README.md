# TP3: Ejecución en Hardware Físico (Bare Metal)

**Objetivo:** trasladar el binario compilado a una computadora real (ej. Lenovo T450), comprender los requerimientos del firmware en hardware físico y sortear las restricciones de seguridad del Secure Boot.

## 3.1 Preparación del medio de arranque (USB)

Para que el firmware UEFI de una PC real reconozca un pendrive como un dispositivo de arranque válido sin un sistema operativo instalado, el almacenamiento debe cumplir estrictamente con los estándares de particionamiento y jerarquía del estándar UEFI:

1. **Identificación y Formateo:** Se verifica la partición del dispositivo con `lsblk` (ej. `/dev/sdb1`) y se formatea en **FAT32**, el único sistema de archivos que la especificación UEFI reconoce nativamente para leer archivos de arranque:
```bash
sudo mkfs.vfat -F 32 /dev/sdb1

```


2. **Montaje y Estructura de Directorios:** Se monta la partición en un directorio de trabajo local (ej. `/mnt/efi_usb`) y se crea la jerarquía estandarizada que el firmware busca automáticamente (`EFI/BOOT`):
```bash
sudo mkdir -p /mnt/efi_usb
sudo mount /dev/sdb1 /mnt/efi_usb
sudo mkdir -p /mnt/efi_usb/EFI/BOOT

```


3. **Instalación de la UEFI Shell:** Se descarga el binario oficial de la UEFI Shell de TianoCore (`Shell.efi`) y se renombra obligatoriamente como `BOOTX64.EFI`. Este es el nombre por defecto que busca el firmware en arquitecturas x86_64 cuando no hay entradas NVRAM configuradas:
```bash
sudo wget https://github.com/tianocore/edk2/raw/UDK2018/ShellBinPkg/UefiShell/X64/Shell.efi -O /mnt/efi_usb/EFI/BOOT/BOOTX64.EFI

```


4. **Copia de la Aplicación:** Se transfiere el binario compilado (`aplicacion.efi`) a la raíz del pendrive, se sincronizan los buffers de escritura y se desmonta de forma segura:
```bash
sudo cp aplicacion.efi /mnt/efi_usb/
sudo sync
sudo umount /mnt/efi_usb

```



## 3.2 Configuración del BIOS/UEFI (Lenovo T450)

El hardware comercial moderno incluye medidas de seguridad estrictas que impiden la ejecución arbitraria de código no verificado. Para probar nuestra aplicación en la Lenovo T450, se ajustan los siguientes parámetros en el BIOS:

1. Se enciende el equipo presionando **F1** para acceder a la utilidad de configuración del BIOS.
2. Se navega hasta la sección **Security -> Secure Boot** y se establece en **Disabled**.
3. Se verifica en la sección **Startup** que el modo de arranque esté configurado estrictamente en **UEFI Only** (desactivando CSM / Legacy Support).
4. Se guardan los cambios presionando **F10** y se reinicia el equipo.

### ❓ Pregunta 6

> ¿Por qué es obligatorio desactivar el *Secure Boot* para ejecutar nuestra aplicación en hardware físico (*bare metal*)?

**Por qué:** El **Secure Boot** implementa una cadena de confianza basada en criptografía de clave pública. El firmware de la motherboard valida que el gestor de arranque (la UEFI Shell descargada) y cualquier imagen posterior posean una **firma digital válida** autorizada por certificados comerciales (habitualmente de Microsoft).

**Por qué importa:** Dado que tanto nuestra UEFI Shell comunitaria como el binario `aplicacion.efi` que desarrollamos son compilados localmente y **no están firmados digitalmente** por ninguna autoridad reconocida por el fabricante, el firmware detectará una violación de integridad y bloqueará la ejecución de inmediato. Al desactivar el Secure Boot, permitimos que la UEFI ejecute código arbitrario sin validación de certificados, habilitando las pruebas de desarrollo en *bare metal*.

## 3.3 Ejecución en Bare Metal

1. Con el pendrive conectado, se enciende la laptop y se presiona inmediatamente **F12** para abrir el menú de selección de booteo (*Boot Menu*), eligiendo la unidad USB.
2. Una vez cargada la consola de la UEFI Shell, el entorno asigna una letra de unidad lógica a los dispositivos detectados (generalmente `FS0:`). Se procede a cambiar de directorio y ejecutar el binario:
```plaintext
Shell> FS0:
FS0:\> ls
FS0:\> aplicacion.efi

```


3. **Comportamiento:** La pantalla se limpia, se imprimen los mensajes nativos a través de `ConOut->OutputString`, se ejecuta la pausa de 5 segundos mediante los servicios de tiempo del firmware (`BootServices->Stall`), y el control retorna de forma limpia a la Shell al completarse la ejecución (`EFI_SUCCESS`), operando de manera autónoma directamente sobre el silicio de la Lenovo T450 sin ningún sistema operativo intermediario.
