# Widget Helper — Frontend

## Archivo: `src/lib/wompi.ts`

Helper para abrir el widget de checkout de Wompi desde el frontend.

## Paso 1: Cargar el script del widget

Agregar en `index.html` (dentro de `<head>` o antes del cierre de `</body>`):

```html
<script src="https://checkout.wompi.co/widget.js"></script>
```

## Paso 2: Crear el helper

```typescript
// src/lib/wompi.ts

export interface WompiWidgetConfig {
  publicKey: string;
  currency: "COP";
  amountInCents: number;
  reference: string;
  integritySignature: string;
  /** URL a la que Wompi redirige después del pago (opcional) */
  redirectUrl?: string;
}

export interface WompiTransactionResult {
  id: string;
  status: string;
  reference: string;
}

// Declaración global para TypeScript
declare global {
  interface Window {
    WidgetCheckout?: any;
  }
}

/**
 * Abre el widget de checkout de Wompi.
 *
 * IMPORTANTE: El callback se pasa a `.open()`, NO al constructor.
 *
 * @returns Promise con el resultado de la transacción
 */
export function openWompiWidget(config: WompiWidgetConfig): Promise<WompiTransactionResult> {
  return new Promise((resolve, reject) => {
    if (!window.WidgetCheckout) {
      reject(new Error("Wompi widget script not loaded"));
      return;
    }

    try {
      const checkout = new window.WidgetCheckout({
        publicKey: config.publicKey,
        currency: config.currency,
        amountInCents: config.amountInCents,
        reference: config.reference,
        "signature:integrity": config.integritySignature,
        redirectUrl: config.redirectUrl ?? window.location.href,
      });

      // El callback va en .open(), NO en el constructor
      checkout.open((result: { transaction: WompiTransactionResult }) => {
        if (result && result.transaction) {
          resolve(result.transaction);
        } else {
          reject(new Error("Widget closed without completing payment"));
        }
      });
    } catch (err) {
      reject(err);
    }
  });
}
```

## Propiedades del Constructor WidgetCheckout

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `publicKey` | string | Clave pública de Wompi |
| `currency` | string | Moneda (`"COP"`) |
| `amountInCents` | number | Monto en centavos |
| `reference` | string | Referencia única del pago |
| `signature:integrity` | string | Firma SHA-256 de integridad |
| `redirectUrl` | string | URL de redirección post-pago |

## Resultado del Widget

Cuando el usuario completa (o cancela) el pago, el callback recibe:

```typescript
{
  transaction: {
    id: "123456-abcd-...",        // ID de transacción Wompi
    status: "APPROVED",           // APPROVED | DECLINED | VOIDED | PENDING
    reference: "PAY-abc-123456"   // La referencia que enviaste
  }
}
```

## Manejo de Estados

```typescript
const result = await openWompiWidget(config);

switch (result.status) {
  case "APPROVED":
    // Pago exitoso — la confirmación real viene por webhook
    // Mostrar mensaje de éxito al usuario
    break;
  case "DECLINED":
    // Pago rechazado — mostrar error y opción de reintentar
    break;
  case "PENDING":
    // Pago en proceso (PSE, efectivo) — informar que se procesará
    break;
  case "VOIDED":
    // Pago anulado
    break;
}
```

## Errores Comunes

1. **"Wompi widget script not loaded"**
   - El script `widget.js` no está en el HTML
   - Se está llamando antes de que el script cargue

2. **Widget no abre / pantalla blanca**
   - `publicKey` inválida o de otro ambiente
   - `amountInCents` es 0 o negativo

3. **"Firma inválida" en el widget**
   - El `integritySignature` no coincide
   - Verificar orden: `reference + amountInCents + "COP" + secret`
   - Verificar que el secret sea el correcto (sandbox vs prod)

4. **Widget cierra sin callback**
   - El usuario cerró manualmente el widget
   - Se resuelve con el reject del Promise

## Nota sobre redirectUrl

Si se configura `redirectUrl`, Wompi redirigirá al usuario a esa URL después del pago con query params:
```
https://tuapp.com/dashboard?id=txn_123&env=test
```

Si NO quieres redirección y prefieres manejar todo en el callback, puedes pasar la URL actual:
```typescript
redirectUrl: window.location.href
```
