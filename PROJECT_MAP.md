# PROJECT_MAP — محرر PDF (PDF Annotation Tool)

## TECH_STACK

| Layer | Technology | Version | Source |
|-------|-----------|---------|--------|
| PDF Rendering | pdf.js (pdfjs-dist) | 5.7.284 | CDN (jsDelivr) |
| PDF Modification | pdf-lib | 1.17.1 | CDN (unpkg) |
| UI Framework | Tailwind CSS v4 Play CDN | 4.1.18 | CDN (jsDelivr) |
| Arabic Font (UI) | Cairo (Google Fonts) | latest | Google Fonts CSS |
| Arabic Font (PDF) | Cairo (TTF embed) | raw from GH | google/fonts repo |
| Hosting | Vercel (static) | — | vercel.json |

## SYSTEM_FLOW

```
[User] → رفع PDF (file input)
         ↓
    FileReader → ArrayBuffer
         ↓
    ┌──────────────────────────────────┐
    │ pdf.js: getDocument → render     │ ← لكل صفحة: canvas + overlays-layer
    │ pdf-lib: PDFDocument.load        │ ← للتصدير النهائي
    └──────────────────────────────────┘
         ↓
[User] → اختيار أداة (نص/صح/خطأ) ← شريط أدوات sticky
         ↓
    نقرة على overlays-layer
         ↓
    ┌──────────────────────────────────┐
    │ إنشاء HTML overlay (input/span)  │
    │ تخزين annotation: {pageNum,type, │
    │   relX, relY, text}              │
    └──────────────────────────────────┘
         ↓
[User] → تنزيل PDF
         ↓
    ┌──────────────────────────────────┐
    │ pdf-lib: embedFont(Cairo)       │
    │ لكل annotation: page.drawText() │
    │   relX→x:  x = relX * pageWidth │
    │   relY→y:  y = h - relY * h     │
    │ pdfDoc.save() → Blob → download │
    └──────────────────────────────────┘
```

## ARCHITECTURE

```
index.html (ملف واحد — لا build step)
├── <head>
│   ├── Tailwind v4 Play CDN (script)
│   ├── Cairo Google Font (link)
│   └── pdf-lib UMD (script)
├── <body dir="rtl">
│   ├── <header>       ← شعار + زر رفع PDF
│   ├── <nav> (sticky) ← أدوات: نص / صح ✔ / خطأ ✘ + زر تنزيل
│   └── <main>#pdfViewer
│       └── .page-container (لكل صفحة)
│           ├── <canvas>         ← pdf.js rendering
│           └── .overlays-layer  ← HTML overlays (pointer-events)
├── type="importmap"    ← pdfjs-dist → CDN URL
└── type="module" script
    ├── State: { pdfDoc, pdfLibDoc, annotations, activeTool, ... }
    ├── loadCairoFontBytes()    ← Google Fonts GitHub raw
    ├── renderAllPages()        ← pdf.js لكل صفحة
    ├── renderAnnotation()      ← DOM overlay لكل علامة
    ├── setupTools()            ← toggle أدوات
    └── handleDownload()        ← pdf-lib → Blob
```

### Data Structure
```js
// Core annotation unit
{
  id: string,            // unique (Date.now + random)
  pageNum: number,       // 1-indexed
  type: 'text' | 'true' | 'false',
  relX: number,           // 0..1 (نسبة من عرض الصفحة)
  relY: number,           // 0..1 (نسبة من ارتفاع الصفحة، من الأعلى)
  text: string,           // للنص فقط
  fontSize: number        // حجم الخط الخاص بهذه العلامة (افتراضي 16)
}
```

### Mark Scaling
علامات ✔/✘ تستخدم مضاعف `×1.75` لتحويل `fontSize` الأساسي إلى حجم العرض:
- `span.style.fontSize = Math.round(ann.fontSize * 1.75)px`
- التغيير عبر (±) يعدّل `ann.fontSize` في الـ data model ويطبق المضاعف عند عرض DOM وعند تصدير PDF

### Coordinate Mapping
```
Screen (overlays-layer) → PDF (pdf-lib)
  x_pdf = relX * pageWidth
  y_pdf = pageHeight - relY * pageHeight  // تحويل top-down → bottom-up
```

## ORPHANS & PENDING

- [x] M1: رفع PDF وعرض أول صفحة على Canvas
- [x] M2: شريط الأدوات مع toggle + وضع overlays
- [x] M3: تصدير PDF مع تضمين الخط والعلامات
- [x] M4: واجهة عربية كاملة (RTL, Cairo, أزرار)
- [x] M5: تحجيم مستقل لكل annotion (popup عند النقر)
- [x] إصلاح مسار pdf.js (v5 عنده فقط `.mjs`, لا `legacy/build`)
- [x] إصلاح تحجيم علامات ✔/✘: كان يضبط `.annotation-content` بدلاً من `.annotation-mark` فأيقونة العلامة لا تتغير
- [x] إصلاح race condition في download (`URL.revokeObjectURL` ← `setTimeout` لـ Safari)
- [x] تحديث `vercel.json` إلى صيغة حديثة (إزالة `@vercel/static` المهجور، إضافة `X-Content-Type-Options` + `Referrer-Policy`)
- [x] إصلاح روابط خط Cairo: تغير اسم الملف من `Cairo%5Bwght%5D.ttf` إلى `Cairo%5Bslnt%2Cwght%5D.ttf` (أضيف محور `slnt`)
- [x] إصلاح تصدير PDF: استبدال `drawText('✔'/'✘')` بـ `drawSvgPath` لرسم العلامات كخطوط SVG (لا تعتمد على خط)، وإزالة fallback Helvetica للنصوص العربية
- [x] إضافة `@pdf-lib/fontkit` CDN وتسجيله بعد `PDFDocument.load` لتضمين الخطوط المخصصة
- [x] إصلاح تحجيم التصدير: ضرب أحجام العناصر بـ `scaleX = pdfWidth / canvasRect.width` (≈ 1/1.5) لمطابقة الحجم البصري بين HTML و PDF
- [ ] اختبار على متصفحات متعددة (Chrome, Firefox, Safari)
- [ ] التحقق من رابط خط Cairo TTF (قد يتغير اسم الملف في مستودع Google Fonts)
- [ ] إضافة مؤشر تحميل أثناء معالجة PDF
