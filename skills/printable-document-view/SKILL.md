# Skill: Printable Document View (PDF-ready)

Creates a document/invoice view component optimized for printing to PDF via `window.print()`. Uses React + Tailwind CSS with a simulated A4 page layout on screen that maps perfectly to a printed page.

## Overview

This skill generates:
1. A **document page component** (full-page standalone view)
2. A **document content component** (the A4 printable area)
3. **Print CSS configuration** (inline `@media print` styles)
4. Optional: a **modal variant** for showing the document as an overlay

## Architecture

```
src/
├── pages/
│   └── DocumentView.tsx        # Page wrapper (navigation, print button)
├── components/
│   └── PrintableDocument.tsx   # A4 content layout with print styles
```

## Dependencies

- **React** (>=18)
- **Tailwind CSS** (>=3)
- **lucide-react** (optional, for icons)

No PDF library needed — uses native `window.print()` which produces clean PDFs when configured properly.

## Implementation Guide

### 1. Document Content Component (`PrintableDocument.tsx`)

The core component renders content in an A4-like layout with embedded print styles.

```tsx
import React from 'react';

interface PrintableDocumentProps {
  title?: string;
  isStandalone?: boolean;  // true = full page, false = modal overlay
  onClose?: () => void;
  children?: React.ReactNode;
}

const PrintableDocument: React.FC<PrintableDocumentProps> = ({
  title,
  isStandalone = false,
  onClose,
  children,
}) => {
  const handlePrint = () => {
    window.print();
  };

  // Standalone: full-page white background
  // Modal: centered overlay with backdrop
  const containerClasses = isStandalone
    ? "min-h-screen bg-white p-8 sm:p-12 print:p-0 print:bg-white"
    : "fixed inset-0 z-50 flex items-center justify-center bg-black/50 p-4 print:p-0 print:bg-white";

  const paperClasses = isStandalone
    ? "relative w-full max-w-[800px] mx-auto bg-white print:max-w-none print:p-0 print:h-[297mm] print:shadow-none"
    : "relative w-full max-w-[800px] max-h-[90vh] overflow-y-auto bg-white p-12 shadow-2xl print:shadow-none print:max-h-none print:p-0 print:h-[297mm]";

  return (
    <div className={containerClasses + " print-container"}>
      <div className={paperClasses + " print-paper"}>
        {/* Action buttons - hidden when printing */}
        <div className="absolute right-0 -top-12 flex gap-2 print:hidden sm:right-0 sm:top-0">
          <button
            onClick={handlePrint}
            className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors"
          >
            Imprimir / Print PDF
          </button>
          {onClose && (
            <button
              onClick={onClose}
              className="px-4 py-2 bg-gray-200 text-gray-700 rounded-md hover:bg-gray-300 transition-colors"
            >
              Cerrar
            </button>
          )}
        </div>

        {/* Document content area - A4 proportions */}
        <div className="text-black font-sans leading-relaxed h-full flex flex-col print:h-[257mm]">
          {children}
        </div>
      </div>

      {/* Critical: inline print styles for PDF generation */}
      <style dangerouslySetInnerHTML={{ __html: `
        @media print {
          @page {
            size: A4;
            margin: 0;
          }
          body {
            background: white !important;
            margin: 0 !important;
            padding: 0 !important;
          }
          body * {
            visibility: hidden;
          }
          .print-container, .print-container * {
            visibility: visible;
          }
          .print-container {
            position: absolute !important;
            left: 0 !important;
            top: 0 !important;
            width: 100% !important;
            height: 100% !important;
            margin: 0 !important;
            padding: 0 !important;
            background: white !important;
          }
          .print-paper {
            position: absolute !important;
            left: 0 !important;
            top: 0 !important;
            width: 100% !important;
            height: 297mm !important;
            padding: 2cm !important;
            box-sizing: border-box !important;
            background: white !important;
            display: block !important;
          }
          .print\\:hidden {
            display: none !important;
          }
        }
      `}} />
    </div>
  );
};

export default PrintableDocument;
```

### 2. Page Wrapper (`DocumentView.tsx`)

Wraps the document content with navigation and external print button.

```tsx
import { useEffect } from 'react';
import PrintableDocument from '@/components/PrintableDocument';

interface DocumentViewProps {
  documentTitle?: string;
}

const DocumentView: React.FC<DocumentViewProps> = ({ documentTitle = 'Document' }) => {
  // Set document title for PDF filename (browsers use document.title as default filename)
  useEffect(() => {
    const originalTitle = document.title;
    document.title = documentTitle;
    return () => { document.title = originalTitle; };
  }, [documentTitle]);

  return (
    <div className="relative min-h-screen">
      {/* Fixed top-left action bar - hidden in print */}
      <div className="fixed top-4 left-4 z-[60] flex gap-2 print:hidden">
        <button
          onClick={() => window.history.back()}
          className="px-3 py-2 bg-white/80 backdrop-blur-sm border rounded-md text-sm hover:bg-white transition"
        >
          ← Volver
        </button>
        <button
          onClick={() => window.print()}
          className="px-3 py-2 bg-blue-600 text-white rounded-md text-sm hover:bg-blue-700 transition"
        >
          Imprimir PDF
        </button>
      </div>

      <PrintableDocument isStandalone={true}>
        {/* Your document content here */}
        <div className="text-left">
          <p>Ciudad, fecha</p>
        </div>
        <div className="flex-grow" />
        <div className="space-y-8 text-center">
          <h1 className="font-bold text-xl">TÍTULO DEL DOCUMENTO</h1>
          <p>Contenido del documento...</p>
        </div>
        <div className="flex-grow" />
        <div className="text-left border-t pt-2">
          <p className="font-bold">Firma</p>
          <p>Nombre completo</p>
        </div>
      </PrintableDocument>
    </div>
  );
};

export default DocumentView;
```

### 3. Multi-page Document Support

For documents that span multiple pages, wrap each page in its own paper container:

```tsx
const MultiPageDocument: React.FC<{ pages: React.ReactNode[] }> = ({ pages }) => {
  return (
    <div className="min-h-screen bg-white p-8 print:p-0 print-container">
      {pages.map((pageContent, index) => (
        <div
          key={index}
          className="relative w-full max-w-[800px] mx-auto bg-white mb-8 print:max-w-none print:mb-0 print:h-[297mm] print:shadow-none print-paper"
          style={{ breakAfter: index < pages.length - 1 ? 'page' : 'auto' }}
        >
          <div className="text-black font-sans leading-relaxed h-full flex flex-col p-[2cm] print:p-[2cm]">
            {pageContent}
          </div>
        </div>
      ))}

      <style dangerouslySetInnerHTML={{ __html: `
        @media print {
          @page { size: A4; margin: 0; }
          body { background: white !important; margin: 0 !important; padding: 0 !important; }
          body * { visibility: hidden; }
          .print-container, .print-container * { visibility: visible; }
          .print-container {
            position: absolute !important; left: 0 !important; top: 0 !important;
            width: 100% !important; margin: 0 !important; padding: 0 !important; background: white !important;
          }
          .print-paper {
            width: 100% !important; height: 297mm !important;
            box-sizing: border-box !important; background: white !important; display: block !important;
          }
          .print\\:hidden { display: none !important; }
        }
      `}} />
    </div>
  );
};
```

## Key Design Patterns

### PDF filename from `document.title`
When the user prints via the browser, the suggested PDF filename comes from `document.title`. Set it dynamically:
```tsx
useEffect(() => {
  const original = document.title;
  document.title = `Factura-${clientName}-${month}-${year}`;
  return () => { document.title = original; };
}, [clientName, month, year]);
```

### Vertical content distribution with flexbox
Use `flex-grow` spacers to vertically distribute content across the full A4 height:
```tsx
<div className="h-full flex flex-col print:h-[257mm]">
  <div>Header / date</div>
  <div className="flex-grow" />  {/* pushes content to center */}
  <div>Main content</div>
  <div className="flex-grow" />  {/* pushes signature to bottom */}
  <div>Signature / footer</div>
</div>
```

### Hide elements during print
Use the Tailwind `print:hidden` utility class on any element that should not appear in the PDF:
```tsx
<button className="print:hidden">Print</button>
```

### Print isolation technique
The CSS uses `visibility: hidden` on all body elements, then `visibility: visible` only on `.print-container` and its children. This ensures ONLY the document content prints, regardless of surrounding UI (sidebars, navbars, footers).

### A4 dimensions reference
- A4 = 210mm × 297mm
- With 2cm margins on all sides: content area = 170mm × 257mm
- `max-w-[800px]` on screen simulates A4 proportions nicely
- `print:h-[297mm]` ensures exact page height in PDF

### Signature image support
To overlay a signature image above the signer's name:
```tsx
<div className="relative w-64 h-24 -mb-8 pointer-events-none">
  <img src={signatureDataUrl} alt="Firma" className="w-full h-full object-contain object-left" />
</div>
<div>
  <p className="font-bold uppercase">Nombre Completo</p>
  <p>C.C. No. 12345678</p>
</div>
```

### Number to words utility (Spanish)
Useful for invoices that require amounts written in letters:
```tsx
function numeroALetras(num: number): string {
  // Converts number to Spanish words (e.g., 1500000 -> "un millón quinientos mil")
  // Full implementation in the skill reference
}
```

### Rich text with bold markers
Allow users to format text with `*bold*` markers:
```tsx
function formatRichText(text: string): (string | React.ReactNode)[] {
  const parts = text.split(/(\*[^*]+\*)/g);
  return parts.map((part, index) => {
    if (part.startsWith('*') && part.endsWith('*')) {
      return <strong key={index}>{part.slice(1, -1)}</strong>;
    }
    return part;
  });
}
```

## Checklist for implementation

- [ ] Create `PrintableDocument.tsx` with inline print CSS
- [ ] Create page/route that uses `PrintableDocument`
- [ ] Set `document.title` dynamically for PDF filename
- [ ] Add `print:hidden` to all non-document UI (nav, buttons, footer)
- [ ] Use `flex-grow` spacers for vertical content distribution
- [ ] Test print preview in browser (Ctrl+P / Cmd+P) to verify layout
- [ ] Verify content fits in A4 (no overflow, no extra pages)
- [ ] Optional: add signature image support
- [ ] Optional: add number-to-words utility for amounts

## Usage prompt example

> "Crea una vista de documento imprimible usando la skill `printable-document-view`. El documento es una [factura/contrato/recibo] que incluye: [campos]. Debe tener botón para imprimir PDF y el nombre del archivo debe ser [patrón]."
