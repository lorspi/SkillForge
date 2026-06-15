# Webhook Handler — Convex HTTP Endpoint

## Archivo: `convex/http.ts`

El webhook recibe notificaciones de Wompi cuando una transacción cambia de estado.

## Implementación Completa

```typescript
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";
import { internal } from "./_generated/api";

const http = httpRouter();

// --- Utilidad para verificar firma del webhook ---

async function sha256Hex(message: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(message);
  const hashBuffer = await crypto.subtle.digest("SHA-256", data);
  return Array.from(new Uint8Array(hashBuffer))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

/**
 * Obtiene un valor anidado de un objeto usando notación de punto.
 * Ejemplo: getNestedValue(obj, "data.transaction.id")
 */
function getNestedValue(obj: any, path: string): any {
  return path.split(".").reduce((current, key) => current?.[key], obj);
}

/**
 * Verifica la firma del webhook de Wompi.
 *
 * Wompi incluye en el payload:
 * - signature.properties: lista de paths a los valores que se concatenan
 * - signature.checksum: SHA256(concat(values) + timestamp + eventsSecret)
 */
export async function verifyWompiSignature(
  event: any,
  eventsSecret: string
): Promise<boolean> {
  if (!event?.signature?.checksum || !event?.signature?.properties?.length) {
    return false;
  }

  const properties: string[] = event.signature.properties;
  let concat = "";

  for (const prop of properties) {
    const value = getNestedValue(event, prop);
    concat += String(value ?? "");
  }

  // Agregar timestamp y secret
  concat += String(event.timestamp) + eventsSecret;

  const computed = await sha256Hex(concat);
  return computed === event.signature.checksum;
}

// --- Webhook endpoint ---

http.route({
  path: "/webhooks/wompi",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    try {
      const body = await request.json();

      // Verificar firma (opcional pero recomendado en producción)
      const eventsSecret = process.env.WOMPI_EVENTS_SECRET;
      if (eventsSecret) {
        const isValid = await verifyWompiSignature(body, eventsSecret);
        if (!isValid) {
          console.error("Webhook signature verification failed");
          // Aún retornar 200 para que Wompi no reintente
          return new Response(JSON.stringify({ received: true }), {
            status: 200,
            headers: { "Content-Type": "application/json" },
          });
        }
      }

      // Solo procesar eventos transaction.updated
      if (body.event === "transaction.updated") {
        const transaction = body.data?.transaction;
        if (!transaction?.reference) {
          return new Response(JSON.stringify({ received: true }), {
            status: 200,
            headers: { "Content-Type": "application/json" },
          });
        }

        const status = transaction.status;
        const reference = transaction.reference;
        const transactionId = transaction.id;

        if (status === "APPROVED") {
          await ctx.runMutation(internal.payments.processApprovedPayment, {
            reference,
            wompiTransactionId: transactionId,
          });
        } else if (status === "DECLINED" || status === "VOIDED") {
          await ctx.runMutation(internal.payments.processDeclinedOrVoidedPayment, {
            reference,
            status: status.toLowerCase() as "declined" | "voided",
            wompiTransactionId: transactionId,
          });
        }
      }

      return new Response(JSON.stringify({ received: true }), {
        status: 200,
        headers: { "Content-Type": "application/json" },
      });
    } catch (error) {
      console.error("Webhook processing error:", error);
      // SIEMPRE retornar 200 para evitar reintentos infinitos de Wompi
      return new Response(JSON.stringify({ received: true }), {
        status: 200,
        headers: { "Content-Type": "application/json" },
      });
    }
  }),
});

export default http;
```

## Estructura del Payload de Wompi

```json
{
  "event": "transaction.updated",
  "data": {
    "transaction": {
      "id": "123456-1234-1234-1234-123456789012",
      "status": "APPROVED",
      "reference": "PAY-m2k4f7-a3b5c1",
      "amount_in_cents": 300000,
      "currency": "COP",
      "payment_method_type": "CARD",
      "created_at": "2024-01-15T10:30:00.000Z"
    }
  },
  "timestamp": 1705312200,
  "sent_at": "2024-01-15T10:30:00.000Z",
  "signature": {
    "checksum": "abc123...",
    "properties": [
      "data.transaction.id",
      "data.transaction.status",
      "data.transaction.amount_in_cents"
    ]
  }
}
```

## Posibles Status de Transacción

| Status | Significado | Acción |
|--------|-------------|--------|
| `APPROVED` | Pago exitoso | Acreditar al usuario |
| `DECLINED` | Rechazado por banco | Marcar como rechazado |
| `VOIDED` | Anulado | Marcar como anulado |
| `PENDING` | En proceso | No hacer nada (esperar) |
| `ERROR` | Error técnico | Loguear para revisión |

## Configuración en Panel de Wompi

1. Ir a **comercios.wompi.co** → Configuración → Eventos
2. Agregar URL: `https://{TU_CONVEX_SITE_URL}/webhooks/wompi`
3. Seleccionar evento: `transaction.updated`
4. Guardar

Para obtener la URL de tu sitio Convex:
```bash
npx convex env get CONVEX_SITE_URL
```

O desde el dashboard de Convex → Settings → URL.

## Notas de Seguridad

1. **Siempre retornar 200**: Incluso si la verificación falla o hay errores internos. Si retornas 4xx/5xx, Wompi reintentará el webhook.

2. **Verificación de firma**: Opcional pero muy recomendado para producción. Usa `WOMPI_EVENTS_SECRET` (diferente del `WOMPI_INTEGRITY_SECRET`).

3. **Idempotencia**: El handler DEBE manejar duplicados sin efectos secundarios. Wompi puede enviar el mismo evento múltiples veces.

4. **No confiar solo en el widget**: El widget frontend puede ser manipulado. La confirmación oficial del pago SIEMPRE viene por webhook.
