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
5. [¿No puedes hacer `br0`? Alternativa con Tailscale](#-no-puedes-hacer-br0-alternativa-con-tailscale)
6. [Probado en](#-probado-en)
7. [Fotos](#-Fotos-con-las-pruebas)
8. [Notas y problemas conocidos](#-notas-y-problemas-conocidos)
9. [Descarga](#-descarga-de-android)

---

## 🧰 Qué necesitas

| Elemento | Método A (móvil) | Método B (PC) |
|---|---|---|
| Móvil antiguo (Android 5.1.1 recomendado, ej. Xperia M2) | ✅ | ✅ |
| Móvil actual con app de máquinas virtuales | ✅ | ❌ |
| PC con Linux | ❌ | ✅ |
| QEMU | ❌ | ✅ |
| Puente de red `br0` (o [Tailscale](#-no-puedes-hacer-br0-alternativa-con-tailscale) si no puedes) | ❌ | ✅ |
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

> ⚠️ **el navegador Chrome** puede no funcionar en la VM con android 9, en este caso se recomienda **[Cromite](https://github.com/uazo/cromite)**

---

## 🖥️ Método B: usando un PC con Linux + QEMU

### Paso 1: instalar QEMU

```bash
sudo apt install qemu-system
```

### Paso 2: preparar y lanzar la VM

1. Asegúrate de tener creado el puente de red **`br0`** (mira la [sección siguiente](#-cómo-crear-el-puente-br0)). Sin él, el script **no te dejará continuar**.
   - ¿No puedes crear `br0` (por ejemplo, porque vas por Wi-Fi)? Salta a la [alternativa con Tailscale](#-no-puedes-hacer-br0-alternativa-con-tailscale).
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
   - Si la VM es **Windows** → usa RDP (Escritorio remoto). Mira la guía completa para todas las versiones: [README-Windows.md](README-Windows.md).
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

## 🔐 ¿No puedes hacer `br0`? Alternativa con Tailscale

Si no puedes crear el puente (vas por Wi-Fi, tu router o tu sistema no te dejan, o simplemente no quieres pelearte con la red), **Tailscale** te salva: crea una red privada entre tus dispositivos y cada uno recibe una IP tipo `100.x.y.z`. Sin puentes, sin `br0`, sin dramas. 🎭

### Antes de empezar

- Crea una cuenta en <https://tailscale.com> (sirve la de Google, Microsoft o GitHub).
- Inicia sesión **con la misma cuenta** en todos los dispositivos que vayas a conectar.

### Opción 1: Tailscale dentro de la VM y en el móvil antiguo

Es la más sencilla si tu VM es **Android** (por ejemplo, Android x86), porque la VM tiene su propia IP de Tailscale.

1. **En la VM Android:** instala Tailscale desde la Play Store o con el APK, e inicia sesión.
2. **En el móvil antiguo (Xperia M2):** instala Tailscale e inicia sesión con la misma cuenta.
3. En la VM, activa la **depuración ADB en red** (puerto `5555`).
4. Mira la IP de Tailscale de la VM dentro de la app (algo como `100.x.y.z`).
5. En el Xperia M2, abre **EasyControl (fork)**, pulsa **`+`** y escribe esa IP de Tailscale.
6. Acepta la conexión en la VM. ✅

### Opción 2: Tailscale en el PC y en la VM (mismo flujo del Método B, sin `br0`)

Es el mismo proceso del [Método B](#-método-b-usando-un-pc-con-linux--qemu), pero usando las **IPs de Tailscale** en lugar de la IP del puente. La VM y el PC se ven entre sí aunque la VM vaya con red NAT normal de QEMU.

1. **En el PC (Linux):** instala Tailscale e inicia sesión:

   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```

2. **En la VM (Android x86):** instala Tailscale e inicia sesión con la **misma cuenta**.
3. En la VM, activa la **depuración ADB en red** (puerto `5555`) y apunta su IP de Tailscale (`100.x.y.z`).
4. En tu PC, conecta ADB usando esa IP de Tailscale:

   ```bash
   adb connect IP_TAILSCALE_DE_LA_VM:5555
   ```

5. Redirige el puerto con `socat`:

   ```bash
   socat TCP-LISTEN:5556,reuseaddr,fork TCP:IP_TAILSCALE_DE_LA_VM:5555
   ```

6. **En el móvil antiguo:** abre EasyControl (fork), pulsa **`+`** y escribe la IP del **PC** con el puerto `5556`:
   - En tu red local: la IP local del PC.
   - Desde fuera de casa (con Tailscale en el móvil antiguo): la IP de Tailscale del PC (`tailscale ip -4`).
7. Acepta la conexión en la VM. ✅

> 💡 En el script de QEMU puedes usar la red por defecto (`-netdev user,id=net0 -device e1000,netdev=net0`) y olvidarte de `tap0` y `br0`. Ajusta el modelo de tarjeta a la que ya uses.

### Opción 3: Tailscale solo en el PC + redirección de puerto de QEMU

Úsala si **no puedes instalar Tailscale dentro de la VM**. Aquí el **PC** es el intermediario.

1. **En el PC (Linux):** instala Tailscale e inicia sesión:

   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```

2. Mira la IP de Tailscale de tu PC:

   ```bash
   tailscale ip -4
   ```

3. **En QEMU**, cambia la red del script para usar red "user" con redirección de puerto, en lugar de `tap0`/`br0`:

   ```bash
   -netdev user,id=net0,hostfwd=tcp::5555-:5555 \
   -device e1000,netdev=net0
   ```

   > Esto manda el puerto `5555` de tu PC al `5555` de la VM. Ajusta el modelo de tarjeta (`e1000`, `virtio-net-pci`...) al que ya uses en tu script.

4. **En el móvil antiguo:** instala Tailscale e inicia sesión con la misma cuenta.
5. En EasyControl (fork), pulsa **`+`** y escribe la **IP de Tailscale del PC** con el puerto `5555`.
6. Acepta la conexión en la VM. ✅

> 💡 Con este método **no necesitas `socat`**: la redirección de QEMU ya hace ese trabajo.

### ⚠️ Ojo con Android 5.1

Las versiones actuales de la app de Tailscale pueden **no instalarse en Android 5.1**. Si te pasa:

- Prueba con una **versión antigua** del APK de Tailscale.
- Si no hay forma, usa el móvil antiguo solo como pantalla en tu red local (con `br0`) y deja Tailscale para dispositivos más modernos.

---

## ✅ Probado en

Pantalla: Sony Xperia M2 con Android 5.1 como móvil antiguo. Todo funcionando en la práctica:

| Sistema remoto | Dónde corre | Cómo se controla |
|---|---|---|
| **Android 12** | VM en un móvil actual (Virtual Master) | EasyControl (fork) |
| **Windows 11 Pro 26H2** (build 26300.9457) | VM en QEMU, PC con AMD 3020e y 8 GB de RAM | Escritorio remoto (RDP) |
| **Windows 10 Enterprise 2016 LTSB** (MiniOS v2018.01, 32 bits) | PC real con Intel Atom N280 y 1 GB de RAM | Escritorio remoto (RDP) |

> 💡 Hasta un Atom N280 con 1 GB de RAM se deja controlar desde el Xperia M2. Y como es una edición **Enterprise**, confirma que estas ediciones funcionan como host de RDP.

## 📷 Fotos con las pruebas:

Windows 11 en QEMU:
<img width="4096" height="3072" alt="21919" src="https://github.com/user-attachments/assets/e4490099-bd4c-499f-988d-c442379f0935" />
<img width="4096" height="3072" alt="21918" src="https://github.com/user-attachments/assets/43fa861e-c64b-45eb-8b41-47b9724b1871" />

Windows 10 en pc real: 

<img width="4096" height="3072" alt="21922" src="https://github.com/user-attachments/assets/6462bc63-842c-409b-bf46-268c4bf29629" />

Virtual master en móvil actual ejecutando contenedor de android 12:

<img width="1080" height="2400" alt="21921" src="https://github.com/user-attachments/assets/861588ab-dd45-4882-9903-6f17ba3febc3" />
<img width="3072" height="4096" alt="21920" src="https://github.com/user-attachments/assets/23bc4d8f-6211-4c7f-9479-3042b610eb3a" />


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

## ⬇️ Descarga de android

- **Descarga Android 8.1 x86_64 aqui**: https://www.mediafire.com/file/33gemzr5u2lsi04/android+8.1.qcow2/file

## 📜 Créditos

- [EasyControl (fork)](https://github.com/daitj/Easycontrol)
- [Cromite](https://github.com/uazo/cromite)
- QEMU, Tailscale y las apps de VM mencionadas, por sus respectivos autores
