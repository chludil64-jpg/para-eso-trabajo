# Automatización de Cotizaciones — Para Eso Trabajo
## ManyChat + Make.com + Firebase

---

## Cómo funciona

```
Cliente escribe en Instagram o Facebook
              ↓
  ManyChat responde automáticamente
  con botones y mensajes del bot
              ↓
  Make.com detecta palabras clave
  cada 15 minutos
              ↓
  Escribe en Firebase → nodo cotizaciones/
              ↓
  Aparece en la app → pestaña 💬 Redes
```

---

## Cuentas necesarias

| Herramienta | Plan | Costo | Link |
|-------------|------|-------|------|
| ManyChat | Free | $0 | manychat.com |
| Make.com | Free (1000 ops/mes) | $0 | make.com |
| Firebase | Spark (gratuito) | $0 | console.firebase.google.com |

---

## PASO 1 — Firebase: agregar regla para cotizaciones

Tus reglas actuales están bien. Solo hay que agregar el nodo `cotizaciones` para que Make.com pueda escribir ahí y la app pueda leerlo.

1. Ir a [Firebase Console](https://console.firebase.google.com)
2. Seleccionar el proyecto **para-eso-trabajo**
3. En el menú izquierdo: **Realtime Database → Rules**
4. Reemplazar las reglas con esto (es igual a las tuyas, solo se agrega `cotizaciones` al final):

```json
{
  "rules": {
    "pedidos": {
      ".read": true,
      ".write": true,
      "$pedido_id": {
        ".validate": "newData.hasChildren(['id', 'cliente_id', 'desc', 'precio', 'estado', 'fecha'])"
      }
    },
    "clientes": {
      ".read": true,
      ".write": true,
      "$cliente_id": {
        ".validate": "newData.hasChildren(['id', 'nombre', 'apellido'])"
      }
    },
    "filamentos": {
      ".read": true,
      ".write": true,
      "$filamento_id": {
        ".validate": "newData.hasChildren(['id', 'material', 'stock', 'precio'])"
      }
    },
    "cotizaciones": {
      ".read": true,
      ".write": true
    }
  }
}
```

5. Clic en **Publicar**

---

## PASO 2 — ManyChat: configurar el bot

### 2.1 Crear el flujo

1. Ir a [ManyChat](https://manychat.com) con tu cuenta conectada a Meta
2. **Automation → Flows → New Flow**
3. Nombre: `Bienvenida Para Eso Trabajo`

---

### 2.2 Triggers (cuándo se activa el bot)

Agregar dos tipos de trigger al flujo:

**Trigger 1 — New Conversation**
Activa el bot cuando alguien escribe por primera vez.

**Trigger 2 — Keywords**
Palabras que activan el bot si alguien las escribe:
```
hola, buenas, info, precio, cotización, cotizacion, quiero, consulta
```

---

### 2.3 Mensaje 1 — Bienvenida con botones

Agregar bloque tipo **Text + Buttons**:

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

Conectar el botón `💬 Quiero una cotización` (y el botón "Sí, quiero cotizar" del paso anterior) a este bloque:

**Texto:**
```
¡Perfecto! 🙌

Para cotizarte necesitamos ver el modelo.
Envianos una 📷 foto o el 🔗 link del modelo que te interesa y te respondemos con el precio a la brevedad.
```

---

### 2.6 Publicar

Clic en **Publish** arriba a la derecha. El bot ya está activo.

---

## PASO 3 — Make.com: capturar mensajes y enviar a Firebase

### 3.1 Crear cuenta

1. Ir a [make.com](https://make.com)
2. Registrarse (plan gratuito)

---

### 3.2 Crear el escenario

1. Clic en **Create a new scenario**
2. Nombre: `PET — Cotizaciones Facebook`

---

### 3.3 Módulo 1 — Trigger: Facebook Messenger

1. Clic en el círculo del inicio → buscar **Facebook Messenger**
2. Seleccionar **Watch Messages**
3. Clic en **Add** → conectar tu cuenta de Facebook
4. Seleccionar la página: **Para Eso Trabajo**
5. **Maximum number of returned messages**: `5`
6. Clic en **OK**

---

### 3.4 Filtro — Solo mensajes con palabras clave

Entre el Módulo 1 y el Módulo 2 agregar un filtro:

1. Pasar el mouse sobre la línea entre módulos → ícono de llave inglesa 🔧 → **Set up a filter**
2. **Label**: `Es cotización`
3. Agregar condiciones con **OR** entre ellas (no AND):

| Campo | Operador | Valor |
|-------|----------|-------|
| Message Text | Contains (case insensitive) | `cotiz` |
| Message Text | Contains (case insensitive) | `precio` |
| Message Text | Contains (case insensitive) | `cuanto` |
| Message Text | Contains (case insensitive) | `cuánto` |
| Message Text | Contains (case insensitive) | `quiero` |
| Message Text | Contains (case insensitive) | `presupuesto` |
| Message Text | Contains (case insensitive) | `modelo` |

> Usar **OR** para que cualquiera de esas palabras active el paso siguiente.

---

### 3.5 Módulo 2 — HTTP: escribir en Firebase

1. Agregar módulo → buscar **HTTP** → seleccionar **Make a request**
2. Completar así:

**URL:**
```
https://para-eso-trabajo-default-rtdb.firebaseio.com/cotizaciones.json
```

**Method:** `POST`

**Body type:** `Raw`

**Content type:** `application/json`

**Body** (copiar exactamente):
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

1. Toggle **Scheduling** (abajo a la izquierda) → activar
2. Intervalo: **Every 15 minutes**
3. Clic en el ícono 💾 **Save**
4. Clic en **Turn on**

---

### 3.7 Escenario para Instagram (opcional)

Repetir los pasos 3.2 a 3.6 con un segundo escenario llamado `PET — Cotizaciones Instagram`:

- Módulo 1: **Instagram Business → Watch Direct Messages**
- En el body del Módulo 2, cambiar `"plataforma": "Facebook"` por `"plataforma": "Instagram"`

> Requiere que Instagram esté configurado como cuenta Business y vinculada a tu Fan Page de Facebook.

---

## PASO 4 — Probar que todo funciona

1. Desde otra cuenta de Instagram o Facebook, mandar un mensaje a tu página con la palabra **"precio"**
2. ManyChat debería responder automáticamente con el menú de bienvenida
3. Esperar hasta 15 minutos (Make.com revisa cada 15 min en plan gratuito)
4. Abrir la app → pestaña **💬 Redes**
5. Debería aparecer la solicitud con el nombre y el mensaje del cliente

---

## Gestión de solicitudes en la app

Cuando llega una cotización nueva a la pestaña **💬 Redes**:

| Botón | Qué hace |
|-------|----------|
| ✅ Respondido | Marca la solicitud como respondida (se atenúa) |
| 📦 Crear pedido | Abre el formulario de nuevo pedido con la descripción pre-cargada |
| 🗑️ Eliminar | Borra la solicitud de la lista |

El badge con el número de pendientes aparece en el ícono 💬 del menú hasta que marcás todas como respondidas.

---

## Limitaciones del plan gratuito

| Limitación | Detalle |
|-----------|---------|
| Make.com polling | Revisa mensajes cada **15 minutos** (no instantáneo) |
| Make.com operaciones | Máximo **1000 por mes** (~33 cotizaciones detectadas por día) |
| ManyChat Free | Sin integración directa con servicios externos |

Si el volumen crece: **Make.com Core ($9/mes)** baja el polling a 1 minuto.

---

## Solución de problemas

**El bot no responde**
→ Verificar que el flujo esté publicado en ManyChat.
→ Verificar que la página de Facebook esté conectada a ManyChat.

**No aparecen solicitudes en la app**
→ Verificar que el escenario de Make.com esté activo (toggle verde).
→ Verificar que las reglas de Firebase incluyan el nodo `cotizaciones`.
→ Esperar hasta 15 minutos desde que se envió el mensaje de prueba.

**Make.com da error 401 o 403 en el módulo HTTP**
→ Las reglas de Firebase no incluyen `cotizaciones`. Verificar el Paso 1.

**Make.com da error de conexión a Facebook**
→ Reconectar la cuenta de Facebook en Make.com: clic en el módulo → Edit → reconnect.

**La app muestra "sin conexión"**
→ Las reglas de Firebase bloquearon la lectura. Verificar que `.read: true` esté en `pedidos`, `clientes` y `filamentos`.
