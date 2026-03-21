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
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query, where, orderBy, limit } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

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
        let currentFilter = 'all';

        // --- PWA MANIFEST ---
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
            btn.innerHTML = '<span class="animate-spin inline-block mr-2">⏳</span> Connexion...';
            
            try {
                await signInWithEmailAndPassword(auth, email, pass);
            } catch (err) {
                document.getElementById('loginError').innerText = "Accès refusé. Vérifiez vos identifiants.";
                document.getElementById('loginError').classList.remove('hidden');
                btn.disabled = false;
                btn.innerText = "Se connecter";
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
            const badgeColors = { admin: 'bg-red-500', livreur: 'bg-amber-500', relais: 'bg-emerald-500' };
            document.getElementById('userBadge').className = `${badgeColors[userRole]} text-[10px] text-white px-3 py-1 rounded-full font-black tracking-widest`;

            // Filtrage des sections UI
            document.querySelectorAll('.role-view').forEach(v => v.classList.add('hidden'));
            if(userRole === 'admin') document.querySelectorAll('.role-view').forEach(v => v.classList.remove('hidden'));
            else if(userRole === 'livreur') document.getElementById('livreurDashboard').classList.remove('hidden');
            else document.getElementById('relaisDashboard').classList.remove('hidden');
        };

        // --- DATA SYNC (Optimisation 48h) ---
        const initDataSync = () => {
            const limitTime = Date.now() - (48 * 60 * 60 * 1000);
            const q = query(
                collection(db, 'artifacts', appId, 'public', 'data', 'missions'),
                where('ca', '>', limitTime)
            );

            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                render();
            });
        };

        // --- ACTIONS ---
        window.creerMission = async () => {
            const id = "CT-" + Math.floor(100000 + Math.random() * 900000);
            const data = {
                id,
                s: 0, // 0: Attente, 1: En Cours, 2: Livré
                en: document.getElementById('en').value,
                eq: document.getElementById('eq').value,
                et: document.getElementById('et').value,
                dn: document.getElementById('dn').value,
                dq: document.getElementById('dq').value,
                dt: document.getElementById('dt').value,
                n: parseInt(document.getElementById('n').value), // Nature
                p: parseInt(document.getElementById('p').value) || 0,
                r: parseInt(document.getElementById('r').value), // Règlement
                ce: currentUser.email,
                ca: Date.now()
            };

            if(!data.dn || !data.dt || !data.p) return alert("Veuillez remplir les champs obligatoires (Destinataire, Tél, Prix)");

            await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), data);
            alert("✅ Expédition enregistrée !");
            document.querySelectorAll('#relaisDashboard input').forEach(i => i.value = "");
        };

        window.validerLivraison = (id) => {
            window.targetMissionId = id;
            document.getElementById('cameraModal').classList.remove('hidden');
        };

        window.processImage = (file) => {
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    const MAX = 800;
                    canvas.width = MAX;
                    canvas.height = img.height * (MAX / img.width);
                    const ctx = canvas.getContext('2d');
                    ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
                    const base64 = canvas.toDataURL('image/jpeg', 0.6); // Compression optimale
                    saveLivraison(base64);
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };

        const saveLivraison = async (photo) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.targetMissionId), {
                s: 2,
                le: currentUser.email,
                img: photo,
                lk: new Date().toISOString().split('T')[0]
            });
            document.getElementById('cameraModal').classList.add('hidden');
        };

        // --- UI RENDER ---
        window.setFilter = (f) => { currentFilter = f; render(); };

        const render = () => {
            // Stats (Admin)
            if(userRole === 'admin') {
                const ca = allMissions.filter(m => m.s === 2).reduce((sum, m) => sum + m.p, 0);
                const count = allMissions.filter(m => m.s === 2).length;
                document.getElementById('statCa').innerText = ca.toLocaleString() + " CFA";
                document.getElementById('statCount').innerText = count;
            }

            // Listes
            const lists = [document.getElementById('listAll'), document.getElementById('listLiv')];
            lists.forEach(l => { if(l) l.innerHTML = ""; });

            const sorted = allMissions.sort((a,b) => b.ca - a.ca);

            sorted.forEach(m => {
                const isMine = m.ce === currentUser.email || userRole === 'admin' || userRole === 'livreur';
                if(!isMine) return;

                const card = `
                    <div onclick="showDetail('${m.id}')" class="bg-white p-5 rounded-3xl border border-slate-100 shadow-sm mb-4 active:scale-95 transition-transform">
                        <div class="flex justify-between items-start mb-3">
                            <span class="text-[9px] font-black text-slate-400 uppercase tracking-tighter">#${m.id}</span>
                            <span class="${m.s === 2 ? 'bg-emerald-100 text-emerald-600' : 'bg-amber-100 text-amber-600'} text-[8px] px-2 py-1 rounded-full font-black uppercase">
                                ${m.s === 2 ? 'Livré' : 'Attente'}
                            </span>
                        </div>
                        <div class="text-sm font-black text-slate-800">${m.dn}</div>
                        <div class="text-[10px] text-slate-400 font-bold uppercase tracking-wide">${m.dq}</div>
                        ${m.s < 2 && userRole !== 'relais' ? `
                            <button onclick="event.stopPropagation(); validerLivraison('${m.id}')" class="mt-4 w-full bg-slate-900 text-white py-3 rounded-2xl text-[10px] font-black uppercase tracking-widest">Confirmer Livraison</button>
                        ` : ''}
                    </div>
                `;

                if(document.getElementById('listAll')) document.getElementById('listAll').innerHTML += card;
                if(m.s === 2 && document.getElementById('listLiv')) document.getElementById('listLiv').innerHTML += card;
            });
        };

        window.showDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            const content = document.getElementById('detailContent');
            content.innerHTML = `
                <div class="p-8 space-y-6">
                    <div class="flex justify-between items-center border-b pb-6">
                        <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-12">
                        <div class="text-right">
                            <p class="text-[10px] font-black text-slate-400 uppercase">Bon d'expédition</p>
                            <p class="text-xl font-black text-slate-900">${m.id}</p>
                        </div>
                    </div>
                    <div class="grid grid-cols-2 gap-8 text-[11px]">
                        <div class="space-y-1">
                            <p class="font-black text-slate-400 uppercase tracking-widest text-[9px]">Expéditeur</p>
                            <p class="font-black text-slate-800 text-sm">${m.en}</p>
                            <p class="text-slate-500 font-bold">${m.eq}</p>
                            <p class="text-slate-500 font-bold">${m.et || ''}</p>
                        </div>
                        <div class="space-y-1">
                            <p class="font-black text-slate-400 uppercase tracking-widest text-[9px]">Destinataire</p>
                            <p class="font-black text-slate-800 text-sm">${m.dn}</p>
                            <p class="text-slate-500 font-bold">${m.dq}</p>
                            <p class="text-slate-500 font-bold">${m.dt}</p>
                        </div>
                    </div>
                    <div class="bg-slate-50 p-6 rounded-3xl space-y-4">
                        <div class="flex justify-between text-xs">
                            <span class="font-bold text-slate-500">Montant Livraison</span>
                            <span class="font-black text-slate-900">${m.p.toLocaleString()} CFA</span>
                        </div>
                        <div class="flex justify-between text-[10px] border-t pt-4">
                            <span class="text-slate-400 uppercase font-black">Règlement</span>
                            <span class="font-black">${['CASH', 'AIRTEL', 'MOOV'][m.r || 0]}</span>
                        </div>
                    </div>
                    ${m.img ? `
                        <div class="space-y-2">
                            <p class="text-[9px] font-black text-slate-400 uppercase tracking-widest">Preuve Image</p>
                            <img src="${m.img}" class="w-full rounded-3xl border shadow-sm">
                        </div>
                    ` : ''}
                </div>
            `;
            document.getElementById('detailModal').classList.remove('hidden');
        };

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background: #f8fafc; color: #1e293b; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        @media print { .no-print { display: none !important; } .print-only { display: block !important; } body { background: white; } }
    </style>
</head>
<body class="max-w-md mx-auto min-h-screen bg-white shadow-2xl overflow-x-hidden">

    <!-- LOGIN -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 z-[1000] flex items-center justify-center p-8">
        <div class="w-full bg-white p-10 rounded-[3.5rem] text-center space-y-8 animate-in fade-in zoom-in duration-500">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-20 mx-auto">
            <div>
                <h1 class="text-2xl font-black text-slate-900">CT241 LOG</h1>
                <p class="text-slate-400 text-[10px] font-black uppercase tracking-[0.2em]">Logistique & Performance</p>
            </div>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" placeholder="Email professionnel" class="w-full p-5 bg-slate-100 rounded-2xl text-xs font-bold outline-none focus:ring-2 ring-slate-200" required>
                <input type="password" id="loginPass" placeholder="Mot de passe" class="w-full p-5 bg-slate-100 rounded-2xl text-xs font-bold outline-none focus:ring-2 ring-slate-200" required>
                <p id="loginError" class="text-red-500 text-[10px] font-bold hidden"></p>
                <button id="loginBtn" class="w-full bg-slate-900 text-white py-5 rounded-2xl font-black text-xs uppercase shadow-2xl tracking-widest active:scale-95 transition-transform">Se connecter</button>
            </form>
        </div>
    </section>

    <!-- MAIN APP -->
    <main id="appContent" class="hidden pb-24">
        <!-- HEADER -->
        <header class="p-6 sticky top-0 bg-white/80 backdrop-blur-xl z-50 flex justify-between items-center border-b border-slate-50 no-print">
            <div class="flex items-center gap-4">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-10">
                <div id="userBadge">...</div>
            </div>
            <button onclick="handleLogout()" class="h-12 w-12 bg-slate-100 rounded-2xl flex items-center justify-center text-xl hover:bg-slate-200 transition-colors">🚪</button>
        </header>

        <div class="p-6 space-y-8 no-print">
            
            <!-- ADMIN VIEW -->
            <section class="role-view hidden space-y-4">
                <div class="grid grid-cols-2 gap-4">
                    <div class="bg-slate-900 p-6 rounded-[2.5rem] text-white shadow-xl relative overflow-hidden">
                        <p class="text-[8px] font-black text-slate-500 uppercase tracking-widest mb-1">CA (48h)</p>
                        <h2 id="statCa" class="text-lg font-black text-emerald-400">0 CFA</h2>
                        <div class="absolute -right-4 -bottom-4 h-12 w-12 bg-white/5 rounded-full"></div>
                    </div>
                    <div class="bg-slate-100 p-6 rounded-[2.5rem] text-slate-900 shadow-sm">
                        <p class="text-[8px] font-black text-slate-400 uppercase tracking-widest mb-1">Livraisons</p>
                        <h2 id="statCount" class="text-lg font-black">0</h2>
                    </div>
                </div>
            </section>

            <!-- DASHBOARD RELAIS (CRÉATION) -->
            <section id="relaisDashboard" class="role-view hidden space-y-6">
                <div class="bg-slate-50 p-8 rounded-[3rem] space-y-6 border border-slate-100">
                    <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-[0.2em]">Nouvelle Course</h3>
                    <div class="space-y-3">
                        <input type="text" id="en" placeholder="Nom Boutique Exp" class="w-full p-4 bg-white rounded-xl text-xs font-bold outline-none shadow-sm">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="eq" placeholder="Quartier Exp" class="p-4 bg-white rounded-xl text-xs font-bold outline-none shadow-sm">
                            <input type="tel" id="et" placeholder="Tél Exp" class="p-4 bg-white rounded-xl text-xs font-bold outline-none shadow-sm">
                        </div>
                    </div>
                    <hr class="border-slate-200 border-dashed">
                    <div class="space-y-3">
                        <input type="text" id="dn" placeholder="Nom Destinataire" class="w-full p-4 bg-white rounded-xl text-xs font-bold outline-none shadow-sm">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="dq" placeholder="Quartier Dest" class="p-4 bg-white rounded-xl text-xs font-bold outline-none shadow-sm">
                            <input type="tel" id="dt" placeholder="Tél Dest" class="p-4 bg-white rounded-xl text-xs font-bold outline-none shadow-sm">
                        </div>
                    </div>
                    <div class="grid grid-cols-2 gap-3">
                        <select id="n" class="p-4 bg-white rounded-xl text-xs font-bold outline-none shadow-sm">
                            <option value="0">Standard</option>
                            <option value="1">Repas</option>
                            <option value="2">Documents</option>
                        </select>
                        <select id="r" class="p-4 bg-white rounded-xl text-xs font-bold outline-none shadow-sm">
                            <option value="0">Cash</option>
                            <option value="1">Airtel Money</option>
                            <option value="2">Moov Money</option>
                        </select>
                    </div>
                    <input type="number" id="p" placeholder="Prix Livraison (CFA)" class="w-full p-5 bg-slate-900 text-white rounded-2xl text-sm font-black outline-none placeholder:text-slate-600">
                    <button onclick="creerMission()" class="w-full bg-emerald-600 text-white py-5 rounded-2xl font-black text-xs uppercase tracking-widest shadow-xl active:scale-95 transition-transform">Valider le départ</button>
                </div>
            </section>

            <!-- TABS & LISTS -->
            <section class="space-y-6">
                <div class="flex gap-4 overflow-x-auto no-scrollbar pb-2">
                    <button onclick="setFilter('all')" class="bg-slate-900 text-white px-6 py-3 rounded-2xl text-[10px] font-black uppercase tracking-widest flex-shrink-0">Toutes</button>
                    <button onclick="setFilter('liv')" class="bg-white border text-slate-400 px-6 py-3 rounded-2xl text-[10px] font-black uppercase tracking-widest flex-shrink-0">Livrées</button>
                </div>
                
                <div id="listAll" class="animate-in fade-in slide-in-from-bottom-4 duration-500"></div>
                <div id="listLiv" class="hidden"></div>
            </section>

        </div>
    </main>

    <!-- DETAIL MODAL -->
    <div id="detailModal" class="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-[200] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-2xl bg-white rounded-[2.5rem] shadow-2xl overflow-y-auto max-h-[90vh]">
            <div id="detailContent"></div>
            <div class="p-6 bg-slate-50 flex gap-2 no-print">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-4 rounded-2xl text-[10px] tracking-widest uppercase">🖨️ Imprimer Bon</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-6 bg-slate-200 text-slate-600 font-black py-4 rounded-2xl text-[10px] uppercase">Fermer</button>
            </div>
        </div>
    </div>

    <!-- CAMERA -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[3000] hidden flex flex-col items-center justify-center p-10 text-white">
        <div class="text-center space-y-8 animate-in zoom-in duration-300">
            <div class="w-24 h-24 bg-white/10 rounded-full flex items-center justify-center text-5xl mx-auto">📸</div>
            <div>
                <h2 class="text-2xl font-black uppercase">Preuve Livraison</h2>
                <p class="text-slate-500 text-[10px] mt-2 font-bold uppercase tracking-widest">Photographiez le colis ou le bon signé</p>
            </div>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-5 rounded-[2rem] shadow-2xl uppercase tracking-widest active:scale-95 transition-transform">Ouvrir l'appareil</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 text-[10px] font-black uppercase tracking-widest">Annuler</button>
        </div>
    </div>

</body>
</html>
