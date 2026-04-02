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
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">

    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
    <script src="https://unpkg.com/html5-qrcode"></script>
    
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
        const appId = 'ct241-service-de-livraison';

        // Fonctions de chemin centralisées pour éviter les erreurs
        const getMissionDoc = (id) => doc(db, 'artifacts', appId, 'public', 'data', 'missions', id);
        const getMissionsColl = () => collection(db, 'artifacts', appId, 'public', 'data', 'missions');

        let currentUser = null;
        let userRole = null;
        let allMissions = [];

        // --- AUTHENTIFICATION ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value.trim().toLowerCase();
            const pass = document.getElementById('loginPass').value;
            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } catch (error) { 
                const errEl = document.getElementById('loginError');
                errEl.innerText = "Accès refusé. Vérifiez vos identifiants.";
                errEl.classList.remove('hidden');
            }
        };

        window.handleLogout = async () => { await signOut(auth); location.reload(); };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                const email = user.email.toLowerCase();
                if (email.includes('admin')) userRole = 'admin';
                else if (email.includes('relais')) userRole = 'relais';
                else if (email.includes('dispatch')) userRole = 'dispatch';
                else userRole = 'livreur';

                document.getElementById('authSection').classList.add('hidden');
                document.getElementById('appContent').classList.remove('hidden');
                document.getElementById('userMail').innerText = email;
                
                setupRoleUI();
                startListeners();
                generateNewID();
            } else {
                document.getElementById('authSection').classList.remove('hidden');
                document.getElementById('appContent').classList.add('hidden');
            }
        });

        const setupRoleUI = () => {
            document.querySelectorAll('.role-view').forEach(el => el.classList.add('hidden'));
            if (userRole === 'admin') {
                document.querySelectorAll('.role-view').forEach(el => el.classList.remove('hidden'));
                document.getElementById('adminPanel').classList.remove('hidden');
            } else if (userRole === 'relais') {
                document.getElementById('createSection').classList.remove('hidden');
                document.getElementById('archiveSection').classList.remove('hidden');
            } else if (userRole === 'dispatch') {
                document.getElementById('dispatchSection').classList.remove('hidden');
                document.getElementById('archiveSection').classList.remove('hidden');
            } else {
                document.getElementById('livreurSection').classList.remove('hidden');
            }
        };

        // --- ACTIONS CRUD ---
        window.deleteMission = async (id) => {
            if (!confirm(`Supprimer définitivement la mission #${id} ?`)) return;
            try {
                // Utilisation de la référence exacte
                await deleteDoc(getMissionDoc(id));
                showToast("Mission supprimée", "success");
            } catch (e) {
                showToast("Erreur de suppression", "error");
                console.error(e);
            }
        };

        window.generateNewID = () => {
            window.currentMissionId = "241-" + Math.floor(100000 + Math.random() * 900000);
            document.getElementById('nextIdDisplay').innerText = window.currentMissionId;
        };

        window.saveMission = async () => {
            const data = {
                id: window.currentMissionId,
                en: document.getElementById('expNom').value,
                eq: document.getElementById('expQuartier').value,
                et: document.getElementById('expTel').value,
                dn: document.getElementById('destNom').value,
                dq: document.getElementById('dq').value,
                dt: document.getElementById('dt').value,
                p: parseFloat(document.getElementById('price').value) || 0,
                s: 0, // 0: Attente, 1: En cours, 2: Livré
                ts: Date.now(),
                date: new Date().toLocaleDateString('fr-FR'),
                author: currentUser.email
            };

            if (!data.en || !data.dn || !data.dt) return showToast("Veuillez remplir les champs obligatoires", "error");

            try {
                // On force l'ID du document pour qu'il soit identique à l'ID métier
                await setDoc(getMissionDoc(data.id), data);
                showToast("Bordereau créé", "success");
                generateNewID();
                document.getElementById('destNom').value = "";
                document.getElementById('dt').value = "";
            } catch (e) {
                showToast("Erreur d'enregistrement", "error");
            }
        };

        // --- ECOUTE TEMPS RÉEL ---
        const startListeners = () => {
            onSnapshot(getMissionsColl(), (snap) => {
                allMissions = [];
                snap.forEach(d => allMissions.push(d.data()));
                refreshTables();
            });
        };

        const refreshTables = () => {
            const disp = document.getElementById('dispatchList');
            const liv = document.getElementById('livreurList');
            const arch = document.getElementById('archiveList');
            
            [disp, liv, arch].forEach(el => if(el) el.innerHTML = "");

            allMissions.sort((a,b) => b.ts - a.ts).forEach(m => {
                const isAdmin = userRole === 'admin';
                
                if (m.s === 0 && (isAdmin || userRole === 'dispatch')) {
                    disp.innerHTML += `
                        <div class="bg-white p-5 rounded-[2rem] shadow-sm border border-slate-100 flex justify-between items-center mb-3">
                            <div>
                                <p class="text-[10px] font-black text-blue-500">#${m.id}</p>
                                <p class="text-sm font-bold text-slate-800">${m.dn}</p>
                            </div>
                            <div class="flex gap-2">
                                ${isAdmin ? `<button onclick="deleteMission('${m.id}')" class="w-10 h-10 flex items-center justify-center text-red-400 hover:bg-red-50 rounded-full transition-colors">✕</button>` : ''}
                                <button onclick="updateDoc(getMissionDoc('${m.id}'), {s:1})" class="bg-blue-600 text-white px-5 py-3 rounded-xl text-[10px] font-black uppercase tracking-wider shadow-lg shadow-blue-200">Assigner</button>
                            </div>
                        </div>`;
                } else if (m.s === 1 && (isAdmin || userRole === 'livreur')) {
                    liv.innerHTML += `
                        <div class="bg-white p-6 rounded-[2.5rem] shadow-xl border-l-8 border-amber-400 mb-4 relative">
                             ${isAdmin ? `<button onclick="deleteMission('${m.id}')" class="absolute top-4 right-4 text-slate-300">✕</button>` : ''}
                            <h4 class="font-black text-lg text-slate-900">${m.dn}</h4>
                            <p class="text-xs text-slate-500 mb-4 font-bold uppercase">${m.dq} • ${m.dt}</p>
                            <button onclick="updateDoc(getMissionDoc('${m.id}'), {s:2, deliveredBy: currentUser.email, deliveryDate: new Date().toLocaleDateString('fr-FR')})" 
                                    class="w-full bg-slate-900 text-white py-4 rounded-2xl font-black text-[11px] uppercase tracking-widest hover:bg-black transition-all">Valider la Livraison</button>
                        </div>`;
                } else if (m.s === 2) {
                    arch.innerHTML += `
                        <tr class="border-b border-slate-800/50 text-[11px]">
                            <td class="py-4 text-white font-bold">#${m.id}</td>
                            <td class="py-4 text-slate-400 font-medium">${m.dn}</td>
                            <td class="py-4 text-emerald-400 font-black text-right">${m.p} CFA</td>
                            ${isAdmin ? `<td class="py-4 text-right"><button onclick="deleteMission('${m.id}')" class="text-red-500 p-2 font-black">✕</button></td>` : ''}
                        </tr>`;
                }
            });
        };

        window.showToast = (msg, type) => {
            const t = document.getElementById('toast');
            t.innerText = msg;
            t.className = `fixed bottom-8 left-1/2 -translate-x-1/2 px-8 py-4 rounded-full text-white font-black text-[11px] uppercase tracking-widest z-[1000] shadow-2xl transition-all ${type==='success'?'bg-emerald-500':'bg-red-500'}`;
            t.classList.remove('hidden');
            setTimeout(() => t.classList.add('hidden'), 3000);
        };

        window.resetArchives = async () => {
            const toPurge = allMissions.filter(m => m.s === 2);
            if (toPurge.length === 0) return showToast("Rien à purger", "error");
            if (!confirm(`Voulez-vous supprimer les ${toPurge.length} missions terminées ?`)) return;

            try {
                for (let m of toPurge) {
                    await deleteDoc(getMissionDoc(m.id));
                }
                showToast("Archives purgées", "success");
            } catch (e) {
                showToast("Erreur lors de la purge", "error");
            }
        };

        window.exportCSV = () => {
            const delivered = allMissions.filter(m => m.s === 2);
            if (delivered.length === 0) return showToast("Aucune donnée à exporter", "error");
            let csv = "ID;Date;Destinataire;Quartier;Prix\n";
            delivered.forEach(m => csv += `${m.id};${m.deliveryDate || m.date};${m.dn};${m.dq};${m.p}\n`);
            const blob = new Blob(["\ufeff" + csv], { type: 'text/csv;charset=utf-8;' });
            const url = URL.createObjectURL(blob);
            const link = document.createElement("a");
            link.href = url;
            link.download = `rapport_ct241_${new Date().toISOString().split('T')[0]}.csv`;
            link.click();
        };

    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; color: #0f172a; }
        .hidden { display: none !important; }
        input::placeholder { color: #94a3b8; font-weight: 700; }
    </style>
</head>
<body class="antialiased">

    <div id="toast" class="hidden"></div>

    <!-- ECRAN DE CONNEXION -->
    <section id="authSection" class="fixed inset-0 bg-[#0f172a] flex items-center justify-center p-6 z-[2000]">
        <div class="bg-white w-full max-w-sm rounded-[3.5rem] p-12 shadow-2xl text-center">
            <div class="w-20 h-20 bg-slate-50 rounded-3xl flex items-center justify-center mx-auto mb-8 shadow-inner">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="w-14 h-14 rounded-xl">
            </div>
            <h1 class="text-3xl font-black mb-2 tracking-tighter">CT241 Log</h1>
            <p class="text-slate-400 text-xs font-bold uppercase tracking-[0.2em] mb-10">Portail Logistique</p>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" placeholder="votre@email.com" required class="w-full p-5 bg-slate-50 rounded-2xl font-bold outline-none border-2 border-transparent focus:border-blue-500 focus:bg-white transition-all text-sm">
                <input type="password" id="loginPass" placeholder="Mot de passe" required class="w-full p-5 bg-slate-50 rounded-2xl font-bold outline-none border-2 border-transparent focus:border-blue-500 focus:bg-white transition-all text-sm">
                <p id="loginError" class="text-[10px] text-red-500 font-black uppercase hidden"></p>
                <button type="submit" class="w-full bg-slate-900 text-white py-6 rounded-2xl font-black uppercase text-[11px] tracking-widest shadow-xl shadow-slate-200 hover:scale-[1.02] active:scale-95 transition-all">S'identifier</button>
            </form>
        </div>
    </section>

    <!-- INTERFACE PRINCIPALE -->
    <main id="appContent" class="hidden min-h-screen pb-20">
        <!-- HEADER -->
        <nav class="bg-white/80 backdrop-blur-md p-6 border-b sticky top-0 z-50 flex justify-between items-center">
            <div class="flex items-center gap-4">
                <div class="bg-slate-900 p-2 rounded-xl">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="w-6 h-6 rounded-md">
                </div>
                <div>
                    <p id="userMail" class="text-[10px] font-black text-slate-400 uppercase tracking-wider">...</p>
                    <p class="text-xs font-black text-slate-900">Dashboard Live</p>
                </div>
            </div>
            <button onclick="handleLogout()" class="bg-red-50 text-red-500 px-5 py-3 rounded-xl font-black text-[10px] uppercase tracking-wider">Quitter</button>
        </nav>

        <div class="max-w-md mx-auto p-6 space-y-8">

            <!-- ADMIN PANEL -->
            <section id="adminPanel" class="hidden bg-slate-900 p-8 rounded-[3rem] shadow-2xl space-y-6">
                <div class="flex items-center justify-between">
                    <p class="text-white/40 text-[10px] font-black uppercase tracking-widest">Outils Super Admin</p>
                    <span class="w-2 h-2 bg-emerald-400 rounded-full animate-pulse"></span>
                </div>
                <div class="grid grid-cols-2 gap-4">
                    <button onclick="exportCSV()" class="bg-blue-600 text-white p-5 rounded-2xl text-[10px] font-black uppercase tracking-widest">Rapport CSV</button>
                    <button onclick="resetArchives()" class="bg-white/5 text-red-400 border border-white/10 p-5 rounded-2xl text-[10px] font-black uppercase tracking-widest">Purger Archives</button>
                </div>
            </section>
            
            <!-- CRÉATION (Relais/Admin) -->
            <section id="createSection" class="role-view hidden bg-white p-10 rounded-[3.5rem] shadow-sm border border-slate-100">
                <div class="flex justify-between items-center mb-8">
                    <h2 class="font-black text-lg tracking-tight">Nouveau Colis</h2>
                    <span id="nextIdDisplay" class="bg-blue-50 text-blue-600 px-4 py-2 rounded-full font-black text-[10px]">...</span>
                </div>
                <div class="space-y-4">
                    <div class="space-y-2">
                        <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Origine</label>
                        <input type="text" id="expNom" placeholder="Nom Boutique / Expéditeur" class="w-full p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="expQuartier" placeholder="Quartier" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                            <input type="tel" id="expTel" placeholder="Téléphone" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                        </div>
                    </div>
                    
                    <div class="h-px bg-slate-100 my-4"></div>

                    <div class="space-y-2">
                        <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Destination</label>
                        <input type="text" id="destNom" placeholder="Nom du Client" class="w-full p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="dq" placeholder="Quartier" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                            <input type="tel" id="dt" placeholder="Tél Client" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                        </div>
                    </div>

                    <div class="pt-4">
                        <input type="number" id="price" placeholder="Prix Livraison (CFA)" class="w-full p-6 bg-blue-50 text-blue-700 rounded-[2rem] text-center text-xl font-black outline-none border-2 border-blue-100 focus:bg-white transition-all">
                    </div>
                    
                    <button onclick="saveMission()" class="w-full bg-slate-900 text-white py-6 rounded-[2rem] font-black text-[11px] uppercase tracking-[0.2em] shadow-xl hover:scale-[1.01] transition-transform active:scale-95">Générer le Bordereau</button>
                </div>
            </section>

            <!-- DISPATCH (Dispatch/Admin) -->
            <section id="dispatchSection" class="role-view hidden">
                <div class="flex items-center gap-3 mb-6 ml-2">
                    <span class="w-2 h-2 bg-blue-500 rounded-full"></span>
                    <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Files d'attente (Dispatch)</h3>
                </div>
                <div id="dispatchList"></div>
            </section>

            <!-- LIVREUR (Livreur/Admin) -->
            <section id="livreurSection" class="role-view hidden">
                <div class="flex items-center gap-3 mb-6 ml-2">
                    <span class="w-2 h-2 bg-amber-400 rounded-full"></span>
                    <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Mes Courses Actives</h3>
                </div>
                <div id="livreurList"></div>
            </section>

            <!-- ARCHIVES (Tout le monde) -->
            <section id="archiveSection" class="role-view hidden bg-[#0f172a] rounded-[3.5rem] p-8 shadow-2xl">
                <h3 class="text-white/40 text-[10px] font-black uppercase mb-6 tracking-widest text-center">Historique des Livraisons</h3>
                <div class="overflow-x-auto">
                    <table class="w-full">
                        <tbody id="archiveList"></tbody>
                    </table>
                </div>
            </section>

        </div>
    </main>

</body>
</html>
