<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>CT241 - Logistique Gabon</title>
    
    <meta name="theme-color" content="#009E60">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="apple-mobile-web-app-title" content="CT241 Gabon">
    <link rel="apple-touch-icon" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">

    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
    <script src="https://unpkg.com/html5-qrcode"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, deleteDoc, onSnapshot, query, where, getDocs } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

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
        let html5QrCode = null;
        let currentMissionForPhoto = null;

        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value.trim().toLowerCase();
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            const err = document.getElementById('loginError');
            
            btn.disabled = true; btn.innerText = "ACCÈS EN COURS...";
            err.classList.add('hidden');

            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } catch (error) { 
                err.innerText = "Accès refusé. Identifiants invalides.";
                err.classList.remove('hidden');
                btn.disabled = false; btn.innerText = "SE CONNECTER";
            }
        };

        window.handleLogout = async () => { await signOut(auth); location.reload(); };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                if (assignRole(user.email)) {
                    document.getElementById('authSection').classList.add('hidden');
                    document.getElementById('appContent').classList.remove('hidden');
                    document.getElementById('userDisplayEmail').innerText = user.email;
                    startListeners();
                    prepareNextId();
                } else {
                    signOut(auth);
                    document.getElementById('loginError').innerText = "Utilisateur non autorisé.";
                    document.getElementById('loginError').classList.remove('hidden');
                }
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
            else return false;

            document.querySelectorAll('.role-section').forEach(s => s.classList.add('hidden'));
            const colors = { admin: 'bg-slate-900', dispatch: 'bg-[#3A75C4]', relais: 'bg-[#009E60]', livreur: 'bg-[#FCD116]' };
            const text = { admin: 'text-white', dispatch: 'text-white', relais: 'text-white', livreur: 'text-black' };

            document.getElementById('badgeDisplay').innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-black uppercase ${text[userRole]} ${colors[userRole]} shadow-sm">${userRole}</span>`;

            if (userRole === 'admin') {
                document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
                document.getElementById('adminControls').classList.remove('hidden');
            }
            else if (userRole === 'relais') { document.getElementById('rubrique1').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'dispatch') { document.getElementById('rubrique2').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'livreur') { document.getElementById('rubrique3').classList.remove('hidden'); document.getElementById('statLivreurSection').classList.remove('hidden'); }
            return true;
        };

        window.deleteMission = async (id) => {
            if (!confirm("⚠️ Action irréversible. Supprimer cette mission ?")) return;
            try {
                await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id));
                showToast("Suppression réussie", "success");
            } catch (e) { showToast("Erreur suppression", "error"); }
        };

        window.resetAccounting = async () => {
            const delivered = allMissions.filter(m => m.s === 2);
            if (delivered.length === 0 || !confirm(`Vider les ${delivered.length} archives ?`)) return;
            try {
                for (const m of delivered) await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', m.id));
                showToast("Comptabilité remise à zéro", "success");
            } catch (e) { showToast("Erreur RAZ", "error"); }
        };

        const startListeners = () => {
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'missions'), (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
                renderStats();
            });
        };

        const prepareNextId = () => {
            window.nextId = "GA-" + Math.floor(Math.random() * 900000 + 100000);
            const el = document.getElementById('displayNextId');
            if(el) el.innerText = window.nextId;
        };

        window.genererMission = async () => {
            const fields = {
                en: document.getElementById('expNom').value,
                eq: document.getElementById('expQuartier').value,
                et: document.getElementById('expTel').value,
                dn: document.getElementById('destNom').value,
                dq: document.getElementById('dq').value,
                dt: document.getElementById('dt').value,
                p: parseFloat(document.getElementById('fraisLivraison').value) || 0,
            };
            if (!fields.en || !fields.dt || !fields.p) return showToast("Informations manquantes", "error");

            try {
                const now = Date.now();
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId), {
                    ...fields, id: window.nextId, s: 0, ca: now, cad: new Date(now).toISOString().split('T')[0], ce: currentUser.email
                });
                showToast("Mission créée avec succès", "success");
                prepareNextId();
                ['destNom', 'dq', 'dt', 'fraisLivraison'].forEach(id => document.getElementById(id).value = "");
            } catch (e) { showToast("Erreur Firestore", "error"); }
        };

        window.publierMission = async (id) => {
            try {
                await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { s: 1 });
                showToast("Colis prêt pour livraison", "success");
            } catch (e) { showToast("Erreur", "error"); }
        };

        window.triggerPhoto = (id) => {
            currentMissionForPhoto = id;
            document.getElementById('cameraModal').classList.remove('hidden');
        };

        window.processImage = async (file) => {
            if (!file) return;
            const reader = new FileReader();
            reader.onload = async (e) => {
                const base64 = e.target.result;
                try {
                    await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionForPhoto), {
                        s: 2, le: currentUser.email, lk: new Date().toISOString().split('T')[0], proof: base64
                    });
                    document.getElementById('cameraModal').classList.add('hidden');
                    showToast("Livré & Encaissé 🇬🇦", "success");
                } catch (err) { showToast("Erreur validation", "error"); }
            };
            reader.readAsDataURL(file);
        };

        window.openQrScanner = () => {
            document.getElementById('qrScannerModal').classList.remove('hidden');
            html5QrCode = new Html5Qrcode("reader");
            html5QrCode.start({ facingMode: "environment" }, { fps: 10, qrbox: 250 }, (text) => {
                const id = text.replace('CT241-', '');
                const m = allMissions.find(x => x.id === id);
                if(m && m.s === 1) {
                    html5QrCode.stop().then(() => {
                        html5QrCode = null;
                        document.getElementById('qrScannerModal').classList.add('hidden');
                        triggerPhoto(id);
                    });
                } else showToast("Code non valide ou mission inactive", "error");
            }).catch(e => console.error(e));
        };

        window.renderUI = () => {
            const contDisp = document.getElementById('containerDispatch');
            const contLiv = document.getElementById('containerLivreur');
            const contArch = document.getElementById('archiveBody');
            if(contDisp) contDisp.innerHTML = "";
            if(contLiv) contLiv.innerHTML = "";
            if(contArch) contArch.innerHTML = "";

            allMissions.sort((a,b) => b.ca - a.ca).forEach(m => {
                const isAdmin = userRole === 'admin';

                if (m.s === 0 && (isAdmin || userRole === 'dispatch')) {
                    if(contDisp) contDisp.innerHTML += `
                        <div class="p-5 bg-white border-l-8 border-[#009E60] rounded-3xl mb-4 flex justify-between items-center shadow-md">
                            <div>
                                <div class="flex items-center gap-2">
                                    <p class="text-[9px] font-black text-[#009E60] uppercase">ID: ${m.id}</p>
                                    ${isAdmin ? `<button onclick="deleteMission('${m.id}')" class="text-red-400 hover:text-red-600">✕</button>` : ''}
                                </div>
                                <p class="text-[12px] font-black text-slate-800">${m.dn}</p>
                                <p class="text-[10px] text-slate-400">Quartier: ${m.dq}</p>
                            </div>
                            <div class="flex gap-2">
                                <button onclick="openBonImpression('${m.id}')" class="bg-slate-100 p-3 rounded-2xl text-slate-400 hover:text-[#3A75C4]"><svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M17 17h2a2 2 0 002-2v-4a2 2 0 00-2-2H5a2 2 0 00-2 2v4a2 2 0 00-2 2h2m2 4h6a2 2 0 002-2v-4a2 2 0 00-2-2H9a2 2 0 00-2 2v4a2 2 0 002 2zm8-12V5a2 2 0 00-2-2H9a2 2 0 00-2 2v4h10z" stroke-width="2"/></svg></button>
                                <button onclick="publierMission('${m.id}')" class="bg-[#009E60] text-white px-5 py-3 rounded-2xl text-[10px] font-black uppercase tracking-widest shadow-md">Lancer</button>
                            </div>
                        </div>`;
                } 
                else if (m.s === 1 && (isAdmin || userRole === 'livreur')) {
                    if(contLiv) contLiv.innerHTML += `
                        <div class="p-6 bg-white border-l-8 border-[#FCD116] rounded-[2.5rem] mb-5 shadow-lg relative overflow-hidden">
                            ${isAdmin ? `<button onclick="deleteMission('${m.id}')" class="absolute top-4 right-4 text-red-300">✕</button>` : ''}
                            <div class="flex justify-between items-start mb-4">
                                <div>
                                    <span class="text-[9px] font-black text-slate-400 uppercase">#${m.id}</span>
                                    <h4 class="font-black text-slate-900 text-lg">${m.dn}</h4>
                                </div>
                                <div class="bg-[#FCD116] text-black px-3 py-1 rounded-full text-[8px] font-black uppercase">EN COURS</div>
                            </div>
                            <div class="space-y-2 mb-6 text-[12px]">
                                <p class="font-bold text-slate-600 flex items-center gap-2">📍 ${m.dq}</p>
                                <a href="tel:${m.dt}" class="text-[#3A75C4] font-black flex items-center gap-2">📞 Appeler: ${m.dt}</a>
                            </div>
                            <div class="grid grid-cols-2 gap-3">
                                <button onclick="window.open('https://www.google.com/maps/dir/?api=1&destination=${encodeURIComponent(m.dq + ', Libreville')}')" class="bg-slate-50 text-slate-900 py-4 rounded-2xl font-black text-[10px] uppercase">Navigation</button>
                                <button onclick="openQrScanner()" class="bg-slate-900 text-white py-4 rounded-2xl font-black text-[10px] uppercase shadow-xl">Livrer</button>
                            </div>
                        </div>`;
                } 
                else if (m.s === 2 && (isAdmin || userRole === 'dispatch' || userRole === 'relais')) {
                    if(contArch) contArch.innerHTML += `
                        <tr class="border-b border-white/5 text-[10px]">
                            <td class="p-4 font-bold text-white uppercase">${m.id}</td>
                            <td class="p-4 text-slate-300">${m.dn}<br><span class="text-[8px] text-[#FCD116] uppercase font-black">${m.le ? m.le.split('@')[0] : '...'}</span></td>
                            <td class="p-4 text-right">
                                <div class="flex items-center justify-end gap-3 font-black text-[#009E60]">
                                    ${m.p.toLocaleString()} CFA
                                    ${isAdmin ? `<button onclick="deleteMission('${m.id}')" class="text-red-500 ml-2">✕</button>` : ''}
                                </div>
                            </td>
                        </tr>`;
                }
            });
        };

        window.openBonImpression = (id) => {
            const m = allMissions.find(x => x.id === id);
            if(!m) return;
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area p-10 bg-white text-slate-900">
                    <div class="flex justify-between items-center border-b-4 border-[#009E60] pb-6 mb-8">
                        <div>
                            <h2 class="font-black text-lg tracking-tight">CT241 LOGISTIQUE</h2>
                            <p class="text-[8px] font-bold text-[#009E60] uppercase">République Gabonaise</p>
                        </div>
                        <h2 class="font-black text-[#3A75C4] text-3xl">#${m.id}</h2>
                    </div>
                    <div class="grid grid-cols-2 gap-8 mb-10 text-[11px]">
                        <div><p class="font-black text-slate-400 uppercase text-[8px] mb-1">Expéditeur</p><p class="font-black">${m.en}</p><p class="text-slate-500">${m.eq}</p></div>
                        <div><p class="font-black text-slate-400 uppercase text-[8px] mb-1">Destinataire</p><p class="font-black">${m.dn}</p><p class="text-slate-500">${m.dq}</p></div>
                    </div>
                    <div id="qrcode" class="flex justify-center my-8 p-6 bg-slate-50 rounded-[2.5rem] w-fit mx-auto border-2 border-slate-100"></div>
                    <div class="text-center pt-6 border-t border-slate-100">
                        <p class="text-[9px] font-black text-slate-400 uppercase mb-1">Total à payer</p>
                        <p class="text-4xl font-black text-[#009E60]">${m.p.toLocaleString()} CFA</p>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
            setTimeout(() => { new QRCode(document.getElementById("qrcode"), { text: "CT241-"+m.id, width: 140, height: 140 }); }, 50);
        };

        const renderStats = () => {
            if(!currentUser) return;
            let count = 0, bonus = 0;
            allMissions.forEach(m => { if(m.s === 2 && m.le === currentUser.email) { count++; if(count > 17) bonus += 700; } });
            if(document.getElementById('myCount')) document.getElementById('myCount').innerText = count;
            if(document.getElementById('myBonus')) document.getElementById('myBonus').innerText = bonus + ' CFA';
        };

        window.showToast = (m, t) => {
            const el = document.getElementById('toast');
            el.innerText = m;
            el.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-8 py-4 rounded-3xl text-white font-black text-[10px] z-[2000] shadow-2xl transition-all ${t==='success'?'bg-[#009E60]':'bg-red-600'}`;
            el.classList.remove('hidden');
            setTimeout(() => el.classList.add('hidden'), 3000);
        };
    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; overflow-x: hidden; }
        .hidden { display: none; }
        .gabon-line { background: linear-gradient(to right, #009E60 33.33%, #FCD116 33.33% 66.66%, #3A75C4 66.66%); height: 5px; }
        @media print { .no-print { display: none; } .print-area { display: block; position: fixed; top: 0; left: 0; width: 100%; height: 100%; z-index: 9999; } }
    </style>
</head>
<body class="antialiased select-none">

    <div id="toast" class="hidden"></div>

    <!-- CONNEXION GABON -->
    <section id="authSection" class="fixed inset-0 bg-[#f8fafc] flex items-center justify-center p-8 z-[1000]">
        <div class="absolute inset-x-0 top-0 gabon-line"></div>
        <div class="w-full max-w-sm space-y-12">
            <div class="text-center space-y-4">
                <div class="w-20 h-20 bg-[#FCD116] mx-auto rounded-[2rem] flex items-center justify-center text-4xl shadow-xl shadow-yellow-200">🇬🇦</div>
                <h1 class="text-3xl font-black tracking-tighter uppercase">CT241 <span class="text-[#3A75C4]">Log</span></h1>
                <p class="text-[9px] text-[#009E60] font-black tracking-[0.3em] uppercase">Efficacité Logistique Gabon</p>
            </div>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" required class="w-full p-6 bg-white rounded-3xl text-xs font-bold shadow-sm outline-none focus:ring-4 ring-emerald-50 border border-slate-100" placeholder="Email professionnel">
                <input type="password" id="loginPass" required class="w-full p-6 bg-white rounded-3xl text-xs font-bold shadow-sm outline-none focus:ring-4 ring-emerald-50 border border-slate-100" placeholder="Code secret">
                <p id="loginError" class="text-[10px] text-red-500 font-black hidden text-center"></p>
                <button id="loginBtn" type="submit" class="w-full bg-[#009E60] text-white font-black py-7 rounded-[2rem] text-xs uppercase tracking-widest shadow-xl active:scale-95 transition-all">Accéder au système</button>
            </form>
        </div>
    </section>

    <!-- APP CONTENT -->
    <main id="appContent" class="hidden min-h-screen pb-32">
        <nav class="bg-white/80 backdrop-blur-xl sticky top-0 z-[100] border-b border-slate-100 p-5">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <div class="w-10 h-10 bg-[#009E60] rounded-xl flex items-center justify-center text-white font-black text-sm shadow-lg shadow-emerald-100">G</div>
                    <div>
                        <p id="userDisplayEmail" class="text-[9px] font-black text-slate-300 uppercase truncate w-32 tracking-tighter">...</p>
                        <div id="badgeDisplay"></div>
                    </div>
                </div>
                <button onclick="handleLogout()" class="w-10 h-10 flex items-center justify-center bg-red-50 rounded-xl text-red-400">
                    <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M17 16l4-4m0 0l-4-4m4 4H7" stroke-width="2.5"/></svg>
                </button>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-6 space-y-8">
            <!-- ADMIN TOOLS -->
            <section id="adminControls" class="hidden bg-slate-900 p-8 rounded-[3rem] shadow-xl space-y-6">
                <h3 class="text-white/40 font-black text-[9px] uppercase tracking-[0.4em]">Administration Centrale</h3>
                <div class="grid grid-cols-2 gap-4">
                    <button onclick="resetAccounting()" class="bg-red-500/10 border border-red-500/20 text-red-500 p-6 rounded-[2rem] text-[9px] font-black uppercase">Vider Archives</button>
                    <button class="bg-[#3A75C4] text-white p-6 rounded-[2rem] text-[9px] font-black uppercase">Rapport Global</button>
                </div>
            </section>

            <!-- STATS -->
            <section id="statLivreurSection" class="role-section hidden grid grid-cols-2 gap-4">
                <div class="bg-white p-6 rounded-[2.5rem] text-center shadow-sm border border-emerald-50">
                    <p class="text-[8px] font-black text-slate-300 uppercase mb-1">Colis Livrés</p>
                    <p id="myCount" class="text-3xl font-black text-slate-800">0</p>
                </div>
                <div class="bg-white p-6 rounded-[2.5rem] text-center shadow-sm border border-yellow-50">
                    <p class="text-[8px] font-black text-slate-300 uppercase mb-1">Prime Cumulée</p>
                    <p id="myBonus" class="text-lg font-black text-[#009E60]">0 CFA</p>
                </div>
            </section>

            <!-- CRÉATION -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[3.5rem] p-10 space-y-8 shadow-xl border border-emerald-50">
                    <div class="flex justify-between items-center">
                        <h2 class="font-black text-[11px] uppercase tracking-[0.2em] text-[#009E60]">Nouveau Colis</h2>
                        <span id="displayNextId" class="bg-[#FCD116] text-black px-4 py-2 rounded-full font-black text-[10px] shadow-sm">...</span>
                    </div>
                    
                    <div class="space-y-4">
                        <input type="text" id="expNom" placeholder="Relais d'origine (Ex: Boutique Libreville)" class="w-full p-6 bg-slate-50 rounded-3xl text-xs font-bold outline-none focus:ring-4 ring-emerald-50">
                        <div class="grid grid-cols-2 gap-4 text-xs">
                            <input type="text" id="expQuartier" placeholder="Quartier Exp." class="p-6 bg-slate-50 rounded-3xl font-bold">
                            <input type="tel" id="expTel" placeholder="Tél Exp." class="p-6 bg-slate-50 rounded-3xl font-bold">
                        </div>
                    </div>

                    <div class="space-y-4">
                        <input type="text" id="destNom" placeholder="Nom du Client" class="w-full p-6 bg-slate-50 rounded-3xl text-xs font-bold outline-none border-2 border-transparent focus:border-[#FCD116]">
                        <div class="grid grid-cols-2 gap-4 text-xs">
                            <input type="text" id="dq" placeholder="Lieu Livraison" class="p-6 bg-slate-50 rounded-3xl font-bold">
                            <input type="tel" id="dt" placeholder="Tél Client" class="p-6 bg-slate-50 rounded-3xl font-bold">
                        </div>
                    </div>

                    <div class="space-y-3">
                        <label class="text-[9px] font-black uppercase text-[#3A75C4] ml-6 tracking-widest">Montant à encaisser (CFA)</label>
                        <input type="number" id="fraisLivraison" placeholder="0" class="w-full p-8 bg-[#3A75C4] text-white rounded-[2.5rem] text-center text-4xl font-black shadow-xl shadow-blue-100 outline-none">
                    </div>

                    <button onclick="genererMission()" class="w-full bg-[#009E60] text-white font-black py-7 rounded-[2.5rem] text-xs uppercase tracking-widest shadow-xl active:scale-95 transition-all">Générer & Imprimer</button>
                </div>
            </section>

            <!-- LISTES -->
            <section id="rubrique2" class="role-section hidden space-y-4">
                <h3 class="text-[9px] font-black text-slate-300 uppercase tracking-widest ml-6">Flux Sortants</h3>
                <div id="containerDispatch"></div>
            </section>

            <section id="rubrique3" class="role-section hidden space-y-4">
                <h3 class="text-[9px] font-black text-[#3A75C4] uppercase tracking-widest ml-6">Missions Actives</h3>
                <div id="containerLivreur"></div>
            </section>

            <section id="rubrique4" class="role-section hidden">
                <div class="bg-slate-900 rounded-[3.5rem] p-10 shadow-2xl border-b-8 border-[#009E60]">
                    <h3 class="text-white/30 font-black text-[10px] uppercase mb-8 tracking-[0.3em]">Registre des livraisons</h3>
                    <div class="overflow-x-auto"><table class="w-full"><tbody id="archiveBody"></tbody></table></div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODALS -->
    <div id="cameraModal" class="fixed inset-0 bg-black/95 z-[2000] hidden flex flex-col items-center justify-center p-10 text-white">
        <div class="text-center space-y-10 max-w-xs">
            <div class="w-24 h-24 bg-[#009E60] rounded-full flex items-center justify-center text-5xl mx-auto shadow-2xl">📸</div>
            <h2 class="text-3xl font-black tracking-tight uppercase">Validation</h2>
            <p class="text-xs text-slate-500 font-bold uppercase tracking-widest leading-relaxed">Photographiez le colis pour valider la fin de mission.</p>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-7 rounded-[2.5rem] uppercase text-[10px] tracking-widest">Activer Caméra</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-red-500 font-bold text-xs uppercase">Annuler</button>
        </div>
    </div>

    <div id="qrScannerModal" class="fixed inset-0 bg-black/98 z-[1500] hidden flex flex-col items-center justify-center p-8">
        <div class="w-full max-w-sm space-y-8 text-center">
            <h3 class="text-white/40 font-black text-[10px] uppercase tracking-widest">Scanner de Bordereau</h3>
            <div id="reader" class="rounded-[4rem] overflow-hidden bg-slate-900 border-4 border-white/5"></div>
            <button onclick="document.getElementById('qrScannerModal').classList.add('hidden'); if(html5QrCode) html5QrCode.stop();" class="w-full bg-red-600 text-white py-6 rounded-[2.5rem] font-black text-xs uppercase">Annuler</button>
        </div>
    </div>

    <div id="detailModal" class="fixed inset-0 bg-slate-950/90 z-[1200] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-md bg-white rounded-[4rem] overflow-hidden shadow-2xl">
            <div id="detailContent"></div>
            <div class="p-8 bg-slate-50 flex gap-4 no-print border-t border-slate-100">
                <button onclick="window.print()" class="flex-[2] bg-slate-900 text-white font-black py-6 rounded-3xl text-xs uppercase tracking-widest">Imprimer</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="flex-1 bg-white border border-slate-200 text-slate-400 font-black py-6 rounded-3xl text-xs uppercase">X</button>
            </div>
        </div>
    </div>

</body>
</html>
