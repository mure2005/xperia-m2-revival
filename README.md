# 📱 Cómo revivir un Xperia M2 con hardware moderno

Guía para usar un móvil antiguo (Xperia M2 con Android 5.1.1 o similar) como **pantalla y mando** de una máquina virtual Android moderna. El móvil viejo no hace el trabajo pesado: solo muestra y controla lo que ocurre en otro dispositivo más potente.

Así puedes usar apps actuales **sin depender de desbloquear el bootloader** (`bootloader unlock allowed: no`).

> 💡 **La idea en una frase:** el móvil nuevo (o tu PC) hace el trabajo duro y el viejo hace de monitor. Es como darle un escritorio nuevo a un abuelito: sigue siendo el mismo, pero ahora llega a sitios.

---

## 📑 Índice

1. [Qué necesitas](#-qué-necesitas)
2. [Método A: usando un móvil actual](#-método-a-usando-un-móvil-actual)
3. [Método B: usando un PC con Linux + QEMU](#-método-b-usando-un-pc-con-linux--qemu)
4. [Cómo crear el puente `br0`](#-cómo-crear-el-puente-br0)
5. [Notas y problemas conocidos](#-notas-y-problemas-conocidos)

---

## 🧰 Qué necesitas

| Elemento | Método A (móvil) | Método B (PC) |
|---|---|---|
| Móvil antiguo (Android 5.1.1 recomendado, ej. Xperia M2) | ✅ | ✅ |
| Móvil actual con app de máquinas virtuales | ✅ | ❌ |
| PC con Linux | ❌ | ✅ |
| QEMU | ❌ | ✅ |
| Puente de red `br0` | ❌ | ✅ |
| [EasyControl (fork)](https://github.com/daitj/Easycontrol) en el móvil antiguo | ✅ | ✅ |
| Tailscale (opcional, para acceso desde fuera de casa) | opcional | opcional |

---

## 📲 Método A: usando un móvil actual

### Paso 1: en tu móvil actual

1. **Descarga una app de máquinas virtuales Android.** Opciones:
   - **Virtual Master** (recomendada)
   - **VMOS**
   - **VPhoneGaGa**
2. **Descarga la máquina virtual** con una de estas versiones de Android: **7, 9, 11 o 12L**.
3. Al configurarla, elige la resolución **540x960**.
4. Entra en los **ajustes de la VM** y busca una opción de **ADB en red**.
   - Si la ves, **actívala**: te dará una IP para conectarte, por ejemplo `0.0.0.0:10000`.
   - Apunta esa IP, la necesitarás en el paso siguiente.

### Paso 2: en el móvil antiguo

5. Descarga e instala **EasyControl (fork)**: <https://github.com/daitj/Easycontrol>
6. Abre EasyControl y pulsa el **`+`** para crear una conexión.
7. En el campo **IP**, escribe la IP de tu móvil actual. Funciona con:
   - Wi-Fi
   - Datos móviles
   - Tailscale
8. Al conectarte, **acepta la conexión desde la máquina virtual** (te aparecerá un aviso de depuración ADB).
   - Si falla varias veces, **no pasa nada: sigue intentándolo**. A veces necesita un par de intentos para ponerse de acuerdo. 😅

### ✅ Resultado

Ya puedes usar apps actuales desde el móvil antiguo.

> ⚠️ **Character.ai:** la app **no funcionará** en este entorno. Usa la versión web desde el navegador: <https://character.ai/>
> Navegadores recomendados: **Chrome** o **[Cromite](https://github.com/uazo/cromite)**.

---

## 🖥️ Método B: usando un PC con Linux + QEMU

### Paso 1: instalar QEMU

```bash
sudo apt install qemu-system
```

### Paso 2: preparar y lanzar la VM

1. Asegúrate de tener creado el puente de red **`br0`** (mira la [sección siguiente](#-cómo-crear-el-puente-br0)). Sin él, el script **no te dejará continuar**.
2. Ejecuta el script de QEMU (el que se comparte aparte).

### Paso 3: conectar el PC con la VM por ADB

3. En tu PC, conecta ADB a la VM:

   ```bash
   adb connect IP_DE_LA_VM:5555
   ```

4. Ejecuta este comando para redirigir el puerto:

   ```bash
   socat TCP-LISTEN:5556,reuseaddr,fork TCP:IP_DE_LA_VM:5555
   ```

   > Sustituye `IP_DE_LA_VM` por la IP real de tu máquina virtual.

### Paso 4: conectar desde el móvil antiguo

5. En el Xperia M2 (o cualquier móvil antiguo):
   - Si la VM es **Android x86** en QEMU → usa **EasyControl (fork)**.
   - Si la VM es **Windows** → usa **Microsoft Remote Desktop**.
6. ✅ **Listo:** ya puedes manejar tu VM de QEMU desde el móvil.

### Paso 5 (opcional): acceso desde fuera de casa

7. Instala **Tailscale** en los dispositivos para conectarte desde fuera de casa.
   Ojo: **irá más lento** que en tu red local, pero funciona.

---

## 🌉 Cómo crear el puente `br0`

Un **puente de red** (bridge) hace que la VM aparezca en tu red como si fuera un dispositivo más. Ejecuta los comandos **uno por uno**.

> ⚠️ Cambia `eth0` por el nombre real de tu interfaz de red. Puedes verlo con:
> ```bash
> ip a
> ```
> (suele llamarse `eth0`, `enp3s0` o similar).

### Opción 1: con `ip` (recomendada)

```bash
# 1. Crear la interfaz de puente virtual (br0)
sudo ip link add name br0 type bridge

# 2. Levantar la interfaz br0
sudo ip link set dev br0 up

# 3. Añadir tu interfaz de red física principal al puente
sudo ip link set dev eth0 master br0

# 4. Crear el puerto TAP (tap0) para QEMU y asociarlo al puente
sudo ip tuntap add dev tap0 mode tap user $(whoami)
sudo ip link set dev tap0 master br0
sudo ip link set dev tap0 up
```

Por último, **reinicia**.

### Opción 2: con `brctl` (si la anterior no te funciona)

Primero instala `bridge-utils`:

```bash
sudo apt install bridge-utils
```

Después:

```bash
# 1. Crear el puente y añadir la interfaz física (eth0)
sudo brctl addbr br0
sudo brctl addif br0 eth0

# 2. Levantar las interfaces
sudo ip link set dev eth0 up
sudo ip link set dev br0 up

# 3. Crear tap0 y asociarlo al bridge
sudo ip tuntap add dev tap0 mode tap user $(whoami)
sudo brctl addif br0 tap0
sudo ip link set dev tap0 up
```

Por último, **reinicia**.

---

## 📝 Notas y problemas conocidos

- **Conexión que falla:** en EasyControl, reintenta varias veces y acepta el aviso de ADB en la VM.
- **Puente solo con cable:** el método con `br0` funciona con interfaces **Ethernet**. Con Wi-Fi normalmente no se puede hacer bridge de esta forma.
- **Conexión del PC al añadir `eth0` al puente:** al meter `eth0` en `br0`, la IP del PC suele pasar a gestionarse desde `br0`. Si te quedas sin red, pide la IP por DHCP en `br0`:
  ```bash
  sudo dhclient br0
  ```
- **Los comandos `ip`/`brctl` no son permanentes:** si reinicias, el puente puede desaparecer. Si te pasa, vuelve a ejecutar los comandos o conviértelos en un script de arranque.
- **Remoto con Tailscale:** funciona, pero con más latencia que en tu red local.
- **Apps que no funcionan:** las que dependen de comprobaciones estrictas del sistema (como Character.ai) mejor úsalas desde el navegador.

---

## 📜 Créditos

- [EasyControl (fork)](https://github.com/daitj/Easycontrol)
- [Cromite](https://github.com/uazo/cromite)
- QEMU, Tailscale y las apps de VM mencionadas, por sus respectivos autores
