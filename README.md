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
        const appId = 'ct241-service-de-livraison';

        // État de l'application
        let currentUser = null;
        let currentRole = 'admin';
        let currentMissionId = null;
        let nextGeneratedId = "";
        let allMissions = [];

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
                prepareNextId();
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
        const prepareNextId = () => {
            nextGeneratedId = "CT-" + Math.floor(Math.random() * 900000 + 100000);
            const idDisplay = document.getElementById('displayNextId');
            const idInput = document.getElementById('visibleIdInput');
            if (idDisplay) idDisplay.innerText = nextGeneratedId;
            if (idInput) idInput.value = nextGeneratedId;
        };

        window.genererMission = async () => {
            if (!currentUser) return;
            
            const fields = {
                client: document.getElementById('clientNom').value,
                telephone: document.getElementById('clientTel').value,
                depart: document.getElementById('ptDepart').value,
                arrivee: document.getElementById('ptArrivee').value,
                contenu: document.getElementById('colisContenu').value,
                service: document.getElementById('typeService').value,
                prix: document.getElementById('prixLivraison').value
            };
            
            if (!fields.client || !fields.telephone) {
                return showToast("Nom et Téléphone requis", "error");
            }

            const id = nextGeneratedId;
            const missionRef = doc(db, 'artifacts', appId, 'public', 'data', 'missions', id);

            try {
                await setDoc(missionRef, {
                    ...fields,
                    id,
                    statut: 'en_attente',
                    created_at: serverTimestamp(),
                    created_by: currentUser.uid,
                    creator_email: currentUser.email
                });
                showToast(`Colis ${id} enregistré !`, "success");
                
                // Reset formulaire
                document.getElementById('clientNom').value = "";
                document.getElementById('clientTel').value = "";
                document.getElementById('colisContenu').value = "";
                document.getElementById('prixLivraison').value = "";
                
                // Préparer le prochain ID
                prepareNextId();
            } catch (e) {
                showToast("Erreur de sauvegarde", "error");
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
                allMissions = missions;
                renderUI(missions);
            }, (error) => {
                console.error("Erreur Firestore:", error);
            });
        };

        window.openArchiveDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            if (!m) return;
            
            const detailContent = document.getElementById('detailContent');
            detailContent.innerHTML = `
                <div class="print-area bg-white p-6 space-y-4 text-slate-900 border-2 border-slate-100 rounded-3xl">
                    <div class="flex justify-between items-start border-b-2 border-slate-100 pb-4">
                        <div>
                            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-10 mb-2">
                            <h2 class="text-xl font-black">REÇU DE LIVRAISON</h2>
                            <p class="text-[10px] text-slate-400 font-bold uppercase">Preuve de Service CT241</p>
                        </div>
                        <div class="text-right">
                            <span class="text-lg font-black font-mono text-emerald-600">${m.id}</span>
                            <p class="text-[10px] text-slate-400">${new Date(m.livre_le).toLocaleString()}</p>
                        </div>
                    </div>
                    
                    <div class="grid grid-cols-2 gap-4 text-xs">
                        <div class="p-3 bg-slate-50 rounded-xl">
                            <p class="font-black uppercase text-[8px] text-slate-400 mb-1">Expéditeur</p>
                            <p class="font-bold">${m.client}</p>
                            <p class="text-[10px]">${m.telephone}</p>
                        </div>
                        <div class="p-3 bg-slate-50 rounded-xl">
                            <p class="font-black uppercase text-[8px] text-slate-400 mb-1">Trajet</p>
                            <p class="font-bold">${m.depart} → ${m.arrivee}</p>
                            <p class="text-[10px] uppercase">${m.service}</p>
                        </div>
                    </div>

                    <div class="p-3 border-2 border-slate-50 rounded-xl space-y-2">
                        <p class="font-black uppercase text-[8px] text-slate-400">Détails Colis</p>
                        <p class="text-xs">${m.contenu || 'Aucune description'}</p>
                        <div class="flex justify-between items-center pt-2 border-t border-slate-50">
                            <span class="font-bold">Total Réglé:</span>
                            <span class="font-black text-sm">${m.prix || '0'} FCFA</span>
                        </div>
                    </div>

                    <div class="space-y-2">
                        <p class="font-black uppercase text-[8px] text-slate-400">Photo de Confirmation</p>
                        <img src="${m.photoUrl}" class="w-full h-48 object-cover rounded-2xl border-2 border-slate-100">
                    </div>

                    <div class="text-center pt-4 border-t border-slate-100 italic text-[9px] text-slate-400">
                        Merci de votre confiance. Pour toute réclamation, munissez-vous de cet identifiant.
                    </div>
                </div>
            `;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        window.printCurrentArchive = () => {
            window.print();
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
                    containers.dispatch.innerHTML += `<div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 shadow-sm space-y-2">
                        <div class="flex justify-between items-center">
                            <span class="text-xs font-black text-slate-900">${m.id}</span>
                            <span class="text-[9px] font-bold px-2 py-0.5 rounded bg-slate-100 uppercase">${m.service}</span>
                        </div>
                        <p class="text-[11px] text-slate-600"><b>${m.client}</b> (${m.telephone})<br>${m.depart} → ${m.arrivee}</p>
                        <button onclick="publierMission('${m.id}')" class="w-full bg-blue-600 text-white text-[10px] py-2 rounded-xl font-bold">LANCER DISPATCH</button>
                    </div>`;
                } else if (m.statut === 'publie') {
                    containers.livreur.innerHTML += `<div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 space-y-3">
                        <div class="flex justify-between font-black text-amber-900 items-center">
                            <span>${m.id}</span>
                            <span class="text-[9px] bg-amber-500 text-white px-2 py-1 rounded-full">EN COURS</span>
                        </div>
                        <div class="text-[11px] text-amber-800 space-y-1">
                            <p><b>Client:</b> ${m.client}</p>
                            <p><b>Contenu:</b> ${m.contenu || 'Non spécifié'}</p>
                            <p><b>Destination:</b> ${m.arrivee}</p>
                        </div>
                        <button onclick="openCamera('${m.id}')" class="w-full bg-amber-500 text-white font-bold py-4 rounded-2xl shadow-lg active:scale-95 transition-transform">VALIDER LIVRAISON</button>
                    </div>`;
                } else if (m.statut === 'livre') {
                    arcCount++;
                    containers.archives.innerHTML += `<tr class="border-b border-slate-700 text-[10px] hover:bg-slate-800 transition cursor-pointer" onclick="openArchiveDetail('${m.id}')">
                        <td class="p-4 text-yellow-500 font-bold">${m.id}</td>
                        <td class="p-4">${m.client}<br><span class="text-slate-500">${m.arrivee}</span></td>
                        <td class="p-4"><img src="${m.photoUrl}" class="w-8 h-8 rounded-lg object-cover"></td>
                        <td class="p-4 text-slate-500">${new Date(m.livre_le).toLocaleDateString()}</td>
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
        body { font-family: 'Plus Jakarta Sans', sans-serif; letter-spacing: -0.01em; }
        .hidden { display: none; }
        .logo-img { max-height: 44px; width: auto; }
        .logo-login { max-height: 140px; width: auto; margin: 0 auto; }
        input, select, textarea { appearance: none; }

        @media print {
            body * { visibility: hidden; }
            .print-area, .print-area * { visibility: visible; }
            .print-area { 
                position: fixed; 
                left: 0; 
                top: 0; 
                width: 100%; 
                border: none !important; 
                padding: 0 !important;
            }
            .no-print { display: none !important; }
        }
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
        <nav class="bg-white text-slate-900 p-3 shadow-md sticky top-0 z-50 border-b border-slate-100 no-print">
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

        <div class="max-w-xl mx-auto p-4 space-y-6 mt-4 no-print">
            <!-- SECTION RELAIS : FORMULAIRE ENRICHI -->
            <section id="rubrique1" class="role-section">
                <div class="bg-white rounded-[2.5rem] shadow-xl border border-slate-100 overflow-hidden">
                    <div class="bg-emerald-600 p-6 text-white flex items-center justify-between">
                        <div class="flex items-center gap-3">
                            <span class="text-2xl">📦</span>
                            <div>
                                <h2 class="font-black text-sm uppercase tracking-tighter leading-none">Réception Colis</h2>
                                <p class="text-[10px] text-emerald-100 opacity-80">Enregistrement nouveau départ</p>
                            </div>
                        </div>
                        <div class="bg-emerald-900/30 px-3 py-1.5 rounded-xl border border-emerald-400/20">
                            <span class="text-[10px] font-black uppercase tracking-wider block opacity-70">Suivi ID</span>
                            <span id="displayNextId" class="text-xs font-black font-mono">CT-XXXXXX</span>
                        </div>
                    </div>
                    
                    <div class="p-8 space-y-5">
                        <div class="space-y-1">
                            <label class="text-[10px] font-black text-slate-400 uppercase ml-2 tracking-widest">Identifiant de suivi (Généré)</label>
                            <input type="text" id="visibleIdInput" readonly class="w-full p-4 bg-slate-100 rounded-2xl font-black text-sm border-2 border-slate-200 text-slate-500 outline-none cursor-not-allowed font-mono">
                        </div>

                        <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
                            <div class="space-y-1">
                                <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Expéditeur</label>
                                <input type="text" id="clientNom" placeholder="Nom complet" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-sm border-2 border-transparent focus:border-emerald-500 outline-none transition-all">
                            </div>
                            <div class="space-y-1">
                                <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Téléphone Client</label>
                                <input type="tel" id="clientTel" placeholder="+241 0X XX XX XX" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-sm border-2 border-transparent focus:border-emerald-500 outline-none transition-all">
                            </div>
                        </div>

                        <div class="space-y-1">
                            <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Description du contenu</label>
                            <textarea id="colisContenu" rows="2" placeholder="Ex: Documents fragiles, Électronique..." class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-sm border-2 border-transparent focus:border-emerald-500 outline-none transition-all"></textarea>
                        </div>

                        <div class="grid grid-cols-2 gap-4">
                            <div class="space-y-1">
                                <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Point de Départ</label>
                                <select id="ptDepart" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-sm border-2 border-transparent focus:border-emerald-500 outline-none">
                                    <option>Nzeng-Ayong</option>
                                    <option>Akanda</option>
                                    <option>Owendo</option>
                                    <option>Louis</option>
                                    <option>Glass</option>
                                </select>
                            </div>
                            <div class="space-y-1">
                                <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Point d'Arrivée</label>
                                <select id="ptArrivee" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-sm border-2 border-transparent focus:border-emerald-500 outline-none">
                                    <option>Akanda</option>
                                    <option>Owendo</option>
                                    <option>Louis</option>
                                    <option>Nzeng-Ayong</option>
                                    <option>Okala</option>
                                </select>
                            </div>
                        </div>

                        <div class="grid grid-cols-2 gap-4">
                            <div class="space-y-1">
                                <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Type de Service</label>
                                <select id="typeService" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-sm border-2 border-transparent focus:border-emerald-500 outline-none">
                                    <option value="standard">Standard (24h)</option>
                                    <option value="express">Express (3h)</option>
                                    <option value="prioritaire">Prioritaire (1h)</option>
                                </select>
                            </div>
                            <div class="space-y-1">
                                <label class="text-[10px] font-black text-slate-400 uppercase ml-2">Prix (FCFA)</label>
                                <input type="number" id="prixLivraison" placeholder="2500" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-sm border-2 border-transparent focus:border-emerald-500 outline-none transition-all">
                            </div>
                        </div>

                        <button onclick="genererMission()" class="w-full bg-emerald-600 text-white font-black py-5 rounded-[1.5rem] shadow-xl hover:bg-emerald-700 transition-all active:scale-[0.98] mt-4 flex items-center justify-center gap-2 tracking-tighter">
                            <span>🚀</span> ENREGISTRER LE COLIS
                        </button>
                    </div>
                </div>
            </section>

            <section id="rubrique2" class="role-section">
                <h3 class="text-xs font-black text-slate-400 uppercase tracking-widest mb-3 ml-4">Colis en attente de Dispatch</h3>
                <div id="containerDispatch"></div>
            </section>

            <section id="rubrique3" class="role-section">
                <h3 class="text-xs font-black text-slate-400 uppercase tracking-widest mb-3 ml-4">Missions Disponibles (Livreur)</h3>
                <div id="containerLivreur"></div>
            </section>

            <section id="rubrique4" class="role-section">
                <div class="bg-slate-900 rounded-[2.5rem] overflow-hidden shadow-2xl">
                    <div class="p-6 flex justify-between items-center border-b border-slate-800">
                        <div>
                            <h2 class="text-white font-black text-sm uppercase">Archives Globales</h2>
                            <p class="text-[9px] text-slate-500">Historique des livraisons réussies</p>
                        </div>
                        <span id="archiveCount" class="bg-yellow-500 text-slate-900 font-black text-[10px] px-3 py-1 rounded-full">0</span>
                    </div>
                    <div class="overflow-x-auto">
                        <table class="w-full text-left">
                            <thead class="bg-slate-800 text-slate-400 text-[9px] uppercase font-black tracking-widest">
                                <tr>
                                    <th class="p-4">ID</th>
                                    <th class="p-4">Client</th>
                                    <th class="p-4">Preuve</th>
                                    <th class="p-4">Date</th>
                                </tr>
                            </thead>
                            <tbody id="archiveBody"></tbody>
                        </table>
                    </div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL DETAIL / IMPRESSION ARCHIVE -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/80 z-[400] hidden flex items-center justify-center p-4">
        <div class="w-full max-w-lg bg-white rounded-[2.5rem] shadow-2xl overflow-hidden animate-in slide-in-from-bottom-10 duration-300">
            <div id="detailContent" class="p-2">
                <!-- Injecté via JS -->
            </div>
            <div class="p-6 bg-slate-50 flex gap-3 no-print">
                <button onclick="printCurrentArchive()" class="flex-1 bg-slate-900 text-white font-black py-4 rounded-2xl flex items-center justify-center gap-2">
                    <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 17h2a2 2 0 002-2v-4a2 2 0 00-2-2H5a2 2 0 00-2 2v4a2 2 0 002 2h2m2 4h6a2 2 0 002-2v-4a2 2 0 00-2-2H9a2 2 0 00-2 2v4a2 2 0 002 2zm8-12V5a2 2 0 00-2-2H9a2 2 0 00-2 2v4h10z" /></svg>
                    IMPRIMER LE REÇU
                </button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-6 bg-slate-200 text-slate-600 font-black py-4 rounded-2xl">FERMER</button>
            </div>
        </div>
    </div>

    <!-- MODAL CAMÉRA -->
    <div id="cameraModal" class="fixed inset-0 bg-slate-950/95 z-[300] hidden flex flex-col items-center justify-center p-6 text-white text-center">
        <div class="w-20 h-20 bg-amber-500 rounded-full flex items-center justify-center mb-6 text-3xl animate-pulse">📸</div>
        <h2 class="text-2xl font-black mb-2">Preuve de Livraison</h2>
        <p class="text-xs text-slate-400 mb-10 max-w-xs">Photographiez le colis déposé ou le destinataire avec le colis.</p>
        <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
        <button onclick="document.getElementById('fileInput').click()" class="w-full max-w-xs bg-white text-slate-900 font-black py-6 rounded-[2rem] text-lg shadow-2xl hover:scale-105 transition-transform">
            ACTIVER L'APPAREIL PHOTO
        </button>
        <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="mt-8 text-slate-500 text-xs font-bold underline uppercase tracking-widest">Annuler la procédure</button>
    </div>

</body>
</html>
