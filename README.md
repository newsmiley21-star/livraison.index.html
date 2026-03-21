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

    <!-- Manifeste Inline pour l'installabilité -->
    <link rel="manifest" id="pwa-manifest">
    
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <script src="https://cdn.tailwindcss.com"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query, where, orderBy } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        // CONFIGURATION FIREBASE
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

        // ÉTATS GLOBAUX
        let currentUser = null;
        let userRole = null;
        let allMissions = [];
        let currentTab = 'all';

        // --- PWA MANIFEST GENERATION ---
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

        // --- AUTHENTIFICATION ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            btn.disabled = true;
            btn.innerHTML = '<span class="animate-spin inline-block mr-2">⏳</span> Authentification...';
            
            try {
                await signInWithEmailAndPassword(auth, email, pass);
            } catch (err) {
                document.getElementById('loginError').innerText = "Identifiants incorrects.";
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
            
            document.getElementById('userBadge').innerText = userRole.toUpperCase();
            const badgeClasses = {
                admin: 'bg-red-500 shadow-red-200',
                livreur: 'bg-amber-500 shadow-amber-200',
                relais: 'bg-indigo-500 shadow-indigo-200'
            };
            document.getElementById('userBadge').className = `${badgeClasses[userRole]} text-[10px] text-white px-4 py-1.5 rounded-full font-black tracking-widest shadow-lg`;

            // Affichage conditionnel des sections
            document.querySelectorAll('.role-view').forEach(v => v.classList.add('hidden'));
            if(userRole === 'admin') document.querySelectorAll('.role-view').forEach(v => v.classList.remove('hidden'));
            else if(userRole === 'livreur') document.getElementById('livreurSection').classList.remove('hidden');
            else document.getElementById('relaisSection').classList.remove('hidden');
        };

        // --- SYNCHRONISATION (Optimisation 48h) ---
        const initDataSync = () => {
            const limitTime = Date.now() - (48 * 60 * 60 * 1000); // 48h glissantes
            const q = query(
                collection(db, 'artifacts', appId, 'public', 'data', 'missions'),
                where('ca', '>', limitTime)
            );

            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
            });
        };

        // --- ACTIONS MÉTIERS ---
        window.creerMission = async () => {
            const id = "CT" + Math.floor(100000 + Math.random() * 900000);
            const data = {
                id,
                s: 0, // Statut : 0=Attente, 1=Récupéré, 2=Livré
                en: document.getElementById('en').value,
                eq: document.getElementById('eq').value,
                et: document.getElementById('et').value,
                dn: document.getElementById('dn').value,
                dq: document.getElementById('dq').value,
                dt: document.getElementById('dt').value,
                p: parseInt(document.getElementById('p').value) || 0,
                r: parseInt(document.getElementById('r').value) || 0, // 0: Cash, 1: Airtel, 2: Moov
                ce: currentUser.email,
                ca: Date.now()
            };

            if(!data.dn || !data.dt || !data.p) return alert("Les champs Destinataire, Tél et Prix sont obligatoires.");

            await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), data);
            alert("✅ Envoi enregistré avec succès !");
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
                    const MAX = 800;
                    let w = img.width;
                    let h = img.height;
                    if (w > MAX) { h *= MAX / w; w = MAX; }
                    canvas.width = w; canvas.height = h;
                    const ctx = canvas.getContext('2d');
                    ctx.drawImage(img, 0, 0, w, h);
                    const base64 = canvas.toDataURL('image/jpeg', 0.6);
                    validerMission(base64);
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
            alert("🚀 Livraison confirmée !");
        };

        // --- RENDU UI ---
        window.switchTab = (tab) => {
            currentTab = tab;
            renderUI();
        };

        const renderUI = () => {
            // Stats Admin
            if(userRole === 'admin') {
                const totalCA = allMissions.filter(m => m.s === 2).reduce((sum, m) => sum + m.p, 0);
                const totalColis = allMissions.length;
                document.getElementById('statCA').innerText = totalCA.toLocaleString() + " CFA";
                document.getElementById('statCount').innerText = totalColis;
            }

            const container = document.getElementById('missionContainer');
            container.innerHTML = "";

            let filtered = allMissions.sort((a,b) => b.ca - a.ca);
            if(currentTab === 'liv') filtered = filtered.filter(m => m.s === 2);
            if(currentTab === 'wait') filtered = filtered.filter(m => m.s < 2);

            // Filtrage par rôle (Relais ne voit que les siens)
            if(userRole === 'relais') filtered = filtered.filter(m => m.ce === currentUser.email);

            filtered.forEach(m => {
                const div = document.createElement('div');
                div.className = "bg-white p-6 rounded-[2rem] shadow-sm border border-slate-100 mb-4 active:scale-[0.98] transition-all cursor-pointer";
                div.onclick = () => showDetail(m.id);
                
                div.innerHTML = `
                    <div class="flex justify-between items-start mb-4">
                        <span class="text-[9px] font-black text-slate-300 tracking-tighter uppercase">#${m.id}</span>
                        <span class="${m.s === 2 ? 'bg-emerald-100 text-emerald-600' : 'bg-amber-100 text-amber-600'} text-[8px] px-3 py-1 rounded-full font-black uppercase tracking-widest">
                            ${m.s === 2 ? 'Livré' : 'En Attente'}
                        </span>
                    </div>
                    <div class="space-y-1">
                        <h4 class="text-sm font-black text-slate-800 uppercase">${m.dn}</h4>
                        <p class="text-[10px] text-slate-400 font-bold uppercase tracking-wide">${m.dq}</p>
                    </div>
                    <div class="mt-4 pt-4 border-t border-slate-50 flex justify-between items-center">
                        <span class="text-xs font-black text-slate-900">${m.p.toLocaleString()} FCFA</span>
                        ${m.s < 2 && (userRole === 'admin' || userRole === 'livreur') ? `
                            <button onclick="event.stopPropagation(); ouvrirCamera('${m.id}')" class="bg-slate-900 text-white px-4 py-2 rounded-xl text-[9px] font-black uppercase tracking-widest">Valider</button>
                        ` : ''}
                    </div>
                `;
                container.appendChild(div);
            });

            // Update tab styles
            document.querySelectorAll('.tab-btn').forEach(btn => {
                const btnTab = btn.getAttribute('data-tab');
                if(btnTab === currentTab) {
                    btn.className = "tab-btn bg-slate-900 text-white px-6 py-2.5 rounded-2xl text-[10px] font-black uppercase tracking-widest";
                } else {
                    btn.className = "tab-btn bg-white text-slate-400 border border-slate-100 px-6 py-2.5 rounded-2xl text-[10px] font-black uppercase tracking-widest";
                }
            });
        };

        window.showDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            const modal = document.getElementById('detailModal');
            const content = document.getElementById('detailContent');
            
            content.innerHTML = `
                <div class="p-8 space-y-8">
                    <div class="flex justify-between items-center border-b border-slate-100 pb-6">
                        <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-12">
                        <div class="text-right">
                            <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Bon d'expédition</p>
                            <p class="text-xl font-black text-slate-900">#${m.id}</p>
                        </div>
                    </div>

                    <div class="grid grid-cols-2 gap-8">
                        <div class="space-y-2">
                            <p class="text-[9px] font-black text-slate-400 uppercase tracking-[0.2em]">Expéditeur</p>
                            <p class="text-sm font-black text-slate-800">${m.en}</p>
                            <p class="text-[11px] text-slate-500 font-bold uppercase">${m.eq}</p>
                            <p class="text-[11px] text-slate-500 font-bold">${m.et || ''}</p>
                        </div>
                        <div class="space-y-2 text-right">
                            <p class="text-[9px] font-black text-slate-400 uppercase tracking-[0.2em]">Destinataire</p>
                            <p class="text-sm font-black text-slate-800">${m.dn}</p>
                            <p class="text-[11px] text-slate-500 font-bold uppercase">${m.dq}</p>
                            <p class="text-[11px] text-slate-500 font-bold">${m.dt}</p>
                        </div>
                    </div>

                    <div class="bg-slate-50 p-6 rounded-[2rem] border border-slate-100 space-y-4">
                        <div class="flex justify-between items-center">
                            <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Montant Course</span>
                            <span class="text-lg font-black text-slate-900">${m.p.toLocaleString()} CFA</span>
                        </div>
                        <div class="flex justify-between items-center pt-4 border-t border-slate-200">
                            <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Règlement</span>
                            <span class="text-xs font-black uppercase text-indigo-600">${['CASH', 'AIRTEL MONEY', 'MOOV MONEY'][m.r || 0]}</span>
                        </div>
                    </div>

                    ${m.img ? `
                        <div class="space-y-3">
                            <p class="text-[9px] font-black text-slate-400 uppercase tracking-widest">Preuve de Livraison</p>
                            <img src="${m.img}" class="w-full rounded-[2rem] shadow-lg border border-slate-100">
                        </div>
                    ` : ''}
                </div>
            `;
            modal.classList.remove('hidden');
        };

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background: #f8fafc; overflow-x: hidden; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        @media print { .no-print { display: none !important; } .print-only { display: block !important; } }
    </style>
</head>
<body class="max-w-md mx-auto min-h-screen bg-white shadow-2xl relative">

    <!-- ÉCRAN LOGIN -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 z-[1000] flex items-center justify-center p-8">
        <div class="w-full bg-white p-10 rounded-[3.5rem] text-center space-y-10 animate-in fade-in zoom-in duration-500">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-24 mx-auto drop-shadow-2xl">
            <div>
                <h1 class="text-3xl font-black text-slate-900 tracking-tighter">CT241 LOG</h1>
                <p class="text-slate-400 text-[10px] font-black uppercase tracking-[0.3em] mt-1">Plateforme Logistique</p>
            </div>
            <form onsubmit="handleLogin(event)" class="space-y-4 text-left">
                <div class="space-y-1">
                    <label class="text-[9px] font-black text-slate-400 uppercase ml-4">Email Pro</label>
                    <input type="email" id="loginEmail" placeholder="admin@ct241.com" class="w-full p-5 bg-slate-100 rounded-2xl text-xs font-bold outline-none focus:ring-2 ring-indigo-500" required>
                </div>
                <div class="space-y-1">
                    <label class="text-[9px] font-black text-slate-400 uppercase ml-4">Clé d'accès</label>
                    <input type="password" id="loginPass" placeholder="••••••••" class="w-full p-5 bg-slate-100 rounded-2xl text-xs font-bold outline-none focus:ring-2 ring-indigo-500" required>
                </div>
                <p id="loginError" class="text-red-500 text-[10px] font-bold hidden text-center"></p>
                <button id="loginBtn" class="w-full bg-slate-900 text-white py-6 rounded-2xl font-black text-xs uppercase shadow-2xl tracking-[0.2em] active:scale-95 transition-transform">Se connecter</button>
            </form>
        </div>
    </section>

    <!-- CONTENU PRINCIPAL -->
    <main id="appContent" class="hidden pb-32">
        <!-- HEADER -->
        <header class="p-6 sticky top-0 bg-white/80 backdrop-blur-xl z-[100] border-b border-slate-50 flex justify-between items-center no-print">
            <div class="flex items-center gap-4">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-10">
                <div id="userBadge" class="animate-pulse">...</div>
            </div>
            <button onclick="handleLogout()" class="h-12 w-12 bg-slate-50 rounded-2xl flex items-center justify-center text-xl hover:bg-slate-100 transition-colors">👋</button>
        </header>

        <div class="p-6 space-y-10 no-print">
            
            <!-- VUE ADMIN : STATS -->
            <section class="role-view hidden space-y-4">
                <div class="grid grid-cols-2 gap-4">
                    <div class="bg-indigo-600 p-7 rounded-[2.5rem] text-white shadow-xl shadow-indigo-100 relative overflow-hidden">
                        <p class="text-[9px] font-black text-indigo-300 uppercase tracking-widest mb-1">Chiffre d'affaires</p>
                        <h2 id="statCA" class="text-xl font-black">0 CFA</h2>
                        <div class="absolute -right-4 -bottom-4 opacity-10 text-6xl">💰</div>
                    </div>
                    <div class="bg-white border border-slate-100 p-7 rounded-[2.5rem] shadow-sm">
                        <p class="text-[9px] font-black text-slate-400 uppercase tracking-widest mb-1">Total Expéditions</p>
                        <h2 id="statCount" class="text-xl font-black text-slate-900">0</h2>
                    </div>
                </div>
            </section>

            <!-- VUE RELAIS : CRÉATION -->
            <section id="relaisSection" class="role-view hidden">
                <div class="bg-slate-900 p-8 rounded-[3rem] shadow-2xl text-white space-y-8">
                    <h3 class="text-xs font-black uppercase tracking-[0.2em] flex items-center gap-3">
                        <span class="w-8 h-8 bg-white/10 rounded-full flex items-center justify-center text-lg">📦</span>
                        Nouvel Envoi
                    </h3>
                    
                    <div class="space-y-4">
                        <div class="grid grid-cols-1 gap-3">
                            <input type="text" id="en" placeholder="Nom de la Boutique / Expéditeur" class="w-full p-4 bg-white/5 rounded-xl text-xs font-bold outline-none border border-white/10 placeholder:text-slate-500 focus:bg-white/10">
                            <div class="grid grid-cols-2 gap-3">
                                <input type="text" id="eq" placeholder="Quartier Exp" class="p-4 bg-white/5 rounded-xl text-xs font-bold outline-none border border-white/10 placeholder:text-slate-500">
                                <input type="tel" id="et" placeholder="Téléphone Exp" class="p-4 bg-white/5 rounded-xl text-xs font-bold outline-none border border-white/10 placeholder:text-slate-500">
                            </div>
                        </div>

                        <div class="pt-4 border-t border-white/10 grid grid-cols-1 gap-3">
                            <input type="text" id="dn" placeholder="Nom du Destinataire" class="w-full p-4 bg-white/5 rounded-xl text-xs font-bold outline-none border border-white/10 placeholder:text-slate-500">
                            <div class="grid grid-cols-2 gap-3">
                                <input type="text" id="dq" placeholder="Quartier Dest" class="p-4 bg-white/5 rounded-xl text-xs font-bold outline-none border border-white/10 placeholder:text-slate-500">
                                <input type="tel" id="dt" placeholder="Téléphone Dest" class="p-4 bg-white/5 rounded-xl text-xs font-bold outline-none border border-white/10 placeholder:text-slate-500">
                            </div>
                        </div>

                        <div class="grid grid-cols-2 gap-3">
                            <select id="r" class="p-4 bg-white/5 rounded-xl text-[10px] font-black uppercase outline-none border border-white/10">
                                <option value="0" class="text-black">Paiement Cash</option>
                                <option value="1" class="text-black">Airtel Money</option>
                                <option value="2" class="text-black">Moov Money</option>
                            </select>
                            <input type="number" id="p" placeholder="Prix Livraison" class="p-4 bg-emerald-500 text-white rounded-xl text-xs font-black outline-none placeholder:text-emerald-100">
                        </div>
                    </div>

                    <button onclick="creerMission()" class="w-full bg-white text-slate-900 py-5 rounded-2xl font-black text-xs uppercase tracking-widest shadow-xl active:scale-95 transition-transform">VALIDER LE DÉPART</button>
                </div>
            </section>

            <!-- VUE LISTE (COMMUNE) -->
            <section class="space-y-6">
                <div class="flex items-center justify-between mb-2">
                    <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Suivi d'activité</h3>
                </div>
                
                <div class="flex gap-3 overflow-x-auto no-scrollbar pb-2">
                    <button data-tab="all" onclick="switchTab('all')" class="tab-btn">Toutes</button>
                    <button data-tab="wait" onclick="switchTab('wait')" class="tab-btn">Attente</button>
                    <button data-tab="liv" onclick="switchTab('liv')" class="tab-btn">Livrées</button>
                </div>

                <div id="missionContainer" class="animate-in fade-in slide-in-from-bottom-6 duration-700"></div>
            </section>
        </div>
    </main>

    <!-- MODALE DÉTAILS -->
    <div id="detailModal" class="fixed inset-0 bg-slate-900/60 backdrop-blur-md z-[500] hidden flex items-center justify-center p-6">
        <div class="w-full max-w-lg bg-white rounded-[3rem] shadow-2xl overflow-y-auto max-h-[90vh]">
            <div id="detailContent"></div>
            <div class="p-8 bg-slate-50 flex gap-3 no-print">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-5 rounded-2xl text-[10px] tracking-widest uppercase shadow-xl">🖨️ Imprimer</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-8 bg-white border border-slate-200 text-slate-400 font-black py-5 rounded-2xl text-[10px] uppercase">Fermer</button>
            </div>
        </div>
    </div>

    <!-- MODALE CAMERA -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[1000] hidden flex flex-col items-center justify-center p-10 text-white">
        <div class="text-center space-y-10 animate-in zoom-in duration-300">
            <div class="w-28 h-28 bg-white/10 rounded-full flex items-center justify-center text-6xl mx-auto border border-white/5">📸</div>
            <div class="space-y-2">
                <h2 class="text-2xl font-black uppercase tracking-tight">Preuve de Livraison</h2>
                <p class="text-slate-500 text-[10px] font-bold uppercase tracking-widest">Photographiez le colis ou le bordereau signé</p>
            </div>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-indigo-500 text-white font-black py-6 rounded-[2.5rem] shadow-2xl shadow-indigo-500/20 uppercase tracking-[0.2em] active:scale-95 transition-transform">OUVRIR L'APPAREIL</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-600 text-[10px] font-black uppercase tracking-widest">Abandonner</button>
        </div>
    </div>

</body>
</html>
