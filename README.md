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

    <link rel="manifest" id="pwa-manifest">
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <script src="https://cdn.tailwindcss.com"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, deleteDoc, onSnapshot, query, orderBy } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

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

        let currentUser = null;
        let userRole = null;
        let allMissions = [];
        let currentMissionId = null;
        let searchQuery = "";
        let filterToday = false;

        // AUTHENTIFICATION
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            try { await signInWithEmailAndPassword(auth, email, pass); }
            catch (error) { document.getElementById('loginError').classList.remove('hidden'); }
        };

        window.handleLogout = async () => { await signOut(auth); window.location.reload(); };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                assignRole(user.email);
                document.getElementById('authSection').classList.add('hidden');
                document.getElementById('appContent').classList.remove('hidden');
                document.getElementById('userDisplayEmail').innerText = user.email;
                startRealtimeListeners();
                prepareNextId();
            } else {
                document.getElementById('authSection').classList.remove('hidden');
                document.getElementById('appContent').classList.add('hidden');
            }
        });

        const assignRole = (email) => {
            const e = email.toLowerCase();
            if (e.includes('admin')) userRole = 'admin';
            else if (e.includes('relais') || e.includes('boutique')) userRole = 'relais';
            else if (e.includes('dispatch')) userRole = 'dispatch';
            else if (e.includes('livreur')) userRole = 'livreur';
            else userRole = 'visiteur';
            
            document.querySelectorAll('.role-section').forEach(s => s.classList.add('hidden'));
            const badge = document.getElementById('badgeDisplay');
            const colors = { admin: 'bg-red-600', dispatch: 'bg-blue-600', relais: 'bg-emerald-600', livreur: 'bg-amber-600' };
            badge.innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-black uppercase text-white ${colors[userRole] || 'bg-slate-500'}">${userRole}</span>`;

            if (userRole === 'admin') {
                document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
            } else if (userRole === 'relais') {
                document.getElementById('rubrique1').classList.remove('hidden');
                document.getElementById('rubrique4').classList.remove('hidden');
            } else if (userRole === 'dispatch') {
                document.getElementById('rubrique2').classList.remove('hidden');
                document.getElementById('rubrique4').classList.remove('hidden');
            } else if (userRole === 'livreur') {
                document.getElementById('rubrique3').classList.remove('hidden');
                document.getElementById('adminDash').classList.remove('hidden');
            }
        };

        // LOGIQUE MÉTIER ET PERSISTANCE
        const prepareNextId = () => {
            window.nextId = "2026-" + Math.floor(Math.random() * 900000 + 100000);
            document.getElementById('displayNextId').innerText = window.nextId;
        };

        window.genererMission = async () => {
            const fields = {
                en: document.getElementById('expNom').value,
                eq: document.getElementById('expQuartier').value,
                et: document.getElementById('expTel').value,
                dn: document.getElementById('destNom').value,
                dq: document.getElementById('destQuartier').value,
                dt: document.getElementById('destTel').value,
                n: parseInt(document.getElementById('natureColis').value),
                v: parseFloat(document.getElementById('valeurDeclaree').value) || 0,
                p: parseFloat(document.getElementById('fraisLivraison').value) || 0,
                r: parseInt(document.getElementById('modeReglement').value),
            };

            if (!fields.en || !fields.dt || !fields.p) return showToast("Champs obligatoires manquants", "error");

            try {
                const now = Date.now();
                // Utilisation de la collection racine stable 'missions'
                await setDoc(doc(db, 'missions', window.nextId), {
                    ...fields, id: window.nextId, s: 0, ca: now, cad: new Date(now).toISOString().split('T')[0], ce: currentUser.email
                });
                showToast("Mission créée avec succès", "success");
                prepareNextId();
                ['expNom','expQuartier','expTel','destNom','destTel','valeurDeclaree','fraisLivraison'].forEach(id => document.getElementById(id).value = "");
            } catch (e) { showToast("Erreur de création", "error"); }
        };

        window.supprimerMission = async (id) => {
            if (confirm("Supprimer définitivement ?")) {
                try {
                    await deleteDoc(doc(db, 'missions', id));
                    showToast("Mission supprimée", "success");
                } catch (e) { showToast("Erreur suppression", "error"); }
            }
        };

        window.publierMission = async (id) => {
            try {
                await updateDoc(doc(db, 'missions', id), { s: 1 });
                showToast("Mission expédiée", "success");
            } catch (e) { showToast("Erreur expédition", "error"); }
        };

        window.toggleFilterToday = () => {
            filterToday = !filterToday;
            document.getElementById('btnFilterToday').className = filterToday ? "bg-yellow-500 text-slate-900 px-5 py-2.5 rounded-2xl font-black text-[10px] uppercase" : "bg-slate-100 text-slate-400 px-5 py-2.5 rounded-2xl font-black text-[10px] uppercase";
            renderUI();
        };

        const startRealtimeListeners = () => {
            // Écoute de la collection racine pour la persistance
            const q = query(collection(db, 'missions'), orderBy('ca', 'desc'));
            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
                renderStats();
            });
        };

        const renderStats = () => {
            const today = new Date().toISOString().split('T')[0];
            const stats = { caTotal: 0, livraisonsTotal: 0, caJour: 0, livreurs: {} };
            
            allMissions.forEach(m => {
                if (m.s === 2) {
                    stats.caTotal += m.p;
                    stats.livraisonsTotal++;
                    if (m.lk === today) stats.caJour += m.p;
                    if (!stats.livreurs[m.le]) stats.livreurs[m.le] = { count: 0, ca: 0, bonus: 0 };
                    stats.livreurs[m.le].count++;
                    stats.livreurs[m.le].ca += m.p;
                    if (stats.livreurs[m.le].count > 17) stats.livreurs[m.le].bonus += 700;
                }
            });

            if (userRole === 'admin') {
                document.getElementById('dashCaTotal').innerText = stats.caTotal.toLocaleString() + ' CFA';
                document.getElementById('dashCaJour').innerText = stats.caJour.toLocaleString() + ' CFA';
                document.getElementById('dashLivTotal').innerText = stats.livraisonsTotal;
                const list = document.getElementById('dashLivreursList');
                list.innerHTML = "";
                Object.entries(stats.livreurs).forEach(([email, data]) => {
                    list.innerHTML += `<div class="flex justify-between items-center p-4 bg-slate-800 rounded-2xl mb-2"><div class="flex flex-col"><span class="text-white font-bold text-[10px]">${email}</span><span class="text-[8px] text-slate-400 uppercase">${data.ca.toLocaleString()} CFA</span></div><div class="text-right"><span class="text-yellow-500 font-black text-[11px] block">${data.count} livr.</span><span class="text-emerald-500 font-black text-[9px]">+${data.bonus} CFA</span></div></div>`;
                });
            }

            if (userRole === 'livreur') {
                const my = stats.livreurs[currentUser.email] || { count: 0, bonus: 0 };
                document.getElementById('dashLivTotal').innerText = my.count;
                document.getElementById('dashCaJour').innerText = my.bonus.toLocaleString() + ' CFA';
            }
        };

        window.renderUI = () => {
            const containerDispatch = document.getElementById('containerDispatch');
            const containerLivreur = document.getElementById('containerLivreur');
            const archiveBody = document.getElementById('archiveBody');
            
            if(containerDispatch) containerDispatch.innerHTML = "";
            if(containerLivreur) containerLivreur.innerHTML = "";
            if(archiveBody) archiveBody.innerHTML = "";
            
            const today = new Date().toISOString().split('T')[0];

            allMissions.forEach(m => {
                if (filterToday && (m.lk !== today && m.cad !== today)) return;
                const isMyColis = m.ce === currentUser.email;
                const delBtn = userRole === 'admin' ? `<button onclick="supprimerMission('${m.id}')" class="p-2 text-red-400"><svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" /></svg></button>` : '';

                if (m.s === 0 && (userRole === 'admin' || userRole === 'dispatch')) {
                    containerDispatch.innerHTML += `<div class="p-5 bg-white rounded-[2rem] mb-4 shadow-sm flex justify-between items-center"><div class="flex flex-col"><span class="text-[10px] font-black text-blue-600">${m.id}</span><span class="font-extrabold text-slate-900 text-sm">${m.dn}</span><span class="text-[10px] text-slate-400 font-bold uppercase">${m.dq}</span></div><div class="flex gap-2">${delBtn}<button onclick="openBonImpression('${m.id}')" class="bg-slate-100 text-[10px] px-4 py-3 rounded-2xl font-black uppercase">Bon</button><button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white text-[10px] px-5 py-3 rounded-2xl font-black uppercase">Expédier</button></div></div>`;
                } else if (m.s === 1 && (userRole === 'admin' || userRole === 'livreur')) {
                    containerLivreur.innerHTML += `<div class="p-6 bg-amber-50 rounded-[2.5rem] mb-5 space-y-4 border border-amber-100"><div class="flex justify-between items-center"><span class="bg-amber-500 text-white text-[9px] font-black px-3 py-1 rounded-full">${m.id}</span><span class="text-[10px] font-black text-amber-700 uppercase">En livraison</span></div><div class="space-y-1"><p class="text-sm font-extrabold text-slate-900">${m.dn}</p><p class="text-[11px] font-bold text-slate-500">${m.dq} • ${m.dt}</p></div><div class="flex gap-3"><button onclick="openBonImpression('${m.id}')" class="flex-1 bg-white border border-amber-200 text-amber-700 font-black py-4 rounded-2xl text-[10px] uppercase">Voir Bon</button><button onclick="openCamera('${m.id}')" class="flex-[2] bg-amber-500 text-white font-black py-4 rounded-2xl text-[10px] uppercase">Valider ✅</button></div></div>`;
                }

                if (m.s === 2) {
                    const canSeeArchive = userRole === 'admin' || userRole === 'dispatch' || (userRole === 'relais' && isMyColis);
                    if (canSeeArchive) {
                        const matchSearch = !searchQuery || m.id.toLowerCase().includes(searchQuery) || m.dn.toLowerCase().includes(searchQuery);
                        if (matchSearch) {
                            archiveBody.innerHTML += `<tr class="border-b border-slate-50 text-[11px]"><td onclick="openArchiveDetail('${m.id}')" class="p-5 font-black text-slate-900">${m.id}</td><td onclick="openArchiveDetail('${m.id}')" class="p-5"><b>${m.dn}</b><br><span class="text-[9px] text-slate-400 font-bold uppercase">${m.lk}</span></td><td class="p-5 text-center">${delBtn}</td><td class="p-5 text-right font-black text-emerald-600">${m.p.toLocaleString()} CFA</td></tr>`;
                        }
                    }
                }
            });
            document.getElementById('archiveCount').innerText = archiveBody.children.length;
        };

        window.openCamera = (id) => { currentMissionId = id; document.getElementById('cameraModal').classList.remove('hidden'); };
        window.processImage = (file) => {
            const reader = new FileReader();
            reader.onload = (e) => finalizeDelivery(e.target.result);
            reader.readAsDataURL(file);
        };
        const finalizeDelivery = async (photoBase64) => {
            try {
                await updateDoc(doc(db, 'missions', currentMissionId), { s: 2, img: photoBase64, le: currentUser.email, lk: new Date().toISOString().split('T')[0] });
                document.getElementById('cameraModal').classList.add('hidden');
                showToast("Mission livrée et archivée", "success");
            } catch (e) { showToast("Erreur validation", "error"); }
        };

        window.handleSearch = (v) => { searchQuery = v.toLowerCase(); renderUI(); };
        const showToast = (m, t) => {
            const el = document.getElementById('toast');
            el.innerText = m; el.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-8 py-4 rounded-3xl text-white font-black text-[11px] z-[600] uppercase tracking-widest ${t==='success'?'bg-emerald-600':'bg-red-600'}`;
            el.classList.remove('hidden'); setTimeout(() => el.classList.add('hidden'), 3000);
        };

        window.openBonImpression = (id) => {
            const m = allMissions.find(x => x.id === id);
            const natureMap = ["📦 Standard", "🍟 Repas", "📁 Documents", "💊 Pharma"];
            const modeMap = ["Espèces", "Airtel Money", "Moov Money"];
            document.getElementById('detailContent').innerHTML = `<div class="print-area bg-white p-10 text-slate-900"><div class="flex justify-between items-start mb-10 pb-8 border-b-4 border-slate-900"><div><h1 class="text-5xl font-black text-slate-900">CT241</h1><p class="text-[11px] font-black uppercase tracking-[0.3em] text-blue-600">Logistique Express</p></div><div class="text-right"><h2 class="text-xl font-black uppercase bg-slate-900 text-white px-4 py-1 rounded-lg">Bon de Livraison</h2><p class="text-sm font-black text-slate-400">Réf: ${m.id}</p></div></div><div class="grid grid-cols-2 gap-12 mb-10"><div><p class="text-[10px] font-black text-slate-400 uppercase">Expéditeur</p><p class="font-extrabold text-lg">${m.en}</p><p class="text-sm font-bold text-slate-500">${m.eq} • ${m.et}</p></div><div class="text-right"><p class="text-[10px] font-black text-slate-400 uppercase">Destinataire</p><p class="font-extrabold text-lg">${m.dn}</p><p class="text-sm font-bold text-slate-500">${m.dq} • ${m.dt}</p></div></div><div class="bg-slate-50 rounded-[2rem] p-8 mb-10 border-2 border-slate-100"><table class="w-full"><tr><td class="py-4 text-[11px] font-black text-slate-400 uppercase">Nature</td><td class="py-4 text-right font-black text-slate-900">${natureMap[m.n]}</td></tr><tr><td class="py-4 text-[11px] font-black text-slate-400 uppercase">Paiement</td><td class="py-4 text-right font-black text-blue-600 uppercase">${modeMap[m.r]}</td></tr></table></div><div class="text-right mb-16"><p class="text-[11px] font-black text-slate-400 uppercase mb-1">Total à percevoir</p><p class="text-5xl font-black text-slate-900">${m.p.toLocaleString()} <span class="text-lg">CFA</span></p></div><div class="grid grid-cols-2 gap-10 mt-20"><div class="border-t-2 border-slate-200 pt-6 text-center text-[10px] font-black uppercase text-slate-300">Visa Expéditeur</div><div class="border-t-2 border-slate-200 pt-6 text-center text-[10px] font-black uppercase text-slate-300">Visa Réception</div></div></div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        window.openArchiveDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            document.getElementById('detailContent').innerHTML = `<div class="p-8 space-y-8"><div class="flex justify-between items-center bg-emerald-50 p-6 rounded-[2rem] border border-emerald-100"><div><span class="text-[10px] font-black text-emerald-600 uppercase mb-1">Livraison Réussie</span><h2 class="text-2xl font-black text-emerald-900">${m.id}</h2></div><div class="w-14 h-14 bg-emerald-500 rounded-2xl flex items-center justify-center text-white text-2xl">✅</div></div><div class="space-y-3"><p class="text-[10px] font-black text-slate-400 uppercase tracking-widest ml-2">Preuve Visuelle</p>${m.img ? `<img src="${m.img}" class="w-full rounded-[2.5rem] border-4 border-white shadow-xl">` : '<div class="h-60 bg-slate-100 rounded-[2.5rem] flex items-center justify-center italic text-slate-400 font-bold">Aucune photo</div>'}</div><div class="grid grid-cols-2 gap-4"><div class="bg-slate-50 p-6 rounded-[2rem]"><p class="text-[9px] font-black text-slate-400 uppercase">Livreur</p><p class="text-xs font-black text-slate-900 truncate">${m.le}</p></div><div class="bg-slate-50 p-6 rounded-[2rem]"><p class="text-[9px] font-black text-slate-400 uppercase">Date</p><p class="text-xs font-black text-slate-900">${m.lk}</p></div></div><div class="bg-slate-900 p-8 rounded-[2.5rem] text-white flex justify-between items-center"><span class="text-[11px] font-black uppercase opacity-50">Recette</span><span class="text-3xl font-black text-emerald-400">${m.p.toLocaleString()} CFA</span></div></div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; color: #0f172a; -webkit-tap-highlight-color: transparent; }
        .hidden { display: none; }
        @media print { body * { visibility: hidden; } .print-area, .print-area * { visibility: visible; } .print-area { position: absolute; left: 0; top: 0; width: 100%; } .no-print { display: none !important; } }
    </style>
</head>
<body class="antialiased select-none">
    <div id="toast" class="hidden"></div>

    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[3.5rem] p-12 space-y-10 shadow-2xl text-center border-4 border-slate-800">
            <div class="relative inline-block"><img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-28 mx-auto drop-shadow-2xl"><div class="absolute -bottom-2 -right-2 bg-slate-900 text-white text-[8px] font-black px-2 py-1 rounded-full uppercase tracking-widest">v2.1</div></div>
            <div><h1 class="text-3xl font-black text-slate-900 uppercase tracking-tighter leading-none">CT241 PORTAIL</h1><p class="text-[10px] font-bold text-slate-400 mt-2 uppercase tracking-[0.2em]">Logistique & Performance</p></div>
            <form onsubmit="handleLogin(event)" class="space-y-5">
                <input type="email" id="loginEmail" required class="w-full p-5 bg-slate-50 border-2 border-slate-100 rounded-[1.5rem] font-bold text-[11px] focus:border-slate-900 outline-none" placeholder="Adresse e-mail">
                <input type="password" id="loginPass" required class="w-full p-5 bg-slate-50 border-2 border-slate-100 rounded-[1.5rem] font-bold text-[11px] focus:border-slate-900 outline-none" placeholder="Mot de passe">
                <p id="loginError" class="text-[9px] text-red-500 font-black uppercase hidden">Identifiants incorrects</p>
                <button type="submit" class="w-full bg-slate-900 text-white font-black py-5 rounded-[1.5rem] shadow-2xl text-[11px] uppercase tracking-widest">Accéder au système</button>
            </form>
        </div>
    </section>

    <main id="appContent" class="hidden min-h-screen pb-32">
        <nav class="bg-white/90 backdrop-blur-xl p-6 sticky top-0 z-50 border-b-2 border-slate-50 shadow-sm">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-4">
                    <div class="p-1 bg-slate-50 rounded-2xl border border-slate-100"><img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-10"></div>
                    <div class="flex flex-col"><span class="text-[10px] font-black text-slate-900 truncate w-28" id="userDisplayEmail">...</span><div id="badgeDisplay" class="mt-0.5"></div></div>
                </div>
                <div class="flex gap-3">
                    <button onclick="toggleFilterToday()" id="btnFilterToday" class="bg-slate-100 text-slate-400 px-5 py-2.5 rounded-2xl font-black text-[10px] uppercase">Aujourd'hui</button>
                    <button onclick="handleLogout()" class="p-3 bg-slate-50 rounded-2xl text-slate-400 border border-slate-100"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m0 0l-4-4m4 4H7" /></svg></button>
                </div>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-6 space-y-10">
            <section id="adminDash" class="role-section hidden space-y-6">
                <div class="grid grid-cols-3 gap-4">
                    <div class="bg-white p-6 rounded-[2rem] shadow-sm border border-slate-100 text-center"><span class="text-[9px] font-black text-slate-300 uppercase block">Recette</span><div id="dashCaTotal" class="text-[12px] font-black text-slate-900">0 CFA</div></div>
                    <div class="bg-white p-6 rounded-[2rem] shadow-sm border border-slate-100 text-center"><span class="text-[9px] font-black text-slate-300 uppercase block">Bonus</span><div id="dashCaJour" class="text-[12px] font-black text-emerald-600">0 CFA</div></div>
                    <div class="bg-white p-6 rounded-[2rem] shadow-sm border border-slate-100 text-center"><span class="text-[9px] font-black text-slate-300 uppercase block">Livrées</span><div id="dashLivTotal" class="text-[12px] font-black text-blue-600">0</div></div>
                </div>
                <div class="bg-slate-900 p-6 rounded-[3rem] space-y-3" id="dashLivreursList"></div>
            </section>

            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[3.5rem] shadow-2xl border-2 border-slate-50 overflow-hidden">
                    <div class="bg-slate-900 p-8 text-white flex justify-between items-center">
                        <div><p class="text-[10px] font-black uppercase opacity-40 tracking-[0.3em] mb-1">Édition Bordereau</p><h2 class="font-black text-lg">MISSION : <span id="displayNextId" class="text-blue-400">...</span></h2></div>
                        <div class="w-14 h-14 bg-white/10 rounded-2xl flex items-center justify-center text-2xl">📦</div>
                    </div>
                    <div class="p-8 space-y-6">
                        <div class="space-y-4">
                            <p class="text-[10px] font-black text-slate-300 uppercase ml-2">Expédition</p>
                            <input type="text" id="expNom" placeholder="Nom Expéditeur" class="w-full p-5 bg-slate-50 rounded-2xl font-bold text-[12px] outline-none">
                            <div class="grid grid-cols-2 gap-4">
                                <input type="text" id="expQuartier" placeholder="Quartier" class="p-5 bg-slate-50 rounded-2xl font-bold text-[12px] outline-none">
                                <input type="tel" id="expTel" placeholder="Contact Tél" class="p-5 bg-slate-50 rounded-2xl font-bold text-[12px] outline-none">
                            </div>
                        </div>
                        <div class="space-y-4">
                            <p class="text-[10px] font-black text-slate-300 uppercase ml-2">Destinataire</p>
                            <input type="text" id="destNom" placeholder="Nom Client" class="w-full p-5 bg-slate-50 rounded-2xl font-bold text-[12px] outline-none">
                            <div class="grid grid-cols-2 gap-4">
                                <select id="destQuartier" class="p-5 bg-slate-50 rounded-2xl font-black text-[11px] outline-none">
                                    <option value="" disabled selected>Quartier Client</option>
                                    <optgroup label="Akanda"><option>Angondjé</option><option>Avorbam</option></optgroup>
                                    <optgroup label="Libreville"><option>Nzeng-Ayong</option><option>Oloumi</option><option>Akébé</option><option>Pk0-Pk12</option></optgroup>
                                    <optgroup label="Autres"><option>Owendo</option></optgroup>
                                </select>
                                <input type="tel" id="destTel" placeholder="Contact Client" class="p-5 bg-slate-50 rounded-2xl font-bold text-[12px] outline-none">
                            </div>
                        </div>
                        <div class="grid grid-cols-2 gap-4">
                            <select id="natureColis" class="p-5 bg-slate-900 text-white rounded-2xl font-black text-[11px] outline-none"><option value="0">📦 Standard</option><option value="1">🍟 Repas</option><option value="2">📁 Documents</option><option value="3">💊 Pharma</option></select>
                            <select id="modeReglement" class="p-5 bg-slate-100 rounded-2xl font-black text-[11px] outline-none"><option value="0">Espèces</option><option value="1">Airtel Money</option><option value="2">Moov Money</option></select>
                        </div>
                        <div class="grid grid-cols-2 gap-4">
                            <input type="number" id="valeurDeclaree" placeholder="Valeur (CFA)" class="p-5 bg-slate-50 rounded-2xl font-bold text-[12px] outline-none">
                            <input type="number" id="fraisLivraison" placeholder="PRIX LIVR." class="p-5 bg-blue-600 text-white rounded-2xl font-black text-[14px] outline-none shadow-xl shadow-blue-600/30">
                        </div>
                        <button onclick="genererMission()" class="w-full bg-slate-900 text-white font-black py-6 rounded-3xl text-[12px] uppercase tracking-[0.2em] border-b-4 border-blue-600">Enregistrer Mission</button>
                    </div>
                </div>
            </section>

            <section id="rubrique2" class="role-section hidden space-y-5">
                <h3 class="text-[11px] font-black text-slate-900 uppercase tracking-widest ml-2">Attente Dispatching</h3>
                <div id="containerDispatch"></div>
            </section>

            <section id="rubrique3" class="role-section hidden space-y-5">
                <h3 class="text-[11px] font-black text-slate-900 uppercase tracking-widest ml-2">Tes Courses Actives</h3>
                <div id="containerLivreur"></div>
            </section>

            <section id="rubrique4" class="role-section hidden">
                <div class="bg-white rounded-[3.5rem] overflow-hidden shadow-2xl border-2 border-slate-50">
                    <div class="p-8 border-b-2 border-slate-50 bg-slate-50/50">
                        <div class="flex justify-between items-center mb-6"><h2 class="text-slate-900 font-black text-sm uppercase tracking-tighter">Archives Flotte</h2><div id="archiveCount" class="bg-slate-900 text-white font-black text-[10px] px-4 py-1.5 rounded-full">0</div></div>
                        <input type="text" oninput="handleSearch(this.value)" placeholder="Rechercher ID ou Client..." class="w-full p-5 bg-white border-2 border-slate-100 rounded-[1.8rem] text-[11px] font-bold outline-none">
                    </div>
                    <div class="overflow-x-auto"><table class="w-full text-left"><tbody id="archiveBody"></tbody></table></div>
                </div>
            </section>
        </div>
    </main>

    <div id="detailModal" class="fixed inset-0 bg-slate-950/98 z-[400] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-2xl bg-white rounded-[4rem] overflow-hidden shadow-2xl">
            <div id="detailContent" class="max-h-[75vh] overflow-y-auto"></div>
            <div class="p-8 bg-slate-50 flex gap-4 no-print border-t border-slate-100">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-5 rounded-3xl text-[11px] uppercase tracking-widest">🖨️ Lancer l'impression</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-10 bg-white border border-slate-200 text-slate-500 font-black py-5 rounded-3xl text-[11px] uppercase">Quitter</button>
            </div>
        </div>
    </div>

    <div id="cameraModal" class="fixed inset-0 bg-black z-[300] hidden flex flex-col items-center justify-center p-8 text-white">
        <div class="text-center space-y-8 max-w-xs">
            <div class="w-24 h-24 bg-white/10 rounded-full flex items-center justify-center text-5xl mx-auto">📸</div>
            <h2 class="text-2xl font-black">Preuve de Livraison</h2>
            <p class="text-slate-500 text-xs">Photo claire du colis ou du bon signé.</p>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-5 rounded-3xl shadow-2xl uppercase text-[11px] tracking-widest">Ouvrir l'appareil</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 text-[10px] font-black underline uppercase">Annuler</button>
        </div>
    </div>
</body>
</html>
