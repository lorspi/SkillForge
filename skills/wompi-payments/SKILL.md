---
name: wompi-payments
description: Skill para integrar Wompi (pasarela de pagos colombiana) en proyectos con Convex backend. Cubre configuración, widget de checkout, firma de integridad, webhook y procesamiento de pagos.
version: 1.0.0
---

# Wompi Payments — Integración con Convex

## Overview

[Wompi](https://wompi.com) es una pasarela de pagos colombiana (propiedad de Bancolombia). Soporta tarjetas de crédito/débito, PSE, Nequi, y otros métodos de pago locales. Es el equivalente colombiano de Stripe.

Esta skill guía la integración completa de Wompi usando:
- **Backend**: Convex (actions, mutations, HTTP endpoints)
- **Frontend**: React con el widget WidgetCheckout de Wompi
- **Moneda**: COP (pesos colombianos) en centavos

## Arquitectura del Flujo de Pago

```
┌─────────┐         ┌──────────┐         ┌─────────┐
│ Cliente │         │  Convex  │         │  Wompi  │
└────┬────┘         └────┬─────┘         └────┬────┘
     │                    │                    │
     │ 1. Solicitar       │                    │
     │    referencia      │                    │
     │───────────────────>│                    │
     │                    │                    │
     │ 2. Referencia +    │                    │
     │    firma + config  │                    │
     │<───────────────────│                    │
     │                    │                    │
     │ 3. Abrir widget    │                    │
     │    Wompi           │                    │
     │─────────────────────────────────────────>
     │                    │                    │
     │ 4. Usuario paga    │                    │
     │    en widget       │                    │
     │                    │                    │
     │                    │ 5. Webhook POST    │
     │                    │    /webhooks/wompi  │
     │                    │<───────────────────│
     │                    │                    │
     │                    │ 6. Procesar pago   │
     │                    │    + acreditar     │
     │                    │                    │
     │ 7. Convex reactivo │                    │
     │    actualiza UI    │                    │
     │<───────────────────│                    │
```

## Requisitos Previos

1. Cuenta de Wompi (sandbox o producción): https://comercios.wompi.co
2. Proyecto Convex configurado
3. React frontend con Vite u otro bundler

## Variables de Entorno (Convex Dashboard)

```
WOMPI_PUBLIC_KEY=pub_stagtest_xxx        # Clave pública para el widget
WOMPI_INTEGRITY_SECRET=stagtest_xxx      # Secreto para firma SHA-256
WOMPI_EVENTS_SECRET=stagtest_xxx         # Secreto para verificar webhooks (opcional)
```

> Las claves de sandbox empiezan con `pub_stagtest_` y `stagtest_`. En producción usan `pub_prod_` y `prod_`.

## Implementación Paso a Paso

### 1. Schema — Tabla de pagos

Agregar al `convex/schema.ts`:

```typescript
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

// Agregar esta tabla al schema existente
payments: defineTable({
  userId: v.optional(v.id("users")),         // Usuario autenticado (opcional)
  reference: v.string(),                      // Referencia única para Wompi
  amountCents: v.number(),                    // Monto en centavos COP
  credits: v.number(),                        // Cantidad de items/créditos comprados
  status: v.union(
    v.literal("pending"),
    v.literal("approved"),
    v.literal("declined"),
    v.literal("voided"),
    v.literal("error")
  ),
  wompiTransactionId: v.optional(v.string()), // ID de transacción de Wompi
  createdAt: v.number(),
  updatedAt: v.number(),
})
  .index("by_reference", ["reference"])
  .index("by_user", ["userId"]),
```

### 2. Backend — Action para crear referencia de pago

Ver [referencia de payments backend](references/payments-backend.md) para la implementación completa.

Puntos clave:
- La referencia debe ser **única** por cada intento de pago
- La firma de integridad evita manipulación del monto en el cliente
- El formato de firma es: `SHA256(reference + amountInCents + "COP" + integritySecret)`

### 3. Backend — Webhook HTTP para procesar pagos

Ver [referencia de webhook](references/webhook-handler.md) para la implementación completa.

Puntos clave:
- Endpoint: `POST /webhooks/wompi`
- El procesamiento debe ser **idempotente** (mismo webhook 2 veces = no-op)
- Siempre responder HTTP 200, incluso si hay errores internos
- Verificar la firma del evento cuando `WOMPI_EVENTS_SECRET` esté configurado

### 4. Frontend — Helper del widget

Ver [referencia del widget](references/widget-helper.md) para la implementación completa.

Puntos clave:
- Cargar script externo: `<script src="https://checkout.wompi.co/widget.js"></script>`
- El callback va en `.open()`, NO en el constructor
- El widget retorna `{ transaction: { id, status, reference } }`

### 5. Frontend — Componente de compra

Ver [referencia del componente](references/purchase-component.md) para un ejemplo completo.

## Firma de Integridad

La firma protege contra manipulación del monto en el cliente:

```typescript
async function sha256Hex(message: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(message);
  const hashBuffer = await crypto.subtle.digest("SHA-256", data);
  return Array.from(new Uint8Array(hashBuffer))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

// Generar firma
const signatureInput = `${reference}${amountInCents}COP${integritySecret}`;
const integritySignature = await sha256Hex(signatureInput);
```

## Verificación de Webhook (Seguridad)

Wompi firma los webhooks con un checksum. Para verificar:

```typescript
async function verifyWompiSignature(event: any, eventsSecret: string): Promise<boolean> {
  if (!event?.signature?.checksum || !event?.signature?.properties?.length) {
    return false;
  }

  // Concatenar valores de las propiedades listadas en signature.properties
  const properties = event.signature.properties;
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
```

## Referencia de Pago

Formato recomendado: `{PREFIX}-{timestamp_base36}-{random6chars}`

```typescript
function generatePaymentReference(prefix = "PAY"): string {
  const timestamp = Date.now().toString(36);
  const randomChars = Array.from(crypto.getRandomValues(new Uint8Array(6)))
    .map((b) => b.toString(36).padStart(2, "0").slice(0, 1))
    .join("");
  return `${prefix}-${timestamp}-${randomChars}`;
}
```

## Estados del Pago

```
pending  → approved   (pago exitoso, ejecutar lógica post-pago)
pending  → declined   (pago rechazado por banco/Wompi)
pending  → voided     (pago anulado)
pending  → error      (error de procesamiento)
```

## Configuración del Webhook en Wompi

1. Ir a https://comercios.wompi.co → Configuración → Webhooks
2. URL del endpoint: `https://{CONVEX_SITE_URL}/webhooks/wompi`
3. Evento: `transaction.updated`
4. El `CONVEX_SITE_URL` se obtiene del dashboard de Convex o con `npx convex env get CONVEX_SITE_URL`

## Sandbox vs Producción

| Aspecto | Sandbox | Producción |
|---------|---------|------------|
| Public Key | `pub_stagtest_...` | `pub_prod_...` |
| Integrity Secret | `stagtest_integrity_...` | `prod_integrity_...` |
| Events Secret | `stagtest_events_...` | `prod_events_...` |
| Widget URL | `https://checkout.wompi.co/widget.js` | `https://checkout.wompi.co/widget.js` (mismo) |
| Tarjeta de prueba | 4242 4242 4242 4242 | Tarjeta real |

## Tarjetas de Prueba (Sandbox)

- **Aprobada**: 4242 4242 4242 4242 (cualquier fecha futura, cualquier CVC)
- **Rechazada**: 4111 1111 1111 1111
- **Nequi sandbox**: usar número 3991111111

## Checklist de Integración

- [ ] Variables de entorno configuradas en Convex Dashboard
- [ ] Tabla `payments` en el schema
- [ ] Action `createPaymentReference` implementada
- [ ] Webhook handler en `convex/http.ts`
- [ ] Script del widget cargado en `index.html`
- [ ] Helper `openWompiWidget()` creado
- [ ] Componente de compra implementado
- [ ] Webhook configurado en panel de Wompi
- [ ] Probado con tarjeta sandbox 4242
- [ ] Verificación de firma de webhook habilitada

## Errores Comunes

1. **"Wompi widget script not loaded"** → Verificar que `<script src="https://checkout.wompi.co/widget.js">` está en `index.html`
2. **Firma inválida** → Verificar que el orden de concatenación es exacto: `reference + amountInCents + "COP" + secret`
3. **Webhook no llega** → Verificar URL en panel de Wompi y que el endpoint responde 200
4. **Monto incorrecto** → Recordar que Wompi trabaja en **centavos** (COP × 100)

## Referencias

- [Backend: Payments Action y Mutations](references/payments-backend.md)
- [Webhook Handler](references/webhook-handler.md)
- [Widget Helper (Frontend)](references/widget-helper.md)
- [Componente de Compra (React)](references/purchase-component.md)
- [Documentación oficial Wompi](https://docs.wompi.co)
