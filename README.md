intern id:CITS2678
import { useState, useRef, useEffect } from "react";

// ── Heuristic NLP Analysis (runs client-side, no API needed) ──────────────────
const SENSATIONAL_WORDS = ["shocking","bombshell","explosive","outrage","miracle","secret","exposed","unbelievable","hoax","cover-up","they don't want you","mainstream media","deep state","big pharma","wake up","sheeple","100%","never before","banned","censored","breaking","urgent","alert","must read","share before deleted"];
const CREDIBLE_SIGNALS = ["according to","study shows","research indicates","officials said","reported by","data suggests","per the","sources confirm","published in","cited by","peer-reviewed","experts say","analysis"];
const CLICKBAIT_PATTERNS = [/^[A-Z\s!?]+$/, /\b(YOU WON'T BELIEVE|WHAT HAPPENS NEXT)\b/i, /\?\?+/, /!!+/];
const OPINION_WORDS = ["I think","I believe","everyone knows","obviously","clearly","always","never","all","none","every","must be","certainly","definitely","absolutely"];

function analyzeText(text) {
  if (!text || text.trim().length < 20) return null;
  const lower = text.toLowerCase();
  const words = lower.split(/\s+/);
  const sentences = text.split(/[.!?]+/).filter(Boolean);

  // Sensationalism score
  const sensMatch = SENSATIONAL_WORDS.filter(w => lower.includes(w));
  const sensScore = Math.min(sensMatch.length / 4, 1);

  // Credibility signals
  const credMatch = CREDIBLE_SIGNALS.filter(w => lower.includes(w));
  const credScore = Math.min(credMatch.length / 4, 1);

  // Clickbait
  const clickbait = CLICKBAIT_PATTERNS.some(p => p.test(text));

  // Opinion/bias
  const opinionMatch = OPINION_WORDS.filter(w => lower.includes(w.toLowerCase()));
  const opinionScore = Math.min(opinionMatch.length / 5, 1);

  // Exclamation overuse
  const exclamations = (text.match(/!/g) || []).length;
  const exclamScore = Math.min(exclamations / 5, 1);

  // ALL CAPS words
  const capsWords = words.filter(w => w.length > 3 && w === w.toUpperCase() && /[A-Z]/.test(w));
  const capsScore = Math.min(capsWords.length / 5, 1);

  // URL presence (crude signal)
  const hasUrl = /https?:\/\/|www\./i.test(text);

  // Avg sentence length (very short = clickbait, very long = dense/legit)
  const avgLen = words.length / Math.max(sentences.length, 1);
  const lengthScore = avgLen < 8 ? 0.3 : avgLen > 20 ? -0.1 : 0;

  // Composite fake score (0 = real, 1 = fake)
  const fakeScore = Math.max(0, Math.min(1,
    sensScore * 0.35 +
    (1 - credScore) * 0.25 +
    opinionScore * 0.15 +
    exclamScore * 0.1 +
    capsScore * 0.1 +
    (clickbait ? 0.2 : 0) +
    lengthScore
  ));

  let verdict, confidence;
  if (fakeScore >= 0.65) { verdict = "LIKELY FAKE"; confidence = Math.round(fakeScore * 100); }
  else if (fakeScore >= 0.40) { verdict = "SUSPICIOUS"; confidence = Math.round(fakeScore * 100); }
  else { verdict = "LIKELY REAL"; confidence = Math.round((1 - fakeScore) * 100); }

  return {
    verdict,
    fakeScore,
    confidence,
    signals: {
      sensationalism: Math.round(sensScore * 100),
      credibility: Math.round(credScore * 100),
      emotionalBias: Math.round(opinionScore * 100),
      clickbait,
      capsAbuse: Math.round(capsScore * 100),
      exclamationAbuse: Math.round(exclamScore * 100),
    },
    matched: {
      sensational: sensMatch.slice(0, 5),
      credible: credMatch.slice(0, 5),
      opinion: opinionMatch.slice(0, 4),
    }
  };
}

// ── Styles ────────────────────────────────────────────────────────────────────
const CSS = `
@import url('https://fonts.googleapis.com/css2?family=Syne:wght@400;600;700;800&family=IBM+Plex+Mono:wght@400;500;600&display=swap');

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --bg: #0a0b0d;
  --surface: #111318;
  --surface2: #181b22;
  --border: rgba(255,255,255,0.07);
  --text: #e8eaf0;
  --muted: rgba(232,234,240,0.4);
  --real: #22c55e;
  --fake: #ef4444;
  --warn: #f59e0b;
  --accent: #6ee7f7;
  --mono: 'IBM Plex Mono', monospace;
  --sans: 'Syne', sans-serif;
}

body { background: var(--bg); color: var(--text); font-family: var(--sans); min-height: 100vh; }

.app {
  min-height: 100vh;
  background: var(--bg);
  position: relative;
  overflow: hidden;
}

/* animated grid background */
.grid-bg {
  position: fixed; inset: 0; pointer-events: none; z-index: 0;
  background-image:
    linear-gradient(rgba(110,231,247,0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(110,231,247,0.03) 1px, transparent 1px);
  background-size: 48px 48px;
}

.scanline {
  position: fixed; top: 0; left: 0; right: 0; height: 2px;
  background: linear-gradient(90deg, transparent, rgba(110,231,247,0.4), transparent);
  animation: scan 4s linear infinite;
  z-index: 1; pointer-events: none;
}
@keyframes scan { from { top: 0; } to { top: 100vh; } }

.content { position: relative; z-index: 2; max-width: 980px; margin: 0 auto; padding: 0 24px 60px; }

/* Header */
.header { padding: 48px 0 40px; }
.header-tag {
  display: inline-flex; align-items: center; gap: 8px;
  font-family: var(--mono); font-size: 0.7rem; color: var(--accent);
  letter-spacing: 0.15em; text-transform: uppercase;
  border: 1px solid rgba(110,231,247,0.25);
  padding: 4px 12px; border-radius: 2px; margin-bottom: 24px;
}
.dot { width: 6px; height: 6px; border-radius: 50%; background: var(--accent); animation: blink 1.2s ease-in-out infinite; }
@keyframes blink { 0%,100% { opacity: 1; } 50% { opacity: 0.2; } }

.header h1 {
  font-size: clamp(2.2rem, 5vw, 4rem);
  font-weight: 800;
  line-height: 1;
  letter-spacing: -0.03em;
  margin-bottom: 12px;
}
.header h1 em { font-style: normal; color: var(--fake); }
.header p { font-family: var(--mono); font-size: 0.78rem; color: var(--muted); line-height: 1.7; max-width: 560px; }

/* Input section */
.input-section { margin-bottom: 32px; }

.input-label {
  font-family: var(--mono); font-size: 0.68rem; color: var(--accent);
  letter-spacing: 0.12em; text-transform: uppercase;
  margin-bottom: 10px; display: block;
}

textarea {
  width: 100%; min-height: 180px;
  background: var(--surface); color: var(--text);
  border: 1px solid var(--border); border-radius: 4px;
  padding: 18px 20px;
  font-family: var(--mono); font-size: 0.85rem; line-height: 1.7;
  resize: vertical; outline: none;
  transition: border-color 0.2s;
}
textarea:focus { border-color: rgba(110,231,247,0.4); }
textarea::placeholder { color: var(--muted); }

.controls { display: flex; gap: 12px; margin-top: 12px; flex-wrap: wrap; align-items: center; }

.btn {
  padding: 12px 24px;
  font-family: var(--sans); font-size: 0.82rem; font-weight: 700;
  letter-spacing: 0.08em; text-transform: uppercase;
  border: none; border-radius: 3px; cursor: pointer;
  transition: all 0.15s ease;
}
.btn-primary { background: var(--accent); color: #0a0b0d; }
.btn-primary:hover { background: #a5f3fc; transform: translateY(-1px); }
.btn-primary:disabled { opacity: 0.4; cursor: not-allowed; transform: none; }
.btn-ghost {
  background: transparent; color: var(--muted);
  border: 1px solid var(--border);
}
.btn-ghost:hover { color: var(--text); border-color: rgba(255,255,255,0.2); }

.char-count { font-family: var(--mono); font-size: 0.68rem; color: var(--muted); margin-left: auto; }

/* Mode tabs */
.tabs { display: flex; gap: 4px; margin-bottom: 24px; background: var(--surface); border: 1px solid var(--border); border-radius: 4px; padding: 4px; }
.tab {
  flex: 1; padding: 9px 12px; text-align: center;
  font-family: var(--mono); font-size: 0.72rem; font-weight: 600; letter-spacing: 0.08em; text-transform: uppercase;
  border-radius: 2px; cursor: pointer; border: none;
  background: transparent; color: var(--muted); transition: all 0.15s;
}
.tab.active { background: var(--surface2); color: var(--accent); }

/* Results */
.result-card {
  background: var(--surface); border: 1px solid var(--border);
  border-radius: 6px; overflow: hidden;
  animation: revealUp 0.35s ease forwards; opacity: 0;
}
@keyframes revealUp { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: translateY(0); } }

.verdict-bar {
  padding: 28px 28px 24px;
  border-bottom: 1px solid var(--border);
  display: flex; align-items: center; gap: 24px; flex-wrap: wrap;
}
.verdict-badge {
  padding: 10px 20px; border-radius: 3px;
  font-family: var(--mono); font-size: 0.9rem; font-weight: 600; letter-spacing: 0.1em;
}
.verdict-badge.fake { background: rgba(239,68,68,0.15); color: var(--fake); border: 1px solid rgba(239,68,68,0.3); }
.verdict-badge.warn { background: rgba(245,158,11,0.15); color: var(--warn); border: 1px solid rgba(245,158,11,0.3); }
.verdict-badge.real { background: rgba(34,197,94,0.15); color: var(--real); border: 1px solid rgba(34,197,94,0.3); }

.verdict-info { flex: 1; min-width: 200px; }
.verdict-title { font-size: 1.5rem; font-weight: 800; margin-bottom: 4px; }
.verdict-title.fake { color: var(--fake); }
.verdict-title.warn { color: var(--warn); }
.verdict-title.real { color: var(--real); }
.verdict-sub { font-family: var(--mono); font-size: 0.72rem; color: var(--muted); }

.confidence-ring {
  position: relative; width: 72px; height: 72px; flex-shrink: 0;
}
.confidence-ring svg { transform: rotate(-90deg); }
.confidence-ring .ring-bg { fill: none; stroke: var(--border); stroke-width: 5; }
.confidence-ring .ring-fill { fill: none; stroke-width: 5; stroke-linecap: round; transition: stroke-dashoffset 0.8s ease; }
.confidence-ring .ring-text {
  position: absolute; inset: 0; display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  font-family: var(--mono); font-size: 0.75rem; font-weight: 600;
}
.confidence-ring .ring-pct { font-size: 1rem; font-weight: 700; }

/* Metrics grid */
.metrics-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(160px, 1fr)); gap: 1px; background: var(--border); }
.metric-cell { background: var(--surface); padding: 20px 20px 18px; }
.metric-label { font-family: var(--mono); font-size: 0.62rem; color: var(--muted); letter-spacing: 0.1em; text-transform: uppercase; margin-bottom: 10px; }
.metric-value { font-size: 1.5rem; font-weight: 800; margin-bottom: 8px; }
.metric-bar { height: 3px; background: var(--border); border-radius: 2px; }
.metric-fill { height: 100%; border-radius: 2px; transition: width 0.7s ease; }

/* Signal tags */
.tags-section { padding: 20px 28px; border-top: 1px solid var(--border); }
.tags-label { font-family: var(--mono); font-size: 0.62rem; color: var(--muted); letter-spacing: 0.1em; text-transform: uppercase; margin-bottom: 10px; }
.tags { display: flex; flex-wrap: wrap; gap: 6px; }
.tag {
  padding: 4px 10px; border-radius: 2px;
  font-family: var(--mono); font-size: 0.68rem;
}
.tag-bad { background: rgba(239,68,68,0.12); color: #fca5a5; border: 1px solid rgba(239,68,68,0.2); }
.tag-good { background: rgba(34,197,94,0.12); color: #86efac; border: 1px solid rgba(34,197,94,0.2); }
.tag-neutral { background: rgba(255,255,255,0.05); color: var(--muted); border: 1px solid var(--border); }

/* AI section */
.ai-section { padding: 24px 28px; border-top: 1px solid var(--border); }
.ai-header { display: flex; align-items: center; gap: 10px; margin-bottom: 16px; }
.ai-dot { width: 8px; height: 8px; border-radius: 50%; background: var(--accent); animation: blink 1.2s infinite; }
.ai-label { font-family: var(--mono); font-size: 0.68rem; color: var(--accent); letter-spacing: 0.1em; text-transform: uppercase; }
.ai-text { font-family: var(--mono); font-size: 0.8rem; line-height: 1.75; color: rgba(232,234,240,0.8); white-space: pre-wrap; }
.ai-loading { display: flex; align-items: center; gap: 10px; font-family: var(--mono); font-size: 0.78rem; color: var(--muted); }
.spinner { width: 16px; height: 16px; border: 2px solid var(--border); border-top-color: var(--accent); border-radius: 50%; animation: spin 0.7s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }

/* Examples */
.examples-section { margin-top: 32px; }
.examples-label { font-family: var(--mono); font-size: 0.68rem; color: var(--muted); letter-spacing: 0.1em; text-transform: uppercase; margin-bottom: 14px; }
.example-cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(260px, 1fr)); gap: 12px; }
.example-card {
  background: var(--surface); border: 1px solid var(--border); border-radius: 4px;
  padding: 14px 16px; cursor: pointer; transition: all 0.15s;
}
.example-card:hover { border-color: rgba(110,231,247,0.3); background: var(--surface2); }
.example-type { font-family: var(--mono); font-size: 0.62rem; letter-spacing: 0.1em; text-transform: uppercase; margin-bottom: 6px; }
.example-type.fake { color: var(--fake); }
.example-type.real { color: var(--real); }
.example-text { font-size: 0.78rem; color: var(--muted); line-height: 1.6; display: -webkit-box; -webkit-line-clamp: 3; -webkit-box-orient: vertical; overflow: hidden; }

/* How it works */
.how-section { margin-top: 48px; border-top: 1px solid var(--border); padding-top: 40px; }
.how-title { font-size: 1.2rem; font-weight: 800; margin-bottom: 24px; }
.how-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 16px; }
.how-card { background: var(--surface); border: 1px solid var(--border); border-radius: 4px; padding: 18px; }
.how-icon { font-size: 1.5rem; margin-bottom: 10px; }
.how-name { font-size: 0.82rem; font-weight: 700; margin-bottom: 6px; }
.how-desc { font-family: var(--mono); font-size: 0.7rem; color: var(--muted); line-height: 1.6; }
`;

const EXAMPLES = [
  {
    type: "fake",
    label: "Likely Fake",
    text: "SHOCKING: Scientists BANNED from revealing the TRUTH about 5G towers!! Big Pharma and Deep State covering up MIRACLE cure that DESTROYS cancer in 24 hours!! They don't want you to know this SECRET!! Share before it's DELETED!! Wake up sheeple!!!",
  },
  {
    type: "real",
    label: "Likely Real",
    text: "According to a new study published in the Journal of Epidemiology, researchers indicate that regular exercise may reduce the risk of cardiovascular disease by up to 30%. Data suggests that even moderate activity, such as a 30-minute daily walk, provides measurable health benefits. Officials from the WHO confirmed these findings align with existing public health guidelines.",
  },
  {
    type: "fake",
    label: "Suspicious",
    text: "Everyone knows the mainstream media is hiding the real unemployment numbers. I believe the actual figure is much higher — clearly the government is lying to us. They always do this before elections. This is obviously a cover-up, no one can deny it anymore.",
  },
];

const HOW_IT_WORKS = [
  { icon: "🔍", name: "Sensationalism Detection", desc: "Scans for loaded trigger words commonly found in misinformation: 'shocking', 'cover-up', 'banned', 'deep state', etc." },
  { icon: "✅", name: "Credibility Signals", desc: "Rewards journalistic language: 'according to', 'study shows', 'officials said', 'peer-reviewed', 'data suggests'." },
  { icon: "🎭", name: "Emotional Bias Analysis", desc: "Detects opinion markers and absolutist language that replace factual reporting with emotional manipulation." },
  { icon: "📊", name: "Feature Vector Scoring", desc: "Combines 6 weighted heuristic features into a composite fake-score using a simple classifier model." },
  { icon: "🤖", name: "LLM Deep Analysis", desc: "Claude AI provides nuanced reasoning, context, and fact-checking insights beyond rule-based signals." },
  { icon: "🎯", name: "Confidence Scoring", desc: "Final verdict presented with a confidence percentage based on combined signal strength." },
];

// ── Main Component ─────────────────────────────────────────────────────────────
export default function App() {
  const [text, setText] = useState("");
  const [result, setResult] = useState(null);
  const [aiAnalysis, setAiAnalysis] = useState("");
  const [aiLoading, setAiLoading] = useState(false);
  const [analyzing, setAnalyzing] = useState(false);
  const [mode, setMode] = useState("heuristic"); // heuristic | ai

  const circumference = 2 * Math.PI * 30;

  const getVerdictClass = (verdict) => {
    if (!verdict) return "";
    if (verdict === "LIKELY FAKE") return "fake";
    if (verdict === "SUSPICIOUS") return "warn";
    return "real";
  };

  const getMetricColor = (val, invert = false) => {
    const v = invert ? 100 - val : val;
    if (v >= 70) return "#ef4444";
    if (v >= 40) return "#f59e0b";
    return "#22c55e";
  };

  const runHeuristic = () => {
    const r = analyzeText(text);
    setResult(r);
    setAiAnalysis("");
  };

  const runAI = async () => {
    const heuristic = analyzeText(text);
    setResult(heuristic);
    setAiLoading(true);
    setAiAnalysis("");

    try {
      const response = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          system: `You are a professional fact-checker and misinformation analyst. Analyze the given news text and provide a detailed, structured assessment. Be concise and direct. Format your response exactly like this:

VERDICT: [LIKELY FAKE / SUSPICIOUS / LIKELY REAL]
CONFIDENCE: [0-100]%

KEY FINDINGS:
• [Finding 1]
• [Finding 2]
• [Finding 3]

RED FLAGS: [List any misinformation indicators, or "None detected"]

CREDIBLE ELEMENTS: [List any credible signals, or "None detected"]

RECOMMENDATION: [One sentence on what the reader should do]`,
          messages: [{ role: "user", content: `Analyze this news article/headline for misinformation:\n\n"${text}"` }],
        }),
      });
      const data = await response.json();
      const aiText = data.content?.map(b => b.text || "").join("") || "Analysis unavailable.";
      setAiAnalysis(aiText);
    } catch (e) {
      setAiAnalysis("AI analysis failed. Please check your connection and try again.");
    }
    setAiLoading(false);
  };

  const handleAnalyze = async () => {
    if (!text.trim() || text.trim().length < 20) return;
    setAnalyzing(true);
    if (mode === "heuristic") {
      runHeuristic();
      setTimeout(() => setAnalyzing(false), 300);
    } else {
      await runAI();
      setAnalyzing(false);
    }
  };

  const loadExample = (ex) => {
    setText(ex.text);
    setResult(null);
    setAiAnalysis("");
  };

  const vClass = result ? getVerdictClass(result.verdict) : "";
  const ringOffset = result ? circumference - (result.confidence / 100) * circumference : circumference;
  const ringColor = vClass === "fake" ? "#ef4444" : vClass === "warn" ? "#f59e0b" : "#22c55e";

  return (
    <>
      <style>{CSS}</style>
      <div className="app">
        <div className="grid-bg" />
        <div className="scanline" />

        <div className="content">
          <header className="header">
            <div className="header-tag">
              <span className="dot" />
              AI System · v2.1 · Active
            </div>
            <h1>Fake News<br /><em>Detector</em></h1>
            <p>
              NLP-powered misinformation analysis using heuristic feature extraction<br />
              + large language model reasoning. Paste any headline or article below.
            </p>
          </header>

          {/* Mode tabs */}
          <div className="tabs">
            <button className={`tab ${mode === "heuristic" ? "active" : ""}`} onClick={() => setMode("heuristic")}>
              Heuristic Analysis
            </button>
            <button className={`tab ${mode === "ai" ? "active" : ""}`} onClick={() => setMode("ai")}>
              AI Deep Analysis
            </button>
          </div>

          {/* Input */}
          <div className="input-section">
            <label className="input-label">// paste news text or headline</label>
            <textarea
              value={text}
              onChange={e => setText(e.target.value)}
              placeholder="Enter a news headline, paragraph, or 
