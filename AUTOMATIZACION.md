# Automatización de Cotizaciones — Para Eso Trabajo
## ManyChat + Make.com + Firebase

---

## Cómo funciona todo junto

```
Cliente escribe en Instagram o Facebook
            ↓
  ManyChat responde automáticamente
  (bot con botones y mensajes)
            ↓
  Make.com detecta palabras clave
  cada 15 minutos
            ↓
  Escribe en Firebase
            ↓
  Aparece en la app → pestaña 💬 Redes
```

---

## PASO 1 — Firebase: habilitar escritura de alertas

Antes de configurar nada, hay que permitir que Make.com escriba en la base de datos.

1. Ir a [Firebase Console](https://console.firebase.google.com)
2. Seleccionar el proyecto **para-eso-trabajo**
3. En el menú izquierdo: **Realtime Database → Rules**
4. Reemplazar las reglas existentes con esto:

```json
{
  "rules": {
    ".read": "auth != null",
    "cotizaciones": {
      ".write": true
    }
  }
}
```

5. Clic en **Publicar**

> ⚠️ Esto permite que Make.com escriba alertas sin autenticación. Solo la ruta `cotizaciones/` queda abierta; el resto sigue protegido.

---

## PASO 2 — ManyChat: configurar el bot

ManyChat maneja la conversación automática con los clientes en Instagram y Facebook.

### 2.1 Crear el flujo principal

1. Ir a [ManyChat](https://manychat.com) → tu cuenta conectada a Meta
2. En el menú: **Automation → Flows → New Flow**
3. Nombrarlo: `Bienvenida Para Eso Trabajo`

---

### 2.2 Configurar el Trigger (cuándo se activa)

Agregar estos triggers al flujo:

- **New Conversation** — se activa cuando alguien escribe por primera vez
- **Keywords** — agregar las palabras: `hola`, `buenas`, `info`, `precio`, `cotización`, `cotizacion`, `quiero`

---

### 2.3 Mensaje 1 — Bienvenida

Agregar un bloque de mensaje tipo **Text + Buttons**:

**Texto:**
```
¡Hola! 👋 Bienvenido/a a Para Eso Trabajo.
¿En qué podemos ayudarte?
```

**Botones (Quick Replies):**
- `🖨️ Ver modelos disponibles`
- `💬 Quiero una cotización`

---

### 2.4 Mensaje 2 — Si elige "Ver modelos"

Conectar el botón `🖨️ Ver modelos disponibles` a un nuevo bloque:

**Texto:**
```
Encontrás todos nuestros modelos disponibles en MakerWorld 👇
https://makerworld.com/es

¿Querés cotizar alguno?
```

**Botón:**
- `Sí, quiero cotizar` → conectar al Mensaje 3

---

### 2.5 Mensaje 3 — Solicitud de cotización

Conectar el botón `💬 Quiero una cotización` (y el botón del paso anterior) a este bloque:

**Texto:**
```
¡Perfecto! 🙌

Para cotizarte necesitamos ver el modelo.

Envianos una 📷 foto o el 🔗 link del modelo que te interesa y te respondemos con el precio a la brevedad.
```

---

### 2.6 Publicar el flujo

1. Clic en **Publish** (arriba a la derecha)
2. El bot ya está activo en Facebook e Instagram

---

## PASO 3 — Make.com: capturar mensajes y enviar a Firebase

Make.com escucha los mensajes de Facebook y cuando detecta una solicitud de cotización, la escribe automáticamente en Firebase para que aparezca en tu app.

### 3.1 Crear cuenta

1. Ir a [make.com](https://make.com)
2. Registrarse con el email del negocio (plan gratuito)

---

### 3.2 Crear el escenario

1. Clic en **Create a new scenario**
2. Nombrarlo: `Para Eso Trabajo — Cotizaciones`

---

### 3.3 Módulo 1 — Trigger: Facebook Messenger

1. Clic en el círculo de inicio → buscar `Facebook Messenger`
2. Seleccionar **Watch Messages**
3. Clic en **Add** para conectar tu cuenta de Facebook
4. Seleccionar la página: **Para Eso Trabajo**
5. En **Maximum number of returned messages**: `5`
6. Clic en **OK**

---

### 3.4 Módulo 2 — Filtro de palabras clave

Entre el Módulo 1 y el Módulo 3 hay que agregar un filtro para que solo pasen los mensajes relevantes.

1. Pasar el mouse sobre la línea que conecta los módulos → clic en la llave inglesa 🔧 → **Set up a filter**
2. **Label**: `Es solicitud de cotización`
3. **Condition**:
   - Campo: `Message Text`
   - Operador: `Contains (case insensitive)`
   - Valor: `cotiz`
4. Clic en **Add AND rule** y repetir para cada palabra:
   - `precio`
   - `cuánto`
   - `cuanto`
   - `quiero`
   - `presupuesto`
   - `modelo`

> Usar **OR** entre las condiciones (no AND) para que cualquiera de esas palabras active el flujo.

---

### 3.5 Módulo 3 — HTTP: escribir en Firebase

1. Agregar módulo → buscar `HTTP` → seleccionar **Make a request**
2. Completar los campos:

| Campo | Valor |
|-------|-------|
| URL | `https://para-eso-trabajo-default-rtdb.firebaseio.com/cotizaciones.json` |
| Method | `POST` |
| Body type | `Raw` |
| Content type | `application/json` |

**Body (copiar y pegar exactamente):**
```json
{
  "id": "{{formatDate(now; \"X\")}}",
  "plataforma": "Facebook",
  "nombre": "{{1.sender.name}}",
  "usuario": "{{1.sender.id}}",
  "mensaje": "{{1.message.text}}",
  "imagen_url": "",
  "fecha": "{{formatDate(now; \"X\") * 1000}}",
  "estado": "pendiente"
}
```

3. Clic en **OK**

---

### 3.6 Activar el escenario

1. Abajo a la izquierda: activar el toggle **Scheduling**
2. Intervalo: **Every 15 minutes** (plan gratuito)
3. Clic en **Save** (ícono del disquete)
4. Clic en **Turn on** para activarlo

---

### 3.7 Para Instagram (opcional)

Repetir los pasos 3.3 a 3.6 pero con un segundo escenario:

- Módulo 1: `Instagram Business → Watch Direct Messages`
- En el body del HTTP cambiar `"plataforma": "Facebook"` por `"plataforma": "Instagram"`

> Requiere que Instagram esté conectado como cuenta Business y vinculada a la Fan Page de Facebook.

---

## PASO 4 — Probar que todo funciona

1. Desde otra cuenta de Instagram o Facebook, enviar un mensaje a la página con la palabra **"precio"** o **"cotización"**
2. ManyChat debería responder automáticamente con el menú de bienvenida
3. Esperar hasta 15 minutos (Make.com polling)
4. Abrir la app → pestaña **💬 Redes** → debería aparecer la solicitud

---

## Gestión de solicitudes en la app

Cuando llega una cotización a la pestaña **💬 Redes**:

| Botón | Acción |
|-------|--------|
| ✅ Respondido | Marca la solicitud como respondida (se atenúa) |
| 📦 Crear pedido | Abre el formulario de nuevo pedido con la descripción pre-cargada |
| 🗑️ Eliminar | Borra la solicitud |

---

## Resumen de cuentas necesarias

| Herramienta | Plan | Costo | Link |
|-------------|------|-------|------|
| ManyChat | Free | $0 | manychat.com |
| Make.com | Free (1000 ops/mes) | $0 | make.com |
| Firebase | Spark (gratuito) | $0 | console.firebase.google.com |

---

## Limitaciones del plan gratuito

- **Make.com**: revisa mensajes cada **15 minutos** (no es instantáneo)
- **Make.com**: máximo **1000 operaciones por mes** (~33 mensajes detectados por día)
- **ManyChat Free**: sin integración directa con servicios externos (por eso usamos Make.com)

Si el volumen crece, Make.com Pro ($9/mes) baja el polling a **1 minuto**.

---

## Solución de problemas

**El bot no responde en Instagram/Facebook**
→ Verificar que el flujo esté publicado en ManyChat y que la página esté conectada.

**No aparecen solicitudes en la app**
→ Verificar que las reglas de Firebase estén guardadas correctamente.
→ Verificar que el escenario de Make.com esté activo (toggle verde).
→ Esperar hasta 15 minutos desde que se envió el mensaje.

**Make.com da error en el módulo HTTP**
→ Verificar que la URL de Firebase sea exacta.
→ Verificar que el Body sea JSON válido (sin errores de tipeo).

**El mensaje llega pero sin nombre de usuario**
→ En Make.com, ir al módulo HTTP y verificar que `{{1.sender.name}}` esté correctamente mapeado desde el Módulo 1.
