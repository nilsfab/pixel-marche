import React, { useState, useRef, useEffect, useCallback } from "react";

// ─────────────────────────────────────────────
//  CONSTANTS
// ─────────────────────────────────────────────
const GRID = 1000;
const MIN_ZOOM = 0.12;
const MAX_ZOOM = 50;

const PALETTE = [
  "#e63946","#f4a261","#ffd166","#2a9d8f","#06d6a0",
  "#118ab2","#6a4c93","#ef476f","#8ac926","#ffbe0b",
  "#457b9d","#ffffff","#888888","#222222",
];

const SIZES = [
  { n: 1,  label: "1 × 1",   price: 1   },
  { n: 5,  label: "5 × 5",   price: 25  },
  { n: 10, label: "10 × 10", price: 100 },
];

const PAY_METHODS = [
  { id:"apple",  name:"Apple Pay",      icon:"🍎", bg:"#000000", fg:"#ffffff" },
  { id:"google", name:"Google Pay",     icon:"G",  bg:"#ffffff", fg:"#111111", border:"1px solid #dddddd" },
  { id:"paypal", name:"PayPal",         icon:"𝐏",  bg:"#003087", fg:"#ffffff" },
  { id:"card",   name:"Carte bancaire", icon:"💳", bg:"#1c1c2e", fg:"#ffffff", border:"1px solid rgba(255,255,255,0.15)" },
];

const BOMBS = [
  { id:"bomb",     name:"Bombe",      icon:"💣", radius:4, price:5,  color:"#FF6B35", desc:"Détruit une zone 4×4" },
  { id:"megabomb", name:"Méga-Bombe", icon:"☢️", radius:8, price:30, color:"#FF3366", desc:"Détruit une zone 8×8" },
];

const STREAK_REWARDS = [1, 2, 3, 5, 7, 10, 15];

const TUTO = [
  { emoji:"🎨", color:"#e8504a", title:"Bienvenue sur PixelMarché !", desc:"Une toile d'un million de pixels à posséder. Chaque pixel vous appartient pour toujours, visible par tous les joueurs du monde entier.", tip:null, visual:"grid" },
  { emoji:"👆", color:"#118ab2", title:"Tapez pour acheter", desc:"Touchez n'importe quelle case vide sur la grille. Une fiche s'ouvre pour choisir la taille, la couleur et le prix.", tip:"Pincez avec 2 doigts pour zoomer et trouver l'emplacement parfait !", visual:"tap" },
  { emoji:"📐", color:"#6a4c93", title:"3 tailles disponibles", desc:"Choisissez la taille selon votre budget. Plus le bloc est grand, plus il est visible !", tip:null, visual:"sizes" },
  { emoji:"🔗", color:"#2a9d8f", title:"Ajoutez votre lien", desc:"Chaque pixel peut pointer vers votre site ou réseau social. Les visiteurs cliquent directement dessus.", tip:"Tous les liens sont vérifiés par notre IA avant publication.", visual:"link" },
  { emoji:"🛒", color:"#f4a261", title:"La boutique", desc:"Pixels gratuits, enchères, bombes… Accédez à des pouvoirs spéciaux depuis le bouton 🛒 en haut à droite.", tip:null, visual:"shop" },
  { emoji:"🏆", color:"#ffd166", title:"Récompenses & Classement", desc:"Connectez-vous chaque jour pour gagner des pixels gratuits. Montez dans le classement mondial !", tip:"Jour 1 → 1 pixel · Jour 7 → 15 pixels gratuits !", visual:"rewards" },
];

// ─────────────────────────────────────────────
//  HELPERS
// ─────────────────────────────────────────────
function luhn(raw) {
  let s = 0;
  for (let i = 0; i < raw.length; i++) {
    let d = +raw[raw.length - 1 - i];
    if (i % 2 === 1) { d *= 2; if (d > 9) d -= 9; }
    s += d;
  }
  return s % 10 === 0;
}

function cardInfo(raw) {
  if (/^4/.test(raw))               return { name:"Visa",       icon:"💙", len:16, cvv:3 };
  if (/^5[1-5]|^2[2-7]/.test(raw)) return { name:"Mastercard", icon:"🔴", len:16, cvv:3 };
  if (/^3[47]/.test(raw))           return { name:"Amex",       icon:"💚", len:15, cvv:4 };
  if (/^6/.test(raw))               return { name:"CB",         icon:"🟠", len:16, cvv:3 };
  return null;
}

function validateCard(n, e, c) {
  const raw = n.replace(/\s/g, "");
  const errs = [];
  const info = cardInfo(raw);
  if (raw.length < 13)               errs.push("Numéro trop court");
  else if (!luhn(raw))               errs.push("Numéro invalide (Luhn)");
  if (info && raw.length !== info.len) errs.push(`${info.name} : ${info.len} chiffres requis`);
  if (/^(.)\1+$/.test(raw))          errs.push("Chiffres identiques détectés");
  if (e.length === 5) {
    const [mm, yy] = e.split("/").map(Number);
    if (mm < 1 || mm > 12) errs.push("Mois invalide");
    if (new Date(2000 + yy, mm) < new Date()) errs.push("Carte expirée");
  }
  if (info && c.length > 0 && c.length !== info.cvv) errs.push(`CVV : ${info.cvv} chiffres pour ${info.name}`);
  return { ok: errs.length === 0, errs, info };
}

async function moderateUrl(url) {
  try {
    const r = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "claude-sonnet-4-20250514",
        max_tokens: 100,
        system: "Modère les liens d'une plateforme de pixels. Refuse: adulte, phishing, malware, haine. Accepte: commercial, portfolio, blog. Réponds UNIQUEMENT en JSON sans markdown: {\"ok\":true,\"reason\":\"...\"}",
        messages: [{ role: "user", content: `URL: ${url}` }],
      }),
    });
    const d = await r.json();
    return JSON.parse((d.content || []).map(c => c.text || "").join("").trim());
  } catch (_) {
    return { ok: true, reason: "Non vérifié" };
  }
}

async function lbRead() {
  try {
    const r = await window.storage.get("pm_lb", true);
    if (r) return JSON.parse(r.value);
  } catch (_) {}
  return {};
}

async function lbWrite(name, count) {
  try {
    const cur = await lbRead();
    cur[name] = count;
    await window.storage.set("pm_lb", JSON.stringify(cur), true);
  } catch (_) {}
}

function today() { return new Date().toISOString().slice(0, 10); }

// ─────────────────────────────────────────────
//  GLOBAL CSS
// ─────────────────────────────────────────────
const CSS = `
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; -webkit-tap-highlight-color: transparent; }
  body { background: #080810; overflow: hidden; font-family: system-ui, sans-serif; }
  input { font-family: inherit; background: transparent; color: #fff; border: none; outline: none; }
  input::placeholder { color: rgba(255,255,255,0.25); }
  button { font-family: inherit; cursor: pointer; border: none; outline: none; }
  @keyframes fadeUp   { from { opacity:0; transform:translateY(12px) } to { opacity:1; transform:translateY(0) } }
  @keyframes slideUp  { from { transform:translateY(100%) } to { transform:translateY(0) } }
  @keyframes popIn    { from { opacity:0; transform:scale(0.88) } to { opacity:1; transform:scale(1) } }
  @keyframes spin     { to { transform:rotate(360deg) } }
  @keyframes toastIn  { from { opacity:0; transform:translateX(-50%) translateY(-10px) } to { opacity:1; transform:translateX(-50%) translateY(0) } }
  @keyframes boom     { 0% { transform:scale(1); opacity:1 } 60% { transform:scale(3.5); opacity:.5 } 100% { transform:scale(0); opacity:0 } }
  @keyframes rewardIn { 0% { transform:scale(0.5) rotate(-8deg); opacity:0 } 65% { transform:scale(1.1) rotate(2deg); opacity:1 } 100% { transform:scale(1) rotate(0deg); opacity:1 } }
  @keyframes tutoIn   { from { opacity:0; transform:translateX(28px) } to { opacity:1; transform:translateX(0) } }
  @keyframes tutoOut  { from { opacity:1; transform:translateX(0) } to { opacity:0; transform:translateX(-28px) } }
  @keyframes floatUp  { 0%,100% { transform:translateY(0) } 50% { transform:translateY(-6px) } }
`;

// ─────────────────────────────────────────────
//  LOGOS
// ─────────────────────────────────────────────
function LogoBanner() {
  const rows = 19, cols = 14, cw = 14;
  return (
    <svg viewBox="0 0 320 220" style={{ maxWidth:"100%", height:60 }}>
      <defs>
        <filter id="lshadow">
          <feDropShadow dx="2" dy="3" stdDeviation="5" floodColor="rgba(0,0,0,0.4)" />
        </filter>
      </defs>
      <rect x="4" y="4" width="200" height="212" rx="6" fill="#f0eeeb" filter="url(#lshadow)" />
      <rect x="12" y="12" width="184" height="196" fill="#e8504a" />
      {Array.from({ length: rows }, (_, i) => (
        <line key={"h" + i} x1="12" y1={12 + i * cw} x2="196" y2={12 + i * cw} stroke="#c0392b" strokeWidth="0.6" opacity="0.5" />
      ))}
      {Array.from({ length: cols }, (_, i) => (
        <line key={"v" + i} x1={12 + i * cw} y1="12" x2={12 + i * cw} y2="208" stroke="#c0392b" strokeWidth="0.6" opacity="0.5" />
      ))}
      <rect x="68" y="54" width="110" height="140" fill="#222" />
      <text x="214" y="68"  fontFamily="Georgia,serif" fontSize="34" fontWeight="700" fill="#e8504a" letterSpacing="-1">Pixel</text>
      <text x="215" y="106" fontFamily="Georgia,serif" fontSize="30" fontWeight="400" fontStyle="italic" fill="#fff" opacity="0.9">Marché</text>
      <line x1="214" y1="114" x2="314" y2="114" stroke="#e8504a" strokeWidth="1.5" opacity="0.3" />
      <text x="215" y="132" fontFamily="'Courier New',monospace" fontSize="7" letterSpacing="2" fill="rgba(255,255,255,0.3)">UN MILLION DE PIXELS</text>
    </svg>
  );
}

function LogoMark({ size = 28 }) {
  const c = 8.8;
  return (
    <svg width={size} height={size} viewBox="0 0 100 100">
      <rect width="100" height="100" rx="18" fill="#f0eeeb" />
      <rect x="6" y="6" width="88" height="88" fill="#e8504a" />
      {Array.from({ length: 11 }, (_, i) => (
        <line key={"h" + i} x1="6" y1={6 + i * c} x2="94" y2={6 + i * c} stroke="#c0392b" strokeWidth="0.6" opacity="0.5" />
      ))}
      {Array.from({ length: 11 }, (_, i) => (
        <line key={"v" + i} x1={6 + i * c} y1="6" x2={6 + i * c} y2="94" stroke="#c0392b" strokeWidth="0.6" opacity="0.5" />
      ))}
      <rect x={6 + 2 * c} y={6 + 2 * c} width={5 * c} height={6 * c} fill="#222" />
    </svg>
  );
}

// ─────────────────────────────────────────────
//  ROOT
// ─────────────────────────────────────────────
export default function App() {
  const [screen,  setScreen]  = useState("loading");
  const [profile, setProfile] = useState(null);
  const [streak,  setStreak]  = useState(null);
  const [reward,  setReward]  = useState(null);
  const [tuto,    setTuto]    = useState(false);

  useEffect(() => {
    (async () => {
      try {
        const rp = await window.storage.get("pm10_profile");
        if (rp) {
          const p = JSON.parse(rp.value);
          let s = { day: 1, last: today(), tokens: STREAK_REWARDS[0] };
          try {
            const rs = await window.storage.get("pm10_streak");
            if (rs) {
              s = JSON.parse(rs.value);
              const t = today();
              if (s.last !== t) {
                const diff = Math.round((new Date(t) - new Date(s.last)) / 864e5);
                s.day = diff === 1 ? Math.min(s.day + 1, STREAK_REWARDS.length) : 1;
                const earned = STREAK_REWARDS[s.day - 1];
                s.tokens = (s.tokens || 0) + earned;
                s.last = t;
                await window.storage.set("pm10_streak", JSON.stringify(s));
                setReward({ day: s.day, earned, tokens: s.tokens });
              }
            }
          } catch (_) {}
          setStreak(s);
          setProfile(p);
          setScreen("grid");
          return;
        }
      } catch (_) {}
      setScreen("onboarding");
    })();
  }, []);

  const doSave = async (p) => {
    try { await window.storage.set("pm10_profile", JSON.stringify(p)); } catch (_) {}
    const s = { day: 1, last: today(), tokens: STREAK_REWARDS[0] };
    try { await window.storage.set("pm10_streak", JSON.stringify(s)); } catch (_) {}
    setStreak(s);
    setReward({ day: 1, earned: STREAK_REWARDS[0], tokens: STREAK_REWARDS[0] });
    setTuto(true);
    setProfile(p);
    setScreen("grid");
  };

  const doLogout = async () => {
    try { await window.storage.delete("pm10_profile"); } catch (_) {}
    setProfile(null); setStreak(null); setScreen("onboarding");
  };

  const doTokenUpdate = async (n) => {
    if (!streak) return;
    const ns = { ...streak, tokens: n };
    setStreak(ns);
    try { await window.storage.set("pm10_streak", JSON.stringify(ns)); } catch (_) {}
  };

  return (
    <React.Fragment>
      <style>{CSS}</style>
      {screen === "loading"    && <Loading />}
      {screen === "onboarding" && <Onboarding onDone={doSave} />}
      {screen === "grid"       && (
        <Grid
          profile={profile}
          streak={streak}
          onLogout={doLogout}
          onTokenUpdate={doTokenUpdate}
          onTuto={() => setTuto(true)}
        />
      )}
      {reward && <DailyReward data={reward} onClose={() => setReward(null)} />}
      {tuto   && <Tutorial onClose={() => setTuto(false)} />}
    </React.Fragment>
  );
}

function Loading() {
  return (
    <div style={{ width:"100vw", height:"100vh", background:"#080810", display:"flex", alignItems:"center", justifyContent:"center" }}>
      <Spinner />
    </div>
  );
}

// ─────────────────────────────────────────────
//  TUTORIAL
// ─────────────────────────────────────────────
function Tutorial({ onClose }) {
  const [step, setStep] = useState(0);
  const [anim, setAnim] = useState("in");
  const cur = TUTO[step];
  const last = step === TUTO.length - 1;

  const go = (n) => {
    setAnim("out");
    setTimeout(() => { setStep(n); setAnim("in"); }, 180);
  };

  return (
    <div style={{ position:"fixed", inset:0, zIndex:500, background:"rgba(4,4,16,0.93)", backdropFilter:"blur(8px)", display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", padding:"20px 18px" }}>
      <button onClick={onClose} style={{ position:"absolute", top:16, right:16, background:"rgba(255,255,255,0.07)", border:"1px solid rgba(255,255,255,0.12)", borderRadius:20, padding:"6px 15px", color:"rgba(255,255,255,0.45)", fontSize:12, fontWeight:600 }}>
        Passer
      </button>

      <div style={{ display:"flex", gap:6, marginBottom:22 }}>
        {TUTO.map((_, i) => (
          <button key={i} onClick={() => go(i)} style={{ width: i === step ? 22 : 7, height:7, borderRadius:4, background: i === step ? cur.color : "rgba(255,255,255,0.2)", border:"none", padding:0, transition:"all 0.25s", cursor:"pointer" }} />
        ))}
      </div>

      <div style={{ background:"#12121e", border:`1px solid ${cur.color}44`, borderRadius:22, padding:"24px 20px", maxWidth:360, width:"100%", boxShadow:`0 0 50px ${cur.color}15`, animation: anim === "in" ? "tutoIn 0.22s ease" : "tutoOut 0.16s ease" }}>
        <TutoVisual type={cur.visual} color={cur.color} />
        <div style={{ textAlign:"center", marginTop:20 }}>
          <div style={{ fontSize:10, fontWeight:800, letterSpacing:2, color:cur.color, marginBottom:7, textTransform:"uppercase" }}>
            Étape {step + 1} / {TUTO.length}
          </div>
          <div style={{ color:"#fff", fontSize:19, fontWeight:900, letterSpacing:-0.4, marginBottom:9, lineHeight:1.25 }}>
            {cur.emoji} {cur.title}
          </div>
          <div style={{ color:"rgba(255,255,255,0.5)", fontSize:13, lineHeight:1.6 }}>
            {cur.desc}
          </div>
          {cur.tip && (
            <div style={{ marginTop:11, padding:"8px 12px", background:`${cur.color}15`, border:`1px solid ${cur.color}33`, borderRadius:10, color:cur.color, fontSize:12, fontStyle:"italic", lineHeight:1.45 }}>
              💡 {cur.tip}
            </div>
          )}
        </div>
      </div>

      <div style={{ display:"flex", gap:10, marginTop:18, width:"100%", maxWidth:360 }}>
        {step > 0 && (
          <button onClick={() => go(step - 1)} style={{ flex:1, padding:"13px", background:"rgba(255,255,255,0.07)", border:"1px solid rgba(255,255,255,0.13)", borderRadius:13, color:"rgba(255,255,255,0.55)", fontSize:14, fontWeight:700 }}>
            ‹ Retour
          </button>
        )}
        <button onClick={() => last ? onClose() : go(step + 1)} style={{ flex:2, padding:"13px", background:`linear-gradient(135deg,${cur.color},${cur.color}aa)`, border:"none", borderRadius:13, color:"#fff", fontSize:15, fontWeight:900, boxShadow:`0 4px 18px ${cur.color}44` }}>
          {last ? "C'est parti ! 🚀" : "Suivant ›"}
        </button>
      </div>
    </div>
  );
}

function TutoVisual({ type, color }) {
  const COLS = ["#e8504a","#ffd166","#06d6a0","#6a4c93","#118ab2","#f4a261","#ef476f","#8ac926"];
  if (type === "grid") {
    const filled = [0,2,5,7,8,11,14,17,20,22,25,28,30];
    return (
      <div style={{ display:"flex", justifyContent:"center" }}>
        <div style={{ display:"grid", gridTemplateColumns:"repeat(8,1fr)", gap:3, animation:"floatUp 3s ease-in-out infinite" }}>
          {Array.from({ length:32 }, (_, i) => (
            <div key={i} style={{ width:28, height:28, borderRadius:5, background: filled.includes(i) ? COLS[i % COLS.length] : "rgba(255,255,255,0.06)", border:"1px solid rgba(255,255,255,0.05)" }} />
          ))}
        </div>
      </div>
    );
  }
  if (type === "tap") {
    return (
      <div style={{ display:"flex", justifyContent:"center", alignItems:"center", gap:14 }}>
        <div style={{ position:"relative" }}>
          <div style={{ width:76, height:76, borderRadius:13, background:"rgba(255,255,255,0.06)", border:"2px solid rgba(255,255,255,0.1)", display:"grid", gridTemplateColumns:"repeat(4,1fr)", gap:2, padding:5 }}>
            {Array.from({ length:16 }, (_, i) => (
              <div key={i} style={{ borderRadius:3, background: i === 5 ? color : "rgba(255,255,255,0.05)" }} />
            ))}
          </div>
          <div style={{ position:"absolute", bottom:-8, right:-8, fontSize:26, animation:"floatUp 1.5s ease-in-out infinite" }}>👆</div>
        </div>
        <div style={{ width:120, padding:"10px 11px", background:"rgba(255,255,255,0.05)", border:"1px solid rgba(255,255,255,0.1)", borderRadius:12 }}>
          <div style={{ color:"rgba(255,255,255,0.4)", fontSize:9, marginBottom:5 }}>Pixel (42, 77)</div>
          <div style={{ display:"flex", gap:3, marginBottom:6 }}>
            {["#e8504a","#ffd166","#06d6a0","#6a4c93"].map(c => (
              <div key={c} style={{ width:16, height:16, borderRadius:4, background:c }} />
            ))}
          </div>
          <div style={{ padding:"4px 8px", background:color, borderRadius:6, color:"#fff", fontSize:9, fontWeight:800, textAlign:"center" }}>1 €</div>
        </div>
      </div>
    );
  }
  if (type === "sizes") {
    return (
      <div style={{ display:"flex", justifyContent:"center", alignItems:"flex-end", gap:16 }}>
        {[{ w:30, label:"1×1",   price:"1€",   c:"#e8504a", d:"2s"   },
          { w:56, label:"5×5",   price:"25€",  c:"#6a4c93", d:"2.3s" },
          { w:80, label:"10×10", price:"100€", c:"#118ab2", d:"2.7s" }].map(item => (
          <div key={item.label} style={{ textAlign:"center", animation:`floatUp ${item.d} ease-in-out infinite` }}>
            <div style={{ width:item.w, height:item.w, borderRadius:Math.max(5, item.w * 0.12), background:item.c, margin:"0 auto 6px", boxShadow:`0 4px 14px ${item.c}55` }} />
            <div style={{ color:"#fff", fontSize:11, fontWeight:800 }}>{item.label}</div>
            <div style={{ color:"#ffd166", fontSize:10, fontWeight:700 }}>{item.price}</div>
          </div>
        ))}
      </div>
    );
  }
  if (type === "link") {
    return (
      <div style={{ display:"flex", justifyContent:"center" }}>
        <div style={{ width:"100%", maxWidth:230, padding:"12px 13px", background:"rgba(255,255,255,0.05)", borderRadius:14, border:"1px solid rgba(255,255,255,0.08)" }}>
          <div style={{ display:"flex", alignItems:"center", gap:9, marginBottom:9 }}>
            <div style={{ width:34, height:34, borderRadius:8, background:color, boxShadow:`0 2px 12px ${color}66`, flexShrink:0 }} />
            <div>
              <div style={{ color:"#fff", fontSize:12, fontWeight:700 }}>Mon Pixel</div>
              <div style={{ color:"rgba(255,255,255,0.35)", fontSize:10 }}>{color}</div>
            </div>
          </div>
          <div style={{ padding:"7px 9px", background:"rgba(6,214,160,0.1)", border:"1px solid rgba(6,214,160,0.25)", borderRadius:8, marginBottom:7 }}>
            <div style={{ color:"rgba(255,255,255,0.3)", fontSize:9, marginBottom:2 }}>🔗 Lien vérifié</div>
            <div style={{ color:"#06d6a0", fontSize:11 }}>https://monsite.fr</div>
          </div>
          <div style={{ padding:"7px", background:"linear-gradient(135deg,#06d6a0,#118ab2)", borderRadius:8, color:"#fff", fontSize:11, fontWeight:800, textAlign:"center" }}>Ouvrir le lien 🔗</div>
        </div>
      </div>
    );
  }
  if (type === "shop") {
    return (
      <div style={{ display:"flex", flexDirection:"column", gap:6, maxWidth:240, margin:"0 auto" }}>
        {[
          { icon:"🟥", label:"Pixel gratuit",  sub:"Récompense journalière", c:"#06d6a0" },
          { icon:"🔨", label:"Enchère",         sub:"Prenez un pixel — 2€",   c:"#ffd166" },
          { icon:"💣", label:"Bombe",           sub:"Détruit 4×4 — 5€",       c:"#FF6B35" },
          { icon:"☢️", label:"Méga-Bombe",      sub:"Détruit 8×8 — 30€",      c:"#FF3366" },
        ].map((item, i) => (
          <div key={i} style={{ display:"flex", alignItems:"center", gap:9, padding:"7px 10px", background:"rgba(255,255,255,0.04)", border:"1px solid rgba(255,255,255,0.07)", borderRadius:10, animation:`floatUp ${2 + i * 0.35}s ease-in-out infinite` }}>
            <span style={{ fontSize:18, width:24, textAlign:"center" }}>{item.icon}</span>
            <div>
              <div style={{ color:"#fff", fontSize:11, fontWeight:700 }}>{item.label}</div>
              <div style={{ color:"rgba(255,255,255,0.38)", fontSize:9 }}>{item.sub}</div>
            </div>
          </div>
        ))}
      </div>
    );
  }
  if (type === "rewards") {
    return (
      <div style={{ textAlign:"center" }}>
        <div style={{ display:"flex", justifyContent:"center", gap:4, flexWrap:"wrap", marginBottom:12 }}>
          {STREAK_REWARDS.map((d, i) => (
            <div key={i} style={{ width:34, height:34, borderRadius:8, background: i < 3 ? "rgba(255,200,0,0.2)" : "rgba(255,255,255,0.05)", border:`1px solid ${i < 3 ? "rgba(255,200,0,0.4)" : "rgba(255,255,255,0.1)"}`, display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center" }}>
              <span style={{ fontSize:7, color: i < 3 ? "#ffd166" : "rgba(255,255,255,0.3)", fontWeight:700 }}>J{i+1}</span>
              <span style={{ fontSize:10, color: i < 3 ? "#ffd166" : "rgba(255,255,255,0.3)", fontWeight:800 }}>{d}</span>
            </div>
          ))}
        </div>
        <div style={{ display:"flex", justifyContent:"center", gap:18, fontSize:26 }}>
          {["🏆","🔥","🎁"].map((e, i) => (
            <span key={i} style={{ animation:`floatUp ${1.5 + i * 0.4}s ease-in-out infinite` }}>{e}</span>
          ))}
        </div>
      </div>
    );
  }
  return null;
}

// ─────────────────────────────────────────────
//  DAILY REWARD
// ─────────────────────────────────────────────
function DailyReward({ data, onClose }) {
  const { day, earned, tokens } = data;
  return (
    <div style={{ position:"fixed", inset:0, background:"rgba(0,0,0,0.85)", zIndex:400, display:"flex", alignItems:"center", justifyContent:"center", padding:20 }} onClick={onClose}>
      <div onClick={e => e.stopPropagation()} style={{ background:"#1a1a2e", border:"1px solid rgba(255,200,0,0.3)", borderRadius:24, padding:28, maxWidth:340, width:"100%", textAlign:"center", animation:"rewardIn 0.5s ease", boxShadow:"0 0 60px rgba(255,200,0,0.12)" }}>
        <div style={{ fontSize:56, marginBottom:8 }}>🎁</div>
        <div style={{ color:"#ffd166", fontSize:11, fontWeight:800, letterSpacing:2, marginBottom:6 }}>RÉCOMPENSE JOURNALIÈRE</div>
        <div style={{ color:"#fff", fontSize:22, fontWeight:900, marginBottom:4 }}>Jour {day} 🔥</div>
        <div style={{ color:"rgba(255,255,255,0.5)", fontSize:13, marginBottom:18 }}>
          {day > 1 ? `${day} jours consécutifs !` : "Bienvenue !"}
        </div>
        <div style={{ background:"linear-gradient(135deg,rgba(255,200,0,0.15),rgba(255,107,53,0.15))", border:"1px solid rgba(255,200,0,0.3)", borderRadius:16, padding:"14px 20px", marginBottom:18, display:"flex", alignItems:"center", justifyContent:"center", gap:12 }}>
          <span style={{ fontSize:30 }}>🟥</span>
          <div>
            <div style={{ color:"#ffd166", fontSize:28, fontWeight:900, lineHeight:1 }}>+{earned}</div>
            <div style={{ color:"rgba(255,255,255,0.5)", fontSize:11, marginTop:2 }}>pixel{earned > 1 ? "s" : ""} gratuit{earned > 1 ? "s" : ""}</div>
          </div>
        </div>
        <div style={{ display:"flex", gap:5, justifyContent:"center", flexWrap:"wrap", marginBottom:16 }}>
          {STREAK_REWARDS.map((d, i) => {
            const done = i < day, cur = i === day - 1;
            return (
              <div key={i} style={{ width:36, height:36, borderRadius:9, background: done ? "rgba(255,200,0,0.18)" : "rgba(255,255,255,0.05)", border:`1px solid ${cur ? "rgba(255,200,0,0.7)" : done ? "rgba(255,200,0,0.3)" : "rgba(255,255,255,0.1)"}`, display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", position:"relative" }}>
                <span style={{ fontSize:8, color: done ? "#ffd166" : "rgba(255,255,255,0.3)", fontWeight:700 }}>J{i+1}</span>
                <span style={{ fontSize:10, color: done ? "#ffd166" : "rgba(255,255,255,0.3)", fontWeight:800 }}>{d}</span>
                {cur && <div style={{ position:"absolute", top:-4, right:-4, width:8, height:8, background:"#ffd166", borderRadius:"50%" }} />}
              </div>
            );
          })}
        </div>
        <div style={{ color:"rgba(255,255,255,0.35)", fontSize:11, marginBottom:16 }}>
          Total : <strong style={{ color:"#ffd166" }}>{tokens} jeton{tokens > 1 ? "s" : ""}</strong>
        </div>
        <button onClick={onClose} style={{ width:"100%", padding:14, background:"linear-gradient(135deg,#ffd700,#f4a261)", borderRadius:13, color:"#111", fontSize:15, fontWeight:900, boxShadow:"0 4px 22px rgba(255,200,0,0.35)" }}>
          Récupérer 🎉
        </button>
      </div>
    </div>
  );
}

// ─────────────────────────────────────────────
//  ONBOARDING
// ─────────────────────────────────────────────
function Onboarding({ onDone }) {
  const [step,    setStep]    = useState(0);
  const [method,  setMethod]  = useState(null);
  const [name,    setName]    = useState("");
  const [email,   setEmail]   = useState("");
  const [cn,      setCn]      = useState("");
  const [ce,      setCe]      = useState("");
  const [cc,      setCc]      = useState("");
  const [busy,    setBusy]    = useState(false);
  const [cv,      setCv]      = useState(null);
  const [err,     setErr]     = useState("");

  const fmtN = v => v.replace(/\D/g, "").slice(0, 16).replace(/(\d{4})(?=\d)/g, "$1 ");
  const fmtE = v => { const d = v.replace(/\D/g, "").slice(0, 4); return d.length > 2 ? d.slice(0, 2) + "/" + d.slice(2) : d; };
  const ci   = cn.replace(/\s/g, "").length >= 4 ? cardInfo(cn.replace(/\s/g, "")) : null;
  const rechk = (n, e, c) => { if (n.replace(/\s/g, "").length >= 13) setCv(validateCard(n, e, c)); else setCv(null); };
  const bdrCol = v => v === null ? "rgba(255,255,255,0.12)" : v.ok ? "rgba(6,214,160,0.5)" : "rgba(255,51,102,0.5)";

  const submit = async () => {
    setErr("");
    if (!name.trim())          return setErr("Entrez votre prénom");
    if (!email.includes("@"))  return setErr("Email invalide");
    if (method.id === "card") {
      const v = validateCard(cn, ce, cc);
      setCv(v);
      if (!v.ok) return setErr(v.errs[0]);
    }
    setBusy(true);
    await new Promise(r => setTimeout(r, 1300));
    onDone({ name: name.trim(), email: email.trim(), method: method.id });
  };

  const inp = (extra = {}) => ({
    width:"100%", padding:"12px 14px",
    background:"rgba(255,255,255,0.07)",
    border:"1px solid rgba(255,255,255,0.12)",
    borderRadius:11, color:"#fff", fontSize:14,
    fontFamily:"system-ui",
    ...extra,
  });

  return (
    <div style={{ width:"100vw", height:"100vh", background:"#080810", display:"flex", flexDirection:"column", overflow:"hidden" }}>
      <div style={{ padding:"36px 24px 18px", textAlign:"center", flexShrink:0 }}>
        <div style={{ display:"flex", justifyContent:"center" }}>
          <LogoBanner />
        </div>
      </div>

      <div style={{ flex:1, overflowY:"auto", padding:"0 22px 48px" }}>
        {step === 0 && !busy && (
          <div style={{ animation:"fadeUp 0.3s ease" }}>
            <p style={{ color:"rgba(255,255,255,0.4)", fontSize:13, textAlign:"center", marginBottom:20 }}>
              Configurez votre paiement une seule fois.<br />
              <span style={{ fontSize:11, color:"rgba(255,255,255,0.25)" }}>Achetez chaque pixel en un seul tap.</span>
            </p>
            {PAY_METHODS.map(m => (
              <button key={m.id} onClick={() => { setMethod(m); setStep(1); }}
                style={{ display:"flex", alignItems:"center", gap:14, width:"100%", padding:"15px 18px", borderRadius:13, background:m.bg, border:m.border || "none", color:m.fg, fontSize:15, fontWeight:700, marginBottom:10, textAlign:"left" }}>
                <span style={{ fontSize:20, width:26, textAlign:"center" }}>{m.icon}</span>
                <span style={{ flex:1 }}>{m.name}</span>
                <span style={{ opacity:0.35 }}>›</span>
              </button>
            ))}
          </div>
        )}

        {step === 1 && !busy && (
          <div style={{ animation:"fadeUp 0.3s ease" }}>
            <div style={{ display:"flex", alignItems:"center", gap:10, marginBottom:20 }}>
              <button onClick={() => setStep(0)} style={{ background:"none", color:"rgba(255,255,255,0.4)", fontSize:22, lineHeight:1, padding:0 }}>‹</button>
              <div style={{ padding:"5px 12px", borderRadius:8, background:method.bg, color:method.fg, fontSize:12, fontWeight:800, border:method.border }}>
                {method.icon} {method.name}
              </div>
            </div>

            <FldRow label="Prénom & Nom">
              <input value={name} onChange={e => setName(e.target.value)} placeholder="Jean Dupont" autoComplete="given-name" style={inp()} />
            </FldRow>
            <FldRow label="Email">
              <input value={email} onChange={e => setEmail(e.target.value)} placeholder="jean@exemple.fr" type="email" inputMode="email" autoComplete="email" style={inp()} />
            </FldRow>

            {method.id === "card" && (
              <React.Fragment>
                <FldRow label="Numéro de carte">
                  <div style={{ position:"relative" }}>
                    <input value={cn} onChange={e => { const f = fmtN(e.target.value); setCn(f); rechk(f, ce, cc); }}
                      placeholder="1234 5678 9012 3456" inputMode="numeric" autoComplete="cc-number"
                      style={inp({ borderColor: bdrCol(cv), paddingRight: ci ? 84 : 14 })} />
                    {ci && (
                      <div style={{ position:"absolute", right:10, top:"50%", transform:"translateY(-50%)", display:"flex", alignItems:"center", gap:4 }}>
                        <span>{ci.icon}</span>
                        <span style={{ color:"rgba(255,255,255,0.45)", fontSize:10, fontWeight:700 }}>{ci.name}</span>
                      </div>
                    )}
                  </div>
                  {cv && (
                    <div style={{ marginTop:5, fontSize:11, color: cv.ok ? "#06d6a0" : "#FF3366", display:"flex", alignItems:"center", gap:4 }}>
                      <span>{cv.ok ? "✓" : "✗"}</span>
                      <span>{cv.ok ? `Valide · Luhn OK · ${ci?.name}` : cv.errs[0]}</span>
                    </div>
                  )}
                </FldRow>

                <div style={{ display:"flex", gap:10 }}>
                  <FldRow label="Expiration" style={{ flex:1 }}>
                    <input value={ce} onChange={e => { const f = fmtE(e.target.value); setCe(f); rechk(cn, f, cc); }}
                      placeholder="MM/AA" inputMode="numeric" autoComplete="cc-exp" style={inp()} />
                  </FldRow>
                  <FldRow label={`CVV${ci ? ` (${ci.cvv})` : ""}`} style={{ flex:1 }}>
                    <input value={cc} onChange={e => { const v = e.target.value.replace(/\D/g, "").slice(0, ci?.cvv || 4); setCc(v); rechk(cn, ce, v); }}
                      placeholder={ci?.cvv === 4 ? "••••" : "•••"} type="password" inputMode="numeric" autoComplete="cc-csc"
                      style={inp({ letterSpacing:4 })} />
                  </FldRow>
                </div>

                <div style={{ display:"flex", alignItems:"center", gap:7, padding:"9px 12px", background:"rgba(6,214,160,0.05)", border:"1px solid rgba(6,214,160,0.12)", borderRadius:9, marginTop:6 }}>
                  <span>🛡️</span>
                  <span style={{ color:"rgba(255,255,255,0.35)", fontSize:10 }}>Luhn · BIN · Expiry · CVV · Anti-fraude</span>
                </div>
              </React.Fragment>
            )}

            {err && (
              <div style={{ marginTop:12, padding:"9px 12px", background:"rgba(255,51,102,0.08)", border:"1px solid rgba(255,51,102,0.25)", borderRadius:9, color:"#FF3366", fontSize:12 }}>
                ✗ {err}
              </div>
            )}

            <button onClick={submit} style={{ width:"100%", padding:15, marginTop:18, background:"linear-gradient(135deg,#e8504a,#c0392b)", borderRadius:13, color:"#fff", fontSize:15, fontWeight:800, boxShadow:"0 4px 22px rgba(232,80,74,0.38)" }}>
              {method.id === "card" ? "Vérifier & accéder →" : `Connecter ${method.name} →`}
            </button>
            <p style={{ textAlign:"center", color:"rgba(255,255,255,0.12)", fontSize:10, marginTop:8 }}>🔒 Prototype de démonstration</p>
          </div>
        )}

        {busy && (
          <div style={{ textAlign:"center", padding:"52px 0", animation:"fadeUp 0.3s ease" }}>
            <Spinner size={44} />
            <div style={{ color:"#fff", fontSize:15, fontWeight:700, marginTop:18 }}>
              {method.id === "card" ? "Vérification bancaire…" : "Connexion…"}
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

// ─────────────────────────────────────────────
//  MAIN GRID
// ─────────────────────────────────────────────
function Grid({ profile, streak, onLogout, onTokenUpdate, onTuto }) {
  // Canvas refs
  const cvs     = useRef(null);
  const cam     = useRef({ x:0, y:0, z:0 });
  const inited  = useRef(false);
  const pixels  = useRef(new Map());  // "x,y" → color
  const blocks  = useRef(new Map());  // "x,y" → {size,color,url,label,owner}
  const p2b     = useRef(new Map());  // "x,y" → "bx,by"
  const drag    = useRef({ on:false, moved:false, sx:0, sy:0, cx:0, cy:0 });
  const pinch   = useRef({ on:false, dist:0 });
  const dirty   = useRef(true);
  const raf     = useRef(null);
  const boomR   = useRef(null);
  const soldR   = useRef(0);

  // Refs mirrored from state for canvas
  const selR    = useRef(null);
  const sizeR   = useRef(1);
  const colorR  = useRef("#e63946");
  const bombR   = useRef(null);
  const aucR    = useRef(false);
  const freeR   = useRef(false);

  // UI state
  const [sold,    setSoldSt]  = useState(0);
  const [tokens,  setTokens]  = useState(streak?.tokens || 0);
  const [menu,    setMenu]    = useState(false);
  const [shop,    setShop]    = useState(false);
  const [lb,      setLb]      = useState(false);
  const [lbData,  setLbData]  = useState([]);
  const [lbLoad,  setLbLoad]  = useState(false);
  const [sheet,   setSheet]   = useState(null);
  const [selPos,  setSelPos]  = useState(null);
  const [viewBlk, setViewBlk] = useState(null);
  const [bomb,    setBomb]    = useState(null);
  const [auction, setAuction] = useState(false);
  const [aucTgt,  setAucTgt]  = useState(null);
  const [freeMode,setFreeMode]= useState(false);
  const [bSize,   setBSize]   = useState(1);
  const [bColor,  setBColor]  = useState("#e63946");
  const [bUrl,    setBUrl]    = useState("");
  const [bLabel,  setBLabel]  = useState("");
  const [buying,  setBuying]  = useState(false);
  const [bStep,   setBStep]   = useState("");
  const [modR,    setModR]    = useState(null);
  const [aColor,  setAColor]  = useState("#e63946");
  const [aUrl,    setAUrl]    = useState("");
  const [aLabel,  setALabel]  = useState("");
  const [aMod,    setAMod]    = useState(null);
  const [toast,   setToast]   = useState(null);
  const [boomAnim,setBoomAnim]= useState(false);

  const method = PAY_METHODS.find(m => m.id === profile.method) || PAY_METHODS[3];

  const setSold = useCallback((u) => {
    soldR.current = typeof u === "function" ? u(soldR.current) : u;
    setSoldSt(soldR.current);
  }, []);

  useEffect(() => { setTokens(streak?.tokens || 0); }, [streak]);

  // Sync state → refs for canvas
  useEffect(() => { selR.current   = selPos;  dirty.current = true; }, [selPos]);
  useEffect(() => { sizeR.current  = bSize;   dirty.current = true; }, [bSize]);
  useEffect(() => { colorR.current = bColor;  dirty.current = true; }, [bColor]);
  useEffect(() => { bombR.current  = bomb;    dirty.current = true; }, [bomb]);
  useEffect(() => { aucR.current   = auction; dirty.current = true; }, [auction]);
  useEffect(() => { freeR.current  = freeMode;dirty.current = true; }, [freeMode]);

  const showToast = useCallback((msg, color) => {
    setToast({ msg, color });
    setTimeout(() => setToast(null), 2500);
  }, []);

  // ── Leaderboard ──────────────────────────────
  const openLb = useCallback(async () => {
    setLb(v => !v);
    setShop(false); setMenu(false);
    setLbLoad(true);
    const data = await lbRead();
    setLbData(Object.entries(data).map(([n, p]) => ({ name:n, pixels:Number(p) })).sort((a,b) => b.pixels - a.pixels));
    setLbLoad(false);
  }, []);

  const saveScore = useCallback(async (count) => {
    await lbWrite(profile.name, count);
  }, [profile.name]);

  // ── Draw ─────────────────────────────────────
  const draw = useCallback(() => {
    const canvas = cvs.current;
    if (!canvas) return;
    const ctx = canvas.getContext("2d");
    const W = canvas.width, H = canvas.height;
    const { x: px, y: py, z: zoom } = cam.current;

    // Background
    ctx.fillStyle = "#080810";
    ctx.fillRect(0, 0, W, H);

    // Dot pattern
    ctx.fillStyle = "rgba(255,255,255,0.018)";
    const sp = 24, ox = ((px * 0.04) % sp + sp) % sp, oy = ((py * 0.04) % sp + sp) % sp;
    for (let bx = ox; bx < W; bx += sp) for (let by = oy; by < H; by += sp) ctx.fillRect(bx, by, 1.5, 1.5);

    // White canvas
    ctx.fillStyle = "#f8f8f0";
    ctx.fillRect(Math.floor(px), Math.floor(py), Math.ceil(GRID * zoom), Math.ceil(GRID * zoom));

    // Visible range
    const gx0 = Math.max(0, Math.floor(-px / zoom));
    const gx1 = Math.min(GRID - 1, Math.ceil((W - px) / zoom));
    const gy0 = Math.max(0, Math.floor(-py / zoom));
    const gy1 = Math.min(GRID - 1, Math.ceil((H - py) / zoom));

    // Draw pixels
    for (const [k, c] of pixels.current) {
      const ci = k.indexOf(","), kx = +k.slice(0, ci), ky = +k.slice(ci + 1);
      if (kx < gx0 || kx > gx1 || ky < gy0 || ky > gy1) continue;
      ctx.fillStyle = c;
      ctx.fillRect(Math.floor(px + kx * zoom), Math.floor(py + ky * zoom), Math.ceil(zoom) + 1, Math.ceil(zoom) + 1);
    }

    // Block outlines
    if (zoom >= 1.5) {
      for (const [k, b] of blocks.current) {
        const ci = k.indexOf(","), bx = +k.slice(0, ci), by = +k.slice(ci + 1);
        if (bx + b.size < gx0 || bx > gx1 || by + b.size < gy0 || by > gy1) continue;
        ctx.strokeStyle = aucR.current && b.owner !== profile.name ? "rgba(255,200,0,0.55)" : "rgba(0,0,0,0.12)";
        ctx.lineWidth = Math.max(0.5, zoom * 0.05);
        ctx.strokeRect(Math.floor(px + bx * zoom) + 0.5, Math.floor(py + by * zoom) + 0.5, Math.ceil(b.size * zoom) - 1, Math.ceil(b.size * zoom) - 1);
        if (b.url && b.size >= 5 && zoom >= 8) {
          ctx.font = `bold ${Math.min(14, zoom * b.size * 0.12)}px system-ui`;
          ctx.textAlign = "center"; ctx.textBaseline = "middle";
          ctx.fillStyle = "rgba(0,0,0,0.22)";
          ctx.fillText("🔗", px + (bx + b.size / 2) * zoom, py + (by + b.size / 2) * zoom);
        }
      }
    }

    // Grid lines at high zoom
    if (zoom >= 12) {
      ctx.strokeStyle = "rgba(0,0,0,0.06)"; ctx.lineWidth = 0.4;
      for (let x = gx0; x <= gx1 + 1; x++) {
        ctx.beginPath(); ctx.moveTo(Math.round(px + x * zoom), Math.round(py + gy0 * zoom));
        ctx.lineTo(Math.round(px + x * zoom), Math.round(py + (gy1 + 1) * zoom)); ctx.stroke();
      }
      for (let y = gy0; y <= gy1 + 1; y++) {
        ctx.beginPath(); ctx.moveTo(Math.round(px + gx0 * zoom), Math.round(py + y * zoom));
        ctx.lineTo(Math.round(px + (gx1 + 1) * zoom), Math.round(py + y * zoom)); ctx.stroke();
      }
    }

    // Selection preview
    const sel = selR.current, ab = bombR.current;
    if (sel) {
      if (ab) {
        const r = ab.radius;
        const sx = Math.floor(px + sel.x * zoom), sy = Math.floor(py + sel.y * zoom), sw = Math.ceil(r * zoom);
        ctx.fillStyle = ab.color + "28"; ctx.fillRect(sx, sy, sw, sw);
        ctx.strokeStyle = ab.color; ctx.lineWidth = Math.max(2, zoom * 0.08);
        ctx.shadowColor = ab.color; ctx.shadowBlur = 14;
        ctx.strokeRect(sx + 0.5, sy + 0.5, sw - 1, sw - 1); ctx.shadowBlur = 0;
        const em = Math.max(12, Math.min(40, sw * 0.55));
        ctx.font = `${em}px system-ui`; ctx.textAlign = "center"; ctx.textBaseline = "middle";
        ctx.fillText(ab.icon, sx + sw / 2, sy + sw / 2);
      } else {
        const sz = sizeR.current, col = colorR.current;
        const sx = Math.floor(px + sel.x * zoom), sy = Math.floor(py + sel.y * zoom), sw = Math.ceil(sz * zoom);
        ctx.fillStyle = col + "55"; ctx.fillRect(sx, sy, sw, sw);
        const accent = freeR.current ? "#06d6a0" : "#e8504a";
        ctx.strokeStyle = accent; ctx.lineWidth = Math.max(2, zoom * 0.09);
        ctx.shadowColor = accent; ctx.shadowBlur = 10;
        ctx.strokeRect(sx + 0.5, sy + 0.5, sw - 1, sw - 1); ctx.shadowBlur = 0;
        if (freeR.current && sw > 22) {
          ctx.font = `bold ${Math.min(16, sw * 0.38)}px system-ui`; ctx.textAlign = "center"; ctx.textBaseline = "middle";
          ctx.fillStyle = "rgba(6,214,160,0.85)"; ctx.fillText("FREE", sx + sw / 2, sy + sw / 2);
        }
      }
    }

    // Boom ring
    if (boomR.current) {
      const b = boomR.current;
      const cx = px + (b.x + b.radius / 2) * zoom, cy = py + (b.y + b.radius / 2) * zoom;
      const rad = b.radius * zoom * 0.5 * b.t * 2.2;
      ctx.beginPath(); ctx.arc(cx, cy, rad, 0, Math.PI * 2);
      ctx.strokeStyle = `rgba(255,107,53,${1 - b.t})`; ctx.lineWidth = 8 * (1 - b.t); ctx.stroke();
    }
  }, [profile.name]);

  // RAF loop
  useEffect(() => {
    const loop = () => { if (dirty.current) { draw(); dirty.current = false; } raf.current = requestAnimationFrame(loop); };
    raf.current = requestAnimationFrame(loop);
    return () => cancelAnimationFrame(raf.current);
  }, [draw]);

  // Resize
  useEffect(() => {
    const canvas = cvs.current;
    const resize = () => {
      const dpr = window.devicePixelRatio || 1;
      canvas.width = canvas.offsetWidth * dpr;
      canvas.height = canvas.offsetHeight * dpr;
      if (!inited.current) {
        const fz = Math.min(canvas.width, canvas.height) / GRID * 0.8;
        cam.current = { x: (canvas.width - GRID * fz) / 2, y: (canvas.height - GRID * fz) / 2, z: fz };
        inited.current = true;
      }
      dirty.current = true;
    };
    resize();
    const ro = new ResizeObserver(resize);
    ro.observe(canvas);
    return () => ro.disconnect();
  }, []);

  const doZoom = useCallback((f, fx, fy) => {
    const { x, y, z } = cam.current;
    const nz = Math.max(MIN_ZOOM, Math.min(MAX_ZOOM, z * f));
    const ratio = nz / z;
    cam.current = { x: fx - (fx - x) * ratio, y: fy - (fy - y) * ratio, z: nz };
    dirty.current = true;
  }, []);

  const resetView = useCallback(() => {
    const c = cvs.current, dpr = window.devicePixelRatio || 1;
    const fz = Math.min(c.offsetWidth * dpr, c.offsetHeight * dpr) / GRID * 0.8;
    cam.current = { x: (c.offsetWidth * dpr - GRID * fz) / 2, y: (c.offsetHeight * dpr - GRID * fz) / 2, z: fz };
    dirty.current = true;
  }, []);

  const toGrid = useCallback((cx, cy) => {
    const c = cvs.current, dpr = window.devicePixelRatio || 1, rect = c.getBoundingClientRect();
    const { x: px, y: py, z: zoom } = cam.current;
    return { gx: Math.floor(((cx - rect.left) * dpr - px) / zoom), gy: Math.floor(((cy - rect.top) * dpr - py) / zoom) };
  }, []);

  const onTap = useCallback((cx, cy) => {
    const { gx, gy } = toGrid(cx, cy);
    if (gx < 0 || gx >= GRID || gy < 0 || gy >= GRID) return;
    const key = `${gx},${gy}`, ab = bombR.current;

    if (ab) {
      const r = ab.radius, tx = Math.max(0, Math.min(GRID - r, gx)), ty = Math.max(0, Math.min(GRID - r, gy));
      setSelPos({ x:tx, y:ty }); setSheet("bomb"); return;
    }
    if (aucR.current) {
      if (p2b.current.has(key)) {
        const bKey = p2b.current.get(key), b = blocks.current.get(bKey);
        const [bx, by] = bKey.split(",").map(Number);
        if (b.owner === profile.name) { showToast("Ce pixel vous appartient !", "#06d6a0"); return; }
        setAucTgt({ ...b, bx, by, bKey });
        setSelPos({ x:bx, y:by });
        setAColor(b.color); setAUrl(b.url || ""); setALabel(b.label || ""); setAMod(null);
        setSheet("auction");
      } else {
        showToast("Tapez un pixel existant pour enchérir", "#ffd166");
      }
      return;
    }
    if (freeR.current) {
      if (p2b.current.has(key)) { showToast("Zone déjà occupée !", "#f4a261"); return; }
      const tx = Math.max(0, Math.min(GRID - 1, gx)), ty = Math.max(0, Math.min(GRID - 1, gy));
      setSelPos({ x:tx, y:ty }); setSheet("free"); dirty.current = true; return;
    }
    if (p2b.current.has(key)) {
      const bKey = p2b.current.get(key), b = blocks.current.get(bKey);
      const [bx, by] = bKey.split(",").map(Number);
      setViewBlk({ ...b, bx, by }); setSelPos(null); setSheet("view");
    } else {
      const sz = sizeR.current;
      const tx = Math.max(0, Math.min(GRID - sz, gx)), ty = Math.max(0, Math.min(GRID - sz, gy));
      setSelPos({ x:tx, y:ty }); setModR(null); setSheet("buy");
    }
    dirty.current = true;
  }, [toGrid, profile.name, showToast]);

  // Mouse events
  const onMD = useCallback(e => { drag.current = { on:true, moved:false, sx:e.clientX, sy:e.clientY, cx:cam.current.x, cy:cam.current.y }; }, []);
  const onMM = useCallback(e => {
    if (!drag.current.on) return;
    const dx = e.clientX - drag.current.sx, dy = e.clientY - drag.current.sy;
    if (Math.abs(dx) > 4 || Math.abs(dy) > 4) drag.current.moved = true;
    if (drag.current.moved) { cam.current.x = drag.current.cx + dx; cam.current.y = drag.current.cy + dy; dirty.current = true; }
  }, []);
  const onMU = useCallback(e => { if (!drag.current.moved) onTap(e.clientX, e.clientY); drag.current.on = false; }, [onTap]);
  const onWhl = useCallback(e => {
    e.preventDefault();
    const rect = cvs.current.getBoundingClientRect(), dpr = window.devicePixelRatio || 1;
    doZoom(e.deltaY < 0 ? 1.12 : 0.89, (e.clientX - rect.left) * dpr, (e.clientY - rect.top) * dpr);
  }, [doZoom]);
  useEffect(() => {
    const c = cvs.current;
    c.addEventListener("wheel", onWhl, { passive:false });
    return () => c.removeEventListener("wheel", onWhl);
  }, [onWhl]);

  // Touch events
  const td = (a, b) => Math.hypot(a.clientX - b.clientX, a.clientY - b.clientY);
  const onTS = useCallback(e => {
    e.preventDefault();
    if (e.touches.length === 1) {
      const t = e.touches[0];
      drag.current = { on:true, moved:false, sx:t.clientX, sy:t.clientY, cx:cam.current.x, cy:cam.current.y };
      pinch.current.on = false;
    } else if (e.touches.length === 2) {
      drag.current.on = false;
      pinch.current = { on:true, dist: td(e.touches[0], e.touches[1]) };
    }
  }, []);
  const onTM = useCallback(e => {
    e.preventDefault();
    if (e.touches.length === 2 && pinch.current.on) {
      const nd = td(e.touches[0], e.touches[1]);
      const mx = (e.touches[0].clientX + e.touches[1].clientX) / 2;
      const my = (e.touches[0].clientY + e.touches[1].clientY) / 2;
      const rect = cvs.current.getBoundingClientRect(), dpr = window.devicePixelRatio || 1;
      doZoom(nd / pinch.current.dist, (mx - rect.left) * dpr, (my - rect.top) * dpr);
      pinch.current.dist = nd;
    } else if (e.touches.length === 1 && drag.current.on) {
      const t = e.touches[0];
      const dx = t.clientX - drag.current.sx, dy = t.clientY - drag.current.sy;
      if (Math.abs(dx) > 5 || Math.abs(dy) > 5) drag.current.moved = true;
      if (drag.current.moved) { cam.current.x = drag.current.cx + dx; cam.current.y = drag.current.cy + dy; dirty.current = true; }
    }
  }, [doZoom]);
  const onTE = useCallback(e => {
    e.preventDefault();
    if (e.changedTouches.length === 1 && !drag.current.moved && !pinch.current.on) onTap(e.changedTouches[0].clientX, e.changedTouches[0].clientY);
    if (e.touches.length < 2) pinch.current.on = false;
    if (e.touches.length === 0) drag.current.on = false;
  }, [onTap]);

  // ── Pixel operations ──────────────────────────
  const hasCol = (x, y, sz) => {
    for (let dx = 0; dx < sz; dx++) for (let dy = 0; dy < sz; dy++) if (pixels.current.has(`${x+dx},${y+dy}`)) return true;
    return false;
  };

  const commit = (x, y, sz, color, url, label, owner) => {
    for (let dx = 0; dx < sz; dx++) for (let dy = 0; dy < sz; dy++) pixels.current.set(`${x+dx},${y+dy}`, color);
    const bKey = `${x},${y}`;
    blocks.current.set(bKey, { size:sz, color, url:url||"", label:label||"", owner });
    for (let dx = 0; dx < sz; dx++) for (let dy = 0; dy < sz; dy++) p2b.current.set(`${x+dx},${y+dy}`, bKey);
  };

  const handleBuy = async () => {
    if (!selPos) return;
    const { x, y } = selPos, sz = bSize;
    if (bUrl.trim()) {
      setBStep("🔍 Modération…");
      const r = await moderateUrl(bUrl.trim());
      setModR(r);
      if (!r.ok) { setBStep(""); return; }
    }
    setBuying(true); setBStep("🔐 Auth…"); await new Promise(r => setTimeout(r, 600));
    setBStep("💳 Traitement…"); await new Promise(r => setTimeout(r, 600));
    commit(x, y, sz, bColor, bUrl, bLabel, profile.name);
    const ns = soldR.current + sz * sz; setSold(ns); dirty.current = true;
    await saveScore(ns);
    showToast(`Bloc ${sz}×${sz} acheté ! 🎉`, bColor);
    setBuying(false); setBStep(""); setSheet(null); setSelPos(null); setBUrl(""); setBLabel(""); setModR(null); dirty.current = true;
  };

  const handleFree = async () => {
    if (!selPos || tokens < 1) return;
    setBuying(true); setBStep("🟥 Placement…"); await new Promise(r => setTimeout(r, 800));
    commit(selPos.x, selPos.y, 1, bColor, "", "", profile.name);
    const ns = soldR.current + 1; setSold(ns); dirty.current = true;
    await saveScore(ns);
    const nt = tokens - 1; setTokens(nt); onTokenUpdate(nt);
    showToast("🟥 Pixel gratuit placé !", "#06d6a0");
    setBuying(false); setBStep(""); setSheet(null); setSelPos(null); setFreeMode(false); dirty.current = true;
  };

  const handleAuction = async () => {
    if (!aucTgt) return;
    if (aUrl.trim()) {
      setBStep("🔍 Modération…");
      const r = await moderateUrl(aUrl.trim());
      setAMod(r);
      if (!r.ok) { setBStep(""); return; }
    }
    setBuying(true); setBStep("🔨 Enchère…"); await new Promise(r => setTimeout(r, 600));
    setBStep("💳 2€…"); await new Promise(r => setTimeout(r, 600));
    const { bx, by, bKey, size } = aucTgt;
    for (let dx = 0; dx < size; dx++) for (let dy = 0; dy < size; dy++) pixels.current.set(`${bx+dx},${by+dy}`, aColor);
    blocks.current.set(bKey, { size, color:aColor, url:aUrl.trim(), label:aLabel.trim(), owner:profile.name });
    dirty.current = true;
    showToast("🏆 Pixel gagné à l'enchère !", "#ffd166");
    setBuying(false); setBStep(""); setSheet(null); setSelPos(null); setAucTgt(null); setAuction(false); dirty.current = true;
  };

  const handleBomb = async () => {
    if (!selPos || !bomb) return;
    const { x, y } = selPos, r = bomb.radius;
    setBuying(true); setBStep(`${bomb.icon} Détonation…`); await new Promise(res => setTimeout(res, 700));
    let d = 0;
    for (let dx = 0; dx < r; dx++) for (let dy = 0; dy < r; dy++) {
      const k = `${x+dx},${y+dy}`;
      if (pixels.current.has(k)) { pixels.current.delete(k); d++; }
      if (p2b.current.has(k)) { blocks.current.delete(p2b.current.get(k)); p2b.current.delete(k); }
    }
    setSold(s => Math.max(0, s - d)); dirty.current = true;
    boomR.current = { x, y, radius:r, t:0 };
    let start = null;
    const anim = ts => {
      if (!start) start = ts;
      const t = Math.min(1, (ts - start) / 600);
      boomR.current = { x, y, radius:r, t }; dirty.current = true;
      if (t < 1) requestAnimationFrame(anim);
      else { boomR.current = null; setBoomAnim(false); dirty.current = true; }
    };
    setBoomAnim(true); requestAnimationFrame(anim);
    showToast(`💥 Zone ${r}×${r} détruite !`, bomb.color);
    setBuying(false); setBStep(""); setSheet(null); setSelPos(null); setBomb(null);
  };

  const closeSheet = () => { setSheet(null); setSelPos(null); setViewBlk(null); setModR(null); setAucTgt(null); dirty.current = true; };
  const cancelBomb = () => { setBomb(null); setSheet(null); setSelPos(null); dirty.current = true; };
  const cancelAuction = () => { setAuction(false); setAucTgt(null); setSheet(null); setSelPos(null); dirty.current = true; };
  const cancelFree = () => { setFreeMode(false); setSheet(null); setSelPos(null); dirty.current = true; };

  const pct = ((sold / 1_000_000) * 100).toFixed(4);
  const si  = SIZES.find(s => s.n === bSize) || SIZES[0];
  const col = hasCol(selPos?.x, selPos?.y, bSize);
  const modBlocked = modR && !modR.ok;
  const aModBlocked = aMod && !aMod.ok;

  return (
    <div style={{ position:"relative", width:"100vw", height:"100vh", overflow:"hidden", background:"#080810" }}>
      <canvas ref={cvs}
        style={{ width:"100%", height:"100%", display:"block", touchAction:"none", cursor: (bomb || auction || freeMode) ? "crosshair" : "default" }}
        onMouseDown={onMD} onMouseMove={onMM} onMouseUp={onMU}
        onTouchStart={onTS} onTouchMove={onTM} onTouchEnd={onTE}
      />

      {boomAnim && <div style={{ position:"absolute", left:"50%", top:"50%", fontSize:70, pointerEvents:"none", animation:"boom 0.6s ease-out forwards", zIndex:50 }}>💥</div>}

      {/* ── TOP BAR ── */}
      <div style={{ position:"absolute", top:0, left:0, right:0, background:"rgba(8,8,16,0.92)", backdropFilter:"blur(14px)", borderBottom:"1px solid rgba(255,255,255,0.07)", zIndex:10 }}>
        <div style={{ display:"flex", alignItems:"center", gap:6, padding:"8px 10px" }}>
          <LogoMark size={26} />
          <div style={{ flex:1 }}>
            <div style={{ color:"#fff", fontSize:13, fontWeight:900, letterSpacing:-0.3, fontFamily:"Georgia,serif" }}>Pixel<em style={{ fontWeight:400 }}>Marché</em></div>
            <div style={{ color:"rgba(255,255,255,0.22)", fontSize:8, letterSpacing:1.5, fontFamily:"monospace" }}>UN MILLION DE PIXELS</div>
          </div>
          <div style={{ textAlign:"right", minWidth:54 }}>
            <div style={{ color: sold === 0 ? "rgba(255,255,255,0.25)" : "#ffd166", fontSize:12, fontWeight:700, whiteSpace:"nowrap" }}>{sold === 0 ? "Vierge" : sold.toLocaleString("fr-FR")}</div>
            <div style={{ color:"rgba(255,255,255,0.2)", fontSize:8, whiteSpace:"nowrap" }}>{sold === 0 ? "aucun pixel" : `${pct}%`}</div>
          </div>
          <div style={{ display:"flex", alignItems:"center", gap:3, padding:"3px 7px", background:"rgba(6,214,160,0.1)", border:"1px solid rgba(6,214,160,0.22)", borderRadius:20, flexShrink:0 }}>
            <span style={{ fontSize:10 }}>🟥</span>
            <span style={{ color:"#06d6a0", fontSize:11, fontWeight:800, minWidth:12, textAlign:"center" }}>{tokens}</span>
          </div>
          {[
            { icon:"❓", act:onTuto,                active:false },
            { icon:"🏆", act:openLb,                active:lb },
            { icon:"🛒", act:() => { setShop(v => !v); setLb(false); setMenu(false); }, active:shop },
            { icon:"👤", act:() => { setMenu(v => !v); setShop(false); setLb(false); }, active:menu },
          ].map(({ icon, act, active }) => (
            <button key={icon} onClick={act} style={{ width:30, height:30, borderRadius:8, background: active ? "rgba(232,80,74,0.2)" : "rgba(255,255,255,0.07)", border:`1px solid ${active ? "rgba(232,80,74,0.4)" : "rgba(255,255,255,0.1)"}`, color: active ? "#e8504a" : "rgba(255,255,255,0.6)", fontSize:14, display:"flex", alignItems:"center", justifyContent:"center", flexShrink:0 }}>{icon}</button>
          ))}
        </div>
        <div style={{ height:2, background:"rgba(255,255,255,0.05)" }}>
          <div style={{ height:"100%", width:`${Math.max(0, parseFloat(pct))}%`, background:"linear-gradient(90deg,#e8504a,#ffd166)", transition:"width 0.8s ease" }} />
        </div>
      </div>

      {/* Mode banners */}
      {bomb && <Banner color={bomb.color} icon={bomb.icon} label={`${bomb.name} — tapez la grille`} onCancel={cancelBomb} />}
      {auction && !bomb && <Banner color="#ffd700" icon="🔨" label="Enchère — tapez un pixel existant" onCancel={cancelAuction} />}
      {freeMode && !bomb && !auction && <Banner color="#06d6a0" icon="🟥" label={`Pixel gratuit (${tokens} restant${tokens > 1 ? "s" : ""}) — tapez pour placer`} onCancel={cancelFree} />}

      {/* ── LEADERBOARD ── */}
      {lb && (
        <div onClick={() => setLb(false)} style={{ position:"fixed", inset:0, zIndex:40 }}>
          <div onClick={e => e.stopPropagation()} style={{ position:"absolute", top:58, right:10, background:"#1a1a2e", border:"1px solid rgba(255,200,0,0.22)", borderRadius:16, padding:14, width:238, maxHeight:"65vh", overflowY:"auto", boxShadow:"0 12px 40px rgba(0,0,0,0.75)", zIndex:41, animation:"popIn 0.2s ease" }}>
            <div style={{ color:"#ffd166", fontSize:11, fontWeight:800, letterSpacing:1.5, marginBottom:12 }}>🏆 CLASSEMENT</div>
            {lbLoad
              ? <div style={{ display:"flex", justifyContent:"center", padding:18 }}><Spinner size={26} /></div>
              : lbData.length === 0
                ? <div style={{ color:"rgba(255,255,255,0.35)", fontSize:12, textAlign:"center", padding:"14px 0" }}>Aucun joueur encore.<br />Soyez le premier ! 🎨</div>
                : lbData.slice(0, 15).map((e, i) => {
                  const medals = ["🥇","🥈","🥉"], isMe = e.name === profile.name;
                  return (
                    <div key={e.name} style={{ display:"flex", alignItems:"center", gap:9, padding:"8px 10px", background: isMe ? "rgba(255,200,0,0.08)" : "rgba(255,255,255,0.03)", border:`1px solid ${isMe ? "rgba(255,200,0,0.25)" : "rgba(255,255,255,0.06)"}`, borderRadius:10, marginBottom:5 }}>
                      <span style={{ fontSize:15, width:20, textAlign:"center", flexShrink:0 }}>{i < 3 ? medals[i] : i + 1}</span>
                      <div style={{ flex:1, minWidth:0 }}>
                        <div style={{ color: isMe ? "#ffd166" : "#fff", fontSize:12, fontWeight: isMe ? 800 : 600, overflow:"hidden", textOverflow:"ellipsis", whiteSpace:"nowrap" }}>{e.name}{isMe ? " (vous)" : ""}</div>
                      </div>
                      <div style={{ color: isMe ? "#ffd166" : "rgba(255,255,255,0.5)", fontSize:11, fontWeight:700, flexShrink:0 }}>{Number(e.pixels).toLocaleString("fr-FR")} px</div>
                    </div>
                  );
                })
            }
            {!lbLoad && lbData.length > 0 && (
              <div style={{ marginTop:8, padding:"7px 10px", background:"rgba(255,255,255,0.03)", borderRadius:8, display:"flex", alignItems:"center", justifyContent:"space-between" }}>
                <span style={{ color:"rgba(255,255,255,0.3)", fontSize:10 }}>Votre score</span>
                <span style={{ color:"#e8504a", fontSize:11, fontWeight:700 }}>{sold.toLocaleString("fr-FR")} px</span>
              </div>
            )}
          </div>
        </div>
      )}

      {/* ── SHOP ── */}
      {shop && (
        <div onClick={() => setShop(false)} style={{ position:"fixed", inset:0, zIndex:40 }}>
          <div onClick={e => e.stopPropagation()} style={{ position:"absolute", top:58, right:44, background:"#1a1a2e", border:"1px solid rgba(255,255,255,0.1)", borderRadius:16, padding:14, minWidth:252, boxShadow:"0 12px 40px rgba(0,0,0,0.7)", zIndex:41, animation:"popIn 0.2s ease" }}>
            <div style={{ color:"rgba(255,255,255,0.4)", fontSize:10, letterSpacing:1.5, fontFamily:"monospace", marginBottom:12 }}>🛒 BOUTIQUE</div>
            <ShopBtn icon="🟥" bg="rgba(6,214,160,0.15)" bdr="rgba(6,214,160,0.35)" name="Pixel gratuit" desc="Utilisez vos jetons de récompense" price="GRATUIT" pc="#06d6a0" badge={`${tokens} jeton${tokens > 1 ? "s" : ""}`} disabled={tokens < 1} onClick={() => { setFreeMode(true); setShop(false); setSheet(null); setSelPos(null); }} />
            <ShopBtn icon="🔨" bg="rgba(255,200,0,0.12)" bdr="rgba(255,200,0,0.3)" name="Enchère" desc="Prenez un pixel existant et recolorez-le" price="2 €" pc="#ffd166" onClick={() => { setAuction(true); setShop(false); setSheet(null); setSelPos(null); setAucTgt(null); }} />
            {BOMBS.map(b => (
              <ShopBtn key={b.id} icon={b.icon} bg={`${b.color}22`} bdr={`${b.color}44`} name={b.name} desc={b.desc} price={`${b.price} €`} pc={b.color} onClick={() => { setBomb(b); setShop(false); setSheet(null); setSelPos(null); }} />
            ))}
            <div style={{ padding:"9px 11px", background:"rgba(255,200,0,0.05)", border:"1px solid rgba(255,200,0,0.12)", borderRadius:10, marginTop:4, display:"flex", alignItems:"center", gap:7 }}>
              <span>🔥</span>
              <div>
                <div style={{ color:"#ffd166", fontSize:11, fontWeight:700 }}>Série : Jour {streak?.day || 1}</div>
                <div style={{ color:"rgba(255,255,255,0.35)", fontSize:10 }}>Demain : +{STREAK_REWARDS[Math.min((streak?.day || 1), STREAK_REWARDS.length - 1)]} pixels</div>
              </div>
            </div>
          </div>
        </div>
      )}

      {/* ── PROFILE ── */}
      {menu && (
        <div onClick={() => setMenu(false)} style={{ position:"fixed", inset:0, zIndex:40 }}>
          <div onClick={e => e.stopPropagation()} style={{ position:"absolute", top:58, right:11, background:"#1a1a2e", border:"1px solid rgba(255,255,255,0.1)", borderRadius:14, padding:13, minWidth:200, boxShadow:"0 12px 36px rgba(0,0,0,0.6)", zIndex:41, animation:"popIn 0.2s ease" }}>
            <div style={{ display:"flex", alignItems:"center", gap:9, marginBottom:11 }}>
              <div style={{ width:32, height:32, borderRadius:8, background:"linear-gradient(135deg,#e8504a,#c0392b)", display:"flex", alignItems:"center", justifyContent:"center", color:"#fff", fontWeight:900, fontSize:14 }}>{profile.name[0]?.toUpperCase()}</div>
              <div>
                <div style={{ color:"#fff", fontWeight:700, fontSize:13 }}>{profile.name}</div>
                <div style={{ color:"rgba(255,255,255,0.32)", fontSize:10 }}>{profile.email}</div>
              </div>
            </div>
            <div style={{ padding:"6px 9px", background:"rgba(255,255,255,0.05)", borderRadius:8, display:"flex", alignItems:"center", justifyContent:"space-between", marginBottom:7 }}>
              <div style={{ display:"flex", alignItems:"center", gap:6 }}><span>{method.icon}</span><span style={{ color:"rgba(255,255,255,0.5)", fontSize:11, fontWeight:600 }}>{method.name}</span></div>
              <div style={{ display:"flex", alignItems:"center", gap:4 }}><span style={{ fontSize:10 }}>🟥</span><span style={{ color:"#06d6a0", fontSize:11, fontWeight:700 }}>{tokens}</span></div>
            </div>
            <div style={{ padding:"6px 9px", background:"rgba(255,200,0,0.05)", border:"1px solid rgba(255,200,0,0.12)", borderRadius:8, marginBottom:9, display:"flex", alignItems:"center", justifyContent:"space-between" }}>
              <div style={{ display:"flex", alignItems:"center", gap:6 }}><span>🔥</span><span style={{ color:"#ffd166", fontSize:11, fontWeight:600 }}>Jour {streak?.day || 1}</span></div>
              <span style={{ color:"rgba(255,255,255,0.3)", fontSize:10 }}>{sold} px</span>
            </div>
            <button onClick={() => { setMenu(false); onTuto(); }} style={{ width:"100%", padding:8, background:"rgba(255,255,255,0.06)", border:"1px solid rgba(255,255,255,0.1)", borderRadius:8, color:"rgba(255,255,255,0.6)", fontSize:11, fontWeight:600, marginBottom:7 }}>❓ Revoir le tutoriel</button>
            <button onClick={() => { setMenu(false); onLogout(); }} style={{ width:"100%", padding:8, background:"rgba(232,80,74,0.1)", border:"1px solid rgba(232,80,74,0.22)", borderRadius:8, color:"#e8504a", fontSize:11, fontWeight:700 }}>Changer de compte</button>
          </div>
        </div>
      )}

      {/* Zoom */}
      <div style={{ position:"absolute", right:11, top:"50%", transform:"translateY(-50%)", display:"flex", flexDirection:"column", gap:7, zIndex:10 }}>
        {[
          ["＋", () => { const c = cvs.current; doZoom(1.3, c.width / 2, c.height / 2); }],
          ["－", () => { const c = cvs.current; doZoom(0.77, c.width / 2, c.height / 2); }],
          ["⊙", resetView],
        ].map(([lbl, fn]) => (
          <button key={lbl} onClick={fn} style={{ width:34, height:34, borderRadius:8, background:"rgba(255,255,255,0.07)", border:"1px solid rgba(255,255,255,0.1)", color:"#fff", fontSize: lbl === "⊙" ? 12 : 17, display:"flex", alignItems:"center", justifyContent:"center", backdropFilter:"blur(8px)" }}>{lbl}</button>
        ))}
      </div>

      {sold === 0 && !sheet && !bomb && !auction && !freeMode && (
        <div style={{ position:"absolute", bottom:70, left:"50%", transform:"translateX(-50%)", background:"rgba(232,80,74,0.09)", backdropFilter:"blur(10px)", border:"1px solid rgba(232,80,74,0.25)", borderRadius:13, padding:"10px 16px", textAlign:"center", pointerEvents:"none", whiteSpace:"nowrap", zIndex:10 }}>
          <div style={{ color:"#e8504a", fontSize:13, fontWeight:700 }}>🎨 La toile est vierge !</div>
          <div style={{ color:"rgba(255,255,255,0.35)", fontSize:11, marginTop:2 }}>Touchez un pixel pour l'acheter</div>
        </div>
      )}
      {!sheet && !toast && sold > 0 && !bomb && !auction && !freeMode && (
        <div style={{ position:"absolute", bottom:14, left:"50%", transform:"translateX(-50%)", background:"rgba(8,8,16,0.8)", backdropFilter:"blur(8px)", color:"rgba(255,255,255,0.38)", fontSize:11, padding:"6px 13px", borderRadius:100, border:"1px solid rgba(255,255,255,0.07)", pointerEvents:"none", whiteSpace:"nowrap", zIndex:10 }}>
          Touchez · Glissez · Pincez pour zoomer
        </div>
      )}
      {toast && (
        <div style={{ position:"absolute", top:60, left:"50%", transform:"translateX(-50%)", background:"rgba(6,214,160,0.1)", backdropFilter:"blur(12px)", border:"1px solid rgba(6,214,160,0.3)", borderRadius:100, padding:"8px 16px", display:"flex", alignItems:"center", gap:8, whiteSpace:"nowrap", animation:"toastIn 0.3s ease", zIndex:30 }}>
          <div style={{ width:13, height:13, borderRadius:3, background:toast.color }} />
          <span style={{ color:"#fff", fontSize:12, fontWeight:600 }}>{toast.msg}</span>
        </div>
      )}

      {/* ── SHEETS ── */}
      {sheet === "buy" && (
        <Sheet onClose={closeSheet}>
          <SheetHdr label="Pixel sélectionné" title={`(${selPos?.x}, ${selPos?.y})`} onClose={closeSheet} />
          <SheetSec label="Taille">
            <div style={{ display:"flex", gap:7 }}>
              {SIZES.map(sz => (
                <button key={sz.n} onClick={() => setBSize(sz.n)} style={{ flex:1, padding:"9px 4px", borderRadius:10, background: bSize===sz.n ? "rgba(232,80,74,0.14)" : "rgba(255,255,255,0.05)", border: bSize===sz.n ? "1px solid rgba(232,80,74,0.45)" : "1px solid rgba(255,255,255,0.07)", textAlign:"center", transition:"all 0.12s" }}>
                  <div style={{ color: bSize===sz.n ? "#e8504a" : "#fff", fontSize:12, fontWeight:800 }}>{sz.label}</div>
                  <div style={{ color: bSize===sz.n ? "#ffd166" : "rgba(255,255,255,0.35)", fontSize:10, fontWeight:700, marginTop:2 }}>{sz.price}€</div>
                </button>
              ))}
            </div>
          </SheetSec>
          <SheetSec label="Couleur"><ColorPicker color={bColor} onChange={setBColor} /></SheetSec>
          <SheetSec label="🔗 Lien cliquable (optionnel)">
            <UrlInput url={bUrl} label={bLabel} onUrl={v => { setBUrl(v); setModR(null); }} onLabel={setBLabel} mod={modR} />
          </SheetSec>
          <Preview color={bColor} label={`${si.label} px`} price={`${si.price} €`} />
          {col && <Warn>⚠️ Zone occupée — choisissez un autre emplacement</Warn>}
          <PayButton method={method} price={`${si.price} €`} busy={buying} step={bStep} disabled={buying || col || modBlocked} onClick={handleBuy} />
          <p style={{ textAlign:"center", color:"rgba(255,255,255,0.1)", fontSize:9, marginTop:6 }}>{profile.name} · {profile.email}</p>
        </Sheet>
      )}

      {sheet === "free" && selPos && (
        <Sheet onClose={cancelFree}>
          <SheetHdr label="🟥 PIXEL GRATUIT" labelColor="#06d6a0" title={`(${selPos.x}, ${selPos.y})`} onClose={cancelFree} />
          <div style={{ padding:12, background:"rgba(6,214,160,0.07)", border:"1px solid rgba(6,214,160,0.22)", borderRadius:13, marginBottom:14, textAlign:"center" }}>
            <div style={{ fontSize:38, marginBottom:6 }}>🟥</div>
            <div style={{ color:"#fff", fontSize:14, fontWeight:700 }}>Pixel 1×1 gratuit</div>
            <div style={{ color:"rgba(255,255,255,0.45)", fontSize:12, marginTop:4 }}>Jetons disponibles : <strong style={{ color:"#06d6a0" }}>{tokens}</strong></div>
          </div>
          <SheetSec label="Couleur"><ColorPicker color={bColor} onChange={setBColor} /></SheetSec>
          <Preview color={bColor} label="1×1 px" price="GRATUIT" />
          <button onClick={handleFree} disabled={buying || tokens < 1} style={{ width:"100%", padding:14, background: tokens < 1 ? "rgba(255,255,255,0.06)" : "linear-gradient(135deg,#06d6a0,#118ab2)", borderRadius:13, color: tokens < 1 ? "rgba(255,255,255,0.2)" : "#fff", fontSize:14, fontWeight:800, display:"flex", alignItems:"center", justifyContent:"center", gap:8, boxShadow: tokens < 1 ? "none" : "0 4px 22px rgba(6,214,160,0.3)" }}>
            {buying ? <React.Fragment><Spinner size={17} color="#fff" /><span>{bStep}</span></React.Fragment> : <React.Fragment><span>🟥</span><span>Placer gratuitement</span><span style={{ opacity:0.4, marginLeft:"auto" }}>0 €</span></React.Fragment>}
          </button>
        </Sheet>
      )}

      {sheet === "auction" && aucTgt && (
        <Sheet onClose={cancelAuction}>
          <SheetHdr label="🔨 ENCHÈRE" labelColor="#ffd166" title={`(${aucTgt.bx},${aucTgt.by})`} onClose={cancelAuction} />
          <div style={{ display:"flex", gap:10, marginBottom:14 }}>
            <div style={{ flex:1, padding:10, background:"rgba(255,255,255,0.04)", borderRadius:10, border:"1px solid rgba(255,255,255,0.07)", textAlign:"center" }}>
              <div style={{ color:"rgba(255,255,255,0.3)", fontSize:9, marginBottom:5 }}>ACTUEL</div>
              <div style={{ width:32, height:32, borderRadius:7, background:aucTgt.color, margin:"0 auto 5px" }} />
              <div style={{ color:"rgba(255,255,255,0.5)", fontSize:10 }}>{aucTgt.owner || "?"}</div>
            </div>
            <div style={{ display:"flex", alignItems:"center", color:"#ffd166", fontSize:18 }}>→</div>
            <div style={{ flex:1, padding:10, background:"rgba(255,200,0,0.08)", borderRadius:10, border:"1px solid rgba(255,200,0,0.25)", textAlign:"center" }}>
              <div style={{ color:"#ffd166", fontSize:9, fontWeight:700, marginBottom:5 }}>VOTRE OFFRE</div>
              <div style={{ width:32, height:32, borderRadius:7, background:aColor, margin:"0 auto 5px", boxShadow:`0 0 10px ${aColor}66` }} />
              <div style={{ color:"#fff", fontSize:10, fontWeight:600 }}>{profile.name}</div>
            </div>
          </div>
          <SheetSec label="Nouvelle couleur"><ColorPicker color={aColor} onChange={setAColor} accent="#ffd700" /></SheetSec>
          <SheetSec label="🔗 Nouveau lien (optionnel)">
            <UrlInput url={aUrl} label={aLabel} onUrl={v => { setAUrl(v); setAMod(null); }} onLabel={setALabel} mod={aMod} />
          </SheetSec>
          <button onClick={handleAuction} disabled={buying || aModBlocked} style={{ width:"100%", padding:14, background: buying || aModBlocked ? "rgba(255,255,255,0.06)" : "linear-gradient(135deg,#ffd700,#f4a261)", borderRadius:13, color: buying || aModBlocked ? "rgba(255,255,255,0.2)" : "#111", fontSize:14, fontWeight:800, display:"flex", alignItems:"center", justifyContent:"center", gap:8, boxShadow: buying || aModBlocked ? "none" : "0 4px 22px rgba(255,200,0,0.3)" }}>
            {buying ? <React.Fragment><Spinner size={17} color="#111" /><span>{bStep}</span></React.Fragment> : <React.Fragment><span style={{ fontSize:18 }}>🔨</span><span>Remporter l'enchère</span><span style={{ opacity:0.5, marginLeft:"auto" }}>2 €</span></React.Fragment>}
          </button>
        </Sheet>
      )}

      {sheet === "bomb" && selPos && bomb && (
        <Sheet onClose={cancelBomb}>
          <SheetHdr label="Zone de destruction" title={`(${selPos.x}, ${selPos.y})`} onClose={cancelBomb} />
          <div style={{ padding:14, background:`${bomb.color}12`, border:`1px solid ${bomb.color}33`, borderRadius:13, marginBottom:14, textAlign:"center" }}>
            <div style={{ fontSize:46, marginBottom:8 }}>{bomb.icon}</div>
            <div style={{ color:"#fff", fontSize:16, fontWeight:900 }}>{bomb.name}</div>
            <div style={{ color:"rgba(255,255,255,0.45)", fontSize:12, marginTop:4 }}>{bomb.desc}</div>
            <div style={{ display:"flex", justifyContent:"center", gap:8, marginTop:12 }}>
              <span style={{ padding:"4px 12px", background:`${bomb.color}22`, border:`1px solid ${bomb.color}44`, borderRadius:20, color:bomb.color, fontSize:11, fontWeight:700 }}>Zone {bomb.radius}×{bomb.radius}</span>
              <span style={{ padding:"4px 12px", background:"rgba(255,209,102,0.1)", border:"1px solid rgba(255,209,102,0.3)", borderRadius:20, color:"#ffd166", fontSize:11, fontWeight:700 }}>{bomb.price} €</span>
            </div>
          </div>
          <Warn>⚠️ Action irréversible — tous les pixels seront supprimés.</Warn>
          <PayButton method={method} price={`${bomb.price} €`} busy={buying} step={bStep} disabled={buying} onClick={handleBomb} label={`Détonation — ${bomb.price} €`} />
          <button onClick={cancelBomb} style={{ width:"100%", padding:11, background:"none", border:"1px solid rgba(255,255,255,0.1)", borderRadius:12, color:"rgba(255,255,255,0.45)", fontSize:13, fontWeight:600, marginTop:8 }}>Annuler</button>
        </Sheet>
      )}

      {sheet === "view" && viewBlk && (
        <Sheet onClose={closeSheet}>
          <div style={{ display:"flex", justifyContent:"space-between", alignItems:"flex-start", marginBottom:14 }}>
            <div style={{ display:"flex", alignItems:"center", gap:12 }}>
              <div style={{ width:44, height:44, borderRadius:10, background:viewBlk.color, boxShadow:`0 4px 16px ${viewBlk.color}77`, flexShrink:0 }} />
              <div>
                <div style={{ color:"rgba(255,255,255,0.3)", fontSize:10, marginBottom:2 }}>Bloc {viewBlk.size}×{viewBlk.size} · ({viewBlk.bx},{viewBlk.by})</div>
                <div style={{ color:"#fff", fontWeight:800, fontSize:14 }}>{viewBlk.label || "Pixel acheté"}</div>
                {viewBlk.owner && <div style={{ color:"rgba(255,255,255,0.3)", fontSize:10 }}>par {viewBlk.owner}</div>}
              </div>
            </div>
            <CloseBtn onClick={closeSheet} />
          </div>
          {viewBlk.url
            ? (
              <React.Fragment>
                <div style={{ padding:"9px 12px", background:"rgba(255,255,255,0.04)", borderRadius:9, marginBottom:12, border:"1px solid rgba(255,255,255,0.06)" }}>
                  <div style={{ color:"rgba(255,255,255,0.28)", fontSize:9, marginBottom:3 }}>🔗 Lien</div>
                  <div style={{ color:"#06d6a0", fontSize:12, wordBreak:"break-all" }}>{viewBlk.url}</div>
                </div>
                <button onClick={() => window.open(viewBlk.url.startsWith("http") ? viewBlk.url : "https://" + viewBlk.url, "_blank")} style={{ width:"100%", padding:14, background:"linear-gradient(135deg,#06d6a0,#118ab2)", borderRadius:13, color:"#fff", fontSize:14, fontWeight:800, boxShadow:"0 4px 18px rgba(6,214,160,0.3)", display:"flex", alignItems:"center", justifyContent:"center", gap:8 }}>
                  🔗 Ouvrir {viewBlk.label || "le lien"}
                </button>
              </React.Fragment>
            )
            : <div style={{ textAlign:"center", padding:"14px 0", color:"rgba(255,255,255,0.28)", fontSize:13 }}>Ce bloc n'a pas de lien associé.</div>
          }
        </Sheet>
      )}
    </div>
  );
}

// ─────────────────────────────────────────────
//  SHARED COMPONENTS
// ─────────────────────────────────────────────
function Banner({ color, icon, label, onCancel }) {
  return (
    <div style={{ position:"absolute", top:58, left:"50%", transform:"translateX(-50%)", background:`${color}18`, backdropFilter:"blur(10px)", border:`1px solid ${color}44`, borderRadius:12, padding:"9px 16px", display:"flex", alignItems:"center", gap:10, zIndex:20, whiteSpace:"nowrap", animation:"popIn 0.2s ease" }}>
      <span style={{ fontSize:18 }}>{icon}</span>
      <span style={{ color:"#fff", fontSize:12, fontWeight:700 }}>{label}</span>
      <button onClick={onCancel} style={{ background:"rgba(255,255,255,0.12)", borderRadius:7, padding:"4px 9px", color:"rgba(255,255,255,0.7)", fontSize:11, fontWeight:600 }}>Annuler</button>
    </div>
  );
}

function ShopBtn({ icon, bg, bdr, name, desc, price, pc, badge, disabled, onClick }) {
  return (
    <button onClick={onClick} disabled={disabled} style={{ display:"flex", alignItems:"center", gap:11, width:"100%", padding:"11px 12px", background:"rgba(255,255,255,0.04)", border:"1px solid rgba(255,255,255,0.07)", borderRadius:12, marginBottom:8, textAlign:"left", opacity: disabled ? 0.45 : 1 }}>
      <div style={{ width:44, height:44, borderRadius:11, background:bg, border:`1px solid ${bdr}`, display:"flex", alignItems:"center", justifyContent:"center", fontSize:22, flexShrink:0 }}>{icon}</div>
      <div style={{ flex:1 }}>
        <div style={{ color:"#fff", fontWeight:800, fontSize:14 }}>{name}</div>
        <div style={{ color:"rgba(255,255,255,0.38)", fontSize:11, marginTop:2 }}>{desc}</div>
      </div>
      <div style={{ textAlign:"right", flexShrink:0 }}>
        <div style={{ color:pc, fontSize:15, fontWeight:900 }}>{price}</div>
        {badge && <div style={{ color:"rgba(255,255,255,0.35)", fontSize:9, marginTop:1 }}>{badge}</div>}
      </div>
    </button>
  );
}

function Sheet({ children, onClose }) {
  return (
    <div style={{ position:"absolute", bottom:0, left:0, right:0, background:"#111119", borderRadius:"20px 20px 0 0", border:"1px solid rgba(255,255,255,0.08)", borderBottom:"none", paddingBottom:"max(env(safe-area-inset-bottom,14px),14px)", boxShadow:"0 -20px 50px rgba(0,0,0,0.7)", animation:"slideUp 0.25s cubic-bezier(0.32,0.72,0,1)", maxHeight:"88vh", overflowY:"auto", zIndex:20 }}>
      <div style={{ display:"flex", justifyContent:"center", padding:"10px 0 0" }}>
        <div style={{ width:28, height:3.5, borderRadius:2, background:"rgba(255,255,255,0.1)" }} />
      </div>
      <div style={{ padding:"10px 17px 0" }}>{children}</div>
    </div>
  );
}

function SheetHdr({ label, labelColor, title, onClose }) {
  return (
    <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:14 }}>
      <div>
        <div style={{ color: labelColor || "rgba(255,255,255,0.3)", fontSize:10, marginBottom:1 }}>{label}</div>
        <div style={{ color:"#fff", fontWeight:900, fontSize:18, letterSpacing:-0.5 }}>{title}</div>
      </div>
      <CloseBtn onClick={onClose} />
    </div>
  );
}

function SheetSec({ label, children }) {
  return (
    <div style={{ marginBottom:13 }}>
      <div style={{ color:"rgba(255,255,255,0.3)", fontSize:10, marginBottom:7 }}>{label}</div>
      {children}
    </div>
  );
}

function CloseBtn({ onClick }) {
  return (
    <button onClick={onClick} style={{ width:30, height:30, borderRadius:8, background:"rgba(255,255,255,0.07)", border:"1px solid rgba(255,255,255,0.1)", color:"rgba(255,255,255,0.55)", fontSize:17, display:"flex", alignItems:"center", justifyContent:"center" }}>×</button>
  );
}

function Warn({ children }) {
  return (
    <div style={{ marginBottom:10, padding:"8px 10px", background:"rgba(244,162,97,0.08)", border:"1px solid rgba(244,162,97,0.2)", borderRadius:8, color:"#f4a261", fontSize:12, fontWeight:600, textAlign:"center" }}>{children}</div>
  );
}

function ColorPicker({ color, onChange, accent = "#e8504a" }) {
  return (
    <div style={{ display:"flex", gap:7, overflowX:"auto", paddingBottom:3, WebkitOverflowScrolling:"touch" }}>
      {PALETTE.map(c => (
        <button key={c} onClick={() => onChange(c)} style={{ width:32, height:32, borderRadius:7, background:c, border:"none", flexShrink:0, outline: color === c ? `3px solid ${accent}` : "2px solid transparent", outlineOffset:2, transform: color === c ? "scale(1.14)" : "scale(1)", transition:"transform 0.1s", boxShadow: color === c ? `0 0 10px ${c}99` : "none" }} />
      ))}
      <label style={{ width:32, height:32, borderRadius:7, border:"2px dashed rgba(255,255,255,0.15)", display:"flex", alignItems:"center", justifyContent:"center", cursor:"pointer", overflow:"hidden", position:"relative", flexShrink:0 }}>
        <span style={{ fontSize:17, color:"rgba(255,255,255,0.25)", pointerEvents:"none" }}>+</span>
        <input type="color" value={color} onChange={e => onChange(e.target.value)} style={{ opacity:0, position:"absolute", inset:0, cursor:"pointer" }} />
      </label>
    </div>
  );
}

function UrlInput({ url, label, onUrl, onLabel, mod }) {
  const blocked = mod && !mod.ok, ok = mod && mod.ok;
  return (
    <React.Fragment>
      <input value={url} onChange={e => onUrl(e.target.value)} placeholder="https://monsite.fr"
        style={{ width:"100%", padding:"10px 12px", background:"rgba(255,255,255,0.06)", border:`1px solid ${blocked ? "rgba(255,51,102,0.45)" : ok ? "rgba(6,214,160,0.4)" : "rgba(255,255,255,0.09)"}`, borderRadius:9, color:"#fff", fontSize:12, marginBottom:6, fontFamily:"system-ui" }} />
      <input value={label} onChange={e => onLabel(e.target.value)} placeholder="Nom affiché"
        style={{ width:"100%", padding:"8px 12px", background:"rgba(255,255,255,0.04)", border:"1px solid rgba(255,255,255,0.07)", borderRadius:9, color:"#fff", fontSize:11, fontFamily:"system-ui" }} />
      {blocked && <div style={{ marginTop:6, padding:"7px 10px", background:"rgba(255,51,102,0.08)", border:"1px solid rgba(255,51,102,0.25)", borderRadius:8, color:"#FF3366", fontSize:11, fontWeight:600 }}>🚫 {mod.reason}</div>}
      {ok && url && <div style={{ marginTop:4, color:"#06d6a0", fontSize:11, display:"flex", alignItems:"center", gap:4 }}><span>✓</span><span>Lien approuvé</span></div>}
    </React.Fragment>
  );
}

function Preview({ color, label, price }) {
  return (
    <div style={{ display:"flex", alignItems:"center", gap:11, padding:"9px 12px", background:"rgba(255,255,255,0.04)", borderRadius:11, marginBottom:13, border:"1px solid rgba(255,255,255,0.05)" }}>
      <div style={{ width:38, height:38, borderRadius:8, background:color, flexShrink:0, transition:"background 0.13s", boxShadow:`0 2px 12px ${color}55` }} />
      <div style={{ flex:1 }}>
        <div style={{ color:"#fff", fontSize:11, fontWeight:600 }}>{label}</div>
        <div style={{ color:"rgba(255,255,255,0.25)", fontSize:9, fontFamily:"monospace" }}>{color.toUpperCase()}</div>
      </div>
      <div style={{ textAlign:"right" }}>
        <div style={{ color:"#ffd166", fontSize:18, fontWeight:900 }}>{price}</div>
        <div style={{ color:"rgba(255,255,255,0.2)", fontSize:8 }}>à vie</div>
      </div>
    </div>
  );
}

function PayButton({ method, price, busy, step, disabled, onClick, label }) {
  return (
    <button onClick={onClick} disabled={disabled} style={{ width:"100%", padding:14, background: disabled && !busy ? "rgba(255,255,255,0.06)" : method.bg, border: method.border || "none", borderRadius:13, color: disabled && !busy ? "rgba(255,255,255,0.2)" : method.fg, fontSize:14, fontWeight:800, display:"flex", alignItems:"center", justifyContent:"center", gap:8, opacity: busy ? 0.8 : 1, transition:"all 0.15s", cursor: disabled ? "not-allowed" : "pointer" }}>
      {busy
        ? <React.Fragment><Spinner size={17} color={method.fg} /><span>{step}</span></React.Fragment>
        : <React.Fragment><span style={{ fontSize:18 }}>{method.icon}</span><span>{label || `Payer avec ${method.name}`}</span><span style={{ opacity:0.4, marginLeft:"auto" }}>{price}</span></React.Fragment>
      }
    </button>
  );
}

function Spinner({ size = 38, color = "#e8504a" }) {
  return (
    <div style={{ width:size, height:size, border:`${Math.max(2, size * 0.07)}px solid rgba(255,255,255,0.08)`, borderTopColor:color, borderRadius:"50%", animation:"spin 0.75s linear infinite", flexShrink:0 }} />
  );
}

function FldRow({ label, children, style }) {
  return (
    <div style={{ marginTop:13, ...style }}>
      <div style={{ color:"rgba(255,255,255,0.36)", fontSize:10, letterSpacing:0.3, marginBottom:5 }}>{label}</div>
      {children}
    </div>
  );
}
