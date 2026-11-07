# 🪟 Controlar Windows (XP → 11) desde un móvil antiguo con RDP

Guía complementaria del [README principal](README.md): cómo usar un **Xperia M2** (o cualquier móvil antiguo) como pantalla de una **máquina virtual Windows** en QEMU, o de un PC con Windows real, mediante **RDP** (Escritorio remoto).

> 💡 Idea rápida: Windows hace el trabajo pesado y el móvil viejo solo enseña la pantalla. El Xperia M2 se jubila, pero con trabajo a distancia. 😄

---

## 📑 Índice

1. [Qué ediciones pueden recibir conexiones RDP](#-qué-ediciones-pueden-recibir-conexiones-rdp)
2. [Aviso de seguridad (importante)](#-aviso-de-seguridad-importante)
3. [Paso 1: activar el Escritorio remoto](#-paso-1-activar-el-escritorio-remoto)
4. [Paso 2: preparar la red](#-paso-2-preparar-la-red)
5. [Paso 3: elegir el cliente en el móvil antiguo](#-paso-3-elegir-el-cliente-en-el-móvil-antiguo)
6. [Paso 4: conectar](#-paso-4-conectar)
7. [Notas para QEMU](#-notas-para-qemu)
8. [Problemas comunes](#-problemas-comunes)

---

## ✅ Qué ediciones pueden recibir conexiones RDP

Todas las versiones de Windows pueden **conectarse** a otro equipo (ser cliente), pero solo algunas ediciones pueden **recibir** conexiones (ser "host"). Eso es lo que necesitas para que tu móvil se conecte a ellas.

| Versión | ✅ Pueden recibir RDP | ❌ No pueden recibir RDP |
|---|---|---|
| **Windows XP** | Professional (y Tablet PC) | Home, Media Center* |
| **Windows Vista** | Business, Enterprise, Ultimate | Home Basic, Home Premium |
| **Windows 7** | Professional, Enterprise, Ultimate | Starter, Home Basic, Home Premium |
| **Windows 8 / 8.1** | Pro, Enterprise | Edición base ("Windows 8") |
| **Windows 10** | Pro, Enterprise, Education, Pro for Workstations, IoT Enterprise | Home |
| **Windows 11** | Pro, Enterprise, Education, Pro for Workstations | Home |
| **Windows Server** (todas) | Todas las ediciones | — |

\* En Media Center, comprueba en *Propiedades del sistema → Remoto* si aparece la opción. Si no aparece, no se puede.

**Notas útiles:**

- Las ediciones **Enterprise** funcionan igual que las Pro para RDP: se activa de la misma manera.
- En Windows de escritorio (XP a 11), normalmente solo cabe **una sesión a la vez**: si te conectas por RDP mientras alguien usa el PC, se bloquea la sesión local. Es normal, no es un fallo.
- Si tienes **Home** y quieres RDP, tus opciones son actualizar a Pro (con licencia) o usar otra herramienta de acceso remoto. Hay parches no oficiales, pero se rompen con las actualizaciones y no los recomiendo.

---

## 🛡️ Aviso de seguridad (importante)

- **XP, Vista y 7 no reciben parches de seguridad** desde hace años y tienen fallos de RDP muy conocidos (por ejemplo BlueKeep, que afecta a XP, Vista y 7). **Nunca** expongas el puerto `3389` a Internet con ellos.
- Úsalos solo en **red local**, en una **VM aislada**, o detrás de una VPN como Tailscale.
- Windows 10 llegó a su fin de soporte general en octubre de 2025, salvo ediciones LTSC o programas de actualizaciones extendidas. Aplica lo mismo si no tiene parches.
- Todas las cuentas que uses para conectar deben tener **contraseña**. Windows suele bloquear el acceso remoto con cuentas sin contraseña.

---

## 🔧 Paso 1: activar el Escritorio remoto

Hazlo **dentro del Windows** al que te quieres conectar (la VM o el PC real).

### Windows XP Professional

1. Clic derecho en **Mi PC → Propiedades**.
2. Pestaña **Remoto**.
3. Marca **Permitir que los usuarios se conecten remotamente a este equipo**.
4. Pulsa **Seleccionar usuarios remotos** y añade tu usuario si no es administrador.

### Windows Vista, 7 y 8 / 8.1

1. Clic derecho en **Equipo (o Este equipo) → Propiedades**.
2. **Configuración de Acceso remoto** (a la izquierda).
3. En la sección Escritorio remoto, elige:
   - En **Vista / 7:** *Permitir conexiones desde equipos que ejecuten cualquier versión de Escritorio remoto (menos seguro)*. Los clientes antiguos de Android suelen necesitar esta opción.
   - En **8 / 8.1:** *Permitir conexiones remotas a este equipo*, y desmarca la casilla de NLA si el cliente no conecta.
4. Pulsa **Seleccionar usuarios** y añade tu usuario si hace falta.

### Windows 10 y 11 (Pro, Enterprise, Education)

1. **Configuración → Sistema → Escritorio remoto**.
2. Activa **Escritorio remoto**.
3. Si el móvil viejo no consigue conectar, entra en las opciones avanzadas y desactiva **Requerir que los dispositivos usen la autenticación a nivel de red (NLA)**.
   > ⚠️ Desactivar NLA es menos seguro. Hazlo solo en red local o con Tailscale.
4. En **Seleccionar usuarios que pueden acceder de forma remota**, añade tu usuario si no es administrador.

### Extras en todas las versiones

- Desactiva la **suspensión** del equipo, o la VM se dormirá y te dejará tirada. 😴
- Comprueba que el firewall permite **Escritorio remoto** en red **privada** (puerto TCP `3389`).
- Si usas **XP SP3** y el cliente exige NLA, necesitarás habilitar CredSSP siguiendo el procedimiento oficial de Microsoft (artículo KB951608).

---

## 🌐 Paso 2: preparar la red

Necesitas saber la **IP** del Windows al que te vas a conectar.

1. Abre el símbolo del sistema (`cmd`) en ese Windows.
2. Ejecuta:

   ```
   ipconfig
   ```

3. Apunta la **dirección IPv4**.

Después, elige cómo conectar:

| Situación | Qué hacer |
|---|---|
| **VM en QEMU con `br0`** | La VM tiene IP en tu red local. Usa esa IP. |
| **PC con Windows real en tu red local** | Usa su IP local. |
| **Windows 10 / 11 + Tailscale** | Instala Tailscale en el Windows y en el móvil, y usa la IP `100.x.y.z`. |
| **XP, Vista, 7, 8.1** | Usa red local con `br0`. Tailscale puede no soportar esas versiones, así que no cuentes con él. |

Si quieres acceder por Tailscale a una VM de Windows que no puede tener Tailscale, mira las opciones del [README principal](README.md) (redirección de puertos con `hostfwd` o `socat`).

---

## 📱 Paso 3: elegir el cliente en el móvil antiguo

| Cliente | Notas |
|---|---|
| **Microsoft Remote Desktop 8.1.56.294** ✅ | Versión antigua compatible con **Android 4.1 o superior** y con procesadores ARM de 32 bits (`armeabi` / `armeabi-v7a`), como el del Xperia M2. Es la recomendada. [Descargar desde APKMirror](https://www.apkmirror.com/apk/microsoft-corporation/microsoft-remote-desktop/microsoft-remote-desktop-8-1-56-294-release/microsoft-remote-desktop-8-1-56-294-android-apk-download/) |
| **aRDP** (basado en FreeRDP) | Alternativa para Android antiguo. Prueba si se instala en tu dispositivo. |

### Cómo instalar el APK antiguo

1. Descarga el APK desde el enlace de APKMirror de la tabla anterior (paquete `com.microsoft.rdc.android`).
2. En el móvil antiguo, permite instalar apps de **orígenes desconocidos** (en Android 5.1 está en *Ajustes → Seguridad*).
3. Abre el APK e instálalo.

> ℹ️ La app oficial de Microsoft Remote Desktop fue retirada en Android y sustituida por **Windows App**, que probablemente no funcione en Android 5.1. Por eso se usa un APK antiguo.
>
> En la misma página de APKMirror aparecen versiones más nuevas para Android 4.1+ (hasta la 8.1.71.387, de junio de 2019). Si la 8.1.56.294 te da problemas, puedes probar una de esas. Las versiones posteriores no indican en la lista la compatibilidad con Android antiguo, así que no las he comprobado.
>
> Descarga siempre APKs de fuentes de confianza. 🙃

---

## 🔌 Paso 4: conectar

1. En el cliente del móvil, añade un equipo nuevo.
2. **PC / Host:** la IP del Windows (por ejemplo `192.168.1.50` o `100.x.y.z`). Puerto por defecto: `3389`.
3. **Usuario y contraseña:** los de tu cuenta de Windows.
4. Si el cliente lo permite, ajusta estas opciones para que vaya fluido en el Xperia M2:
   - **Resolución:** `540x960` (la del Xperia M2).
   - **Color:** 16 bits.
   - **Desactiva** fondo de escritorio, animaciones y temas.
5. Conecta y acepta el aviso del certificado si aparece (es normal en RDP).

✅ Listo, ya tienes Windows en el Xperia M2.

---

## 💻 Notas para QEMU

Si tu Windows es una VM en QEMU:

- **Tarjeta de red:**
  - **XP:** usa `rtl8139`, que tiene driver en XP.
  - **Vista a 11:** `e1000` suele funcionar sin instalar drivers. También puedes usar `virtio-net-pci` si instalas los drivers `virtio-win`.
- **Con el puente `br0`**, algo así:

  ```bash
  -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
  -device e1000,netdev=net0
  ```

  Cambia `e1000` por `rtl8139` si la VM es XP.
- **Windows 11** exige **UEFI y TPM 2.0** para instalarse. En QEMU se suele resolver con OVMF y `swtpm`. Es otra aventura aparte. 🏔️
- Para fluidez, usa **KVM** si tu CPU lo permite (`-enable-kvm`).

---

## 🆘 Problemas comunes

| Síntoma | Posible causa y solución |
|---|---|
| **No conecta** | Comprueba que la IP es correcta, que RDP está activado y que el firewall permite el puerto `3389`. |
| **Error de NLA o "autenticación"** | Desactiva NLA en Windows (solo en red local) o usa un cliente más moderno. |
| **Error de cifrado o certificado** | Puede ser incompatibilidad TLS entre un cliente muy viejo y un Windows moderno (o al revés). Prueba otro cliente. |
| **Usuario o contraseña rechazados** | Asegúrate de que la cuenta tiene contraseña y está en *Usuarios de Escritorio remoto*. |
| **Pantalla negra o muy lenta** | Baja la resolución y la profundidad de color, y desactiva efectos visuales. |
| **Se desconecta solo** | Revisa que Windows no se esté suspendiendo. |
| **La VM no tiene red** | Revisa `br0`, `tap0` y el modelo de tarjeta de red de QEMU. |
