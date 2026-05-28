import { useState } from "react";

// ─── REAL USPS FCPIS RATES FOR BRAZIL (retail, effective July 13 2025) ────────
// Source: ParcelPath / USPS Notice 123
// Weight brackets in ounces → price in USD
const FCPIS_BRAZIL = [
  { maxOz: 8,  price: 23.14 },
  { maxOz: 32, price: 35.51 },
  { maxOz: 48, price: 53.58 },
  { maxOz: 64, price: 62.93 },
];

// Priority Mail International Brazil (weight-based, retail approx)
// USPS groups Brazil separately; approximate retail rates
const PMI_BRAZIL = [
  { maxLbs: 1,  price: 40.50 },
  { maxLbs: 2,  price: 50.25 },
  { maxLbs: 3,  price: 59.80 },
  { maxLbs: 4,  price: 69.00 },
  { maxLbs: 5,  price: 78.50 },
  { maxLbs: 10, price: 115.00 },
  { maxLbs: 20, price: 185.00 },
  { maxLbs: 70, price: 390.00 },
];

// Priority Mail Express International Brazil — starts at $62.70
const PMEI_BRAZIL = [
  { maxLbs: 1,  price: 62.70 },
  { maxLbs: 2,  price: 72.00 },
  { maxLbs: 3,  price: 82.50 },
  { maxLbs: 4,  price: 93.00 },
  { maxLbs: 5,  price: 104.00 },
  { maxLbs: 10, price: 155.00 },
  { maxLbs: 20, price: 250.00 },
  { maxLbs: 70, price: 520.00 },
];

// USPS dimensional weight divisor for international = 166 (in³ ÷ 166 = lbs)
const DIM_DIVISOR = 166;

function calcDimWeight(l, w, h) {
  if (!l || !w || !h) return 0;
  return (l * w * h) / DIM_DIVISOR;
}

function getBillableWeight(actualLbs, dimWeightLbs) {
  return Math.max(actualLbs, dimWeightLbs);
}

function lookupFCPIS(oz) {
  if (oz > 64) return null; // over 4 lbs — not eligible
  for (const tier of FCPIS_BRAZIL) {
    if (oz <= tier.maxOz) return tier.price;
  }
  return null;
}

function lookupPMI(lbs) {
  for (const tier of PMI_BRAZIL) {
    if (lbs <= tier.maxLbs) return tier.price;
  }
  return PMI_BRAZIL[PMI_BRAZIL.length - 1].price;
}

function lookupPMEI(lbs) {
  for (const tier of PMEI_BRAZIL) {
    if (lbs <= tier.maxLbs) return tier.price;
  }
  return PMEI_BRAZIL[PMEI_BRAZIL.length - 1].price;
}


const IMPORT_THRESHOLD = 50;

function calcImportTax(val) {
  if (!val || val <= IMPORT_THRESHOLD) return 0;
  return (val - IMPORT_THRESHOLD) * 0.6;
}

const BRAZIL_STATES = [
  "Acre","Alagoas","Amapá","Amazonas","Bahia","Ceará","Distrito Federal",
  "Espírito Santo","Goiás","Maranhão","Mato Grosso","Mato Grosso do Sul",
  "Minas Gerais","Pará","Paraíba","Paraná","Pernambuco","Piauí",
  "Rio de Janeiro","Rio Grande do Norte","Rio Grande do Sul","Rondônia",
  "Roraima","Santa Catarina","São Paulo","Sergipe","Tocantins"
];

const UNIT_SYSTEMS = { imperial: "lbs / in", metric: "kg / cm" };

export default function App() {
  const [unit, setUnit] = useState("imperial");
  const [weight, setWeight] = useState("");
  const [length, setLength] = useState("");
  const [width, setWidth]   = useState("");
  const [height, setHeight] = useState("");
  const [state, setState]   = useState("");
  const [declared, setDeclared] = useState("");
  const [results, setResults]   = useState(null);

  const wLabel = unit === "imperial" ? "lbs" : "kg";
  const dLabel = unit === "imperial" ? "in" : "cm";

  function toImperial() {
    const w = parseFloat(weight) || 0;
    const l = parseFloat(length) || 0;
    const wi = parseFloat(width) || 0;
    const h = parseFloat(height) || 0;
    if (unit === "metric") {
      return {
        weightLbs: w * 2.20462,
        lengthIn:  l / 2.54,
        widthIn:   wi / 2.54,
        heightIn:  h / 2.54,
      };
    }
    return { weightLbs: w, lengthIn: l, widthIn: wi, heightIn: h };
  }

  function calculate() {
    const { weightLbs, lengthIn, widthIn, heightIn } = toImperial();
    if (!weightLbs || weightLbs <= 0) return;

    const actualOz  = weightLbs * 16;
    const dimLbs    = calcDimWeight(lengthIn, widthIn, heightIn);
    const dimOz     = dimLbs * 16;
    const billLbs   = getBillableWeight(weightLbs, dimLbs);
    const billOz    = billLbs * 16;
    const dimApplied = dimLbs > weightLbs && (lengthIn && widthIn && heightIn);

    const fcpisPrice = lookupFCPIS(billOz); // null if >64 oz
    const pmiPrice   = lookupPMI(billLbs);
    const pmeiPrice  = lookupPMEI(billLbs);

    const importTax  = calcImportTax(parseFloat(declared) || 0);
    const decVal     = parseFloat(declared) || 0;

    function buildService(name, basePrice, days, eligible, note) {
      if (!eligible) return null;
      const total = basePrice + importTax;
      return { name, basePrice, importTax, total, days, note };
    }

    setResults({
      weightLbs: weightLbs.toFixed(2),
      dimLbs: dimLbs > 0 ? dimLbs.toFixed(2) : null,
      billLbs: billLbs.toFixed(2),
      dimApplied,
      actualOz: actualOz.toFixed(1),
      billOz: billOz.toFixed(1),
      decVal,
      importTax,
      services: [
        buildService(
          "First-Class Package Intl",
          fcpisPrice,
          "2–4 weeks",
          fcpisPrice !== null,
          billOz > 64 ? "Not eligible — package exceeds 4 lbs (64 oz)" : "Max 4 lbs · Under $400 value · Most economical"
        ),
        buildService("Priority Mail International", pmiPrice, "6–10 business days",  true, "Up to 70 lbs · Includes $200 insurance"),
        buildService("Priority Mail Express Intl",  pmeiPrice, "3–5 business days", true, "Fastest · Money-back guarantee · Starts at $62.70"),
      ].filter(Boolean),
    });
  }

  function reset() {
    setWeight(""); setLength(""); setWidth(""); setHeight("");
    setState(""); setDeclared(""); setResults(null);
  }

  const canCalc = (parseFloat(weight) || 0) > 0;

  return (
    <div style={{
      minHeight:"100vh", background:"#009C3B",
      fontFamily:"'DM Serif Display','Georgia',serif",
      display:"flex", alignItems:"center", justifyContent:"center",
      padding:"28px 16px",
    }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=DM+Serif+Display&family=IBM+Plex+Sans:wght@300;400;500;600&display=swap');
        *{box-sizing:border-box;margin:0;padding:0}
        body{background:#009C3B}
        .sans{font-family:'IBM Plex Sans','Segoe UI',sans-serif}
        .field{width:100%;background:#fff;border:1.5px solid #FFDF00;border-radius:8px;
               padding:11px 13px;font-size:14px;font-family:'IBM Plex Sans',sans-serif;
               color:#002776;transition:border-color .18s;outline:none}
        .field:focus{border-color:#002776}
        .field option{background:#fff}
        .field::-webkit-outer-spin-button,.field::-webkit-inner-spin-button{-webkit-appearance:none}
        .pill{display:inline-flex;border:2px solid #FFDF00;border-radius:20px;overflow:hidden}
        .pill-btn{padding:5px 14px;font-size:12px;font-family:'IBM Plex Sans',sans-serif;
                  font-weight:500;cursor:pointer;border:none;transition:background .15s,color .15s}
        .pill-btn.active{background:#002776;color:#FFDF00}
        .pill-btn:not(.active){background:#fff;color:#002776}
        .go-btn{width:100%;background:#FFDF00;border:none;border-radius:8px;padding:14px;
                color:#002776;font-family:'IBM Plex Sans',sans-serif;font-weight:700;font-size:14px;
                letter-spacing:.03em;cursor:pointer;transition:background .15s,transform .12s}
        .go-btn:hover:not(:disabled){background:#e5c800;transform:translateY(-1px)}
        .go-btn:disabled{background:#d4c96a;color:#8a8a50;cursor:not-allowed}
        .reset-btn{width:100%;background:transparent;border:1.5px solid #FFDF00;border-radius:8px;
                   padding:11px;font-family:'IBM Plex Sans',sans-serif;font-size:13px;color:#009C3B;
                   cursor:pointer;margin-top:10px;transition:border-color .15s,color .15s}
        .reset-btn:hover{border-color:#fff;color:#002776}
        .svc-card{background:#fff;border:2px solid #FFDF00;border-radius:10px;
                  padding:16px;margin-bottom:12px;transition:border-color .2s,box-shadow .2s}
        .svc-card:hover{border-color:#002776;box-shadow:0 2px 12px rgba(0,39,118,.15)}
        .tag{display:inline-block;padding:2px 9px;border-radius:12px;font-size:11px;
             font-family:'IBM Plex Sans',sans-serif;font-weight:600}
        .dim-notice{background:#fffbe6;border:1px solid #FFDF00;border-radius:7px;
                    padding:10px 13px;font-size:12px;font-family:'IBM Plex Sans',sans-serif;
                    color:#002776;margin-bottom:14px;line-height:1.6}
        .row{display:flex;justify-content:space-between;align-items:center;
             padding:7px 0;border-bottom:1px solid #e8f5ee}
        .row:last-child{border-bottom:none}
        .ineligible{opacity:.5;pointer-events:none;filter:grayscale(30%)}
        .ineligible-banner{font-size:11px;color:#c0392b;font-family:'IBM Plex Sans',sans-serif;
                           margin-top:4px;font-style:italic}
      `}</style>

      <div style={{width:"100%",maxWidth:"430px"}}>

        {/* Header */}
        <div style={{marginBottom:"24px"}}>
          <h1 style={{fontSize:"26px",color:"#FFDF00",letterSpacing:"-0.02em",textShadow:"0 1px 4px rgba(0,0,0,.2)",marginBottom:"8px"}}>Shipping Calculator</h1>
          <div style={{background:"rgba(0,0,0,.2)",border:"1px solid #FFDF00",borderRadius:"8px",padding:"10px 14px"}}>
            <p className="sans" style={{fontSize:"12px",color:"#FFDF00",fontWeight:600,marginBottom:"2px"}}>⚠️ Estimate Only</p>
            <p className="sans" style={{fontSize:"12px",color:"#d4f5e0",lineHeight:"1.5"}}>
              This is only an estimate. The actual shipping quote will be shared with you once the item is purchased.
            </p>
          </div>
        </div>

        {/* Main card */}
        <div style={{background:"#fff",border:"2px solid #FFDF00",borderRadius:"14px",padding:"24px"}}>

          {/* Unit toggle */}
          <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:"20px"}}>
            <span className="sans" style={{fontSize:"12px",fontWeight:600,color:"#002776",letterSpacing:".05em",textTransform:"uppercase"}}>Units</span>
            <div className="pill">
              <button className={`pill-btn${unit==="imperial"?" active":""}`} onClick={()=>setUnit("imperial")}>lbs / in</button>
              <button className={`pill-btn${unit==="metric"?" active":""}`}   onClick={()=>setUnit("metric")}>kg / cm</button>
            </div>
          </div>

          {/* Weight */}
          <div style={{marginBottom:"14px"}}>
            <label className="sans" style={{display:"block",fontSize:"11px",fontWeight:600,color:"#002776",letterSpacing:".06em",textTransform:"uppercase",marginBottom:"7px"}}>
              Actual Weight ({wLabel})
            </label>
            <div style={{position:"relative"}}>
              <input className="field" type="number" min="0" step="0.1"
                     placeholder={unit==="imperial"?"e.g. 1.0":"e.g. 0.45"}
                     value={weight} onChange={e=>{setWeight(e.target.value);setResults(null)}}
                     style={{paddingRight:"46px"}} />
              <span className="sans" style={{position:"absolute",right:"13px",top:"50%",transform:"translateY(-50%)",fontSize:"12px",fontWeight:600,color:"#009C3B"}}>{wLabel}</span>
            </div>
          </div>

          {/* Dimensions */}
          <div style={{marginBottom:"14px"}}>
            <label className="sans" style={{display:"block",fontSize:"11px",fontWeight:600,color:"#002776",letterSpacing:".06em",textTransform:"uppercase",marginBottom:"7px"}}>
              Package Dimensions ({dLabel}) <span style={{color:"#009C3B",fontWeight:400,textTransform:"none"}}>— for dimensional weight</span>
            </label>
            <div style={{display:"grid",gridTemplateColumns:"1fr 1fr 1fr",gap:"8px"}}>
              {[["L",length,setLength],["W",width,setWidth],["H",height,setHeight]].map(([lbl,val,set])=>(
                <div key={lbl} style={{position:"relative"}}>
                  <input className="field" type="number" min="0" step="0.1"
                         placeholder={lbl} value={val}
                         onChange={e=>{set(e.target.value);setResults(null)}}
                         style={{paddingLeft:"28px"}} />
                  <span className="sans" style={{position:"absolute",left:"10px",top:"50%",transform:"translateY(-50%)",fontSize:"11px",fontWeight:600,color:"#009C3B"}}>{lbl}</span>
                </div>
              ))}
            </div>
            <p className="sans" style={{fontSize:"11px",color:"#009C3B",marginTop:"5px"}}>
              USPS uses L×W×H ÷ 166 for dim weight · Billable = whichever is greater
            </p>
          </div>

          {/* State */}
          <div style={{marginBottom:"14px"}}>
            <label className="sans" style={{display:"block",fontSize:"11px",fontWeight:600,color:"#002776",letterSpacing:".06em",textTransform:"uppercase",marginBottom:"7px"}}>
              Destination State
            </label>
            <select className="field" value={state} onChange={e=>{setState(e.target.value);setResults(null)}}>
              <option value="">Select a state…</option>
              {BRAZIL_STATES.map(s=><option key={s}>{s}</option>)}
            </select>
            <p className="sans" style={{fontSize:"11px",color:"#009C3B",marginTop:"5px"}}>
              USPS treats all of Brazil as one pricing group
            </p>
          </div>

          {/* Declared value */}
          <div style={{marginBottom:"20px"}}>
            <label className="sans" style={{display:"block",fontSize:"11px",fontWeight:600,color:"#002776",letterSpacing:".06em",textTransform:"uppercase",marginBottom:"7px"}}>
              Declared Value (USD) <span style={{color:"#009C3B",fontWeight:400,textTransform:"none"}}>— for import tax estimate</span>
            </label>
            <div style={{position:"relative"}}>
              <input className="field" type="number" min="0" step="1"
                     placeholder="e.g. 120" value={declared}
                     onChange={e=>{setDeclared(e.target.value);setResults(null)}}
                     style={{paddingLeft:"28px"}} />
              <span className="sans" style={{position:"absolute",left:"12px",top:"50%",transform:"translateY(-50%)",fontSize:"13px",fontWeight:600,color:"#009C3B"}}>$</span>
            </div>
          </div>

          <button className="go-btn" disabled={!canCalc} onClick={calculate}>
            CALCULATE SHIPPING
          </button>

          {/* ── RESULTS ── */}
          {results && (
            <div style={{marginTop:"24px"}}>
              <div style={{height:"1px",background:"linear-gradient(to right,transparent,#e8e4de,transparent)",marginBottom:"20px"}} />

              {/* Weight summary */}
              {results.dimApplied && (
                <div className="dim-notice">
                  ⚠️ <strong>Dimensional weight applies.</strong><br/>
                  Actual: {results.weightLbs} lbs &nbsp;·&nbsp; Dim weight: {results.dimLbs} lbs<br/>
                  <strong>Billable weight: {results.billLbs} lbs ({results.billOz} oz)</strong>
                </div>
              )}
              {!results.dimApplied && results.dimLbs && (
                <div className="dim-notice" style={{background:"#f0f8f0",borderColor:"#b0d8b0",color:"#1a5c1a"}}>
                  ✓ Actual weight ({results.weightLbs} lbs) is greater than dim weight ({results.dimLbs} lbs). No surcharge.
                </div>
              )}

              {/* Import tax note */}
              {results.decVal > 0 && (
                <div className="dim-notice" style={{
                  background: results.importTax > 0 ? "#fff8f0":"#f0f8f0",
                  borderColor: results.importTax > 0 ? "#f0d0a0":"#b0d8b0",
                  color: results.importTax > 0 ? "#7a4400":"#1a5c1a",
                  marginBottom:"14px"
                }}>
                  {results.importTax > 0
                    ? `🏛️ Brazil import tax: 60% on $${(results.decVal - 50).toFixed(0)} above the $50 threshold = $${results.importTax.toFixed(2)}`
                    : "✓ Declared value under $50 — no Brazilian import tax"}
                </div>
              )}

              <p className="sans" style={{fontSize:"11px",fontWeight:600,color:"#002776",letterSpacing:".07em",textTransform:"uppercase",marginBottom:"12px"}}>
                Shipping Options
              </p>

              {results.services.map((svc, i) => {
                const eligible = svc.basePrice !== null;
                return (
                  <div key={i} className={`svc-card${!eligible?" ineligible":""}`}>
                    <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start",marginBottom:"8px"}}>
                      <div>
                        <div className="sans" style={{fontSize:"14px",fontWeight:600,color:"#002776"}}>{svc.name}</div>
                        <div className="sans" style={{fontSize:"11px",color:"#009C3B",marginTop:"2px"}}>{svc.days}</div>
                        {!eligible && <div className="ineligible-banner">{svc.note}</div>}
                      </div>
                      <div style={{textAlign:"right"}}>
                        <div style={{fontSize:"20px",color:"#009C3B",fontWeight:"bold",letterSpacing:"-0.02em"}}>
                          ${svc.total.toFixed(2)}
                        </div>
                        <div className="sans" style={{fontSize:"10px",color:"#009C3B"}}>total est.</div>
                      </div>
                    </div>

                    {eligible && (
                      <div style={{borderTop:"1px solid #e8f5ee",paddingTop:"8px"}}>
                        <div className="row">
                          <span className="sans" style={{fontSize:"12px",color:"#009C3B"}}>USPS shipping</span>
                          <span className="sans" style={{fontSize:"12px",fontWeight:500,color:"#002776"}}>${svc.basePrice.toFixed(2)}</span>
                        </div>
                        {svc.importTax > 0 && (
                          <div className="row">
                            <span className="sans" style={{fontSize:"12px",color:"#009C3B"}}>Import tax (est.)</span>
                            <span className="sans" style={{fontSize:"12px",fontWeight:500,color:"#FFDF00",background:"#002776",padding:"1px 7px",borderRadius:"4px"}}>${svc.importTax.toFixed(2)}</span>
                          </div>
                        )}
                        <p className="sans" style={{fontSize:"11px",color:"#009C3B",marginTop:"7px",fontStyle:"italic"}}>{svc.note}</p>
                      </div>
                    )}
                  </div>
                );
              })}

              <p className="sans" style={{fontSize:"11px",color:"#009C3B",marginTop:"8px",textAlign:"center",lineHeight:"1.6"}}>
                Retail USPS rates · Service fee is your charge · Import tax subject to Brazilian customs discretion
              </p>
            </div>
          )}

          <button className="reset-btn" onClick={reset}>Reset</button>
        </div>

        <p className="sans" style={{fontSize:"11px",color:"#d4f5e0",textAlign:"center",marginTop:"14px"}}>
          USPS rates effective July 13 2025 · Brazil Receita Federal §50 threshold
        </p>
      </div>
    </div>
  );
}
