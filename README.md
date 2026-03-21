<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CT241 - Système de Gestion Logistique</title>
    <!-- Favicon -->
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <script src="https://cdn.tailwindcss.com"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        // Configuration Firebase
        const firebaseConfig = {
            apiKey: "AIzaSyAdNSFmL45rSo9SxJJkvUPWeext0f7RX_Q",
            authDomain: "ct241-service-de-livraison.firebaseapp.com",
            databaseURL: "https://ct241-service-de-livraison-default-rtdb.firebaseio.com",
            projectId: "ct241-service-de-livraison",
            storageBucket: "ct241-service-de-livraison.firebasestorage.app",
            messagingSenderId: "297254676010",
            appId: "1:297254676010:web:01e3765686c8d478618553",
            measurementId: "G-GMPJY3R4T3"
        };

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = 'ct241-service-de-livraison';

        let currentUser = null;
        let userRole = null; // Défini par l'email
        let allMissions = [];
        let searchTerm = "";
        let currentMissionId = null;
        let nextGeneratedId = "";

        // --- AUTHENTIFICATION ET DROITS ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            const errorEl = document.getElementById('loginError');

            btn.disabled = true;
            btn.innerText = "Identification...";
            errorEl.classList.add('hidden');

            try {
                await signInWithEmailAndPassword(auth, email, pass);
            } catch (error) {
                errorEl.innerText = "Accès refusé. Vérifiez vos identifiants.";
                errorEl.classList.remove('hidden');
                btn.disabled = false;
                btn.innerText = "Se connecter";
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
            else userRole = 'visiteur';

            // Affichage restrictif des sections
            document.querySelectorAll('.role-section').forEach(s => s.classList.add('hidden'));
            
            const badge = document.getElementById('badgeDisplay');
            const colors = { admin: 'bg-red-600', dispatch: 'bg-blue-600', relais: 'bg-emerald-600', livreur: 'bg-amber-600' };
            badge.innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-black uppercase text-white ${colors[userRole] || 'bg-slate-500'}">${userRole}</span>`;

            if (userRole === 'admin') {
                document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
            } else if (userRole === 'relais') {
                document.getElementById('rubrique1').classList.remove('hidden'); // Création
                document.getElementById('rubrique4').classList.remove('hidden'); // Archives (Suivi)
            } else if (userRole === 'dispatch') {
                document.getElementById('rubrique2').classList.remove('hidden'); // Missions à lancer
                document.getElementById('rubrique4').classList.remove('hidden'); // Archives
            } else if (userRole === 'livreur') {
                document.getElementById('rubrique3').classList.remove('hidden'); // Missions dispo
            }
        };

        // --- MISSIONS LOGIQUE ---
        const prepareNextId = () => {
            nextGeneratedId = "CT-" + Math.floor(Math.random() * 900000 + 100000);
            document.getElementById('displayNextId').innerText = nextGeneratedId;
            document.getElementById('visibleIdInput').value = nextGeneratedId;
        };

        window.genererMission = async () => {
            const fields = {
                client: document.getElementById('clientNom').value,
                telephone: document.getElementById('clientTel').value,
                depart: document.getElementById('ptDepart').value,
                arrivee: document.getElementById('ptArrivee').value,
                contenu: document.getElementById('colisContenu').value,
                service: document.getElementById('typeService').value,
                prix: document.getElementById('prixLivraison').value
            };
            if (!fields.client || !fields.telephone) return showToast("Informations incomplètes", "error");

            try {
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', nextGeneratedId), {
                    ...fields, id: nextGeneratedId, statut: 'en_attente', created_at: serverTimestamp(), created_by: currentUser.uid, creator_email: currentUser.email
                });
                showToast("Colis enregistré !", "success");
                ['clientNom', 'clientTel', 'colisContenu', 'prixLivraison'].forEach(id => document.getElementById(id).value = "");
                prepareNextId();
            } catch (e) { showToast("Erreur système", "error"); }
        };

        window.publierMission = async (id) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { statut: 'publie' });
        };

        window.openCamera = (id) => {
            currentMissionId = id;
            document.getElementById('cameraModal').classList.remove('hidden');
        };

        window.processImage = (file) => {
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    const MAX = 800;
                    canvas.width = MAX;
                    canvas.height = img.height * (MAX / img.width);
                    canvas.getContext('2d').drawImage(img, 0, 0, canvas.width, canvas.height);
                    finalizeDelivery(canvas.toDataURL('image/jpeg', 0.6));
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };

        const finalizeDelivery = async (photo) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId), {
                statut: 'livre', photoUrl: photo, livre_le: Date.now(), livreur_id: currentUser.uid, livreur_email: currentUser.email
            });
            document.getElementById('cameraModal').classList.add('hidden');
            showToast("Livré et validé !", "success");
        };

        const startListeners = () => {
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'missions'), (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
            });
        };

        window.handleSearch = (val) => {
            searchTerm = val.toLowerCase();
            renderUI();
        };

        window.renderUI = () => {
            const containers = {
                dispatch: document.getElementById('containerDispatch'),
                livreur: document.getElementById('containerLivreur'),
                archives: document.getElementById('archiveBody')
            };
            Object.values(containers).forEach(c => c.innerHTML = "");
            
            let arcCount = 0;
            const sorted = allMissions.sort((a,b) => (b.created_at?.seconds || 0) - (a.created_at?.seconds || 0));

            sorted.forEach(m => {
                // Filtre supplémentaire : Les boutiques/relais ne voient QUE leurs colis créés dans l'archive
                const isCreator = m.created_by === currentUser.uid;
                
                if (m.statut === 'en_attente' && (userRole === 'admin' || userRole === 'dispatch')) {
                    containers.dispatch.innerHTML += `<div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 shadow-sm">
                        <div class="flex justify-between mb-2"><span class="text-xs font-black">${m.id}</span><span class="text-[9px] font-bold px-2 py-0.5 rounded bg-slate-100">${m.service}</span></div>
                        <p class="text-[11px] mb-2"><b>${m.client}</b> (${m.telephone})<br>${m.depart} → ${m.arrivee}</p>
                        <button onclick="publierMission('${m.id}')" class="w-full bg-blue-600 text-white text-[10px] py-2 rounded-xl font-bold">DISPATCHER</button>
                    </div>`;
                } else if (m.statut === 'publie' && (userRole === 'admin' || userRole === 'livreur')) {
                    containers.livreur.innerHTML += `<div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 space-y-3 text-amber-900">
                        <div class="flex justify-between font-black items-center"><span>${m.id}</span><span class="text-[9px] bg-amber-500 text-white px-2 py-1 rounded-full">MISSION DISPO</span></div>
                        <p class="text-[11px]"><b>Destination:</b> ${m.arrivee}<br><b>Client:</b> ${m.client}</p>
                        <button onclick="openCamera('${m.id}')" class="w-full bg-amber-500 text-white font-bold py-4 rounded-2xl shadow-lg">CONFIRMER LIVRAISON</button>
                    </div>`;
                } else if (m.statut === 'livre') {
                    const matchesSearch = m.id.toLowerCase().includes(searchTerm) || m.client.toLowerCase().includes(searchTerm);
                    
                    // Sécurité : Les relais ne voient que leurs colis. Les admin/dispatch voient tout.
                    const canSeeArchive = (userRole === 'admin' || userRole === 'dispatch' || (userRole === 'relais' && isCreator));

                    if (matchesSearch && canSeeArchive) {
                        arcCount++;
                        containers.archives.innerHTML += `<tr class="border-b border-slate-700 text-[10px] hover:bg-slate-800 transition cursor-pointer" onclick="openArchiveDetail('${m.id}')">
                            <td class="p-4 text-yellow-500 font-bold">${m.id}</td>
                            <td class="p-4">${m.client}<br><span class="text-slate-500">${m.arrivee}</span></td>
                            <td class="p-4"><img src="${m.photoUrl}" class="w-8 h-8 rounded-lg object-cover"></td>
                            <td class="p-4 text-slate-500">${new Date(m.livre_le).toLocaleDateString()}</td>
                        </tr>`;
                    }
                }
            });
            document.getElementById('archiveCount').innerText = arcCount;
        };

        window.openArchiveDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            if (!m) return;
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area bg-white p-6 space-y-4 text-slate-900 rounded-3xl">
                    <div class="flex justify-between items-start border-b-2 border-slate-100 pb-4">
                        <div><img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-10 mb-2"><h2 class="text-xl font-black">REÇU CT241</h2></div>
                        <div class="text-right"><span class="text-lg font-black font-mono text-emerald-600">${m.id}</span><p class="text-[10px] text-slate-400">${new Date(m.livre_le).toLocaleString()}</p></div>
                    </div>
                    <div class="grid grid-cols-2 gap-4 text-xs">
                        <div class="p-3 bg-slate-50 rounded-xl"><p class="font-black uppercase text-[8px] text-slate-400">Expéditeur</p><p class="font-bold">${m.client}</p></div>
                        <div class="p-3 bg-slate-50 rounded-xl"><p class="font-black uppercase text-[8px] text-slate-400">Destination</p><p class="font-bold">${m.arrivee}</p></div>
                    </div>
                    <img src="${m.photoUrl}" class="w-full h-48 object-cover rounded-2xl border-2 border-slate-100">
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        window.printCurrentArchive = () => window.print();

        const showToast = (msg, type) => {
            const t = document.getElementById('toast');
            t.innerText = msg;
            t.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-sm shadow-xl z-[500] ${type==='success'?'bg-emerald-600':'bg-red-600'}`;
            t.classList.remove('hidden');
            setTimeout(() => t.classList.add('hidden'), 3000);
        };
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; }
        .hidden { display: none; }
        @media print {
            body * { visibility: hidden; }
            .print-area, .print-area * { visibility: visible; }
            .print-area { position: fixed; left: 0; top: 0; width: 100%; border: none; }
            .no-print { display: none !important; }
        }
    </style>
</head>
<body class="bg-slate-50 min-h-screen">

    <div id="toast" class="hidden"></div>

    <!-- CONNEXION RESTRICTIVE -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[2.5rem] p-10 space-y-8 shadow-2xl">
            <div class="text-center space-y-4">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-20 mx-auto">
                <h1 class="text-2xl font-black text-slate-900 leading-tight">Logistique Sécurisée<br>CT241 GABON</h1>
            </div>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl font-semibold outline-none focus:border-slate-900" placeholder="E-mail (ex: livreur@ct241.com)">
                <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl font-semibold outline-none focus:border-slate-900" placeholder="Code d'accès">
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden text-center"></p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl active:scale-95 transition">ACCÉDER AU PORTAIL</button>
            </form>
            <p class="text-[9px] text-slate-400 text-center font-medium">Les accès sont configurés selon votre profil : admin, relais, dispatch ou livreur.</p>
        </div>
    </section>

    <!-- APP CONTENT -->
    <main id="appContent" class="hidden pb-24">
        <nav class="bg-white p-3 shadow-md sticky top-0 z-50 border-b border-slate-100 no-print">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-8">
                    <span class="text-xs font-black truncate max-w-[150px]" id="userDisplayEmail">...</span>
                </div>
                <div class="flex items-center gap-4">
                    <div id="badgeDisplay"></div>
                    <button onclick="handleLogout()" class="p-2 text-slate-400 hover:text-red-500"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m4 4H7" /></svg></button>
                </div>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            <!-- SECTION : RÉCEPTION COLIS (RELAIS / ADMIN) -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl border border-slate-100 overflow-hidden">
                    <div class="bg-emerald-600 p-6 text-white flex justify-between items-center">
                        <h2 class="font-black text-sm uppercase">Réception Nouveau Colis</h2>
                        <span id="displayNextId" class="text-xs font-mono font-black bg-emerald-900/30 px-3 py-1 rounded-lg">...</span>
                    </div>
                    <div class="p-6 space-y-4">
                        <input type="hidden" id="visibleIdInput">
                        <div class="grid grid-cols-2 gap-4">
                            <input type="text" id="clientNom" placeholder="Nom Expéditeur" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-xs outline-none">
                            <input type="tel" id="clientTel" placeholder="Téléphone" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-xs outline-none">
                        </div>
                        <textarea id="colisContenu" placeholder="Description courte (Ex: Sac d'habits)" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-xs outline-none"></textarea>
                        <div class="grid grid-cols-2 gap-4">
                            <select id="ptDepart" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs outline-none"><option>Libreville (Nzeng-Ayong)</option><option>Port-Gentil</option><option>Akanda</option></select>
                            <select id="ptArrivee" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs outline-none"><option>Libreville (Centre)</option><option>Owendo</option><option>Akanda</option></select>
                        </div>
                        <div class="grid grid-cols-2 gap-4">
                            <select id="typeService" class="p-4 bg-slate-50 rounded-2xl font-bold text-xs outline-none"><option value="standard">Standard (24h)</option><option value="express">Express (3h)</option></select>
                            <input type="number" id="prixLivraison" placeholder="Prix (FCFA)" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-xs outline-none">
                        </div>
                        <button onclick="genererMission()" class="w-full bg-emerald-600 text-white font-black py-5 rounded-2xl shadow-lg active:scale-95 transition">ENREGISTRER EN STOCK</button>
                    </div>
                </div>
            </section>

            <!-- SECTION : DISPATCH (DISPATCH / ADMIN) -->
            <section id="rubrique2" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-3 ml-2">Colis en attente de Livraison</h3>
                <div id="containerDispatch"></div>
            </section>

            <!-- SECTION : LIVRAISON (LIVREUR / ADMIN) -->
            <section id="rubrique3" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-3 ml-2">Missions de Livraison Dispos</h3>
                <div id="containerLivreur"></div>
            </section>

            <!-- SECTION : ARCHIVES / SUIVI (TOUS AVEC FILTRE RELAIS) -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-slate-900 rounded-[2.5rem] overflow-hidden shadow-2xl">
                    <div class="p-6 space-y-4">
                        <div class="flex justify-between items-center">
                            <h2 class="text-white font-black text-sm uppercase">Suivi & Archives</h2>
                            <span id="archiveCount" class="bg-yellow-500 text-slate-900 font-black text-[10px] px-3 py-1 rounded-full">0</span>
                        </div>
                        <div class="relative no-print">
                            <input type="text" onkeyup="handleSearch(this.value)" placeholder="Chercher par ID ou Nom..." class="w-full bg-slate-800 border border-slate-700 rounded-xl py-3 pl-4 text-xs text-white outline-none focus:border-yellow-500 transition-all">
                        </div>
                    </div>
                    <div class="overflow-x-auto">
                        <table class="w-full text-left">
                            <thead class="bg-slate-800 text-slate-400 text-[9px] uppercase font-black">
                                <tr><th class="p-4">ID</th><th class="p-4">Client</th><th class="p-4">Photo</th><th class="p-4">Date</th></tr>
                            </thead>
                            <tbody id="archiveBody"></tbody>
                        </table>
                    </div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL DETAIL -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/80 z-[400] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-lg bg-white rounded-[2.5rem] shadow-2xl overflow-hidden animate-in zoom-in duration-200">
            <div id="detailContent"></div>
            <div class="p-6 bg-slate-50 flex gap-3 no-print">
                <button onclick="printCurrentArchive()" class="flex-1 bg-slate-900 text-white font-black py-4 rounded-2xl">🖨️ IMPRIMER REÇU</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-6 bg-slate-200 text-slate-600 font-black py-4 rounded-2xl">FERMER</button>
            </div>
        </div>
    </div>

    <!-- MODAL CAMERA -->
    <div id="cameraModal" class="fixed inset-0 bg-slate-950/95 z-[300] hidden flex flex-col items-center justify-center p-6 text-white">
        <div class="text-center space-y-6">
            <div class="w-20 h-20 bg-amber-500 rounded-full flex items-center justify-center mx-auto text-3xl">📸</div>
            <h2 class="text-2xl font-black">Preuve Photo Obligatoire</h2>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full max-w-xs bg-white text-slate-900 font-black py-6 rounded-2xl text-lg">PRENDRE LA PHOTO</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 font-bold uppercase text-[10px] tracking-widest">Retour</button>
        </div>
    </div>

</body>
</html>
