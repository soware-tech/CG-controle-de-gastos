import React, { useState, useEffect, useMemo, useRef } from 'react';
import { 
  PlusCircle, 
  Trash2, 
  ChevronLeft, 
  ChevronRight, 
  ScanQrCode, 
  Loader2, 
  Sparkles, 
  Camera, 
  Image as ImageIcon, 
  CheckCircle2,
  TrendingUp,
  Users,
  X,
  Maximize,
  UserPlus,
  UserMinus,
  Settings,
  CreditCard,
  Receipt,
  Wallet,
  CalendarDays,
  BarChart3,
  Files,
  Save,
  RotateCcw,
  Tag
} from 'lucide-react';

const apiKey = "";

const App = () => {
  const [activeTab, setActiveTab] = useState('lancamentos');
  const [currentMonth, setCurrentMonth] = useState(new Date().getMonth());
  const [currentYear, setCurrentYear] = useState(new Date().getFullYear());
  const [isScanning, setIsScanning] = useState(false);
  const [scanInput, setScanInput] = useState('');
  const [feedback, setFeedback] = useState(null);
  const [showScanner, setShowScanner] = useState(false);
  const [showConfigAdmin, setShowConfigAdmin] = useState(false);
  const [newUser, setNewUser] = useState('');
  const [newCategory, setNewCategory] = useState('');
  
  const [pendingExpenses, setPendingExpenses] = useState([]);
  const [expenses, setExpenses] = useState([]);
  const [users, setUsers] = useState([
    { id: '1', name: 'Aline', cards: ['1234'] },
    { id: '2', name: 'Roney', cards: ['5678'] }
  ]);

  const [categories, setCategories] = useState([
    'Aluguel', 'Beleza', 'Casa', 'Combustível', 'Energia', 'Estacionamento', 
    'Manutenção Carro', 'Feira', 'Fast Food', 'Delivery', 'Gás', 
    'Gastos Médicos', 'Farmácia', 'IPVA', 'Licenciamentos', 'Internet', 
    'Celular', 'Mercado', 'Padaria', 'Pets', 'Presentes', 'Seguros', 
    'Streamings', 'Vestuários', 'Variedades', 'Viagens', 'Gastos Lilie'
  ]);

  const monthNames = ["Janeiro", "Fevereiro", "Março", "Abril", "Maio", "Junho", "Julho", "Agosto", "Setembro", "Outubro", "Novembro", "Dezembro"];

  const [formData, setFormData] = useState({
    user: 'Aline',
    description: '',
    category: 'Mercado',
    amount: '',
    date: new Date().toISOString().split('T')[0]
  });

  const fileInputRef = useRef(null);
  const videoRef = useRef(null);
  const canvasRef = useRef(null);

  useEffect(() => {
    const savedExpenses = localStorage.getItem('expenses_v9');
    const savedUsers = localStorage.getItem('users_v9');
    const savedCategories = localStorage.getItem('categories_v9');
    if (savedExpenses) setExpenses(JSON.parse(savedExpenses));
    if (savedUsers) setUsers(JSON.parse(savedUsers));
    if (savedCategories) setCategories(JSON.parse(savedCategories));
  }, []);

  useEffect(() => {
    localStorage.setItem('expenses_v9', JSON.stringify(expenses));
    localStorage.setItem('users_v9', JSON.stringify(users));
    localStorage.setItem('categories_v9', JSON.stringify(categories));
  }, [expenses, users, categories]);

  // --- IA & PROCESSAMENTO ---

  const callGemini = async (prompt, imageData = null) => {
    try {
      const userCardsInfo = users.map(u => `${u.name}: cartões final ${u.cards.join(', ')}`).join('; ');
      const systemPrompt = `Você é um assistente de finanças. Use estes dados para identificar quem pagou: ${userCardsInfo}. Retorne sempre JSON puro. Se não identificar o usuário, retorne null no campo usuario. Categorias disponíveis: ${categories.join(', ')}`;
      const payload = {
        contents: [{ parts: [{ text: prompt }, ...(imageData ? [{ inlineData: { mimeType: "image/png", data: imageData } }] : [])] }],
        systemInstruction: { parts: [{ text: systemPrompt }] },
        generationConfig: { responseMimeType: "application/json" }
      };
      const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      const result = await res.json();
      return JSON.parse(result.candidates?.[0]?.content?.parts?.[0]?.text || "null");
    } catch (error) { return null; }
  };

  // --- FUNÇÕES DE CÂMERA (SCANNER) ---

  const startCamera = async () => {
    setShowScanner(true);
    setTimeout(async () => {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
        if (videoRef.current) {
          videoRef.current.srcObject = stream;
        }
      } catch (err) {
        setFeedback("Erro ao acessar câmera");
        setShowScanner(false);
      }
    }, 100);
  };

  const stopCamera = () => {
    if (videoRef.current && videoRef.current.srcObject) {
      const tracks = videoRef.current.srcObject.getTracks();
      tracks.forEach(track => track.stop());
    }
    setShowScanner(false);
  };

  const capturePhoto = async () => {
    if (!videoRef.current || !canvasRef.current) return;
    const context = canvasRef.current.getContext('2d');
    canvasRef.current.width = videoRef.current.videoWidth;
    canvasRef.current.height = videoRef.current.videoHeight;
    context.drawImage(videoRef.current, 0, 0);
    const base64Data = canvasRef.current.toDataURL('image/png').split(',')[1];
    
    stopCamera();
    setIsScanning(true);
    setFeedback("Analisando foto...");

    const data = await callGemini("Extraia os dados deste cupom fiscal. JSON: {\"estabelecimento\": \"nome\", \"valor\": 0.00, \"usuario\": \"nome\", \"categoria\": \"sugerir_uma\"}", base64Data);
    
    if (data) {
      processAIResult(data);
    } else {
      setFeedback("Não foi possível ler o cupom");
    }
    setIsScanning(false);
  };

  const processAIResult = (data) => {
    const newPending = {
      id: Math.random().toString(36).substr(2, 9),
      user: data.usuario || users[0].name,
      description: data.estabelecimento || "Cupom Escaneado",
      category: categories.includes(data.categoria) ? data.categoria : (categories[0] || "Geral"),
      amount: data.valor || 0,
      date: new Date().toISOString().split('T')[0]
    };
    setPendingExpenses(prev => [newPending, ...prev]);
    setFeedback("Cupom identificado!");
    setTimeout(() => setFeedback(null), 3000);
  };

  const handleImageUpload = async (e) => {
    const files = Array.from(e.target.files).slice(0, 5);
    if (files.length === 0) return;

    setIsScanning(true);
    setFeedback(`Processando ${files.length} arquivo(s)...`);

    for (const file of files) {
      const base64Data = await new Promise((resolve) => {
        const reader = new FileReader();
        reader.onloadend = () => resolve(reader.result.split(',')[1]);
        reader.readAsDataURL(file);
      });

      const data = await callGemini("Extraia os dados deste cupom. JSON: {\"estabelecimento\": \"nome\", \"valor\": 0.00, \"usuario\": \"nome\", \"categoria\": \"sugerir_uma\"}", base64Data);
      if (data) processAIResult(data);
    }
    setIsScanning(false);
    e.target.value = null;
  };

  const savePendingExpense = (id) => {
    const item = pendingExpenses.find(p => p.id === id);
    if (item) {
      const { id: oldId, ...cleanItem } = item;
      setExpenses([ { ...cleanItem, id: Date.now() }, ...expenses]);
      setPendingExpenses(pendingExpenses.filter(p => p.id !== id));
    }
  };

  const saveAllPending = () => {
    const newItems = pendingExpenses.map(p => ({
      ...p,
      id: Date.now() + Math.random(),
      amount: parseFloat(p.amount)
    }));
    setExpenses([...newItems, ...expenses]);
    setPendingExpenses([]);
    setFeedback("Salvar todos!");
    setTimeout(() => setFeedback(null), 2000);
  };

  // --- CÁLCULOS E ESTATÍSTICAS ---

  const filteredExpenses = useMemo(() => {
    return expenses.filter(e => {
      const d = new Date(e.date + 'T00:00:00');
      return d.getMonth() === currentMonth && d.getFullYear() === currentYear;
    });
  }, [expenses, currentMonth, currentYear]);

  const yearlyExpenses = useMemo(() => {
    return expenses.filter(e => {
      const d = new Date(e.date + 'T00:00:00');
      return d.getFullYear() === currentYear;
    });
  }, [expenses, currentYear]);

  const monthlyStats = useMemo(() => {
    const total = filteredExpenses.reduce((acc, curr) => acc + Number(curr.amount), 0);
    const byUser = users.reduce((acc, u) => {
      acc[u.name] = filteredExpenses.filter(e => e.user === u.name).reduce((sum, curr) => sum + Number(curr.amount), 0);
      return acc;
    }, {});
    const byCategory = categories.reduce((acc, cat) => {
      const sum = filteredExpenses.filter(e => e.category === cat).reduce((s, curr) => s + Number(curr.amount), 0);
      if (sum > 0) acc[cat] = sum;
      return acc;
    }, {});
    return { total, byUser, byCategory };
  }, [filteredExpenses, users, categories]);

  const yearlyStats = useMemo(() => {
    const total = yearlyExpenses.reduce((acc, curr) => acc + Number(curr.amount), 0);
    const byUser = users.reduce((acc, u) => {
      acc[u.name] = yearlyExpenses.filter(e => e.user === u.name).reduce((sum, curr) => sum + Number(curr.amount), 0);
      return acc;
    }, {});
    const byMonth = monthNames.map((name, index) => {
      const monthTotal = yearlyExpenses
        .filter(e => new Date(e.date + 'T00:00:00').getMonth() === index)
        .reduce((sum, curr) => sum + Number(curr.amount), 0);
      return { name, total: monthTotal };
    });
    const maxMonth = Math.max(...byMonth.map(m => m.total), 1);
    return { total, byUser, byMonth, maxMonth };
  }, [yearlyExpenses, users]);

  // --- UI ACTIONS ---

  const addUser = () => {
    if (newUser.trim() && !users.find(u => u.name === newUser.trim())) {
      setUsers([...users, { id: Date.now().toString(), name: newUser.trim(), cards: [] }]);
      setNewUser('');
    }
  };

  const removeUser = (id) => {
    if (users.length > 1) setUsers(users.filter(u => u.id !== id));
  };

  const addCategory = () => {
    if (newCategory.trim() && !categories.includes(newCategory.trim())) {
      setCategories([...categories, newCategory.trim()]);
      setNewCategory('');
    }
  };

  const removeCategory = (catName) => {
    if (categories.length > 1) {
      setCategories(categories.filter(c => c !== catName));
    }
  };

  const addCardToUser = (userId, cardNum) => {
    if (cardNum.length === 4) setUsers(users.map(u => u.id === userId ? { ...u, cards: [...u.cards, cardNum] } : u));
  };

  const removeCardFromUser = (userId, idx) => {
    setUsers(users.map(u => u.id === userId ? { ...u, cards: u.cards.filter((_, i) => i !== idx) } : u));
  };

  const handleSubmitManual = (e) => {
    e.preventDefault();
    if (!formData.amount) return;
    setExpenses([{ ...formData, id: Date.now(), amount: parseFloat(formData.amount) }, ...expenses]);
    setFormData({ ...formData, description: '', amount: '' });
  };

  return (
    <div className="min-h-screen bg-[#f8fafc] text-slate-900 pb-20">
      {/* NAVBAR */}
      <nav className="bg-white border-b border-slate-200 sticky top-0 z-20">
        <div className="max-w-6xl mx-auto px-4 flex justify-between items-center h-16">
          <div className="flex items-center gap-2">
            <div className="bg-emerald-600 p-1.5 rounded text-white font-bold text-sm">CG</div>
            <span className="font-bold text-lg text-slate-700 hidden sm:block">Controle de Gastos</span>
          </div>
          <div className="flex bg-slate-100 p-1 rounded-xl">
            {['lancamentos', 'dashboard', 'anual'].map(tab => (
              <button 
                key={tab}
                onClick={() => setActiveTab(tab)}
                className={`px-3 sm:px-5 py-1.5 rounded-lg text-[10px] sm:text-xs font-black transition-all uppercase ${activeTab === tab ? 'bg-white shadow-sm text-emerald-700' : 'text-slate-400'}`}
              >
                {tab === 'lancamentos' ? 'Lançar' : tab === 'dashboard' ? 'Mensal' : 'Anual'}
              </button>
            ))}
          </div>
          <button onClick={() => setShowConfigAdmin(true)} className="p-2 text-slate-400 hover:text-emerald-600"><Settings size={20} /></button>
        </div>
      </nav>

      {/* HEADER DINÂMICO */}
      <div className="bg-emerald-700 text-white py-4 shadow-inner">
        <div className="max-w-5xl mx-auto px-4 flex items-center justify-between">
          <button onClick={() => {
             if(activeTab === 'anual') setCurrentYear(currentYear-1);
             else { let nm = currentMonth-1; let ny = currentYear; if(nm<0){nm=11; ny--;} setCurrentMonth(nm); setCurrentYear(ny); }
          }} className="p-1 hover:bg-emerald-600 rounded-full"><ChevronLeft /></button>
          <div className="text-center">
            {activeTab !== 'anual' && <h2 className="font-bold text-lg">{monthNames[currentMonth]}</h2>}
            <p className={`${activeTab === 'anual' ? 'text-xl font-black' : 'text-[10px] opacity-70'} uppercase tracking-widest`}>{currentYear}</p>
          </div>
          <button onClick={() => {
             if(activeTab === 'anual') setCurrentYear(currentYear+1);
             else { let nm = currentMonth+1; let ny = currentYear; if(nm>11){nm=0; ny++;} setCurrentMonth(nm); setCurrentYear(ny); }
          }} className="p-1 hover:bg-emerald-600 rounded-full"><ChevronRight /></button>
        </div>
      </div>

      <main className="max-w-5xl mx-auto p-4 md:p-6">
        
        {activeTab === 'lancamentos' && (
          <div className="space-y-6">
            
            {/* INPUTS DE CARREGAMENTO INTELIGENTE */}
            <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
              <div className="bg-white border border-emerald-100 rounded-2xl p-4 shadow-sm space-y-3">
                <div className="flex items-center gap-3">
                  <div className="bg-emerald-100 p-2 rounded-xl text-emerald-600"><Camera size={20} /></div>
                  <h4 className="text-xs font-black text-slate-700 uppercase">Scanner ao Vivo</h4>
                </div>
                <button onClick={startCamera} className="w-full bg-emerald-600 text-white font-bold py-2.5 rounded-xl text-xs uppercase flex items-center justify-center gap-2 shadow-sm">
                  Abrir Câmera
                </button>
              </div>

              <div className="bg-white border border-blue-100 rounded-2xl p-4 shadow-sm space-y-3">
                <div className="flex items-center gap-3">
                  <div className="bg-blue-100 p-2 rounded-xl text-blue-600"><Files size={20} /></div>
                  <h4 className="text-xs font-black text-slate-700 uppercase">Upload (Max 5)</h4>
                </div>
                <input type="file" multiple accept="image/*" className="hidden" ref={fileInputRef} onChange={handleImageUpload} />
                <button onClick={() => fileInputRef.current.click()} disabled={isScanning} className="w-full bg-blue-50 border-2 border-dashed border-blue-200 text-blue-700 rounded-xl text-xs font-bold py-2.5 flex items-center justify-center gap-2">
                   {isScanning ? <Loader2 size={16} className="animate-spin" /> : <ImageIcon size={16} />} Galeria
                </button>
              </div>

              <div className="bg-white border border-slate-100 rounded-2xl p-4 shadow-sm space-y-3">
                <div className="flex items-center gap-3">
                  <div className="bg-slate-100 p-2 rounded-xl text-slate-600"><ScanQrCode size={20} /></div>
                  <h4 className="text-xs font-black text-slate-700 uppercase">Link de Nota</h4>
                </div>
                <div className="flex gap-2">
                  <input type="text" placeholder="Cole o link..." className="flex-1 bg-slate-50 border-slate-100 rounded-lg text-xs p-2" value={scanInput} onChange={(e) => setScanInput(e.target.value)}/>
                  <button onClick={async () => {
                    setIsScanning(true);
                    const data = await callGemini(`Analise o link e extraia dados. JSON: {"estabelecimento": "nome", "valor": 0.0, "usuario": "nome"}`, null);
                    if(data) processAIResult(data);
                    setScanInput('');
                    setIsScanning(false);
                  }} disabled={isScanning} className="bg-slate-800 text-white px-3 py-2 rounded-lg font-bold text-xs">
                    {isScanning ? <Loader2 size={16} className="animate-spin" /> : 'PROCESSAR'}
                  </button>
                </div>
              </div>
            </div>

            {/* LISTA DE PENDENTES (IA) */}
            {pendingExpenses.length > 0 && (
              <div className="bg-amber-50 border border-amber-200 rounded-2xl overflow-hidden shadow-md">
                <div className="p-4 bg-amber-100/50 flex justify-between items-center border-b border-amber-200">
                  <h3 className="text-xs font-black text-amber-800 uppercase flex items-center gap-2">
                    <Sparkles size={14}/> Validar Lançamentos IA ({pendingExpenses.length})
                  </h3>
                  <button onClick={saveAllPending} className="bg-amber-600 text-white px-3 py-1 rounded-lg text-[10px] font-bold uppercase flex items-center gap-1">
                    <Save size={12}/> Salvar Todos
                  </button>
                </div>
                <div className="p-2 space-y-2">
                  {pendingExpenses.map(p => (
                    <div key={p.id} className="bg-white p-3 rounded-xl border border-amber-100 grid grid-cols-1 md:grid-cols-6 gap-2 items-center">
                      <select className="text-[10px] font-bold border-slate-100 rounded bg-slate-50 uppercase" value={p.user} onChange={e => setPendingExpenses(pendingExpenses.map(x => x.id === p.id ? {...x, user: e.target.value} : x))}>
                        {users.map(u => <option key={u.id} value={u.name}>{u.name}</option>)}
                      </select>
                      <input className="text-xs border-slate-100 rounded col-span-2" value={p.description} onChange={e => setPendingExpenses(pendingExpenses.map(x => x.id === p.id ? {...x, description: e.target.value} : x))}/>
                      <select className="text-[10px] border-slate-100 rounded bg-slate-50" value={p.category} onChange={e => setPendingExpenses(pendingExpenses.map(x => x.id === p.id ? {...x, category: e.target.value} : x))}>
                        {categories.map(c => <option key={c} value={c}>{c}</option>)}
                      </select>
                      <div className="font-black text-xs text-right">R$ {parseFloat(p.amount).toFixed(2)}</div>
                      <div className="flex justify-end gap-2">
                        <button onClick={() => setPendingExpenses(pendingExpenses.filter(x => x.id !== p.id))} className="p-1.5 text-slate-300 hover:text-red-500"><Trash2 size={14}/></button>
                        <button onClick={() => savePendingExpense(p.id)} className="p-1.5 text-emerald-500 hover:bg-emerald-50 rounded"><CheckCircle2 size={16}/></button>
                      </div>
                    </div>
                  ))}
                </div>
              </div>
            )}

            {/* LANÇAMENTO MANUAL */}
            <form onSubmit={handleSubmitManual} className="bg-white rounded-2xl shadow-sm border border-slate-200 p-6 grid grid-cols-1 md:grid-cols-5 gap-4 items-end">
                <div className="space-y-1">
                  <label className="text-[10px] font-bold text-slate-400 uppercase">Usuário</label>
                  <select className="w-full bg-slate-50 border-slate-200 rounded-lg text-sm" value={formData.user} onChange={(e) => setFormData({...formData, user: e.target.value})}>
                    {users.map(u => <option key={u.id} value={u.name}>{u.name}</option>)}
                  </select>
                </div>
                <div className="space-y-1">
                  <label className="text-[10px] font-bold text-slate-400 uppercase">Categoria</label>
                  <select className="w-full bg-slate-50 border-slate-200 rounded-lg text-sm" value={formData.category} onChange={(e) => setFormData({...formData, category: e.target.value})}>
                    {categories.map(c => <option key={c} value={c}>{c}</option>)}
                  </select>
                </div>
                <div className="space-y-1">
                  <label className="text-[10px] font-bold text-slate-400 uppercase">Local</label>
                  <input type="text" placeholder="Ex: Mercado Livre" className="w-full bg-slate-50 border-slate-200 rounded-lg text-sm" value={formData.description} onChange={(e) => setFormData({...formData, description: e.target.value})}/>
                </div>
                <div className="space-y-1">
                  <label className="text-[10px] font-bold text-slate-400 uppercase">Valor R$</label>
                  <input type="number" step="0.01" className="w-full border-emerald-200 rounded-lg text-sm font-bold" value={formData.amount} onChange={(e) => setFormData({...formData, amount: e.target.value})}/>
                </div>
                <button type="submit" className="bg-emerald-600 text-white font-bold py-2 rounded-lg text-sm uppercase shadow-md hover:bg-emerald-700 transition-colors">Salvar</button>
            </form>

            {/* TABELA DE GASTOS DO MÊS */}
            <div className="bg-white rounded-2xl shadow-sm border border-slate-200 overflow-hidden">
              <div className="px-6 py-4 border-b border-slate-100 flex justify-between items-center">
                <h3 className="text-xs font-black text-slate-500 uppercase tracking-widest">Registos de {monthNames[currentMonth]}</h3>
              </div>
              <div className="overflow-x-auto">
                <table className="w-full text-left">
                  <thead className="bg-slate-50">
                    <tr>
                      <th className="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Data</th>
                      <th className="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Usuário</th>
                      <th className="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Local/Cat</th>
                      <th className="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase text-right">Valor</th>
                      <th className="px-6 py-3 w-10"></th>
                    </tr>
                  </thead>
                  <tbody className="divide-y divide-slate-100">
                    {filteredExpenses.map(e => (
                      <tr key={e.id} className="hover:bg-slate-50 transition-colors">
                        <td className="px-6 py-3 text-xs text-slate-500">{new Date(e.date + 'T00:00:00').toLocaleDateString('pt-BR', {day: '2-digit', month: '2-digit'})}</td>
                        <td className="px-6 py-3 text-[10px] font-black uppercase text-slate-600">{e.user}</td>
                        <td className="px-6 py-3">
                          <p className="text-xs font-bold leading-none">{e.description || 'Sem descrição'}</p>
                          <p className="text-[9px] text-slate-400 uppercase mt-1">{e.category}</p>
                        </td>
                        <td className="px-6 py-3 text-sm font-black text-right">R$ {Number(e.amount).toLocaleString('pt-BR', {minimumFractionDigits: 2})}</td>
                        <td className="px-6 py-3"><button onClick={() => setExpenses(expenses.filter(x => x.id !== e.id))} className="text-slate-300 hover:text-red-500"><Trash2 size={14}/></button></td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        )}

        {/* DASHBOARD MENSAL */}
        {activeTab === 'dashboard' && (
          <div className="space-y-6">
            <div className="flex flex-col md:flex-row gap-4 items-stretch">
              <div className="flex-1 grid grid-cols-2 gap-4">
                {users.map(u => (
                  <div key={u.id} className="bg-white p-4 rounded-2xl border border-slate-200 shadow-sm flex items-center gap-3">
                    <div className="bg-slate-50 p-2 rounded-xl text-slate-400"><Users size={18}/></div>
                    <div>
                      <p className="text-[9px] font-bold text-slate-400 uppercase leading-none mb-1">{u.name}</p>
                      <h4 className="text-sm font-black text-slate-800">R$ {(monthlyStats.byUser[u.name] || 0).toLocaleString('pt-BR', {minimumFractionDigits: 2})}</h4>
                    </div>
                  </div>
                ))}
              </div>
              <div className="md:w-64 bg-emerald-50 border border-emerald-100 p-4 rounded-2xl shadow-sm flex items-center justify-between">
                <div>
                  <h3 className="text-[9px] font-black text-emerald-600 uppercase tracking-wider mb-0.5">Gasto Total</h3>
                  <div className="text-xl font-black text-emerald-900 leading-tight">
                    R$ {monthlyStats.total.toLocaleString('pt-BR', {minimumFractionDigits: 2})}
                  </div>
                </div>
                <div className="bg-emerald-600 p-2 rounded-lg text-white"><Wallet size={20}/></div>
              </div>
            </div>
            <div className="bg-white rounded-2xl border border-slate-200 p-6 shadow-sm">
              <h3 className="font-bold text-slate-700 text-xs uppercase tracking-widest flex items-center gap-2 mb-6">
                <TrendingUp size={16} className="text-emerald-500"/> Categorias do Mês
              </h3>
              <div className="grid grid-cols-1 md:grid-cols-2 gap-x-12 gap-y-5">
                {Object.entries(monthlyStats.byCategory).sort(([,a],[,b]) => b-a).map(([cat, val]) => (
                  <div key={cat} className="space-y-1.5">
                    <div className="flex justify-between text-[10px] font-bold uppercase">
                      <span className="text-slate-500">{cat}</span>
                      <span className="text-slate-900 font-black">R$ {val.toLocaleString('pt-BR')}</span>
                    </div>
                    <div className="w-full bg-slate-100 rounded-full h-1.5 overflow-hidden">
                      <div className="bg-emerald-500 h-full transition-all duration-700" style={{ width: `${(val / (monthlyStats.total || 1)) * 100}%` }}></div>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          </div>
        )}

        {/* DASHBOARD ANUAL */}
        {activeTab === 'anual' && (
          <div className="space-y-6">
            <div className="flex flex-col md:flex-row gap-4 items-stretch">
              <div className="flex-1 grid grid-cols-2 gap-4">
                {users.map(u => (
                  <div key={u.id} className="bg-white p-6 rounded-2xl border border-slate-200 shadow-sm flex flex-col justify-center">
                    <p className="text-[10px] font-bold text-slate-400 uppercase mb-1">{u.name} no Ano</p>
                    <h4 className="text-xl font-black text-slate-800">R$ {(yearlyStats.byUser[u.name] || 0).toLocaleString('pt-BR', {minimumFractionDigits: 2})}</h4>
                  </div>
                ))}
              </div>
              <div className="md:w-72 bg-slate-900 text-white p-6 rounded-2xl shadow-xl flex items-center justify-between">
                <div>
                  <h3 className="text-[10px] font-black text-emerald-400 uppercase tracking-widest mb-1">Total Anual {currentYear}</h3>
                  <div className="text-2xl font-black tracking-tighter">
                    R$ {yearlyStats.total.toLocaleString('pt-BR', {minimumFractionDigits: 2})}
                  </div>
                </div>
                <div className="bg-white/10 p-3 rounded-xl"><CalendarDays size={24}/></div>
              </div>
            </div>
            <div className="bg-white rounded-2xl border border-slate-200 p-6 shadow-sm">
              <h3 className="font-bold text-slate-700 text-xs uppercase tracking-widest flex items-center gap-2 mb-8">
                <BarChart3 size={16} className="text-emerald-500"/> Evolução Mensal
              </h3>
              <div className="flex items-end justify-between h-48 gap-2 px-2">
                {yearlyStats.byMonth.map((m, idx) => (
                  <div key={idx} className="flex-1 flex flex-col items-center group relative">
                    <div className="absolute -top-10 bg-slate-800 text-white text-[9px] px-2 py-1 rounded opacity-0 group-hover:opacity-100 transition-opacity pointer-events-none z-10 font-bold">
                      R$ {m.total.toLocaleString('pt-BR')}
                    </div>
                    <div 
                      className={`w-full rounded-t-md transition-all duration-500 ${idx === currentMonth ? 'bg-emerald-500' : 'bg-slate-100 group-hover:bg-slate-200'}`}
                      style={{ height: `${(m.total / yearlyStats.maxMonth) * 100}%`, minHeight: m.total > 0 ? '4px' : '0' }}
                    />
                    <span className={`text-[8px] mt-2 font-bold uppercase ${idx === currentMonth ? 'text-emerald-600' : 'text-slate-400'}`}>
                      {m.name.substring(0, 3)}
                    </span>
                  </div>
                ))}
              </div>
            </div>
          </div>
        )}
      </main>

      {/* OVERLAY DO SCANNER */}
      {showScanner && (
        <div className="fixed inset-0 z-[60] bg-black flex flex-col items-center justify-center">
          <div className="relative w-full max-w-lg aspect-[3/4] bg-slate-900 overflow-hidden shadow-2xl">
            <video ref={videoRef} autoPlay playsInline className="w-full h-full object-cover" />
            <div className="absolute inset-0 border-[2px] border-white/30 m-8 pointer-events-none flex items-center justify-center">
               <div className="w-full h-0.5 bg-emerald-500/50 absolute animate-scan-line shadow-[0_0_15px_rgba(16,185,129,0.5)]"></div>
            </div>
            
            {/* CONTROLES DO SCANNER */}
            <div className="absolute bottom-8 left-0 right-0 flex justify-around items-center px-8">
              <button onClick={stopCamera} className="bg-white/10 backdrop-blur-md p-4 rounded-full text-white hover:bg-white/20 transition-all">
                <X size={24} />
              </button>
              <button onClick={capturePhoto} className="w-20 h-20 bg-white rounded-full border-4 border-emerald-500 flex items-center justify-center shadow-lg active:scale-95 transition-transform">
                <div className="w-16 h-16 rounded-full bg-white border-2 border-slate-200"></div>
              </button>
              <div className="w-14"></div> {/* Spacer */}
            </div>
          </div>
          <canvas ref={canvasRef} className="hidden" />
          <p className="text-white/60 text-xs font-bold uppercase tracking-widest mt-6">Enquadre o Cupom Fiscal</p>
        </div>
      )}

      {/* MODAL CONFIGURAÇÕES (GERIR USUÁRIOS E CATEGORIAS) */}
      {showConfigAdmin && (
        <div className="fixed inset-0 z-50 bg-slate-900/60 backdrop-blur-sm flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-2xl rounded-3xl shadow-2xl overflow-hidden flex flex-col max-h-[85vh]">
            <div className="p-6 border-b border-slate-100 flex justify-between items-center bg-slate-50">
              <h3 className="font-black text-slate-800 uppercase tracking-widest text-sm flex items-center gap-2">
                <Settings size={18} className="text-emerald-600" /> Configurações do App
              </h3>
              <button onClick={() => setShowConfigAdmin(false)} className="text-slate-400 hover:text-slate-600"><X size={20} /></button>
            </div>
            
            <div className="p-6 space-y-8 overflow-y-auto">
              
              {/* SEÇÃO USUÁRIOS */}
              <section className="space-y-4">
                <h4 className="text-[10px] font-black text-emerald-600 uppercase tracking-[2px] border-b border-emerald-100 pb-2 flex items-center gap-2">
                  <Users size={14}/> Gerenciar Usuários
                </h4>
                <div className="flex gap-2">
                  <input type="text" placeholder="Nome do novo usuário..." className="flex-1 bg-slate-50 border-slate-200 rounded-xl text-sm" value={newUser} onChange={(e) => setNewUser(e.target.value)}/>
                  <button onClick={addUser} className="bg-emerald-600 text-white px-4 py-2 rounded-xl font-bold text-xs uppercase shadow-sm">Adicionar</button>
                </div>
                <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                  {users.map(u => (
                    <div key={u.id} className="border border-slate-100 rounded-2xl p-4 bg-slate-50/50">
                      <div className="flex justify-between mb-3">
                        <span className="font-black text-[11px] uppercase text-slate-700 tracking-wider">{u.name}</span>
                        <button onClick={() => removeUser(u.id)} className="text-slate-300 hover:text-red-500 transition-colors"><Trash2 size={16}/></button>
                      </div>
                      <div className="flex flex-wrap gap-2">
                        {u.cards.map((c, i) => <span key={i} className="bg-white border border-slate-200 px-2 py-1 rounded text-[10px] font-bold flex items-center gap-2 shadow-sm text-slate-500">****{c} <X size={10} className="cursor-pointer text-slate-300 hover:text-red-500" onClick={() => removeCardFromUser(u.id, i)}/></span>)}
                        <input maxLength="4" placeholder="Adic. 4 dígitos..." className="w-24 text-[10px] border-slate-200 rounded px-2 py-1 bg-white" onKeyDown={e => {if(e.key==='Enter'){addCardToUser(u.id, e.target.value); e.target.value='';}}}/>
                      </div>
                    </div>
                  ))}
                </div>
              </section>

              {/* SEÇÃO CATEGORIAS */}
              <section className="space-y-4">
                <h4 className="text-[10px] font-black text-emerald-600 uppercase tracking-[2px] border-b border-emerald-100 pb-2 flex items-center gap-2">
                  <Tag size={14}/> Gerenciar Categorias
                </h4>
                <div className="flex gap-2">
                  <input type="text" placeholder="Nome da nova categoria..." className="flex-1 bg-slate-50 border-slate-200 rounded-xl text-sm" value={newCategory} onChange={(e) => setNewCategory(e.target.value)}/>
                  <button onClick={addCategory} className="bg-emerald-600 text-white px-4 py-2 rounded-xl font-bold text-xs uppercase shadow-sm">Adicionar</button>
                </div>
                <div className="flex flex-wrap gap-2 max-h-48 overflow-y-auto p-2 bg-slate-50 rounded-2xl border border-slate-100">
                  {categories.map(cat => (
                    <div key={cat} className="bg-white border border-slate-200 px-3 py-1.5 rounded-xl text-[10px] font-black uppercase text-slate-600 flex items-center gap-3 shadow-sm group">
                      {cat}
                      <button onClick={() => removeCategory(cat)} className="text-slate-300 group-hover:text-red-500 transition-colors">
                        <X size={12}/>
                      </button>
                    </div>
                  ))}
                </div>
              </section>

            </div>
            
            <div className="p-6 bg-slate-50 border-t border-slate-100 text-center">
              <button onClick={() => setShowConfigAdmin(false)} className="bg-slate-800 text-white px-8 py-2.5 rounded-xl font-bold text-xs uppercase shadow-lg hover:bg-slate-700 transition-all">Fechar</button>
            </div>
          </div>
        </div>
      )}

      {feedback && (
        <div className="fixed bottom-24 left-1/2 -translate-x-1/2 z-50 animate-bounce bg-slate-800 text-white text-[10px] font-black px-6 py-2.5 rounded-full flex items-center gap-2 uppercase border border-slate-700">
          {isScanning ? <Loader2 size={14} className="animate-spin text-emerald-400" /> : <CheckCircle2 size={14} className="text-emerald-400" />} {feedback}
        </div>
      )}

      <style>{`
        @keyframes scan-line {
          0% { top: 0%; }
          100% { top: 100%; }
        }
        .animate-scan-line {
          animation: scan-line 2s linear infinite;
        }
      `}</style>
    </div>
  );
};

export default App;
