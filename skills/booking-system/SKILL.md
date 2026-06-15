---
inclusion: manual
---

# Booking/Scheduling System Implementation Skill

Guía completa para implementar un sistema de agendamiento multi-paso en cualquier proyecto React + TypeScript. Independiente de diseño visual, contenido o integraciones específicas.

## Arquitectura General

El sistema sigue un patrón de **wizard multi-paso con state machine** contenido en un drawer/modal lateral:

```
TriggerButton (abre el drawer)
  └─ DrawerContainer (Sheet/Dialog lateral, reset al reabrir)
       └─ FormOrchestrator (state machine central)
            ├─ Step1: Información del cliente
            ├─ Step2: Selección de fecha y hora
            ├─ Step3: Confirmación y envío
            └─ SuccessView: Estado final confirmado
```

---

## Stack Tecnológico Recomendado

### Frontend
- **React 18+** con TypeScript
- **shadcn/ui** (Sheet, Calendar, Button, Skeleton, Select, Input)
- **Tailwind CSS** para estilos
- **date-fns** para manipulación de fechas
- **react-day-picker** (viene con shadcn Calendar)
- **lucide-react** para íconos

### Backend (API de disponibilidad y reservas)
- Cualquier backend que exponga 2 endpoints REST:
  - `GET /availability?startDate=YYYY-MM-DD&endDate=YYYY-MM-DD`
  - `POST /bookings` (crea la reserva)
- Opciones comunes: Cloudflare Workers, Next.js API routes, Express, Firebase Functions

### Integraciones de calendario
- Google Calendar API (recomendado para Google Meet automático)
- Microsoft Graph API (para Outlook/Teams)
- Cal.com, Calendly (APIs externas)
- Base de datos propia con lógica de disponibilidad

---

## Implementación Paso a Paso

### 1. Tipos y Contratos (types.ts)

```typescript
/** Paso actual del flujo */
export type BookingStep = 1 | 2 | 3 | "success";

/** Un slot de tiempo disponible */
export interface TimeSlot {
  date: string;           // "2025-08-15" (YYYY-MM-DD)
  startTime: string;      // "14:00" (hora UTC o timezone del server)
  endTime: string;        // "14:30"
  timezone: string;       // IANA timezone del servidor
  startISO: string;       // "2025-08-15T14:00:00Z" (UTC completo)
  endISO: string;         // "2025-08-15T14:30:00Z"
}

/** Payload enviado al backend para crear la reserva */
export interface BookingPayload {
  name: string;
  email: string;
  service: string;
  company?: string;
  message?: string;
  slotStartISO: string;
  slotEndISO: string;
  clientTimezone: string;  // IANA timezone del navegador
  locale?: string;
}

/** Respuesta del backend al confirmar la reserva */
export interface ConfirmedEvent {
  eventId: string;
  slotStartISO: string;
  slotEndISO: string;
  meetLink?: string;       // URL de videoconferencia (si aplica)
  calendarLink?: string;   // Link al evento en el calendario
}

/** Estado completo del formulario */
export interface BookingFormState {
  step: BookingStep;
  // Step 1
  name: string;
  email: string;
  service: string;
  company: string;
  message: string;
  // Step 2
  selectedDate: Date | null;
  selectedSlot: TimeSlot | null;
  availableSlots: TimeSlot[];
  slotsLoading: boolean;
  slotsError: string | null;
  // Step 3 / Success
  confirmedEvent: ConfirmedEvent | null;
  bookingLoading: boolean;
  bookingError: string | null;
  retryCount: number;
  // Timezone
  clientTimezone: string;
}

/** Error personalizado con código HTTP */
export class AppointmentError extends Error {
  constructor(message: string, public readonly status: number) {
    super(message);
    this.name = "AppointmentError";
  }
}
```

### 2. Cliente API (appointmentClient.ts)

```typescript
const API_BASE_URL = import.meta.env.VITE_BOOKING_API_URL;

export async function getAvailability(startDate: string, endDate: string): Promise<TimeSlot[]> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 10000);

  try {
    const response = await fetch(
      `${API_BASE_URL}/availability?startDate=${encodeURIComponent(startDate)}&endDate=${encodeURIComponent(endDate)}`,
      { signal: controller.signal }
    );
    clearTimeout(timeoutId);

    if (!response.ok) {
      const body = await response.json();
      throw new AppointmentError(body.error || "Failed to fetch availability", response.status);
    }

    const body = await response.json();
    return body.data;
  } catch (error) {
    clearTimeout(timeoutId);
    if (error instanceof AppointmentError) throw error;
    if (error instanceof DOMException && error.name === "AbortError") {
      throw new AppointmentError("Request timed out", 408);
    }
    throw new AppointmentError(error instanceof Error ? error.message : "Network error", 0);
  }
}

export async function createBooking(payload: BookingPayload): Promise<ConfirmedEvent> {
  const response = await fetch(`${API_BASE_URL}/bookings`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload),
  });

  if (!response.ok) {
    const body = await response.json();
    throw new AppointmentError(body.error || "Failed to create booking", response.status);
  }

  const body = await response.json();
  return body.data;
}
```

### 3. Utilidades de Timezone (timezoneUtils.ts)

Usar exclusivamente la API `Intl.DateTimeFormat` del navegador — sin librerías externas de timezone.

```typescript
export function detectClientTimezone(): string {
  try {
    return Intl.DateTimeFormat().resolvedOptions().timeZone || "UTC";
  } catch {
    return "UTC";
  }
}

export function formatSlotForDisplay(
  utcStartISO: string,
  utcEndISO: string,
  timezone: string,
  locale: string = "en"
): string {
  const start = new Date(utcStartISO);
  const end = new Date(utcEndISO);
  const intlLocale = locale === "es" ? "es-CO" : "en-US";

  const fmt = new Intl.DateTimeFormat(intlLocale, {
    hour: "numeric", minute: "2-digit", hour12: true, timeZone: timezone,
  });
  const fmtTz = new Intl.DateTimeFormat(intlLocale, {
    hour: "numeric", minute: "2-digit", hour12: true, timeZone: timezone, timeZoneName: "short",
  });

  const tzAbbr = fmtTz.formatToParts(end).find(p => p.type === "timeZoneName")?.value ?? "";
  return `${fmt.format(start)} – ${fmt.format(end)} (${tzAbbr})`;
}

export function formatSlotDual(
  utcStartISO: string,
  clientTimezone: string,
  serverTimezone: string,
  locale: string = "en"
) {
  const date = new Date(utcStartISO);
  const intlLocale = locale === "es" ? "es-CO" : "en-US";
  const format = (tz: string) => new Intl.DateTimeFormat(intlLocale, {
    hour: "numeric", minute: "2-digit", hour12: true, timeZone: tz, timeZoneName: "short",
  }).format(date);

  return { client: format(clientTimezone), server: format(serverTimezone) };
}
```

### 4. Componente Orquestador (BookingForm.tsx)

Patrón clave: un solo `useState<BookingFormState>` que controla toda la máquina de estados.

**Transiciones:**
- Step 1 → Step 2: Al validar info del cliente, se dispara `fetchAvailability()`
- Step 2 → Step 3: Al seleccionar un slot y confirmar
- Step 3 → Success: Al recibir respuesta exitosa del backend
- Step 3 → Step 2 (HTTP 409): Si el slot ya fue tomado, vuelve a step 2 con disponibilidad fresca
- Cualquier step → step anterior: Navegación back preservando todos los datos ingresados

**Lógica de reinicio:** El componente padre (Drawer) remonta el formulario con un `key` incremental al reabrir, evitando un `resetForm()` manual.

**Fetch de disponibilidad:**
- Rango: próximos 14 días desde hoy
- Filtro frontend: excluir slots que inicien en menos de 4 horas
- El backend también debe validar este filtro para evitar race conditions

### 5. Step 1: Información del Cliente

**Campos estándar:**
- Nombre completo (requerido, max 100 chars)
- Email (requerido, validación RFC 5322)
- Empresa (opcional, max 100 chars)
- Servicio (requerido, select con opciones del negocio)
- Mensaje (opcional, max 500 chars con contador)

**Validación:**
- Client-side en submit con `noValidate` en el form (control total de UX)
- Errores inline con `aria-describedby` apuntando al mensaje de error
- Limpieza progresiva: cada campo limpia su error al corregirse

### 6. Step 2: Selección de Slot (patrón sub-steps)

El step 2 usa internamente dos vistas:

**Sub-step "calendar":**
- Componente `<Calendar>` (react-day-picker via shadcn)
- Dots indicadores en días con disponibilidad (`modifiers` + `modifiersClassNames`)
- Días sin slots deshabilitados via prop `disabled`
- Al seleccionar un día, transiciona al sub-step "time"

**Sub-step "time":**
- Grid de 2 columnas con botones por cada slot
- Formato: hora local del cliente con abreviación de timezone
- `aria-pressed` para indicar selección
- Botón "Back" vuelve al calendario (no al Step 1)

**Estados adicionales:**
- Loading: Skeletons animados con `aria-busy="true"`
- Error: Mensaje + botón "Retry"
- Vacío: Mensaje informativo + alternativa de contacto

### 7. Step 3: Confirmación

**Contenido:**
- Resumen tipo `<dl>` con todos los datos del booking
- Display dual de timezone: hora del cliente + hora del servidor/prestador
- Botón "Confirmar" con estado loading (spinner icon + texto)

**Manejo de errores:**
- Error genérico: muestra mensaje + botón "Retry"
- Máximo de reintentos (3): deshabilita retry, muestra mensaje final
- HTTP 409 (slot tomado): redirección automática a Step 2 con re-fetch de disponibilidad

### 8. Vista de Éxito

- Ícono de check animado (pulse-glow)
- Detalles confirmados: fecha/hora, servicio, email
- Nota sobre invitación de calendario enviada
- Acciones: "Cerrar" (si está en drawer) o "Agendar otra" (reset)

### 9. Drawer/Container

```typescript
// Patrón de reset via key prop
const [formKey, setFormKey] = useState(0);

useEffect(() => {
  if (open && !prevOpenRef.current) {
    setFormKey(k => k + 1); // Remonta el form con estado limpio
  }
  prevOpenRef.current = open;
}, [open]);

return (
  <Sheet open={open} onOpenChange={onOpenChange}>
    <SheetContent side="right" className="w-full sm:max-w-md overflow-y-auto">
      <BookingForm key={formKey} onDone={() => onOpenChange(false)} />
    </SheetContent>
  </Sheet>
);
```

### 10. Trigger (BookingCTA)

Componente self-contained que maneja su propio estado open/close:

```typescript
export function BookingCTA({ service, children, className, size }) {
  const [open, setOpen] = useState(false);
  return (
    <>
      <Button size={size} className={className} onClick={() => setOpen(true)}>
        {children}
      </Button>
      <BookingDrawer open={open} onOpenChange={setOpen} defaultService={service} />
    </>
  );
}
```

---

## Contrato de API del Backend

### GET /availability

**Query params:**
- `startDate`: YYYY-MM-DD (inicio del rango)
- `endDate`: YYYY-MM-DD (fin del rango)

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "date": "2025-08-15",
      "startTime": "14:00",
      "endTime": "14:30",
      "timezone": "America/Bogota",
      "startISO": "2025-08-15T19:00:00Z",
      "endISO": "2025-08-15T19:30:00Z"
    }
  ]
}
```

### POST /bookings

**Request body:** `BookingPayload` (ver tipos arriba)

**Response (201):**
```json
{
  "success": true,
  "data": {
    "eventId": "abc123",
    "slotStartISO": "2025-08-15T19:00:00Z",
    "slotEndISO": "2025-08-15T19:30:00Z",
    "meetLink": "https://meet.google.com/xxx",
    "calendarLink": "https://calendar.google.com/event?eid=xxx"
  }
}
```

**Error responses:**
- 400: Validación fallida (`{ "success": false, "error": "Invalid email format" }`)
- 409: Slot ya tomado (`{ "success": false, "error": "This time slot is no longer available" }`)
- 500: Error interno del servidor

---

## Accesibilidad (WCAG 2.1 AA)

- `aria-current="step"` en el paso activo del indicador de progreso
- `aria-invalid` y `aria-describedby` en campos con error de validación
- `role="alert"` con `aria-live="polite"` en mensajes de error dinámicos
- Mínimo **44×44px** en todos los targets táctiles (botones, slots)
- Focus trap automático en el drawer (Radix Sheet built-in)
- `aria-busy="true"` en contenedores durante estados de carga
- `aria-pressed` en botones toggle de selección de slot
- `aria-label` descriptivo en grupos de slots y navegación
- Escape cierra el drawer (Radix built-in)

---

## Variables de Entorno

```env
# URL base del backend de agendamiento
VITE_BOOKING_API_URL=http://localhost:8787

# (Opcional) Protección anti-bot - Cloudflare Turnstile
VITE_TURNSTILE_SITE_KEY=xxx
```

---

## Checklist de Implementación

- [ ] Definir tipos (TimeSlot, BookingPayload, ConfirmedEvent, BookingFormState, AppointmentError)
- [ ] Crear cliente API con timeout (10s para availability) y manejo de errores tipado
- [ ] Crear utilidades de timezone (detección automática, formato para display, formato dual)
- [ ] Implementar Step 1 — formulario con validación inline y limpieza progresiva de errores
- [ ] Implementar Step 2 — calendario con dots + grid de slots (sub-steps internos)
- [ ] Implementar Step 3 — resumen con confirmación, loading, retry, y manejo de 409
- [ ] Implementar vista de éxito con detalles confirmados y acciones
- [ ] Crear BookingForm (orquestador) con state machine completa
- [ ] Crear BookingDrawer con Sheet y reset via key prop
- [ ] Crear BookingCTA como trigger self-contained
- [ ] Agregar StepIndicator visual (3 círculos numerados con líneas conectoras)
- [ ] Configurar variable de entorno `VITE_BOOKING_API_URL`
- [ ] Implementar backend: endpoint GET /availability
- [ ] Implementar backend: endpoint POST /bookings con double-check de disponibilidad
- [ ] Agregar i18n si el proyecto lo requiere
- [ ] Verificar accesibilidad (aria-*, focus trap, touch targets 44px)
- [ ] Agregar protección anti-bot si es necesario (Turnstile, reCAPTCHA)

---

## Personalización por Proyecto

| Aspecto | Qué cambiar |
|---------|-------------|
| Campos del Step 1 | Agregar/quitar campos según necesidad del negocio |
| Opciones de servicio | Lista de opciones en el select |
| Duración del slot | 15, 30, 45, 60 min según el tipo de cita |
| Ventana de disponibilidad | Próximos N días (default: 14) |
| Tiempo mínimo de anticipación | N horas antes del slot (default: 4) |
| Timezone de referencia del negocio | La timezone del prestador del servicio |
| Integración de calendario | Google Calendar, Outlook, base de datos propia |
| Protección anti-bot | Turnstile, reCAPTCHA, honeypot |
| Notificaciones | Email, SMS, WhatsApp al confirmar |
| Idiomas soportados | Configurar según sistema i18n del proyecto |
| Videoconferencia automática | Google Meet, Zoom, Teams, o ninguna |
| Diseño visual | Adaptar a design system del proyecto (colores, tipografía, espaciado) |
| Animaciones | Transiciones entre steps, éxito (slide-up, pulse-glow) |

---

## Principios Clave

1. **UTC everywhere.** Almacenar y transmitir tiempos siempre en UTC ISO 8601. Convertir a timezone local solo para display.
2. **Double-check en backend.** El backend DEBE verificar disponibilidad al momento de crear la reserva, no confiar en el GET previo.
3. **Key prop para reset.** Más limpio y confiable que un `resetForm()` con muchos campos.
4. **Sub-steps en Step 2.** Mejora la UX en mobile evitando scroll con calendario + slots en una sola vista.
5. **No depender de librerías de timezone externas.** `Intl.DateTimeFormat` es suficiente para el 99% de los casos.
6. **Filtros redundantes.** Aplicar filtro de tiempo mínimo tanto en frontend como en backend.
7. **Graceful degradation en 409.** El conflicto de slot es esperado — redirigir suavemente, no mostrar error bloqueante.
