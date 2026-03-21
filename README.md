<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CT241 - Logistique Permanente</title>
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <script src="https://cdn.tailwindcss.com"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

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
        let searchTerm = "";
        let currentMissionId = null;

        // --- AUTHENTIFICATION ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            const errorEl = document.getElementById('loginError');

            btn.disabled = true;
            btn.innerText = "Authentification...";
            try {
                await signInWithEmailAndPassword(auth, email, pass);
            } catch (error) {
                errorEl.innerText = "Accès refusé. Identifiants incorrects.";
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
            }
        };

        // --- GESTION DES DONNÉES ---
        const prepareNextId = () => {
            const id = "CT-" + Math.floor(Math.random() * 900000 + 100000);
            document.getElementById('displayNextId').innerText = id;
            window.nextId = id;
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
            if (!fields.client || !fields.telephone) return showToast("Informations manquantes", "error");

            try {
                const missionRef = doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId);
                await setDoc(missionRef, {
                    ...fields, 
                    id: window.nextId, 
                    statut: 'en_attente', 
                    created_at: serverTimestamp(), 
                    creator_email: currentUser.email,
                    creator_uid: currentUser.uid
                });
                showToast("Colis enregistré en archive", "success");
                ['clientNom', 'clientTel', 'colisContenu', 'prixLivraison'].forEach(id => document.getElementById(id).value = "");
                prepareNextId();
            } catch (e) { showToast("Erreur d'enregistrement", "error"); }
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
                statut: 'livre', 
                photoUrl: photo, 
                livre_le: Date.now(), 
                livreur_email: currentUser.email
            });
            document.getElementById('cameraModal').classList.add('hidden');
            showToast("Livraison validée !", "success");
        };

        const startListeners = () => {
            const q = collection(db, 'artifacts', appId, 'public', 'data', 'missions');
            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(doc => allMissions.push(doc.data()));
                renderUI();
            }, (err) => console.error("Firestore Error:", err));
        };

        window.handleSearch = (val) => {
            searchTerm = val.toLowerCase();
            renderUI();
        };

        window.renderUI = () => {
            const cont = {
                dispatch: document.getElementById('containerDispatch'),
                livreur: document.getElementById('containerLivreur'),
                archives: document.getElementById('archiveBody')
            };
            Object.values(cont).forEach(c => c.innerHTML = "");
            
            let count = 0;
            // Tri décroissant pour que le plus récent soit en haut
            const sorted = allMissions.sort((a,b) => {
                const timeA = a.created_at?.toMillis ? a.created_at.toMillis() : (a.livre_le || 0);
                const timeB = b.created_at?.toMillis ? b.created_at.toMillis() : (b.livre_le || 0);
                return timeB - timeA;
            });

            sorted.forEach(m => {
                const isMyColis = m.creator_email === currentUser.email;
                const matchesSearch = m.id.toLowerCase().includes(searchTerm) || m.client.toLowerCase().includes(searchTerm);

                if (m.statut === 'en_attente' && (userRole === 'admin' || userRole === 'dispatch')) {
                    cont.dispatch.innerHTML += `<div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 shadow-sm flex justify-between items-center">
                        <div class="text-[11px]">
                            <span class="font-black text-blue-600 block">${m.id}</span>
                            <b>${m.client}</b><br><span class="text-slate-400">${m.arrivee}</span>
                        </div>
                        <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white text-[10px] px-4 py-2 rounded-xl font-bold">LANCER</button>
                    </div>`;
                } else if (m.statut === 'publie' && (userRole === 'admin' || userRole === 'livreur')) {
                    cont.livreur.innerHTML += `<div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 space-y-3">
                        <div class="flex justify-between font-black items-center"><span class="text-amber-700">${m.id}</span><span class="text-[8px] bg-amber-500 text-white px-2 py-1 rounded-full uppercase">Disponible</span></div>
                        <p class="text-[11px] text-amber-900"><b>Lieu:</b> ${m.arrivee}<br><b>Contact:</b> ${m.client} (${m.telephone})</p>
                        <button onclick="openCamera('${m.id}')" class="w-full bg-amber-500 text-white font-black py-4 rounded-2xl shadow-lg">SCANNER LIVRAISON</button>
                    </div>`;
                } 
                
                // ARCHIVES PERMANENTES (Filtre par boutique ou vue globale pour admin)
                const canSeeArchive = (userRole === 'admin' || userRole === 'dispatch' || (userRole === 'relais' && isMyColis));
                if (canSeeArchive && matchesSearch) {
                    count++;
                    const statusColor = m.statut === 'livre' ? 'text-emerald-500' : 'text-slate-400';
                    const statusText = m.statut === 'livre' ? 'LIVRÉ' : (m.statut === 'publie' ? 'EN ROUTE' : 'STOCK');
                    
                    cont.archives.innerHTML += `<tr class="border-b border-slate-800 text-[10px] hover:bg-slate-800/50 transition cursor-pointer" onclick="openArchiveDetail('${m.id}')">
                        <td class="p-4 font-bold text-white">${m.id}</td>
                        <td class="p-4 text-slate-300"><b>${m.client}</b><br><span class="text-[9px] uppercase">${statusText}</span></td>
                        <td class="p-4"><div class="w-8 h-8 rounded-lg bg-slate-700 flex items-center justify-center overflow-hidden">${m.photoUrl ? `<img src="${m.photoUrl}" class="object-cover w-full h-full">` : '📦'}</div></td>
                        <td class="p-4 text-right ${statusColor} font-black">${m.prix || 0}</td>
                    </tr>`;
                }
            });
            document.getElementById('archiveCount').innerText = count;
        };

        window.openArchiveDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            if (!m) return;
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area bg-white p-6 space-y-4 text-slate-900 rounded-3xl">
                    <div class="flex justify-between border-b pb-4">
                        <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-10">
                        <div class="text-right"><h2 class="font-black text-xl">${m.id}</h2><p class="text-[9px] text-slate-400">Statut: ${m.statut.toUpperCase()}</p></div>
                    </div>
                    <div class="grid grid-cols-2 gap-4 text-[10px]">
                        <div class="p-3 bg-slate-50 rounded-xl"><b>Expéditeur:</b><br>${m.client}<br>${m.telephone}</div>
                        <div class="p-3 bg-slate-50 rounded-xl"><b>Destination:</b><br>${m.arrivee}<br>Service: ${m.service}</div>
                    </div>
                    <div class="p-3 border-2 border-dashed border-slate-100 rounded-xl text-center">
                        <p class="text-[10px] font-bold text-slate-400">MONTANT À PERCEVOIR</p>
                        <p class="text-2xl font-black text-slate-900">${m.prix || 0} FCFA</p>
                    </div>
                    ${m.photoUrl ? `<img src="${m.photoUrl}" class="w-full h-48 object-cover rounded-2xl">` : `<div class="h-48 bg-slate-100 rounded-2xl flex items-center justify-center text-slate-300 italic">Preuve photo non disponible</div>`}
                    <p class="text-[8px] text-center text-slate-400">CT241 Logistique - Document généré le ${new Date().toLocaleDateString()}</p>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        const showToast = (msg, type) => {
            const t = document.getElementById('toast');
            t.innerText = msg;
            t.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-xs shadow-2xl z-[500] ${type==='success'?'bg-emerald-600':'bg-red-600'}`;
            t.classList.remove('hidden');
            setTimeout(() => t.classList.add('hidden'), 3000);
        };
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; }
        .hidden { display: none; }
        @media print {
            body * { visibility: hidden; }
            .print-area, .print-area * { visibility: visible; }
            .print-area { position: fixed; left: 0; top: 0; width: 100%; }
        }
    </style>
</head>
<body class="min-h-screen">

    <div id="toast" class="hidden"></div>

    <!-- CONNEXION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[3rem] p-10 space-y-8 shadow-2xl text-center">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-24 mx-auto">
            <div class="space-y-2">
                <h1 class="text-2xl font-black text-slate-900">CT241 PORTAIL</h1>
                <p class="text-[10px] text-slate-400 font-bold uppercase tracking-widest">Système Logistique GABON</p>
            </div>
            <form onsubmit="handleLogin(event)" class="space-y-3">
                <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs outline-none focus:border-slate-900" placeholder="E-mail professionnel">
                <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs outline-none focus:border-slate-900" placeholder="Code secret">
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden"></p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl active:scale-95 transition">ACCÉDER AU PORTAIL</button>
            </form>
        </div>
    </section>

    <!-- INTERFACE -->
    <main id="appContent" class="hidden pb-24">
        <nav class="bg-white/80 backdrop-blur-md p-4 sticky top-0 z-50 border-b border-slate-100 no-print">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-8">
                    <div class="flex flex-col">
                        <span class="text-[10px] font-black leading-none" id="userDisplayEmail">...</span>
                        <div id="badgeDisplay" class="mt-1"></div>
                    </div>
                </div>
                <button onclick="handleLogout()" class="w-10 h-10 bg-slate-50 rounded-xl flex items-center justify-center text-slate-400 hover:text-red-500">
                    <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m4 4H7" /></svg>
                </button>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            <!-- RELAIS : CRÉATION -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl border border-slate-100 overflow-hidden">
                    <div class="bg-emerald-600 p-6 text-white flex justify-between items-center">
                        <h2 class="font-black text-xs uppercase">Réception Colis</h2>
                        <span id="displayNextId" class="text-[10px] font-mono font-black bg-white/20 px-3 py-1 rounded-lg">...</span>
                    </div>
                    <div class="p-6 space-y-4">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="clientNom" placeholder="Nom Expéditeur" class="p-4 bg-slate-50 rounded-2xl font-bold text-[10px] outline-none border border-transparent focus:border-emerald-500">
                            <input type="tel" id="clientTel" placeholder="Téléphone" class="p-4 bg-slate-50 rounded-2xl font-bold text-[10px] outline-none border border-transparent focus:border-emerald-500">
                        </div>
                        <textarea id="colisContenu" placeholder="Nature du colis..." class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[10px] outline-none min-h-[80px] border border-transparent focus:border-emerald-500"></textarea>
                        <div class="grid grid-cols-2 gap-3">
                            <select id="ptDepart" class="p-4 bg-slate-50 rounded-2xl font-black text-[10px] outline-none"><option>Libreville (Relais)</option><option>Port-Gentil</option><option>Franceville</option></select>
                            <select id="ptArrivee" class="p-4 bg-slate-50 rounded-2xl font-black text-[10px] outline-none"><option>Libreville (Client)</option><option>Akanda</option><option>Owendo</option></select>
                        </div>
                        <div class="grid grid-cols-2 gap-3">
                            <select id="typeService" class="p-4 bg-slate-50 rounded-2xl font-black text-[10px] outline-none"><option value="standard">Standard</option><option value="express">Express</option></select>
                            <input type="number" id="prixLivraison" placeholder="Montant (CFA)" class="p-4 bg-slate-50 rounded-2xl font-bold text-[10px] outline-none border border-transparent focus:border-emerald-500">
                        </div>
                        <button onclick="genererMission()" class="w-full bg-emerald-600 text-white font-black py-5 rounded-2xl shadow-lg active:scale-95 transition text-xs">ENREGISTRER LE COLIS</button>
                    </div>
                </div>
            </section>

            <!-- GESTIONNAIRE : DISPATCH -->
            <section id="rubrique2" class="role-section hidden">
                <div class="flex items-center gap-2 mb-4 ml-2">
                    <div class="w-2 h-2 bg-blue-500 rounded-full animate-pulse"></div>
                    <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Flux à dispatcher</h3>
                </div>
                <div id="containerDispatch"></div>
            </section>

            <!-- LIVREUR : MISSIONS -->
            <section id="rubrique3" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-4 ml-2">Missions de livraison</h3>
                <div id="containerLivreur"></div>
            </section>

            <!-- ARCHIVES (PERMANENTES) -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-slate-900 rounded-[2.5rem] overflow-hidden shadow-2xl">
                    <div class="p-6 space-y-4">
                        <div class="flex justify-between items-center">
                            <h2 class="text-white font-black text-xs uppercase">Historique & Suivi</h2>
                            <span id="archiveCount" class="bg-yellow-500 text-slate-900 font-black text-[9px] px-3 py-1 rounded-full">0</span>
                        </div>
                        <input type="text" onkeyup="handleSearch(this.value)" placeholder="Rechercher par ID ou Nom..." class="w-full bg-white/5 border border-white/10 rounded-xl py-3 px-4 text-[10px] text-white outline-none focus:border-yellow-500 transition-all">
                    </div>
                    <div class="overflow-x-auto">
                        <table class="w-full text-left">
                            <thead class="bg-white/5 text-slate-500 text-[8px] uppercase font-black">
                                <tr><th class="p-4">ID</th><th class="p-4">Détails</th><th class="p-4">Aperçu</th><th class="p-4 text-right">Prix</th></tr>
                            </thead>
                            <tbody id="archiveBody"></tbody>
                        </table>
                    </div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL DETAIL -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/90 z-[400] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-sm bg-white rounded-[2.5rem] shadow-2xl overflow-hidden animate-in zoom-in duration-200">
            <div id="detailContent"></div>
            <div class="p-6 bg-slate-50 flex gap-2 no-print">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-4 rounded-2xl text-[10px]">🖨️ IMPRIMER</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-6 bg-slate-200 text-slate-600 font-black py-4 rounded-2xl text-[10px]">RETOUR</button>
            </div>
        </div>
    </div>

    <!-- CAMERA -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[300] hidden flex flex-col items-center justify-center p-8 text-white">
        <div class="text-center space-y-8">
            <div class="text-6xl">📸</div>
            <div class="space-y-2">
                <h2 class="text-2xl font-black">Validation Photo</h2>
                <p class="text-xs text-slate-400">La photo du colis est obligatoire pour valider la livraison.</p>
            </div>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-5 rounded-3xl text-sm shadow-2xl">OUVRIR LA CAMÉRA</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 text-[10px] font-bold uppercase tracking-widest">Annuler la livraison</button>
        </div>
    </div>

</body>
</html>
