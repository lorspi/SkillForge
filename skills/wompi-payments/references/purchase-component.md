# Componente de Compra — React + Convex

## Ejemplo: Componente de compra de créditos

Este componente permite al usuario seleccionar cantidad, ver el total, y pagar via Wompi.

```tsx
// src/components/PurchaseCredits.tsx

import { useState } from "react";
import { useAction } from "convex/react";
import { api } from "../../convex/_generated/api";
import type { Id } from "../../convex/_generated/dataModel";
import { openWompiWidget } from "@/lib/wompi";

type PurchaseState = "idle" | "loading" | "processing" | "success" | "error";

interface PurchaseCreditsProps {
  userId: Id<"users">;
  /** Precio unitario en COP (no centavos) */
  unitPrice: number;
  /** Máximo de items por compra */
  maxQuantity?: number;
}

export function PurchaseCredits({
  userId,
  unitPrice,
  maxQuantity = 10,
}: PurchaseCreditsProps) {
  const [quantity, setQuantity] = useState(1);
  const [state, setState] = useState<PurchaseState>("idle");
  const [errorMessage, setErrorMessage] = useState("");

  const createPaymentReference = useAction(api.payments.createPaymentReference);

  const totalCOP = quantity * unitPrice;

  const handlePurchase = async () => {
    setState("loading");
    setErrorMessage("");

    try {
      // 1. Crear referencia de pago en el backend
      const config = await createPaymentReference({
        userId,
        credits: quantity,
      });

      // 2. Abrir widget de Wompi
      setState("processing");
      const result = await openWompiWidget({
        publicKey: config.publicKey,
        currency: config.currency,
        amountInCents: config.amountInCents,
        reference: config.reference,
        integritySignature: config.integritySignature,
      });

      // 3. Manejar resultado
      if (result.status === "APPROVED") {
        setState("success");
      } else {
        setState("error");
        setErrorMessage("El pago no fue aprobado. Intenta de nuevo.");
      }
    } catch (err) {
      setState("error");
      if (err instanceof Error && err.message.includes("not loaded")) {
        setErrorMessage("El sistema de pago no cargó. Recarga la página.");
      } else {
        setErrorMessage("Error al procesar el pago. Intenta de nuevo.");
      }
      console.error("Payment error:", err);
    }
  };

  // Formatear precio en COP
  const formatCOP = (amount: number) =>
    new Intl.NumberFormat("es-CO", {
      style: "currency",
      currency: "COP",
      minimumFractionDigits: 0,
    }).format(amount);

  return (
    <div className="space-y-4">
      {state === "success" && (
        <div className="p-4 bg-green-50 border border-green-200 rounded-lg text-center">
          <p className="text-green-700 font-medium">
            ¡Pago exitoso! Tu compra se procesará en unos segundos.
          </p>
          <button
            onClick={() => { setState("idle"); setQuantity(1); }}
            className="mt-2 text-sm underline"
          >
            Comprar más
          </button>
        </div>
      )}

      {state === "error" && (
        <div className="p-4 bg-red-50 border border-red-200 rounded-lg text-center">
          <p className="text-red-700">{errorMessage}</p>
          <button
            onClick={() => setState("idle")}
            className="mt-2 px-4 py-2 bg-blue-600 text-white rounded-lg"
          >
            Reintentar
          </button>
        </div>
      )}

      {(state === "idle" || state === "loading" || state === "processing") && (
        <>
          {/* Selector de cantidad */}
          <div className="flex items-center justify-center gap-4">
            <button
              onClick={() => setQuantity((q) => Math.max(1, q - 1))}
              disabled={quantity <= 1 || state !== "idle"}
              className="w-10 h-10 border rounded-lg disabled:opacity-40"
            >
              −
            </button>
            <span className="text-2xl font-bold">{quantity}</span>
            <button
              onClick={() => setQuantity((q) => Math.min(maxQuantity, q + 1))}
              disabled={quantity >= maxQuantity || state !== "idle"}
              className="w-10 h-10 border rounded-lg disabled:opacity-40"
            >
              +
            </button>
          </div>

          {/* Total */}
          <div className="text-center p-3 bg-gray-50 rounded-lg">
            <p className="text-sm text-gray-500">Total a pagar</p>
            <p className="text-xl font-bold">{formatCOP(totalCOP)}</p>
          </div>

          {/* Botón de pago */}
          <button
            onClick={handlePurchase}
            disabled={state !== "idle"}
            className="w-full py-3 bg-blue-600 hover:bg-blue-700 disabled:opacity-60 text-white font-semibold rounded-lg"
          >
            {state === "loading" && "Preparando pago..."}
            {state === "processing" && "Procesando..."}
            {state === "idle" && `Pagar ${formatCOP(totalCOP)}`}
          </button>
        </>
      )}
    </div>
  );
}
```

## Flujo del Componente

1. **idle** → Usuario selecciona cantidad y ve el total
2. **loading** → Se llama a `createPaymentReference` en Convex
3. **processing** → Widget de Wompi está abierto, usuario pagando
4. **success** → Pago aprobado (la acreditación viene por webhook)
5. **error** → Algo falló, mostrar mensaje y opción de reintentar

## Variante: Pago de monto fijo (sin selector)

Para pagos de un solo item a precio fijo:

```tsx
export function SinglePayment({ userId, price }: { userId: Id<"users">; price: number }) {
  const [state, setState] = useState<PurchaseState>("idle");
  const createPaymentReference = useAction(api.payments.createPaymentReference);

  const handlePay = async () => {
    setState("loading");
    try {
      const config = await createPaymentReference({ userId, credits: 1 });
      setState("processing");
      const result = await openWompiWidget({ ...config });
      setState(result.status === "APPROVED" ? "success" : "error");
    } catch {
      setState("error");
    }
  };

  return (
    <button onClick={handlePay} disabled={state !== "idle"}>
      Pagar {formatCOP(price)}
    </button>
  );
}
```

## Consideraciones UX

1. **No bloquear la UI esperando webhook**: El widget devuelve el status inmediatamente, pero la acreditación real se confirma via webhook. Usa la reactividad de Convex (`useQuery`) para actualizar la UI cuando el saldo cambie.

2. **Feedback visual**: Siempre mostrar un spinner o indicador cuando el estado es `loading` o `processing`.

3. **Manejo de cierre**: Si el usuario cierra el widget sin pagar, el Promise se rechaza. Capturar ese caso y volver a `idle`.

4. **Accesibilidad**: Incluir `aria-label` en botones de incremento/decremento y estados de carga.

## Integración con Convex Reactivity

Para que la UI se actualice automáticamente cuando el webhook procese el pago:

```tsx
import { useQuery } from "convex/react";
import { api } from "../../convex/_generated/api";

function Dashboard({ userId }: { userId: Id<"users"> }) {
  // Este query se actualiza automáticamente cuando el webhook acredita
  const user = useQuery(api.users.get, { userId });

  return (
    <div>
      <p>Créditos disponibles: {user?.credits ?? 0}</p>
      <PurchaseCredits userId={userId} unitPrice={3000} />
    </div>
  );
}
```
