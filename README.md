from pathlib import Path
import zipfile, os, json

base_dir = Path("/mnt/data/aiva_ai_mobile_futurista")
if base_dir.exists():
    # remove existing to avoid stale content
    import shutil
    shutil.rmtree(base_dir)
base_dir.mkdir(parents=True, exist_ok=True)
src = base_dir / "src"
src.mkdir()

files = {
    "package.json": json.dumps({
      "name": "aiva-ai-mobile",
      "version": "1.0.0",
      "private": True,
      "scripts": {
        "dev": "vite",
        "build": "vite build",
        "preview": "vite preview"
      },
      "dependencies": {
        "react": "^18.2.0",
        "react-dom": "^18.2.0"
      },
      "devDependencies": {
        "vite": "^5.0.0"
      }
    }, indent=2, ensure_ascii=False),
    "index.html": """<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>AIVA AI</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>""",
    "src/main.jsx": """import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './styles.css'

createRoot(document.getElementById('root')).render(<App />)
""",
    "src/App.jsx": r"""import React, {useEffect, useState, useRef} from 'react'

const DEFAULT_WELCOME = `Bem-vindo à sua AIVA, Márcio — seu mundo de performance e prosperidade.`

function nowTimeStr(){
  return new Date().toLocaleString()
}

export default function App(){
  const [messages, setMessages] = useState(() => {
    const raw = localStorage.getItem('aiva_messages')
    return raw ? JSON.parse(raw) : [{from:'aiva', text: DEFAULT_WELCOME}]
  })
  const [input, setInput] = useState('')
  const [notes, setNotes] = useState(() => JSON.parse(localStorage.getItem('aiva_notes')||'[]'))
  const [alarms, setAlarms] = useState(() => JSON.parse(localStorage.getItem('aiva_alarms')||'[]'))
  const [finance, setFinance] = useState({usdbrl:'-', btc:'-'})
  const [listening, setListening] = useState(false)
  const recognitionRef = useRef(null)
  const msgsRef = useRef(null)

  useEffect(()=>{ localStorage.setItem('aiva_messages', JSON.stringify(messages)) }, [messages])
  useEffect(()=>{ localStorage.setItem('aiva_notes', JSON.stringify(notes)) }, [notes])
  useEffect(()=>{ localStorage.setItem('aiva_alarms', JSON.stringify(alarms)) }, [alarms])

  // simple polling for indicators (public endpoints)
  useEffect(()=>{
    async function fetchIndicators(){
      try{
        const btcRes = await fetch('https://api.coindesk.com/v1/bpi/currentprice/BTC.json')
        const btcJson = await btcRes.json()
        const btcPrice = btcJson?.bpi?.USD?.rate_float ? btcJson.bpi.USD.rate_float : '-'
        const exRes = await fetch('https://api.exchangerate.host/latest?base=USD&symbols=BRL')
        const exJson = await exRes.json()
        const usdbrl = exJson?.rates?.BRL ? exJson.rates.BRL : '-'
        setFinance({usdbrl, btc: btcPrice})
        addSystemMessage(`📈 Atualização de mercado (${nowTimeStr()}): USD-BRL ${usdbrl} • BTC (USD) ${btcPrice}`)
      }catch(e){
        console.warn('Erro ao buscar indicadores', e)
      }
    }
    fetchIndicators()
    const iv = setInterval(fetchIndicators, 1000 * 60 * 5) // a cada 5 minutos
    return ()=>clearInterval(iv)
  }, [])

  // Alarm checker
  useEffect(()=>{
    const iv = setInterval(()=>{
      const now = Date.now()
      const due = alarms.filter(a=>!a.triggered && new Date(a.time).getTime() <= now)
      due.forEach(a=>{
        triggerAlarm(a)
      })
    }, 1000)
    return ()=>clearInterval(iv)
  }, [alarms])

  useEffect(()=>{
    // prepare speech recognition if available
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
    if(SpeechRecognition){
      const r = new SpeechRecognition()
      r.continuous = false
      r.lang = 'pt-BR'
      r.interimResults = false
      r.onresult = (ev) => {
        const t = ev.results[0][0].transcript
        handleVoiceTranscript(t)
        setListening(false)
      }
      r.onerror = (e)=>{
        console.warn('Speech error', e)
        setListening(false)
      }
      recognitionRef.current = r
    }
  }, [])

  function addSystemMessage(text){
    setMessages(m=>[...m, {from:'aiva', text, time: nowTimeStr()}])
    scrollBottom()
  }

  function scrollBottom(){
    setTimeout(()=>{ if(msgsRef.current) msgsRef.current.scrollTop = msgsRef.current.scrollHeight }, 100)
  }

  function handleSend(){
    if(!input.trim()) return
    const userMsg = {from:'user', text: input, time: nowTimeStr()}
    setMessages(m=>[...m, userMsg])
    setInput('')
    setTimeout(()=>processUserMessage(userMsg.text), 300)
  }

  function processUserMessage(text){
    const low = text.toLowerCase()
    if(low.startsWith('aiva,') || low.startsWith('aiva ')){
      // strip name
      const cmd = text.replace(/^[Aa]iva[,:]?\s*/i, '')
      runCommand(cmd)
    } else {
      // assume general chat
      runCommand(text)
    }
  }

  function runCommand(cmd){
    // simple command parser
    const low = cmd.toLowerCase()
    if(low.startsWith('anote') || low.startsWith('nota') || low.startsWith('anotar')){
      const note = cmd.replace(/^(anote|nota|anotar)[:\\s]*/i,'')
      if(note.trim()){ setNotes(n=>[...n, {text:note, time: nowTimeStr()}]); addSystemMessage('✔️ Nota salva.') }
      else addSystemMessage('Diga o texto da nota após o comando.')
      return
    }
    if(low.startsWith('registr') && low.match(/\\d+/)){
      // simple finance register: \"registrar venda 250\"
      const v = cmd.match(/(-?\\d+[\\.,]?\\d*)/)
      if(v){ addSystemMessage(`Registro financeiro salvo: R$ ${v[0]}`); return }
    }
    if(low.includes('mostrar notas') || low.includes('minhas notas')){
      const lines = notes.map((n,i)=>`${i+1}. ${n.text} (${n.time})`).join('\\n')||'Sem notas.'
      addSystemMessage(`🗒️ Suas notas:\\n${lines}`)
      return
    }
    if(low.includes('alarme') || low.includes('acorde')){
      // parse time like 6:30 or 07:00
      const m = cmd.match(/(\\d{1,2}:\\d{2})/)
      if(m){
        const t = m[1]
        const today = new Date()
        const [hh,mm] = t.split(':').map(x=>parseInt(x,10))
        const dt = new Date(today.getFullYear(), today.getMonth(), today.getDate(), hh, mm)
        if(dt.getTime() <= Date.now()) dt.setDate(dt.getDate()+1)
        const alarm = {id:Date.now(), time: dt.toISOString(), msg: `Bom dia, Márcio 💪 — hora de construir sua mentalidade bilionária!`, triggered:false}
        setAlarms(a=>[...a, alarm])
        addSystemMessage(`⏰ Alarme definido para ${dt.toLocaleString()}`)
      } else addSystemMessage('Diga o horário no formato HH:MM. Ex: \"AIVA, acorde às 06:30\"')
      return
    }
    if(low.includes('resumo do dia') || low.includes('resumo')){
      const s = `Resumo (${nowTimeStr()}):\\nMercado: USD-BRL ${finance.usdbrl} • BTC ${finance.btc}\\nNotas: ${notes.length}\\nPróximos alarmes: ${alarms.filter(a=>!a.triggered).length}`
      addSystemMessage(s)
      return
    }

    // default fallback small talk / reflection
    if(low.includes('reflex') || low.includes('mentalidade') || low.includes('bilion')){
      addSystemMessage('💭 Reflexão do dia: Disciplina + Consistência = Capital.')
      return
    }

    // fallback
    addSystemMessage('Entendi. Posso anotar isso ou transformar em lembrete. Tente começar o comando com \"AIVA, ...\" para ações rápidas.')
  }

  function triggerAlarm(a){
    // mark triggered
    setAlarms(prev=>prev.map(x=> x.id===a.id?{...x, triggered:true}:x))
    const text = a.msg || 'Alarme AIVA'
    addSystemMessage(`🔔 ${text}`)
    // try Notification API
    if(window.Notification && Notification.permission === 'granted'){
      new Notification('AIVA', {body:text})
    }
    // play beep
    const ctx = new (window.AudioContext || window.webkitAudioContext)()
    const o = ctx.createOscillator(); const g = ctx.createGain();
    o.type = 'sine'; o.frequency.value = 880; g.gain.value = 0.1; o.connect(g); g.connect(ctx.destination); o.start();
    setTimeout(()=>{ o.stop(); ctx.close() }, 1200)
  }

  function handleVoiceTranscript(t){
    addSystemMessage(`(áudio) ${t}`)
    processUserMessage(t)
  }

  function startListening(){
    if(!recognitionRef.current){ alert('Seu navegador não suporta reconhecimento de voz. Você pode usar envio de áudio.'); return }
    setListening(true)
    recognitionRef.current.start()
  }

  return (
    <div className="app-root">
      <header className="topbar">
        <h1>AIVA AI</h1>
        <div className="status">{new Date().toLocaleTimeString()}</div>
      </header>
      <main>
        <section className="left">
          <div className="panel">
            <h2>Painel</h2>
            <div className="indicators">
              <div>USD-BRL: {finance.usdbrl}</div>
              <div>BTC (USD): {finance.btc}</div>
            </div>
            <div className="next">
              <h3>Próximos alarmes</h3>
              <ul>
                {alarms.filter(a=>!a.triggered).map(a=> <li key={a.id}>{new Date(a.time).toLocaleString()} — {a.msg}</li>)}
                {alarms.filter(a=>!a.triggered).length===0 && <li>Sem alarmes</li>}
              </ul>
            </div>
            <div className="notes">
              <h3>Notas rápidas</h3>
              <ul>
                {notes.map((n,i)=> <li key={i}>{n.text} <small>({n.time})</small></li>)}
                {notes.length===0 && <li>Sem notas</li>}
              </ul>
            </div>
          </div>
        </section>

        <section className="chat">
          <div className="msgs" ref={msgsRef}>
            {messages.map((m,i)=> (
              <div key={i} className={`msg ${m.from==='aiva'?'aiva':'user'}`}>
                <div className="mtext">{m.text}</div>
                <div className="mtime">{m.time||''}</div>
              </div>
            ))}
          </div>

          <div className="composer">
            <input value={input} onChange={e=>setInput(e.target.value)} placeholder="Diga 'AIVA, ...' para comandos rápidos" onKeyDown={e=> e.key==='Enter' && handleSend()} />
            <button onClick={handleSend}>Enviar</button>
            <button onClick={()=>{ if(listening) { recognitionRef.current && recognitionRef.current.stop(); setListening(false)} else startListening() }}>
              {listening? 'Ouvindo...':'Ouvir (AIVA...)'}</button>
            <button onClick={()=>{ if(window.Notification && Notification.permission !== 'granted') Notification.requestPermission().then(()=>alert('Permissão de notificações pedida.')) }}>Ativar Notif</button>
          </div>
        </section>
      </main>

      <footer className="footer">{DEFAULT_WELCOME}</footer>
    </div>
  )
}""",
    "src/styles.css": """:root{
  --bg:#0b0b0d; --panel:#0f1113; --muted:#9aa4ad; --accent:#cfa12b; --accent2:#1e90ff;
}
*{box-sizing:border-box}
body{margin:0;font-family:Inter,Arial,Helvetica,sans-serif;background:var(--bg);color:#e6eef6}
.app-root{display:flex;flex-direction:column;height:100vh}
.topbar{display:flex;justify-content:space-between;align-items:center;padding:12px 16px;background:linear-gradient(90deg,#071018, #0b0b0d);border-bottom:1px solid rgba(255,255,255,0.03)}
.topbar h1{margin:0;font-size:18px;letter-spacing:1px}
.main{display:flex;flex:1}
main{display:flex;flex:1;overflow:hidden}
.left{width:36%;padding:12px}
.panel{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(0,0,0,0.02));padding:12px;border-radius:12px}
.chat{flex:1;display:flex;flex-direction:column;padding:12px}
.msgs{flex:1;overflow:auto;padding:8px}
.msg{margin-bottom:10px;max-width:75%}
.msg.aiva{align-self:flex-start;background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));padding:10px;border-radius:10px;color:#e6eef6}
.msg.user{align-self:flex-end;background:#17202a;padding:10px;border-radius:10px}
.composer{display:flex;gap:8px;align-items:center;padding-top:8px}
.composer input{flex:1;padding:10px;border-radius:10px;border:1px solid rgba(255,255,255,0.03);background:transparent;color:#e6eef6}
.composer button{padding:10px 12px;border-radius:8px;border:none;background:var(--accent);color:#0b0b0d}
.indicators{display:flex;gap:12px}
footer.footer{padding:8px 12px;text-align:center;color:var(--muted);border-top:1px solid rgba(255,255,255,0.02)}""",
    "README.md": """# AIVA AI — Mobile (Fase 1 Plus)\n\nVersão mobile otimizada (tema escuro) para deploy na Vercel.\n\n## Instruções rápidas\n1. Faça o download do arquivo ZIP.\n2. No GitHub, vá ao repositório `AIVA-AI` e clique em Add file → Upload files.\n3. Envie todo o conteúdo do ZIP (ou faça unzip e envie os arquivos).\n4. Volte à Vercel e importe o repositório; faça deploy com Framework: Vite.\n\n## Recursos\n- Despertador via Notification API\n- Reconhecimento de voz no navegador (SpeechRecognition)\n- Indicadores básicos: BTC (CoinDesk) e USD-BRL (exchangerate.host)\n- Notas rápidas e painel\n"""
}

# write files
for rel, content in files.items():
    path = base_dir / rel
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(content, encoding='utf-8')

# create zip
zip_path = Path("/mnt/data/AIVA_AI_Mobile_Futurista.zip")
if zip_path.exists():
    zip_path.unlink()
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as zf:
    for root, dirs, filenames in os.walk(base_dir):
        for fname in filenames:
            full = os.path.join(root, fname)
            arcname = os.path.relpath(full, base_dir)
            zf.write(full, arcname)

str(zip_path)
