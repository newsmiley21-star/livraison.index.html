<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>CT241 - Logistique & Cloud</title>
    
    <!-- PWA Meta Tags -->
    <meta name="theme-color" content="#0f172a">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="apple-mobile-web-app-title" content="CT241 Log">
    <link rel="apple-touch-icon" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">

    <script src="https://cdn.tailwindcss.com"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut, signInWithCustomToken, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Configuration Firebase (Variables d'environnement)
        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'ct241-logistique';

        let currentUser = null;
        let userRole = null;
        let allMissions = [];
        let currentMissionId = null;
        let searchQuery = "";

        // --- AUTHENTIFICATION ---
        const initAuth = async () => {
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                await signInWithCustomToken(auth, __initial_auth_token);
            } else {
                // Par défaut, on peut utiliser l'anonyme ou attendre le formulaire de login
            }
        };
        initAuth();

        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            const errorEl = document.getElementById('loginError');
            btn.disabled = true;
            btn.innerText = "Connexion...";
            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } catch (error) {
                errorEl.innerText = "Identifiants invalides.";
                errorEl.classList.remove('hidden');
                btn.disabled = false;
                btn.innerText = "ACCÉDER AU PORTAIL";
            }
        };

        window.handleLogout = async () => { await signOut(auth); location.reload(); };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                assignRole(user.email || "livreur@ct241.com"); // Fallback si anonyme pour test
                document.getElementById('authSection').classList.add('hidden');
                document.getElementById('appContent').classList.remove('hidden');
                document.getElementById('userDisplayEmail').innerText = user.email || "Utilisateur";
                startListeners();
                prepareNextId();
            } else {
                document.getElementById('authSection').classList.remove('hidden');
            }
        });

        const assignRole = (email) => {
            const e = email.toLowerCase();
            if (e.includes('admin')) userRole = 'admin';
            else if (e.includes('relais') || e.includes('boutique')) userRole = 'relais';
            else if (e.includes('dispatch')) userRole = 'dispatch';
            else userRole = 'livreur';
            
            document.querySelectorAll('.role-section').forEach(s => s.classList.add('hidden'));
            const badge = document.getElementById('badgeDisplay');
            const colors = { admin: 'bg-red-600', dispatch: 'bg-blue-600', relais: 'bg-emerald-600', livreur: 'bg-amber-600' };
            badge.innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-black uppercase text-white ${colors[userRole]}">${userRole}</span>`;

            if (userRole === 'admin') document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
            else if (userRole === 'relais') { document.getElementById('rubrique1').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'dispatch') { document.getElementById('rubrique2').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'livreur') { document.getElementById('rubrique3').classList.remove('hidden'); document.getElementById('statLivreurSection').classList.remove('hidden'); }
        };

        // --- GESTION DES MISSIONS (CLOUDSYNC) ---
        const prepareNextId = () => {
            window.nextId = "CT-" + Math.floor(Math.random() * 900000 + 100000);
            document.getElementById('displayNextId').innerText = window.nextId;
        };

        const getMissionsRef = () => collection(db, 'artifacts', appId, 'public', 'data', 'missions');

        window.genererMission = async () => {
            if (!currentUser) return;
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
                s: 0, // 0: Créé, 1: En cours, 2: Livré
                ca: Date.now(),
                ce: currentUser.email,
                id: window.nextId
            };

            try {
                await setDoc(doc(getMissionsRef(), window.nextId), fields);
                showToast("Bordereau enregistré dans le Cloud", "success");
                ['expNom', 'expQuartier', 'expTel', 'destNom', 'destQuartier', 'destTel', 'valeurDeclaree', 'fraisLivraison'].forEach(id => document.getElementById(id).value = "");
                prepareNextId();
            } catch (e) { showToast("Erreur de sauvegarde", "error"); }
        };

        window.publierMission = async (id) => {
            await updateDoc(doc(getMissionsRef(), id), { s: 1 });
            showToast("Mission assignée aux livreurs", "success");
        };

        window.openCamera = (id) => { currentMissionId = id; document.getElementById('cameraModal').classList.remove('hidden'); };
        
        window.processImage = (file) => {
            if(!file) return;
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    canvas.width = 600; canvas.height = img.height * (600 / img.width);
                    canvas.getContext('2d').drawImage(img, 0, 0, canvas.width, canvas.height);
                    finalizeDelivery(canvas.toDataURL('image/jpeg', 0.6));
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };

        const finalizeDelivery = async (photo) => {
            await updateDoc(doc(getMissionsRef(), currentMissionId), {
                s: 2, img: photo, le: currentUser.email, lk: new Date().toISOString().split('T')[0]
            });
            document.getElementById('cameraModal').classList.add('hidden');
            showToast("Livraison validée et archivée", "success");
        };

        const startListeners = () => {
            onSnapshot(getMissionsRef(), (snap) => {
                allMissions = snap.docs.map(doc => doc.data());
                renderUI();
                renderStats();
            }, (err) => console.error("Erreur Sync:", err));
        };

        // --- RENDU UI ---
        const renderStats = () => {
            const today = new Date().toISOString().split('T')[0];
            const stats = { caTotal: 0, livraisonsTotal: 0, caJour: 0, livreurs: {} };
            
            allMissions.forEach(m => {
                if (m.s === 2) {
                    stats.caTotal += m.p || 0;
                    stats.livraisonsTotal++;
                    if (m.lk === today) stats.caJour += m.p || 0;
                    const lEmail = m.le || 'Anonyme';
                    if (!stats.livreurs[lEmail]) stats.livreurs[lEmail] = { count: 0, ca: 0 };
                    stats.livreurs[lEmail].count++;
                }
            });

            if (userRole === 'admin') {
                document.getElementById('dashCaTotal').innerText = stats.caTotal.toLocaleString() + ' CFA';
                document.getElementById('dashCaJour').innerText = stats.caJour.toLocaleString() + ' CFA';
                document.getElementById('dashLivTotal').innerText = stats.livraisonsTotal;
                const list = document.getElementById('dashLivreursList');
                list.innerHTML = Object.entries(stats.livreurs).map(([email, data]) => `
                    <div class="flex justify-between p-3 bg-slate-800 rounded-xl mb-2 text-[10px]">
                        <span class="text-white truncate w-32">${email}</span>
                        <span class="text-yellow-500 font-black">${data.count} LIVRAISONS</span>
                    </div>`).join('');
            }

            if (userRole === 'livreur') {
                const my = stats.livreurs[currentUser.email] || { count: 0 };
                document.getElementById('myCount').innerText = my.count;
                document.getElementById('progBar').style.width = Math.min((my.count / 17) * 100, 100) + '%';
            }
        };

        window.renderUI = () => {
            const containers = { 
                dispatch: document.getElementById('containerDispatch'), 
                livreur: document.getElementById('containerLivreur'), 
                archives: document.getElementById('archiveBody') 
            };
            Object.values(containers).forEach(c => { if(c) c.innerHTML = "" });

            allMissions.sort((a,b) => (b.ca || 0) - (a.ca || 0)).forEach(m => {
                const isMyColis = m.ce === currentUser.email;
                
                // Dispatch
                if (m.s === 0 && (userRole === 'admin' || userRole === 'dispatch')) {
                    containers.dispatch.innerHTML += `
                    <div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 shadow-sm flex justify-between items-center">
                        <div class="text-[11px]"><span class="font-black text-blue-600 block">${m.id}</span><b>${m.dn}</b><br><span class="text-slate-400">${m.dq}</span></div>
                        <div class="flex gap-2">
                            <button onclick="openBonImpression('${m.id}')" class="bg-slate-100 text-slate-600 text-[9px] px-3 py-2 rounded-xl font-bold uppercase">Bordereau</button>
                            <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white text-[10px] px-4 py-2 rounded-xl font-bold">DISPATCH</button>
                        </div>
                    </div>`;
                } 
                // Livreurs
                else if (m.s === 1 && (userRole === 'admin' || userRole === 'livreur')) {
                    containers.livreur.innerHTML += `
                    <div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 space-y-3">
                        <div class="flex justify-between font-black items-center text-amber-700"><span>${m.id}</span><span class="text-[9px] bg-amber-500 text-white px-2 py-1 rounded-full uppercase">À LIVRER</span></div>
                        <p class="text-[11px]"><b>Dest :</b> ${m.dn} (${m.dt})<br><b>Lieu :</b> ${m.dq}</p>
                        <div class="flex gap-2">
                             <button onclick="openBonImpression('${m.id}')" class="flex-1 bg-white border border-amber-200 text-amber-700 font-black py-4 rounded-2xl text-[10px]">VOIR BON</button>
                             <button onclick="openCamera('${m.id}')" class="flex-[2] bg-amber-500 text-white font-black py-4 rounded-2xl shadow-lg">VALIDER LIVRAISON</button>
                        </div>
                    </div>`;
                }
                
                // Archives
                if ((userRole === 'admin' || userRole === 'dispatch' || (userRole === 'relais' && isMyColis)) && m.s === 2) {
                    if (!searchQuery || m.id.toLowerCase().includes(searchQuery) || (m.dn || "").toLowerCase().includes(searchQuery)) {
                        containers.archives.innerHTML += `
                        <tr class="border-b border-slate-800 text-[10px] hover:bg-slate-800 transition cursor-pointer" onclick="openArchiveDetail('${m.id}')">
                            <td class="p-4 font-bold text-white">${m.id}</td>
                            <td class="p-4 text-slate-300">${m.dn}</td>
                            <td class="p-4 text-center"><span class="text-emerald-400 font-black">LIVRÉ</span></td>
                            <td class="p-4 text-right text-slate-400">${(m.p || 0).toLocaleString()}</td>
                        </tr>`;
                    }
                }
            });
            const c = document.getElementById('archiveCount'); if(c) c.innerText = containers.archives.children.length;
        };

        // --- MODAL DE DÉTAIL / BORDEREAU ---
        window.openBonImpression = (id) => {
            const m = allMissions.find(x => x.id === id);
            const natureMap = ["📦 Standard", "🍟 Repas", "📁 Documents", "💊 Pharma"];
            const regMap = ["Espèces", "Airtel Money", "Moov Money"];
            
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area bg-white p-8 text-slate-900 min-h-screen font-sans">
                    <div class="flex justify-between border-b-2 pb-6 mb-8">
                        <div><h1 class="text-3xl font-black">CT241</h1><p class="text-[10px] font-bold text-slate-400 uppercase">Logistique & Performance</p></div>
                        <div class="text-right"><h2 class="text-xl font-black uppercase">Bon de Livraison</h2><p class="text-xs font-bold text-slate-500">${m.id}</p></div>
                    </div>
                    <div class="grid grid-cols-2 gap-8 mb-10">
                        <div class="p-4 bg-slate-50 rounded-2xl">
                            <p class="text-[9px] font-black text-slate-400 uppercase mb-2">Expéditeur</p>
                            <p class="font-black">${m.en}</p><p class="text-xs">${m.eq}</p><p class="text-xs font-bold">${m.et}</p>
                        </div>
                        <div class="p-4 bg-slate-900 text-white rounded-2xl">
                            <p class="text-[9px] font-black text-white/50 uppercase mb-2">Destinataire</p>
                            <p class="font-black">${m.dn}</p><p class="text-xs">${m.dq}</p><p class="text-xs font-bold">${m.dt}</p>
                        </div>
                    </div>
                    <table class="w-full mb-8">
                        <tr class="bg-slate-100 text-[10px] font-black uppercase text-slate-500 border-b">
                            <th class="p-4 text-left">Description</th><th class="p-4">Nature</th><th class="p-4 text-right">Frais</th>
                        </tr>
                        <tr class="border-b">
                            <td class="p-4 text-sm font-bold">Livraison Express CT241</td>
                            <td class="p-4 text-center text-xs">${natureMap[m.n]}</td>
                            <td class="p-4 text-right font-black">${m.p.toLocaleString()} CFA</td>
                        </tr>
                    </table>
                    <div class="flex justify-between items-center p-6 bg-slate-50 rounded-3xl">
                        <div class="text-xs"><b>Règlement :</b> ${regMap[m.r]}</div>
                        <div class="text-right"><p class="text-[10px] font-bold text-slate-400 uppercase">Total à percevoir</p><p class="text-2xl font-black">${m.p.toLocaleString()} CFA</p></div>
                    </div>
                    <div class="mt-12 grid grid-cols-2 gap-12 text-center">
                        <div class="border-t pt-4"><p class="text-[9px] font-black uppercase text-slate-300">Visa Boutique</p><div class="h-20"></div></div>
                        <div class="border-t pt-4 font-black"><p class="text-[9px] font-black uppercase text-slate-900">Signature Client</p><div class="h-20"></div></div>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        window.openArchiveDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            document.getElementById('detailContent').innerHTML = `
                <div class="p-6 space-y-4">
                    <h2 class="text-xl font-black">Preuve de Livraison - ${m.id}</h2>
                    ${m.img ? `<img src="${m.img}" class="w-full rounded-3xl shadow-xl">` : '<div class="p-10 bg-slate-100 rounded-3xl text-center italic">Aucune photo</div>'}
                    <div class="p-4 bg-slate-50 rounded-2xl space-y-2 text-xs">
                        <p><b>Livreur :</b> ${m.le}</p>
                        <p><b>Statut :</b> Livré le ${m.lk}</p>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        const showToast = (msg, type) => {
            const t = document.getElementById('toast'); t.innerText = msg;
            t.className = `fixed bottom-24 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-xs shadow-2xl z-[500] ${type==='success'?'bg-emerald-600':'bg-red-600'}`;
            t.classList.remove('hidden'); setTimeout(() => t.classList.add('hidden'), 3000);
        };
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; }
        .hidden { display: none; }
        @media print { 
            body * { visibility: hidden; } 
            .print-area, .print-area * { visibility: visible; } 
            .print-area { position: fixed; left: 0; top: 0; width: 100%; margin: 0; } 
        }
    </style>
</head>
<body class="min-h-screen">

    <div id="toast" class="hidden"></div>

    <!-- AUTHENTICATION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[3rem] p-10 space-y-8 shadow-2xl text-center">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-20 mx-auto">
            <h1 class="text-xl font-black">PORTAIL CT241</h1>
            <form onsubmit="handleLogin(event)" class="space-y-4 text-left">
                <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="E-mail">
                <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="Mot de passe">
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden"></p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl">CONNEXION</button>
            </form>
        </div>
    </section>

    <!-- APP CONTENT -->
    <main id="appContent" class="hidden pb-24">
        <nav class="bg-white/80 backdrop-blur-md p-4 sticky top-0 z-50 border-b flex justify-between items-center">
            <div class="flex items-center gap-2">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-6">
                <span id="userDisplayEmail" class="text-[10px] font-black truncate max-w-[100px]">...</span>
                <div id="badgeDisplay"></div>
            </div>
            <button onclick="handleLogout()" class="text-slate-400 p-2">S'échapper</button>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            
            <!-- ADMIN DASH -->
            <section id="adminDash" class="role-section hidden space-y-4">
                <div class="grid grid-cols-3 gap-3">
                    <div class="bg-slate-900 p-4 rounded-3xl text-white text-center">
                        <span class="text-[7px] font-black text-slate-500 uppercase">CA Total</span>
                        <div id="dashCaTotal" class="text-[10px] font-black text-emerald-400">0</div>
                    </div>
                    <div class="bg-slate-900 p-4 rounded-3xl text-white text-center">
                        <span class="text-[7px] font-black text-slate-500 uppercase">CA Jour</span>
                        <div id="dashCaJour" class="text-[10px] font-black text-yellow-500">0</div>
                    </div>
                    <div class="bg-slate-900 p-4 rounded-3xl text-white text-center">
                        <span class="text-[7px] font-black text-slate-500 uppercase">Livrées</span>
                        <div id="dashLivTotal" class="text-[10px] font-black text-blue-400">0</div>
                    </div>
                </div>
                <div id="dashLivreursList" class="bg-slate-900 p-4 rounded-3xl"></div>
            </section>

            <!-- STATS LIVREUR -->
            <section id="statLivreurSection" class="role-section hidden">
                <div class="bg-slate-900 p-6 rounded-[2.5rem] text-white space-y-4">
                    <div class="flex justify-between items-end">
                        <div><p class="text-[9px] font-black text-slate-400 uppercase">Courses</p><h2 id="myCount" class="text-4xl font-black">0</h2></div>
                        <div class="w-1/2">
                             <div class="w-full h-2 bg-slate-800 rounded-full overflow-hidden mb-2">
                                <div id="progBar" class="h-full bg-emerald-500" style="width: 0%"></div>
                             </div>
                             <p class="text-[7px] text-slate-500 font-bold">Objectif Bonus : 17/jour</p>
                        </div>
                    </div>
                </div>
            </section>

            <!-- CRÉATION BORDEREAU -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl p-6 space-y-5 border border-slate-100">
                    <h2 class="font-black text-xs uppercase tracking-widest text-emerald-600">Nouveau Colis : <span id="displayNextId">...</span></h2>
                    <div class="space-y-3">
                        <input type="text" id="expNom" placeholder="Expéditeur (Boutique)" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-xs">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="expQuartier" placeholder="Quartier Origine" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs">
                            <input type="tel" id="expTel" placeholder="Tél Expéditeur" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs">
                        </div>
                    </div>
                    <div class="space-y-3">
                        <input type="text" id="destNom" placeholder="Destinataire (Client)" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-xs">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="destQuartier" placeholder="Destination" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs">
                            <input type="tel" id="destTel" placeholder="Tél Client" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs">
                        </div>
                    </div>
                    <div class="grid grid-cols-2 gap-3">
                        <select id="natureColis" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs">
                            <option value="0">📦 Standard</option><option value="1">🍟 Repas</option><option value="2">📁 Docs</option><option value="3">💊 Pharma</option>
                        </select>
                        <select id="modeReglement" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs">
                            <option value="0">Espèces</option><option value="1">Airtel Money</option><option value="2">Moov Money</option>
                        </select>
                    </div>
                    <div class="grid grid-cols-2 gap-3">
                        <input type="number" id="valeurDeclaree" placeholder="Valeur Marchandise" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs">
                        <input type="number" id="fraisLivraison" placeholder="TOTAL À PERCEVOIR" class="p-4 bg-emerald-600 text-white rounded-2xl font-black text-xs">
                    </div>
                    <button onclick="genererMission()" class="w-full bg-slate-900 text-white font-black py-5 rounded-3xl shadow-xl uppercase text-[10px] tracking-widest">ENREGISTRER AU CLOUD</button>
                </div>
            </section>

            <!-- CONTAINERS DYNAMIQUES -->
            <section id="rubrique2" class="role-section hidden"><div id="containerDispatch"></div></section>
            <section id="rubrique3" class="role-section hidden"><div id="containerLivreur"></div></section>

            <!-- ARCHIVES -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-slate-900 rounded-[2.5rem] overflow-hidden">
                    <div class="p-6 flex justify-between items-center"><h2 class="text-white font-black text-xs uppercase">Archives Cloud</h2><div id="archiveCount" class="bg-emerald-500 text-white px-3 py-1 rounded-full text-[9px] font-black">0</div></div>
                    <div class="overflow-x-auto"><table class="w-full"><tbody id="archiveBody"></tbody></table></div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODALS -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/90 z-[400] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-2xl bg-white rounded-[2.5rem] overflow-y-auto max-h-[90vh]">
            <div id="detailContent"></div>
            <div class="p-6 bg-slate-50 flex gap-2 no-print">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-4 rounded-2xl text-[10px]">🖨️ IMPRIMER</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-6 bg-slate-200 text-slate-600 font-black py-4 rounded-2xl text-[10px]">FERMER</button>
            </div>
        </div>
    </div>

    <div id="cameraModal" class="fixed inset-0 bg-black z-[300] hidden flex flex-col items-center justify-center p-8 text-white">
        <div class="text-center space-y-8 animate-in zoom-in duration-300">
            <div class="text-6xl">📸</div>
            <h2 class="text-2xl font-black">Preuve de Livraison</h2>
            <p class="text-slate-500 text-xs">Prenez une photo du colis livré ou du bon signé.</p>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-5 rounded-3xl shadow-2xl uppercase tracking-widest text-xs">OUVRIR L'APPAREIL</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 font-bold uppercase text-[9px]">ANNULER</button>
        </div>
    </div>

</body>
</html>
