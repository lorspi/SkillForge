# Payments Backend — Convex Action y Mutations

## Archivo: `convex/payments.ts`

Este archivo contiene toda la lógica de backend para pagos con Wompi.

## Utilidades

```typescript
/**
 * Compute SHA-256 hash de un string, retornado como hex minúscula.
 * Usado para la firma de integridad de Wompi.
 */
export async function sha256Hex(message: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(message);
  const hashBuffer = await crypto.subtle.digest("SHA-256", data);
  return Array.from(new Uint8Array(hashBuffer))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

/**
 * Genera una referencia de pago única.
 * Formato: "{PREFIX}-{timestamp}-{random6chars}"
 */
export function generatePaymentReference(prefix = "PAY"): string {
  const timestamp = Date.now().toString(36);
  const randomChars = Array.from(crypto.getRandomValues(new Uint8Array(6)))
    .map((b) => b.toString(36).padStart(2, "0").slice(0, 1))
    .join("");
  return `${prefix}-${timestamp}-${randomChars}`;
}
```

## Internal Mutation: Insertar pago pendiente

```typescript
import { internalMutation } from "./_generated/server";
import { v } from "convex/values";

/**
 * Crea un registro de pago pendiente en la base de datos.
 */
export const insertPendingPayment = internalMutation({
  args: {
    userId: v.optional(v.id("users")),
    reference: v.string(),
    amountCents: v.number(),
    credits: v.number(),
  },
  handler: async (ctx, args) => {
    const now = Date.now();
    await ctx.db.insert("payments", {
      userId: args.userId,
      reference: args.reference,
      amountCents: args.amountCents,
      credits: args.credits,
      status: "pending",
      createdAt: now,
      updatedAt: now,
    });
  },
});
```

## Action: Crear referencia de pago

```typescript
import { action } from "./_generated/server";
import { v } from "convex/values";
import { internal } from "./_generated/api";

/**
 * Crea una referencia de pago y genera la configuración del widget Wompi.
 *
 * Flujo:
 * 1. Valida la cantidad de items/créditos
 * 2. Calcula el monto total en centavos COP
 * 3. Genera una referencia única
 * 4. Computa la firma SHA-256 de integridad
 * 5. Crea registro pendiente en la BD
 * 6. Retorna la configuración para el widget
 */
export const createPaymentReference = action({
  args: {
    userId: v.optional(v.id("users")),
    credits: v.number(),
  },
  returns: v.object({
    reference: v.string(),
    amountInCents: v.number(),
    integritySignature: v.string(),
    publicKey: v.string(),
    currency: v.literal("COP"),
  }),
  handler: async (ctx, args) => {
    const { userId, credits } = args;

    // Validar cantidad
    if (!Number.isInteger(credits) || credits < 1 || credits > 10) {
      throw new Error("Credits must be an integer between 1 and 10");
    }

    // Calcular monto (adaptar según tu lógica de precios)
    const unitPriceCents = 300000; // $3,000 COP = 300,000 centavos
    const amountInCents = credits * unitPriceCents;

    // Generar referencia única
    const reference = generatePaymentReference();

    // Obtener variables de entorno
    const publicKey = process.env.WOMPI_PUBLIC_KEY;
    if (!publicKey) {
      throw new Error("Missing WOMPI_PUBLIC_KEY environment variable");
    }

    const integritySecret = process.env.WOMPI_INTEGRITY_SECRET;
    if (!integritySecret) {
      throw new Error("Missing WOMPI_INTEGRITY_SECRET environment variable");
    }

    // Generar firma de integridad
    // IMPORTANTE: El orden es reference + amountInCents + "COP" + secret
    const signatureInput = `${reference}${amountInCents}COP${integritySecret}`;
    const integritySignature = await sha256Hex(signatureInput);

    // Crear registro de pago pendiente
    await ctx.runMutation(internal.payments.insertPendingPayment, {
      userId: userId ?? undefined,
      reference,
      amountCents: amountInCents,
      credits,
    });

    return {
      reference,
      amountInCents,
      integritySignature,
      publicKey,
      currency: "COP" as const,
    };
  },
});
```

## Internal Query: Buscar pago por referencia

```typescript
import { internalQuery } from "./_generated/server";
import { v } from "convex/values";

export const getPaymentByReference = internalQuery({
  args: { reference: v.string() },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("payments")
      .withIndex("by_reference", (q) => q.eq("reference", args.reference))
      .unique();
  },
});
```

## Internal Mutation: Procesar pago aprobado

```typescript
/**
 * Procesa un pago aprobado: actualiza status y ejecuta lógica post-pago.
 * 
 * IMPORTANTE: Esta mutación es idempotente. Si el pago ya fue aprobado,
 * retorna sin hacer cambios.
 */
export const processApprovedPayment = internalMutation({
  args: {
    reference: v.string(),
    wompiTransactionId: v.string(),
  },
  handler: async (ctx, args) => {
    const payment = await ctx.db
      .query("payments")
      .withIndex("by_reference", (q) => q.eq("reference", args.reference))
      .unique();

    if (!payment) {
      return { success: false, reason: "payment_not_found" };
    }

    // Idempotencia: si ya está aprobado, skip
    if (payment.status === "approved") {
      return { success: true, reason: "already_approved" };
    }

    // Actualizar status del pago
    await ctx.db.patch(payment._id, {
      status: "approved",
      wompiTransactionId: args.wompiTransactionId,
      updatedAt: Date.now(),
    });

    // === LÓGICA POST-PAGO ===
    // Aquí va tu lógica específica: agregar créditos, activar plan, etc.
    if (payment.userId) {
      const user = await ctx.db.get(payment.userId);
      if (user) {
        await ctx.db.patch(payment.userId, {
          credits: (user.credits ?? 0) + payment.credits,
        });
      }
    }

    return { success: true, reason: "processed" };
  },
});
```

## Internal Mutation: Procesar pago rechazado/anulado

```typescript
export const processDeclinedOrVoidedPayment = internalMutation({
  args: {
    reference: v.string(),
    status: v.union(v.literal("declined"), v.literal("voided")),
    wompiTransactionId: v.string(),
  },
  handler: async (ctx, args) => {
    const payment = await ctx.db
      .query("payments")
      .withIndex("by_reference", (q) => q.eq("reference", args.reference))
      .unique();

    if (!payment) {
      return { success: false, reason: "payment_not_found" };
    }

    // No sobreescribir estados terminales
    if (["approved", "declined", "voided"].includes(payment.status)) {
      return { success: true, reason: "already_terminal" };
    }

    await ctx.db.patch(payment._id, {
      status: args.status,
      wompiTransactionId: args.wompiTransactionId,
      updatedAt: Date.now(),
    });

    return { success: true, reason: "processed" };
  },
});
```

## Notas Importantes

1. **Firma de integridad**: El orden de concatenación DEBE ser exacto: `reference + amountInCents + "COP" + secret`. Cualquier cambio invalida la firma.

2. **Idempotencia**: Los handlers de webhook DEBEN ser idempotentes. Wompi puede enviar el mismo evento múltiples veces.

3. **Precios en centavos**: Wompi trabaja en centavos. $3,000 COP = 300,000 centavos.

4. **crypto.subtle**: Disponible nativamente en el runtime de Convex (no necesita dependencias externas).

5. **process.env**: En Convex, las variables de entorno solo están disponibles en `actions`, no en `queries` ni `mutations`.
