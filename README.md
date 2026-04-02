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
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">

    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
    <script src="https://unpkg.com/html5-qrcode"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query, where } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

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

        // --- AUTHENTIFICATION ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value.trim().toLowerCase();
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            const err = document.getElementById('loginError');
            
            btn.disabled = true; btn.innerText = "Vérification...";
            err.classList.add('hidden');

            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } catch (error) { 
                err.innerText = "Identifiants invalides.";
                err.classList.remove('hidden');
                btn.disabled = false; btn.innerText = "ACCÉDER AU PORTAIL";
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
                    document.getElementById('loginError').innerText = "Rôle non autorisé.";
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
            const colors = { admin: 'bg-red-600', dispatch: 'bg-blue-600', relais: 'bg-emerald-600', livreur: 'bg-amber-600' };
            document.getElementById('badgeDisplay').innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-black uppercase text-white ${colors[userRole]}">${userRole}</span>`;

            if (userRole === 'admin') document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
            else if (userRole === 'relais') { document.getElementById('rubrique1').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'dispatch') { document.getElementById('rubrique2').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'livreur') { document.getElementById('rubrique3').classList.remove('hidden'); document.getElementById('statLivreurSection').classList.remove('hidden'); }
            return true;
        };

        // --- GPS & MAPS ---
        window.openGoogleMaps = (destination) => {
            const encodedAddr = encodeURIComponent(destination + ", Libreville");
            const url = `https://www.google.com/maps/dir/?api=1&destination=${encodedAddr}&travelmode=driving`;
            window.open(url, '_blank');
        };

        // --- GESTION DES DONNÉES ---
        const startListeners = () => {
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'missions'), (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
                renderStats();
            });
        };

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
                n: 1, // Nature standard
                p: parseFloat(document.getElementById('fraisLivraison').value) || 0,
            };
            if (!fields.en || !fields.dt || !fields.p) return showToast("Informations manquantes", "error");

            try {
                const now = Date.now();
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId), {
                    ...fields, id: window.nextId, s: 0, ca: now, cad: new Date(now).toISOString().split('T')[0], ce: currentUser.email
                });
                showToast("Mission enregistrée", "success");
                prepareNextId();
            } catch (e) { showToast("Erreur Firestore", "error"); }
        };

        window.publierMission = async (id) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { s: 1 });
            showToast("Mission envoyée aux livreurs", "success");
        };

        // --- PHOTO & VALIDATION ---
        window.triggerPhoto = (id) => {
            currentMissionForPhoto = id;
            document.getElementById('cameraModal').classList.remove('hidden');
        };

        window.processImage = async (file) => {
            if (!file) return;
            const reader = new FileReader();
            reader.onload = async (e) => {
                const base64 = e.target.result;
                await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionForPhoto), {
                    s: 2, 
                    le: currentUser.email, 
                    lk: new Date().toISOString().split('T')[0],
                    proof: base64
                });
                document.getElementById('cameraModal').classList.add('hidden');
                showToast("Livraison validée avec photo", "success");
            };
            reader.readAsDataURL(file);
        };

        // --- SCANNER QR ---
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
                if(html5QrCode) html5QrCode.stop();
                document.getElementById('qrScannerModal').classList.add('hidden');
                // On demande la photo après le scan réussi
                triggerPhoto(id);
            } else {
                showToast("Bordereau invalide", "error");
            }
        };

        // --- RENDU UI ---
        window.renderUI = () => {
            const contDisp = document.getElementById('containerDispatch');
            const contLiv = document.getElementById('containerLivreur');
            const contArch = document.getElementById('archiveBody');
            
            if(contDisp) contDisp.innerHTML = "";
            if(contLiv) contLiv.innerHTML = "";
            if(contArch) contArch.innerHTML = "";

            allMissions.sort((a,b) => b.ca - a.ca).forEach(m => {
                if (m.s === 0 && (userRole === 'admin' || userRole === 'dispatch')) {
                    contDisp.innerHTML += `
                        <div class="p-5 bg-white border border-slate-100 rounded-3xl mb-4 flex justify-between items-center shadow-sm">
                            <div>
                                <p class="text-[9px] font-black text-blue-600 mb-1">#${m.id}</p>
                                <p class="text-[11px] font-black text-slate-900">${m.dn}</p>
                                <p class="text-[10px] text-slate-400 font-bold">${m.dq}</p>
                            </div>
                            <div class="flex gap-2">
                                <button onclick="openBonImpression('${m.id}')" class="bg-slate-50 p-3 rounded-2xl text-slate-400"><svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M17 17h2a2 2 0 002-2v-4a2 2 0 00-2-2H5a2 2 0 00-2 2v4a2 2 0 002 2h2m2 4h6a2 2 0 002-2v-4a2 2 0 00-2-2H9a2 2 0 00-2 2v4a2 2 0 002 2zm8-12V5a2 2 0 00-2-2H9a2 2 0 00-2 2v4h10z"></path></svg></button>
                                <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white px-5 py-3 rounded-2xl text-[10px] font-black uppercase tracking-widest">Assigner</button>
                            </div>
                        </div>`;
                } else if (m.s === 1 && (userRole === 'admin' || userRole === 'livreur')) {
                    contLiv.innerHTML += `
                        <div class="p-6 bg-white border-l-8 border-amber-400 rounded-[2rem] mb-5 shadow-lg">
                            <div class="flex justify-between items-start mb-4">
                                <div>
                                    <span class="text-[10px] font-black text-slate-300">#${m.id}</span>
                                    <h4 class="font-black text-slate-900 text-base leading-tight">${m.dn}</h4>
                                </div>
                                <div class="bg-amber-100 text-amber-700 px-3 py-1 rounded-full text-[8px] font-black uppercase">En cours</div>
                            </div>
                            <div class="space-y-2 mb-6">
                                <p class="text-[11px] text-slate-500 font-bold flex items-center gap-2">📍 ${m.dq}</p>
                                <a href="tel:${m.dt}" class="text-[11px] text-blue-600 font-black flex items-center gap-2 italic">📞 Appeler le client (${m.dt})</a>
                            </div>
                            <div class="grid grid-cols-2 gap-3">
                                <button onclick="openGoogleMaps('${m.dq}')" class="bg-slate-50 text-slate-900 py-4 rounded-2xl font-black text-[10px] uppercase tracking-tighter flex items-center justify-center gap-2">🗺️ Itinéraire</button>
                                <button onclick="openQrScanner()" class="bg-slate-900 text-white py-4 rounded-2xl font-black text-[10px] uppercase tracking-tighter flex items-center justify-center gap-2 shadow-xl shadow-slate-200">📸 Scanner & Livrer</button>
                            </div>
                        </div>`;
                } else if (m.s === 2) {
                    contArch.innerHTML += `
                        <tr class="border-b border-slate-800 text-[10px]">
                            <td class="p-4 font-bold text-white">${m.id}</td>
                            <td class="p-4 text-slate-300">
                                ${m.dn}<br>
                                <span class="text-[8px] text-slate-500">Livré par: ${m.le.split('@')[0]}</span>
                            </td>
                            <td class="p-4 text-right text-emerald-400 font-black">${m.p.toLocaleString()} CFA</td>
                        </tr>`;
                }
            });
        };

        window.openBonImpression = (id) => {
            const m = allMissions.find(x => x.id === id);
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area p-10 bg-white text-slate-900">
                    <div class="flex justify-between items-center border-b-4 border-slate-900 pb-6 mb-8">
                        <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-12 rounded-xl">
                        <div class="text-right">
                            <h2 class="font-black text-sm tracking-widest">BORDEREAU CT241</h2>
                            <h2 class="font-black text-blue-600 text-2xl tracking-tighter">#${m.id}</h2>
                        </div>
                    </div>
                    <div class="grid grid-cols-2 gap-10 mb-10">
                        <div class="space-y-1"><p class="text-[10px] font-black text-slate-300 uppercase">Expéditeur</p><p class="font-black text-sm">${m.en}</p><p class="text-xs font-bold text-slate-500">${m.eq} - ${m.et}</p></div>
                        <div class="space-y-1"><p class="text-[10px] font-black text-slate-300 uppercase">Destinataire</p><p class="font-black text-sm">${m.dn}</p><p class="text-xs font-bold text-slate-500">${m.dq} - ${m.dt}</p></div>
                    </div>
                    <div id="qrcode" class="flex justify-center my-10 p-6 bg-slate-50 rounded-[2.5rem] w-fit mx-auto border-2 border-slate-100"></div>
                    <div class="text-center pt-8 border-t-2 border-dashed border-slate-200">
                        <p class="text-[10px] font-black uppercase text-slate-400 mb-2 tracking-[0.2em]">Net à percevoir</p>
                        <p class="text-4xl font-black text-slate-900">${m.p.toLocaleString()} CFA</p>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
            setTimeout(() => new QRCode(document.getElementById("qrcode"), { text: "CT241-"+m.id, width: 160, height: 160 }), 100);
        };

        const renderStats = () => {
            let count = 0, bonus = 0;
            allMissions.forEach(m => { if(m.s === 2 && m.le === currentUser.email) { count++; if(count > 17) bonus += 700; } });
            if(document.getElementById('myCount')) document.getElementById('myCount').innerText = count;
            if(document.getElementById('myBonus')) document.getElementById('myBonus').innerText = bonus + ' CFA';
        };

        window.showToast = (m, t) => {
            const el = document.getElementById('toast');
            el.innerText = m;
            el.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-8 py-4 rounded-3xl text-white font-black text-xs z-[600] shadow-2xl transition-all ${t==='success'?'bg-emerald-600 shadow-emerald-200':'bg-red-600 shadow-red-200'}`;
            el.classList.remove('hidden');
            setTimeout(() => el.classList.add('hidden'), 3000);
        };
    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; color: #0f172a; }
        .hidden { display: none; }
        @media print { .no-print { display: none; } .print-area { display: block; position: fixed; top: 0; left: 0; width: 100%; height: 100%; z-index: 9999; } }
    </style>
</head>
<body class="antialiased select-none">

    <div id="toast" class="hidden"></div>

    <!-- CONNEXION -->
    <section id="authSection" class="fixed inset-0 bg-[#0f172a] flex items-center justify-center p-8 z-[1000]">
        <div class="w-full max-w-sm bg-white rounded-[3.5rem] p-12 space-y-10 text-center shadow-2xl">
            <div class="space-y-4">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-24 mx-auto rounded-[2rem] shadow-xl">
                <h1 class="text-3xl font-black tracking-tighter">CT241</h1>
                <p class="text-[10px] text-slate-400 font-bold uppercase tracking-widest">Portail de Service</p>
            </div>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" required class="w-full p-6 bg-slate-50 rounded-3xl text-xs font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all" placeholder="E-mail Agent">
                <input type="password" id="loginPass" required class="w-full p-6 bg-slate-50 rounded-3xl text-xs font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all" placeholder="Mot de passe">
                <p id="loginError" class="text-[10px] text-red-500 font-black hidden"></p>
                <button id="loginBtn" type="submit" class="w-full bg-[#0f172a] text-white font-black py-6 rounded-3xl text-xs uppercase tracking-[0.2em] shadow-xl hover:scale-[1.02] active:scale-95 transition-all">Connexion</button>
            </form>
        </div>
    </section>

    <!-- APP CONTENT -->
    <main id="appContent" class="hidden min-h-screen">
        <!-- TOP NAV -->
        <nav class="bg-white/80 backdrop-blur-xl sticky top-0 z-[100] p-5 border-b border-slate-100">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-4">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-10 rounded-xl shadow-lg shadow-slate-200">
                    <div>
                        <p id="userDisplayEmail" class="text-[9px] font-black text-slate-300 uppercase tracking-tight truncate w-32">...</p>
                        <div id="badgeDisplay"></div>
                    </div>
                </div>
                <button onclick="handleLogout()" class="w-12 h-12 flex items-center justify-center bg-slate-50 rounded-2xl text-slate-400 hover:text-red-500 transition-colors">
                    <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M17 16l4-4m0 0l-4-4m4 4H7" stroke-width="2.5"/></svg>
                </button>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-6 space-y-8">
            <!-- Stats Livreur -->
            <section id="statLivreurSection" class="role-section hidden grid grid-cols-2 gap-4">
                <div class="bg-white p-7 rounded-[2.5rem] text-center shadow-sm border border-slate-50">
                    <p class="text-[9px] font-black text-slate-300 uppercase mb-2 tracking-widest">Courses</p>
                    <p id="myCount" class="text-4xl font-black">0</p>
                </div>
                <div class="bg-white p-7 rounded-[2.5rem] text-center shadow-sm border border-slate-50">
                    <p class="text-[9px] font-black text-slate-300 uppercase mb-2 tracking-widest">Bonus</p>
                    <p id="myBonus" class="text-2xl font-black text-emerald-500">0 CFA</p>
                </div>
            </section>

            <!-- Création (Relais) -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[3rem] p-10 space-y-6 shadow-2xl shadow-slate-200 border border-slate-100">
                    <div class="flex justify-between items-center">
                        <h2 class="font-black text-sm uppercase tracking-tighter">Nouveau Colis</h2>
                        <span id="displayNextId" class="text-blue-600 font-black text-xs">...</span>
                    </div>
                    <div class="space-y-3">
                        <input type="text" id="expNom" placeholder="Boutique / Point Relais" class="w-full p-5 bg-slate-50 rounded-2xl text-xs font-bold outline-none border-2 border-transparent focus:border-blue-50">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="expQuartier" placeholder="Quartier Exp." class="p-5 bg-slate-50 rounded-2xl text-xs font-bold">
                            <input type="tel" id="expTel" placeholder="Tél Exp." class="p-5 bg-slate-50 rounded-2xl text-xs font-bold">
                        </div>
                    </div>
                    <div class="h-px bg-slate-100 mx-2"></div>
                    <div class="space-y-3">
                        <input type="text" id="destNom" placeholder="Nom du Client" class="w-full p-5 bg-slate-50 rounded-2xl text-xs font-bold">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="dq" placeholder="Quartier Dest." class="p-5 bg-slate-50 rounded-2xl text-xs font-bold">
                            <input type="tel" id="dt" placeholder="Tél Client" class="p-5 bg-slate-50 rounded-2xl text-xs font-bold">
                        </div>
                    </div>
                    <input type="number" id="fraisLivraison" placeholder="MONTANT À PERCEVOIR (CFA)" class="w-full p-6 bg-blue-600 text-white rounded-[2rem] text-center text-lg font-black placeholder:text-blue-300 outline-none shadow-xl shadow-blue-100">
                    <button onclick="genererMission()" class="w-full bg-[#0f172a] text-white font-black py-6 rounded-[2rem] text-xs uppercase tracking-widest hover:bg-black transition-all">Enregistrer Bordereau</button>
                </div>
            </section>

            <!-- Dispatch -->
            <section id="rubrique2" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-5 ml-4">Prêt pour Expédition</h3>
                <div id="containerDispatch"></div>
            </section>

            <!-- Livreur Missions -->
            <section id="rubrique3" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-5 ml-4">Missions Actuelles</h3>
                <div id="containerLivreur"></div>
            </section>

            <!-- Archives -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-slate-900 rounded-[3rem] p-10 shadow-2xl overflow-hidden">
                    <h3 class="text-white font-black text-[11px] uppercase mb-8 tracking-[0.3em] opacity-50">Journal CT241</h3>
                    <div class="overflow-x-auto">
                        <table class="w-full text-left">
                            <tbody id="archiveBody"></tbody>
                        </table>
                    </div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL CAMERA -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[1000] hidden flex flex-col items-center justify-center p-8 text-white">
        <div class="text-center space-y-8 max-w-xs">
            <div class="w-28 h-28 bg-white/10 rounded-full flex items-center justify-center text-6xl mx-auto">📸</div>
            <h2 class="text-2xl font-black tracking-tight">Preuve de Livraison</h2>
            <p class="text-slate-500 text-xs font-bold">Prenez une photo claire du bordereau signé ou du colis remis au client.</p>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-6 rounded-[2rem] shadow-2xl uppercase text-xs tracking-widest">Ouvrir l'appareil photo</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-400 font-bold text-xs uppercase tracking-widest">Plus tard</button>
        </div>
    </div>

    <!-- MODAL SCANNER QR -->
    <div id="qrScannerModal" class="fixed inset-0 bg-black/95 z-[900] hidden flex flex-col items-center justify-center p-8">
        <div class="w-full max-w-sm space-y-8">
            <h2 class="text-xl font-black text-white text-center uppercase tracking-widest">Validation par QR</h2>
            <div id="reader" class="rounded-[3rem] overflow-hidden bg-slate-800 border-4 border-white/5 shadow-2xl"></div>
            <button onclick="document.getElementById('qrScannerModal').classList.add('hidden'); if(html5QrCode) html5QrCode.stop();" class="w-full bg-red-600 text-white py-6 rounded-[2rem] font-black text-xs uppercase tracking-widest">Annuler le Scan</button>
        </div>
    </div>

    <!-- MODAL DÉTAIL / IMPRESSION -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/80 z-[500] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-md bg-white rounded-[3.5rem] overflow-hidden shadow-2xl">
            <div id="detailContent"></div>
            <div class="p-8 bg-slate-50 flex gap-4 no-print border-t border-slate-100">
                <button onclick="window.print()" class="flex-[2] bg-slate-900 text-white font-black py-5 rounded-3xl text-xs uppercase tracking-widest hover:bg-black transition-all">🖨️ Imprimer</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="flex-1 bg-white border border-slate-200 text-slate-500 font-black py-5 rounded-3xl text-xs uppercase tracking-widest">Fermer</button>
            </div>
        </div>
    </div>

</body>
</html>
