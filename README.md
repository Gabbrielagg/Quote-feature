<!-- jsPDF y autoTable para generar la cotización en PDF -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js" defer></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.28/jspdf.plugin.autotable.min.js" defer></script>


<script>
function tryInsertQuoteButton() {
  maquimportInsertQuoteButtonInSlideCart();
}

  async function loadImageAsBase64(url) {

  const res = await fetch(url);
  const blob = await res.blob();

  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onloadend = () => resolve(reader.result); 
    reader.readAsDataURL(blob);
  });

}

async function maquimportGenerateQuotePdf(quotationNumber) {
  try {
    const clientData = getMaquimportUserData();
    const res = await fetch('/cart.js');
    if (!res.ok) throw new Error('No se pudo obtener el carrito');
    const cart = await res.json();

    if (!cart.items || cart.items.length === 0) {
      alert('Tu carrito está vacío.');
      return;
    }

    if (!window.jspdf || !window.jspdf.jsPDF) {
      alert('No se pudo cargar el generador de PDF. Intenta nuevamente.');
      return;
    }

    const { jsPDF } = window.jspdf;
    const doc = new jsPDF();
    const pageWidth = doc.internal.pageSize.getWidth();

    // ===== SELLO NÚMERO DE COTIZACIÓN =====
if (quotationNumber) {
  const boxWidth = 60;
  const boxHeight = 22;
  const boxX = pageWidth - boxWidth - 14;
  const boxY = 20;

  // borde azul marca
  doc.setDrawColor(21, 92, 118);
  doc.setLineWidth(0.8);
  doc.rect(boxX, boxY, boxWidth, boxHeight);

  // texto
  doc.setTextColor(21, 92, 118);
  doc.setFont('helvetica', 'bold');

  doc.setFontSize(10);
  doc.text('Cotización', boxX + boxWidth / 2, boxY + 8, { align: 'center' });

  doc.setFontSize(11);
  doc.text(`N° ${quotationNumber}`, boxX + boxWidth / 2, boxY + 15, { align: 'center' });

  // reset color
  doc.setTextColor(0, 0, 0);
}


    // ===== LOGO CIRCULAR SUPERIOR =====
if (window.MAQUIMPORT_PDF && window.MAQUIMPORT_PDF.logo) {
  try {
    const logoBase64 = await loadImageAsBase64(window.MAQUIMPORT_PDF.logo);

    const logoSize = 44;
    const logoX = 8;
    const logoY = 8;

    doc.addImage(
      logoBase64,
      'PNG',
      logoX,
      logoY,
      logoSize,
      logoSize
    );
  } catch (e) {
    console.warn('No se pudo agregar el logo al PDF', e);
  }
}


    // ===== SISTEMA DE DOS COLUMNAS =====
const marginX = 14;
const columnGap = 10;
const columnWidth = (pageWidth - marginX * 2 - columnGap) / 2;

const leftX = marginX;
const rightX = marginX + columnWidth + columnGap;
// ===== FECHA =====
    const fecha = new Date().toLocaleDateString('es-CL');
// ===== COLUMNA IZQUIERDA · EMISOR =====
let leftY = 57; // punto de partida vertical

doc.setFont('helvetica', 'italic');
doc.setFontSize(11);
doc.setTextColor(0, 0, 0);
doc.text('Cotización emitida por', leftX, leftY);

leftY += 6;
doc.setFontSize(11);
doc.setFont('helvetica', 'bold');
doc.text('Maquimport Maquinaria e Importaciones SpA', leftX, leftY);

leftY += 7;
doc.setFont('helvetica', 'normal');
doc.setFontSize(10);
doc.text('www.maquimport.cl', leftX, leftY);

leftY += 5;
doc.text('ventas@maquimport.cl', leftX, leftY);

leftY += 5;
doc.text('+56 9 9537 2766', leftX, leftY);

leftY += 7;
doc.setFontSize(10);
doc.text('Fecha de emisión: ' + fecha, leftX, leftY);
// ===== COLUMNA DERECHA · CLIENTE =====
let rightY = 57;

doc.setFont('helvetica', 'italic');
doc.setFontSize(11);
doc.setTextColor(0, 0, 0);
doc.text('Cotización para', rightX, rightY);

rightY += 6;
doc.setFontSize(11);
doc.setFont('helvetica', 'bold');
doc.text(clientData.company_name, rightX, rightY);

doc.setFont('helvetica', 'normal');
doc.setFontSize(10);

rightY += 6;
doc.text('RUT: ' + clientData.rut, rightX, rightY);


rightY += 5;
doc.text(clientData.address_city, rightX, rightY);

rightY += 5;
doc.text('Contacto: ' + clientData.contact_name, rightX, rightY);


rightY += 5;
doc.text('Tel: ' + clientData.phone, rightX, rightY);

rightY += 5;
doc.text('Email: ' + clientData.email, rightX, rightY);

const headerEndY = Math.max(leftY, rightY);


    
    // ===== TÍTULO LISTA DE PRODUCTOS =====
    let tablaStartY = headerEndY + 12;

    doc.setFont('helvetica', 'bold');
    doc.setFontSize(11);
    doc.setTextColor(0, 0, 0);
    doc.text('Lista de productos y precios', 14, tablaStartY);

    // ===== TABLA DE PRODUCTOS =====
    const bodyRows = cart.items.map(function (item, index) {
      const titulo = item.product_title || '';
      const variante = item.variant_title ? ' (' + item.variant_title + ')' : '';
      const cantidad = item.quantity;
      const precioUnit = item.price / 100;
      const totalLinea = item.final_line_price / 100;

      return [
        index + 1,
        titulo + variante,
        cantidad,
        'CLP ' + precioUnit.toLocaleString('es-CL'),
        'CLP ' + totalLinea.toLocaleString('es-CL')
      ];
    });

    const subtotal = cart.items_subtotal_price / 100;

    doc.autoTable({
      startY: tablaStartY + 4,
      head: [['#', 'Producto', 'Cant.', 'Precio unitario Neto', 'Total Neto']],
      body: bodyRows,
      styles: {
        fontSize: 9,
        textColor: [40, 40, 40], // gris oscuro
        fillColor: [245, 247, 249]  // 👈 fondo suave para TODAS las filas
      },
      headStyles: {
        fillColor: [0x15, 0x5c, 0x76], // #155C76
        textColor: [255, 255, 255],   // blanco
      },
      alternateRowStyles: {
      fillColor: [245, 247, 249] // 👈 refuerzo (por seguridad visual)
       },
      columnStyles: {
      0: { halign: 'center', cellWidth: 10 },   // #
      1: { cellWidth: 78 },                     // Producto
      2: { halign: 'center', cellWidth: 18 },   // Cant.
      3: { halign: 'right', cellWidth: 36 },    // Precio unitario Neto
      4: { halign: 'right', cellWidth: 40 }     // Total Neto
      },
      didParseCell: function (data) {
    if (data.section === 'head') {
      if (data.column.index === 3 || data.column.index === 4) {
        data.cell.styles.halign = 'right';
      }
    }
  }

    });

    let finalY = doc.lastAutoTable.finalY || (tablaStartY + 20);

    // ===== RESUMEN DE VALORES =====
var ivaRate = 0.19;
var ivaMonto = subtotal * ivaRate;
var totalFinal = subtotal + ivaMonto;

// micro ajuste visual del resumen
var microOffset = 6; // 

// posiciones base (con micro ajuste)
var labelX = pageWidth - 90 - microOffset;
var valueX = pageWidth - 10 - microOffset;
var summaryY = finalY + 10;
// línea divisoria
doc.setDrawColor(220, 220, 220);
doc.setLineWidth(0.3);
doc.line(14, summaryY - 6, pageWidth - 14, summaryY - 6);

// textos
doc.setFontSize(9);
doc.setTextColor(0, 0, 0);
doc.setFont('helvetica', 'normal');

// ajustes finos SOLO del contenido del resumen
var labelTextOffset = 6;   // mueve los conceptos a la derecha
var valueTextOffset = 0;   // precios quietos


doc.text('Subtotal Neto:', labelX + labelTextOffset, summaryY);
doc.text('CLP ' + subtotal.toLocaleString('es-CL'),valueX + valueTextOffset, summaryY, { align: 'right' });

doc.text('IVA 19%:', labelX + labelTextOffset, summaryY + 7);
doc.text('CLP ' + ivaMonto.toLocaleString('es-CL'), valueX, summaryY + 7, { align: 'right' });

doc.setFontSize(10);
doc.setFont('helvetica', 'bold');
doc.setTextColor(21, 92, 118);
doc.text('Valor total:', labelX + labelTextOffset, summaryY + 14);
doc.text('CLP ' + totalFinal.toLocaleString('es-CL'), valueX, summaryY + 14, { align: 'right' });

// ===== RECUADRO RESUMEN (AQUÍ VA) =====
var boxPaddingLeft  = 0.2; // 👈 acerca la línea izquierda al texto
var boxPaddingRight = 2.5;   // mantenemos respiración a la derecha

var boxLeftX  = labelX + 0.8;
var boxRightX = valueX + boxPaddingRight;

var boxY      = summaryY - 6;
var boxHeight = 22;
var boxWidth  = boxRightX - boxLeftX;

doc.setDrawColor(220, 220, 220);
doc.setLineWidth(0.3);
doc.rect(boxLeftX, boxY, boxWidth, boxHeight, 'S');


// ===== BLOQUE TEXTO FINAL (ordenado) =====
let currentY = finalY + 40;

// ===== VALIDEZ (7 días en bold) =====
doc.setFontSize(10);
doc.setTextColor(0, 0, 0);

// parte 1 (italic)
doc.setFont('helvetica', 'italic');
const textPart1 = '• Esta cotización tiene una validez de ';
doc.text(textPart1, 14, currentY);

// ancho parte 1
const part1Width = doc.getTextWidth(textPart1);

// parte 2 (bold)
doc.setFont('helvetica', 'bold');
const textPart2 = '7 días corridos';
doc.text(textPart2, 14 + part1Width, currentY);

// ancho parte 2
const part2Width = doc.getTextWidth(textPart2);

// parte 3 (italic)
doc.setFont('helvetica', 'italic');
const textPart3 = ' desde la fecha de emisión.';
doc.text(textPart3, 14 + part1Width + part2Width, currentY);

currentY += 7;

// ===== CONDICIONES COMERCIALES =====
doc.setFont('helvetica', 'normal');
doc.text(
  'Los valores están sujetos a disponibilidad de stock, y a las condiciones comerciales vigentes al momento de la compra.',
  14,
  currentY
);

currentY += 10;

// ===== CONTACTO =====
doc.text(
  '• Si tienes alguna duda o necesitas asistencia para avanzar con tu cotización, puedes contactarnos',
  14,
  currentY
);

currentY += 7;

doc.text(
  'vía WhatsApp al +56 9 9537 2766 o escribirnos a ventas@maquimport.cl.',
  14,
  currentY
);


currentY += 10;

// ===== LINK RECUPERACIÓN DE CARRITO =====
const cartRestoreUrl = `${window.location.origin}/cart/c/${cart.token}`;

doc.setFont('helvetica', 'normal');
doc.setTextColor(0, 0, 0);

doc.text(
  '• Recuerda que puedes finalizar tu compra a través del siguiente enlace:',
  14,
  currentY
);

currentY += 6;

// link en color marca
doc.setTextColor(21, 92, 118);
doc.textWithLink(
  cartRestoreUrl,
  14,
  currentY,
  { url: cartRestoreUrl }
);

// underline manual
const linkWidth = doc.getTextWidth(cartRestoreUrl);
doc.setLineWidth(0.5);
doc.line(14, currentY + 1.2, 14 + linkWidth, currentY + 1.2);

// reset color
doc.setTextColor(0, 0, 0);


// reset
doc.setFont('helvetica', 'normal');
doc.setTextColor(0, 0, 0);
doc.setFontSize(9);



    // ===== FIRMA + FRASE FINAL =====
    try {
      if (window.MAQUIMPORT_PDF && window.MAQUIMPORT_PDF.firma) {
        let firmaY = finalY + 70;
        if (firmaY > 250) firmaY = 250; // por si la tabla es muy larga

        const firmaWidth  = 70;
        const firmaHeight = 60;
        const firmaX      = pageWidth - firmaWidth - 6;

        const firmaBase64 = await loadImageAsBase64(window.MAQUIMPORT_PDF.firma);
        doc.addImage(
        firmaBase64,
        'PNG',
        firmaX,
        firmaY,
        firmaWidth,
        firmaHeight
        );


        doc.setFont('helvetica', 'italic');
        doc.setFontSize(10);
        doc.text(
          'Comprometidos con el éxito de tu proyecto.',
          firmaX -3,
          firmaY + 40
        );
      }
    } catch (e) {
      console.warn('No se pudo agregar la firma en el PDF', e);
    }

    // ===== DESCARGA =====
    doc.save(`maquimport-cotizacion-${quotationNumber || 'sin-numero'}.pdf`);
  } catch (e) {
    console.error(e);
    alert('Hubo un problema al generar la cotización. Intenta nuevamente.');
  }
}

  // Inserta el botón dentro del Slide Cart
  window.maquimportInsertQuoteButtonInSlideCart = function () {
    // 1) Ubicar el contenedor principal del slide cart
    const slideCart = document.querySelector('#slidecarthq');
    if (!slideCart) return;

    // 2) Buscar el botón de "Finalizar compra" dentro del slide cart
    const checkoutBtn = slideCart.querySelector('button.button.full');
    if (!checkoutBtn) return;

    const footerContainer = checkoutBtn.parentElement;
    if (!footerContainer) return;

    // Evitar duplicados
    if (footerContainer.querySelector('#maquimport-quote-pdf-btn')) return;

    // 3) Crear el botón de cotización
    const quoteBtn = document.createElement('button');
    quoteBtn.id = 'maquimport-quote-pdf-btn';
    quoteBtn.type = 'button';
    quoteBtn.innerText = 'Generar cotización PDF';

    // Mismo estilo visual que el botón de checkout
    quoteBtn.className = checkoutBtn.className;

    // 4) Insertar el botón justo ANTES de "Finalizar compra"
    footerContainer.insertBefore(quoteBtn, checkoutBtn);


  }

 document.addEventListener('DOMContentLoaded', function () {

  // intento inmediato
  tryInsertQuoteButton();

  // === FASE 2A: modal datos empresa ===
  initMaquimportCompanyModal();

  // === FASE 2A: conexión botón ↔ modal ===

});
let tries = 0;
const interval = setInterval(function () {
  tries++;
  tryInsertQuoteButton();
  if (tries > 20) {
    clearInterval(interval);
  }
}, 500);
// Observer por si la app re-renderiza el DOM
const observer = new MutationObserver(function () {
  tryInsertQuoteButton();
});

observer.observe(document.body, {
  childList: true,
  subtree: true
});

</script>

<!-- Modal: Datos de Empresa -->
<div id="maquimport-company-modal" class="maquimport-modal" aria-hidden="true">

  <div class="maquimport-modal-overlay"></div>

  <div class="maquimport-modal-content" role="dialog" aria-modal="true" aria-labelledby="maquimport-modal-title">

    <!-- Header -->
    <header class="maquimport-modal-header">
      <h2 id="maquimport-modal-title">
        Ingresa los datos de tu empresa
      </h2>
      <button type="button" class="maquimport-modal-close" aria-label="Cerrar modal">
        ×
      </button>
    </header>

    <!-- Body -->
    <div class="maquimport-modal-body">

      <p class="maquimport-modal-description">
        Usaremos esta información para generar tu cotización en PDF. No se solicitará nuevamente.
      </p>

      <form id="maquimport-company-form" novalidate>

        <div class="form-group">
          <label for="company_name">Nombre de la empresa o Razón social </label>
          <input
            type="text"
            id="company_name"
            name="company_name"
            placeholder="Ej: Maquimport Maquinaria e Importaciones SpA"
            required
          >
        </div>

        <div class="form-group">
          <label for="rut">RUT de la empresa</label>
          <input
            type="text"
            id="rut"
            name="rut"
            placeholder="Ej: 76.123.456-7"
            required
          >
        </div>

         <div class="form-group">
          <label for="address_city">Dirección y ciudad</label>
          <input
            type="text"
            id="address_city"
            name="address_city"
            placeholder="Ej: Av. Industrial 1234, Viña del Mar"
            required
          >
        </div>

        <div class="form-group">
  <label for="contact_name">Nombre del contacto</label>
  <input
    type="text"
    id="contact_name"
    name="contact_name"
    placeholder="Ej: Nombre completo del contacto"
    required
  >
</div>


        
        <div class="form-group">
          <label for="phone">Teléfono de contacto</label>
          <input
            type="tel"
            id="phone"
            name="phone"
            placeholder="Ej: +56 9 1234 5678"
            required
          >
        </div>

        <div class="form-group">
          <label for="email">Correo electrónico</label>
          <input
            type="email"
            id="email"
            name="email"
            placeholder="Ej: compras@empresa.cl"
            required
          >
        </div>

        <p class="maquimport-modal-legal">
          Al continuar, confirmas que la información ingresada es correcta.
        </p>

        <!-- Footer / Actions -->
        <div class="maquimport-modal-actions">
          <button type="button" class="btn-secondary maquimport-modal-cancel">
            Cancelar
          </button>

         <button type="submit" class="btn-primary" id="maquimport-submit-btn">
            Guardar y descargar cotización
            
          </button>
        </div>

      </form>
    </div>

  </div>
</div>

<script>
  // === MAQUIMPORT · MODAL DATOS DE EMPRESA ===
// === Referencias del modal ===

// === Estado interno PDF ===
window.__maquimportPdfGenerated = false;

function closeSlideCartIfOpen() {
  const closeBtn = document.querySelector('button[aria-label="close cart"]');
  if (closeBtn) {
    closeBtn.click();
  }
}

  const MAQUIMPORT_USER_STORAGE_KEY = 'maquimport_user_data';

  function requireMaquimportUserData(callback) {
  const data = getMaquimportUserData();

  if (data) {
    // Ya hay datos → seguimos directo
    callback(data);
  } else {
    // No hay datos → abrimos modal
    openMaquimportModal();
  }
}

  function getMaquimportUserData() {
    const data = localStorage.getItem(MAQUIMPORT_USER_STORAGE_KEY);
    return data ? JSON.parse(data) : null;
  }

  function saveMaquimportUserData(data) {
    localStorage.setItem(
      MAQUIMPORT_USER_STORAGE_KEY,
      JSON.stringify(data)
    );
  }
  // === Reset formulario (modo ingreso) ===
function resetMaquimportForm() {
  const form = document.getElementById('maquimport-company-form');
  if (!form) return;

  form.reset();
}
function fillMaquimportForm(data) {
  const form = document.getElementById('maquimport-company-form');
  if (!form || !data) return;

  form.company_name.value = data.company_name || '';
  form.rut.value = data.rut || '';
  form.address_city.value = data.address_city || '';
  form.contact_name.value = data.contact_name || '';
  form.phone.value = data.phone || '';
  form.email.value = data.email || '';
}
function openMaquimportModalWithMode() {
  const modalTitle = document.getElementById('maquimport-modal-title');
  const modalDescription = document.querySelector('.maquimport-modal-description');
  const submitBtn = document.querySelector('#maquimport-company-form button[type="submit"]');

  if (!modalTitle || !modalDescription || !submitBtn) {
    console.warn('[Maquimport] Modal DOM no listo');
    return;
  }

  const existingData = getMaquimportUserData();

  if (existingData) {
    // 🟢 MODO CONFIRMACIÓN
    modalTitle.textContent = 'Confirma los datos para esta cotización';
    modalDescription.innerHTML =
      'Verifica que la información sea correcta. Si necesitas cambiar algún dato, puedes hacerlo directamente aquí antes de generar la cotización en PDF.<br><em>Usaremos estos datos para emitir el documento.</em>';

    submitBtn.textContent = 'Confirmar datos y generar cotización';
    fillMaquimportForm(existingData);
  } else {
    // 🔵 MODO INGRESO
    modalTitle.textContent = 'Ingresa los datos de tu empresa';
    modalDescription.textContent =
      'Usaremos esta información para generar tu cotización en PDF.';

    submitBtn.textContent = 'Guardar y generar cotización';
    resetMaquimportForm();
  }

  openMaquimportModal();
}


  function openMaquimportModal() {
  const modal = document.getElementById('maquimport-company-modal');
  if (!modal) return;

  // 1) Cerrar el carrito
  closeSlideCartIfOpen();

  // 2) Abrir el modal (delay leve para mobile)
  setTimeout(() => {
    modal.setAttribute('aria-hidden', 'false');
    modal.classList.add('is-open');
  }, 150);
}


  function closeMaquimportModal() {
    const modal = document.getElementById('maquimport-company-modal');
    if (!modal) return;
    modal.setAttribute('aria-hidden', 'true');
    modal.classList.remove('is-open');
  }

  function isValidEmail(email) {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  function isValidPhone(phone) {
    return phone.replace(/\D/g, '').length >= 8;
  }

  function validateMaquimportForm(form) {
    const data = {
      company_name: form.company_name.value.trim(),
      rut: form.rut.value.trim(),
      address_city: form.address_city.value.trim(),
       contact_name: form.contact_name.value.trim(),
      phone: form.phone.value.trim(),
      email: form.email.value.trim()
    };

    if (Object.values(data).some(value => !value)) {
      return { valid: false, message: 'Todos los campos son obligatorios.' };
    }

    if (!isValidPhone(data.phone)) {
      return { valid: false, message: 'Ingresa un teléfono válido.' };
    }

    if (!isValidEmail(data.email)) {
      return { valid: false, message: 'Ingresa un correo electrónico válido.' };
    }

    return { valid: true, data };
  }
  // === Utilidades · Toast descarga ===
function isMobileDevice() {
  return /Android|iPhone|iPad|iPod/i.test(navigator.userAgent);
}

function showMaquimportDownloadToast() {
  const toast = document.getElementById('maquimport-download-toast');
  const messageEl = toast?.querySelector('.maquimport-toast-message');
  if (!toast || !messageEl) return;

  if (isMobileDevice()) {
    messageEl.innerHTML =
      'La cotización se ha generado. Puedes guardarla o compartirla desde las opciones de tu dispositivo.';
  } else {
    messageEl.innerHTML =
    'La cotización <strong>se ha descargado</strong> y se encuentra en tu <strong>carpeta de descargas</strong>. Si tu navegador lo solicita, selecciona dónde deseas guardarla.';
  }

  toast.setAttribute('aria-hidden', 'false');
  toast.classList.add('is-visible');

  setTimeout(() => {
    toast.classList.remove('is-visible');
    toast.setAttribute('aria-hidden', 'true');
  }, 6000);
}


  async function handleMaquimportFormSubmit(event) {
    console.log('[DEBUG] submit capturado');
    event.preventDefault();

    const form = event.target;
    const validation = validateMaquimportForm(form);

    if (!validation.valid) {
      alert(validation.message); // luego lo pasamos a UI
      return;
    }
saveMaquimportUserData(validation.data);
closeMaquimportModal();

// ==============================
// Crear cotización en Supabase
// ==============================
let quotationNumber = null;

try {
  // obtener carrito actual
  const cartRes = await fetch('/cart.js');
  const cart = await cartRes.json();

  // DEBUG – antes del fetch
console.log('[DEBUG] Enviando cotización a Supabase', {
  company_name: validation.data.company_name,
  email: validation.data.email,
  cart_token: cart.token
});

  const res = await fetch(
  'https://hkwoxtgiwymgrakadpvp.functions.supabase.co/create-quotation',
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
       'apikey': 'sb_publishable_tgOw6ifuyd07pMwHLG7X9A_SW7xEurt'
    },
    body: JSON.stringify({
      company_name: validation.data.company_name,
      company_rut: validation.data.rut,
      contact_name: validation.data.contact_name,
      email: validation.data.email,
      phone: validation.data.phone,
      address_city: validation.data.address_city,

      cart_token: cart.token,
      cart_restore_url: `${window.location.origin}/cart/c/${cart.token}`,

      items: cart.items,
      subtotal_net: cart.items_subtotal_price / 100,
      iva_19: (cart.items_subtotal_price / 100) * 0.19,
      total_gross: (cart.items_subtotal_price / 100) * 1.19
    })
  });

  if (res.ok) {
    const data = await res.json();
    quotationNumber = data.quotation_number;
  }
} catch (err) {
  console.warn('Error creando cotización en Supabase:', err);
}

setTimeout(() => {
  maquimportGenerateQuotePdf(quotationNumber);

  // Marcamos que se generó un PDF
  window.__maquimportPdfGenerated = true;

  // Desktop: mostramos mensaje inmediato
  if (!isMobileDevice()) {
    setTimeout(() => {
      showMaquimportDownloadToast();
    }, 200);
  }

}, 150);

    // 🔜 aquí luego llamamos al PDF
    // maquimportGenerateQuotePdf();
  }

  function initMaquimportCompanyModal() {
    console.log('[DEBUG] initMaquimportCompanyModal ejecutada');
    const form = document.getElementById('maquimport-company-form');
    const closeBtn = document.querySelector('.maquimport-modal-close');
    const cancelBtn = document.querySelector('.maquimport-modal-cancel');

    if (form) {
      form.addEventListener('submit', handleMaquimportFormSubmit);
    }

    if (closeBtn) {
      closeBtn.addEventListener('click', closeMaquimportModal);
    }

    if (cancelBtn) {
      cancelBtn.addEventListener('click', closeMaquimportModal);
    }
  }
// === LISTENER GLOBAL BOTÓN COTIZACIÓN ===
document.addEventListener('click', function (event) {
  const btn = event.target.closest('#maquimport-quote-pdf-btn');
  if (!btn) return;

  event.preventDefault();
  openMaquimportModalWithMode();
});
m

// === Mostrar mensaje al volver desde visor PDF (mobile) ===
document.addEventListener('visibilitychange', function () {
  if (document.visibilityState === 'visible') {
    if (isMobileDevice() && window.__maquimportPdfGenerated) {
      showMaquimportDownloadToast();
      window.__maquimportPdfGenerated = false;
    }
  }
});

</script>

<!-- Toast / Aviso post-descarga -->
<div id="maquimport-download-toast" class="maquimport-toast" aria-hidden="true">
  <div class="maquimport-toast-content">
    <h3 class="maquimport-toast-title">Cotización generada correctamente</h3>
    <p class="maquimport-toast-message"></p>
  </div>
</div>

</body>

</html>

