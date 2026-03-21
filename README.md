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
        import { getFirestore, collection, doc, setDoc, updateDoc, deleteDoc, onSnapshot, query, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        // Configuration Firebase
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

        // --- PWA SETUP ---
        const manifest = {
            "name": "CT241 Logistique",
            "short_name": "CT241",
            "start_url": ".",
            "display": "standalone",
            "background_color": "#0f172a",
            "theme_color": "#0f172a",
            "description": "Système de gestion logistique CT241 Gabon",
            "icons": [
                { "src": "https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png", "sizes": "192x192", "type": "image/png" },
                { "src": "https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png", "sizes": "512x512", "type": "image/png" }
            ]
        };
        const blob = new Blob([JSON.stringify(manifest)], {type: 'application/json'});
        document.getElementById('pwa-manifest').setAttribute('href', URL.createObjectURL(blob));

        let currentUser = null;
        let userRole = null;
        let allMissions = [];
        let currentMissionId = null;
        let searchQuery = "";
        let filterToday = false;

        // --- LOGIQUE AUTHENTIFICATION ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            const errorEl = document.getElementById('loginError');

            btn.disabled = true;
            btn.innerText = "Connexion en cours...";
            try {
                await signInWithEmailAndPassword(auth, email, pass);
            } catch (error) {
                errorEl.innerText = "Identifiants invalides.";
                errorEl.classList.remove('hidden');
                btn.disabled = false;
                btn.innerText = "ACCÉDER AU PORTAIL";
            }
        };

        window.handleLogout = async () => {
            await signOut(auth);
            location.reload();
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
            else if (e.includes('dispatch') || e.includes('gestionnaire')) userRole = 'dispatch';
            else if (e.includes('livreur')) userRole = 'livreur';
            
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
                document.getElementById('statLivreurSection').classList.remove('hidden');
            }
        };

        // --- GESTION MISSIONS ---
        const prepareNextId = () => {
            const id = "2026-" + Math.floor(Math.random() * 900000 + 100000);
            document.getElementById('displayNextId').innerText = id;
            window.nextId = id;
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
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId), {
                    ...fields, 
                    id: window.nextId, 
                    s: 0, 
                    ca: now,
                    cad: new Date(now).toISOString().split('T')[0], // Cache date pour filtre rapide
                    ce: currentUser.email
                });
                showToast("Bon créé avec succès !", "success");
                ['expNom', 'expQuartier', 'expTel', 'destNom', 'destQuartier', 'destTel', 'valeurDeclaree', 'fraisLivraison'].forEach(id => document.getElementById(id).value = "");
                prepareNextId();
            } catch (e) { showToast("Erreur lors de la création", "error"); }
        };

        window.publierMission = async (id) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { s: 1 });
            showToast("Mission publiée", "success");
        };

        window.supprimerMission = async (id) => {
            if (!confirm("Supprimer définitivement cette mission ?")) return;
            try {
                await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id));
                showToast("Mission supprimée", "success");
            } catch (e) { showToast("Erreur suppression", "error"); }
        };

        window.toggleFilterToday = () => {
            filterToday = !filterToday;
            const btn = document.getElementById('btnFilterToday');
            btn.className = filterToday 
                ? "bg-yellow-500 text-slate-900 px-4 py-2 rounded-xl font-black text-[10px] shadow-lg" 
                : "bg-slate-800 text-slate-400 px-4 py-2 rounded-xl font-black text-[10px]";
            renderUI();
        };

        window.exportPerformance = () => {
            const today = new Date().toISOString().split('T')[0];
            let csv = "Livreur;Missions;CA;Bonus\n";
            const stats = calculateStats();
            Object.entries(stats.livreurs).forEach(([email, data]) => {
                csv += `${email};${data.count};${data.ca};${data.bonus}\n`;
            });
            
            const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
            const link = document.createElement("a");
            link.href = URL.createObjectURL(blob);
            link.setAttribute("download", `Rapport_Performance_${today}.csv`);
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
            showToast("Rapport téléchargé", "success");
        };

        const calculateStats = () => {
            const today = new Date().toISOString().split('T')[0];
            const stats = { caTotal: 0, livraisonsTotal: 0, caJour: 0, livreurs: {} };
            allMissions.forEach(m => {
                if (m.s === 2) {
                    stats.caTotal += m.p || 0;
                    stats.livraisonsTotal++;
                    if (m.lk === today) stats.caJour += m.p || 0;
                    const lEmail = m.le || 'Inconnu';
                    if (!stats.livreurs[lEmail]) stats.livreurs[lEmail] = { count: 0, ca: 0, bonus: 0 };
                    stats.livreurs[lEmail].count++;
                    stats.livreurs[lEmail].ca += m.p || 0;
                    if (stats.livreurs[lEmail].count > 17) stats.livreurs[lEmail].bonus += 700;
                }
            });
            return stats;
        };

        const startListeners = () => {
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'missions'), (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
                renderStats();
            });
        };

        const renderStats = () => {
            const stats = calculateStats();
            if (userRole === 'admin') {
                document.getElementById('dashCaTotal').innerText = stats.caTotal.toLocaleString() + ' CFA';
                document.getElementById('dashCaJour').innerText = stats.caJour.toLocaleString() + ' CFA';
                document.getElementById('dashLivTotal').innerText = stats.livraisonsTotal;
                const listLivreurs = document.getElementById('dashLivreursList');
                listLivreurs.innerHTML = "";
                Object.entries(stats.livreurs).forEach(([email, data]) => {
                    listLivreurs.innerHTML += `
                        <div class="flex justify-between items-center p-3 bg-slate-800 rounded-xl mb-2 text-[10px]">
                            <div class="truncate w-32"><span class="text-slate-400 block uppercase font-bold">Livreur</span><span class="text-white">${email}</span></div>
                            <div class="text-center"><span class="text-slate-400 block uppercase font-bold">Missions</span><span class="text-yellow-500 font-black">${data.count}</span></div>
                            <div class="text-right"><span class="text-slate-400 block uppercase font-bold">Bonus</span><span class="text-emerald-500 font-black">+${data.bonus} CFA</span></div>
                        </div>`;
                });
            }
            if (userRole === 'livreur') {
                const myData = stats.livreurs[currentUser.email] || { count: 0, bonus: 0 };
                document.getElementById('myCount').innerText = myData.count;
                document.getElementById('myBonus').innerText = myData.bonus + ' CFA';
                document.getElementById('progBar').style.width = Math.min((myData.count / 17) * 100, 100) + '%';
                document.getElementById('progLabel').innerText = myData.count >= 17 ? 'Bonus Débloqué !' : `Objectif Bonus : ${myData.count}/17 missions`;
            }
        };

        window.renderUI = () => {
            const containers = {
                dispatch: document.getElementById('containerDispatch'),
                livreur: document.getElementById('containerLivreur'),
                archives: document.getElementById('archiveBody')
            };
            Object.values(containers).forEach(c => { if(c) c.innerHTML = "" });
            const todayStr = new Date().toISOString().split('T')[0];

            allMissions.sort((a,b) => (b.ca || 0) - (a.ca || 0)).forEach(m => {
                const isToday = m.lk === todayStr || m.cad === todayStr;
                if (filterToday && !isToday) return;

                const isMyColis = m.ce === currentUser.email;
                const adminDelBtn = userRole === 'admin' ? `<button onclick="supprimerMission('${m.id}')" class="p-2 text-red-400 bg-red-500/10 rounded-lg"><svg class="w-3 h-3" fill="currentColor" viewBox="0 0 24 24"><path d="M3 6h18v2H3V6m2 3h14v13c0 1.1-.9 2-2 2H7c-1.1 0-2-.9-2-2V9m3 3v8h2v-8H8m4 0v8h2v-8h-2m4 0v8h2v-8h-2Z"/></svg></button>` : '';

                if (m.s === 0 && (userRole === 'admin' || userRole === 'dispatch') && containers.dispatch) {
                    containers.dispatch.innerHTML += `
                    <div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 shadow-sm flex justify-between items-center animate-in slide-in-from-right">
                        <div class="text-[11px]"><span class="font-black text-blue-600 block">${m.id}</span><b>${m.dn}</b><br><span class="text-slate-400">${m.dq}</span></div>
                        <div class="flex gap-2">
                            ${adminDelBtn}
                            <button onclick="openBonImpression('${m.id}')" class="bg-slate-100 text-slate-600 text-[9px] px-3 py-2 rounded-xl font-bold">BON</button>
                            <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white text-[10px] px-5 py-2 rounded-xl font-bold">EXPÉDIER</button>
                        </div>
                    </div>`;
                } 
                else if (m.s === 1 && (userRole === 'admin' || userRole === 'livreur') && containers.livreur) {
                    containers.livreur.innerHTML += `
                    <div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 space-y-3 shadow-md animate-in slide-in-from-bottom">
                        <div class="flex justify-between font-black items-center text-amber-700"><span>${m.id}</span><span class="text-[9px] bg-amber-500 text-white px-2 py-1 rounded-full uppercase">En cours</span></div>
                        <p class="text-[11px] text-amber-900"><b>Lieu:</b> ${m.dq}<br><b>Contact:</b> ${m.dn} (${m.dt})</p>
                        <div class="flex gap-2">
                             <button onclick="openBonImpression('${m.id}')" class="flex-1 bg-white border border-amber-200 text-amber-700 font-black py-4 rounded-2xl text-[10px]">REÇU</button>
                             <button onclick="openCamera('${m.id}')" class="flex-[2] bg-amber-500 text-white font-black py-4 rounded-2xl shadow-lg">VALIDER LIVRAISON</button>
                        </div>
                    </div>`;
                }
                
                if ((userRole === 'admin' || userRole === 'dispatch' || (userRole === 'relais' && isMyColis)) && m.s === 2 && containers.archives) {
                    const matchSearch = !searchQuery || m.id.toLowerCase().includes(searchQuery) || (m.dn || "").toLowerCase().includes(searchQuery) || (m.dq || "").toLowerCase().includes(searchQuery);
                    if (matchSearch) {
                        containers.archives.innerHTML += `
                        <tr class="border-b border-slate-800 text-[10px] hover:bg-slate-800 transition cursor-pointer">
                            <td onclick="openArchiveDetail('${m.id}')" class="p-4 font-bold text-white">${m.id}</td>
                            <td onclick="openArchiveDetail('${m.id}')" class="p-4 text-slate-300"><b>${m.dn}</b></td>
                            <td class="p-4 text-center">
                                <div class="flex items-center justify-center gap-2">
                                    <span class="text-emerald-400 font-black">LIVRÉ</span>
                                    ${adminDelBtn}
                                </div>
                            </td>
                            <td class="p-4 text-right text-slate-400">${(m.p || 0).toLocaleString()}</td>
                        </tr>`;
                    }
                }
            });
            document.getElementById('archiveCount').innerText = containers.archives.children.length;
        };

        // --- CAMERA & MODALS (Reste identique mais simplifié) ---
        window.openCamera = (id) => { currentMissionId = id; document.getElementById('cameraModal').classList.remove('hidden'); };
        window.processImage = (file) => {
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    canvas.width = 800; canvas.height = img.height * (800 / img.width);
                    canvas.getContext('2d').drawImage(img, 0, 0, canvas.width, canvas.height);
                    finalizeDelivery(canvas.toDataURL('image/jpeg', 0.5));
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };
        const finalizeDelivery = async (photo) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId), { s: 2, img: photo, le: currentUser.email, lk: new Date().toISOString().split('T')[0] });
            document.getElementById('cameraModal').classList.add('hidden');
            showToast("Clôturé !", "success");
        };

        window.handleSearch = (v) => { searchQuery = v.toLowerCase(); renderUI(); };
        const showToast = (m, t) => {
            const el = document.getElementById('toast');
            el.innerText = m; el.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-xs shadow-2xl z-[500] ${t==='success'?'bg-emerald-600':'bg-red-600'}`;
            el.classList.remove('hidden'); setTimeout(() => el.classList.add('hidden'), 3000);
        };

        window.openArchiveDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            if (!m) return;
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area bg-white p-6 space-y-4 text-slate-900 rounded-3xl">
                    <div class="flex justify-between items-center border-b-2 border-slate-100 pb-4">
                        <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-10">
                        <div class="text-right font-black text-xl text-emerald-600">${m.id}</div>
                    </div>
                    ${m.img ? `<img src="${m.img}" class="w-full h-64 object-cover rounded-2xl">` : '<div class="h-20 flex items-center justify-center italic">Pas de photo</div>'}
                    <div class="p-4 bg-slate-50 rounded-2xl text-[10px]">
                        <p><b>Livreur :</b> ${m.le}</p><p><b>Date :</b> ${m.lk}</p><p><b>Client :</b> ${m.dn} (${m.dq})</p>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        window.openBonImpression = (id) => {
            const m = allMissions.find(x => x.id === id);
            const natureMap = ["Standard", "Repas", "Documents", "Pharma"];
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area bg-white p-8 text-slate-900 font-serif">
                    <div class="flex justify-between items-center border-b-4 border-slate-900 pb-4 mb-6">
                        <h1 class="text-3xl font-black">CT241</h1>
                        <div class="text-right"><h2 class="text-lg font-black uppercase">Bon de Livraison</h2><p class="text-[10px]">N° ${m.id}</p></div>
                    </div>
                    <div class="space-y-6 text-[12px]">
                        <div><h3 class="bg-slate-100 p-1 font-black mb-1">EXPÉDITEUR</h3><p><b>${m.en}</b> (${m.eq})</p></div>
                        <div><h3 class="bg-slate-100 p-1 font-black mb-1">DESTINATAIRE</h3><p><b>${m.dn}</b> (${m.dq})</p></div>
                        <div class="border-y-2 border-slate-200 py-4 text-center"><p class="text-3xl font-black">${m.p.toLocaleString()} CFA</p></div>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; overflow-x: hidden; -webkit-tap-highlight-color: transparent; }
        .hidden { display: none; }
        @media print { body * { visibility: hidden; } .print-area, .print-area * { visibility: visible; } .print-area { position: fixed; left: 0; top: 0; width: 100%; } .no-print { display: none !important; } }
    </style>
</head>
<body class="min-h-screen">

    <div id="toast" class="hidden"></div>

    <!-- AUTHENTICATION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[3rem] p-10 space-y-8 shadow-2xl text-center">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-24 mx-auto">
            <h1 class="text-2xl font-black text-slate-900 uppercase">Portail CT241</h1>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="E-mail">
                <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="Code d'accès">
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden"></p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl">ACCÉDER AU SYSTÈME</button>
            </form>
        </div>
    </section>

    <!-- MAIN APP -->
    <main id="appContent" class="hidden pb-24">
        <nav class="bg-white/80 backdrop-blur-lg p-4 sticky top-0 z-50 border-b border-slate-100">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-8">
                    <div class="flex flex-col"><span class="text-[9px] font-black text-slate-900" id="userDisplayEmail">...</span><div id="badgeDisplay"></div></div>
                </div>
                <div class="flex items-center gap-2">
                    <button onclick="toggleFilterToday()" id="btnFilterToday" class="bg-slate-100 text-slate-400 px-4 py-2 rounded-xl font-black text-[10px] transition-all">JOUR</button>
                    <button onclick="handleLogout()" class="p-2 bg-slate-100 rounded-full text-slate-400"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m4 4H7" /></svg></button>
                </div>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            <!-- ADMIN DASH -->
            <section id="adminDash" class="role-section hidden space-y-4">
                <div class="flex justify-between items-center mb-2">
                    <h3 class="text-[10px] font-black text-slate-400 uppercase">Dashboard Performance</h3>
                    <button onclick="exportPerformance()" class="bg-emerald-600 text-white px-4 py-2 rounded-xl text-[9px] font-black shadow-lg">EXPORTER CSV</button>
                </div>
                <div class="grid grid-cols-3 gap-3">
                    <div class="bg-slate-900 p-4 rounded-3xl text-white shadow-lg text-center">
                        <span class="text-[8px] uppercase font-black text-slate-500">Total CA</span>
                        <div id="dashCaTotal" class="text-xs font-black text-emerald-400">0</div>
                    </div>
                    <div class="bg-slate-900 p-4 rounded-3xl text-white shadow-lg text-center">
                        <span class="text-[8px] uppercase font-black text-slate-500">CA Jour</span>
                        <div id="dashCaJour" class="text-xs font-black text-yellow-500">0</div>
                    </div>
                    <div class="bg-slate-900 p-4 rounded-3xl text-white shadow-lg text-center">
                        <span class="text-[8px] uppercase font-black text-slate-500">Livrées</span>
                        <div id="dashLivTotal" class="text-xs font-black text-blue-400">0</div>
                    </div>
                </div>
                <div class="bg-slate-900 p-6 rounded-[2.5rem]">
                    <div id="dashLivreursList"></div>
                </div>
            </section>

            <!-- LIVREUR STATS -->
            <section id="statLivreurSection" class="role-section hidden">
                <div class="bg-slate-900 p-6 rounded-[2.5rem] text-white space-y-4 shadow-xl">
                    <div class="flex justify-between items-end">
                        <div><p class="text-[10px] uppercase font-black text-slate-400">Livrées</p><h2 id="myCount" class="text-5xl font-black">0</h2></div>
                        <div class="text-right"><p class="text-[10px] uppercase font-black text-emerald-400">Bonus</p><h2 id="myBonus" class="text-2xl font-black text-emerald-500">0</h2></div>
                    </div>
                    <div class="w-full h-3 bg-slate-800 rounded-full overflow-hidden">
                        <div id="progBar" class="h-full bg-emerald-500 transition-all duration-700" style="width: 0%"></div>
                    </div>
                    <p id="progLabel" class="text-[9px] font-bold text-center uppercase tracking-widest text-slate-400"></p>
                </div>
            </section>

            <!-- BOUTIQUE : CRÉATION -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl border border-slate-100 overflow-hidden">
                    <div class="bg-emerald-600 p-6 text-white flex justify-between items-center">
                        <div><p class="text-[10px] font-black uppercase opacity-70">📦 Nouveau Bon</p><h2 class="font-black text-xs">ID : <span id="displayNextId">...</span></h2></div>
                    </div>
                    <div class="p-6 space-y-4">
                        <input type="text" id="expNom" placeholder="Nom Boutique / Expéditeur" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="expQuartier" placeholder="Quartier" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                            <input type="tel" id="expTel" placeholder="Tél Exp." class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        </div>
                        <input type="text" id="destNom" placeholder="Destinataire" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        <div class="grid grid-cols-2 gap-3">
                            <select id="destQuartier" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                                <option value="" disabled selected>Zone</option>
                                <option>Angondjé</option><option>Nzeng-Ayong</option><option>Oloumi</option><option>Akanda</option><option>Owendo</option><option>Pk0-12</option>
                            </select>
                            <input type="tel" id="destTel" placeholder="Tél Dest." class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        </div>
                        <div class="grid grid-cols-2 gap-3">
                            <select id="natureColis" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                                <option value="0">📦 Standard</option><option value="1">🍟 Repas</option><option value="2">📁 Docs</option><option value="3">💊 Pharma</option>
                            </select>
                            <select id="modeReglement" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                                <option value="0">Espèces</option><option value="1">Airtel Money</option><option value="2">Moov Money</option>
                            </select>
                        </div>
                        <div class="grid grid-cols-2 gap-3">
                            <input type="number" id="valeurDeclaree" placeholder="Valeur Colis" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                            <input type="number" id="fraisLivraison" placeholder="Frais Livraison" class="p-4 bg-slate-900 text-white rounded-2xl font-black text-[11px] outline-none">
                        </div>
                        <button onclick="genererMission()" class="w-full bg-emerald-600 text-white font-black py-5 rounded-3xl shadow-xl active:scale-95 transition-all text-[11px] uppercase">
                            VALIDER L'ENVOI
                        </button>
                    </div>
                </div>
            </section>

            <!-- DISPATCH -->
            <section id="rubrique2" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase mb-4 ml-2">En attente de Dispatch</h3>
                <div id="containerDispatch"></div>
            </section>

            <!-- LIVREUR -->
            <section id="rubrique3" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase mb-4 ml-2">Mes Missions</h3>
                <div id="containerLivreur"></div>
            </section>

            <!-- ARCHIVES -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-slate-900 rounded-[2.5rem] overflow-hidden shadow-2xl">
                    <div class="p-6">
                        <div class="flex justify-between items-center mb-6">
                            <h2 class="text-white font-black text-xs uppercase">Archives Digitales</h2>
                            <div id="archiveCount" class="bg-yellow-500 text-slate-900 font-black text-[9px] px-3 py-1 rounded-full">0</div>
                        </div>
                        <input type="text" oninput="handleSearch(this.value)" placeholder="Rechercher ID, Client..." class="w-full p-4 bg-white/10 rounded-2xl text-[11px] text-white font-bold outline-none border border-transparent focus:border-emerald-500">
                    </div>
                    <table class="w-full text-left">
                        <tbody id="archiveBody"></tbody>
                    </table>
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
        <div class="text-center space-y-6">
            <div class="text-6xl">📸</div>
            <h2 class="text-2xl font-black">Preuve</h2>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-5 rounded-3xl px-12 uppercase text-xs">OUVRIR APPAREIL</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 text-[10px] font-bold underline">ANNULER</button>
        </div>
    </div>

</body>
</html>
