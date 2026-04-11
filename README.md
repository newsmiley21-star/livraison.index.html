<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>CT241 - Logistique Gabon</title>
    
    <!-- Couleurs du drapeau Gabonais -->
    <meta name="theme-color" content="#009E60">
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
        import { getFirestore, collection, doc, setDoc, updateDoc, deleteDoc, onSnapshot, query, where, getDocs } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        // Configuration Firebase (Inchangée)
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
            
            btn.disabled = true; btn.innerText = "CONNEXION EN COURS...";
            err.classList.add('hidden');

            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } catch (error) { 
                err.innerText = "Accès refusé. Vérifiez vos accès.";
                err.classList.remove('hidden');
                btn.disabled = false; btn.innerText = "ACCÉDER AU SYSTÈME";
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
                    document.getElementById('loginError').innerText = "Rôle non configuré.";
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
            
            // Couleurs basées sur le rôle et le drapeau
            const colors = { admin: 'bg-slate-900', dispatch: 'bg-[#3A75C4]', relais: 'bg-[#009E60]', livreur: 'bg-[#FCD116]' };
            const textColors = { admin: 'text-white', dispatch: 'text-white', relais: 'text-white', livreur: 'text-black' };

            document.getElementById('badgeDisplay').innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-black uppercase ${textColors[userRole]} ${colors[userRole]} shadow-sm">${userRole}</span>`;

            if (userRole === 'admin') {
                document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
                document.getElementById('adminControls').classList.remove('hidden');
            }
            else if (userRole === 'relais') { document.getElementById('rubrique1').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'dispatch') { document.getElementById('rubrique2').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'livreur') { document.getElementById('rubrique3').classList.remove('hidden'); document.getElementById('statLivreurSection').classList.remove('hidden'); }
            return true;
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
                showToast("Mission enregistrée avec succès", "success");
                prepareNextId();
                document.getElementById('destNom').value = "";
                document.getElementById('dq').value = "";
                document.getElementById('dt').value = "";
                document.getElementById('fraisLivraison').value = "";
            } catch (e) { showToast("Erreur de connexion", "error"); }
        };

        window.publierMission = async (id) => {
            try {
                await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { s: 1 });
                showToast("Colis en attente de ramassage", "success");
            } catch (e) { showToast("Erreur", "error"); }
        };

        // --- VALIDATION PHOTOS ---
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
                        s: 2, 
                        le: currentUser.email, 
                        lk: new Date().toISOString().split('T')[0],
                        proof: base64
                    });
                    document.getElementById('cameraModal').classList.add('hidden');
                    showToast("Livraison confirmée 🇬🇦", "success");
                } catch (err) { showToast("Erreur validation", "error"); }
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
                if(html5QrCode) { await html5QrCode.stop(); html5QrCode = null; }
                document.getElementById('qrScannerModal').classList.add('hidden');
                triggerPhoto(id);
            } else {
                showToast("QR Code invalide", "error");
            }
        };

        // --- GPS ---
        window.openGoogleMaps = (destination) => {
            const url = `https://www.google.com/maps/dir/?api=1&destination=${encodeURIComponent(destination + ", Libreville, Gabon")}&travelmode=driving`;
            window.open(url, '_blank');
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
                const isAdmin = userRole === 'admin';

                if (m.s === 0 && (isAdmin || userRole === 'dispatch')) {
                    if(contDisp) contDisp.innerHTML += `
                        <div class="p-5 bg-white border-l-8 border-[#009E60] rounded-3xl mb-4 flex justify-between items-center shadow-lg">
                            <div>
                                <p class="text-[9px] font-black text-[#009E60] mb-1">REF: ${m.id}</p>
                                <p class="text-[12px] font-black text-slate-800">${m.dn}</p>
                                <p class="text-[10px] text-slate-400">Vers: ${m.dq}</p>
                            </div>
                            <div class="flex gap-2">
                                <button onclick="openBonImpression('${m.id}')" class="bg-slate-100 p-3 rounded-2xl text-slate-500 hover:text-[#3A75C4]"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M17 17h2a2 2 0 002-2v-4a2 2 0 00-2-2H5a2 2 0 00-2 2v4a2 2 0 00-2 2h2m2 4h6a2 2 0 002-2v-4a2 2 0 00-2-2H9a2 2 0 00-2 2v4a2 2 0 002 2zm8-12V5a2 2 0 00-2-2H9a2 2 0 00-2 2v4h10z" stroke-width="2"></path></svg></button>
                                <button onclick="publierMission('${m.id}')" class="bg-[#009E60] text-white px-6 py-3 rounded-2xl text-[10px] font-black uppercase shadow-md active:scale-95 transition-all">Lancer</button>
                            </div>
                        </div>`;
                } 
                else if (m.s === 1 && (isAdmin || userRole === 'livreur')) {
                    if(contLiv) contLiv.innerHTML += `
                        <div class="p-6 bg-white border-l-8 border-[#FCD116] rounded-[2.5rem] mb-5 shadow-xl relative overflow-hidden">
                            <div class="flex justify-between items-start mb-4">
                                <div>
                                    <span class="text-[9px] font-black text-[#3A75C4] uppercase">Mission Logistique</span>
                                    <h4 class="font-black text-slate-900 text-lg">${m.dn}</h4>
                                </div>
                                <div class="bg-[#FCD116] text-black px-3 py-1 rounded-full text-[8px] font-black">EN ATTENTE</div>
                            </div>
                            <div class="space-y-2 mb-6">
                                <p class="text-[12px] text-slate-600 font-bold">📍 Destination : ${m.dq}</p>
                                <a href="tel:${m.dt}" class="text-[12px] text-[#3A75C4] font-black flex items-center gap-2">📞 Appeler : ${m.dt}</a>
                            </div>
                            <div class="grid grid-cols-2 gap-3">
                                <button onclick="openGoogleMaps('${m.dq}')" class="bg-slate-100 text-slate-900 py-4 rounded-2xl font-black text-[10px] uppercase">Navigation</button>
                                <button onclick="openQrScanner()" class="bg-[#009E60] text-white py-4 rounded-2xl font-black text-[10px] uppercase shadow-lg shadow-emerald-200">Livrer</button>
                            </div>
                        </div>`;
                } 
                else if (m.s === 2 && (isAdmin || userRole === 'dispatch' || userRole === 'relais')) {
                    if(contArch) contArch.innerHTML += `
                        <tr class="border-b border-white/5 text-[10px]">
                            <td class="p-4 font-bold text-white">${m.id}</td>
                            <td class="p-4 text-slate-300">
                                ${m.dn}<br>
                                <span class="text-[8px] text-[#FCD116] uppercase font-black">Livreur : ${m.le ? m.le.split('@')[0] : '...'}</span>
                            </td>
                            <td class="p-4 text-right">
                                <span class="text-[#009E60] font-black">${m.p.toLocaleString()} CFA</span>
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
                    <div class="flex justify-between items-center border-b-8 border-[#009E60] pb-6 mb-8">
                        <div>
                            <h2 class="font-black text-xl tracking-tighter">CT241 LOGISTIQUE</h2>
                            <p class="text-[8px] font-bold text-emerald-600">REPUBLIQUE GABONAISE</p>
                        </div>
                        <h2 class="font-black text-[#3A75C4] text-3xl tracking-tighter">#${m.id}</h2>
                    </div>
                    <div class="grid grid-cols-2 gap-10 mb-10">
                        <div class="space-y-1"><p class="text-[9px] font-black text-slate-400 uppercase">Expéditeur</p><p class="font-black text-sm">${m.en}</p><p class="text-xs font-bold text-slate-500">${m.eq}</p></div>
                        <div class="space-y-1"><p class="text-[9px] font-black text-slate-400 uppercase">Destinataire</p><p class="font-black text-sm">${m.dn}</p><p class="text-xs font-bold text-slate-500">${m.dq}</p></div>
                    </div>
                    <div id="qrcode" class="flex justify-center my-10 p-6 bg-slate-50 rounded-[3rem] w-fit mx-auto border-4 border-slate-100"></div>
                    <div class="text-center pt-8 border-t-4 border-double border-slate-200">
                        <p class="text-[10px] font-black uppercase text-slate-400 mb-2">Montant à encaisser</p>
                        <p class="text-5xl font-black text-[#009E60]">${m.p.toLocaleString()} CFA</p>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
            setTimeout(() => {
                const qrContainer = document.getElementById("qrcode");
                if(qrContainer) new QRCode(qrContainer, { text: "CT241-"+m.id, width: 160, height: 160 });
            }, 100);
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
            el.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-8 py-4 rounded-full text-white font-black text-xs z-[600] shadow-2xl transition-all ${t==='success'?'bg-[#009E60]':'bg-red-600'}`;
            el.classList.remove('hidden');
            setTimeout(() => el.classList.add('hidden'), 3000);
        };
    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; color: #0f172a; }
        .hidden { display: none; }
        .gabon-flag-gradient { background: linear-gradient(to right, #009E60 33%, #FCD116 33% 66%, #3A75C4 66%); height: 4px; }
        @media print { .no-print { display: none; } .print-area { display: block; position: fixed; top: 0; left: 0; width: 100%; height: 100%; z-index: 9999; } }
    </style>
</head>
<body class="antialiased select-none bg-emerald-50/20">

    <div id="toast" class="hidden"></div>

    <!-- AUTHENTIFICATION STYLE GABON -->
    <section id="authSection" class="fixed inset-0 bg-white flex items-center justify-center p-8 z-[1000] overflow-hidden">
        <!-- Background Decor -->
        <div class="absolute inset-x-0 top-0 h-4 gabon-flag-gradient"></div>
        <div class="absolute w-[800px] h-[800px] bg-[#009E60]/5 rounded-full -top-96 -right-96"></div>
        <div class="absolute w-[600px] h-[600px] bg-[#3A75C4]/5 rounded-full -bottom-40 -left-40"></div>
        
        <div class="w-full max-w-sm space-y-12 relative z-10">
            <div class="text-center space-y-4">
                <div class="w-24 h-24 bg-[#FCD116] mx-auto rounded-[2.5rem] flex items-center justify-center text-4xl shadow-xl shadow-yellow-200">🇬🇦</div>
                <h1 class="text-4xl font-black tracking-tighter">CT241 <span class="text-[#3A75C4]">LOG</span></h1>
                <p class="text-[10px] text-emerald-700 font-black uppercase tracking-[0.3em]">Excellence Logistique Gabon</p>
            </div>
            
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <div class="space-y-2">
                    <label class="text-[9px] font-black uppercase text-slate-400 ml-4">Email Professionnel</label>
                    <input type="email" id="loginEmail" required class="w-full p-6 bg-slate-50 rounded-3xl text-xs font-bold border-2 border-transparent focus:border-[#009E60] outline-none transition-all" placeholder="votre.nom@ct241.ga">
                </div>
                <div class="space-y-2">
                    <label class="text-[9px] font-black uppercase text-slate-400 ml-4">Code d'accès</label>
                    <input type="password" id="loginPass" required class="w-full p-6 bg-slate-50 rounded-3xl text-xs font-bold border-2 border-transparent focus:border-[#009E60] outline-none transition-all" placeholder="••••••••">
                </div>
                <p id="loginError" class="text-[10px] text-red-500 font-black hidden text-center"></p>
                <button id="loginBtn" type="submit" class="w-full bg-[#009E60] text-white font-black py-6 rounded-3xl text-xs uppercase tracking-widest shadow-xl shadow-emerald-200 active:scale-95 transition-all">ACCÉDER AU PORTAIL</button>
            </form>
            
            <p class="text-center text-[8px] font-bold text-slate-300 uppercase tracking-widest">Opéré par CT241 Gabon</p>
        </div>
    </section>

    <!-- APP CONTENT -->
    <main id="appContent" class="hidden min-h-screen">
        <!-- HEADER NATIONAL -->
        <nav class="bg-white/90 backdrop-blur-xl sticky top-0 z-[100] border-b border-slate-100">
            <div class="h-1.5 gabon-flag-gradient"></div>
            <div class="max-w-xl mx-auto p-5 flex justify-between items-center">
                <div class="flex items-center gap-4">
                    <div class="w-12 h-12 bg-emerald-100 rounded-2xl flex items-center justify-center text-emerald-700 font-black text-lg">G</div>
                    <div>
                        <p id="userDisplayEmail" class="text-[9px] font-black text-slate-900 uppercase truncate w-32">...</p>
                        <div id="badgeDisplay"></div>
                    </div>
                </div>
                <button onclick="handleLogout()" class="w-12 h-12 flex items-center justify-center bg-red-50 rounded-2xl text-red-400 hover:text-red-600 transition-colors shadow-sm">
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M17 16l4-4m0 0l-4-4m4 4H7" stroke-width="2.5"/></svg>
                </button>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-6 space-y-8 pb-32">
            <!-- ADMIN & STATS -->
            <section id="adminControls" class="hidden bg-slate-950 p-8 rounded-[3rem] shadow-2xl space-y-6">
                <h3 class="text-white font-black text-[10px] uppercase tracking-[0.4em] opacity-50">Administration</h3>
                <div class="grid grid-cols-2 gap-4">
                    <button onclick="downloadReport()" class="bg-[#3A75C4] text-white p-6 rounded-[2rem] text-[10px] font-black uppercase shadow-lg shadow-blue-900/20">Rapport CSV</button>
                    <button onclick="resetAccounting()" class="bg-red-500 text-white p-6 rounded-[2rem] text-[10px] font-black uppercase shadow-lg shadow-red-900/20">RAZ Compta</button>
                </div>
            </section>

            <!-- Stats Livreur -->
            <section id="statLivreurSection" class="role-section hidden grid grid-cols-2 gap-4">
                <div class="bg-white p-8 rounded-[3rem] text-center shadow-xl border-b-4 border-[#009E60]">
                    <p class="text-[9px] font-black text-slate-400 uppercase mb-2 tracking-widest">Colis Livrés</p>
                    <p id="myCount" class="text-4xl font-black text-slate-900">0</p>
                </div>
                <div class="bg-white p-8 rounded-[3rem] text-center shadow-xl border-b-4 border-[#FCD116]">
                    <p class="text-[9px] font-black text-slate-400 uppercase mb-2 tracking-widest">Prime Total</p>
                    <p id="myBonus" class="text-xl font-black text-[#009E60]">0 CFA</p>
                </div>
            </section>

            <!-- Création Colis (Relais) -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[3.5rem] p-10 space-y-8 shadow-2xl border border-emerald-100">
                    <div class="flex justify-between items-center">
                        <h2 class="font-black text-lg text-emerald-950">Nouveau Colis</h2>
                        <span id="displayNextId" class="bg-[#FCD116] px-4 py-2 rounded-full text-black font-black text-[10px] shadow-sm">...</span>
                    </div>
                    
                    <div class="space-y-4">
                        <div class="space-y-2">
                            <label class="text-[9px] font-black uppercase text-slate-400 ml-4">Source</label>
                            <input type="text" id="expNom" placeholder="Nom du Point Relais" class="w-full p-6 bg-slate-50 rounded-3xl text-xs font-bold outline-none focus:ring-4 ring-emerald-50">
                        </div>
                        <div class="grid grid-cols-2 gap-4">
                            <input type="text" id="expQuartier" placeholder="Quartier" class="p-6 bg-slate-50 rounded-3xl text-xs font-bold">
                            <input type="tel" id="expTel" placeholder="Contact" class="p-6 bg-slate-50 rounded-3xl text-xs font-bold">
                        </div>
                    </div>

                    <div class="h-px bg-slate-100 mx-10"></div>

                    <div class="space-y-4">
                        <div class="space-y-2">
                            <label class="text-[9px] font-black uppercase text-slate-400 ml-4">Destinataire</label>
                            <input type="text" id="destNom" placeholder="Nom du Client" class="w-full p-6 bg-slate-50 rounded-3xl text-xs font-bold outline-none">
                        </div>
                        <div class="grid grid-cols-2 gap-4">
                            <input type="text" id="dq" placeholder="Quartier de livraison" class="p-6 bg-slate-50 rounded-3xl text-xs font-bold border-2 border-transparent focus:border-[#FCD116]">
                            <input type="tel" id="dt" placeholder="Contact client" class="p-6 bg-slate-50 rounded-3xl text-xs font-bold">
                        </div>
                    </div>

                    <div class="space-y-2">
                        <label class="text-[9px] font-black uppercase text-[#3A75C4] ml-4">Frais de livraison (CFA)</label>
                        <input type="number" id="fraisLivraison" placeholder="0" class="w-full p-8 bg-[#3A75C4] text-white rounded-[2.5rem] text-center text-3xl font-black outline-none placeholder:text-blue-300 shadow-xl shadow-blue-100">
                    </div>

                    <button onclick="genererMission()" class="w-full bg-[#009E60] text-white font-black py-7 rounded-[2.5rem] text-xs uppercase tracking-[0.2em] shadow-2xl shadow-emerald-200 active:scale-95 transition-all">Enregistrer & Créer Bordereau</button>
                </div>
            </section>

            <!-- Dispatch List -->
            <section id="rubrique2" class="role-section hidden space-y-6">
                <h3 class="text-[10px] font-black text-emerald-700 uppercase tracking-[0.3em] ml-4">En attente de ramassage</h3>
                <div id="containerDispatch" class="space-y-4"></div>
            </section>

            <!-- Livreur Missions -->
            <section id="rubrique3" class="role-section hidden space-y-6">
                <h3 class="text-[10px] font-black text-[#3A75C4] uppercase tracking-[0.3em] ml-4">Courses actives</h3>
                <div id="containerLivreur" class="space-y-4"></div>
            </section>

            <!-- Archives -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-emerald-950 rounded-[3.5rem] p-10 shadow-2xl border-b-8 border-[#3A75C4]">
                    <h3 class="text-white font-black text-[10px] uppercase mb-10 tracking-[0.4em] opacity-40">Registre des livraisons</h3>
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
            <div class="w-32 h-32 bg-[#FCD116] rounded-full flex items-center justify-center text-6xl mx-auto shadow-2xl shadow-yellow-500/20">📦</div>
            <h2 class="text-2xl font-black tracking-tight">Preuve de Livraison</h2>
            <p class="text-xs text-slate-400">Veuillez photographier le colis remis au client pour valider l'encaissement.</p>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-[#009E60] text-white font-black py-7 rounded-[2.5rem] uppercase text-xs tracking-widest shadow-2xl shadow-emerald-900">Activer la caméra</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 font-bold text-xs uppercase">Annuler</button>
        </div>
    </div>

    <!-- MODAL SCANNER QR -->
    <div id="qrScannerModal" class="fixed inset-0 bg-black/95 z-[900] hidden flex flex-col items-center justify-center p-8">
        <div class="w-full max-w-sm space-y-8 text-center">
            <h3 class="text-white font-black text-xs uppercase tracking-widest">Scannez le bordereau</h3>
            <div id="reader" class="rounded-[4rem] overflow-hidden bg-slate-900 border-8 border-[#009E60] shadow-2xl"></div>
            <button onclick="document.getElementById('qrScannerModal').classList.add('hidden'); if(html5QrCode) html5QrCode.stop();" class="w-full bg-red-600 text-white py-6 rounded-[2.5rem] font-black text-xs uppercase">Fermer le scanner</button>
        </div>
    </div>

    <!-- MODAL DÉTAIL / IMPRESSION -->
    <div id="detailModal" class="fixed inset-0 bg-emerald-950/80 z-[500] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-md bg-white rounded-[4rem] overflow-hidden shadow-2xl">
            <div id="detailContent"></div>
            <div class="p-8 bg-slate-50 flex gap-4 no-print">
                <button onclick="window.print()" class="flex-[2] bg-emerald-800 text-white font-black py-6 rounded-[2.5rem] text-xs uppercase shadow-xl">Imprimer Bordereau</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="flex-1 bg-white border border-slate-200 text-slate-400 font-black py-6 rounded-[2.5rem] text-xs uppercase">Fermer</button>
            </div>
        </div>
    </div>

</body>
</html>
