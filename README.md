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

        // Configuration Firebase personnalisée
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

        // Initialisation
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = 'ct241-service-de-livraison'; // Identifiant unique lié à votre projet

        // État de l'application
        let currentUser = null;
        let currentRole = 'admin';
        let currentMissionId = null;
        const LOGO_URL = "https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png";

        // --- AUTHENTIFICATION ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            const errorEl = document.getElementById('loginError');

            btn.disabled = true;
            btn.innerText = "Connexion...";
            errorEl.classList.add('hidden');

            try {
                await signInWithEmailAndPassword(auth, email, pass);
            } catch (error) {
                errorEl.innerText = "Identifiants invalides. Veuillez réessayer.";
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
            const loginSection = document.getElementById('authSection');
            const appSection = document.getElementById('appContent');
            
            if (user) {
                currentUser = user;
                loginSection.classList.add('hidden');
                appSection.classList.remove('hidden');
                document.getElementById('userDisplayEmail').innerText = user.email;
                switchRole('admin');
                startListeners();
            } else {
                currentUser = null;
                loginSection.classList.remove('hidden');
                appSection.classList.add('hidden');
            }
        });

        // --- GESTION DES RÔLES ---
        window.switchRole = (role) => {
            currentRole = role;
            document.querySelectorAll('.role-section').forEach(s => s.classList.add('hidden'));
            
            const badge = document.getElementById('badgeDisplay');
            const colors = { 
                admin: 'bg-red-500', 
                gestionnaire: 'bg-blue-500', 
                relais: 'bg-emerald-500', 
                livreur: 'bg-amber-500' 
            };
            badge.innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-bold uppercase text-white ${colors[role]}">${role}</span>`;

            if (role === 'admin') {
                document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
            } else {
                const sectionMap = {
                    relais: 'rubrique1',
                    gestionnaire: ['rubrique2', 'rubrique4'],
                    livreur: 'rubrique3'
                };
                const targets = Array.isArray(sectionMap[role]) ? sectionMap[role] : [sectionMap[role]];
                targets.forEach(id => document.getElementById(id).classList.remove('hidden'));
            }
        };

        // --- LOGIQUE MÉTIER ---
        window.genererMission = async () => {
            if (!currentUser) return;
            const client = document.getElementById('clientNom').value;
            const dep = document.getElementById('ptDepart').value;
            const arr = document.getElementById('ptArrivee').value;
            
            if (!client) return showToast("Nom du client requis", "error");

            const id = "CT-" + Math.floor(Math.random() * 900000 + 100000);
            const missionRef = doc(db, 'artifacts', appId, 'public', 'data', 'missions', id);

            try {
                await setDoc(missionRef, {
                    id, client, depart: dep, arrivee: arr,
                    statut: 'en_attente',
                    created_at: serverTimestamp(),
                    created_by: currentUser.uid,
                    creator_email: currentUser.email
                });
                showToast(`Mission ${id} créée !`, "success");
                document.getElementById('clientNom').value = "";
            } catch (e) {
                showToast("Erreur de permission base de données", "error");
            }
        };

        window.publierMission = async (id) => {
            if (!currentUser) return;
            const ref = doc(db, 'artifacts', appId, 'public', 'data', 'missions', id);
            await updateDoc(ref, { statut: 'publie' });
        };

        window.openCamera = (id) => {
            currentMissionId = id;
            document.getElementById('cameraModal').classList.remove('hidden');
        };

        window.processImage = (file) => {
            if (!file) return;
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    const ctx = canvas.getContext('2d');
                    const MAX_WIDTH = 800;
                    canvas.width = MAX_WIDTH;
                    canvas.height = img.height * (MAX_WIDTH / img.width);
                    ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
                    finalizeDelivery(canvas.toDataURL('image/jpeg', 0.6));
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };

        const finalizeDelivery = async (photoData) => {
            if (!currentUser) return;
            const ref = doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId);
            await updateDoc(ref, {
                statut: 'livre',
                photoUrl: photoData,
                livre_le: Date.now(),
                livreur_id: currentUser.uid
            });
            document.getElementById('cameraModal').classList.add('hidden');
            showToast("Livraison confirmée !", "success");
        };

        const startListeners = () => {
            if (!currentUser) return;
            const q = collection(db, 'artifacts', appId, 'public', 'data', 'missions');
            onSnapshot(q, (snapshot) => {
                const missions = [];
                snapshot.forEach(doc => missions.push(doc.data()));
                renderUI(missions);
            }, (error) => {
                console.error("Erreur Firestore:", error);
                showToast("Erreur de synchronisation", "error");
            });
        };

        const renderUI = (missions) => {
            const containers = {
                dispatch: document.getElementById('containerDispatch'),
                livreur: document.getElementById('containerLivreur'),
                archives: document.getElementById('archiveBody')
            };
            Object.values(containers).forEach(c => c.innerHTML = "");
            
            let arcCount = 0;
            missions.sort((a,b) => (b.created_at?.seconds || 0) - (a.created_at?.seconds || 0)).forEach(m => {
                if (m.statut === 'en_attente') {
                    containers.dispatch.innerHTML += `<div class="flex justify-between items-center p-4 bg-white border border-slate-100 rounded-xl mb-2 shadow-sm">
                        <div class="text-xs font-bold">${m.id}<br><span class="font-normal text-slate-500">${m.depart} → ${m.arrivee}</span></div>
                        <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white text-[10px] px-3 py-2 rounded-lg font-bold">PUBLIER</button>
                    </div>`;
                } else if (m.statut === 'publie') {
                    containers.livreur.innerHTML += `<div class="p-4 bg-amber-50 border border-amber-200 rounded-2xl mb-4 space-y-3">
                        <div class="flex justify-between font-black text-amber-900"><span>${m.id}</span><span class="text-[10px] bg-amber-200 px-2 py-1 rounded">EN COURS</span></div>
                        <p class="text-xs text-amber-800 font-medium"><b>Client:</b> ${m.client}<br><b>Trajet:</b> ${m.depart} vers ${m.arrivee}</p>
                        <button onclick="openCamera('${m.id}')" class="w-full bg-amber-500 text-white font-bold py-3 rounded-xl shadow-md">SCANNER PREUVE</button>
                    </div>`;
                } else if (m.statut === 'livre') {
                    arcCount++;
                    containers.archives.innerHTML += `<tr class="border-b border-slate-700 text-[10px]">
                        <td class="p-3 text-yellow-500 font-bold">${m.id}</td>
                        <td class="p-3">${m.depart}<br>→ ${m.arrivee}</td>
                        <td class="p-3"><img src="${m.photoUrl}" class="w-8 h-8 rounded cursor-pointer" onclick="window.open('${m.photoUrl}')"></td>
                        <td class="p-3 text-slate-500">${new Date(m.livre_le).toLocaleDateString()}</td>
                    </tr>`;
                }
            });
            document.getElementById('archiveCount').innerText = arcCount;
        };

        const showToast = (msg, type) => {
            const t = document.getElementById('toast');
            t.innerText = msg;
            t.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-sm shadow-xl z-[300] transition-all ${type==='success'?'bg-emerald-500':'bg-red-500'}`;
            t.style.display = 'block';
            setTimeout(() => t.style.display = 'none', 3000);
        };
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; }
        .hidden { display: none; }
        .logo-img { max-height: 44px; width: auto; }
        .logo-login { max-height: 140px; width: auto; margin: 0 auto; }
    </style>
</head>
<body class="bg-slate-50 min-h-screen">

    <div id="toast" class="hidden"></div>

    <!-- ÉCRAN DE CONNEXION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[2.5rem] shadow-2xl p-10 space-y-8 animate-in fade-in zoom-in duration-300">
            <div class="text-center space-y-4">
                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" alt="Logo CT241" class="logo-login">
                <div class="space-y-1">
                    <h1 class="text-2xl font-black tracking-tighter text-slate-900">Bienvenue</h1>
                    <p class="text-[10px] font-bold text-slate-400 uppercase tracking-widest">Système Logistique Sécurisé</p>
                </div>
            </div>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <div class="space-y-1">
                    <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Email Professionnel</label>
                    <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-slate-900 outline-none transition-all font-semibold" placeholder="votre@email.com">
                </div>
                <div class="space-y-1">
                    <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Mot de passe</label>
                    <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-slate-900 outline-none transition-all font-semibold" placeholder="••••••••">
                </div>
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden bg-red-50 p-2 rounded-lg text-center"></p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-extrabold py-4 rounded-2xl hover:bg-slate-800 transition-all active:scale-95 shadow-xl">
                    SE CONNECTER
                </button>
            </form>
            <p class="text-[9px] text-slate-400 text-center font-medium italic">Accès restreint au personnel CT241 Gabon.</p>
        </div>
    </section>

    <!-- CONTENU APPLICATION -->
    <main id="appContent" class="hidden pb-24">
        <nav class="bg-white text-slate-900 p-3 shadow-md sticky top-0 z-50 border-b border-slate-100">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" alt="Logo CT241" class="logo-img">
                    <div class="hidden sm:block">
                        <h1 class="text-sm font-black leading-none">CT241 GABON</h1>
                        <p class="text-[8px] text-slate-400 font-bold" id="userDisplayEmail">...</p>
                    </div>
                </div>
                <div class="flex items-center gap-3">
                    <select id="roleSelector" onchange="switchRole(this.value)" class="bg-slate-100 text-[10px] font-black border-none rounded-lg px-2 py-1 outline-none">
                        <option value="admin">ADMIN</option>
                        <option value="relais">RELAIS</option>
                        <option value="gestionnaire">DISPATCH</option>
                        <option value="livreur">LIVREUR</option>
                    </select>
                    <button onclick="handleLogout()" class="p-2 text-slate-400 hover:text-red-500 transition">
                        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m0 0l-4-4m4 4H7m6 4v1a3 3 0 01-3 3H6a3 3 0 01-3-3V7a3 3 0 013-3h4a3 3 0 013 3v1" /></svg>
                    </button>
                </div>
            </div>
            <div class="max-w-xl mx-auto mt-2 flex justify-end" id="badgeDisplay"></div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6 mt-4">
            <section id="rubrique1" class="role-section">
                <div class="bg-white rounded-3xl shadow-lg border border-slate-100 overflow-hidden">
                    <div class="bg-emerald-600 p-5 text-white font-extrabold flex items-center gap-2"><span>📦</span> Relais : Création</div>
                    <div class="p-6 space-y-4">
                        <input type="text" id="clientNom" placeholder="Nom du client" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-sm border-2 border-transparent focus:border-emerald-500 outline-none">
                        <div class="grid grid-cols-2 gap-2">
                            <select id="ptDepart" class="p-3 bg-slate-50 rounded-xl text-xs font-bold"><option>Nzeng-Ayong</option><option>Akanda</option><option>Owendo</option></select>
                            <select id="ptArrivee" class="p-3 bg-slate-50 rounded-xl text-xs font-bold"><option>Akanda</option><option>Owendo</option><option>Louis</option></select>
                        </div>
                        <button onclick="genererMission()" class="w-full bg-emerald-600 text-white font-black py-4 rounded-2xl shadow-lg">ENREGISTRER COLIS</button>
                    </div>
                </div>
            </section>

            <section id="rubrique2" class="role-section">
                <h3 class="text-xs font-black text-slate-400 uppercase tracking-widest mb-3 ml-2">En attente de Dispatch</h3>
                <div id="containerDispatch"></div>
            </section>

            <section id="rubrique3" class="role-section">
                <h3 class="text-xs font-black text-slate-400 uppercase tracking-widest mb-3 ml-2">Missions Disponibles</h3>
                <div id="containerLivreur"></div>
            </section>

            <section id="rubrique4" class="role-section">
                <div class="bg-slate-900 rounded-3xl overflow-hidden shadow-2xl">
                    <div class="p-5 flex justify-between items-center"><h2 class="text-white font-black text-xs">ARCHIVES GLOBALES</h2><span id="archiveCount" class="text-yellow-400 font-bold text-xs">0</span></div>
                    <div class="overflow-x-auto"><table class="w-full text-left"><tbody id="archiveBody"></tbody></table></div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL CAMÉRA -->
    <div id="cameraModal" class="fixed inset-0 bg-slate-950/95 z-[300] hidden flex flex-col items-center justify-center p-6 text-white text-center">
        <div class="w-16 h-16 bg-amber-500 rounded-full flex items-center justify-center mb-6 text-2xl animate-bounce">📷</div>
        <h2 class="text-xl font-black mb-2">Preuve de Livraison</h2>
        <p class="text-xs text-slate-400 mb-8 max-w-xs">Prenez une photo du colis entre les mains du destinataire.</p>
        <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
        <button onclick="document.getElementById('fileInput').click()" class="w-full max-w-xs bg-white text-slate-900 font-black py-5 rounded-2xl text-lg shadow-2xl">OUVRIR LA CAMÉRA</button>
        <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="mt-6 text-slate-500 text-xs font-bold underline uppercase">Fermer</button>
    </div>

</body>
</html>
