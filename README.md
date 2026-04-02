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

        // Helper pour les chemins Firestore (Respect des règles de sécurité)
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
            const btn = e.target.querySelector('button');
            
            btn.innerText = "Connexion...";
            btn.disabled = true;

            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } catch (error) { 
                const errEl = document.getElementById('loginError');
                errEl.innerText = "Identifiants incorrects.";
                errEl.classList.remove('hidden');
                btn.innerText = "S'identifier";
                btn.disabled = false;
            }
        };

        window.handleLogout = async () => { 
            await signOut(auth); 
            location.reload(); 
        };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                const email = user.email.toLowerCase();
                
                // Détermination du rôle par l'email
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
                document.getElementById('archiveSection').classList.remove('hidden');
            }
        };

        // --- CRUD ACTIONS ---
        window.deleteMission = async (id) => {
            if (!confirm(`Supprimer définitivement la mission #${id} ?`)) return;
            try {
                await deleteDoc(getMissionDoc(id));
                showToast("Mission supprimée", "success");
            } catch (e) {
                showToast("Erreur lors de la suppression", "error");
                console.error(e);
            }
        };

        window.generateNewID = () => {
            window.currentMissionId = "241-" + Math.floor(100000 + Math.random() * 900000);
            const el = document.getElementById('nextIdDisplay');
            if (el) el.innerText = window.currentMissionId;
        };

        window.saveMission = async () => {
            const data = {
                id: window.currentMissionId,
                en: document.getElementById('expNom').value.trim(),
                eq: document.getElementById('expQuartier').value.trim(),
                et: document.getElementById('expTel').value.trim(),
                dn: document.getElementById('destNom').value.trim(),
                dq: document.getElementById('dq').value.trim(),
                dt: document.getElementById('dt').value.trim(),
                p: parseFloat(document.getElementById('price').value) || 0,
                s: 0, // Statut : 0=Attente, 1=Cours, 2=Livré
                ts: Date.now(),
                date: new Date().toLocaleDateString('fr-FR'),
                author: currentUser.email
            };

            if (!data.en || !data.dn || !data.dt) return showToast("Champs obligatoires manquants", "error");

            try {
                // On utilise l'ID métier comme ID de document Firestore pour la cohérence
                await setDoc(getMissionDoc(data.id), data);
                showToast("Bordereau enregistré", "success");
                generateNewID();
                // Reset partiel
                document.getElementById('destNom').value = "";
                document.getElementById('dt').value = "";
            } catch (e) {
                showToast("Erreur d'enregistrement", "error");
            }
        };

        // --- ECOUTEURS TEMPS REEL ---
        const startListeners = () => {
            const q = query(getMissionsColl(), orderBy("ts", "desc"));
            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(d => allMissions.push(d.data()));
                refreshTables();
            }, (err) => console.error("Erreur Snapshot:", err));
        };

        const refreshTables = () => {
            const disp = document.getElementById('dispatchList');
            const liv = document.getElementById('livreurList');
            const arch = document.getElementById('archiveList');
            
            if (disp) disp.innerHTML = "";
            if (liv) liv.innerHTML = "";
            if (arch) arch.innerHTML = "";

            allMissions.forEach(m => {
                const isAdmin = userRole === 'admin';
                
                // 1. Dispatch (Missions en attente s=0)
                if (m.s === 0 && (isAdmin || userRole === 'dispatch')) {
                    if (disp) disp.innerHTML += `
                        <div class="bg-white p-5 rounded-[2rem] shadow-sm border border-slate-100 flex justify-between items-center mb-3">
                            <div>
                                <p class="text-[10px] font-black text-blue-500">#${m.id}</p>
                                <p class="text-sm font-bold text-slate-800">${m.dn}</p>
                                <p class="text-[10px] text-slate-400 font-bold uppercase">${m.dq}</p>
                            </div>
                            <div class="flex gap-2">
                                ${isAdmin ? `<button onclick="deleteMission('${m.id}')" class="w-10 h-10 flex items-center justify-center text-red-300 hover:text-red-500 hover:bg-red-50 rounded-full transition-all">✕</button>` : ''}
                                <button onclick="updateStatus('${m.id}', 1)" class="bg-blue-600 text-white px-5 py-3 rounded-xl text-[10px] font-black uppercase tracking-wider shadow-lg shadow-blue-100">Assigner</button>
                            </div>
                        </div>`;
                } 
                // 2. Livreur (Missions en cours s=1)
                else if (m.s === 1 && (isAdmin || userRole === 'livreur')) {
                    if (liv) liv.innerHTML += `
                        <div class="bg-white p-6 rounded-[2.5rem] shadow-xl border-l-8 border-amber-400 mb-4 relative overflow-hidden">
                             ${isAdmin ? `<button onclick="deleteMission('${m.id}')" class="absolute top-4 right-4 text-slate-200 hover:text-red-500 transition-colors">✕</button>` : ''}
                            <div class="mb-4">
                                <span class="bg-amber-50 text-amber-600 text-[9px] font-black px-3 py-1 rounded-full uppercase">En cours</span>
                                <h4 class="font-black text-lg text-slate-900 mt-2">${m.dn}</h4>
                                <p class="text-xs text-slate-500 font-bold uppercase tracking-tight">${m.dq} • ${m.dt}</p>
                            </div>
                            <div class="flex gap-2">
                                <a href="tel:${m.dt}" class="flex-1 bg-slate-100 text-slate-600 py-4 rounded-2xl font-black text-[10px] uppercase text-center">Appeler</a>
                                <button onclick="updateStatus('${m.id}', 2)" 
                                        class="flex-[2] bg-slate-900 text-white py-4 rounded-2xl font-black text-[10px] uppercase tracking-widest hover:bg-emerald-600 transition-all">Terminer</button>
                            </div>
                        </div>`;
                } 
                // 3. Archives (Livrées s=2)
                else if (m.s === 2 && (isAdmin || userRole === 'relais' || userRole === 'dispatch' || userRole === 'livreur')) {
                    if (arch) arch.innerHTML += `
                        <tr class="border-b border-slate-800/50 text-[11px] group">
                            <td class="py-4 text-white/90 font-bold">#${m.id}</td>
                            <td class="py-4 text-slate-400 font-medium">
                                <div class="font-bold text-white/70">${m.dn}</div>
                                <div class="text-[9px] uppercase">${m.dq}</div>
                            </td>
                            <td class="py-4 text-emerald-400 font-black text-right">${m.p} CFA</td>
                            <td class="py-4 text-right">
                                ${isAdmin ? `<button onclick="deleteMission('${m.id}')" class="text-red-500/30 hover:text-red-500 p-2 font-black transition-colors">✕</button>` : ''}
                            </td>
                        </tr>`;
                }
            });
        };

        window.updateStatus = async (id, newStatus) => {
            try {
                const updateData = { s: newStatus };
                if (newStatus === 2) {
                    updateData.deliveryDate = new Date().toLocaleDateString('fr-FR');
                    updateData.deliveredBy = currentUser.email;
                }
                await updateDoc(getMissionDoc(id), updateData);
                showToast(newStatus === 2 ? "Livraison validée" : "Statut mis à jour", "success");
            } catch (e) {
                showToast("Erreur de mise à jour", "error");
            }
        };

        window.showToast = (msg, type) => {
            const t = document.getElementById('toast');
            t.innerText = msg;
            t.className = `fixed bottom-8 left-1/2 -translate-x-1/2 px-8 py-4 rounded-full text-white font-black text-[11px] uppercase tracking-widest z-[1000] shadow-2xl transition-all duration-300 ${type==='success'?'bg-emerald-500':'bg-red-500'}`;
            t.classList.remove('hidden');
            setTimeout(() => t.classList.add('hidden'), 3000);
        };

        // --- FONCTIONS ADMIN ---
        window.resetArchives = async () => {
            const toPurge = allMissions.filter(m => m.s === 2);
            if (toPurge.length === 0) return showToast("Aucune archive à purger", "error");
            if (!confirm(`Voulez-vous supprimer les ${toPurge.length} missions terminées définitivement ?`)) return;

            try {
                for (let m of toPurge) {
                    await deleteDoc(getMissionDoc(m.id));
                }
                showToast("Archives purgées avec succès", "success");
            } catch (e) {
                showToast("Erreur lors de la purge", "error");
            }
        };

        window.exportCSV = () => {
            const delivered = allMissions.filter(m => m.s === 2);
            if (delivered.length === 0) return showToast("Rien à exporter", "error");
            
            let csv = "ID;Date;Livreur;Destinataire;Quartier;Prix\n";
            delivered.forEach(m => {
                csv += `${m.id};${m.deliveryDate || m.date};${m.deliveredBy || 'N/A'};${m.dn};${m.dq};${m.p}\n`;
            });
            
            const blob = new Blob(["\ufeff" + csv], { type: 'text/csv;charset=utf-8;' });
            const url = URL.createObjectURL(blob);
            const link = document.createElement("a");
            link.setAttribute("href", url);
            link.setAttribute("download", `rapport_ct241_${new Date().toISOString().split('T')[0]}.csv`);
            link.click();
        };

    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; color: #0f172a; overflow-x: hidden; }
        .hidden { display: none !important; }
        input::placeholder { color: #cbd5e1; font-weight: 700; }
        
        /* Custom Scrollbar */
        ::-webkit-scrollbar { width: 0px; }
        
        .role-view { animation: fadeIn 0.4s ease-out; }
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }
    </style>
</head>
<body class="antialiased">

    <div id="toast" class="hidden"></div>

    <!-- ECRAN DE CONNEXION -->
    <section id="authSection" class="fixed inset-0 bg-[#0f172a] flex items-center justify-center p-6 z-[2000]">
        <div class="bg-white w-full max-w-sm rounded-[3.5rem] p-10 shadow-2xl text-center border-t-8 border-blue-500">
            <div class="w-20 h-20 bg-slate-50 rounded-3xl flex items-center justify-center mx-auto mb-8 shadow-inner overflow-hidden border border-slate-100">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="w-full h-full object-cover">
            </div>
            <h1 class="text-3xl font-black mb-2 tracking-tighter">CT241 Log</h1>
            <p class="text-slate-400 text-[10px] font-black uppercase tracking-[0.3em] mb-10">Access Control</p>
            
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <div class="text-left">
                    <label class="text-[9px] font-black text-slate-400 uppercase ml-4 mb-1 block">Identifiant</label>
                    <input type="email" id="loginEmail" placeholder="votre@email.com" required class="w-full p-5 bg-slate-50 rounded-2xl font-bold outline-none border-2 border-transparent focus:border-blue-500 focus:bg-white transition-all text-sm">
                </div>
                <div class="text-left">
                    <label class="text-[9px] font-black text-slate-400 uppercase ml-4 mb-1 block">Clé d'accès</label>
                    <input type="password" id="loginPass" placeholder="••••••••" required class="w-full p-5 bg-slate-50 rounded-2xl font-bold outline-none border-2 border-transparent focus:border-blue-500 focus:bg-white transition-all text-sm">
                </div>
                <p id="loginError" class="text-[10px] text-red-500 font-black uppercase hidden bg-red-50 py-2 rounded-lg"></p>
                <button type="submit" class="w-full bg-slate-900 text-white py-6 rounded-2xl font-black uppercase text-[11px] tracking-widest shadow-xl shadow-slate-200 hover:bg-blue-600 transition-all active:scale-95">S'identifier</button>
            </form>
        </div>
    </section>

    <!-- INTERFACE PRINCIPALE -->
    <main id="appContent" class="hidden min-h-screen pb-20">
        <!-- HEADER -->
        <nav class="bg-white/90 backdrop-blur-xl p-5 border-b sticky top-0 z-50 flex justify-between items-center shadow-sm">
            <div class="flex items-center gap-4">
                <div class="bg-slate-900 w-10 h-10 rounded-xl flex items-center justify-center overflow-hidden">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="w-8 h-8 rounded-lg object-cover">
                </div>
                <div>
                    <p id="userMail" class="text-[9px] font-black text-slate-400 uppercase tracking-widest">Utilisateur</p>
                    <div class="flex items-center gap-2">
                        <span class="w-1.5 h-1.5 bg-emerald-500 rounded-full"></span>
                        <p class="text-[11px] font-black text-slate-900 uppercase">Connecté</p>
                    </div>
                </div>
            </div>
            <button onclick="handleLogout()" class="bg-red-50 text-red-500 px-4 py-2.5 rounded-xl font-black text-[10px] uppercase tracking-wider border border-red-100">Déconnexion</button>
        </nav>

        <div class="max-w-md mx-auto p-6 space-y-10">

            <!-- ADMIN PANEL -->
            <section id="adminPanel" class="hidden bg-slate-900 p-8 rounded-[3rem] shadow-2xl space-y-6 border-b-4 border-blue-500">
                <div class="flex items-center justify-between">
                    <p class="text-white/40 text-[10px] font-black uppercase tracking-[0.3em]">Outils Administration</p>
                    <span class="px-3 py-1 bg-blue-500/20 text-blue-400 rounded-full text-[9px] font-black uppercase">Root Mode</span>
                </div>
                <div class="grid grid-cols-2 gap-4">
                    <button onclick="exportCSV()" class="bg-blue-600 text-white p-5 rounded-2xl text-[10px] font-black uppercase tracking-widest shadow-lg shadow-blue-900/50">Exporter CSV</button>
                    <button onclick="resetArchives()" class="bg-white/5 text-red-400 border border-white/10 p-5 rounded-2xl text-[10px] font-black uppercase tracking-widest hover:bg-red-500/10 transition-colors">Purger Tout</button>
                </div>
            </section>
            
            <!-- CRÉATION (Relais/Admin) -->
            <section id="createSection" class="role-view hidden bg-white p-10 rounded-[3.5rem] shadow-xl border border-slate-100">
                <div class="flex justify-between items-center mb-8">
                    <div>
                        <h2 class="font-black text-2xl tracking-tighter">Nouveau Colis</h2>
                        <p class="text-[10px] text-slate-400 font-bold uppercase tracking-widest">Enregistrement</p>
                    </div>
                    <div class="text-right">
                        <span id="nextIdDisplay" class="bg-blue-50 text-blue-600 px-4 py-2 rounded-xl font-black text-xs">...</span>
                    </div>
                </div>
                
                <div class="space-y-6">
                    <div class="space-y-3">
                        <label class="text-[10px] font-black text-slate-500 uppercase ml-4">Informations Expéditeur</label>
                        <input type="text" id="expNom" placeholder="Nom de l'Expéditeur / Boutique" class="w-full p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="expQuartier" placeholder="Quartier" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                            <input type="tel" id="expTel" placeholder="Contact" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                        </div>
                    </div>
                    
                    <div class="h-px bg-slate-100 mx-4"></div>

                    <div class="space-y-3">
                        <label class="text-[10px] font-black text-slate-500 uppercase ml-4">Informations Destinataire</label>
                        <input type="text" id="destNom" placeholder="Nom complet du destinataire" class="w-full p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="dq" placeholder="Quartier" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                            <input type="tel" id="dt" placeholder="Numéro Tél" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                        </div>
                    </div>

                    <div class="pt-4">
                        <label class="text-[10px] font-black text-blue-500 uppercase ml-4 mb-2 block">Montant Livraison</label>
                        <input type="number" id="price" placeholder="Prix (CFA)" class="w-full p-6 bg-blue-50 text-blue-700 rounded-[2rem] text-center text-2xl font-black outline-none border-2 border-blue-100 focus:bg-white transition-all shadow-inner">
                    </div>
                    
                    <button onclick="saveMission()" class="w-full bg-slate-900 text-white py-6 rounded-[2rem] font-black text-[11px] uppercase tracking-[0.2em] shadow-xl shadow-slate-200 hover:bg-blue-600 transition-all active:scale-95">Valider le Bordereau</button>
                </div>
            </section>

            <!-- DISPATCH SECTION -->
            <section id="dispatchSection" class="role-view hidden">
                <div class="flex items-center justify-between mb-6 ml-2">
                    <div class="flex items-center gap-3">
                        <span class="w-2 h-2 bg-blue-500 rounded-full animate-pulse"></span>
                        <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Attente Dispatching</h3>
                    </div>
                </div>
                <div id="dispatchList" class="space-y-3">
                    <div class="p-10 text-center text-slate-300 font-bold text-xs">Chargement des flux...</div>
                </div>
            </section>

            <!-- LIVREUR SECTION -->
            <section id="livreurSection" class="role-view hidden">
                <div class="flex items-center gap-3 mb-6 ml-2">
                    <span class="w-2 h-2 bg-amber-400 rounded-full animate-pulse"></span>
                    <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Courses en cours</h3>
                </div>
                <div id="livreurList" class="space-y-4">
                    <div class="p-10 text-center text-slate-300 font-bold text-xs">Aucune course active.</div>
                </div>
            </section>

            <!-- ARCHIVES (History) -->
            <section id="archiveSection" class="role-view hidden bg-[#0f172a] rounded-[3.5rem] p-8 shadow-2xl border-b-4 border-emerald-500">
                <div class="flex justify-between items-center mb-8">
                    <h3 class="text-white/40 text-[10px] font-black uppercase tracking-widest">Rapport d'activité</h3>
                    <span class="bg-emerald-500/20 text-emerald-400 px-3 py-1 rounded-full text-[9px] font-black uppercase">Terminé</span>
                </div>
                <div class="overflow-hidden">
                    <table class="w-full">
                        <tbody id="archiveList">
                            <tr><td class="py-10 text-center text-slate-600 text-xs font-bold">Initialisation de l'historique...</td></tr>
                        </tbody>
                    </table>
                </div>
            </section>

        </div>
    </main>

    <footer class="fixed bottom-0 left-0 right-0 p-4 pointer-events-none">
        <p class="text-center text-[8px] font-black text-slate-300 uppercase tracking-[0.5em] mb-2">CT241 Logistics OS v2.0</p>
    </footer>

</body>
</html>
