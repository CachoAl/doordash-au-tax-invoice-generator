# doordash-au-tax-invoice-generator
Browser-based tool that turns a NetSuite "Ads &amp; Promos" invoice into a DoorDash Australia manual GST tax invoice — auto-extracts the figures, applies the taxable/redemptions rules, checks zero variance, and exports the PDF. No backend.
[index.html](https://github.com/user-attachments/files/30320408/index.html)
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>DoorDash AU — Manual Tax Invoice Generator</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;600;700&family=Inter:wght@400;500;600&family=IBM+Plex+Mono:wght@400;500;600&display=swap" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
<style>
  :root{
    --ink:#14171A; --canvas:#E9ECE8; --panel:#FFFFFF; --line:#D6DAD5;
    --muted:#5B6560; --faint:#8A938D; --dd:#EB1700; --dd-soft:#FBE9E6;
    --ok:#1E7A46; --ok-soft:#E4F2EA; --bad:#C0362C; --bad-soft:#FBE7E5;
    --field:#F4F6F3;
    --mono:'IBM Plex Mono',ui-monospace,monospace;
    --ui:'Inter',system-ui,sans-serif;
    --display:'Space Grotesk',system-ui,sans-serif;
  }
  *{box-sizing:border-box}
  body{margin:0;color:var(--ink);font-family:var(--ui);font-size:14px;line-height:1.45;-webkit-font-smoothing:antialiased;
    background:
      radial-gradient(1100px 520px at 8% -8%, #ffffff 0%, rgba(255,255,255,0) 60%),
      radial-gradient(900px 520px at 108% 0%, #f3f5f2 0%, rgba(243,245,242,0) 55%),
      linear-gradient(158deg,#eef1ed 0%,#e4e8e3 62%,#dce1db 100%);
    background-attachment:fixed}
  .app{display:grid;grid-template-columns:minmax(0,500px) minmax(0,1fr);gap:0;min-height:100vh}
  /* ---------- left: console (bento stack) ---------- */
  .console{padding:24px 22px 64px;overflow-y:auto;max-height:100vh}
  .brand{display:flex;align-items:baseline;gap:10px;margin:4px 4px 2px}
  .brand h1{font-family:var(--display);font-weight:700;font-size:23px;letter-spacing:-.02em;margin:0}
  .brand .rule{font-family:var(--mono);font-size:10px;color:var(--dd);letter-spacing:.14em;text-transform:uppercase;border:1px solid var(--dd);border-radius:5px;padding:2px 6px}
  .sub{color:var(--muted);font-size:12.5px;margin:0 4px 20px;max-width:44ch}
  /* glass bento card */
  .step{
    background:rgba(255,255,255,.7);
    -webkit-backdrop-filter:blur(16px) saturate(150%); backdrop-filter:blur(16px) saturate(150%);
    border:1px solid rgba(255,255,255,.75);
    box-shadow:0 1px 1px rgba(20,23,26,.03), 0 14px 34px -16px rgba(20,23,26,.22);
    border-radius:22px;padding:20px;margin-bottom:16px}
  .step-h{display:flex;align-items:center;gap:10px;margin-bottom:14px}
  .step-n{font-family:var(--mono);font-size:11px;color:#fff;background:linear-gradient(150deg,#2a2f33,#14171A);width:22px;height:22px;border-radius:7px;display:grid;place-items:center;flex:none;box-shadow:0 2px 6px rgba(20,23,26,.25)}
  .step-h h2{font-family:var(--display);font-weight:600;font-size:13px;letter-spacing:.03em;text-transform:uppercase;margin:0;color:var(--ink)}
  /* dropzone */
  .drop{display:flex;align-items:center;gap:14px;border:1.5px dashed var(--line);border-radius:12px;background:var(--field);padding:15px 16px;cursor:pointer;transition:.15s}
  .drop:hover,.drop.hot{border-color:var(--dd);background:var(--dd-soft)}
  .drop-ic{width:44px;height:44px;border-radius:11px;background:#fff;border:1px solid var(--line);display:grid;place-items:center;flex:none;transition:.15s}
  .drop:hover .drop-ic,.drop.hot .drop-ic{border-color:var(--dd)}
  .drop-ic svg{width:22px;height:22px;display:block}
  .drop-txt{display:flex;flex-direction:column;gap:2px;min-width:0}
  .drop-txt b{font-weight:600;font-size:13.5px}
  .drop-txt small{color:var(--faint);font-size:11.5px}
  .drop input{display:none}
  .loaded{display:none;align-items:center;gap:9px;margin-top:10px;font-size:12.5px;color:var(--ok);font-weight:500;background:var(--ok-soft);border:1px solid #cfe6d9;border-radius:9px;padding:9px 12px}
  .loaded.show{display:flex}
  .loaded .chk{font-weight:700}
  .loaded #loadedName{overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
  .loaded .replace{margin-left:auto;background:none;border:none;color:var(--muted);font-size:11.5px;text-decoration:underline;cursor:pointer;font-family:var(--ui);flex:none}
  .loaded .replace:hover{color:var(--ink)}
  .paste-toggle{background:none;border:none;color:var(--muted);font-size:11.5px;text-decoration:underline;cursor:pointer;padding:8px 0 0;font-family:var(--ui)}
  textarea.paste{display:none;width:100%;height:120px;margin-top:8px;font-family:var(--mono);font-size:11px;border:1px solid var(--line);border-radius:8px;padding:9px;resize:vertical}
  textarea.paste.show{display:block}
  /* fields */
  .grid2{display:grid;grid-template-columns:1fr 1fr;gap:10px 12px}
  .fld{display:flex;flex-direction:column;gap:4px}
  .fld.full{grid-column:1/-1}
  .fld label{font-size:10.5px;text-transform:uppercase;letter-spacing:.06em;color:var(--muted);font-weight:600}
  .fld input,.fld textarea{border:1px solid var(--line);border-radius:7px;background:var(--field);padding:8px 9px;font-family:var(--ui);font-size:13px;color:var(--ink);width:100%}
  .fld input:focus,.fld textarea:focus{outline:none;border-color:var(--ink);background:#fff}
  .fld textarea{resize:vertical;min-height:82px;line-height:1.4}
  .fld .hint{font-size:10.5px;color:var(--faint)}
  .fld.abn input{border-color:var(--dd);background:var(--dd-soft);font-family:var(--mono);letter-spacing:.02em}
  .fld.abn label{color:var(--dd)}
  .fld.locked input{background:#eef1ee;color:var(--muted);border-style:dashed}
  .badge-fixed{font-family:var(--mono);font-size:9px;color:var(--faint);border:1px solid var(--line);border-radius:3px;padding:0 4px;margin-left:6px}
  /* line items */
  table.items{width:100%;border-collapse:collapse;margin-top:2px}
  table.items th{font-size:9.5px;text-transform:uppercase;letter-spacing:.05em;color:var(--muted);font-weight:600;text-align:right;padding:0 4px 6px;border-bottom:1px solid var(--line)}
  table.items th:first-child{text-align:left}
  table.items td{padding:4px 3px;vertical-align:middle}
  table.items input{width:100%;border:1px solid transparent;border-radius:6px;background:transparent;padding:6px 5px;font-family:var(--mono);font-size:12px;text-align:right;color:var(--ink)}
  table.items input.desc{text-align:left;font-family:var(--ui)}
  table.items input:hover{background:var(--field)}
  table.items input:focus{outline:none;border-color:var(--ink);background:#fff}
  table.items td.incl{font-family:var(--mono);font-size:12px;text-align:right;color:var(--muted);padding-right:6px;white-space:nowrap}
  .taxchk{display:grid;place-items:center}
  .taxchk input{width:15px;height:15px;accent-color:var(--dd);cursor:pointer}
  .rm{background:none;border:none;color:var(--faint);cursor:pointer;font-size:15px;line-height:1;padding:2px 4px}
  .rm:hover{color:var(--bad)}
  .additem{margin-top:8px;background:none;border:1px dashed var(--line);border-radius:7px;color:var(--muted);padding:7px;width:100%;cursor:pointer;font-family:var(--ui);font-size:12px}
  .additem:hover{border-color:var(--ink);color:var(--ink)}
  .tax-legend{font-size:10.5px;color:var(--faint);margin-top:8px}
  .tax-legend b{color:var(--muted)}
  /* variance meter — signature */
  .meter{margin-top:2px;border:1px solid rgba(255,255,255,.75);border-radius:20px;overflow:hidden;
    background:rgba(255,255,255,.7);-webkit-backdrop-filter:blur(16px) saturate(150%);backdrop-filter:blur(16px) saturate(150%);
    box-shadow:0 1px 1px rgba(20,23,26,.03), 0 14px 34px -16px rgba(20,23,26,.22)}
  .meter-row{display:flex;justify-content:space-between;align-items:center;padding:12px 16px;font-size:12.5px}
  .meter-row+.meter-row{border-top:1px solid var(--line)}
  .meter-row .k{color:var(--muted)}
  .meter-row .v{font-family:var(--mono);font-weight:500}
  .meter-row.ns .v{color:var(--ink)}
  .meter-status{padding:12px 14px;display:flex;align-items:center;gap:10px;font-weight:600;font-size:13px;transition:.2s}
  .meter-status .dot{width:10px;height:10px;border-radius:50%;flex:none}
  .meter-status.ok{background:var(--ok-soft);color:var(--ok)}
  .meter-status.ok .dot{background:var(--ok)}
  .meter-status.bad{background:var(--bad-soft);color:var(--bad)}
  .meter-status.bad .dot{background:var(--bad)}
  .meter-status .var{margin-left:auto;font-family:var(--mono)}
  /* download */
  .go{margin-top:16px;width:100%;background:linear-gradient(160deg,#ff3b1f,#EB1700);color:#fff;border:none;border-radius:16px;padding:16px;font-family:var(--display);font-weight:600;font-size:14.5px;letter-spacing:.02em;cursor:pointer;transition:.18s;box-shadow:0 10px 24px -10px rgba(235,23,0,.6)}
  .go:hover{transform:translateY(-1px);box-shadow:0 14px 30px -10px rgba(235,23,0,.7)}
  .go:disabled{background:#cdd2cd;cursor:not-allowed;box-shadow:none;transform:none}
  .go-note{text-align:center;font-size:11px;color:var(--faint);margin-top:9px}
  .go-alt{margin-top:9px;width:100%;background:rgba(255,255,255,.6);color:var(--ink);border:1px solid var(--line);border-radius:14px;padding:12px;font-family:var(--display);font-weight:600;font-size:13px;cursor:pointer;transition:.15s;-webkit-backdrop-filter:blur(8px);backdrop-filter:blur(8px)}
  .go-alt:hover{border-color:var(--ink)}
  /* print: isolate the invoice so it exports clean anywhere */
  @media print{
    .app{display:block!important}
    .console,.stagelabel{display:none!important}
    .stage{padding:0!important;display:block!important;max-height:none!important;overflow:visible!important}
    .stage-inner{transform:none!important;width:auto!important}
    .invoice{box-shadow:none!important;border-radius:0!important;margin:0 auto!important}
    body{background:#fff!important}
    @page{size:A4;margin:0}
  }
  /* ---------- right: preview ---------- */
  .stage{padding:40px 40px 48px;display:flex;justify-content:center;align-items:flex-start;overflow:auto;max-height:100vh}
  .stage-inner{width:794px;transform-origin:top center}
  .stagelabel{width:794px;font-family:var(--mono);font-size:10.5px;color:var(--muted);letter-spacing:.14em;text-transform:uppercase;margin:0 auto 14px;display:flex;justify-content:space-between}
  /* ===== the actual tax invoice (A4) ===== */
  .invoice{width:794px;min-height:1123px;background:#fff;box-shadow:0 20px 60px -20px rgba(20,23,26,.35), 0 4px 14px rgba(20,23,26,.08);border-radius:6px;padding:54px 54px 40px;font-family:Arial,Helvetica,sans-serif;color:#111;font-size:12px}
  .invoice, .invoice *{font-family:Arial,Helvetica,sans-serif}
  .inv-top{display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:40px}
  .inv-logo{display:flex;align-items:center}
  .inv-logo .ddlogo{height:30px;width:auto;display:block}
  .inv-title{font-weight:700;font-size:31px;color:#111}
  .inv-cols{display:flex;justify-content:space-between;gap:30px;margin-bottom:56px}
  .inv-L{width:52%}
  .inv-R{width:44%}
  .inv-entity{font-weight:700;font-size:12px}
  .inv-abn{font-size:12px;margin-top:1px}
  .inv-mi-h{font-weight:700;margin-top:20px}
  .inv-mi{white-space:pre-line;font-size:12px;line-height:1.5}
  .inv-mi-abn{font-size:12px;margin-top:1px}
  .inv-meta{width:100%;border-collapse:collapse}
  .inv-meta td{padding:2px 0;font-size:12px;vertical-align:top}
  .inv-meta td.k{font-weight:700;white-space:nowrap;padding-right:14px}
  .inv-meta td.v{text-align:right}
  table.inv-tbl{width:100%;border-collapse:collapse;margin-bottom:2px}
  table.inv-tbl thead th{background:#eceff1;font-weight:700;font-size:12px;padding:10px 12px;text-align:right;vertical-align:bottom}
  table.inv-tbl thead th:first-child{text-align:left}
  table.inv-tbl tbody td{padding:6px 12px;font-size:12px;text-align:right}
  table.inv-tbl tbody td:first-child{text-align:left}
  .inv-total-row{display:flex;justify-content:space-between;align-items:flex-end;margin-top:26px;padding:0 12px}
  .inv-total-row .tl{font-weight:700;font-size:12px}
  .inv-total-line{flex:1;border-top:1px solid #111;margin:0 40px 4px;max-width:200px}
  .inv-total-row .tv{font-weight:700;font-size:12px}
  .inv-notes-h{font-weight:700;margin-top:52px}
  .inv-notes{font-size:12px;line-height:1.5;margin-top:4px;color:#222}
  .inv-foot{background:#eceff1;margin:42px -54px -40px;padding:14px 54px;display:flex;justify-content:space-between;font-size:12px}
  .inv-foot b{font-weight:700}
  .empty-note{color:#bbb;font-style:italic}
  /* responsive */
  @media (max-width:1180px){
    .app{grid-template-columns:1fr}
    .console{max-height:none;border-right:none;border-bottom:1px solid var(--line)}
    .stage{max-height:none}
    .stage-inner,.invoice,.stagelabel{transform:scale(.62);transform-origin:top left}
    .stage{overflow-x:auto}
  }
</style>
</head>
<body>
<div class="app">
  <!-- ===================== CONSOLE ===================== -->
  <aside class="console">
    <div class="brand"><h1>Manual Tax Invoice</h1><span class="rule">AU</span></div>
    <p class="sub">Drop a NetSuite “Ads and Promos” invoice, check the figures, add the ABN, and download the tax invoice PDF.</p>

    <!-- STEP 1 -->
    <section class="step">
      <div class="step-h"><span class="step-n">1</span><h2>Source invoice</h2></div>
      <label class="drop" id="drop">
        <span class="drop-ic">
          <svg viewBox="0 0 24 24" fill="none" stroke="#EB1700" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><path d="M12 16V4"/><path d="M8 8l4-4 4 4"/><path d="M4 15v3a2 2 0 002 2h12a2 2 0 002-2v-3"/></svg>
        </span>
        <span class="drop-txt"><b>Drop the NetSuite invoice</b><small>or click to browse · PDF</small></span>
        <input type="file" id="file" accept="application/pdf">
      </label>
      <div class="loaded" id="loaded"><span class="chk">✓</span><span id="loadedName"></span><button type="button" class="replace" id="replaceFile">Replace</button></div>
      <button class="paste-toggle" id="pasteToggle">Can’t read a scanned PDF? Paste the invoice text instead</button>
      <textarea class="paste" id="pasteBox" placeholder="Paste the full text of the NetSuite invoice here, then click outside the box…"></textarea>
    </section>

    <!-- STEP 2 -->
    <section class="step">
      <div class="step-h"><span class="step-n">2</span><h2>Invoice details</h2></div>
      <div class="grid2">
        <div class="fld"><label>Invoice date</label><input id="f_date" placeholder="—"><span class="hint">Default: 6th of the next month</span></div>
        <div class="fld"><label>Invoice #</label><input id="f_num" placeholder="—"></div>
        <div class="fld"><label>Period start</label><input id="f_pstart" placeholder="—"></div>
        <div class="fld"><label>Period end</label><input id="f_pend" placeholder="—"></div>
        <div class="fld full"><label>Service</label><input id="f_service" placeholder="—"></div>
        <div class="fld"><label>Currency</label><input id="f_currency" value="Australian Dollar"></div>
        <div class="fld"><label>PO</label><input id="f_po" placeholder="(blank)"></div>
      </div>
    </section>

    <!-- STEP 3 -->
    <section class="step">
      <div class="step-h"><span class="step-n">3</span><h2>Merchant &amp; ABN</h2></div>
      <div class="grid2">
        <div class="fld full"><label>Merchant information</label><textarea id="f_merchant" placeholder="Pre-filled from the invoice Bill To — edit if the client name differs (e.g. Oporto)"></textarea><span class="hint">Pre-filled from Bill To. Change it only when the merchant name/address must differ.</span></div>
        <div class="fld abn full"><label>Merchant ABN — required</label><input id="f_abn" placeholder="Enter the merchant ABN exactly as provided"><span class="hint">Not in NetSuite. Must come from the merchant — a wrong ABN is a legal risk.</span></div>
      </div>
    </section>

    <!-- STEP 4 -->
    <section class="step">
      <div class="step-h"><span class="step-n">4</span><h2>Line items</h2></div>
      <table class="items">
        <thead><tr><th style="width:38%">Description</th><th>Price excl.</th><th>GST</th><th>Incl.</th><th style="width:34px">Tax</th><th style="width:22px"></th></tr></thead>
        <tbody id="rows"></tbody>
      </table>
      <button class="additem" id="addItem">+ Add line</button>
      <p class="tax-legend"><b>Taxable rule:</b> everything is taxable except event type <b>redemptions</b> (GST = NA). Untick “Tax” to make a line non-taxable.</p>
    </section>

    <!-- VARIANCE METER -->
    <div class="meter">
      <div class="meter-row ns"><span class="k">NetSuite total (AUD)</span><input id="f_nstotal" class="v" style="border:none;background:none;text-align:right;width:120px;font-family:var(--mono)" placeholder="0.00"></div>
      <div class="meter-row"><span class="k">Tax invoice total</span><span class="v" id="m_computed">0.00</span></div>
      <div class="meter-status bad" id="m_status"><span class="dot"></span><span id="m_label">Awaiting invoice</span><span class="var" id="m_var">—</span></div>
    </div>

    <button class="go" id="download" disabled>Download tax invoice PDF</button>
    <button class="go-alt" id="printBtn">Print / Save as PDF</button>
    <p class="go-note" id="goNote">Load an invoice and enter the ABN to enable</p>
  </aside>

  <!-- ===================== STAGE / PREVIEW ===================== -->
  <main class="stage">
    <div>
      <div class="stagelabel"><span>Preview · live</span><span id="stageFile">tax_invoice.pdf</span></div>
      <div class="stage-inner">
        <div class="invoice" id="invoice">
          <div class="inv-top">
            <div class="inv-logo">
              <img class="ddlogo" alt="DoorDash" src="data:image/png;base64,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">
            </div>
            <div class="inv-title">Tax Invoice</div>
          </div>
          <div class="inv-cols">
            <div class="inv-L">
              <div class="inv-entity">DoorDash Technologies Australia Pty Ltd</div>
              <div class="inv-abn">ABN: 96 634 446 030</div>
              <div class="inv-mi-h">Merchant Information:</div>
              <div class="inv-mi" id="p_merchant"><span class="empty-note">—</span></div>
              <div class="inv-mi-abn" id="p_abn"></div>
            </div>
            <div class="inv-R">
              <table class="inv-meta">
                <tr><td class="k">Invoice Date</td><td class="v" id="p_date">—</td></tr>
                <tr><td class="k">Invoice #</td><td class="v" id="p_num">—</td></tr>
                <tr><td class="k">Period Start Date</td><td class="v" id="p_pstart">—</td></tr>
                <tr><td class="k">Period End Date</td><td class="v" id="p_pend">—</td></tr>
                <tr><td class="k">Service</td><td class="v" id="p_service">—</td></tr>
                <tr><td class="k">Currency</td><td class="v" id="p_currency">Australian Dollar</td></tr>
                <tr><td class="k">PO</td><td class="v" id="p_po"></td></tr>
              </table>
            </div>
          </div>
          <table class="inv-tbl">
            <thead><tr><th>Description</th><th>Price (GST-<br>exclusive)</th><th>GST</th><th>Price (GST-<br>inclusive)</th></tr></thead>
            <tbody id="p_rows"></tbody>
          </table>
          <div class="inv-total-row"><span class="tl">Total</span><span class="inv-total-line"></span><span class="tv" id="p_total">—</span></div>
          <div class="inv-notes-h">Notes:</div>
          <div class="inv-notes">You do not need to make a separate payment to settle this tax invoice. Please refer to the separately issued pay statement for the amount due. For all delivery details related to this tax invoice, please access the URL listed on the related pay statement or contact AR@doordash.com.</div>
          <div class="inv-foot"><span><b>Thank you for being a DoorDash partner!</b></span><span><b>Need help?</b> help@doordash.com</span></div>
        </div>
      </div>
    </div>
  </main>
</div>

<script>
pdfjsLib.GlobalWorkerOptions.workerSrc = "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js";

const MONTHS = {jan:'January',feb:'February',mar:'March',apr:'April',may:'May',jun:'June',jul:'July',aug:'August',sep:'September',oct:'October',nov:'November',dec:'December'};
const MFULL = Object.values(MONTHS);
const num = s => parseFloat(String(s).replace(/[^0-9.\-]/g,'')) || 0;
const money = n => (Math.round(n*100)/100).toLocaleString('en-AU',{minimumFractionDigits:2,maximumFractionDigits:2});

let items = [];   // {description, priceExcl, gst(number|'NA'), taxable}

/* ---------------- PDF → {text, items} ---------------- */
async function extractPdf(arrayBuffer){
  const pdf = await pdfjsLib.getDocument({data:arrayBuffer}).promise;
  let out=''; const positioned=[];
  for(let p=1;p<=pdf.numPages;p++){
    const page = await pdf.getPage(p);
    const tc = await page.getTextContent();
    const raw = tc.items.filter(t=>t.str && t.str.trim()!=='')
      .map(t=>({x:t.transform[4], y:t.transform[5] - (p-1)*100000, s:t.str})); // page offset: pages stack top→bottom
    positioned.push(...raw);
    // clustered flat text (used for header fields only)
    const byY = raw.slice().sort((a,b)=>b.y-a.y);
    let cur=null; const lines=[];
    byY.forEach(t=>{
      if(cur && Math.abs(t.y-cur.y)<=3) cur.parts.push(t);
      else { cur={y:t.y, parts:[t]}; lines.push(cur); }
    });
    lines.forEach(l=>{
      const s=l.parts.sort((a,b)=>a.x-b.x).map(o=>o.s).join(' ').replace(/\s+/g,' ').trim();
      if(s) out+=s+'\n';
    });
  }
  return {text:out, items:positioned};
}

/* ---------------- geometry table parser (PDF uploads) ---------------- */
function parseItemsTable(pItems){
  const nz = s=>String(s).replace(/\s+/g,' ').trim();
  const amtRe=/^\$?[\d,]+\.\d{2}$/;
  const totalItem  = pItems.find(i=>/^total/i.test(nz(i.s)));
  const totalY     = totalItem ? totalItem.y : -Infinity;
  const headerItem = pItems.find(i=>/^amount$/i.test(nz(i.s)));
  const headerY    = headerItem ? headerItem.y : Infinity;
  const rows = pItems.filter(i=>amtRe.test(nz(i.s)) && i.y < headerY-1 && i.y > totalY+1)
                     .sort((a,b)=>b.y-a.y);
  const uuidItems = pItems.filter(i=>/^[0-9a-f]{8}/i.test(nz(i.s)));
  const campX = uuidItems.length ? Math.min(...uuidItems.map(i=>i.x)) : Infinity;
  const ys = rows.map(r=>r.y);
  const raws=[];
  rows.forEach((amt,i)=>{
    const yTop = i===0 ? headerY : (ys[i-1]+ys[i])/2;
    const yBot = i===rows.length-1 ? totalY : (ys[i]+ys[i+1])/2;
    const inRow = pItems.filter(it=> it.y<yTop && it.y>yBot);
    let label = inRow.filter(it=> it.x < campX-5)
      .sort((a,b)=> Math.abs(a.y-b.y)>2 ? b.y-a.y : a.x-b.x)
      .map(it=>it.s).join(' ').replace(/\s+/g,' ').trim();
    const ci = label.search(/[0-9a-f]{4,}-[0-9a-f]{2,}|[0-9a-f]{8}/i); // cut off any campaign-id fragment
    if(ci>0) label = label.slice(0,ci).trim();
    const tail = inRow.filter(it=> Math.abs(it.y-amt.y)<=3)
      .sort((a,b)=>a.x-b.x).map(it=>it.s).join(' ');
    let ev=''; let m = tail.match(/UTC\s+([A-Za-z][A-Za-z ]*?)\s+[\d,]+\s+\$?[\d,]+\.\d{2}/);
    if(!m) m = tail.match(/([A-Za-z][A-Za-z ]*?)\s+[\d,]+\s+\$?[\d,]+\.\d{2}\s*$/);
    if(m) ev=m[1].trim();
    raws.push({label, eventType:ev, amount:num(amt.s)});
  });
  return raws;
}

/* ---------------- text row parser (paste fallback) ---------------- */
function parseTextRows(text){
  const hIdx = text.search(/Event Count\s+Amount/);
  const tIdx = text.search(/Total\s*\(AUD\)/);
  const body = (hIdx>=0 && tIdx>hIdx) ? text.slice(text.indexOf('Amount',hIdx)+6, tIdx) : text;
  const rowRe = /([\s\S]*?)UTC\s+([A-Za-z][A-Za-z ]*?)\s+([\d,]+)\s+\$([\d,]+\.\d{2})/g;
  const uuidRe = /[0-9a-f]{8}-/i;
  const raws=[]; let m;
  while((m=rowRe.exec(body))!==null){
    const pre=m[1]; const ui=pre.search(uuidRe);
    const label=(ui>=0?pre.slice(0,ui):pre).replace(/\s+/g,' ').trim();
    raws.push({label, eventType:m[2].trim(), amount:num(m[4])});
  }
  return raws;
}

/* ---------------- group raw lines → invoice items ---------------- */
function groupLines(raws){
  const groups={};
  raws.forEach(l=>{
    const isTax=/^tax$/i.test(l.eventType);
    const base=isTax? l.label.replace(/\s*-\s*Tax$/i,'').trim() : l.label;
    if(!base) return;
    if(!groups[base]) groups[base]={base,price:0,gst:0,taxable:true};
    if(isTax) groups[base].gst+=l.amount;
    else { groups[base].price+=l.amount; if(/^redemptions$/i.test(l.eventType)) groups[base].taxable=false; }
  });
  return Object.values(groups)
    .map(g=>({description:g.base, priceExcl:g.price, gst:g.taxable?g.gst:'NA', taxable:g.taxable}))
    .sort((a,b)=>b.priceExcl-a.priceExcl);
}

/* ---------------- parse NetSuite invoice ---------------- */
function parseInvoice(raw, positioned){
  const text = String(raw).replace(/\r\n/g,'\n').replace(/\r/g,'\n');
  const r = {};
  r.invoiceNumber = (text.match(/Invoice #\s*[:#]?\s*([A-Za-z0-9_\-]+)/)||[])[1] || '';
  r.date          = (text.match(/(?:^|\n)\s*Date\s+(\d{1,2}\/\d{1,2}\/\d{4})/)||[])[1] || '';
  r.service       = ((text.match(/Service\s+(Ads[^\n]*|[^\n]*Promos[^\n]*|[^\n]+)/)||[])[1]||'').trim();
  r.total         = num((text.match(/Total\s*\(AUD\)\s*\$?([\d,]+\.\d{2})/)||[])[1]||'0');
  const bt = text.match(/Bill To\s*\n([\s\S]*?)\n\s*Product Type/);
  r.billTo = bt ? bt[1].split('\n').map(s=>s.trim()).filter(Boolean).join('\n') : '';
  const raws = (positioned && positioned.length) ? parseItemsTable(positioned) : parseTextRows(text);
  r.items = groupLines(raws);
  return r;
}

/* ---------------- date helpers ---------------- */
function serviceMonthYear(service){
  const m = service.match(/-\s*([A-Za-z]{3,})\.?\s+(\d{4})/);
  if(!m) return null;
  const key = m[1].slice(0,3).toLowerCase();
  const full = MONTHS[key] || (MFULL.find(x=>x.toLowerCase().startsWith(m[1].toLowerCase())));
  return full ? {month:full, year:+m[2]} : null;
}
function expandService(service){
  return service.replace(/-\s*([A-Za-z]{3})\.?\s+(\d{4})/, (whole,mon,yr)=>{
    const f = MONTHS[mon.toLowerCase()]; return f ? `- ${f} ${yr}` : whole;
  });
}
function lastDay(monthName, year){
  const idx = MFULL.indexOf(monthName);
  return new Date(year, idx+1, 0).getDate();
}
function defaultInvoiceDate(dateStr){ // dateStr = M/D/YYYY (NetSuite). → 6th of next month.
  const p = dateStr.split('/'); if(p.length!==3) return '';
  let mo = +p[0]-1, yr = +p[2];        // 0-indexed month
  mo += 1; if(mo>11){mo=0;yr++;}
  return `${MFULL[mo]} 6, ${yr}`;
}

/* ---------------- apply parsed data to form ---------------- */
function applyParsed(r){
  if(r.invoiceNumber) setVal('f_num', r.invoiceNumber + '_Tax');
  if(r.date) setVal('f_date', defaultInvoiceDate(r.date));
  if(r.service) setVal('f_service', expandService(r.service));
  const my = serviceMonthYear(r.service||'');
  if(my){ setVal('f_pstart', `${my.month} 1, ${my.year}`); setVal('f_pend', `${my.month} ${lastDay(my.month,my.year)}, ${my.year}`); }
  if(r.billTo) setVal('f_merchant', r.billTo);
  if(r.total) setVal('f_nstotal', money(r.total));
  items = (r.items||[]).map(i=>({...i}));
  renderRows();
  render();
}
function setVal(id,v){ document.getElementById(id).value = v; }

/* ---------------- line item rows ---------------- */
function renderRows(){
  const tb = document.getElementById('rows'); tb.innerHTML='';
  items.forEach((it,i)=>{
    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td><input class="desc" value="${escAttr(it.description)}" data-i="${i}" data-k="description"></td>
      <td><input value="${it.priceExcl!==''?money(it.priceExcl):''}" data-i="${i}" data-k="priceExcl"></td>
      <td><input value="${it.gst==='NA'?'NA':(it.gst!==''?money(it.gst):'')}" data-i="${i}" data-k="gst" ${it.taxable?'':'disabled'}></td>
      <td class="incl">${inclOf(it)}</td>
      <td class="taxchk"><input type="checkbox" data-i="${i}" data-k="taxable" ${it.taxable?'checked':''}></td>
      <td><button class="rm" data-rm="${i}" title="Remove">×</button></td>`;
    tb.appendChild(tr);
  });
}
function inclOf(it){
  const p = num(it.priceExcl);
  const g = it.taxable ? num(it.gst) : 0;
  return money(p+g);
}
function escAttr(s){ return String(s).replace(/"/g,'&quot;'); }

document.getElementById('rows').addEventListener('input', e=>{
  const i=+e.target.dataset.i, k=e.target.dataset.k; if(k==null) return;
  if(k==='taxable') return;
  if(k==='priceExcl'||k==='gst') items[i][k] = e.target.value.trim()==='' ? '' : (/^na$/i.test(e.target.value.trim()) ? 'NA' : num(e.target.value));
  else items[i][k] = e.target.value;
  // live incl update for this row only
  e.target.closest('tr').querySelector('.incl').textContent = inclOf(items[i]);
  render();
});
document.getElementById('rows').addEventListener('change', e=>{
  if(e.target.dataset.k==='taxable'){
    const i=+e.target.dataset.i;
    items[i].taxable = e.target.checked;
    if(!e.target.checked) items[i].gst='NA'; else if(items[i].gst==='NA') items[i].gst=0;
    renderRows(); render();
  }
});
document.getElementById('rows').addEventListener('click', e=>{
  if(e.target.dataset.rm!=null){ items.splice(+e.target.dataset.rm,1); renderRows(); render(); }
});
document.getElementById('addItem').addEventListener('click', ()=>{
  items.push({description:'',priceExcl:'',gst:0,taxable:true}); renderRows(); render();
});

/* ---------------- render preview + meter ---------------- */
function txt(id,v){ document.getElementById(id).textContent = v; }
function render(){
  txt('p_date', gv('f_date')||'—');
  txt('p_num', gv('f_num')||'—');
  txt('p_pstart', gv('f_pstart')||'—');
  txt('p_pend', gv('f_pend')||'—');
  txt('p_service', gv('f_service')||'—');
  txt('p_currency', gv('f_currency')||'—');
  txt('p_po', gv('f_po')||'');
  const merch = gv('f_merchant');
  document.getElementById('p_merchant').innerHTML = merch ? escHtml(merch) : '<span class="empty-note">—</span>';
  const abn = gv('f_abn');
  document.getElementById('p_abn').textContent = abn ? ('ABN: '+abn) : '';
  document.getElementById('stageFile').textContent = document.getElementById('goNote') && gv('f_num') ? gv('f_num')+'.pdf' : 'tax_invoice.pdf';

  // rows
  const pr = document.getElementById('p_rows'); pr.innerHTML='';
  let total = 0;
  items.forEach(it=>{
    const p = num(it.priceExcl), g = it.taxable?num(it.gst):0, incl = p+g;
    if(it.description||it.priceExcl!=='') total += incl;
    const tr = document.createElement('tr');
    tr.innerHTML = `<td>${escHtml(it.description||'')}</td><td>${it.priceExcl!==''?money(p):''}</td><td>${it.taxable?money(g):'NA'}</td><td>${money(incl)}</td>`;
    pr.appendChild(tr);
  });
  txt('p_total', items.length? money(total) : '—');

  // variance meter
  const ns = num(gv('f_nstotal'));
  txt('m_computed', money(total));
  const variance = Math.round((ns-total)*100)/100;
  const st = document.getElementById('m_status');
  const hasData = items.length>0 && ns>0;
  if(!hasData){ st.className='meter-status bad'; txt('m_label','Awaiting invoice'); txt('m_var','—'); }
  else if(variance===0){ st.className='meter-status ok'; txt('m_label','Reconciled — zero variance'); txt('m_var','$0.00'); }
  else { st.className='meter-status bad'; txt('m_label','Variance — does not match'); txt('m_var',(variance>0?'+':'−')+'$'+money(Math.abs(variance))); }

  // enable download
  const ready = hasData && variance===0 && gv('f_abn').trim()!=='';
  const btn = document.getElementById('download'); btn.disabled = !ready;
  const note = document.getElementById('goNote');
  if(ready) note.textContent = 'Ready — matches NetSuite to the cent';
  else if(!items.length) note.textContent = 'Load an invoice to begin';
  else if(gv('f_abn').trim()==='') note.textContent = 'Enter the merchant ABN to enable';
  else if(variance!==0) note.textContent = 'Total must match NetSuite before download';
}
function gv(id){ return document.getElementById(id).value.trim(); }
function escHtml(s){ return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }

// re-render on any field edit
['f_date','f_num','f_pstart','f_pend','f_service','f_currency','f_po','f_merchant','f_abn','f_nstotal']
  .forEach(id=>document.getElementById(id).addEventListener('input', render));

/* ---------------- file handling ---------------- */
const drop = document.getElementById('drop'), fileInput = document.getElementById('file');
fileInput.addEventListener('change', e=>{ if(e.target.files[0]) handleFile(e.target.files[0]); });
document.getElementById('replaceFile').addEventListener('click', e=>{ e.preventDefault(); e.stopPropagation(); fileInput.value=''; fileInput.click(); });
['dragover','dragenter'].forEach(ev=>drop.addEventListener(ev,e=>{e.preventDefault();drop.classList.add('hot');}));
['dragleave','drop'].forEach(ev=>drop.addEventListener(ev,e=>{e.preventDefault();drop.classList.remove('hot');}));
drop.addEventListener('drop', e=>{ const f=e.dataTransfer.files[0]; if(f) handleFile(f); });

async function handleFile(file){
  try{
    const buf = await file.arrayBuffer();
    const {text, items:pItems} = await extractPdf(buf);
    if(!/Product Type|Ads and Promos/i.test(text) && !/Invoice #/i.test(text)){
      alert("Couldn't find invoice text in that PDF. If it's a scan, use “Paste the invoice text instead”.");
    }
    const r = parseInvoice(text, pItems);
    applyParsed(r);
    document.getElementById('loadedName').textContent = file.name;
    document.getElementById('loaded').classList.add('show');
  }catch(err){
    console.error(err);
    alert("Couldn't read that PDF. Try the paste-text option below.");
  }
}

/* paste fallback */
const pasteToggle=document.getElementById('pasteToggle'), pasteBox=document.getElementById('pasteBox');
pasteToggle.addEventListener('click',()=>pasteBox.classList.toggle('show'));
pasteBox.addEventListener('change',()=>{ if(pasteBox.value.trim()){ applyParsed(parseInvoice(pasteBox.value)); } });

/* ---------------- download PDF ---------------- */
document.getElementById('download').addEventListener('click', async ()=>{
  const el = document.getElementById('invoice');
  const name = (gv('f_num')||'tax_invoice') + '.pdf';
  const btn = document.getElementById('download'); const note = document.getElementById('goNote');
  try{
    if(document.fonts && document.fonts.ready) await document.fonts.ready;
    if(typeof html2pdf === 'undefined') throw new Error('pdf-lib-unavailable');
    btn.textContent = 'Generating…';
    await html2pdf().set({
      margin:0, filename:name,
      image:{type:'jpeg',quality:0.98},
      html2canvas:{scale:2.5, useCORS:true, backgroundColor:'#ffffff'},
      jsPDF:{unit:'px', format:[794,1123], orientation:'portrait'}
    }).from(el).save();
    btn.textContent = 'Download tax invoice PDF';
    note.textContent = 'If nothing downloaded, use “Print / Save as PDF”.';
  }catch(err){
    console.error(err);
    btn.textContent = 'Download tax invoice PDF';
    note.textContent = 'Download blocked here — opening print dialog instead…';
    window.print();
  }
});
document.getElementById('printBtn').addEventListener('click', ()=>window.print());

render();
</script>
</body>
</html>
