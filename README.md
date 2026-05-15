<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CG - Controle de Gastos</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;800;900&display=swap');
        
        body { font-family: 'Inter', sans-serif; }

        @keyframes scan-line {
            0% { top: 0%; }
            100% { top: 100%; }
        }
        .animate-scan-line {
            position: absolute;
            width: 100%;
            height: 2px;
            background: rgba(16, 185, 129, 0.5);
            box-shadow: 0 0 15px rgba(16, 185, 129, 0.5);
            animation: scan-line 2s linear infinite;
        }

        .tab-active {
            background: white;
            box-shadow: 0 1px 2px rgba(0,0,0,0.05);
            color: #047857; /* emerald-700 */
        }

        /* Esconder scrollbar mas manter funcionalidade */
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
    </style>
</head>
<body class="bg-[#f8fafc] text-slate-900 pb-20">

    <!-- NAVBAR -->
    <nav class="bg-white border-b border-slate-200 sticky top-0 z-20">
        <div class="max-w-6xl mx-auto px-4 flex justify-between items-center h-16">
            <div class="flex items-center gap-2">
                <div class="bg-emerald-600 p-1.5 rounded text-white font-bold text-sm">CG</div>
                <span class="font-bold text-lg text-slate-700 hidden sm:block">Controle de Gastos</span>
            </div>
            <div class="flex bg-slate-100 p-1 rounded-xl" id="tab-container">
                <button onclick="switchTab('lancamentos')" id="tab-lancamentos" class="px-3 sm:px-5 py-1.5 rounded-lg text-[10px] sm:text-xs font-black transition-all uppercase tab-active">Lançar</button>
                <button onclick="switchTab('dashboard')" id="tab-dashboard" class="px-3 sm:px-5 py-1.5 rounded-lg text-[10px] sm:text-xs font-black transition-all uppercase text-slate-400">Mensal</button>
                <button onclick="switchTab('anual')" id="tab-anual" class="px-3 sm:px-5 py-1.5 rounded-lg text-[10px] sm:text-xs font-black transition-all uppercase text-slate-400">Anual</button>
            </div>
            <button onclick="toggleConfig(true)" class="p-2 text-slate-400 hover:text-emerald-600">
                <i data-lucide="settings" size="20"></i>
            </button>
        </div>
    </nav>

    <!-- HEADER DINÂMICO -->
    <div class="bg-emerald-700 text-white py-4 shadow-inner">
        <div class="max-w-5xl mx-auto px-4 flex items-center justify-between">
            <button onclick="changePeriod(-1)" class="p-1 hover:bg-emerald-600 rounded-full"><i data-lucide="chevron-left"></i></button>
            <div class="text-center">
                <h2 id="display-month" class="font-bold text-lg">Janeiro</h2>
                <p id="display-year" class="text-[10px] opacity-70 uppercase tracking-widest font-black">2024</p>
            </div>
            <button onclick="changePeriod(1)" class="p-1 hover:bg-emerald-600 rounded-full"><i data-lucide="chevron-right"></i></button>
        </div>
    </div>

    <main class="max-w-5xl mx-auto p-4 md:p-6">
        
        <!-- SEÇÃO LANÇAMENTOS -->
        <div id="section-lancamentos" class="space-y-6">
            <!-- INPUTS INTELIGENTES -->
            <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
                <div class="bg-white border border-emerald-100 rounded-2xl p-4 shadow-sm space-y-3">
                    <div class="flex items-center gap-3">
                        <div class="bg-emerald-100 p-2 rounded-xl text-emerald-600"><i data-lucide="camera" size="20"></i></div>
                        <h4 class="text-xs font-black text-slate-700 uppercase">Scanner ao Vivo</h4>
                    </div>
                    <button onclick="startCamera()" class="w-full bg-emerald-600 text-white font-bold py-2.5 rounded-xl text-xs uppercase flex items-center justify-center gap-2 shadow-sm">
                        Abrir Câmera
                    </button>
                </div>

                <div class="bg-white border border-blue-100 rounded-2xl p-4 shadow-sm space-y-3">
                    <div class="flex items-center gap-3">
                        <div class="bg-blue-100 p-2 rounded-xl text-blue-600"><i data-lucide="files" size="20"></i></div>
                        <h4 class="text-xs font-black text-slate-700 uppercase">Upload</h4>
                    </div>
                    <input type="file" id="file-upload" multiple accept="image/*" class="hidden" onchange="handleFileUpload(event)">
                    <button onclick="document.getElementById('file-upload').click()" class="w-full bg-blue-50 border-2 border-dashed border-blue-200 text-blue-700 rounded-xl text-xs font-bold py-2.5 flex items-center justify-center gap-2">
                        <i data-lucide="image" size="16"></i> Galeria
                    </button>
                </div>

                <div class="bg-white border border-slate-100 rounded-2xl p-4 shadow-sm space-y-3">
                    <div class="flex items-center gap-3">
                        <div class="bg-slate-100 p-2 rounded-xl text-slate-600"><i data-lucide="scan-qr-code" size="20"></i></div>
                        <h4 class="text-xs font-black text-slate-700 uppercase">Link de Nota</h4>
                    </div>
                    <div class="flex gap-2">
                        <input type="text" id="link-input" placeholder="Cole o link..." class="flex-1 bg-slate-50 border-slate-100 rounded-lg text-xs p-2">
                        <button onclick="processLink()" class="bg-slate-800 text-white px-3 py-2 rounded-lg font-bold text-xs">PROCESSAR</button>
                    </div>
                </div>
            </div>

            <!-- PENDENTES IA -->
            <div id="pending-container" class="hidden bg-amber-50 border border-amber-200 rounded-2xl overflow-hidden shadow-md">
                <div class="p-4 bg-amber-100/50 flex justify-between items-center border-b border-amber-200">
                    <h3 class="text-xs font-black text-amber-800 uppercase flex items-center gap-2">
                        <i data-lucide="sparkles" size="14"></i> Validar Lançamentos IA (<span id="pending-count">0</span>)
                    </h3>
                    <button onclick="saveAllPending()" class="bg-amber-600 text-white px-3 py-1 rounded-lg text-[10px] font-bold uppercase flex items-center gap-1">
                        <i data-lucide="save" size="12"></i> Salvar Todos
                    </button>
                </div>
                <div id="pending-list" class="p-2 space-y-2"></div>
            </div>

            <!-- FORM MANUAL -->
            <form id="manual-form" class="bg-white rounded-2xl shadow-sm border border-slate-200 p-6 grid grid-cols-1 md:grid-cols-5 gap-4 items-end">
                <div class="space-y-1">
                    <label class="text-[10px] font-bold text-slate-400 uppercase">Usuário</label>
                    <select id="form-user" class="w-full bg-slate-50 border-slate-200 rounded-lg text-sm"></select>
                </div>
                <div class="space-y-1">
                    <label class="text-[10px] font-bold text-slate-400 uppercase">Categoria</label>
                    <select id="form-category" class="w-full bg-slate-50 border-slate-200 rounded-lg text-sm"></select>
                </div>
                <div class="space-y-1">
                    <label class="text-[10px] font-bold text-slate-400 uppercase">Local</label>
                    <input type="text" id="form-description" placeholder="Ex: Mercado Livre" class="w-full bg-slate-50 border-slate-200 rounded-lg text-sm">
                </div>
                <div class="space-y-1">
                    <label class="text-[10px] font-bold text-slate-400 uppercase">Valor R$</label>
                    <input type="number" id="form-amount" step="0.01" class="w-full border-emerald-200 rounded-lg text-sm font-bold">
                </div>
                <button type="submit" class="bg-emerald-600 text-white font-bold py-2 rounded-lg text-sm uppercase shadow-md hover:bg-emerald-700 transition-colors">Salvar</button>
            </form>

            <!-- TABELA -->
            <div class="bg-white rounded-2xl shadow-sm border border-slate-200 overflow-hidden">
                <div class="px-6 py-4 border-b border-slate-100 flex justify-between items-center">
                    <h3 class="text-xs font-black text-slate-500 uppercase tracking-widest">Registos do Mês</h3>
                </div>
                <div class="overflow-x-auto">
                    <table class="w-full text-left">
                        <thead class="bg-slate-50">
                            <tr>
                                <th class="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Data</th>
                                <th class="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Usuário</th>
                                <th class="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase">Local/Cat</th>
                                <th class="px-6 py-3 text-[10px] font-bold text-slate-400 uppercase text-right">Valor</th>
                                <th class="px-6 py-3 w-10"></th>
                            </tr>
                        </thead>
                        <tbody id="expense-table-body" class="divide-y divide-slate-100"></tbody>
                    </table>
                </div>
            </div>
        </div>

        <!-- DASHBOARD MENSAL -->
        <div id="section-dashboard" class="hidden space-y-6">
            <div class="flex flex-col md:flex-row gap-4 items-stretch">
                <div id="user-stats-container" class="flex-1 grid grid-cols-2 gap-4"></div>
                <div class="md:w-64 bg-emerald-50 border border-emerald-100 p-4 rounded-2xl shadow-sm flex items-center justify-between">
                    <div>
                        <h3 class="text-[9px] font-black text-emerald-600 uppercase tracking-wider mb-0.5">Gasto Total</h3>
                        <div id="monthly-total" class="text-xl font-black text-emerald-900 leading-tight">R$ 0,00</div>
                    </div>
                    <div class="bg-emerald-600 p-2 rounded-lg text-white"><i data-lucide="wallet" size="20"></i></div>
                </div>
            </div>
            <div class="bg-white rounded-2xl border border-slate-200 p-6 shadow-sm">
                <h3 class="font-bold text-slate-700 text-xs uppercase tracking-widest flex items-center gap-2 mb-6">
                    <i data-lucide="trending-up" class="text-emerald-500" size="16"></i> Categorias do Mês
                </h3>
                <div id="category-stats-list" class="grid grid-cols-1 md:grid-cols-2 gap-x-12 gap-y-5"></div>
            </div>
        </div>

        <!-- DASHBOARD ANUAL -->
        <div id="section-anual" class="hidden space-y-6">
            <div class="flex flex-col md:flex-row gap-4 items-stretch">
                <div id="user-yearly-stats" class="flex-1 grid grid-cols-2 gap-4"></div>
                <div class="md:w-72 bg-slate-900 text-white p-6 rounded-2xl shadow-xl flex items-center justify-between">
                    <div>
                        <h3 id="annual-title" class="text-[10px] font-black text-emerald-400 uppercase tracking-widest mb-1">Total Anual 2024</h3>
                        <div id="annual-total" class="text-2xl font-black tracking-tighter">R$ 0,00</div>
                    </div>
                    <div class="bg-white/10 p-3 rounded-xl"><i data-lucide="calendar-days" size="24"></i></div>
                </div>
            </div>
            <div class="bg-white rounded-2xl border border-slate-200 p-6 shadow-sm">
                <h3 class="font-bold text-slate-700 text-xs uppercase tracking-widest flex items-center gap-2 mb-8">
                    <i data-lucide="bar-chart-3" class="text-emerald-500" size="16"></i> Evolução Mensal
                </h3>
                <div id="monthly-chart" class="flex items-end justify-between h-48 gap-2 px-2">
                    <!-- Gerado dinamicamente -->
                </div>
            </div>
        </div>
    </main>

    <!-- MODAL SCANNER -->
    <div id="scanner-modal" class="fixed inset-0 z-[60] bg-black hidden flex-col items-center justify-center">
        <div class="relative w-full max-w-lg aspect-[3/4] bg-slate-900 overflow-hidden shadow-2xl">
            <video id="video-feed" autoplay playsinline class="w-full h-full object-cover"></video>
            <div class="absolute inset-0 border-[2px] border-white/30 m-8 pointer-events-none">
                <div class="animate-scan-line"></div>
            </div>
            <div class="absolute bottom-8 left-0 right-0 flex justify-around items-center px-8">
                <button onclick="stopCamera()" class="bg-white/10 backdrop-blur-md p-4 rounded-full text-white hover:bg-white/20">
                    <i data-lucide="x" size="24"></i>
                </button>
                <button onclick="capturePhoto()" class="w-20 h-20 bg-white rounded-full border-4 border-emerald-500 flex items-center justify-center shadow-lg active:scale-95">
                    <div class="w-16 h-16 rounded-full bg-white border-2 border-slate-200"></div>
                </button>
                <div class="w-14"></div>
            </div>
        </div>
        <canvas id="photo-canvas" class="hidden"></canvas>
        <p class="text-white/60 text-xs font-bold uppercase tracking-widest mt-6">Enquadre o Cupom Fiscal</p>
    </div>

    <!-- MODAL CONFIG -->
    <div id="config-modal" class="fixed inset-0 z-50 bg-slate-900/60 backdrop-blur-sm hidden items-center justify-center p-4">
        <div class="bg-white w-full max-w-2xl rounded-3xl shadow-2xl overflow-hidden flex flex-col max-h-[85vh]">
            <div class="p-6 border-b border-slate-100 flex justify-between items-center bg-slate-50">
                <h3 class="font-black text-slate-800 uppercase tracking-widest text-sm flex items-center gap-2">
                    <i data-lucide="settings" size="18" class="text-emerald-600"></i> Configurações
                </h3>
                <button onclick="toggleConfig(false)" class="text-slate-400"><i data-lucide="x" size="20"></i></button>
            </div>
            <div class="p-6 space-y-8 overflow-y-auto no-scrollbar">
                <!-- Usuários -->
                <section class="space-y-4">
                    <h4 class="text-[10px] font-black text-emerald-600 uppercase tracking-[2px] border-b border-emerald-100 pb-2">Gerenciar Usuários</h4>
                    <div class="flex gap-2">
                        <input type="text" id="new-user-name" placeholder="Nome..." class="flex-1 bg-slate-50 border-slate-200 rounded-xl text-sm p-2">
                        <button onclick="addUser()" class="bg-emerald-600 text-white px-4 py-2 rounded-xl font-bold text-xs">ADD</button>
                    </div>
                    <div id="user-list" class="grid grid-cols-1 sm:grid-cols-2 gap-4"></div>
                </section>
                <!-- Categorias -->
                <section class="space-y-4">
                    <h4 class="text-[10px] font-black text-emerald-600 uppercase tracking-[2px] border-b border-emerald-100 pb-2">Gerenciar Categorias</h4>
                    <div class="flex gap-2">
                        <input type="text" id="new-cat-name" placeholder="Nome..." class="flex-1 bg-slate-50 border-slate-200 rounded-xl text-sm p-2">
                        <button onclick="addCategory()" class="bg-emerald-600 text-white px-4 py-2 rounded-xl font-bold text-xs">ADD</button>
                    </div>
                    <div id="category-list" class="flex flex-wrap gap-2 p-2 bg-slate-50 rounded-2xl"></div>
                </section>
            </div>
            <div class="p-4 bg-slate-50 border-t text-center">
                <button onclick="toggleConfig(false)" class="bg-slate-800 text-white px-8 py-2 rounded-xl font-bold text-xs uppercase">Fechar</button>
            </div>
        </div>
    </div>

    <!-- FEEDBACK TOAST -->
    <div id="toast" class="fixed bottom-24 left-1/2 -translate-x-1/2 z-50 hidden bg-slate-800 text-white text-[10px] font-black px-6 py-2.5 rounded-full items-center gap-2 uppercase border border-slate-700 shadow-xl">
        <span id="toast-icon"><i data-lucide="check-circle-2" size="14"></i></span>
        <span id="toast-text">Feedback</span>
    </div>

    <script>
        // --- ESTADO INICIAL ---
        const apiKey = ""; // Substituir se necessário
        let expenses = JSON.parse(localStorage.getItem('expenses_v9')) || [];
        let users = JSON.parse(localStorage.getItem('users_v9')) || [
            { id: '1', name: 'Aline', cards: ['1234'] },
            { id: '2', name: 'Roney', cards: ['5678'] }
        ];
        let categories = JSON.parse(localStorage.getItem('categories_v9')) || [
            'Aluguel', 'Beleza', 'Casa', 'Combustível', 'Energia', 'Estacionamento', 
            'Manutenção Carro', 'Feira', 'Fast Food', 'Delivery', 'Gás', 
            'Mercado', 'Farmácia', 'Internet', 'Celular', 'Padaria', 'Pets'
        ];

        let pendingExpenses = [];
        let currentTab = 'lancamentos';
        let currentMonth = new Date().getMonth();
        let currentYear = new Date().getFullYear();
        const monthNames = ["Janeiro", "Fevereiro", "Março", "Abril", "Maio", "Junho", "Julho", "Agosto", "Setembro", "Outubro", "Novembro", "Dezembro"];

        // --- INICIALIZAÇÃO ---
        window.onload = () => {
            lucide.createIcons();
            updateUI();
            setupForm();
        };

        function saveLocal() {
            localStorage.setItem('expenses_v9', JSON.stringify(expenses));
            localStorage.setItem('users_v9', JSON.stringify(users));
            localStorage.setItem('categories_v9', JSON.stringify(categories));
        }

        // --- GESTÃO DE UI ---
        function switchTab(tab) {
            currentTab = tab;
            document.querySelectorAll('#tab-container button').forEach(b => b.classList.remove('tab-active', 'text-slate-400'));
            document.getElementById(`tab-${tab}`).classList.add('tab-active');
            
            document.getElementById('section-lancamentos').classList.add('hidden');
            document.getElementById('section-dashboard').classList.add('hidden');
            document.getElementById('section-anual').classList.add('hidden');
            document.getElementById(`section-${tab}`).classList.remove('hidden');
            
            updateUI();
        }

        function changePeriod(delta) {
            if (currentTab === 'anual') {
                currentYear += delta;
            } else {
                currentMonth += delta;
                if (currentMonth < 0) { currentMonth = 11; currentYear--; }
                if (currentMonth > 11) { currentMonth = 0; currentYear++; }
            }
            updateUI();
        }

        function updateUI() {
            document.getElementById('display-month').innerText = currentTab === 'anual' ? 'Anual' : monthNames[currentMonth];
            document.getElementById('display-year').innerText = currentYear;
            
            if (currentTab === 'lancamentos') renderExpenses();
            if (currentTab === 'dashboard') renderDashboard();
            if (currentTab === 'anual') renderAnnual();
            
            setupForm(); // Atualiza dropdowns
            lucide.createIcons();
        }

        function setupForm() {
            const uSel = document.getElementById('form-user');
            const cSel = document.getElementById('form-category');
            if(uSel && cSel) {
                uSel.innerHTML = users.map(u => `<option value="${u.name}">${u.name}</option>`).join('');
                cSel.innerHTML = categories.map(c => `<option value="${c}">${c}</option>`).join('');
            }
        }

        // --- LANÇAMENTOS ---
        function renderExpenses() {
            const list = expenses.filter(e => {
                const d = new Date(e.date + 'T00:00:00');
                return d.getMonth() === currentMonth && d.getFullYear() === currentYear;
            }).sort((a,b) => new Date(b.date) - new Date(a.date));

            const tbody = document.getElementById('expense-table-body');
            tbody.innerHTML = list.map(e => `
                <tr class="hover:bg-slate-50 transition-colors">
                    <td class="px-6 py-3 text-xs text-slate-500">${new Date(e.date + 'T00:00:00').toLocaleDateString('pt-BR', {day:'2-digit', month:'2-digit'})}</td>
                    <td class="px-6 py-3 text-[10px] font-black uppercase text-slate-600">${e.user}</td>
                    <td class="px-6 py-3">
                        <p class="text-xs font-bold leading-none">${e.description || 'Sem descrição'}</p>
                        <p class="text-[9px] text-slate-400 uppercase mt-1">${e.category}</p>
                    </td>
                    <td class="px-6 py-3 text-sm font-black text-right">R$ ${Number(e.amount).toLocaleString('pt-BR', {minimumFractionDigits:2})}</td>
                    <td class="px-6 py-3">
                        <button onclick="deleteExpense(${e.id})" class="text-slate-300 hover:text-red-500"><i data-lucide="trash-2" size="14"></i></button>
                    </td>
                </tr>
            `).join('');
            lucide.createIcons();
        }

        document.getElementById('manual-form').onsubmit = (e) => {
            e.preventDefault();
            const amount = document.getElementById('form-amount').value;
            if(!amount) return;
            
            const newExp = {
                id: Date.now(),
                user: document.getElementById('form-user').value,
                category: document.getElementById('form-category').value,
                description: document.getElementById('form-description').value,
                amount: parseFloat(amount),
                date: new Date().toISOString().split('T')[0]
            };
            
            expenses.unshift(newExp);
            saveLocal();
            updateUI();
            e.target.reset();
            showToast("Lançamento salvo!");
        };

        function deleteExpense(id) {
            expenses = expenses.filter(e => e.id !== id);
            saveLocal();
            updateUI();
        }

        // --- IA & GEMINI ---
        async function callGemini(prompt, imageData = null) {
            if (!apiKey) { showToast("API Key não configurada", true); return null; }
            
            const userCardsInfo = users.map(u => `${u.name}: cartões final ${u.cards.join(', ')}`).join('; ');
            const systemPrompt = `Você é um assistente de finanças. Use estes dados para identificar quem pagou: ${userCardsInfo}. Retorne sempre JSON puro. Se não identificar o usuário, retorne null no campo usuario. Categorias disponíveis: ${categories.join(', ')}`;
            
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

            try {
                const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });
                const result = await res.json();
                return JSON.parse(result.candidates?.[0]?.content?.parts?.[0]?.text || "null");
            } catch (error) {
                console.error(error);
                return null;
            }
        }

        async function processAIResult(data) {
            if(!data) return;
            const newItem = {
                id: Math.random().toString(36).substr(2, 9),
                user: data.usuario || users[0].name,
                description: data.estabelecimento || "Cupom Escaneado",
                category: categories.includes(data.categoria) ? data.categoria : (categories[0] || "Geral"),
                amount: data.valor || 0,
                date: new Date().toISOString().split('T')[0]
            };
            pendingExpenses.unshift(newItem);
            renderPending();
        }

        function renderPending() {
            const container = document.getElementById('pending-container');
            const list = document.getElementById('pending-list');
            const count = document.getElementById('pending-count');
            
            if (pendingExpenses.length === 0) {
                container.classList.add('hidden');
                return;
            }
            
            container.classList.remove('hidden');
            count.innerText = pendingExpenses.length;
            
            list.innerHTML = pendingExpenses.map(p => `
                <div class="bg-white p-3 rounded-xl border border-amber-100 grid grid-cols-1 md:grid-cols-6 gap-2 items-center">
                    <select onchange="updatePending('${p.id}', 'user', this.value)" class="text-[10px] font-bold border-slate-100 rounded bg-slate-50 uppercase">
                        ${users.map(u => `<option value="${u.name}" ${p.user === u.name ? 'selected' : ''}>${u.name}</option>`).join('')}
                    </select>
                    <input class="text-xs border-slate-100 rounded col-span-2 p-1" value="${p.description}" onchange="updatePending('${p.id}', 'description', this.value)">
                    <select onchange="updatePending('${p.id}', 'category', this.value)" class="text-[10px] border-slate-100 rounded bg-slate-50">
                        ${categories.map(c => `<option value="${c}" ${p.category === c ? 'selected' : ''}>${c}</option>`).join('')}
                    </select>
                    <div class="font-black text-xs text-right">R$ ${parseFloat(p.amount).toFixed(2)}</div>
                    <div class="flex justify-end gap-2">
                        <button onclick="removePending('${p.id}')" class="p-1.5 text-slate-300 hover:text-red-500"><i data-lucide="trash-2" size="14"></i></button>
                        <button onclick="confirmPending('${p.id}')" class="p-1.5 text-emerald-500 hover:bg-emerald-50 rounded"><i data-lucide="check-circle-2" size="16"></i></button>
                    </div>
                </div>
            `).join('');
            lucide.createIcons();
        }

        function updatePending(id, field, value) {
            const idx = pendingExpenses.findIndex(p => p.id === id);
            if(idx !== -1) pendingExpenses[idx][field] = value;
        }

        function removePending(id) {
            pendingExpenses = pendingExpenses.filter(p => p.id !== id);
            renderPending();
        }

        function confirmPending(id) {
            const item = pendingExpenses.find(p => p.id === id);
            if(item) {
                const { id: _, ...clean } = item;
                expenses.unshift({ ...clean, id: Date.now() });
                saveLocal();
                removePending(id);
                updateUI();
                showToast("Lançamento confirmado!");
            }
        }

        function saveAllPending() {
            pendingExpenses.forEach(p => {
                const { id: _, ...clean } = p;
                expenses.unshift({ ...clean, id: Date.now() + Math.random() });
            });
            pendingExpenses = [];
            saveLocal();
            updateUI();
            renderPending();
            showToast("Todos salvos!");
        }

        // --- CÂMERA ---
        let stream = null;
        async function startCamera() {
            document.getElementById('scanner-modal').classList.remove('hidden');
            try {
                stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
                document.getElementById('video-feed').srcObject = stream;
            } catch (err) {
                showToast("Erro ao acessar câmera", true);
                stopCamera();
            }
        }

        function stopCamera() {
            if (stream) stream.getTracks().forEach(t => t.stop());
            document.getElementById('scanner-modal').classList.add('hidden');
        }

        async function capturePhoto() {
            const video = document.getElementById('video-feed');
            const canvas = document.getElementById('photo-canvas');
            const context = canvas.getContext('2d');
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            context.drawImage(video, 0, 0);
            
            const base64 = canvas.toDataURL('image/png').split(',')[1];
            stopCamera();
            showToast("IA Analisando...", false, true);
            
            const data = await callGemini("Extraia os dados deste cupom fiscal. JSON: {\"estabelecimento\": \"nome\", \"valor\": 0.00, \"usuario\": \"nome\", \"categoria\": \"sugerir_uma\"}", base64);
            if(data) processAIResult(data);
            hideToast();
        }

        async function handleFileUpload(e) {
            const files = Array.from(e.target.files).slice(0, 5);
            if(files.length === 0) return;
            
            showToast(`Processando ${files.length} arquivos...`, false, true);
            for(const file of files) {
                const base64 = await new Promise(r => {
                    const fr = new FileReader();
                    fr.onload = () => r(fr.result.split(',')[1]);
                    fr.readAsDataURL(file);
                });
                const data = await callGemini("Extraia os dados deste cupom. JSON: {\"estabelecimento\": \"nome\", \"valor\": 0.00, \"usuario\": \"nome\", \"categoria\": \"sugerir_uma\"}", base64);
                if(data) processAIResult(data);
            }
            hideToast();
            e.target.value = null;
        }

        async function processLink() {
            const input = document.getElementById('link-input');
            if(!input.value) return;
            showToast("Processando link...", false, true);
            const data = await callGemini(`Analise o link e extraia dados de gasto. JSON: {"estabelecimento": "nome", "valor": 0.0, "usuario": "nome", "categoria": "sugerir"}`, null);
            if(data) processAIResult(data);
            input.value = '';
            hideToast();
        }

        // --- DASHBOARDS ---
        function renderDashboard() {
            const filtered = expenses.filter(e => {
                const d = new Date(e.date + 'T00:00:00');
                return d.getMonth() === currentMonth && d.getFullYear() === currentYear;
            });

            const total = filtered.reduce((acc, curr) => acc + Number(curr.amount), 0);
            document.getElementById('monthly-total').innerText = `R$ ${total.toLocaleString('pt-BR', {minimumFractionDigits:2})}`;

            // Stats por Usuário
            const userStats = users.map(u => {
                const sum = filtered.filter(e => e.user === u.name).reduce((s, curr) => s + Number(curr.amount), 0);
                return { name: u.name, total: sum };
            });

            document.getElementById('user-stats-container').innerHTML = userStats.map(u => `
                <div class="bg-white p-4 rounded-2xl border border-slate-200 shadow-sm flex items-center gap-3">
                    <div class="bg-slate-50 p-2 rounded-xl text-slate-400"><i data-lucide="users" size="18"></i></div>
                    <div>
                        <p class="text-[9px] font-bold text-slate-400 uppercase leading-none mb-1">${u.name}</p>
                        <h4 class="text-sm font-black text-slate-800">R$ ${u.total.toLocaleString('pt-BR', {minimumFractionDigits:2})}</h4>
                    </div>
                </div>
            `).join('');

            // Stats por Categoria
            const catStats = categories.map(cat => {
                const sum = filtered.filter(e => e.category === cat).reduce((s, curr) => s + Number(curr.amount), 0);
                return { name: cat, total: sum };
            }).filter(c => c.total > 0).sort((a,b) => b.total - a.total);

            document.getElementById('category-stats-list').innerHTML = catStats.map(c => `
                <div class="space-y-1.5">
                    <div class="flex justify-between text-[10px] font-bold uppercase">
                        <span class="text-slate-500">${c.name}</span>
                        <span class="text-slate-900 font-black">R$ ${c.total.toLocaleString('pt-BR')}</span>
                    </div>
                    <div class="w-full bg-slate-100 rounded-full h-1.5 overflow-hidden">
                        <div class="bg-emerald-500 h-full transition-all duration-700" style="width: ${(c.total / (total || 1)) * 100}%"></div>
                    </div>
                </div>
            `).join('');
            lucide.createIcons();
        }

        function renderAnnual() {
            const filtered = expenses.filter(e => new Date(e.date + 'T00:00:00').getFullYear() === currentYear);
            const total = filtered.reduce((acc, curr) => acc + Number(curr.amount), 0);
            
            document.getElementById('annual-title').innerText = `Total Anual ${currentYear}`;
            document.getElementById('annual-total').innerText = `R$ ${total.toLocaleString('pt-BR', {minimumFractionDigits:2})}`;

            document.getElementById('user-yearly-stats').innerHTML = users.map(u => {
                const sum = filtered.filter(e => e.user === u.name).reduce((s, curr) => s + Number(curr.amount), 0);
                return `
                    <div class="bg-white p-6 rounded-2xl border border-slate-200 shadow-sm flex flex-col justify-center">
                        <p class="text-[10px] font-bold text-slate-400 uppercase mb-1">${u.name} no Ano</p>
                        <h4 class="text-xl font-black text-slate-800">R$ ${sum.toLocaleString('pt-BR', {minimumFractionDigits:2})}</h4>
                    </div>
                `;
            }).join('');

            // Gráfico
            const monthData = monthNames.map((m, i) => {
                const sum = filtered.filter(e => new Date(e.date + 'T00:00:00').getMonth() === i).reduce((s, curr) => s + Number(curr.amount), 0);
                return sum;
            });
            const max = Math.max(...monthData, 1);

            document.getElementById('monthly-chart').innerHTML = monthData.map((val, i) => `
                <div class="flex-1 flex flex-col items-center group relative">
                    <div class="absolute -top-10 bg-slate-800 text-white text-[9px] px-2 py-1 rounded opacity-0 group-hover:opacity-100 transition-opacity z-10 font-bold whitespace-nowrap">
                        R$ ${val.toLocaleString('pt-BR')}
                    </div>
                    <div class="w-full rounded-t-md transition-all duration-500 ${i === currentMonth ? 'bg-emerald-500' : 'bg-slate-100 group-hover:bg-slate-200'}" 
                         style="height: ${(val / max) * 100}%; min-height: ${val > 0 ? '4px' : '0'}"></div>
                    <span class="text-[8px] mt-2 font-bold uppercase ${i === currentMonth ? 'text-emerald-600' : 'text-slate-400'}">${monthNames[i].substring(0,3)}</span>
                </div>
            `).join('');
            lucide.createIcons();
        }

        // --- CONFIGURAÇÕES ---
        function toggleConfig(show) {
            document.getElementById('config-modal').style.display = show ? 'flex' : 'none';
            if(show) renderConfigLists();
        }

        function renderConfigLists() {
            // Lista Usuários
            document.getElementById('user-list').innerHTML = users.map(u => `
                <div class="border border-slate-100 rounded-2xl p-4 bg-slate-50/50">
                    <div class="flex justify-between mb-3">
                        <span class="font-black text-[11px] uppercase text-slate-700 tracking-wider">${u.name}</span>
                        <button onclick="removeUser('${u.id}')" class="text-slate-300 hover:text-red-500"><i data-lucide="trash-2" size="16"></i></button>
                    </div>
                    <div class="flex flex-wrap gap-2">
                        ${u.cards.map((c, i) => `
                            <span class="bg-white border border-slate-200 px-2 py-1 rounded text-[10px] font-bold flex items-center gap-2 shadow-sm text-slate-500">
                                ****${c} <i data-lucide="x" size="10" class="cursor-pointer text-slate-300 hover:text-red-500" onclick="removeCard('${u.id}', ${i})"></i>
                            </span>
                        `).join('')}
                        <input maxlength="4" placeholder="Adic. 4 dígitos..." class="w-24 text-[10px] border-slate-200 rounded px-2 py-1" onkeydown="if(event.key==='Enter'){ addCard('${u.id}', this.value); this.value=''; }">
                    </div>
                </div>
            `).join('');

            // Lista Categorias
            document.getElementById('category-list').innerHTML = categories.map(cat => `
                <div class="bg-white border border-slate-200 px-3 py-1.5 rounded-xl text-[10px] font-black uppercase text-slate-600 flex items-center gap-3 shadow-sm group">
                    ${cat}
                    <button onclick="removeCategory('${cat}')" class="text-slate-300 group-hover:text-red-500"><i data-lucide="x" size="12"></i></button>
                </div>
            `).join('');
            lucide.createIcons();
        }

        function addUser() {
            const name = document.getElementById('new-user-name').value.trim();
            if(name && !users.find(u => u.name === name)) {
                users.push({ id: Date.now().toString(), name, cards: [] });
                saveLocal(); renderConfigLists(); setupForm();
                document.getElementById('new-user-name').value = '';
            }
        }

        function removeUser(id) {
            if(users.length > 1) { users = users.filter(u => u.id !== id); saveLocal(); renderConfigLists(); setupForm(); }
        }

        function addCard(userId, num) {
            if(num.length === 4) {
                const u = users.find(x => x.id === userId);
                if(u) u.cards.push(num);
                saveLocal(); renderConfigLists();
            }
        }

        function removeCard(userId, idx) {
            const u = users.find(x => x.id === userId);
            if(u) u.cards.splice(idx, 1);
            saveLocal(); renderConfigLists();
        }

        function addCategory() {
            const name = document.getElementById('new-cat-name').value.trim();
            if(name && !categories.includes(name)) {
                categories.push(name);
                saveLocal(); renderConfigLists(); setupForm();
                document.getElementById('new-cat-name').value = '';
            }
        }

        function removeCategory(name) {
            if(categories.length > 1) { categories = categories.filter(c => c !== name); saveLocal(); renderConfigLists(); setupForm(); }
        }

        // --- UTILITÁRIOS ---
        function showToast(text, error = false, loading = false) {
            const toast = document.getElementById('toast');
            const icon = document.getElementById('toast-icon');
            toast.classList.remove('hidden', 'bg-slate-800', 'bg-red-600', 'animate-bounce');
            toast.classList.add('flex');
            
            if (loading) {
                toast.classList.add('bg-slate-800', 'animate-pulse');
                icon.innerHTML = `<i data-lucide="loader-2" class="animate-spin text-emerald-400"></i>`;
            } else if (error) {
                toast.classList.add('bg-red-600');
                icon.innerHTML = `<i data-lucide="x-circle"></i>`;
            } else {
                toast.classList.add('bg-slate-800', 'animate-bounce');
                icon.innerHTML = `<i data-lucide="check-circle-2" class="text-emerald-400"></i>`;
            }
            
            document.getElementById('toast-text').innerText = text;
            lucide.createIcons();
            if(!loading) setTimeout(hideToast, 3000);
        }

        function hideToast() {
            document.getElementById('toast').classList.add('hidden');
            document.getElementById('toast').classList.remove('flex');
        }

    </script>
</body>
</html>
