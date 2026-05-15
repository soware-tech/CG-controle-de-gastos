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
  
  const videoRef = useRef(null);
  const canvasRef = useRef(null);

  const [expenses, setExpenses] = useState(() => {
    const saved = localStorage.getItem('expenses_v9');
    return saved ? JSON.parse(saved) : [];
  });

  const [users, setUsers] = useState(() => {
    const saved = localStorage.getItem('users_v9');
    return saved ? JSON.parse(saved) : [
      { id: '1', name: 'Aline', cards: ['1234'] },
      { id: '2', name: 'Roney', cards: ['5678'] }
    ];
  });

  const [categories, setCategories] = useState(() => {
    const saved = localStorage.getItem('categories_v9');
    return saved ? JSON.parse(saved) : [
      'Aluguel', 'Beleza', 'Casa', 'Combustível', 'Energia', 'Estacionamento', 
      'Manutenção Carro', 'Feira', 'Fast Food', 'Delivery', 'Gás', 
      'Mercado', 'Farmácia', 'Internet', 'Celular', 'Padaria', 'Pets'
    ];
  });

  const [pendingExpenses, setPendingExpenses] = useState([]);

  useEffect(() => {
    localStorage.setItem('expenses_v9', JSON.stringify(expenses));
  }, [expenses]);

  useEffect(() => {
    localStorage.setItem('users_v9', JSON.stringify(users));
  }, [users]);

  useEffect(() => {
    localStorage.setItem('categories_v9', JSON.stringify(categories));
  }, [categories]);

  const showToast = (message) => {
    setFeedback(message);
    setTimeout(() => setFeedback(null), 3000);
  };

  // --- LÓGICA DE IA (GEMINI) ---
  const callGemini = async (prompt, base64Image = null) => {
    setIsScanning(true);
    const userContext = users.map(u => `${u.name} (finais: ${u.cards.join(', ')})`).join('; ');
    
    const payload = {
      contents: [{
        parts: [
          { text: prompt },
          ...(base64Image ? [{ inlineData: { mimeType: "image/png", data: base64Image } }] : [])
        ]
      }],
      systemInstruction: {
        parts: [{
          text: `Você é um extrator de recibos. Analise os dados e retorne APENAS um JSON válido.
          Usuários: ${userContext}. Categorias: ${categories.join(', ')}.
          O JSON deve ser: {"estabelecimento": "string", "valor": number, "usuario": "nome_do_usuario", "categoria": "categoria_mais_proxima"}`
        }]
      },
      generationConfig: { responseMimeType: "application/json" }
    };

    try {
      const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      const data = await response.json();
      const result = JSON.parse(data.candidates[0].content.parts[0].text);
      return result;
    } catch (error) {
      console.error("Erro na IA:", error);
      showToast("Erro ao processar com IA");
      return null;
    } finally {
      setIsScanning(false);
    }
  };

  const addPendingItem = (data) => {
    if (!data) return;
    const newItem = {
      id: Date.now() + Math.random(),
      user: data.usuario || users[0].name,
      description: data.estabelecimento || "Nova Compra",
      category: categories.includes(data.categoria) ? data.categoria : categories[0],
      amount: data.valor || 0,
      date: new Date().toISOString().split('T')[0]
    };
    setPendingExpenses(prev => [newItem, ...prev]);
    showToast("Mapeamento concluído!");
  };

  // --- FERRAMENTAS DE CAPTURA ---

  // 1. Scanner ao Vivo
  const startScanner = async () => {
    setShowScanner(true);
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
      if (videoRef.current) videoRef.current.srcObject = stream;
    } catch (err) {
      showToast("Erro ao abrir câmera");
      setShowScanner(false);
    }
  };

  const capturePhoto = async () => {
    if (!videoRef.current || !canvasRef.current) return;
    const context = canvasRef.current.getContext('2d');
    canvasRef.current.width = videoRef.current.videoWidth;
    canvasRef.current.height = videoRef.current.videoHeight;
    context.drawImage(videoRef.current, 0, 0);
    
    const base64 = canvasRef.current.toDataURL('image/png').split(',')[1];
    
    // Parar câmera
    const stream = videoRef.current.srcObject;
    stream.getTracks().forEach(track => track.stop());
    setShowScanner(false);

    showToast("Lendo Cupom...");
    const result = await callGemini("Extraia os dados deste cupom fiscal.", base64);
    addPendingItem(result);
  };

  // 2. Upload de Arquivos
  const handleFileUpload = async (e) => {
    const file = e.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = async (event) => {
      const base64 = event.target.result.split(',')[1];
      showToast("Analisando imagem...");
      const result = await callGemini("Extraia os dados desta imagem de recibo.", base64);
      addPendingItem(result);
    };
    reader.readAsDataURL(file);
  };

  // 3. Link de Nota
  const handleLinkProcess = async () => {
    if (!scanInput.trim()) return;
    showToast("Acessando link...");
    const result = await callGemini(`Extraia os dados de gasto deste link de nota fiscal: ${scanInput}`);
    addPendingItem(result);
    setScanInput('');
  };

  // --- OUTRAS FUNÇÕES ---
  const handleManualSubmit = (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);
    const newExp = {
      id: Date.now(),
      user: formData.get('user'),
      category: formData.get('category'),
      description: formData.get('description'),
      amount: parseFloat(formData.get('amount')),
      date: new Date().toISOString().split('T')[0]
    };
    setExpenses([newExp, ...expenses]);
    e.target.reset();
    showToast("Gasto salvo!");
  };

  const confirmPending = (id) => {
    const item = pendingExpenses.find(p => p.id === id);
    if (item) {
      const { id: oldId, ...cleanItem } = item;
      setExpenses([{...cleanItem, id: Date.now()}, ...expenses]);
      setPendingExpenses(pendingExpenses.filter(p => p.id !== id));
    }
  };

  const removePending = (id) => {
    setPendingExpenses(pendingExpenses.filter(p => p.id !== id));
  };

  const filteredExpenses = useMemo(() => {
    return expenses.filter(e => {
      const d = new Date(e.date + 'T00:00:00');
      return d.getMonth() === currentMonth && d.getFullYear() === currentYear;
    });
  }, [expenses, currentMonth, currentYear]);

  const monthNames = ["Janeiro", "Fevereiro", "Março", "Abril", "Maio", "Junho", "Julho", "Agosto", "Setembro", "Outubro", "Novembro", "Dezembro"];

  return (
    <div className="min-h-screen bg-[#f8fafc] text-slate-900 pb-20 font-['Inter']">
      
      {/* NAVBAR */}
      <nav className="bg-white border-b border-slate-200 sticky top-0 z-40">
        <div className="max-w-6xl mx-auto px-4 flex justify-between items-center h-16">
          <div className="flex items-center gap-2">
            <div className="bg-emerald-600 p-1.5 rounded text-white font-bold text-sm">CG</div>
            <span className="font-bold text-lg text-slate-700 hidden sm:block">Controle de Gastos</span>
          </div>
          <div className="flex bg-slate-100 p-1 rounded-xl">
            <button onClick={() => setActiveTab('lancamentos')} className={`px-4 py-1.5 rounded-lg text-xs font-black transition-all uppercase ${activeTab === 'lancamentos' ? 'bg-white shadow-sm text-emerald-700' : 'text-slate-400'}`}>Lançar</button>
            <button onClick={() => setActiveTab('dashboard')} className={`px-4 py-1.5 rounded-lg text-xs font-black transition-all uppercase ${activeTab === 'dashboard' ? 'bg-white shadow-sm text-emerald-700' : 'text-slate-400'}`}>Mensal</button>
            <button onClick={() => setActiveTab('anual')} className={`px-4 py-1.5 rounded-lg text-xs font-black transition-all uppercase ${activeTab === 'anual' ? 'bg-white shadow-sm text-emerald-700' : 'text-slate-400'}`}>Anual</button>
          </div>
          <button onClick={() => setShowConfigAdmin(true)} className="p-2 text-slate-400 hover:text-emerald-600 transition-colors">
            <Settings size={20} />
          </button>
        </div>
      </nav>

      {/* HEADER CALENDÁRIO */}
      <div className="bg-emerald-700 text-white py-4 shadow-inner">
        <div className="max-w-5xl mx-auto px-4 flex items-center justify-between">
          <button onClick={() => {
            if(currentMonth === 0) { setCurrentMonth(11); setCurrentYear(currentYear-1); }
            else { setCurrentMonth(currentMonth-1); }
          }} className="p-2 hover:bg-white/10 rounded-full"><ChevronLeft /></button>
          <div className="text-center">
            <h2 className="font-black text-lg uppercase tracking-tight">{activeTab === 'anual' ? 'Visão Anual' : monthNames[currentMonth]}</h2>
            <p className="text-[10px] opacity-70 font-black tracking-widest uppercase">{currentYear}</p>
          </div>
          <button onClick={() => {
            if(currentMonth === 11) { setCurrentMonth(0); setCurrentYear(currentYear+1); }
            else { setCurrentMonth(currentMonth+1); }
          }} className="p-2 hover:bg-white/10 rounded-full"><ChevronRight /></button>
        </div>
      </div>

      <main className="max-w-5xl mx-auto p-4 md:p-6">
        
        {activeTab === 'lancamentos' && (
          <div className="space-y-6">
            
            {/* FERRAMENTAS IA */}
            <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
              <div className="bg-white border border-emerald-100 rounded-2xl p-4 shadow-sm hover:shadow-md transition-shadow">
                <div className="flex items-center gap-3 mb-3">
                  <div className="bg-emerald-100 p-2 rounded-xl text-emerald-600"><Camera size={20}/></div>
                  <h4 className="text-xs font-black text-slate-700 uppercase">Scanner ao Vivo</h4>
                </div>
                <button onClick={startScanner} className="w-full bg-emerald-600 text-white font-bold py-2.5 rounded-xl text-xs uppercase flex items-center justify-center gap-2 hover:bg-emerald-700 shadow-sm active:scale-95 transition-all">
                  <Maximize size={14}/> Abrir Câmera
                </button>
              </div>

              <div className="bg-white border border-blue-100 rounded-2xl p-4 shadow-sm hover:shadow-md transition-shadow">
                <div className="flex items-center gap-3 mb-3">
                  <div className="bg-blue-100 p-2 rounded-xl text-blue-600"><Files size={20}/></div>
                  <h4 className="text-xs font-black text-slate-700 uppercase">Upload</h4>
                </div>
                <label className="w-full bg-blue-50 border-2 border-dashed border-blue-200 text-blue-700 rounded-xl text-xs font-bold py-2.5 flex items-center justify-center gap-2 cursor-pointer hover:bg-blue-100 transition-all">
                  <ImageIcon size={14}/> Galeria de Fotos
                  <input type="file" className="hidden" accept="image/*" onChange={handleFileUpload} />
                </label>
              </div>

              <div className="bg-white border border-slate-200 rounded-2xl p-4 shadow-sm hover:shadow-md transition-shadow">
                <div className="flex items-center gap-3 mb-3">
                  <div className="bg-slate-100 p-2 rounded-xl text-slate-600"><ScanQrCode size={20}/></div>
                  <h4 className="text-xs font-black text-slate-700 uppercase">Link de Nota</h4>
                </div>
                <div className="flex gap-2">
                  <input 
                    type="text" 
                    placeholder="Cole o link..." 
                    value={scanInput}
                    onChange={(e) => setScanInput(e.target.value)}
                    className="flex-1 bg-slate-50 border border-slate-100 rounded-lg text-xs p-2 focus:ring-1 focus:ring-slate-300 outline-none"
                  />
                  <button onClick={handleLinkProcess} className="bg-slate-800 text-white px-3 py-2 rounded-lg font-bold text-xs hover:bg-slate-700 active:scale-95 transition-all">OK</button>
                </div>
              </div>
            </div>

            {/* PENDENTES IA */}
            {pendingExpenses.length > 0 && (
              <div className="bg-amber-50 border border-amber-200 rounded-2xl overflow-hidden shadow-md">
                <div className="p-4 bg-amber-100/50 flex justify-between items-center border-b border-amber-200">
                  <h3 className="text-xs font-black text-amber-800 uppercase flex items-center gap-2">
                    <Sparkles size={14} className="animate-pulse" /> Validar Mapeamento IA ({pendingExpenses.length})
                  </h3>
                  <button onClick={() => {
                    pendingExpenses.forEach(p => confirmPending(p.id));
                  }} className="bg-amber-600 text-white px-3 py-1 rounded-lg text-[10px] font-bold uppercase flex items-center gap-1 hover:bg-amber-700">
                    <Save size={12}/> Salvar Todos
                  </button>
                </div>
                <div className="p-2 space-y-2">
                  {pendingExpenses.map(p => (
                    <div key={p.id} className="bg-white p-3 rounded-xl border border-amber-100 grid grid-cols-1 md:grid-cols-6 gap-2 items-center">
                      <div className="md:col-span-1">
                        <select 
                          className="w-full text-[10px] font-bold border-slate-100 rounded bg-slate-50 uppercase"
                          value={p.user}
                          onChange={(e) => {
                            setPendingExpenses(prev => prev.map(item => item.id === p.id ? {...item, user: e.target.value} : item));
                          }}
                        >
                          {users.map(u => <option key={u.id} value={u.name}>{u.name}</option>)}
                        </select>
                      </div>
                      <div className="md:col-span-2">
                        <input 
                          className="w-full text-xs border-slate-100 rounded p-1 font-medium"
                          value={p.description}
                          onChange={(e) => {
                            setPendingExpenses(prev => prev.map(item => item.id === p.id ? {...item, description: e.target.value} : item));
                          }}
                        />
                      </div>
                      <div className="md:col-span-1">
                        <select 
                          className="w-full text-[10px] border-slate-100 rounded bg-slate-50"
                          value={p.category}
                          onChange={(e) => {
                            setPendingExpenses(prev => prev.map(item => item.id === p.id ? {...item, category: e.target.value} : item));
                          }}
                        >
                          {categories.map(c => <option key={c} value={c}>{c}</option>)}
                        </select>
                      </div>
                      <div className="text-right font-black text-xs text-amber-700">R$ {p.amount.toFixed(2)}</div>
                      <div className="flex justify-end gap-2">
                        <button onClick={() => removePending(p.id)} className="text-slate-300 hover:text-red-500"><Trash2 size={14}/></button>
                        <button onClick={() => confirmPending(p.id)} className="text-emerald-500 hover:scale-110 transition-transform"><CheckCircle2 size={16}/></button>
                      </div>
                    </div>
                  ))}
                </div>
              </div>
            )}

            {/* FORMULÁRIO MANUAL */}
            <form onSubmit={handleManualSubmit} className="bg-white rounded-2xl shadow-sm border border-slate-200 p-6 grid grid-cols-1 md:grid-cols-5 gap-4 items-end">
              <div className="space-y-1">
                <label className="text-[10px] font-bold text-slate-400 uppercase">Usuário</label>
                <select name="user" className="w-full bg-slate-50 border-slate-200 rounded-lg text-sm p-2 outline-none focus:ring-1 focus:ring-emerald-500 transition-all">
                  {users.map(u => <option key={u.id} value={u.name}>{u.name}</option>)}
                </select>
              </div>
              <div className="space-y-1">
                <label className="text-[10px] font-bold text-slate-400 uppercase">Categoria</label>
                <select name="category" className="w-full bg-slate-50 border-slate-200 rounded-lg text-sm p-2 outline-none focus:ring-1 focus:ring-emerald-500 transition-all">
                  {categories.map(c => <option key={c} value={c}>{c}</option>)}
                </select>
              </div>
              <div className="space-y-1">
                <label className="text-[10px] font-bold text-slate-400 uppercase">Local</label>
                <input name="description" required type="text" placeholder="Ex: Mercado Livre" className="w-full bg-slate-50 border-slate-200 rounded-lg text-sm p-2 outline-none focus:ring-1 focus:ring-emerald-500 transition-all" />
              </div>
              <div className="space-y-1">
                <label className="text-[10px] font-bold text-slate-400 uppercase">Valor R$</label>
                <input name="amount" required step="0.01" type="number" placeholder="0,00" className="w-full border-emerald-200 bg-emerald-50/30 rounded-lg text-sm p-2 font-bold outline-none focus:ring-1 focus:ring-emerald-500 transition-all" />
              </div>
              <button type="submit" className="bg-emerald-600 text-white font-bold py-2.5 rounded-lg text-sm uppercase shadow-md hover:bg-emerald-700 active:scale-95 transition-all">Salvar</button>
            </form>

            {/* TABELA GASTOS */}
            <div className="bg-white rounded-2xl shadow-sm border border-slate-200 overflow-hidden">
              <div className="px-6 py-4 border-b border-slate-100 flex justify-between items-center">
                <h3 className="text-xs font-black text-slate-500 uppercase tracking-widest">Registos do Mês</h3>
                <span className="text-[10px] bg-slate-100 text-slate-600 px-2 py-1 rounded-full font-bold">{filteredExpenses.length} itens</span>
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
                      <tr key={e.id} className="hover:bg-slate-50/80 transition-colors">
                        <td className="px-6 py-4 text-xs text-slate-400">{new Date(e.date + 'T00:00:00').toLocaleDateString('pt-BR', {day:'2-digit', month:'2-digit'})}</td>
                        <td className="px-6 py-4">
                          <span className="text-[10px] font-black uppercase text-slate-600 px-2 py-0.5 bg-slate-100 rounded-md">{e.user}</span>
                        </td>
                        <td className="px-6 py-4">
                          <p className="text-xs font-bold text-slate-800 leading-none mb-1">{e.description}</p>
                          <p className="text-[9px] text-slate-400 uppercase font-bold flex items-center gap-1">
                            <Tag size={8} /> {e.category}
                          </p>
                        </td>
                        <td className="px-6 py-4 text-sm font-black text-right text-slate-900">R$ {e.amount.toLocaleString('pt-BR', {minimumFractionDigits: 2})}</td>
                        <td className="px-6 py-4">
                          <button onClick={() => setExpenses(expenses.filter(item => item.id !== e.id))} className="text-slate-300 hover:text-red-500 transition-colors"><Trash2 size={14}/></button>
                        </td>
                      </tr>
                    ))}
                    {filteredExpenses.length === 0 && (
                      <tr>
                        <td colSpan="5" className="px-6 py-12 text-center text-slate-300 italic text-xs">Nenhum registro encontrado.</td>
                      </tr>
                    )}
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        )}

        {activeTab === 'dashboard' && (
          <div className="space-y-6">
            <div className="flex flex-col md:flex-row gap-4">
              <div className="flex-1 grid grid-cols-2 gap-4">
                {users.map(u => {
                  const total = filteredExpenses.filter(e => e.user === u.name).reduce((sum, curr) => sum + curr.amount, 0);
                  return (
                    <div key={u.id} className="bg-white p-4 rounded-2xl border border-slate-200 shadow-sm flex items-center gap-3">
                      <div className="bg-slate-50 p-3 rounded-2xl text-slate-400"><Users size={20}/></div>
                      <div>
                        <p className="text-[9px] font-black text-slate-400 uppercase mb-0.5 tracking-wider">{u.name}</p>
                        <h4 className="text-base font-black text-slate-800 tracking-tight">R$ {total.toLocaleString('pt-BR', {minimumFractionDigits: 2})}</h4>
                      </div>
                    </div>
                  );
                })}
              </div>
              <div className="md:w-64 bg-emerald-50 border border-emerald-100 p-4 rounded-2xl shadow-sm flex items-center justify-between">
                <div>
                  <h3 className="text-[9px] font-black text-emerald-600 uppercase tracking-widest mb-0.5">Gasto Mensal</h3>
                  <div className="text-xl font-black text-emerald-900 tracking-tighter">
                    R$ {filteredExpenses.reduce((sum, curr) => sum + curr.amount, 0).toLocaleString('pt-BR', {minimumFractionDigits: 2})}
                  </div>
                </div>
                <div className="bg-emerald-600 p-3 rounded-xl text-white shadow-lg shadow-emerald-200"><Wallet size={20}/></div>
              </div>
            </div>

            <div className="bg-white rounded-2xl border border-slate-200 p-6 shadow-sm">
              <h3 className="font-black text-slate-700 text-xs uppercase tracking-widest flex items-center gap-2 mb-8">
                <TrendingUp size={16} className="text-emerald-500" /> Categorias do Mês
              </h3>
              <div className="grid grid-cols-1 md:grid-cols-2 gap-x-12 gap-y-6">
                {categories.map(cat => {
                  const catTotal = filteredExpenses.filter(e => e.category === cat).reduce((sum, curr) => sum + curr.amount, 0);
                  const total = filteredExpenses.reduce((sum, curr) => sum + curr.amount, 0);
                  const percent = total > 0 ? (catTotal / total) * 100 : 0;
                  if (catTotal === 0) return null;
                  return (
                    <div key={cat} className="space-y-2">
                      <div className="flex justify-between text-[10px] font-black uppercase">
                        <span className="text-slate-500 tracking-wide">{cat}</span>
                        <span className="text-slate-900">R$ {catTotal.toLocaleString('pt-BR')}</span>
                      </div>
                      <div className="w-full bg-slate-100 rounded-full h-1.5 overflow-hidden">
                        <div className="bg-emerald-500 h-full transition-all duration-1000 ease-out rounded-full" style={{ width: `${percent}%` }}></div>
                      </div>
                    </div>
                  );
                }).filter(x => x !== null)}
              </div>
            </div>
          </div>
        )}

        {activeTab === 'anual' && (
          <div className="space-y-6">
            <div className="flex flex-col md:flex-row gap-4">
              <div className="flex-1 grid grid-cols-2 gap-4">
                {users.map(u => {
                  const yearly = expenses.filter(e => new Date(e.date + 'T00:00:00').getFullYear() === currentYear && e.user === u.name).reduce((sum, curr) => sum + curr.amount, 0);
                  return (
                    <div key={u.id} className="bg-white p-6 rounded-2xl border border-slate-200 shadow-sm">
                      <p className="text-[10px] font-black text-slate-400 uppercase mb-1 tracking-widest">{u.name}</p>
                      <h4 className="text-2xl font-black text-slate-800 tracking-tighter">R$ {yearly.toLocaleString('pt-BR', {minimumFractionDigits: 2})}</h4>
                    </div>
                  );
                })}
              </div>
              <div className="md:w-72 bg-slate-900 text-white p-6 rounded-2xl shadow-xl flex items-center justify-between">
                <div>
                  <h3 className="text-[10px] font-black text-emerald-400 uppercase tracking-[0.2em] mb-1">Total {currentYear}</h3>
                  <div className="text-3xl font-black tracking-tighter">
                    R$ {expenses.filter(e => new Date(e.date + 'T00:00:00').getFullYear() === currentYear).reduce((sum, curr) => sum + curr.amount, 0).toLocaleString('pt-BR', {minimumFractionDigits: 2})}
                  </div>
                </div>
                <div className="bg-white/10 p-4 rounded-2xl backdrop-blur-md"><CalendarDays size={24}/></div>
              </div>
            </div>

            <div className="bg-white rounded-2xl border border-slate-200 p-6 shadow-sm">
              <h3 className="font-black text-slate-700 text-xs uppercase tracking-widest flex items-center gap-2 mb-10">
                <BarChart3 size={16} className="text-emerald-500" /> Evolução Mensal
              </h3>
              <div className="flex items-end justify-between h-48 gap-2 px-2">
                {monthNames.map((m, i) => {
                  const mTotal = expenses.filter(e => {
                    const d = new Date(e.date + 'T00:00:00');
                    return d.getMonth() === i && d.getFullYear() === currentYear;
                  }).reduce((sum, curr) => sum + curr.amount, 0);
                  const yearlyMax = Math.max(...monthNames.map((_, idx) => expenses.filter(e => {
                    const d = new Date(e.date + 'T00:00:00');
                    return d.getMonth() === idx && d.getFullYear() === currentYear;
                  }).reduce((sum, curr) => sum + curr.amount, 0)), 1);
                  const height = (mTotal / yearlyMax) * 100;
                  return (
                    <div key={m} className="flex-1 flex flex-col items-center group relative h-full justify-end">
                      <div className="absolute -top-8 bg-slate-800 text-white text-[9px] px-2 py-1 rounded opacity-0 group-hover:opacity-100 transition-opacity z-10 font-bold whitespace-nowrap">
                        R$ {mTotal.toLocaleString('pt-BR')}
                      </div>
                      <div 
                        className={`w-full rounded-t-md transition-all duration-700 ease-out ${i === currentMonth ? 'bg-emerald-500' : 'bg-slate-100 group-hover:bg-emerald-100'}`} 
                        style={{ height: `${height}%`, minHeight: mTotal > 0 ? '4px' : '0' }}
                      ></div>
                      <span className={`text-[9px] mt-3 font-black uppercase ${i === currentMonth ? 'text-emerald-600' : 'text-slate-400'}`}>{m.substring(0,3)}</span>
                    </div>
                  );
                })}
              </div>
            </div>
          </div>
        )}
      </main>

      {/* SCANNER MODAL */}
      {showScanner && (
        <div className="fixed inset-0 z-50 bg-black flex flex-col items-center justify-center">
          <div className="relative w-full max-w-lg aspect-[3/4] bg-slate-900 overflow-hidden">
            <video ref={videoRef} autoPlay playsInline className="w-full h-full object-cover" />
            <div className="absolute inset-0 border-2 border-emerald-500/30 m-12 pointer-events-none">
              <div className="absolute top-0 left-0 w-full h-1 bg-emerald-500/50 animate-scan-line shadow-[0_0_15px_rgba(16,185,129,0.5)]"></div>
            </div>
            <div className="absolute bottom-10 left-0 right-0 flex justify-around items-center px-10">
              <button onClick={() => {
                const stream = videoRef.current.srcObject;
                stream.getTracks().forEach(track => track.stop());
                setShowScanner(false);
              }} className="p-4 bg-white/10 rounded-full text-white backdrop-blur-md">
                <X size={24}/>
              </button>
              <button onClick={capturePhoto} className="w-20 h-20 bg-white rounded-full border-4 border-emerald-500 flex items-center justify-center shadow-2xl active:scale-90 transition-all">
                <div className="w-16 h-16 rounded-full border-2 border-slate-200 bg-white"></div>
              </button>
              <div className="w-14"></div>
            </div>
          </div>
          <canvas ref={canvasRef} className="hidden" />
          <p className="text-white/60 text-[10px] font-black uppercase tracking-[0.3em] mt-8">Posicione o Cupom Fiscal</p>
        </div>
      )}

      {/* CONFIG MODAL */}
      {showConfigAdmin && (
        <div className="fixed inset-0 z-50 bg-slate-900/80 backdrop-blur-sm flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-2xl rounded-[2.5rem] shadow-2xl overflow-hidden animate-in zoom-in-95 duration-200">
            <div className="p-8 border-b border-slate-100 flex justify-between items-center bg-slate-50">
              <h3 className="font-black text-slate-800 uppercase tracking-widest text-sm flex items-center gap-2">
                <Settings size={20} className="text-emerald-600" /> Painel de Controle
              </h3>
              <button onClick={() => setShowConfigAdmin(false)} className="text-slate-400 hover:text-red-500 transition-colors"><X size={24}/></button>
            </div>
            
            <div className="p-8 space-y-10 max-h-[70vh] overflow-y-auto no-scrollbar">
              
              <section className="space-y-6">
                <div className="flex items-center justify-between border-b border-emerald-100 pb-2">
                  <h4 className="text-[10px] font-black text-emerald-600 uppercase tracking-[0.2em] flex items-center gap-2">
                    <Users size={14}/> Gerenciar Usuários
                  </h4>
                </div>
                <div className="flex gap-2">
                  <input 
                    type="text" 
                    value={newUser}
                    onChange={(e) => setNewUser(e.target.value)}
                    placeholder="Nome do Usuário..." 
                    className="flex-1 bg-slate-50 border border-slate-200 rounded-xl text-xs p-3 outline-none focus:ring-2 focus:ring-emerald-500/20"
                  />
                  <button 
                    onClick={() => {
                      if(newUser.trim()) {
                        setUsers([...users, { id: Date.now().toString(), name: newUser, cards: [] }]);
                        setNewUser('');
                      }
                    }}
                    className="bg-emerald-600 text-white px-5 rounded-xl font-bold text-xs uppercase shadow-md hover:bg-emerald-700 active:scale-95 transition-all"
                  >
                    <UserPlus size={16}/>
                  </button>
                </div>
                <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                  {users.map(u => (
                    <div key={u.id} className="border border-slate-100 rounded-2xl p-4 bg-slate-50/50 hover:bg-white transition-colors group">
                      <div className="flex justify-between items-center mb-4">
                        <span className="font-black text-xs uppercase text-slate-700 tracking-wide">{u.name}</span>
                        <button onClick={() => setUsers(users.filter(x => x.id !== u.id))} className="text-slate-200 hover:text-red-500 group-hover:text-slate-400 transition-colors">
                          <UserMinus size={14}/>
                        </button>
                      </div>
                      <div className="space-y-3">
                        <div className="flex flex-wrap gap-2">
                          {u.cards.map((c, idx) => (
                            <span key={idx} className="bg-white border border-slate-200 px-2 py-1 rounded-lg text-[9px] font-bold text-slate-500 flex items-center gap-1.5 shadow-sm">
                              <CreditCard size={10} /> ****{c}
                              <button onClick={() => {
                                const newUsers = [...users];
                                const userIdx = newUsers.findIndex(x => x.id === u.id);
                                newUsers[userIdx].cards.splice(idx, 1);
                                setUsers(newUsers);
                              }} className="text-slate-300 hover:text-red-500"><X size={10}/></button>
                            </span>
                          ))}
                        </div>
                        <input 
                          type="text" 
                          maxLength="4"
                          placeholder="Adic. final cartão (4 dígitos)"
                          onKeyDown={(e) => {
                            if(e.key === 'Enter' && e.target.value.length === 4) {
                              const newUsers = [...users];
                              const userIdx = newUsers.findIndex(x => x.id === u.id);
                              newUsers[userIdx].cards.push(e.target.value);
                              setUsers(newUsers);
                              e.target.value = '';
                            }
                          }}
                          className="w-full text-[10px] bg-white border border-slate-100 rounded-lg p-2 outline-none focus:ring-1 focus:ring-emerald-500"
                        />
                      </div>
                    </div>
                  ))}
                </div>
              </section>

              <section className="space-y-6">
                <div className="flex items-center justify-between border-b border-emerald-100 pb-2">
                  <h4 className="text-[10px] font-black text-emerald-600 uppercase tracking-[0.2em] flex items-center gap-2">
                    <Tag size={14}/> Gerenciar Categorias
                  </h4>
                </div>
                <div className="flex gap-2">
                  <input 
                    type="text" 
                    value={newCategory}
                    onChange={(e) => setNewCategory(e.target.value)}
                    placeholder="Nova Categoria..." 
                    className="flex-1 bg-slate-50 border border-slate-200 rounded-xl text-xs p-3 outline-none focus:ring-2 focus:ring-emerald-500/20"
                  />
                  <button 
                    onClick={() => {
                      if(newCategory.trim()) {
                        setCategories([...categories, newCategory]);
                        setNewCategory('');
                      }
                    }}
                    className="bg-emerald-600 text-white px-5 rounded-xl font-bold text-xs uppercase shadow-md hover:bg-emerald-700 active:scale-95 transition-all"
                  >
                    ADD
                  </button>
                </div>
                <div className="flex flex-wrap gap-2 p-4 bg-slate-50 rounded-[1.5rem] border border-slate-100">
                  {categories.map(cat => (
                    <div key={cat} className="bg-white border border-slate-200 px-3 py-1.5 rounded-xl text-[10px] font-black uppercase text-slate-600 flex items-center gap-3 shadow-sm group">
                      {cat}
                      <button onClick={() => setCategories(categories.filter(c => c !== cat))} className="text-slate-200 group-hover:text-red-500 transition-colors">
                        <X size={12}/>
                      </button>
                    </div>
                  ))}
                </div>
              </section>

            </div>
            
            <div className="p-8 bg-slate-50 border-t border-slate-100 text-center">
              <button onClick={() => setShowConfigAdmin(false)} className="bg-slate-800 text-white px-10 py-3 rounded-2xl font-bold text-xs uppercase shadow-lg hover:bg-slate-700 transition-all active:scale-95">Fechar</button>
            </div>
          </div>
        </div>
      )}

      {/* FEEDBACK TOAST */}
      {feedback && (
        <div className="fixed bottom-24 left-1/2 -translate-x-1/2 z-[60] bg-slate-900 text-white text-[10px] font-black px-6 py-3 rounded-full flex items-center gap-3 uppercase border border-white/10 shadow-2xl animate-in slide-in-from-bottom-5">
          {isScanning ? <Loader2 size={14} className="animate-spin text-emerald-400" /> : <CheckCircle2 size={14} className="text-emerald-400" />} 
          {feedback}
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
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
      `}</style>
    </div>
  );
};

export default App;
