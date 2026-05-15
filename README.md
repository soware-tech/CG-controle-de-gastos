<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Controle de Gastos</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@300;400;500;600;700;800&display=swap');
        
        :root {
            --primary: #059669;
            --primary-light: #ecfdf5;
        }

        body { 
            font-family: 'Plus Jakarta Sans', sans-serif; 
            -webkit-tap-highlight-color: transparent;
            overscroll-behavior-y: contain;
            background-color: #f8fafc;
            color: #1e293b;
        }

        .glass {
            background: rgba(255, 255, 255, 0.75);
            backdrop-filter: blur(16px);
            -webkit-backdrop-filter: blur(16px);
        }

        .card-shadow {
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05), 0 2px 4px -1px rgba(0, 0, 0, 0.03);
        }

        .active-tab-glow {
            box-shadow: 0 0 20px rgba(16, 185, 129, 0.2);
        }

        .animate-scan { animation: scan 2.5s linear infinite; }
        @keyframes scan { 
            0% { top: 0%; opacity: 0.2; } 
            50% { opacity: 1; }
            100% { top: 100%; opacity: 0.2; } 
        }

        /* Esconder scrollbar */
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }

        input, select { outline: none; }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useMemo, useRef } = React;
        const apiKey = ""; // Canvas injeta automaticamente se configurado

        const Icon = ({ name, size = 20, className = "" }) => {
            return <i data-lucide={name} style={{width: size, height: size}} className={`inline-block ${className}`}></i>;
        };

        const App = () => {
            const [activeTab, setActiveTab] = useState('lancamentos');
            const [currentMonth, setCurrentMonth] = useState(new Date().getMonth());
            const [currentYear, setCurrentYear] = useState(new Date().getFullYear());
            const [isScanning, setIsScanning] = useState(false);
            const [feedback, setFeedback] = useState(null);
            const [showScanner, setShowScanner] = useState(false);
            const [showConfig, setShowConfig] = useState(false);
            
            const [newUserInput, setNewUserInput] = useState('');
            const [newCardInput, setNewCardInput] = useState({ userId: null, value: '' });
            const [newCategoryInput, setNewCategoryInput] = useState('');

            // Dados do App
            const [expenses, setExpenses] = useState(() => JSON.parse(localStorage.getItem('expenses_pro_v1') || '[]'));
            const [pendingExpenses, setPendingExpenses] = useState([]);
            const [users, setUsers] = useState(() => JSON.parse(localStorage.getItem('users_pro_v1') || '[{"id":"1","name":"Aline","cards":["1234"]},{"id":"2","name":"Roney","cards":["5678"]}]'));
            const [categories, setCategories] = useState(() => JSON.parse(localStorage.getItem('categories_pro_v1') || '["Mercado", "Farmácia", "Lazer", "Casa", "Aluguel", "Combustível", "Padaria", "Pets", "Delivery"]'));

            const [formData, setFormData] = useState({
                user: users[0]?.name || '',
                description: '',
                category: categories[0] || 'Geral',
                amount: '',
                date: new Date().toISOString().split('T')[0]
            });

            const videoRef = useRef(null);
            const canvasRef = useRef(null);
            const fileInputRef = useRef(null);

            // Sincronização de Ícones e LocalStorage
            useEffect(() => { if (window.lucide) window.lucide.createIcons(); }, [activeTab, showScanner, showConfig, pendingExpenses, expenses]);
            
            useEffect(() => {
                localStorage.setItem('expenses_pro_v1', JSON.stringify(expenses));
                localStorage.setItem('users_pro_v1', JSON.stringify(users));
                localStorage.setItem('categories_pro_v1', JSON.stringify(categories));
            }, [expenses, users, categories]);

            const notify = (msg, type = 'success') => {
                setFeedback({ msg, type });
                setTimeout(() => setFeedback(null), 3000);
            };

            const callGemini = async (prompt, imageData = null) => {
                try {
                    const userCardsInfo = users.map(u => `${u.name}: cartões final ${u.cards.join(', ')}`).join('; ');
                    const systemPrompt = `Você é um assistente financeiro de elite. Dados para identificar pagador: ${userCardsInfo}. Categorias: ${categories.join(', ')}. SEMPRE retorne JSON puro. Se não souber o usuário, use null.`;
                    
                    const payload = {
                        contents: [{ 
                            parts: [
                                { text: prompt }, 
                                ...(imageData ? [{ inlineData: { mimeType: "image/png", data: imageData } }] : [])
                            ] 
                        }],
                        systemInstruction: { parts: [{ text: systemPrompt }] },
                        generationConfig: { responseMimeType: "application/json" }
                    };

                    const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent?key=${apiKey}`, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });
                    
                    const result = await response.json();
                    return JSON.parse(result.candidates[0].content.parts[0].text);
                } catch (e) {
                    console.error("Erro IA:", e);
                    return null;
                }
            };

            const startCamera = async () => {
                setShowScanner(true);
                try {
                    const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
                    if (videoRef.current) videoRef.current.srcObject = stream;
                } catch (err) {
                    notify("Erro na câmera", "error");
                    setShowScanner(false);
                }
            };

            const capture = async () => {
                const context = canvasRef.current.getContext('2d');
                canvasRef.current.width = videoRef.current.videoWidth;
                canvasRef.current.height = videoRef.current.videoHeight;
                context.drawImage(videoRef.current, 0, 0);
                const base64 = canvasRef.current.toDataURL('image/png').split(',')[1];
                
                videoRef.current.srcObject.getTracks().forEach(t => t.stop());
                setShowScanner(false);
                setIsScanning(true);
                
                const data = await callGemini("Extraia: estabelecimento, valor (number), usuario, categoria.", base64);
                if (data) processAIResult(data);
                else notify("Erro na leitura", "error");
                setIsScanning(false);
            };

            const handleFileUpload = async (e) => {
                const files = Array.from(e.target.files);
                if (!files.length) return;
                setIsScanning(true);
                notify(`Processando ${files.length} arquivo(s)...`);

                for (const file of files) {
                    const base64 = await new Promise(r => {
                        const reader = new FileReader();
                        reader.onload = () => r(reader.result.split(',')[1]);
                        reader.readAsDataURL(file);
                    });
                    const data = await callGemini("Extraia: estabelecimento, valor, usuario, categoria.", base64);
                    if (data) processAIResult(data);
                }
                setIsScanning(false);
                e.target.value = null;
            };

            const processAIResult = (data) => {
                const newItem = {
                    id: Math.random().toString(36).substr(2, 9),
                    user: data.usuario || users[0].name,
                    description: data.estabelecimento || "Cupom Escaneado",
                    category: categories.includes(data.categoria) ? data.categoria : (categories[0] || "Geral"),
                    amount: parseFloat(data.valor) || 0,
                    date: new Date().toISOString().split('T')[0]
                };
                setPendingExpenses(prev => [newItem, ...prev]);
                notify("Cupom identificado!");
            };

            const addUser = () => {
                if (!newUserInput.trim()) return;
                setUsers([...users, { id: Date.now().toString(), name: newUserInput.trim(), cards: [] }]);
                setNewUserInput('');
                notify("Usuário adicionado");
            };

            const addCard = (userId) => {
                const val = newCardInput.value.trim();
                if (val.length !== 4 || isNaN(val)) {
                    notify("Digite 4 dígitos", "error");
                    return;
                }
                setUsers(users.map(u => u.id === userId ? { ...u, cards: [...u.cards, val] } : u));
                setNewCardInput({ userId: null, value: '' });
                notify("Cartão vinculado");
            };

            const addCategory = () => {
                if (!newCategoryInput.trim() || categories.includes(newCategoryInput.trim())) return;
                setCategories([...categories, newCategoryInput.trim()]);
                setNewCategoryInput('');
                notify("Categoria criada");
            };

            const filteredExpenses = useMemo(() => {
                return expenses.filter(e => {
                    const d = new Date(e.date + 'T00:00:00');
                    return d.getMonth() === currentMonth && d.getFullYear() === currentYear;
                }).sort((a, b) => new Date(b.date) - new Date(a.date));
            }, [expenses, currentMonth, currentYear]);

            const yearlyStats = useMemo(() => {
                const months = Array(12).fill(0);
                expenses.forEach(e => {
                    const d = new Date(e.date + 'T00:00:00');
                    if (d.getFullYear() === currentYear) {
                        months[d.getMonth()] += Number(e.amount);
                    }
                });
                return months;
            }, [expenses, currentYear]);

            const totalMonth = filteredExpenses.reduce((acc, curr) => acc + Number(curr.amount), 0);
            const maxMonthValue = Math.max(...yearlyStats, 1);

            return (
                <div className="min-h-screen pb-28">
                    {/* Feedback Toast */}
                    {feedback && (
                        <div className={`fixed top-6 left-1/2 -translate-x-1/2 z-[150] px-6 py-3 rounded-2xl shadow-2xl text-white font-bold animate-in fade-in slide-in-from-top-4 duration-300 ${feedback.type === 'error' ? 'bg-rose-500' : 'bg-emerald-600'}`}>
                            {feedback.msg}
                        </div>
                    )}

                    {/* Navbar Superior */}
                    <header className="bg-white/80 backdrop-blur-md border-b sticky top-0 z-50 px-4 h-16 flex items-center justify-between">
                        <div className="flex items-center gap-2">
                            <div className="bg-emerald-600 text-white p-2 rounded-xl">
                                <Icon name="Wallet" size={18} />
                            </div>
                            <span className="font-extrabold text-slate-800 tracking-tight">FINANÇAS PRO</span>
                        </div>
                        <button onClick={() => setShowConfig(true)} className="p-2 text-slate-400 hover:text-emerald-600 transition-colors">
                            <Icon name="Settings" size={22} />
                        </button>
                    </header>

                    {/* Seletor de Período */}
                    <div className="bg-emerald-600 text-white p-6 shadow-lg relative overflow-hidden">
                        <div className="absolute top-0 right-0 w-32 h-32 bg-white/10 rounded-full -mr-16 -mt-16 blur-2xl"></div>
                        <div className="max-w-sm mx-auto flex justify-between items-center relative z-10">
                            <button onClick={() => {
                                if (activeTab === 'anual') setCurrentYear(y => y - 1);
                                else {
                                    if (currentMonth === 0) { setCurrentMonth(11); setCurrentYear(y => y - 1); }
                                    else setCurrentMonth(m => m - 1);
                                }
                            }} className="p-2 bg-white/10 rounded-xl hover:bg-white/20 transition-all">
                                <Icon name="ChevronLeft" size={24} />
                            </button>
                            
                            <div className="text-center">
                                {activeTab !== 'anual' && (
                                    <div className="text-2xl font-black uppercase tracking-tighter">
                                        {["Jan", "Fev", "Mar", "Abr", "Mai", "Jun", "Jul", "Ago", "Set", "Out", "Nov", "Dez"][currentMonth]}
                                    </div>
                                )}
                                <div className={`${activeTab === 'anual' ? 'text-2xl font-black' : 'text-emerald-200 text-xs font-bold'} tracking-widest`}>
                                    {currentYear}
                                </div>
                            </div>

                            <button onClick={() => {
                                if (activeTab === 'anual') setCurrentYear(y => y + 1);
                                else {
                                    if (currentMonth === 11) { setCurrentMonth(0); setCurrentYear(y => y + 1); }
                                    else setCurrentMonth(m => m + 1);
                                }
                            }} className="p-2 bg-white/10 rounded-xl hover:bg-white/20 transition-all">
                                <Icon name="ChevronRight" size={24} />
                            </button>
                        </div>
                    </div>

                    <main className="p-4 max-w-md mx-auto space-y-5">
                        
                        {/* TAB: LANÇAMENTOS */}
                        {activeTab === 'lancamentos' && (
                            <>
                                {/* Ações de Entrada */}
                                <div className="grid grid-cols-3 gap-3">
                                    <button onClick={startCamera} className="bg-white p-4 rounded-3xl border border-slate-100 card-shadow flex flex-col items-center gap-2 active:scale-95 transition-all">
                                        <div className="bg-emerald-50 text-emerald-600 p-3 rounded-2xl"><Icon name="Camera" size={24} /></div>
                                        <span className="text-[10px] font-bold text-slate-500 uppercase">Câmera</span>
                                    </button>
                                    <button onClick={() => fileInputRef.current.click()} className="bg-white p-4 rounded-3xl border border-slate-100 card-shadow flex flex-col items-center gap-2 active:scale-95 transition-all">
                                        <div className="bg-blue-50 text-blue-600 p-3 rounded-2xl"><Icon name="Image" size={24} /></div>
                                        <span className="text-[10px] font-bold text-slate-500 uppercase">Arquivos</span>
                                        <input type="file" multiple accept="image/*" className="hidden" ref={fileInputRef} onChange={handleFileUpload} />
                                    </button>
                                    <button onClick={() => {
                                        const url = prompt("Cole a URL da NF-e:");
                                        if (url) {
                                            setIsScanning(true);
                                            callGemini(`Analise os dados deste link de nota fiscal: ${url}`).then(d => {
                                                if (d) processAIResult(d);
                                                setIsScanning(false);
                                            });
                                        }
                                    }} className="bg-white p-4 rounded-3xl border border-slate-100 card-shadow flex flex-col items-center gap-2 active:scale-95 transition-all">
                                        <div className="bg-amber-50 text-amber-600 p-3 rounded-2xl"><Icon name="Link" size={24} /></div>
                                        <span className="text-[10px] font-bold text-slate-500 uppercase">Link NF</span>
                                    </button>
                                </div>

                                {/* Itens Pendentes de IA */}
                                {pendingExpenses.length > 0 && (
                                    <div className="space-y-3">
                                        <div className="flex justify-between items-center px-1">
                                            <h3 className="text-xs font-black text-amber-600 uppercase tracking-widest flex items-center gap-2">
                                                <Icon name="Sparkles" size={14} /> Validar Lançamentos
                                            </h3>
                                            <button 
                                                onClick={() => {
                                                    setExpenses([...pendingExpenses.map(p => ({...p, id: Date.now() + Math.random()})), ...expenses]);
                                                    setPendingExpenses([]);
                                                    notify("Tudo salvo!");
                                                }}
                                                className="text-[10px] font-black text-emerald-600 underline"
                                            >
                                                SALVAR TODOS
                                            </button>
                                        </div>
                                        {pendingExpenses.map(p => (
                                            <div key={p.id} className="bg-amber-50 border-2 border-amber-200 p-4 rounded-[2rem] shadow-lg animate-in zoom-in-95">
                                                <div className="space-y-3">
                                                    <input 
                                                        className="w-full bg-white border-amber-100 rounded-xl p-3 font-bold text-slate-700 text-sm"
                                                        value={p.description}
                                                        onChange={e => setPendingExpenses(prev => prev.map(x => x.id === p.id ? {...x, description: e.target.value} : x))}
                                                    />
                                                    <div className="flex gap-2">
                                                        <select 
                                                            className="flex-1 bg-white border-amber-100 rounded-xl p-3 font-bold text-slate-600 text-xs"
                                                            value={p.user}
                                                            onChange={e => setPendingExpenses(prev => prev.map(x => x.id === p.id ? {...x, user: e.target.value} : x))}
                                                        >
                                                            {users.map(u => <option key={u.id} value={u.name}>{u.name}</option>)}
                                                        </select>
                                                        <input 
                                                            type="number"
                                                            className="w-24 bg-white border-amber-100 rounded-xl p-3 font-black text-emerald-600 text-right"
                                                            value={p.amount}
                                                            onChange={e => setPendingExpenses(prev => prev.map(x => x.id === p.id ? {...x, amount: parseFloat(e.target.value)} : x))}
                                                        />
                                                    </div>
                                                    <div className="flex gap-2">
                                                        <button 
                                                            onClick={() => setPendingExpenses(prev => prev.filter(x => x.id !== p.id))}
                                                            className="p-3 bg-white border-amber-200 text-rose-500 rounded-xl"
                                                        >
                                                            <Icon name="Trash2" size={18} />
                                                        </button>
                                                        <button 
                                                            onClick={() => {
                                                                setExpenses([{...p, id: Date.now()}, ...expenses]);
                                                                setPendingExpenses(prev => prev.filter(x => x.id !== p.id));
                                                                notify("Salvo!");
                                                            }}
                                                            className="flex-1 bg-emerald-600 text-white font-bold py-3 rounded-xl shadow-md flex items-center justify-center gap-2"
                                                        >
                                                            <Icon name="Check" size={18} /> Confirmar
                                                        </button>
                                                    </div>
                                                </div>
                                            </div>
                                        ))}
                                    </div>
                                )}

                                {/* Lista de Gastos Realizados */}
                                <div className="space-y-3">
                                    <div className="flex justify-between items-center px-1">
                                        <h3 className="text-xs font-black text-slate-400 uppercase tracking-widest">Histórico do Mês</h3>
                                        <span className="text-xs font-bold text-slate-700">R$ {totalMonth.toLocaleString('pt-BR', {minimumFractionDigits: 2})}</span>
                                    </div>
                                    {filteredExpenses.length === 0 ? (
                                        <div className="text-center py-12 bg-white rounded-[2rem] border-2 border-dashed border-slate-100 text-slate-300 font-medium">
                                            Nenhum registro encontrado
                                        </div>
                                    ) : (
                                        filteredExpenses.map(e => (
                                            <div key={e.id} className="bg-white p-4 rounded-3xl card-shadow border border-slate-50 flex items-center justify-between group">
                                                <div className="flex items-center gap-3">
                                                    <div className="w-10 h-10 bg-slate-50 text-slate-400 rounded-2xl flex items-center justify-center">
                                                        <Icon name="Tag" size={18} />
                                                    </div>
                                                    <div>
                                                        <div className="font-bold text-slate-700 leading-tight">{e.description}</div>
                                                        <div className="flex gap-2 items-center mt-0.5">
                                                            <span className="text-[9px] font-black bg-emerald-50 text-emerald-600 px-2 py-0.5 rounded-full uppercase">{e.user}</span>
                                                            <span className="text-[10px] text-slate-400">{e.category}</span>
                                                        </div>
                                                    </div>
                                                </div>
                                                <div className="flex flex-col items-end">
                                                    <div className="font-black text-slate-800 tracking-tight">R$ {Number(e.amount).toLocaleString('pt-BR', {minimumFractionDigits:2})}</div>
                                                    <button onClick={() => setExpenses(expenses.filter(x => x.id !== e.id))} className="text-slate-200 hover:text-rose-500 p-1 opacity-0 group-hover:opacity-100 transition-opacity">
                                                        <Icon name="Trash2" size={14} />
                                                    </button>
                                                </div>
                                            </div>
                                        ))
                                    )}
                                </div>
                            </>
                        )}

                        {/* TAB: MENSAL (RESUMO) */}
                        {activeTab === 'mensal' && (
                            <div className="space-y-6 animate-in fade-in slide-in-from-bottom-4">
                                <div className="bg-slate-900 rounded-[2.5rem] p-8 text-white relative overflow-hidden shadow-2xl">
                                    <div className="absolute top-0 right-0 w-40 h-40 bg-emerald-500/20 rounded-full -mr-20 -mt-20 blur-3xl"></div>
                                    <div className="text-emerald-400 font-bold text-xs uppercase tracking-widest mb-1">Gasto Total Mensal</div>
                                    <div className="text-4xl font-black tracking-tighter">R$ {totalMonth.toLocaleString('pt-BR', {minimumFractionDigits:2})}</div>
                                    
                                    <div className="mt-8 space-y-5">
                                        {users.map(u => {
                                            const val = filteredExpenses.filter(e => e.user === u.name).reduce((s, c) => s + Number(c.amount), 0);
                                            const perc = totalMonth > 0 ? (val / totalMonth) * 100 : 0;
                                            return (
                                                <div key={u.id} className="space-y-1.5">
                                                    <div className="flex justify-between text-xs font-bold uppercase tracking-wide">
                                                        <span className="text-white/60">{u.name}</span>
                                                        <span>R$ {val.toLocaleString('pt-BR')}</span>
                                                    </div>
                                                    <div className="h-2 bg-white/10 rounded-full overflow-hidden">
                                                        <div className="h-full bg-emerald-500 rounded-full transition-all duration-1000" style={{width: `${perc}%`}}></div>
                                                    </div>
                                                </div>
                                            )
                                        })}
                                    </div>
                                </div>

                                <div className="bg-white p-6 rounded-[2.5rem] card-shadow border border-slate-100">
                                    <h4 className="text-xs font-black text-slate-400 uppercase tracking-widest mb-4">Por Categoria</h4>
                                    <div className="space-y-4">
                                        {categories.map(cat => {
                                            const val = filteredExpenses.filter(e => e.category === cat).reduce((s, c) => s + Number(c.amount), 0);
                                            if (val === 0) return null;
                                            return (
                                                <div key={cat} className="flex items-center justify-between">
                                                    <div className="flex items-center gap-3">
                                                        <div className="w-8 h-8 bg-slate-50 rounded-xl flex items-center justify-center text-slate-400">
                                                            <Icon name="Tag" size={14} />
                                                        </div>
                                                        <span className="text-sm font-bold text-slate-600">{cat}</span>
                                                    </div>
                                                    <span className="text-sm font-black text-slate-800">R$ {val.toLocaleString('pt-BR')}</span>
                                                </div>
                                            )
                                        })}
                                    </div>
                                </div>
                            </div>
                        )}

                        {/* TAB: ANUAL */}
                        {activeTab === 'anual' && (
                            <div className="space-y-6 animate-in fade-in slide-in-from-bottom-4">
                                <div className="bg-white p-6 rounded-[2.5rem] card-shadow border border-slate-100">
                                    <h4 className="text-xs font-black text-slate-400 uppercase tracking-widest mb-6">Comparativo Mensal {currentYear}</h4>
                                    <div className="flex items-end justify-between h-48 px-2">
                                        {yearlyStats.map((val, i) => (
                                            <div key={i} className="flex flex-col items-center gap-2 group relative">
                                                <div className="absolute -top-8 left-1/2 -translate-x-1/2 bg-slate-800 text-white text-[8px] px-1.5 py-0.5 rounded opacity-0 group-hover:opacity-100 transition-opacity">
                                                    R${val.toFixed(0)}
                                                </div>
                                                <div 
                                                    className="w-5 bg-emerald-500 rounded-t-lg transition-all duration-700 group-hover:bg-emerald-400" 
                                                    style={{height: `${(val / maxMonthValue) * 100}%`, minHeight: val > 0 ? '4px' : '0'}}
                                                ></div>
                                                <span className="text-[9px] font-black text-slate-300 uppercase">
                                                    {["J", "F", "M", "A", "M", "J", "J", "A", "S", "O", "N", "D"][i]}
                                                </span>
                                            </div>
                                        ))}
                                    </div>
                                </div>

                                <div className="bg-emerald-50 p-6 rounded-[2.5rem] border border-emerald-100">
                                    <div className="flex justify-between items-center">
                                        <div>
                                            <div className="text-[10px] font-black text-emerald-600 uppercase tracking-widest">Média Mensal</div>
                                            <div className="text-2xl font-black text-emerald-800">
                                                R$ {(yearlyStats.reduce((a,b)=>a+b,0) / 12).toLocaleString('pt-BR', {maximumFractionDigits:0})}
                                            </div>
                                        </div>
                                        <div className="bg-emerald-600 text-white p-4 rounded-3xl shadow-lg">
                                            <Icon name="TrendingUp" size={24} />
                                        </div>
                                    </div>
                                </div>
                            </div>
                        )}
                    </main>

                    {/* Navigation Bar Inferior */}
                    <nav className="fixed bottom-0 left-0 right-0 glass border-t h-20 px-8 flex items-center justify-around z-[100]">
                        <button onClick={() => setActiveTab('lancamentos')} className={`flex flex-col items-center gap-1 transition-colors ${activeTab === 'lancamentos' ? 'text-emerald-600' : 'text-slate-300'}`}>
                            <Icon name="PlusCircle" size={26} className={activeTab === 'lancamentos' ? 'active-tab-glow' : ''} />
                            <span className="text-[9px] font-black uppercase tracking-widest">Início</span>
                        </button>
                        <button onClick={() => setActiveTab('mensal')} className={`flex flex-col items-center gap-1 transition-colors ${activeTab === 'mensal' ? 'text-emerald-600' : 'text-slate-300'}`}>
                            <Icon name="PieChart" size={26} className={activeTab === 'mensal' ? 'active-tab-glow' : ''} />
                            <span className="text-[9px] font-black uppercase tracking-widest">Resumo</span>
                        </button>
                        <button onClick={() => setActiveTab('anual')} className={`flex flex-col items-center gap-1 transition-colors ${activeTab === 'anual' ? 'text-emerald-600' : 'text-slate-300'}`}>
                            <Icon name="BarChart3" size={26} className={activeTab === 'anual' ? 'active-tab-glow' : ''} />
                            <span className="text-[9px] font-black uppercase tracking-widest">Anual</span>
                        </button>
                    </nav>

                    {/* Scanner Modal Fullscreen */}
                    {showScanner && (
                        <div className="fixed inset-0 z-[200] bg-black flex flex-col">
                            <video ref={videoRef} autoPlay playsInline className="w-full h-full object-cover" />
                            
                            <div className="absolute inset-8 border-2 border-emerald-500/30 rounded-[3rem] pointer-events-none">
                                <div className="absolute inset-x-0 top-0 h-1 bg-emerald-400 animate-scan shadow-[0_0_15px_rgba(52,211,153,1)]"></div>
                            </div>

                            <button onClick={() => {
                                videoRef.current.srcObject.getTracks().forEach(t => t.stop());
                                setShowScanner(false);
                            }} className="absolute top-10 right-6 text-white p-3 bg-white/10 rounded-full backdrop-blur-md">
                                <Icon name="X" size={24} />
                            </button>

                            <div className="absolute bottom-12 inset-x-0 flex flex-col items-center gap-6">
                                <p className="text-white/70 text-xs font-bold uppercase tracking-widest bg-black/40 px-4 py-2 rounded-full backdrop-blur-sm">
                                    Aponte para o Cupom Fiscal
                                </p>
                                <button onClick={capture} className="w-20 h-20 bg-white rounded-full border-[6px] border-emerald-500 flex items-center justify-center active:scale-90 transition-all shadow-2xl">
                                    <div className="w-12 h-12 bg-emerald-500 rounded-full opacity-20 animate-pulse"></div>
                                </button>
                            </div>
                            <canvas ref={canvasRef} className="hidden" />
                        </div>
                    )}

                    {/* Modal de Configurações (Admin) */}
                    {showConfig && (
                        <div className="fixed inset-0 z-[250] bg-slate-900/60 backdrop-blur-sm flex items-end sm:items-center justify-center p-4">
                            <div className="bg-white w-full max-w-md rounded-[2.5rem] overflow-hidden shadow-2xl animate-in slide-in-from-bottom-10">
                                <div className="p-6 border-b flex justify-between items-center bg-slate-50">
                                    <div className="flex items-center gap-2">
                                        <Icon name="Settings" size={18} className="text-emerald-600" />
                                        <h2 className="font-black text-slate-800 uppercase tracking-widest text-xs">Configurações</h2>
                                    </div>
                                    <button onClick={() => setShowConfig(false)} className="text-slate-400 bg-white p-2 rounded-full shadow-sm">
                                        <Icon name="X" size={20} />
                                    </button>
                                </div>
                                
                                <div className="p-6 space-y-8 max-h-[75vh] overflow-y-auto no-scrollbar pb-10">
                                    
                                    {}
                                    <section className="space-y-4">
                                        <h3 className="text-[10px] font-black text-slate-400 uppercase tracking-widest flex items-center gap-2">
                                            <Icon name="Users" size={12} /> Usuários e Cartões
                                        </h3>
                                        
                                        <div className="space-y-3">
                                            {users.map(u => (
                                                <div key={u.id} className="bg-slate-50 p-4 rounded-3xl border border-slate-100 transition-all">
                                                    <div className="flex justify-between items-center mb-3">
                                                        <span className="font-extrabold text-slate-700 text-sm">{u.name}</span>
                                                        <button onClick={() => setUsers(users.filter(x => x.id !== u.id))} className="text-rose-400 hover:text-rose-600 p-1">
                                                            <Icon name="UserMinus" size={16}/>
                                                        </button>
                                                    </div>
                                                    
                                                    <div className="flex flex-wrap gap-2 mb-3">
                                                        {u.cards.map((c, idx) => (
                                                            <span key={idx} className="bg-white border border-slate-200 text-[10px] font-bold px-2 py-1 rounded-lg flex items-center gap-1.5 shadow-sm">
                                                                <Icon name="CreditCard" size={10} className="text-emerald-500" /> 
                                                                <span className="text-slate-600">•••• {c}</span>
                                                                <button onClick={() => {
                                                                    const newCards = u.cards.filter((_, i) => i !== idx);
                                                                    setUsers(users.map(x => x.id === u.id ? {...x, cards: newCards} : x));
                                                                }} className="text-slate-300 hover:text-rose-500 ml-1">×</button>
                                                            </span>
                                                        ))}
                                                    </div>

                                                    {/* Input para novo cartão funcional */}
                                                    {newCardInput.userId === u.id ? (
                                                        <div className="flex gap-2 animate-in fade-in zoom-in-95 duration-200">
                                                            <input 
                                                                autoFocus
                                                                maxLength={4}
                                                                placeholder="Final (4 dígitos)"
                                                                className="flex-1 bg-white border border-emerald-200 rounded-xl px-3 py-2 text-[11px] font-bold outline-none ring-2 ring-emerald-500/10"
                                                                value={newCardInput.value}
                                                                onChange={e => setNewCardInput({...newCardInput, value: e.target.value.replace(/\D/g, '')})}
                                                                onKeyPress={e => e.key === 'Enter' && addCard(u.id)}
                                                            />
                                                            <button onClick={() => addCard(u.id)} className="bg-emerald-600 text-white px-3 rounded-xl">
                                                                <Icon name="Check" size={14} />
                                                            </button>
                                                            <button onClick={() => setNewCardInput({userId: null, value: ''})} className="bg-slate-200 text-slate-500 px-3 rounded-xl">
                                                                <Icon name="X" size={14} />
                                                            </button>
                                                        </div>
                                                    ) : (
                                                        <button 
                                                            onClick={() => setNewCardInput({userId: u.id, value: ''})}
                                                            className="text-[10px] font-bold text-emerald-600 flex items-center gap-1 hover:opacity-70"
                                                        >
                                                            <Icon name="Plus" size={12} /> Vincular Cartão
                                                        </button>
                                                    )}
                                                </div>
                                            ))}
                                            
                                            {/* Input para novo usuário */}
                                            <div className="flex gap-2 mt-4">
                                                <input 
                                                    placeholder="Nome do Novo Usuário"
                                                    className="flex-1 bg-white border-2 border-dashed border-slate-200 rounded-2xl px-4 py-3 text-xs font-bold outline-none focus:border-emerald-300 focus:bg-emerald-50/30 transition-all"
                                                    value={newUserInput}
                                                    onChange={e => setNewUserInput(e.target.value)}
                                                    onKeyPress={e => e.key === 'Enter' && addUser()}
                                                />
                                                <button onClick={addUser} className="bg-slate-800 text-white px-5 rounded-2xl active:scale-95 transition-all">
                                                    <Icon name="UserPlus" size={18} />
                                                </button>
                                            </div>
                                        </div>
                                    </section>

                                    {}
                                    <section className="space-y-4">
                                        <h3 className="text-[10px] font-black text-slate-400 uppercase tracking-widest flex items-center gap-2">
                                            <Icon name="Grid" size={12} /> Categorias
                                        </h3>
                                        <div className="flex flex-wrap gap-2">
                                            {categories.map(cat => (
                                                <span key={cat} className="bg-emerald-50 text-emerald-700 text-[10px] font-bold px-3 py-1.5 rounded-xl border border-emerald-100 flex items-center gap-2 shadow-sm">
                                                    {cat}
                                                    <button onClick={() => setCategories(categories.filter(c => c !== cat))} className="text-emerald-300 hover:text-emerald-600">×</button>
                                                </span>
                                            ))}
                                        </div>
                                        <div className="flex gap-2">
                                            <input 
                                                placeholder="Nova Categoria"
                                                className="flex-1 bg-slate-50 border border-slate-100 rounded-xl px-4 py-2 text-xs font-bold outline-none focus:ring-2 ring-emerald-500/20"
                                                value={newCategoryInput}
                                                onChange={e => setNewCategoryInput(e.target.value)}
                                                onKeyPress={e => e.key === 'Enter' && addCategory()}
                                            />
                                            <button onClick={addCategory} className="bg-emerald-100 text-emerald-600 px-4 rounded-xl font-bold text-xs hover:bg-emerald-200">
                                                Criar
                                            </button>
                                        </div>
                                    </section>
                                    
                                    <div className="pt-4">
                                        <button onClick={() => {
                                            if(confirm("Deseja apagar todos os gastos cadastrados? Esta ação é irreversível.")) {
                                                setExpenses([]);
                                                notify("Histórico removido", "error");
                                            }
                                        }} className="w-full py-4 bg-rose-50 text-rose-500 rounded-2xl font-black text-[10px] uppercase tracking-widest flex items-center justify-center gap-2 border border-rose-100 active:bg-rose-100 transition-colors">
                                            <Icon name="AlertTriangle" size={14} /> Limpar Todos os Gastos
                                        </button>
                                    </div>
                                </div>
                            </div>
                        </div>
                    )}

                    {/* Overlay de Processamento IA */}
                    {isScanning && (
                        <div className="fixed inset-0 z-[300] bg-white/90 backdrop-blur-md flex flex-col items-center justify-center">
                            <div className="relative">
                                <div className="w-24 h-24 border-4 border-emerald-100 border-t-emerald-600 rounded-full animate-spin"></div>
                                <div className="absolute inset-0 flex items-center justify-center text-emerald-600">
                                    <Icon name="Sparkles" size={32} className="animate-pulse" />
                                </div>
                            </div>
                            <h3 className="mt-8 font-black text-slate-800 tracking-tight uppercase text-sm">IA Inteligente Analisando...</h3>
                            <p className="text-slate-400 text-xs mt-2 font-medium">Extraindo dados do seu cupom fiscal</p>
                        </div>
                    )}
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
