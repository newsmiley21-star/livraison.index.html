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
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, deleteDoc, onSnapshot, query, orderBy } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        // Configuration Firebase (Vérifiée)
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

        // Variables d'état globales
        let currentUser = null;
        let userRole = null;
        let allMissions = [];
        let currentMissionId = null;
        let searchQuery = "";
        let filterToday = false;

        // --- AUTHENTIFICATION ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            
            btn.disabled = true;
            btn.innerText = "Connexion...";
            
            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } catch (error) { 
                console.error(error);
                document.getElementById('loginError').classList.remove('hidden');
                btn.disabled = false; 
                btn.innerText = "ACCÉDER AU PORTAIL";
            }
        };

        window.handleLogout = async () => { 
            await signOut(auth); 
            window.location.reload(); 
        };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                assignRole(user.email);
                document.getElementById('authSection').classList.add('hidden');
                document.getElementById('appContent').classList.remove('hidden');
                document.getElementById('userDisplayEmail').innerText = user.email;
                startListeners();
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
                document.getElementById('adminDash').classList.remove('hidden'); // Stat livreur
            }
        };

        // --- GESTION DES MISSIONS ---
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

            if (!fields.en || !fields.dt || !fields.p) return showToast("Champs obligatoires !", "error");

            try {
                const now = Date.now();
                const missionRef = doc(db, 'missions', window.nextId); // Chemin simplifié
                await setDoc(missionRef, {
                    ...fields, 
                    id: window.nextId, 
                    s: 0, 
                    ca: now, 
                    cad: new Date(now).toISOString().split('T')[0], 
                    ce: currentUser.email
                });
                showToast("Mission créée !", "success");
                prepareNextId();
            } catch (e) { 
                console.error(e);
                showToast("Erreur lors de la création", "error"); 
            }
        };

        window.supprimerMission = async (id) => {
            if (confirm("Supprimer cette mission définitivement ?")) {
                try {
                    await deleteDoc(doc(db, 'missions', id));
                    showToast("Mission supprimée", "success");
                } catch (e) { showToast("Erreur suppression", "error"); }
            }
        };

        window.publierMission = async (id) => {
            try {
                await updateDoc(doc(db, 'missions', id), { s: 1 });
                showToast("Mission expédiée au livreur", "success");
            } catch (e) { showToast("Erreur", "error"); }
        };

        window.toggleFilterToday = () => {
            filterToday = !filterToday;
            document.getElementById('btnFilterToday').className = filterToday 
                ? "bg-yellow-500 text-slate-900 px-4 py-2 rounded-xl font-black text-[10px] uppercase" 
                : "bg-slate-100 text-slate-400 px-4 py-2 rounded-xl font-black text-[10px] uppercase";
            renderUI();
        };

        const startListeners = () => {
            // Écouteur temps réel sur la collection missions
            const q = query(collection(db, 'missions'));
            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
                renderStats();
            }, (error) => {
                console.error("Erreur de lecture : ", error);
            });
        };

        const renderStats = () => {
            const today = new Date().toISOString().split('T')[0];
            const stats = { caTotal: 0, livraisonsTotal: 0, caJour: 0, livreurs: {} };
            
            allMissions.forEach(m => {
                if (m.s === 2) {
                    stats.caTotal += m.p || 0;
                    stats.livraisonsTotal++;
                    if (m.lk === today) stats.caJour += m.p || 0;
                    
                    const l = m.le || 'Inconnu';
                    if (!stats.livreurs[l]) stats.livreurs[l] = { count: 0, ca: 0, bonus: 0 };
                    stats.livreurs[l].count++;
                    stats.livreurs[l].ca += m.p || 0;
                    if (stats.livreurs[l].count > 17) stats.livreurs[l].bonus += 700;
                }
            });

            if (userRole === 'admin') {
                document.getElementById('dashCaTotal').innerText = stats.caTotal.toLocaleString() + ' CFA';
                document.getElementById('dashCaJour').innerText = stats.caJour.toLocaleString() + ' CFA';
                document.getElementById('dashLivTotal').innerText = stats.livraisonsTotal;
                
                const list = document.getElementById('dashLivreursList');
                list.innerHTML = "";
                Object.entries(stats.livreurs).forEach(([email, data]) => {
                    list.innerHTML += `
                    <div class="flex justify-between p-3 bg-slate-800 rounded-xl mb-2 text-[10px]">
                        <span class="text-white truncate w-32">${email}</span>
                        <span class="text-yellow-500 font-black">${data.count} missions</span>
                        <span class="text-emerald-500 font-black">+${data.bonus} CFA</span>
                    </div>`;
                });
            }

            if (userRole === 'livreur') {
                const my = stats.livreurs[currentUser.email] || { count: 0, bonus: 0 };
                // Mise à jour de l'UI spécifique livreur si les éléments existent
                const countEl = document.getElementById('dashLivTotal');
                if(countEl) countEl.innerText = my.count;
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

            // Tri par date de création décroissante
            allMissions.sort((a,b) => b.ca - a.ca).forEach(m => {
                if (filterToday && (m.lk !== today && m.cad !== today)) return;
                
                const isMyColis = m.ce === currentUser.email;
                const delBtn = userRole === 'admin' ? `<button onclick="supprimerMission('${m.id}')" class="p-2 text-red-400 bg-red-500/10 rounded-lg"><svg class="w-3 h-3" fill="currentColor" viewBox="0 0 24 24"><path d="M3 6h18v2H3V6m2 3h14v13c0 1.1-.9 2-2 2H7c-1.1 0-2-.9-2-2V9m3 3v8h2v-8H8m4 0v8h2v-8h-2m4 0v8h2v-8h-2Z"/></svg></button>` : '';

                // 1. Dispatch (Missions créées s=0)
                if (m.s === 0 && (userRole === 'admin' || userRole === 'dispatch')) {
                    containerDispatch.innerHTML += `
                    <div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 shadow-sm flex justify-between items-center animate-in fade-in slide-in-from-left duration-300">
                        <div class="text-[11px]"><span class="font-black text-blue-600 block">${m.id}</span><b>${m.dn}</b><br><span class="text-slate-400">${m.dq}</span></div>
                        <div class="flex gap-2">
                            ${delBtn}
                            <button onclick="openBonImpression('${m.id}')" class="bg-slate-100 text-slate-600 text-[9px] px-3 py-2 rounded-xl font-bold uppercase">Bon</button>
                            <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white text-[10px] px-4 py-2 rounded-xl font-bold uppercase">Expédier</button>
                        </div>
                    </div>`;
                } 
                // 2. Livreur (Missions publiées s=1)
                else if (m.s === 1 && (userRole === 'admin' || userRole === 'livreur')) {
                    containerLivreur.innerHTML += `
                    <div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 space-y-3">
                        <div class="flex justify-between font-black text-amber-700"><span>${m.id}</span><span class="text-[9px] uppercase">À Livrer</span></div>
                        <p class="text-[11px] text-amber-900"><b>Client:</b> ${m.dn} (${m.dt})<br><b>Zone:</b> ${m.dq}</p>
                        <div class="flex gap-2">
                             <button onclick="openBonImpression('${m.id}')" class="flex-1 bg-white border border-amber-200 text-amber-700 font-black py-4 rounded-2xl text-[10px]">BON</button>
                             <button onclick="openCamera('${m.id}')" class="flex-[2] bg-amber-500 text-white font-black py-4 rounded-2xl shadow-lg">LIVRÉ ✅</button>
                        </div>
                    </div>`;
                }

                // 3. Archives (Missions livrées s=2)
                if (m.s === 2) {
                    const canSeeArchive = userRole === 'admin' || userRole === 'dispatch' || (userRole === 'relais' && isMyColis);
                    if (canSeeArchive) {
                        const matchSearch = !searchQuery || m.id.toLowerCase().includes(searchQuery) || m.dn.toLowerCase().includes(searchQuery);
                        if (matchSearch) {
                            archiveBody.innerHTML += `
                            <tr class="border-b border-slate-50 text-[10px] hover:bg-slate-50 transition">
                                <td onclick="openArchiveDetail('${m.id}')" class="p-4 font-bold text-slate-900">${m.id}</td>
                                <td onclick="openArchiveDetail('${m.id}')" class="p-4 text-slate-600"><b>${m.dn}</b><br><span class="text-[8px] text-slate-400">${m.lk}</span></td>
                                <td class="p-4 text-center">${delBtn}</td>
                                <td class="p-4 text-right text-emerald-600 font-black">${(m.p || 0).toLocaleString()}</td>
                            </tr>`;
                        }
                    }
                }
            });
            const counter = document.getElementById('archiveCount');
            if(counter) counter.innerText = archiveBody.children.length;
        };

        // --- APPAREIL PHOTO & PREUVE ---
        window.openCamera = (id) => { 
            currentMissionId = id; 
            document.getElementById('cameraModal').classList.remove('hidden'); 
        };

        window.processImage = (file) => {
            if(!file) return;
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    const MAX_WIDTH = 800;
                    let width = img.width;
                    let height = img.height;

                    if (width > MAX_WIDTH) {
                        height *= MAX_WIDTH / width;
                        width = MAX_WIDTH;
                    }

                    canvas.width = width;
                    canvas.height = height;
                    const ctx = canvas.getContext('2d');
                    ctx.drawImage(img, 0, 0, width, height);
                    finalizeDelivery(canvas.toDataURL('image/jpeg', 0.7));
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };

        const finalizeDelivery = async (photoBase64) => {
            try {
                await updateDoc(doc(db, 'missions', currentMissionId), { 
                    s: 2, 
                    img: photoBase64, 
                    le: currentUser.email, 
                    lk: new Date().toISOString().split('T')[0] 
                });
                document.getElementById('cameraModal').classList.add('hidden');
                showToast("Livraison Validée !", "success");
            } catch (e) { showToast("Erreur validation", "error"); }
        };

        window.handleSearch = (v) => { searchQuery = v.toLowerCase(); renderUI(); };

        const showToast = (m, t) => {
            const el = document.getElementById('toast');
            el.innerText = m; 
            el.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-xs shadow-2xl z-[500] animate-bounce ${t==='success'?'bg-emerald-600':'bg-red-600'}`;
            el.classList.remove('hidden'); 
            setTimeout(() => el.classList.add('hidden'), 3000);
        };

        // --- DÉTAILS & BON D'IMPRESSION ---
        window.openBonImpression = (id) => {
            const m = allMissions.find(x => x.id === id);
            const natureMap = ["Standard", "Repas", "Documents", "Pharma"];
            const modeMap = ["Espèces", "Airtel Money", "Moov Money"];
            
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area bg-white p-8 text-slate-900 min-h-screen">
                    <div class="flex justify-between items-start mb-8 pb-6 border-b-2 border-slate-100">
                        <div>
                            <h1 class="text-4xl font-black text-slate-900">CT241</h1>
                            <p class="text-[10px] font-bold uppercase tracking-widest text-slate-400">Logistique Express</p>
                        </div>
                        <div class="text-right">
                            <h2 class="text-lg font-black uppercase">Bon de Livraison</h2>
                            <p class="text-sm font-bold text-blue-600">${m.id}</p>
                        </div>
                    </div>
                    <div class="grid grid-cols-2 gap-8 mb-8">
                        <div>
                            <p class="text-[9px] font-black text-slate-400 uppercase mb-2">Expéditeur</p>
                            <p class="font-black text-sm">${m.en}</p>
                            <p class="text-xs">${m.eq} • ${m.et}</p>
                        </div>
                        <div>
                            <p class="text-[9px] font-black text-slate-400 uppercase mb-2">Destinataire</p>
                            <p class="font-black text-sm">${m.dn}</p>
                            <p class="text-xs">${m.dq} • ${m.dt}</p>
                        </div>
                    </div>
                    <div class="bg-slate-50 p-4 rounded-2xl mb-8">
                        <table class="w-full text-xs">
                            <tr class="border-b border-slate-200"><td class="py-2 text-slate-500">Nature</td><td class="py-2 text-right font-bold">${natureMap[m.n]}</td></tr>
                            <tr class="border-b border-slate-200"><td class="py-2 text-slate-500">Valeur Marchandise</td><td class="py-2 text-right font-bold">${m.v.toLocaleString()} CFA</td></tr>
                            <tr><td class="py-2 text-slate-500">Mode Paiement</td><td class="py-2 text-right font-bold">${modeMap[m.r]}</td></tr>
                        </table>
                    </div>
                    <div class="text-right mb-12">
                        <p class="text-[10px] font-black text-slate-400 uppercase">Total Frais Livraison</p>
                        <p class="text-3xl font-black text-slate-900">${m.p.toLocaleString()} CFA</p>
                    </div>
                    <div class="grid grid-cols-2 gap-4 mt-20">
                        <div class="border-t border-slate-200 pt-2 text-[10px] text-center font-bold text-slate-400">Signature Expéditeur</div>
                        <div class="border-t border-slate-200 pt-2 text-[10px] text-center font-bold text-slate-400">Signature Client</div>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        window.openArchiveDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            document.getElementById('detailContent').innerHTML = `
                <div class="p-6 space-y-6">
                    <div class="flex justify-between items-center">
                        <h2 class="text-xl font-black">${m.id}</h2>
                        <span class="bg-emerald-100 text-emerald-700 px-4 py-1 rounded-full text-[10px] font-black">LIVRÉ</span>
                    </div>
                    ${m.img ? `<img src="${m.img}" class="w-full rounded-3xl border shadow-sm">` : '<div class="h-40 bg-slate-100 rounded-3xl flex items-center justify-center italic">Aucune preuve photo</div>'}
                    <div class="grid grid-cols-2 gap-4">
                        <div class="bg-slate-50 p-4 rounded-2xl">
                            <p class="text-[9px] font-bold text-slate-400 uppercase">Livreur</p>
                            <p class="text-xs font-black">${m.le || 'N/A'}</p>
                        </div>
                        <div class="bg-slate-50 p-4 rounded-2xl">
                            <p class="text-[9px] font-bold text-slate-400 uppercase">Date Livraison</p>
                            <p class="text-xs font-black">${m.lk || 'N/A'}</p>
                        </div>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; color: #0f172a; }
        .hidden { display: none; }
        @media print { 
            body * { visibility: hidden; } 
            .print-area, .print-area * { visibility: visible; } 
            .print-area { position: absolute; left: 0; top: 0; width: 100%; } 
            .no-print { display: none !important; }
        }
    </style>
</head>
<body class="antialiased">

    <div id="toast" class="hidden"></div>

    <!-- ÉCRAN DE CONNEXION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[3rem] p-10 space-y-8 shadow-2xl text-center">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-20 mx-auto">
            <h1 class="text-2xl font-black text-slate-900 uppercase tracking-tighter">PORTAIL CT241</h1>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs focus:ring-2 focus:ring-slate-900 outline-none" placeholder="E-mail">
                <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs focus:ring-2 focus:ring-slate-900 outline-none" placeholder="Mot de passe">
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden">Email ou mot de passe invalide.</p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl hover:bg-black transition-all active:scale-95">ACCÉDER AU PORTAIL</button>
            </form>
        </div>
    </section>

    <!-- CONTENU PRINCIPAL -->
    <main id="appContent" class="hidden min-h-screen pb-24">
        <nav class="bg-white/80 backdrop-blur-md p-4 sticky top-0 z-50 border-b border-slate-100">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-8">
                    <div>
                        <span class="text-[9px] font-black text-slate-900 block truncate w-24" id="userDisplayEmail">...</span>
                        <div id="badgeDisplay"></div>
                    </div>
                </div>
                <div class="flex gap-2">
                    <button onclick="toggleFilterToday()" id="btnFilterToday" class="bg-slate-100 text-slate-400 px-4 py-2 rounded-xl font-black text-[10px] uppercase transition-colors">Aujourd'hui</button>
                    <button onclick="handleLogout()" class="p-2 bg-slate-100 rounded-full text-slate-400 hover:text-red-500"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m0 0l-4-4m4 4H7" /></svg></button>
                </div>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            <!-- DASHBOARD / ANALYTICS -->
            <section id="adminDash" class="role-section hidden space-y-4">
                <div class="grid grid-cols-3 gap-3">
                    <div class="bg-white p-4 rounded-3xl shadow-sm border border-slate-100 text-center">
                        <span class="text-[8px] font-black text-slate-400 uppercase">Global CA</span>
                        <div id="dashCaTotal" class="text-[11px] font-black text-slate-900">0</div>
                    </div>
                    <div class="bg-white p-4 rounded-3xl shadow-sm border border-slate-100 text-center">
                        <span class="text-[8px] font-black text-slate-400 uppercase">Journée</span>
                        <div id="dashCaJour" class="text-[11px] font-black text-emerald-600">0</div>
                    </div>
                    <div class="bg-white p-4 rounded-3xl shadow-sm border border-slate-100 text-center">
                        <span class="text-[8px] font-black text-slate-400 uppercase">Livrées</span>
                        <div id="dashLivTotal" class="text-[11px] font-black text-blue-600">0</div>
                    </div>
                </div>
                <div class="bg-slate-900 p-4 rounded-[2rem] space-y-2 hidden" id="dashLivreursList"></div>
            </section>

            <!-- FORMULAIRE DE CRÉATION (RELAIS) -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl border border-slate-100 overflow-hidden">
                    <div class="bg-slate-900 p-6 text-white flex justify-between items-center">
                        <div><p class="text-[9px] font-black uppercase opacity-50 tracking-widest">Nouveau Bordereau</p><h2 class="font-black text-sm">ID : <span id="displayNextId">...</span></h2></div>
                        <div class="w-10 h-10 bg-white/10 rounded-xl flex items-center justify-center text-xl">📦</div>
                    </div>
                    <div class="p-6 space-y-4">
                        <input type="text" id="expNom" placeholder="Nom Boutique / Expéditeur" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none border border-transparent focus:border-slate-900">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="expQuartier" placeholder="Quartier Exp" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                            <input type="tel" id="expTel" placeholder="Tél Exp" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        </div>
                        <div class="h-px bg-slate-100 my-2"></div>
                        <input type="text" id="destNom" placeholder="Nom du Destinataire" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none border border-transparent focus:border-blue-600">
                        <div class="grid grid-cols-2 gap-3">
                            <select id="destQuartier" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                                <option value="" disabled selected>Choisir Zone</option>
                                <option>Angondjé</option><option>Nzeng-Ayong</option><option>Oloumi</option><option>Akanda</option><option>Owendo</option><option>Pk0-12s</option><option>Glass</option>
                            </select>
                            <input type="tel" id="destTel" placeholder="Tél Destinataire" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        </div>
                        <div class="grid grid-cols-2 gap-3">
                            <select id="natureColis" class="p-4 bg-slate-900 text-white rounded-2xl font-black text-[10px] outline-none">
                                <option value="0">📦 Standard</option><option value="1">🍟 Repas</option><option value="2">📁 Documents</option><option value="3">💊 Pharma</option>
                            </select>
                            <select id="modeReglement" class="p-4 bg-slate-100 rounded-2xl font-black text-[10px] outline-none">
                                <option value="0">Espèces</option><option value="1">Airtel Money</option><option value="2">Moov Money</option>
                            </select>
                        </div>
                        <div class="grid grid-cols-2 gap-3">
                            <input type="number" id="valeurDeclaree" placeholder="Valeur Colis" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                            <input type="number" id="fraisLivraison" placeholder="PRIX LIV" class="p-4 bg-blue-600 text-white rounded-2xl font-black text-[11px] outline-none placeholder:text-blue-200">
                        </div>
                        <button onclick="genererMission()" class="w-full bg-slate-900 text-white font-black py-5 rounded-3xl shadow-xl hover:bg-black transition-all text-[11px] uppercase tracking-[0.2em] active:scale-95">Enregistrer Mission</button>
                    </div>
                </div>
            </section>

            <!-- DISPATCH -->
            <section id="rubrique2" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-4 ml-2">Missions en attente (Dispatch)</h3>
                <div id="containerDispatch" class="space-y-3"></div>
            </section>

            <!-- LIVREUR -->
            <section id="rubrique3" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-4 ml-2">Mes Livraisons Actives</h3>
                <div id="containerLivreur" class="space-y-4"></div>
            </section>

            <!-- ARCHIVES -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] overflow-hidden shadow-xl border border-slate-100">
                    <div class="p-6 border-b border-slate-50">
                        <div class="flex justify-between items-center mb-4">
                            <h2 class="text-slate-900 font-black text-xs uppercase">Historique Livraisons</h2>
                            <div id="archiveCount" class="bg-slate-900 text-white font-black text-[9px] px-3 py-1 rounded-full">0</div>
                        </div>
                        <input type="text" oninput="handleSearch(this.value)" placeholder="Rechercher ID ou Client..." class="w-full p-4 bg-slate-50 rounded-2xl text-[11px] font-bold outline-none border border-transparent focus:border-slate-300">
                    </div>
                    <div class="overflow-x-auto">
                        <table class="w-full text-left">
                            <tbody id="archiveBody"></tbody>
                        </table>
                    </div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL DÉTAILS -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/95 z-[400] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-2xl bg-white rounded-[3rem] overflow-hidden shadow-2xl animate-in slide-in-from-bottom duration-300">
            <div id="detailContent" class="max-h-[70vh] overflow-y-auto"></div>
            <div class="p-6 bg-slate-50 flex gap-3 no-print">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-4 rounded-2xl text-[11px] uppercase tracking-widest">🖨️ Imprimer</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-6 bg-white border border-slate-200 text-slate-500 font-black py-4 rounded-2xl text-[11px] uppercase">Fermer</button>
            </div>
        </div>
    </div>

    <!-- MODAL PHOTO -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[300] hidden flex flex-col items-center justify-center p-8 text-white">
        <div class="text-center space-y-6 max-w-xs">
            <div class="text-6xl">📸</div>
            <h2 class="text-xl font-black">Preuve de Livraison</h2>
            <p class="text-slate-500 text-xs font-bold uppercase tracking-widest">Prenez une photo du colis ou du bon signé par le client.</p>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-5 rounded-3xl shadow-2xl uppercase text-[11px] tracking-widest active:scale-95 transition-transform">Ouvrir l'appareil</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-400 text-[10px] font-bold underline uppercase">Annuler</button>
        </div>
    </div>

</body>
</html>
