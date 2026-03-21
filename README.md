<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>CT241 - Logistique & Performance</title>
    
    <!-- PWA & Mobile Meta -->
    <meta name="theme-color" content="#0f172a">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <script src="https://cdn.tailwindcss.com"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query, where, limit, orderBy } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        // CONFIGURATION FIREBASE
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
        let activeMissions = []; // Uniquement les missions récentes
        let currentMissionId = null;

        // --- AUTHENTIFICATION ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            btn.disabled = true;
            try {
                await signInWithEmailAndPassword(auth, email, pass);
            } catch (err) {
                document.getElementById('loginError').innerText = "Accès refusé.";
                document.getElementById('loginError').classList.remove('hidden');
                btn.disabled = false;
            }
        };

        window.handleLogout = async () => { await signOut(auth); location.reload(); };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                assignRole(user.email);
                document.getElementById('authSection').classList.add('hidden');
                document.getElementById('appContent').classList.remove('hidden');
                startOptimizedListener(); // Lancer l'écouteur économe
                prepareNextId();
            } else {
                document.getElementById('authSection').classList.remove('hidden');
                document.getElementById('appContent').classList.add('hidden');
            }
        });

        const assignRole = (email) => {
            const e = email.toLowerCase();
            if (e.includes('admin')) userRole = 'admin';
            else if (e.includes('livreur')) userRole = 'livreur';
            else userRole = 'relais';
            
            // Affichage badges
            const badge = document.getElementById('badgeDisplay');
            const colors = { admin: 'bg-red-500', livreur: 'bg-amber-500', relais: 'bg-emerald-500' };
            badge.innerHTML = `<span class="${colors[userRole]} text-white text-[10px] px-2 py-0.5 rounded-full font-bold uppercase">${userRole}</span>`;
            
            // Filtrage visuel des rubriques
            document.querySelectorAll('.role-view').forEach(v => v.classList.add('hidden'));
            if(userRole === 'admin') document.querySelectorAll('.role-view').forEach(v => v.classList.remove('hidden'));
            else if(userRole === 'livreur') document.getElementById('viewLivreur').classList.remove('hidden');
            else document.getElementById('viewRelais').classList.remove('hidden');
        };

        // --- ÉCOUTEUR OPTIMISÉ (RÉDUCTION TÉLÉCHARGEMENTS) ---
        const startOptimizedListener = () => {
            // CONSEIL APPLIQUÉ : On ne télécharge que les missions des dernières 24H
            const hier = Date.now() - (24 * 60 * 60 * 1000);
            
            const qRecent = query(
                collection(db, 'artifacts', appId, 'public', 'data', 'missions'),
                where('ca', '>', hier)
            );

            onSnapshot(qRecent, (snap) => {
                activeMissions = [];
                snap.forEach(doc => activeMissions.push(doc.data()));
                renderMissions();
                if(userRole === 'admin') renderStats();
            });
        };

        // --- ACTIONS (SCHEMA MINIFIÉ) ---
        const prepareNextId = () => { window.nextId = "2026-" + Math.floor(100000 + Math.random() * 900000); };

        window.creerMission = async () => {
            const data = {
                id: window.nextId,
                s: 0, // Statut attendu
                en: document.getElementById('en').value, // Exp Nom
                eq: document.getElementById('eq').value, // Exp Quartier
                dn: document.getElementById('dn').value, // Dest Nom
                dq: document.getElementById('dq').value, // Dest Quartier
                dt: document.getElementById('dt').value, // Dest Tel
                p: parseInt(document.getElementById('p').value) || 0, // Prix
                ce: currentUser.email,
                ca: Date.now()
            };

            if(!data.dn || !data.dt || !data.p) return alert("Champs obligatoires !");

            await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', data.id), data);
            alert("Mission créée !");
            prepareNextId();
        };

        window.validerLivraison = async (id) => {
            currentMissionId = id;
            document.getElementById('cameraModal').classList.remove('hidden');
        };

        window.processImage = (file) => {
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    const MAX = 600; // Taille encore plus petite pour économiser
                    canvas.width = MAX;
                    canvas.height = img.height * (MAX / img.width);
                    const ctx = canvas.getContext('2d');
                    ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
                    
                    // Finalisation avec compression 0.5 (très léger)
                    const base64 = canvas.toDataURL('image/jpeg', 0.5);
                    saveFinal(base64);
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };

        const saveFinal = async (photo) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId), {
                s: 2,
                le: currentUser.email,
                lk: new Date().toISOString().split('T')[0],
                img: photo
            });
            document.getElementById('cameraModal').classList.add('hidden');
            alert("Livraison confirmée !");
        };

        const renderMissions = () => {
            const listLivreur = document.getElementById('listLivreur');
            const listRelais = document.getElementById('listRelais');
            if(listLivreur) listLivreur.innerHTML = "";
            if(listRelais) listRelais.innerHTML = "";

            activeMissions.forEach(m => {
                // Vue Livreur (Missions en attente de livraison s=0 ou s=1)
                if(m.s < 2 && listLivreur) {
                    listLivreur.innerHTML += `
                        <div class="bg-white p-4 rounded-2xl shadow-sm border border-slate-100 mb-3">
                            <div class="flex justify-between items-start mb-2">
                                <span class="text-[10px] font-black text-blue-600">${m.id}</span>
                                <span class="bg-blue-100 text-blue-700 text-[8px] px-2 py-0.5 rounded-full font-black">À LIVRER</span>
                            </div>
                            <div class="text-xs font-bold text-slate-800">${m.dn}</div>
                            <div class="text-[10px] text-slate-500 mb-3">${m.dq} | ${m.dt}</div>
                            <button onclick="validerLivraison('${m.id}')" class="w-full bg-slate-900 text-white text-[10px] font-black py-3 rounded-xl uppercase">Confirmer Livraison</button>
                        </div>
                    `;
                }
                // Vue Relais (Suivi des siennes)
                if(m.ce === currentUser.email && listRelais) {
                    const statusText = m.s === 2 ? 'LIVRÉ' : 'EN COURS';
                    const statusColor = m.s === 2 ? 'text-emerald-500' : 'text-amber-500';
                    listRelais.innerHTML += `
                        <div class="flex justify-between items-center p-3 border-b border-slate-50">
                            <div class="text-[10px]"><b>${m.id}</b><br><span class="text-slate-400">${m.dn}</span></div>
                            <div class="text-[9px] font-black ${statusColor}">${statusText}</div>
                        </div>
                    `;
                }
            });
        };

        const renderStats = () => {
            const ca = activeMissions.reduce((acc, m) => acc + (m.s === 2 ? m.p : 0), 0);
            document.getElementById('statCa').innerText = ca.toLocaleString() + " CFA";
            document.getElementById('statCount').innerText = activeMissions.filter(m => m.s === 2).length;
        };
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700;900&display=swap');
        body { font-family: 'Inter', sans-serif; background: #f8fafc; color: #1e293b; }
        .hidden { display: none; }
    </style>
</head>
<body class="max-w-md mx-auto min-h-screen relative shadow-2xl bg-white">

    <!-- AUTH -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 z-[100] flex items-center justify-center p-8">
        <div class="w-full space-y-8 text-center bg-white p-10 rounded-[3rem]">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-20 mx-auto">
            <h1 class="text-xl font-black uppercase">CT241 LOGISTIQUE</h1>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" placeholder="Email" class="w-full p-4 bg-slate-100 rounded-2xl text-xs font-bold outline-none" required>
                <input type="password" id="loginPass" placeholder="Mot de passe" class="w-full p-4 bg-slate-100 rounded-2xl text-xs font-bold outline-none" required>
                <p id="loginError" class="text-red-500 text-[10px] font-bold hidden"></p>
                <button id="loginBtn" class="w-full bg-slate-900 text-white py-5 rounded-2xl font-black text-xs uppercase shadow-xl">Se connecter</button>
            </form>
        </div>
    </section>

    <!-- CONTENT -->
    <main id="appContent" class="hidden">
        <!-- HEADER -->
        <header class="p-6 flex justify-between items-center border-b border-slate-50 sticky top-0 bg-white/80 backdrop-blur-md z-40">
            <div class="flex items-center gap-3">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-8">
                <div id="badgeDisplay"></div>
            </div>
            <button onclick="handleLogout()" class="p-2 bg-slate-100 rounded-xl text-slate-400">🚪</button>
        </header>

        <div class="p-6 space-y-8">
            <!-- ADMIN STATS -->
            <section class="role-view hidden space-y-4">
                <div class="grid grid-cols-2 gap-4">
                    <div class="bg-slate-900 p-5 rounded-3xl text-white">
                        <span class="text-[8px] font-black text-slate-400 uppercase">CA (24h)</span>
                        <div id="statCa" class="text-lg font-black text-emerald-400">0 CFA</div>
                    </div>
                    <div class="bg-slate-900 p-5 rounded-3xl text-white">
                        <span class="text-[8px] font-black text-slate-400 uppercase">Livraisons</span>
                        <div id="statCount" class="text-lg font-black text-blue-400">0</div>
                    </div>
                </div>
            </section>

            <!-- FORMULAIRE CRÉATION (RELAIS/ADMIN) -->
            <section id="viewRelais" class="role-view hidden space-y-4">
                <div class="bg-slate-50 p-6 rounded-[2rem] space-y-4">
                    <h2 class="text-[10px] font-black uppercase text-slate-400">Nouvelle Expédition</h2>
                    <input type="text" id="en" placeholder="Nom Boutique" class="w-full p-4 bg-white rounded-xl text-xs font-bold outline-none border border-slate-200">
                    <input type="text" id="eq" placeholder="Quartier Exp." class="w-full p-4 bg-white rounded-xl text-xs font-bold outline-none border border-slate-200">
                    <hr class="border-slate-200">
                    <input type="text" id="dn" placeholder="Nom Destinataire" class="w-full p-4 bg-white rounded-xl text-xs font-bold outline-none border border-slate-200">
                    <div class="grid grid-cols-2 gap-2">
                        <input type="text" id="dq" placeholder="Quartier Dest." class="p-4 bg-white rounded-xl text-xs font-bold outline-none border border-slate-200">
                        <input type="tel" id="dt" placeholder="Tél Dest." class="p-4 bg-white rounded-xl text-xs font-bold outline-none border border-slate-200">
                    </div>
                    <input type="number" id="p" placeholder="Prix Livraison (CFA)" class="w-full p-4 bg-slate-900 text-white rounded-xl text-xs font-black outline-none">
                    <button onclick="creerMission()" class="w-full bg-emerald-600 text-white py-4 rounded-2xl font-black text-xs uppercase shadow-lg">Enregistrer Mission</button>
                </div>
                
                <div class="space-y-3">
                    <h3 class="text-[10px] font-black uppercase text-slate-400">Mes envois récents</h3>
                    <div id="listRelais" class="bg-white rounded-2xl border border-slate-100 overflow-hidden"></div>
                </div>
            </section>

            <!-- VUE LIVREUR -->
            <section id="viewLivreur" class="role-view hidden space-y-4">
                <h2 class="text-sm font-black uppercase">Tes missions du jour</h2>
                <div id="listLivreur"></div>
            </section>
        </div>
    </main>

    <!-- CAMERA MODAL -->
    <div id="cameraModal" class="fixed inset-0 bg-black/95 z-[500] hidden flex flex-col items-center justify-center p-10 text-white">
        <div class="text-center space-y-6">
            <div class="text-6xl">📸</div>
            <h3 class="text-xl font-black uppercase">Preuve de livraison</h3>
            <p class="text-slate-500 text-xs">Prenez une photo du colis pour valider.</p>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black py-5 rounded-2xl font-black text-xs uppercase">Ouvrir l'appareil</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 text-xs uppercase font-bold">Annuler</button>
        </div>
    </div>

</body>
</html>
