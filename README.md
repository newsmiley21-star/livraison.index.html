<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>CT241 - Logistique & Performance</title>
    
    <!-- PWA Meta Tags -->
    <meta name="theme-color" content="#0f172a">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="apple-mobile-web-app-title" content="CT241 Log">
    <link rel="apple-touch-icon" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">

    <!-- Manifeste Inline -->
    <link rel="manifest" id="pwa-manifest">
    
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <script src="https://cdn.tailwindcss.com"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query, orderBy } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        const firebaseConfig = {
            apiKey: "AIzaSyAdNSFmL45rSo9SxJJkvUPWeext0f7RX_Q",
            authDomain: "ct241-service-de-livraison.firebaseapp.com",
            projectId: "ct241-service-de-livraison",
            storageBucket: "ct241-service-de-livraison.firebasestorage.app",
            messagingSenderId: "297254676010",
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
        let searchQuery = "";

        // PWA Manifest
        const manifest = {
            "name": "CT241 Logistique",
            "short_name": "CT241",
            "start_url": ".",
            "display": "standalone",
            "background_color": "#0f172a",
            "theme_color": "#0f172a",
            "icons": [{ "src": "https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png", "sizes": "512x512", "type": "image/png" }]
        };
        document.getElementById('pwa-manifest').setAttribute('href', 'data:application/json;base64,' + btoa(JSON.stringify(manifest)));

        // Authentification
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            btn.disabled = true;
            btn.innerHTML = '<span class="animate-spin inline-block mr-2">⏳</span> Connexion...';
            try {
                await signInWithEmailAndPassword(auth, email, pass);
            } catch (err) {
                document.getElementById('loginError').innerText = "Identifiants non reconnus.";
                document.getElementById('loginError').classList.remove('hidden');
                btn.disabled = false;
                btn.innerText = "SE CONNECTER";
            }
        };

        window.handleLogout = async () => { await signOut(auth); location.reload(); };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                assignRole(user.email);
                document.getElementById('authSection').classList.add('hidden');
                document.getElementById('appContent').classList.remove('hidden');
                initDataSync();
            } else {
                document.getElementById('authSection').classList.remove('hidden');
                document.getElementById('appContent').classList.add('hidden');
            }
        });

        const assignRole = (email) => {
            const e = email.toLowerCase();
            if (e.includes('admin')) userRole = 'admin';
            else if (e.includes('livreur')) userRole = 'livreur';
            else userRole = 'relais';
            
            const badge = document.getElementById('userBadge');
            badge.innerText = userRole.toUpperCase();
            const styles = {
                admin: 'bg-red-500 shadow-red-200',
                livreur: 'bg-amber-500 shadow-amber-200',
                relais: 'bg-indigo-500 shadow-indigo-200'
            };
            badge.className = `${styles[userRole]} text-[10px] text-white px-4 py-1.5 rounded-full font-black tracking-[0.2em] shadow-lg`;

            document.querySelectorAll('.role-view').forEach(v => v.classList.add('hidden'));
            if(userRole === 'admin') {
                document.querySelectorAll('.role-view').forEach(v => v.classList.remove('hidden'));
            } else if(userRole === 'livreur') {
                document.getElementById('livreurSection').classList.remove('hidden');
            } else {
                document.getElementById('relaisSection').classList.remove('hidden');
            }
        };

        const initDataSync = () => {
            const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'missions'), orderBy('ca', 'desc'));
            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
            });
        };

        window.creerMission = async () => {
            const id = "CT" + Math.floor(100000 + Math.random() * 900000);
            const data = {
                id,
                s: 0,
                en: document.getElementById('en').value,
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

            if(!data.dn || !data.dt || !data.p) return alert("Veuillez remplir les champs Destinataire, Tél et Prix.");

            await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), data);
            alert("📦 Mission créée avec succès !");
            document.querySelectorAll('#relaisSection input').forEach(i => i.value = "");
        };

        window.ouvrirCamera = (id) => {
            window.targetId = id;
            document.getElementById('cameraModal').classList.remove('hidden');
        };

        window.processImage = (file) => {
            if(!file) return;
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    const MAX = 1000;
                    let w = img.width, h = img.height;
                    if (w > MAX) { h *= MAX / w; w = MAX; }
                    canvas.width = w; canvas.height = h;
                    canvas.getContext('2d').drawImage(img, 0, 0, w, h);
                    validerMission(canvas.toDataURL('image/jpeg', 0.7));
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };

        const validerMission = async (photo) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.targetId), {
                s: 2,
                img: photo,
                lk: Date.now(),
                le: currentUser.email
            });
            document.getElementById('cameraModal').classList.add('hidden');
            alert("✅ Livraison confirmée !");
        };

        window.switchTab = (tab) => { currentTab = tab; renderUI(); };
        window.handleSearch = (val) => { searchQuery = val.toLowerCase(); renderUI(); };

        const renderUI = () => {
            if(userRole === 'admin') {
                const ca = allMissions.filter(m => m.s === 2).reduce((sum, m) => sum + m.p, 0);
                document.getElementById('statCA').innerText = ca.toLocaleString() + " CFA";
                document.getElementById('statCount').innerText = allMissions.length;
            }

            const container = document.getElementById('missionContainer');
            container.innerHTML = "";

            let filtered = [...allMissions];
            if(currentTab === 'liv') filtered = filtered.filter(m => m.s === 2);
            if(currentTab === 'wait') filtered = filtered.filter(m => m.s < 2);
            if(userRole === 'relais') filtered = filtered.filter(m => m.ce === currentUser.email);
            if(searchQuery) {
                filtered = filtered.filter(m => 
                    m.id.toLowerCase().includes(searchQuery) || 
                    m.dn.toLowerCase().includes(searchQuery) ||
                    m.en.toLowerCase().includes(searchQuery)
                );
            }

            filtered.forEach(m => {
                const card = document.createElement('div');
                card.className = "bg-white p-6 rounded-[2.5rem] shadow-sm border border-slate-100 mb-4 active:scale-[0.98] transition-all duration-300";
                card.onclick = () => showDetail(m.id);
                card.innerHTML = `
                    <div class="flex justify-between items-start mb-4">
                        <span class="text-[9px] font-black text-slate-300 tracking-tighter uppercase">#${m.id}</span>
                        <div class="flex items-center gap-2">
                            <span class="${m.s === 2 ? 'bg-emerald-100 text-emerald-600' : 'bg-amber-100 text-amber-600'} text-[8px] px-3 py-1 rounded-full font-black uppercase tracking-widest">
                                ${m.s === 2 ? 'Livré' : 'En Cours'}
                            </span>
                        </div>
                    </div>
                    <div class="space-y-1">
                        <h4 class="text-sm font-black text-slate-800 uppercase leading-none">${m.dn}</h4>
                        <p class="text-[10px] text-slate-400 font-bold uppercase tracking-tight">${m.dq} • ${m.dt}</p>
                    </div>
                    <div class="mt-5 pt-4 border-t border-slate-50 flex justify-between items-center">
                        <div class="flex flex-col">
                            <span class="text-[8px] font-black text-slate-400 uppercase tracking-widest">Prix</span>
                            <span class="text-xs font-black text-slate-900">${m.p.toLocaleString()} FCFA</span>
                        </div>
                        ${m.s < 2 && (userRole === 'admin' || userRole === 'livreur') ? `
                            <button onclick="event.stopPropagation(); ouvrirCamera('${m.id}')" class="bg-slate-900 text-white px-5 py-2.5 rounded-2xl text-[9px] font-black uppercase tracking-widest shadow-lg active:bg-indigo-600 transition-colors">Valider</button>
                        ` : ''}
                    </div>
                `;
                container.appendChild(card);
            });

            document.querySelectorAll('.tab-btn').forEach(btn => {
                const bt = btn.getAttribute('data-tab');
                btn.className = bt === currentTab ? 
                    "tab-btn bg-slate-900 text-white px-6 py-3 rounded-2xl text-[10px] font-black uppercase tracking-widest shadow-xl transition-all" : 
                    "tab-btn bg-white text-slate-400 border border-slate-100 px-6 py-3 rounded-2xl text-[10px] font-black uppercase tracking-widest hover:bg-slate-50 transition-all";
            });
        };

        window.showDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            const modal = document.getElementById('detailModal');
            const content = document.getElementById('detailContent');
            content.innerHTML = `
                <div class="p-8 space-y-8 animate-in slide-in-from-bottom-10 duration-500">
                    <div class="flex justify-between items-center border-b border-slate-100 pb-6">
                        <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-14">
                        <div class="text-right">
                            <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Bordereau</p>
                            <p class="text-2xl font-black text-slate-900 leading-none">#${m.id}</p>
                        </div>
                    </div>

                    <div class="grid grid-cols-2 gap-10">
                        <div class="space-y-3">
                            <p class="text-[9px] font-black text-slate-400 uppercase tracking-[0.2em]">Expéditeur</p>
                            <div>
                                <p class="text-sm font-black text-slate-800 uppercase">${m.en || 'Non spécifié'}</p>
                                <p class="text-[11px] text-slate-500 font-bold mt-1 uppercase">${m.eq || ''}</p>
                                <p class="text-[11px] text-slate-500 font-bold">${m.et || ''}</p>
                            </div>
                        </div>
                        <div class="space-y-3 text-right">
                            <p class="text-[9px] font-black text-slate-400 uppercase tracking-[0.2em]">Destinataire</p>
                            <div>
                                <p class="text-sm font-black text-slate-800 uppercase">${m.dn}</p>
                                <p class="text-[11px] text-slate-500 font-bold mt-1 uppercase">${m.dq}</p>
                                <p class="text-[11px] text-slate-500 font-bold">${m.dt}</p>
                            </div>
                        </div>
                    </div>

                    <div class="bg-slate-50 p-6 rounded-[2.5rem] border border-slate-100 space-y-5">
                        <div class="flex justify-between items-center">
                            <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Net à Payer</span>
                            <span class="text-xl font-black text-slate-900">${m.p.toLocaleString()} CFA</span>
                        </div>
                        <div class="flex justify-between items-center pt-5 border-t border-slate-200">
                            <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Mode de Règlement</span>
                            <span class="text-[11px] font-black uppercase text-indigo-600">${['CASH', 'AIRTEL MONEY', 'MOOV MONEY'][m.r || 0]}</span>
                        </div>
                    </div>

                    ${m.img ? `
                        <div class="space-y-4">
                            <p class="text-[9px] font-black text-slate-400 uppercase tracking-widest">Preuve de Livraison</p>
                            <img src="${m.img}" class="w-full rounded-[2.5rem] shadow-2xl border-4 border-white">
                        </div>
                    ` : `
                        <div class="py-10 text-center border-2 border-dashed border-slate-100 rounded-[2.5rem]">
                            <p class="text-[10px] font-black text-slate-300 uppercase tracking-widest">En attente de preuve</p>
                        </div>
                    `}
                </div>
            `;
            modal.classList.remove('hidden');
        };

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background: #fdfdfd; -webkit-tap-highlight-color: transparent; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        input::placeholder { color: #cbd5e1; }
        @media print { .no-print { display: none !important; } .print-only { display: block !important; } }
    </style>
</head>
<body class="max-w-md mx-auto min-h-screen relative bg-white shadow-2xl overflow-x-hidden">

    <!-- LOGIN -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 z-[999] flex items-center justify-center p-8">
        <div class="w-full bg-white p-12 rounded-[4rem] text-center space-y-12 animate-in fade-in zoom-in duration-700">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-28 mx-auto drop-shadow-2xl">
            <div>
                <h1 class="text-4xl font-black text-slate-900 tracking-tighter">CT241 LOG</h1>
                <p class="text-slate-400 text-[10px] font-black uppercase tracking-[0.4em] mt-2">Logistique & Performance</p>
            </div>
            <form onsubmit="handleLogin(event)" class="space-y-5 text-left">
                <div class="space-y-1.5">
                    <label class="text-[10px] font-black text-slate-400 uppercase ml-5">Utilisateur</label>
                    <input type="email" id="loginEmail" placeholder="nom@ct241.com" class="w-full p-6 bg-slate-50 rounded-3xl text-[13px] font-bold outline-none focus:ring-4 ring-slate-100 transition-all" required>
                </div>
                <div class="space-y-1.5">
                    <label class="text-[10px] font-black text-slate-400 uppercase ml-5">Mot de passe</label>
                    <input type="password" id="loginPass" placeholder="••••••••" class="w-full p-6 bg-slate-50 rounded-3xl text-[13px] font-bold outline-none focus:ring-4 ring-slate-100 transition-all" required>
                </div>
                <p id="loginError" class="text-red-500 text-[10px] font-black hidden text-center uppercase tracking-widest"></p>
                <button id="loginBtn" class="w-full bg-slate-900 text-white py-6 rounded-3xl font-black text-[11px] uppercase shadow-2xl tracking-[0.3em] active:scale-95 transition-all mt-4">SE CONNECTER</button>
            </form>
        </div>
    </section>

    <!-- APP CONTENT -->
    <main id="appContent" class="hidden min-h-screen pb-40">
        <!-- HEADER -->
        <header class="p-8 sticky top-0 bg-white/90 backdrop-blur-2xl z-50 border-b border-slate-50 flex justify-between items-center no-print">
            <div class="flex items-center gap-5">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-12">
                <div id="userBadge" class="animate-pulse">...</div>
            </div>
            <button onclick="handleLogout()" class="h-14 w-14 bg-slate-50 rounded-[1.5rem] flex items-center justify-center text-2xl hover:bg-red-50 transition-colors">🚪</button>
        </header>

        <div class="p-8 space-y-12">
            
            <!-- ADMIN STATS -->
            <section class="role-view hidden animate-in slide-in-from-top-4 duration-500">
                <div class="grid grid-cols-2 gap-5">
                    <div class="bg-indigo-600 p-8 rounded-[3rem] text-white shadow-2xl shadow-indigo-100 relative overflow-hidden group">
                        <p class="text-[10px] font-black text-indigo-200 uppercase tracking-widest mb-2">CA Total</p>
                        <h2 id="statCA" class="text-2xl font-black">0 CFA</h2>
                        <div class="absolute -right-6 -bottom-6 opacity-10 text-8xl group-hover:scale-110 transition-transform">💎</div>
                    </div>
                    <div class="bg-white border border-slate-100 p-8 rounded-[3rem] shadow-sm">
                        <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2">Envois</p>
                        <h2 id="statCount" class="text-2xl font-black text-slate-900">0</h2>
                    </div>
                </div>
            </section>

            <!-- RELAIS FORM -->
            <section id="relaisSection" class="role-view hidden animate-in slide-in-from-bottom-8 duration-500">
                <div class="bg-slate-900 p-10 rounded-[3.5rem] shadow-2xl text-white space-y-10">
                    <h3 class="text-xs font-black uppercase tracking-[0.3em] flex items-center gap-4">
                        <span class="w-10 h-10 bg-white/10 rounded-2xl flex items-center justify-center text-xl">🚀</span>
                        Enregistrement
                    </h3>
                    
                    <div class="space-y-6">
                        <div class="grid grid-cols-1 gap-4">
                            <input type="text" id="en" placeholder="Nom de l'Expéditeur" class="w-full p-5 bg-white/5 rounded-2xl text-[13px] font-bold outline-none border border-white/10 focus:bg-white/10 transition-all">
                            <div class="grid grid-cols-2 gap-4">
                                <input type="text" id="eq" placeholder="Quartier Exp." class="p-5 bg-white/5 rounded-2xl text-[13px] font-bold outline-none border border-white/10">
                                <input type="tel" id="et" placeholder="Tél. Exp." class="p-5 bg-white/5 rounded-2xl text-[13px] font-bold outline-none border border-white/10">
                            </div>
                        </div>

                        <div class="pt-6 border-t border-white/5 grid grid-cols-1 gap-4">
                            <input type="text" id="dn" placeholder="Nom du Destinataire" class="w-full p-5 bg-white/5 rounded-2xl text-[13px] font-bold outline-none border border-white/10">
                            <div class="grid grid-cols-2 gap-4">
                                <input type="text" id="dq" placeholder="Quartier Dest." class="p-5 bg-white/5 rounded-2xl text-[13px] font-bold outline-none border border-white/10">
                                <input type="tel" id="dt" placeholder="Tél. Destinataire" class="p-5 bg-white/5 rounded-2xl text-[13px] font-bold outline-none border border-white/10">
                            </div>
                        </div>

                        <div class="grid grid-cols-2 gap-4">
                            <select id="r" class="p-5 bg-white/5 rounded-2xl text-[10px] font-black uppercase outline-none border border-white/10">
                                <option value="0" class="text-slate-900">Paiement Cash</option>
                                <option value="1" class="text-slate-900">Airtel Money</option>
                                <option value="2" class="text-slate-900">Moov Money</option>
                            </select>
                            <input type="number" id="p" placeholder="Montant (CFA)" class="p-5 bg-emerald-500 text-white rounded-2xl text-[13px] font-black outline-none placeholder:text-emerald-200">
                        </div>
                    </div>

                    <button onclick="creerMission()" class="w-full bg-white text-slate-900 py-6 rounded-3xl font-black text-[11px] uppercase tracking-[0.3em] shadow-xl active:scale-95 transition-all">VALIDER LA MISSION</button>
                </div>
            </section>

            <!-- LISTE -->
            <section class="space-y-8 no-print">
                <div class="space-y-6">
                    <input type="text" oninput="handleSearch(this.value)" placeholder="Rechercher par ID ou Nom..." class="w-full p-6 bg-slate-50 border border-slate-100 rounded-3xl text-[13px] font-bold outline-none focus:ring-4 ring-slate-100 transition-all">
                    
                    <div class="flex gap-4 overflow-x-auto no-scrollbar pb-2">
                        <button data-tab="all" onclick="switchTab('all')" class="tab-btn">Toutes</button>
                        <button data-tab="wait" onclick="switchTab('wait')" class="tab-btn">En Cours</button>
                        <button data-tab="liv" onclick="switchTab('liv')" class="tab-btn">Livrées</button>
                    </div>
                </div>

                <div id="missionContainer"></div>
            </section>
        </div>
    </main>

    <!-- MODAL DETAIL -->
    <div id="detailModal" class="fixed inset-0 bg-slate-900/60 backdrop-blur-xl z-[500] hidden flex items-center justify-center p-6">
        <div class="w-full max-w-lg bg-white rounded-[4rem] shadow-2xl overflow-y-auto max-h-[92vh]">
            <div id="detailContent"></div>
            <div class="p-8 bg-slate-50 flex gap-4 no-print border-t border-slate-100">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-6 rounded-3xl text-[11px] tracking-[0.2em] uppercase shadow-2xl active:scale-95 transition-all">🖨️ IMPRIMER</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-8 bg-white border border-slate-200 text-slate-400 font-black py-6 rounded-3xl text-[11px] uppercase">Fermer</button>
            </div>
        </div>
    </div>

    <!-- MODAL CAMERA -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[1000] hidden flex flex-col items-center justify-center p-12 text-white">
        <div class="text-center space-y-12 animate-in zoom-in duration-500">
            <div class="w-32 h-32 bg-white/10 rounded-full flex items-center justify-center text-7xl mx-auto border border-white/5 drop-shadow-2xl">📸</div>
            <div class="space-y-3">
                <h2 class="text-3xl font-black uppercase tracking-tight">Livraison</h2>
                <p class="text-slate-500 text-[11px] font-bold uppercase tracking-widest leading-relaxed">Capturez la preuve de remise<br>(Colis ou Bordereau signé)</p>
            </div>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-slate-900 font-black py-6 rounded-[2.5rem] shadow-2xl uppercase tracking-[0.3em] active:scale-95 transition-all px-12">DÉMARRER LA CAMÉRA</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-600 text-[11px] font-black uppercase tracking-widest pt-4">Annuler la procédure</button>
        </div>
    </div>

</body>
</html>
