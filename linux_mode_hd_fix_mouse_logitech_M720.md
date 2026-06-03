# Linux Mouse HID Fix (Logitech M720)

Este repositório documenta a solução para o problema de pareamento Bluetooth em que o dispositivo é reconhecido pelo sistema, mas não envia eventos de movimento (cursor travado) devido a uma classificação incorreta de `Device Type` no `udev`.

## O problema

Em distribuições Linux (especialmente Debian Testing/Trixie e Ubuntu recentes), alguns periféricos Logitech são identificados incorretamente pelo kernel como um teclado genérico, mesmo sendo mouses. Como o `libinput` não recebe a tag de pointer, o sistema ignora o fluxo de dados de coordenadas do dispositivo.

## Diagnóstico

Para confirmar se você tem o mesmo problema, verifique se o dispositivo Bluetooth HID é listado como teclado no `dmesg`:

```bash
dmesg | grep -i "Bluetooth HID"
```

Saída esperada quando há erro:

```text
hid-generic ...: BLUETOOTH HID v0.13 Keyboard [M720 Triathlon]
```

## Solução: regra `udev`

A correção consiste em criar uma regra personalizada no `udev` que força o sistema a reconhecer o dispositivo como um mouse (`ID_INPUT_MOUSE=1`).

### Passo 1 — Identificar o dispositivo

Use `lsusb` ou verifique o log do kernel para encontrar o Vendor ID e Product ID do dispositivo. O exemplo abaixo usa `046d:b015`.

### Passo 2 — Criar a regra

Crie o arquivo de regra:

```bash
sudo nano /etc/udev/rules.d/99-logitech-m720.rules
```

Adicione a configuração abaixo:

```udev
# Força a classificação como mouse para o Logitech M720
ACTION=="add|change", KERNEL=="event*", ATTRS{idVendor}=="046d", ATTRS{idProduct}=="b015", ENV{ID_INPUT_MOUSE}="1", ENV{ID_INPUT}="1"
```

### Passo 3 — Aplicar as mudanças

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

## Como o `udev` funciona

`udev` é o gerenciador de dispositivos do Linux que roda em userspace. Ele intercepta eventos do kernel quando um novo hardware é conectado.

- Regras: arquivos em `/etc/udev/rules.d/` processam atributos do dispositivo, como `idVendor`, `idProduct` e `SUBSYSTEM`.
- Ação: quando um dispositivo corresponde aos critérios, o `udev` pode renomear interfaces, definir permissões ou injetar variáveis de ambiente (`ENV{...}`) usadas pelo sistema para categorizar o hardware.
- Prioridade: o prefixo `99-` garante que a regra seja processada por último, sobrescrevendo configurações padrão que possam causar conflito.
