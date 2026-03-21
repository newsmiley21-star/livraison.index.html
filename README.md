<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>CT241 - Logistique & Performance</title>
    
    <meta name="theme-color" content="#0f172a">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap" rel="stylesheet">
    
    <style>
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; color: #0f172a; }
        .chart-container { position: relative; width: 100%; height: 250px; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
    </style>
</head>
<body class="antialiased">

    <!-- LOGIN SECTION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 z-[100] flex items-center justify-center p-6">
        <div class="w-full max-w-sm bg-white p-8 rounded-[2.5rem] shadow-2xl animate-in fade-in zoom-in duration-300">
            <div class="text-center mb-8">
                <div class="inline-block p-4 bg-slate-50 rounded-3xl mb-4">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" alt="Logo" class="w-16 h-16 object-contain">
                </div>
                <h1 class="text-2xl font-black text-slate-900 tracking-tight">CT241 LOGISTIQUE</h1>
                <p class="text-slate-400 text-xs font-bold uppercase tracking-widest mt-2">Accès Sécurisé</p>
            </div>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" placeholder="Email professionnel" class="w-full p-4 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-indigo-500 outline-none transition-all">
                <input type="password" id="loginPass" placeholder="Mot de passe" class="w-full p-4 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-indigo-500 outline-none transition-all">
                <button id="loginBtn" class="w-full bg-slate-900 text-white py-4 rounded-2xl font-black text-xs tracking-widest hover:bg-slate-800 transition-colors">SE CONNECTER</button>
            </form>
        </div>
    </section>

    <!-- MAIN APP SECTION -->
    <main id="appContent" class="hidden min-h-screen pb-24">
        
        <!-- Navigation Haute -->
        <nav class="sticky top-0 bg-white/80 backdrop-blur-md border-b border-slate-100 z-40">
            <div class="max-w-md mx-auto px-6 py-4 flex justify-between items-center">
                <div class="flex items-center gap-2">
                    <span id="userBadge" class="text-[10px] font-black bg-indigo-600 text-white px-3 py-1 rounded-full shadow-lg shadow-indigo-200 uppercase">...</span>
                </div>
                <div class="flex items-center gap-4">
                    <button onclick="toggleDoc()" class="text-xs font-black text-slate-400 hover:text-indigo-600 transition-colors">DOCS</button>
                    <button onclick="handleLogout()" class="w-8 h-8 flex items-center justify-center bg-slate-100 rounded-full text-sm">🚪</button>
                </div>
            </div>
        </nav>

        <div class="max-w-md mx-auto px-6 pt-6 space-y-8">
            
            <!-- VUE ADMIN : STATS & EXPORT -->
            <section id="adminView" class="role-view hidden space-y-4">
                <div class="bg-indigo-600 p-6 rounded-[2.5rem] text-white shadow-xl shadow-indigo-100">
                    <p class="text-[10px] font-bold opacity-70 uppercase tracking-widest mb-1">C.A du Jour (Livrées)</p>
                    <div id="caDisplay" class="text-3xl font-black">0 CFA</div>
                    <div class="mt-4 flex gap-2">
                        <div class="flex-1 bg-white/10 p-3 rounded-2xl">
                            <p class="text-[8px] font-bold uppercase opacity-60">Missions</p>
                            <p id="countDisplay" class="text-lg font-black">0</p>
                        </div>
                        <div class="flex-1 bg-white/10 p-3 rounded-2xl">
                            <p class="text-[8px] font-bold uppercase opacity-60">Réussite</p>
                            <p id="ratioDisplay" class="text-lg font-black">0%</p>
                        </div>
                    </div>
                </div>
                <button onclick="exportData()" class="w-full bg-white border-2 border-slate-100 text-slate-900 p-4 rounded-2xl font-black text-[10px] uppercase tracking-widest flex items-center justify-center gap-2">
                    📊 Télécharger Rapport Excel (CSV)
                </button>
            </section>

            <!-- VUE RELAIS : FORMULAIRE CRÉATION -->
            <section id="relaisView" class="role-view hidden">
                <div class="bg-slate-900 p-6 rounded-[2.5rem] text-white space-y-6">
                    <div class="flex justify-between items-center">
                        <h2 class="text-xs font-black uppercase tracking-widest">Nouveau Colis</h2>
                        <span class="text-xl">📦</span>
                    </div>
                    <div class="space-y-3">
                        <div class="space-y-1">
                            <label class="text-[9px] font-bold text-slate-400 uppercase ml-2">Expéditeur</label>
                            <input type="text" id="en" placeholder="Nom Boutique / Client" class="w-full p-4 bg-white/5 rounded-2xl text-xs font-bold border border-white/10 outline-none focus:border-indigo-500 transition-all">
                        </div>
                        <div class="grid grid-cols-2 gap-2">
                            <input type="text" id="eq" placeholder="Quartier Exp." class="w-full p-4 bg-white/5 rounded-2xl text-xs font-bold border border-white/10 outline-none">
                            <input type="tel" id="et" placeholder="Téléphone Exp." class="w-full p-4 bg-white/5 rounded-2xl text-xs font-bold border border-white/10 outline-none">
                        </div>
                        <div class="h-px bg-white/10 my-2"></div>
                        <div class="space-y-1">
                            <label class="text-[9px] font-bold text-indigo-400 uppercase ml-2">Destinataire</label>
                            <input type="text" id="dn" placeholder="Nom du Client" class="w-full p-4 bg-white/10 rounded-2xl text-xs font-bold border border-white/20 outline-none focus:border-indigo-500 transition-all">
                        </div>
                        <div class="grid grid-cols-2 gap-2">
                            <input type="text" id="dq" placeholder="Quartier Dest." class="w-full p-4 bg-white/5 rounded-2xl text-xs font-bold border border-white/10 outline-none">
                            <input type="tel" id="dt" placeholder="Téléphone Dest." class="w-full p-4 bg-white/5 rounded-2xl text-xs font-bold border border-white/10 outline-none">
                        </div>
                        <div class="grid grid-cols-2 gap-2">
                            <select id="r" class="p-4 bg-white/5 rounded-2xl text-xs font-bold border border-white/10 outline-none">
                                <option value="0">Cash</option>
                                <option value="1">Airtel Money</option>
                                <option value="2">Moov Money</option>
                            </select>
                            <input type="number" id="p" placeholder="Prix CFA" class="p-4 bg-emerald-500 text-white rounded-2xl text-xs font-black placeholder:text-emerald-100 outline-none shadow-lg shadow-emerald-500/20">
                        </div>
                    </div>
                    <button onclick="createMission()" class="w-full bg-white text-slate-900 py-4 rounded-2xl font-black text-xs uppercase tracking-widest shadow-xl">CRÉER LA MISSION</button>
                </div>
            </section>

            <!-- LISTE DES MISSIONS (COMMUN) -->
            <section class="space-y-6">
                <div class="flex items-center justify-between">
                    <h2 class="text-sm font-black text-slate-900 uppercase tracking-widest">Flux des Missions</h2>
                    <div class="flex bg-slate-100 p-1 rounded-xl">
                        <button onclick="setTab('all')" id="btn-all" class="px-4 py-1.5 text-[9px] font-black rounded-lg transition-all bg-white shadow-sm">TOUT</button>
                        <button onclick="setTab('wait')" id="btn-wait" class="px-4 py-1.5 text-[9px] font-black rounded-lg transition-all">ATTENTE</button>
                    </div>
                </div>
                
                <div id="list" class="space-y-3">
                    <!-- Les missions s'injectent ici -->
                </div>
            </section>
        </div>

        <!-- MODAL DOCUMENTATION TECHNIQUE -->
        <div id="docModal" class="fixed inset-0 bg-slate-900/90 z-[100] hidden backdrop-blur-sm overflow-y-auto p-6 no-scrollbar">
            <div class="max-w-md mx-auto bg-white rounded-[2.5rem] p-8 space-y-8 animate-in slide-in-from-bottom duration-500">
                <div class="flex justify-between items-center border-b border-slate-100 pb-4">
                    <h3 class="text-xl font-black">Documentation</h3>
                    <button onclick="toggleDoc()" class="text-2xl">✕</button>
                </div>
                
                <div class="space-y-6">
                    <div>
                        <h4 class="text-[10px] font-black text-indigo-600 uppercase mb-2">Structure des Données</h4>
                        <div class="bg-slate-50 p-4 rounded-2xl font-mono text-[10px] space-y-1">
                            <p><span class="text-slate-400">s:</span> Statut (0:Attente, 2:Livré)</p>
                            <p><span class="text-slate-400">r:</span> Règlement (0:Cash, 1:Airtel, 2:Moov)</p>
                            <p><span class="text-slate-400">ca:</span> Timestamp (via Date.now())</p>
                            <p><span class="text-slate-400">img:</span> Photo Base64 (si s=2)</p>
                        </div>
                    </div>
                    
                    <div>
                        <h4 class="text-[10px] font-black text-indigo-600 uppercase mb-2">Règles de Sécurité</h4>
                        <p class="text-[11px] text-slate-600 leading-relaxed italic">
                            Les suppressions sont interdites par Firestore. Les lectures sont filtrées par l'application sur les dernières 24h pour optimiser les performances.
                        </p>
                    </div>

                    <div class="chart-container">
                        <canvas id="docChart"></canvas>
                    </div>
                </div>
                
                <button onclick="toggleDoc()" class="w-full bg-slate-900 text-white py-4 rounded-2xl font-black text-[10px]">RETOUR</button>
            </div>
        </div>
    </main>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query, orderBy, where } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        const firebaseConfig = {
            apiKey: "AIzaSyAdNSFmL45rSo9SxJJkvUPWeext0f7RX_Q",
            authDomain: "ct241-service-de-livraison.firebaseapp.com",
            projectId: "ct241-service-de-livraison",
            appId: "1:297254676010:web:01e3765686c8d478618553"
        };

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = 'ct241-service-de-livraison';

        let currentUser = null;
        let userRole = null;
        let allMissions = [];
        let currentTab = 'all';

        // GESTION CONNEXION
        window.handleLogin = async (e) => {
            e.preventDefault();
            const btn = document.getElementById('loginBtn');
            btn.disabled = true;
            btn.innerText = "VERIFICATION...";
            try {
                await signInWithEmailAndPassword(auth, loginEmail.value, loginPass.value);
            } catch (err) {
                alert("Identifiants incorrects.");
                btn.disabled = false;
                btn.innerText = "SE CONNECTER";
            }
        };

        window.handleLogout = () => signOut(auth).then(() => location.reload());

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                assignRole(user.email);
                document.getElementById('authSection').classList.add('hidden');
                document.getElementById('appContent').classList.remove('hidden');
                initData();
            } else {
                document.getElementById('authSection').classList.remove('hidden');
                document.getElementById('appContent').classList.add('hidden');
            }
        });

        const assignRole = (email) => {
            const e = email.toLowerCase();
            userRole = e.includes('admin') ? 'admin' : (e.includes('livreur') ? 'livreur' : 'relais');
            document.getElementById('userBadge').innerText = userRole;
            
            document.querySelectorAll('.role-view').forEach(v => v.classList.add('hidden'));
            if(userRole === 'admin') {
                document.getElementById('adminView').classList.remove('hidden');
                document.getElementById('relaisView').classList.remove('hidden');
            } else if(userRole === 'livreur') {
                // Pas de section spécifique
            } else {
                document.getElementById('relaisView').classList.remove('hidden');
            }
        };

        // FLUX DE DONNÉES (FILTRE 24H)
        const initData = () => {
            const hier = Date.now() - (24 * 60 * 60 * 1000);
            const q = query(
                collection(db, 'artifacts', appId, 'public', 'data', 'missions'),
                where('ca', '>', hier),
                orderBy('ca', 'desc')
            );
            
            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                render();
            }, (error) => console.error("Erreur flux:", error));
        };

        // EXPORT EXCEL
        window.exportData = () => {
            let csv = "\uFEFFID;Date;Expediteur;Destinataire;Quartier;Prix;Reglement;Statut\n";
            allMissions.forEach(m => {
                const date = new Date(m.ca).toLocaleString('fr-FR');
                const reg = m.r === 0 ? "Cash" : (m.r === 1 ? "Airtel" : "Moov");
                const st = m.s === 2 ? "LIVRE" : "ATTENTE";
                csv += `${m.id};${date};${m.en};${m.dn};${m.dq};${m.p};${reg};${st}\n`;
            });
            const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
            const link = document.createElement("a");
            link.href = URL.createObjectURL(blob);
            link.download = `CT241_Rapport_${new Date().toISOString().split('T')[0]}.csv`;
            link.click();
        };

        // OPÉRATIONS MISSION
        window.createMission = async () => {
            const id = "CT" + Math.floor(100000 + Math.random() * 900000);
            const data = {
                id, s: 0,
                en: document.getElementById('en').value || "Client Comptant",
                eq: document.getElementById('eq').value,
                et: document.getElementById('et').value,
                dn: document.getElementById('dn').value,
                dq: document.getElementById('dq').value,
                dt: document.getElementById('dt').value,
                p: parseInt(document.getElementById('p').value) || 0,
                r: parseInt(document.getElementById('r').value) || 0,
                ce: currentUser.email,
                ca: Date.now()
            };
            if(!data.dn || !data.p) return alert("Nom et Prix obligatoires.");
            await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), data);
            document.querySelectorAll('#relaisView input').forEach(i => i.value = "");
        };

        window.validerLivraison = (id) => {
            const input = document.createElement('input');
            input.type = 'file';
            input.accept = 'image/*';
            input.capture = 'camera';
            input.onchange = (e) => {
                const file = e.target.files[0];
                const reader = new FileReader();
                reader.onload = async (re) => {
                    await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), {
                        s: 2, img: re.target.result, le: currentUser.email
                    });
                };
                reader.readAsDataURL(file);
            };
            input.click();
        };

        window.toggleDoc = () => {
            const modal = document.getElementById('docModal');
            modal.classList.toggle('hidden');
            if(!modal.classList.contains('hidden')) initDocChart();
        };

        window.setTab = (t) => {
            currentTab = t;
            document.getElementById('btn-all').classList.toggle('bg-white', t === 'all');
            document.getElementById('btn-all').classList.toggle('shadow-sm', t === 'all');
            document.getElementById('btn-wait').classList.toggle('bg-white', t === 'wait');
            document.getElementById('btn-wait').classList.toggle('shadow-sm', t === 'wait');
            render();
        };

        const render = () => {
            const container = document.getElementById('list');
            container.innerHTML = "";
            
            if(userRole === 'admin') {
                const livrees = allMissions.filter(m => m.s === 2);
                const totalCa = livrees.reduce((a, b) => a + b.p, 0);
                document.getElementById('caDisplay').innerText = totalCa.toLocaleString() + " CFA";
                document.getElementById('countDisplay').innerText = allMissions.length;
                const ratio = allMissions.length ? Math.round((livrees.length / allMissions.length) * 100) : 0;
                document.getElementById('ratioDisplay').innerText = ratio + "%";
            }

            let filtered = allMissions;
            if(currentTab === 'wait') filtered = allMissions.filter(m => m.s === 0);

            filtered.forEach(m => {
                const card = document.createElement('div');
                card.className = "bg-white p-5 rounded-3xl shadow-sm border border-slate-100 animate-in slide-in-from-right duration-300";
                card.innerHTML = `
                    <div class="flex justify-between items-start mb-3">
                        <div class="flex flex-col">
                            <span class="text-[9px] font-black text-slate-300">#${m.id}</span>
                            <span class="text-xs font-black text-slate-900 uppercase">${m.dn}</span>
                        </div>
                        <span class="text-[8px] font-black px-3 py-1 rounded-full ${m.s === 2 ? 'bg-emerald-100 text-emerald-600' : 'bg-amber-100 text-amber-600'}">
                            ${m.s === 2 ? 'LIVRÉ' : 'EN ATTENTE'}
                        </span>
                    </div>
                    <div class="flex items-center gap-2 mb-4">
                        <div class="w-2 h-2 rounded-full bg-slate-200"></div>
                        <span class="text-[10px] text-slate-400 font-bold">${m.dq} • ${m.dt}</span>
                    </div>
                    <div class="flex justify-between items-center pt-3 border-t border-slate-50">
                        <div class="flex flex-col">
                            <span class="text-[8px] font-bold text-slate-400 uppercase">A Encaisser</span>
                            <span class="text-sm font-black text-indigo-600">${m.p.toLocaleString()} CFA</span>
                        </div>
                        ${m.s === 0 && (userRole === 'admin' || userRole === 'livreur') ? `
                            <button onclick="validerLivraison('${m.id}')" class="bg-slate-900 text-white text-[9px] font-black px-5 py-3 rounded-2xl shadow-lg shadow-slate-200 uppercase tracking-widest">Valider</button>
                        ` : ''}
                        ${m.s === 2 ? `
                            <div class="w-10 h-10 rounded-xl bg-slate-50 overflow-hidden border border-slate-100">
                                <img src="${m.img || ''}" class="w-full h-full object-cover grayscale opacity-50">
                            </div>
                        ` : ''}
                    </div>
                `;
                container.appendChild(card);
            });
        };

        const initDocChart = () => {
            const ctx = document.getElementById('docChart').getContext('2d');
            new Chart(ctx, {
                type: 'line',
                data: {
                    labels: ['08h', '10h', '12h', '14h', '16h', '18h'],
                    datasets: [{
                        label: 'Activité',
                        data: [12, 19, 3, 5, 2, 3],
                        borderColor: '#4f46e5',
                        tension: 0.4,
                        fill: false
                    }]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                    plugins: { legend: { display: false } },
                    scales: { y: { display: false }, x: { grid: { display: false } } }
                }
            });
        };
    </script>
</body>
</html>
