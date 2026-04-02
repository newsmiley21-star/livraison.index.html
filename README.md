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
    
    <!-- Scripts Externes : UI & QR -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
    <script src="https://unpkg.com/html5-qrcode"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, deleteDoc, onSnapshot, query, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

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
        let currentMissionId = null;
        let searchQuery = "";
        let filterToday = false;

        // --- AUTH & ROLES ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value.trim();
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            const errorEl = document.getElementById('loginError');
            
            btn.disabled = true;
            btn.innerText = "Vérification...";
            errorEl.classList.add('hidden');

            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } 
            catch (error) { 
                console.error("Erreur login:", error.code);
                errorEl.innerText = "Email ou mot de passe incorrect.";
                errorEl.classList.remove('hidden');
                btn.disabled = false; 
                btn.innerText = "ACCÉDER AU PORTAIL";
            }
        };

        window.handleLogout = async () => { await signOut(auth); location.reload(); };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                const success = assignRole(user.email);
                if (success) {
                    document.getElementById('authSection').classList.add('hidden');
                    document.getElementById('appContent').classList.remove('hidden');
                    document.getElementById('userDisplayEmail').innerText = user.email;
                    startListeners();
                    prepareNextId();
                } else {
                    // Si l'utilisateur est authentifié mais n'a pas de rôle reconnu
                    signOut(auth);
                    const errorEl = document.getElementById('loginError');
                    errorEl.innerText = "Accès refusé : rôle non reconnu pour cet e-mail.";
                    errorEl.classList.remove('hidden');
                    document.getElementById('authSection').classList.remove('hidden');
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
            else return false; // Rôle non trouvé
            
            // Réinitialisation de l'affichage des sections
            document.querySelectorAll('.role-section').forEach(s => s.classList.add('hidden'));
            const badge = document.getElementById('badgeDisplay');
            const colors = { admin: 'bg-red-600', dispatch: 'bg-blue-600', relais: 'bg-emerald-600', livreur: 'bg-amber-600' };
            badge.innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-black uppercase text-white ${colors[userRole] || 'bg-slate-500'}">${userRole}</span>`;

            // Affichage selon le rôle
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
                document.getElementById('statLivreurSection').classList.remove('hidden'); 
            }
            return true;
        };

        // --- MISSIONS LOGIQUE ---
        const prepareNextId = () => {
            window.nextId = "2026-" + Math.floor(Math.random() * 900000 + 100000);
            const el = document.getElementById('displayNextId');
            if(el) el.innerText = window.nextId;
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
            if (!fields.en || !fields.dt || !fields.p) return showToast("Champs obligatoires !", "error");

            try {
                const now = Date.now();
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId), {
                    ...fields, id: window.nextId, s: 0, ca: now, cad: new Date(now).toISOString().split('T')[0], ce: currentUser.email
                });
                showToast("Mission créée !", "success");
                prepareNextId();
                renderUI();
            } catch (e) { showToast("Erreur création", "error"); }
        };

        const startListeners = () => {
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'missions'), (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
                renderStats();
            }, (err) => console.error("Erreur Firestore:", err));
        };

        const renderStats = () => {
            const stats = { caTotal: 0, caJour: 0, livraisonsTotal: 0, livreurs: {} };
            const today = new Date().toISOString().split('T')[0];

            allMissions.forEach(m => {
                if (m.s === 2) {
                    stats.caTotal += m.p || 0;
                    stats.livraisonsTotal++;
                    if (m.lk === today) stats.caJour += m.p || 0;
                    const l = m.le || 'Inconnu';
                    if (!stats.livreurs[l]) stats.livreurs[l] = { count: 0, bonus: 0 };
                    stats.livreurs[l].count++;
                    if (stats.livreurs[l].count > 17) stats.livreurs[l].bonus += 700;
                }
            });

            if (userRole === 'admin') {
                const caT = document.getElementById('dashCaTotal');
                const caJ = document.getElementById('dashCaJour');
                if(caT) caT.innerText = stats.caTotal.toLocaleString() + ' CFA';
                if(caJ) caJ.innerText = stats.caJour.toLocaleString() + ' CFA';
            }
            if (userRole === 'livreur') {
                const my = stats.livreurs[currentUser.email] || { count: 0, bonus: 0 };
                const mC = document.getElementById('myCount');
                const mB = document.getElementById('myBonus');
                if(mC) mC.innerText = my.count;
                if(mB) mB.innerText = my.bonus + ' CFA';
            }
        };

        window.renderUI = () => {
            const contDisp = document.getElementById('containerDispatch');
            const contLiv = document.getElementById('containerLivreur');
            const contArch = document.getElementById('archiveBody');
            if(contDisp) contDisp.innerHTML = "";
            if(contLiv) contLiv.innerHTML = "";
            if(contArch) contArch.innerHTML = "";

            allMissions.forEach(m => {
                if (m.s === 0 && (userRole === 'admin' || userRole === 'dispatch')) {
                    if(contDisp) contDisp.innerHTML += `
                        <div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 flex justify-between items-center">
                            <div class="text-[11px] font-bold"><span>${m.id}</span><br>${m.dn} (${m.dq})</div>
                            <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white px-4 py-2 rounded-xl text-[10px] font-black">EXPÉDIER</button>
                        </div>`;
                } else if (m.s === 1 && (userRole === 'admin' || userRole === 'livreur')) {
                    if(contLiv) contLiv.innerHTML += `
                        <div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4">
                            <div class="flex justify-between font-black mb-2"><span>${m.id}</span><span class="text-amber-600 text-[9px]">EN COURS</span></div>
                            <p class="text-[11px] mb-4">${m.dn} - ${m.dq}<br>Tél: ${m.dt}</p>
                            <button onclick="openQrScanner()" class="w-full bg-slate-900 text-white py-4 rounded-2xl font-black text-[10px]">SCANNER QR POUR LIVRER</button>
                        </div>`;
                } else if (m.s === 2 && (userRole === 'admin' || userRole === 'dispatch')) {
                    if(contArch) contArch.innerHTML += `<tr class="border-b border-slate-800 text-[10px]"><td class="p-4 font-bold text-white">${m.id}</td><td class="p-4 text-slate-300">${m.dn}</td><td class="p-4 text-right text-emerald-400 font-black">${m.p}</td></tr>`;
                }
            });
        };

        window.publierMission = async (id) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { s: 1 });
            showToast("Mission envoyée aux livreurs", "success");
        };

        // --- QR SCANNER ---
        let html5QrCode = null;
        window.openQrScanner = () => {
            document.getElementById('qrScannerModal').classList.remove('hidden');
            html5QrCode = new Html5Qrcode("reader");
            html5QrCode.start({ facingMode: "environment" }, { fps: 10, qrbox: 250 }, (text) => {
                const missionId = text.replace('CT241-', '');
                validateMission(missionId);
            }).catch(e => console.error(e));
        };

        const validateMission = async (id) => {
            const m = allMissions.find(x => x.id === id);
            if(m && m.s === 1) {
                await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), {
                    s: 2, le: currentUser.email, lk: new Date().toISOString().split('T')[0]
                });
                if(html5QrCode) html5QrCode.stop();
                document.getElementById('qrScannerModal').classList.add('hidden');
                showToast("Livraison validée !", "success");
            } else {
                showToast("QR Invalide ou déjà livré", "error");
            }
        };

        window.showToast = (m, t) => {
            const el = document.getElementById('toast');
            el.innerText = m;
            el.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-xs z-[600] ${t==='success'?'bg-emerald-600':'bg-red-600'}`;
            el.classList.remove('hidden');
            setTimeout(() => el.classList.add('hidden'), 3000);
        };

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; }
        .hidden { display: none; }
    </style>
</head>
<body class="min-h-screen">

    <div id="toast" class="hidden"></div>

    <!-- AUTHENTICATION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[3rem] p-10 space-y-8 shadow-2xl text-center">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-20 mx-auto">
            <h1 class="text-xl font-black text-slate-900 uppercase">Connexion CT241</h1>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="E-mail (ex: livreur1@ct241.com)">
                <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="Mot de passe">
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden"></p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl hover:bg-black transition-colors">SE CONNECTER</button>
            </form>
            <p class="text-[9px] text-slate-400 uppercase font-bold">L'e-mail doit contenir "livreur", "admin", "relais" ou "dispatch".</p>
        </div>
    </section>

    <!-- APP CONTENT -->
    <main id="appContent" class="hidden pb-24">
        <nav class="bg-white p-4 sticky top-0 z-50 border-b border-slate-100">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <div class="flex flex-col"><span class="text-[9px] font-black text-slate-900" id="userDisplayEmail">...</span><div id="badgeDisplay"></div></div>
                </div>
                <button onclick="handleLogout()" class="p-2 bg-slate-100 rounded-full text-slate-400 hover:text-red-500"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m0 0l-4-4m4 4H7" /></svg></button>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            <!-- Stats Livreur -->
            <section id="statLivreurSection" class="role-section hidden grid grid-cols-2 gap-3">
                <div class="bg-white p-4 rounded-3xl border border-slate-100"><span class="text-[8px] font-black text-slate-400 uppercase">Livraisons</span><div id="myCount" class="text-xl font-black">0</div></div>
                <div class="bg-white p-4 rounded-3xl border border-slate-100"><span class="text-[8px] font-black text-slate-400 uppercase">Bonus</span><div id="myBonus" class="text-xl font-black text-emerald-600">0</div></div>
            </section>

            <!-- Création (Relais / Admin) -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl p-6 space-y-4 border border-slate-100">
                    <h2 class="font-black text-sm uppercase">Nouveau Bordereau <span id="displayNextId" class="text-blue-600">...</span></h2>
                    <input type="text" id="expNom" placeholder="Nom Boutique" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                    <div class="grid grid-cols-2 gap-3">
                        <input type="text" id="expQuartier" placeholder="Quartier" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        <input type="tel" id="expTel" placeholder="Tél" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                    </div>
                    <hr class="border-slate-50">
                    <input type="text" id="destNom" placeholder="Destinataire" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                    <div class="grid grid-cols-2 gap-3">
                        <input type="text" id="dq" placeholder="Quartier Destination" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        <input type="tel" id="dt" placeholder="Tél Client" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                    </div>
                    <div class="grid grid-cols-2 gap-3">
                        <select id="natureColis" class="p-4 bg-slate-900 text-white rounded-2xl font-black text-[10px]"><option value="0">📦 Standard</option></select>
                        <input type="number" id="fraisLivraison" placeholder="PRIX LIVRAISON" class="p-4 bg-blue-600 text-white rounded-2xl font-black text-[11px] outline-none placeholder:text-blue-200">
                    </div>
                    <button onclick="genererMission()" class="w-full bg-slate-900 text-white font-black py-5 rounded-3xl text-[11px] uppercase tracking-widest">Enregistrer</button>
                </div>
            </section>

            <section id="rubrique2" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase mb-4">Dispatching</h3>
                <div id="containerDispatch"></div>
            </section>

            <section id="rubrique3" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase mb-4">Mes Livraisons</h3>
                <div id="containerLivreur"></div>
            </section>
        </div>
    </main>

    <!-- SCANNER MODAL -->
    <div id="qrScannerModal" class="fixed inset-0 bg-black z-[500] hidden flex flex-col items-center justify-center p-6 text-white">
        <h2 class="text-xl font-black mb-4 uppercase">Scanner le bordereau</h2>
        <div id="reader" class="w-full max-w-sm rounded-3xl overflow-hidden border-2 border-white/20"></div>
        <button onclick="document.getElementById('qrScannerModal').classList.add('hidden')" class="mt-8 bg-red-600 px-10 py-4 rounded-2xl font-black text-[10px]">FERMER</button>
    </div>

</body>
</html>
