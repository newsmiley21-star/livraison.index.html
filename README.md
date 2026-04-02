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
    
    <!-- Scripts UI, QR Generation & QR Scanning -->
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
        let html5QrCode = null;

        // --- AUTH & ROLE LOGIC (FIXED) ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value.trim().toLowerCase();
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            const err = document.getElementById('loginError');
            
            btn.disabled = true;
            btn.innerText = "Vérification...";
            err.classList.add('hidden');

            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } catch (error) { 
                err.innerText = "Email ou mot de passe incorrect.";
                err.classList.remove('hidden');
                btn.disabled = false; btn.innerText = "ACCÉDER AU PORTAIL";
            }
        };

        window.handleLogout = async () => { await signOut(auth); location.reload(); };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                const roleIdentified = assignRole(user.email);
                if (roleIdentified) {
                    document.getElementById('authSection').classList.add('hidden');
                    document.getElementById('appContent').classList.remove('hidden');
                    document.getElementById('userDisplayEmail').innerText = user.email;
                    startListeners();
                    prepareNextId();
                } else {
                    signOut(auth);
                    const err = document.getElementById('loginError');
                    err.innerText = "Erreur : Votre email n'est associé à aucun rôle (livreur, admin, etc.)";
                    err.classList.remove('hidden');
                }
            } else {
                document.getElementById('authSection').classList.remove('hidden');
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
            const colors = { admin: 'bg-red-600', dispatch: 'bg-blue-600', relais: 'bg-emerald-600', livreur: 'bg-amber-600' };
            document.getElementById('badgeDisplay').innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-black uppercase text-white ${colors[userRole]}">${userRole}</span>`;

            if (userRole === 'admin') document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
            else if (userRole === 'relais') { document.getElementById('rubrique1').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'dispatch') { document.getElementById('rubrique2').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'livreur') { document.getElementById('rubrique3').classList.remove('hidden'); document.getElementById('statLivreurSection').classList.remove('hidden'); }
            return true;
        };

        // --- MISSION MANAGEMENT ---
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
                dq: document.getElementById('dq').value,
                dt: document.getElementById('dt').value,
                n: parseInt(document.getElementById('natureColis').value),
                p: parseFloat(document.getElementById('fraisLivraison').value) || 0,
                r: 0, // Default mode
                v: 0 // Default value
            };
            if (!fields.en || !fields.dt || !fields.p) return showToast("Champs obligatoires !", "error");

            try {
                const now = Date.now();
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId), {
                    ...fields, id: window.nextId, s: 0, ca: now, cad: new Date(now).toISOString().split('T')[0], ce: currentUser.email
                });
                showToast("Mission créée avec succès", "success");
                prepareNextId();
            } catch (e) { showToast("Erreur lors de la création", "error"); }
        };

        const startListeners = () => {
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'missions'), (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
                renderStats();
            });
        };

        // --- QR SCANNER LOGIC ---
        window.openQrScanner = () => {
            document.getElementById('qrScannerModal').classList.remove('hidden');
            html5QrCode = new Html5Qrcode("reader");
            const config = { fps: 10, qrbox: { width: 250, height: 250 } };
            html5QrCode.start({ facingMode: "environment" }, config, (decodedText) => {
                const missionId = decodedText.replace('CT241-', '');
                handleValidation(missionId);
            }).catch(err => {
                showToast("Impossible d'accéder à la caméra", "error");
                closeQrScanner();
            });
        };

        window.closeQrScanner = () => {
            if (html5QrCode) {
                html5QrCode.stop().then(() => {
                    document.getElementById('qrScannerModal').classList.add('hidden');
                }).catch(() => document.getElementById('qrScannerModal').classList.add('hidden'));
            } else {
                document.getElementById('qrScannerModal').classList.add('hidden');
            }
        };

        const handleValidation = async (id) => {
            const m = allMissions.find(x => x.id === id);
            if (m && m.s === 1) {
                await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), {
                    s: 2, le: currentUser.email, lk: new Date().toISOString().split('T')[0]
                });
                showToast("Livraison Validée !", "success");
                closeQrScanner();
                if(!document.getElementById('cameraModal').classList.contains('hidden')) {
                    document.getElementById('cameraModal').classList.add('hidden');
                }
            } else {
                showToast("Code QR non valide pour cette étape", "error");
            }
        };

        // --- UI RENDERING ---
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
                        <div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 flex justify-between items-center shadow-sm">
                            <div class="text-[11px] font-bold"><span class="text-blue-600">${m.id}</span><br>${m.dn} (${m.dq})</div>
                            <div class="flex gap-2">
                                <button onclick="openBonImpression('${m.id}')" class="bg-slate-100 text-slate-600 px-3 py-2 rounded-xl text-[9px] font-black uppercase">Voir Bon</button>
                                <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white px-4 py-2 rounded-xl text-[10px] font-black">EXPÉDIER</button>
                            </div>
                        </div>`;
                } else if (m.s === 1 && (userRole === 'admin' || userRole === 'livreur')) {
                    if(contLiv) contLiv.innerHTML += `
                        <div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 shadow-sm">
                            <div class="flex justify-between font-black mb-3"><span>${m.id}</span><span class="text-amber-600 text-[9px] uppercase tracking-widest">En cours</span></div>
                            <p class="text-[11px] mb-5 text-amber-900"><b>Lieu :</b> ${m.dq}<br><b>Client :</b> ${m.dn} (${m.dt})</p>
                            <div class="space-y-2">
                                <button onclick="openQrScanner()" class="w-full bg-slate-900 text-white py-4 rounded-2xl font-black text-[11px] flex items-center justify-center gap-2 shadow-lg">
                                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v1m6 11h2m-6 0h-2v4m0-11v3m0 0h3m-3 3h3m5 1.412V17a1 1 0 01-1 1h-2.586l-2.828 2.828A1 1 0 0110 20.414V18H7a1 1 0 01-1-1v-2.586l-2.828-2.828A1 1 0 013.586 10H6V7a1 1 0 011-1h2.586l2.828-2.828A1 1 0 0114 3.586V6h3a1 1 0 011 1v2.586l2.828 2.828z" /></svg>
                                    SCANNER POUR VALIDER
                                </button>
                                <button onclick="openCamera('${m.id}')" class="w-full border-2 border-amber-200 text-amber-700 py-3 rounded-2xl font-bold text-[10px]">ECHEC SCAN ? PHOTO PREUVE</button>
                            </div>
                        </div>`;
                } else if (m.s === 2) {
                    if(contArch) contArch.innerHTML += `<tr class="border-b border-slate-800 text-[10px]"><td class="p-4 font-bold text-white">${m.id}</td><td class="p-4 text-slate-300">${m.dn}</td><td class="p-4 text-right text-emerald-400 font-black">${m.p.toLocaleString()} CFA</td></tr>`;
                }
            });
        };

        window.publierMission = async (id) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { s: 1 });
            showToast("Mission transmise aux livreurs", "success");
        };

        window.openBonImpression = (id) => {
            const m = allMissions.find(x => x.id === id);
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area p-8 bg-white text-slate-900 min-h-[500px]">
                    <div class="flex justify-between border-b-4 border-slate-900 pb-6 mb-6">
                        <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-16">
                        <div class="text-right"><h1 class="text-2xl font-black uppercase">Bordereau CT241</h1><p class="font-bold text-blue-600">ID: ${m.id}</p></div>
                    </div>
                    <div class="grid grid-cols-2 gap-8 mb-8">
                        <div><p class="text-[9px] font-black text-slate-400 uppercase mb-2">Expéditeur</p><p class="font-black text-sm">${m.en}</p><p class="text-xs">${m.eq}</p></div>
                        <div><p class="text-[9px] font-black text-slate-400 uppercase mb-2">Destinataire</p><p class="font-black text-sm">${m.dn}</p><p class="text-xs">${m.dq}</p><p class="font-bold mt-1">${m.dt}</p></div>
                    </div>
                    <div class="flex justify-between items-end border-t border-slate-100 pt-6">
                         <div id="qrcode" class="p-2 border-2 border-slate-100 rounded-xl"></div>
                         <div class="text-right"><p class="text-[10px] font-bold text-slate-400 uppercase">Montant à collecter</p><p class="text-3xl font-black">${m.p.toLocaleString()} CFA</p></div>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
            setTimeout(() => {
                new QRCode(document.getElementById("qrcode"), { text: "CT241-" + m.id, width: 100, height: 100 });
            }, 100);
        };

        window.openCamera = (id) => { currentMissionId = id; document.getElementById('cameraModal').classList.remove('hidden'); };
        
        window.processImage = (file) => {
            if(!file) return;
            const reader = new FileReader();
            reader.onload = async (e) => {
                await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId), {
                    s: 2, le: currentUser.email, lk: new Date().toISOString().split('T')[0], img: e.target.result
                });
                document.getElementById('cameraModal').classList.add('hidden');
                showToast("Livraison Validée par Photo", "success");
            };
            reader.readAsDataURL(file);
        };

        const renderStats = () => {
            const stats = { count: 0, bonus: 0 };
            const today = new Date().toISOString().split('T')[0];
            allMissions.forEach(m => {
                if (m.s === 2 && m.le === currentUser.email) {
                    stats.count++;
                    if (stats.count > 17) stats.bonus += 700;
                }
            });
            const mC = document.getElementById('myCount');
            const mB = document.getElementById('myBonus');
            if(mC) mC.innerText = stats.count;
            if(mB) mB.innerText = stats.bonus + ' CFA';
        };

        window.showToast = (m, t) => {
            const el = document.getElementById('toast');
            el.innerText = m;
            el.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-[10px] z-[600] shadow-2xl ${t==='success'?'bg-emerald-600':'bg-red-600'}`;
            el.classList.remove('hidden');
            setTimeout(() => el.classList.add('hidden'), 3000);
        };

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; }
        .hidden { display: none; }
        @media print { 
            body * { visibility: hidden; } 
            .print-area, .print-area * { visibility: visible; } 
            .print-area { position: absolute; left: 0; top: 0; width: 100%; border: none !important; } 
            .no-print { display: none !important; }
        }
    </style>
</head>
<body class="min-h-screen overflow-x-hidden">

    <div id="toast" class="hidden"></div>

    <!-- AUTH SECTION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[3rem] p-10 space-y-8 shadow-2xl text-center">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-24 mx-auto">
            <h1 class="text-2xl font-black text-slate-900 uppercase">CT241 PORTAIL</h1>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="Email (ex: livreur1@ct241.com)">
                <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="Mot de passe">
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden"></p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl hover:bg-black transition-colors">SE CONNECTER</button>
            </form>
            <p class="text-[8px] text-slate-400 font-black uppercase">Accès restreint aux agents CT241</p>
        </div>
    </section>

    <!-- MAIN APP -->
    <main id="appContent" class="hidden">
        <nav class="bg-white/80 backdrop-blur-md p-4 sticky top-0 z-50 border-b border-slate-100">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-8">
                    <div class="flex flex-col"><span class="text-[9px] font-black text-slate-900" id="userDisplayEmail">...</span><div id="badgeDisplay"></div></div>
                </div>
                <button onclick="handleLogout()" class="p-2 bg-slate-100 rounded-full text-slate-400 hover:text-red-500"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m0 0l-4-4m4 4H7" /></svg></button>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            <!-- Stats Livreur -->
            <section id="statLivreurSection" class="role-section hidden grid grid-cols-2 gap-3">
                <div class="bg-white p-5 rounded-[2rem] shadow-sm border border-slate-100 text-center"><span class="text-[9px] font-black text-slate-400 uppercase">Livraisons</span><div id="myCount" class="text-2xl font-black text-slate-900">0</div></div>
                <div class="bg-white p-5 rounded-[2rem] shadow-sm border border-slate-100 text-center"><span class="text-[9px] font-black text-slate-400 uppercase">Bonus Cumulé</span><div id="myBonus" class="text-xl font-black text-emerald-600">0</div></div>
            </section>

            <!-- Création Bordereau -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl p-8 space-y-5 border border-slate-100">
                    <h2 class="font-black text-sm uppercase tracking-tight">Nouveau Bordereau <span id="displayNextId" class="text-blue-600 ml-2">...</span></h2>
                    <input type="text" id="expNom" placeholder="Nom Boutique" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                    <div class="grid grid-cols-2 gap-3">
                        <input type="text" id="expQuartier" placeholder="Zone Expédition" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        <input type="tel" id="expTel" placeholder="Tél Boutique" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                    </div>
                    <hr class="border-slate-50 my-2">
                    <input type="text" id="destNom" placeholder="Nom du Client" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none border-2 border-transparent focus:border-blue-600">
                    <div class="grid grid-cols-2 gap-3">
                        <input type="text" id="dq" placeholder="Quartier Client" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                        <input type="tel" id="dt" placeholder="Tél Client" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                    </div>
                    <div class="grid grid-cols-2 gap-3">
                        <select id="natureColis" class="p-4 bg-slate-900 text-white rounded-2xl font-black text-[10px]"><option value="0">📦 STANDARD</option></select>
                        <input type="number" id="fraisLivraison" placeholder="PRIX LIVRAISON" class="p-4 bg-blue-600 text-white rounded-2xl font-black text-[11px] outline-none placeholder:text-blue-200">
                    </div>
                    <button onclick="genererMission()" class="w-full bg-slate-900 text-white font-black py-5 rounded-3xl text-[11px] uppercase tracking-widest shadow-xl">Enregistrer le Colis</button>
                </div>
            </section>

            <section id="rubrique2" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-4 ml-2">À Dispatcher</h3>
                <div id="containerDispatch"></div>
            </section>

            <section id="rubrique3" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-4 ml-2">Mes Livraisons Actives</h3>
                <div id="containerLivreur"></div>
            </section>

            <section id="rubrique4" class="role-section hidden">
                <div class="bg-slate-900 rounded-[2.5rem] overflow-hidden shadow-2xl p-6">
                    <h3 class="text-white font-black text-xs uppercase mb-4">Historique des Livraisons</h3>
                    <div class="overflow-x-auto">
                        <table class="w-full text-left">
                            <tbody id="archiveBody"></tbody>
                        </table>
                    </div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL SCANNER QR -->
    <div id="qrScannerModal" class="fixed inset-0 bg-black/95 z-[500] hidden flex flex-col items-center justify-center p-6 text-white">
        <div class="w-full max-w-sm space-y-6">
            <h2 class="text-xl font-black text-center uppercase tracking-widest">Scanner le Code QR</h2>
            <div id="reader" class="rounded-[2rem] overflow-hidden bg-slate-800 aspect-square border-4 border-white/10"></div>
            <button onclick="closeQrScanner()" class="w-full bg-red-600 py-5 rounded-3xl font-black text-[11px] uppercase">Annuler le Scan</button>
        </div>
    </div>

    <!-- MODAL DÉTAIL / BORDEREAU -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/90 z-[400] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-2xl bg-white rounded-[3rem] overflow-hidden shadow-2xl">
            <div id="detailContent"></div>
            <div class="p-6 bg-slate-50 flex gap-3 no-print">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-4 rounded-2xl text-[11px] uppercase">Imprimer le Bon</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-8 bg-white border border-slate-200 text-slate-500 font-bold py-4 rounded-2xl text-[11px]">Fermer</button>
            </div>
        </div>
    </div>

    <!-- MODAL CAMERA PREUVE -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[300] hidden flex flex-col items-center justify-center p-8 text-white">
        <div class="text-center space-y-8 max-w-xs">
            <div class="w-20 h-20 bg-white/10 rounded-full flex items-center justify-center text-4xl mx-auto">📸</div>
            <h2 class="text-2xl font-black">Preuve Manuelle</h2>
            <p class="text-slate-500 text-xs italic">Si le QR est illisible, prenez une photo du bordereau signé ou du colis devant le client.</p>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-5 rounded-3xl uppercase text-[11px] tracking-widest">Prendre la Photo</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-600 text-[10px] font-bold underline">Annuler</button>
        </div>
    </div>

</body>
</html>
