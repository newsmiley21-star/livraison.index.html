<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CT241 - Logistique & Performance</title>
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

        // --- MISSIONS & STATS ---
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
                prix: parseFloat(document.getElementById('prixLivraison').value) || 0
            };
            if (!fields.client || !fields.telephone) return showToast("Données incomplètes", "error");

            try {
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId), {
                    ...fields, id: window.nextId, statut: 'en_attente', created_at: serverTimestamp(), creator_email: currentUser.email
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
            const today = new Date().toISOString().split('T')[0];
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId), {
                statut: 'livre', 
                photoUrl: photo, 
                livre_le: Date.now(), 
                livre_date_key: today,
                livreur_email: currentUser.email
            });
            document.getElementById('cameraModal').classList.add('hidden');
            showToast("Mission accomplie !", "success");
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
            const today = new Date().toISOString().split('T')[0];
            const stats = { caTotal: 0, livraisonsTotal: 0, caJour: 0, livreurs: {} };

            allMissions.forEach(m => {
                if (m.statut === 'livre') {
                    stats.caTotal += m.prix || 0;
                    stats.livraisonsTotal++;
                    if (m.livre_date_key === today) stats.caJour += m.prix || 0;

                    // Groupement par livreur
                    const lEmail = m.livreur_email || 'Inconnu';
                    if (!stats.livreurs[lEmail]) stats.livreurs[lEmail] = { count: 0, ca: 0, bonus: 0 };
                    stats.livreurs[lEmail].count++;
                    stats.livreurs[lEmail].ca += m.prix || 0;
                    
                    // Calcul Bonus CT241: 700 CFA après la 17ème mission
                    if (stats.livreurs[lEmail].count > 17) {
                        stats.livreurs[lEmail].bonus += 700;
                    }
                }
            });

            // Affichage Dashboard Admin
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

            // Affichage Ma Performance (Livreur)
            if (userRole === 'livreur') {
                const myData = stats.livreurs[currentUser.email] || { count: 0, bonus: 0 };
                document.getElementById('myCount').innerText = myData.count;
                document.getElementById('myBonus').innerText = myData.bonus + ' CFA';
                
                // Barre de progression vers bonus (Objectif 17)
                const prog = Math.min((myData.count / 17) * 100, 100);
                document.getElementById('progBar').style.width = prog + '%';
                document.getElementById('progLabel').innerText = myData.count >= 17 ? 'Bonus Activé !' : `Objectif Bonus : ${myData.count}/17 missions`;
            }
        };

        window.renderUI = () => {
            const containers = {
                dispatch: document.getElementById('containerDispatch'),
                livreur: document.getElementById('containerLivreur'),
                archives: document.getElementById('archiveBody')
            };
            Object.values(containers).forEach(c => c.innerHTML = "");

            const sorted = allMissions.sort((a,b) => (b.livre_le || 0) - (a.livre_le || 0));

            sorted.forEach(m => {
                const isMyColis = m.creator_email === currentUser.email;
                if (m.statut === 'en_attente' && (userRole === 'admin' || userRole === 'dispatch')) {
                    containers.dispatch.innerHTML += `<div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 shadow-sm flex justify-between items-center">
                        <div class="text-[11px]"><span class="font-black text-blue-600 block">${m.id}</span><b>${m.client}</b><br><span class="text-slate-400">${m.arrivee}</span></div>
                        <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white text-[10px] px-5 py-2 rounded-xl font-bold">DISPATCHER</button>
                    </div>`;
                } else if (m.statut === 'publie' && (userRole === 'admin' || userRole === 'livreur')) {
                    containers.livreur.innerHTML += `<div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 space-y-3">
                        <div class="flex justify-between font-black items-center text-amber-700"><span>${m.id}</span><span class="text-[9px] bg-amber-500 text-white px-2 py-1 rounded-full uppercase">Disponible</span></div>
                        <p class="text-[11px] text-amber-900"><b>Lieu:</b> ${m.arrivee}<br><b>Contact:</b> ${m.client} (${m.telephone})</p>
                        <button onclick="openCamera('${m.id}')" class="w-full bg-amber-500 text-white font-black py-4 rounded-2xl shadow-lg active:scale-95 transition">VALIDER LIVRAISON</button>
                    </div>`;
                }
                
                // ARCHIVES PERMANENTES
                const canSeeArchive = (userRole === 'admin' || userRole === 'dispatch' || (userRole === 'relais' && isMyColis));
                if (canSeeArchive && m.statut === 'livre') {
                    containers.archives.innerHTML += `<tr class="border-b border-slate-800 text-[10px] hover:bg-slate-800 transition cursor-pointer" onclick="openArchiveDetail('${m.id}')">
                        <td class="p-4 font-bold text-white">${m.id}</td>
                        <td class="p-4 text-slate-300"><b>${m.client}</b></td>
                        <td class="p-4 text-center"><span class="text-emerald-400 font-black">LIVRÉ</span></td>
                        <td class="p-4 text-right text-slate-400">${m.prix.toLocaleString()}</td>
                    </tr>`;
                }
            });
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
                    <div class="grid grid-cols-2 gap-4 text-xs">
                        <div class="p-3 bg-slate-50 rounded-xl"><p class="text-[8px] text-slate-400 uppercase font-black">Expéditeur</p><b>${m.client}</b></div>
                        <div class="p-3 bg-slate-50 rounded-xl"><p class="text-[8px] text-slate-400 uppercase font-black">Livreur</p><b>${m.livreur_email}</b></div>
                    </div>
                    <img src="${m.photoUrl}" class="w-full h-48 object-cover rounded-2xl border-4 border-slate-100">
                    <div class="text-center font-black text-2xl py-2">${m.prix.toLocaleString()} CFA</div>
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
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; overflow-x: hidden; }
        .hidden { display: none; }
        @media print { body * { visibility: hidden; } .print-area, .print-area * { visibility: visible; } .print-area { position: fixed; left: 0; top: 0; width: 100%; } }
    </style>
</head>
<body class="min-h-screen">

    <div id="toast" class="hidden"></div>

    <!-- AUTHENTICATION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[3rem] p-10 space-y-8 shadow-2xl text-center">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-24 mx-auto">
            <h1 class="text-2xl font-black text-slate-900">CT241 GABON</h1>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs outline-none" placeholder="E-mail Pro">
                <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs outline-none" placeholder="Mot de passe">
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden"></p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl transition">CONNEXION</button>
            </form>
        </div>
    </section>

    <!-- CONTENT -->
    <main id="appContent" class="hidden pb-24">
        <nav class="bg-white/80 backdrop-blur-md p-4 sticky top-0 z-50 border-b border-slate-100 no-print">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-8">
                    <div class="flex flex-col"><span class="text-[9px] font-black leading-none" id="userDisplayEmail">...</span><div id="badgeDisplay" class="mt-1"></div></div>
                </div>
                <button onclick="handleLogout()" class="p-2 text-slate-400"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m4 4H7" /></svg></button>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            
            <!-- ADMIN DASHBOARD -->
            <section id="adminDash" class="role-section hidden space-y-4">
                <div class="grid grid-cols-3 gap-3">
                    <div class="bg-slate-900 p-4 rounded-3xl text-white shadow-lg">
                        <span class="text-[8px] uppercase font-black text-slate-400">CA Total</span>
                        <div id="dashCaTotal" class="text-xs font-black text-emerald-400">0</div>
                    </div>
                    <div class="bg-slate-900 p-4 rounded-3xl text-white shadow-lg">
                        <span class="text-[8px] uppercase font-black text-slate-400">CA Jour</span>
                        <div id="dashCaJour" class="text-xs font-black text-yellow-500">0</div>
                    </div>
                    <div class="bg-slate-900 p-4 rounded-3xl text-white shadow-lg">
                        <span class="text-[8px] uppercase font-black text-slate-400">Missions</span>
                        <div id="dashLivTotal" class="text-xs font-black text-blue-400">0</div>
                    </div>
                </div>
                <div class="bg-slate-900 p-6 rounded-[2.5rem] shadow-2xl">
                    <h3 class="text-white font-black text-[10px] uppercase mb-4">Performance par Livreur</h3>
                    <div id="dashLivreursList"></div>
                </div>
            </section>

            <!-- LIVREUR STATS -->
            <section id="statLivreurSection" class="role-section hidden">
                <div class="bg-slate-900 p-6 rounded-[2.5rem] text-white shadow-2xl space-y-5">
                    <div class="flex justify-between items-end">
                        <div><p class="text-[10px] uppercase font-black text-slate-400">Mes Livraisons</p><h2 id="myCount" class="text-4xl font-black">0</h2></div>
                        <div class="text-right"><p class="text-[10px] uppercase font-black text-emerald-400">Bonus Cumulé</p><h2 id="myBonus" class="text-2xl font-black text-emerald-500">0 CFA</h2></div>
                    </div>
                    <div class="space-y-2">
                        <div class="flex justify-between text-[9px] font-bold uppercase"><span id="progLabel">Objectif Bonus</span><span>17 Missions</span></div>
                        <div class="w-full h-3 bg-slate-800 rounded-full overflow-hidden">
                            <div id="progBar" class="h-full bg-emerald-500 transition-all duration-700" style="width: 0%"></div>
                        </div>
                    </div>
                </div>
            </section>

            <!-- BOUTIQUE : RECEPTION -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl border border-slate-100 overflow-hidden">
                    <div class="bg-emerald-600 p-6 text-white flex justify-between items-center"><h2 class="font-black text-xs uppercase">Enregistrement Colis</h2><span id="displayNextId" class="text-[10px] font-mono font-black bg-white/20 px-3 py-1 rounded-lg">...</span></div>
                    <div class="p-6 space-y-4">
                        <div class="grid grid-cols-2 gap-3">
                            <input type="text" id="clientNom" placeholder="Nom Expéditeur" class="p-4 bg-slate-50 rounded-2xl font-bold text-[10px] outline-none border border-transparent focus:border-emerald-500">
                            <input type="tel" id="clientTel" placeholder="Téléphone" class="p-4 bg-slate-50 rounded-2xl font-bold text-[10px] outline-none border border-transparent focus:border-emerald-500">
                        </div>
                        <textarea id="colisContenu" placeholder="Nature du colis..." class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[10px] outline-none min-h-[80px] border border-transparent focus:border-emerald-500"></textarea>
                        <div class="grid grid-cols-2 gap-3">
                            <select id="ptDepart" class="p-4 bg-slate-50 rounded-2xl font-black text-[10px] outline-none"><option>Libreville (Relais)</option><option>Akanda</option></select>
                            <select id="ptArrivee" class="p-4 bg-slate-50 rounded-2xl font-black text-[10px] outline-none"><option>Libreville (Client)</option><option>Owendo</option></select>
                        </div>
                        <div class="grid grid-cols-2 gap-3">
                            <select id="typeService" class="p-4 bg-slate-50 rounded-2xl font-black text-[10px] outline-none"><option value="standard">Standard</option><option value="express">Express</option></select>
                            <input type="number" id="prixLivraison" placeholder="Montant (CFA)" class="p-4 bg-slate-50 rounded-2xl font-bold text-[10px] outline-none border border-transparent focus:border-emerald-500">
                        </div>
                        <button onclick="genererMission()" class="w-full bg-emerald-600 text-white font-black py-5 rounded-2xl shadow-lg active:scale-95 transition text-xs">CRÉER LA MISSION</button>
                    </div>
                </div>
            </section>

            <!-- GESTIONNAIRE : DISPATCH -->
            <section id="rubrique2" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-4 ml-2">Flux à dispatcher</h3>
                <div id="containerDispatch"></div>
            </section>

            <!-- LIVREUR : MISSIONS -->
            <section id="rubrique3" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-4 ml-2">Missions de livraison</h3>
                <div id="containerLivreur"></div>
            </section>

            <!-- ARCHIVES -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-slate-900 rounded-[2.5rem] overflow-hidden shadow-2xl">
                    <div class="p-6 flex justify-between items-center"><h2 class="text-white font-black text-xs uppercase">Historique Livraisons</h2><div id="archiveCount" class="bg-yellow-500 text-slate-900 font-black text-[9px] px-3 py-1 rounded-full">0</div></div>
                    <div class="overflow-x-auto">
                        <table class="w-full text-left"><thead class="bg-white/5 text-slate-500 text-[8px] uppercase font-black"><tr><th class="p-4">ID</th><th class="p-4">Client</th><th class="p-4 text-center">Statut</th><th class="p-4 text-right">CFA</th></tr></thead><tbody id="archiveBody"></tbody></table>
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
            <h2 class="text-2xl font-black">Photo Obligatoire</h2>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-5 rounded-3xl shadow-2xl">PRENDRE LA PHOTO</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 text-[10px] font-bold uppercase">Annuler</button>
        </div>
    </div>

</body>
</html>
