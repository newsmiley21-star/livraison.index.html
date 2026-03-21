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
        import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        // Configuration Firebase
        const firebaseConfig = typeof __firebase_config !== 'undefined' 
            ? JSON.parse(__firebase_config) 
            : {
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
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'ct241-service-de-livraison';

        let user = null;
        let userRole = 'admin'; // Par défaut admin pour le test ou accès complet
        let allMissions = [];
        let currentMissionId = null;

        // --- AUTHENTIFICATION ---
        const initAuth = async () => {
            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (err) {
                console.error("Auth error:", err);
            }
        };

        onAuthStateChanged(auth, (u) => {
            user = u;
            if (user) {
                document.getElementById('userDisplayEmail').innerText = user.email || "Utilisateur Anonyme";
                assignRole(user.email || "");
                startListeners();
                prepareNextId();
                document.getElementById('appContent').classList.remove('hidden');
            }
        });

        const assignRole = (email) => {
            const e = email.toLowerCase();
            if (e.includes('admin')) userRole = 'admin';
            else if (e.includes('relais')) userRole = 'relais';
            else if (e.includes('dispatch')) userRole = 'dispatch';
            else if (e.includes('livreur')) userRole = 'livreur';
            
            // UI Update
            document.querySelectorAll('.role-section').forEach(s => s.classList.add('hidden'));
            if (userRole === 'admin') {
                document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
            } else if (userRole === 'relais') {
                document.getElementById('rubrique1').classList.remove('hidden');
                document.getElementById('rubrique4').classList.remove('hidden');
            } else if (userRole === 'livreur') {
                document.getElementById('rubrique3').classList.remove('hidden');
                document.getElementById('statLivreurSection').classList.remove('hidden');
            } else {
                document.getElementById('rubrique2').classList.remove('hidden');
                document.getElementById('rubrique4').classList.remove('hidden');
            }
        };

        // --- GESTION DES MISSIONS ---
        const prepareNextId = () => {
            const id = "2026-" + Math.floor(Math.random() * 900000 + 100000);
            const el = document.getElementById('displayNextId');
            if (el) el.innerText = id;
            window.nextId = id;
        };

        window.genererMission = async () => {
            if (!user) return;
            const data = {
                exp_nom: document.getElementById('expNom').value,
                exp_quartier: document.getElementById('expQuartier').value,
                exp_tel: document.getElementById('expTel').value,
                dest_nom: document.getElementById('destNom').value,
                dest_quartier: document.getElementById('destQuartier').value,
                dest_tel: document.getElementById('destTel').value,
                nature: document.getElementById('natureColis').value,
                valeur: parseFloat(document.getElementById('valeurDeclaree').value) || 0,
                prix: parseFloat(document.getElementById('fraisLivraison').value) || 0,
                reglement: document.getElementById('modeReglement').value,
                id: window.nextId,
                statut: 'en_attente',
                created_at: serverTimestamp(),
                creator_uid: user.uid
            };

            if (!data.exp_nom || !data.dest_tel || !data.prix) {
                return showToast("Champs obligatoires manquants", "error");
            }

            try {
                // RÈGLE 1: Chemin strict /artifacts/{appId}/public/data/{collection}
                const missionRef = doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId);
                await setDoc(missionRef, data);
                showToast("Mission créée avec succès", "success");
                resetForm();
                prepareNextId();
            } catch (e) {
                console.error(e);
                showToast("Erreur système lors de la création", "error");
            }
        };

        const resetForm = () => {
            ['expNom', 'expQuartier', 'expTel', 'destNom', 'destTel', 'valeurDeclaree', 'fraisLivraison'].forEach(id => {
                const el = document.getElementById(id);
                if (el) el.value = "";
            });
        };

        window.publierMission = async (id) => {
            if (!user) return;
            try {
                const ref = doc(db, 'artifacts', appId, 'public', 'data', 'missions', id);
                await updateDoc(ref, { statut: 'publie' });
                showToast("Mission publiée", "success");
            } catch (e) { showToast("Erreur système", "error"); }
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
                    const MAX = 600;
                    canvas.width = MAX;
                    canvas.height = img.height * (MAX / img.width);
                    canvas.getContext('2d').drawImage(img, 0, 0, canvas.width, canvas.height);
                    finalizeDelivery(canvas.toDataURL('image/jpeg', 0.5));
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };

        const finalizeDelivery = async (photo) => {
            if (!user) return;
            try {
                const today = new Date().toISOString().split('T')[0];
                const ref = doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId);
                await updateDoc(ref, {
                    statut: 'livre',
                    photoUrl: photo,
                    livre_le: Date.now(),
                    livre_date_key: today,
                    livreur_uid: user.uid,
                    livreur_email: user.email || "Anonyme"
                });
                document.getElementById('cameraModal').classList.add('hidden');
                showToast("Livraison confirmée", "success");
            } catch (e) { showToast("Erreur système", "error"); }
        };

        const startListeners = () => {
            // RÈGLE 2: Requête simple sur la collection
            const q = collection(db, 'artifacts', appId, 'public', 'data', 'missions');
            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(d => allMissions.push(d.data()));
                renderUI();
                renderStats();
            }, (err) => console.error("Firestore Listen Error:", err));
        };

        const renderUI = () => {
            const containers = {
                dispatch: document.getElementById('containerDispatch'),
                livreur: document.getElementById('containerLivreur'),
                archives: document.getElementById('archiveBody')
            };
            Object.values(containers).forEach(c => { if(c) c.innerHTML = ""; });

            // Tri en mémoire JS (RÈGLE 2)
            const sorted = [...allMissions].sort((a,b) => (b.livre_le || 0) - (a.livre_le || 0));

            sorted.forEach(m => {
                if (m.statut === 'en_attente' && (userRole === 'admin' || userRole === 'dispatch')) {
                    containers.dispatch.innerHTML += `
                        <div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 shadow-sm flex justify-between items-center">
                            <div class="text-[11px]"><span class="font-black text-blue-600 block">${m.id}</span><b>${m.dest_nom}</b><br><span class="text-slate-400">${m.dest_quartier}</span></div>
                            <div class="flex gap-2">
                                <button onclick="openBonImpression('${m.id}')" class="bg-slate-100 text-slate-600 text-[9px] px-3 py-2 rounded-xl font-bold">BON</button>
                                <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white text-[10px] px-5 py-2 rounded-xl font-bold">DISPATCHER</button>
                            </div>
                        </div>`;
                } else if (m.statut === 'publie' && (userRole === 'admin' || userRole === 'livreur')) {
                    containers.livreur.innerHTML += `
                        <div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 space-y-3">
                            <div class="flex justify-between font-black items-center text-amber-700"><span>${m.id}</span><span class="text-[9px] bg-amber-500 text-white px-2 py-1 rounded-full uppercase">Disponible</span></div>
                            <p class="text-[11px] text-amber-900"><b>Lieu:</b> ${m.dest_quartier}<br><b>Contact:</b> ${m.dest_nom} (${m.dest_tel})</p>
                            <div class="flex gap-2">
                                <button onclick="openBonImpression('${m.id}')" class="flex-1 bg-white border border-amber-200 text-amber-700 font-black py-4 rounded-2xl text-[10px]">REÇU COURSE</button>
                                <button onclick="openCamera('${m.id}')" class="flex-[2] bg-amber-500 text-white font-black py-4 rounded-2xl shadow-lg">LIVRER</button>
                            </div>
                        </div>`;
                }

                if (m.statut === 'livre' && (userRole === 'admin' || userRole === 'dispatch')) {
                    containers.archives.innerHTML += `
                        <tr class="border-b border-slate-800 text-[10px] hover:bg-slate-800 transition cursor-pointer" onclick="openArchiveDetail('${m.id}')">
                            <td class="p-4 font-bold text-white">${m.id}</td>
                            <td class="p-4 text-slate-300">${m.dest_nom}</td>
                            <td class="p-4 text-center"><span class="text-emerald-400 font-black">LIVRÉ</span></td>
                            <td class="p-4 text-right text-slate-400">${(m.prix || 0).toLocaleString()}</td>
                        </tr>`;
                }
            });
        };

        const renderStats = () => {
            const today = new Date().toISOString().split('T')[0];
            let caTotal = 0; let count = 0;
            allMissions.forEach(m => {
                if(m.statut === 'livre') {
                    caTotal += m.prix || 0;
                    if(m.livre_date_key === today) count++;
                }
            });
            const caEl = document.getElementById('dashCaTotal');
            if(caEl) caEl.innerText = caTotal.toLocaleString() + ' CFA';
        };

        const showToast = (msg, type) => {
            const t = document.getElementById('toast');
            t.innerText = msg;
            t.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-xs shadow-2xl z-[500] ${type==='success'?'bg-emerald-600':'bg-red-600'}`;
            t.classList.remove('hidden');
            setTimeout(() => t.classList.add('hidden'), 3000);
        };

        window.openBonImpression = (id) => {
            const m = allMissions.find(x => x.id === id);
            if (!m) return;
            document.getElementById('detailContent').innerHTML = `
                <div class="p-8 bg-white text-slate-900">
                    <h2 class="text-2xl font-black mb-4">BON DE LIVRAISON ${m.id}</h2>
                    <p class="text-sm"><b>Destinataire:</b> ${m.dest_nom}</p>
                    <p class="text-sm"><b>Lieu:</b> ${m.dest_quartier}</p>
                    <p class="text-sm"><b>Frais:</b> ${m.prix} CFA</p>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        window.openArchiveDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            if (!m) return;
            document.getElementById('detailContent').innerHTML = `
                <div class="p-8 bg-white text-slate-900">
                    <h2 class="text-2xl font-black text-emerald-600 mb-4">DÉTAIL LIVRAISON ${m.id}</h2>
                    ${m.photoUrl ? `<img src="${m.photoUrl}" class="w-full rounded-xl mb-4">` : ''}
                    <p><b>Livreur:</b> ${m.livreur_email}</p>
                    <p><b>Montant:</b> ${m.prix} CFA</p>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        initAuth();
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; }
        .hidden { display: none; }
        @media print { .no-print { display: none; } }
    </style>
</head>
<body class="min-h-screen">
    <div id="toast" class="hidden"></div>

    <main id="appContent" class="hidden pb-24">
        <nav class="bg-white/80 backdrop-blur-md p-4 sticky top-0 z-50 border-b border-slate-100 no-print">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-8">
                    <div class="flex flex-col"><span class="text-[9px] font-black" id="userDisplayEmail">Chargement...</span></div>
                </div>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            <!-- DASHBOARD ADMIN -->
            <section id="adminDash" class="role-section hidden">
                <div class="bg-slate-900 p-6 rounded-[2.5rem] text-white shadow-xl">
                    <span class="text-[10px] uppercase font-black opacity-50">Chiffre d'Affaire Global</span>
                    <div id="dashCaTotal" class="text-3xl font-black text-emerald-400">0 CFA</div>
                </div>
            </section>

            <!-- STATS LIVREUR -->
            <section id="statLivreurSection" class="role-section hidden">
                <div class="bg-slate-900 p-6 rounded-[2.5rem] text-white">
                    <h3 class="text-xs font-black uppercase">Ma Performance</h3>
                    <div class="mt-4 h-2 bg-slate-800 rounded-full overflow-hidden">
                        <div id="progBar" class="h-full bg-emerald-500 w-0 transition-all"></div>
                    </div>
                </div>
            </section>

            <!-- FORMULAIRE (RELAIS/ADMIN) -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl border border-slate-100 overflow-hidden">
                    <div class="bg-emerald-600 p-6 text-white">
                        <h2 class="font-black text-xs uppercase">Nouveau Bon : <span id="displayNextId">...</span></h2>
                    </div>
                    <div class="p-6 space-y-4">
                        <input type="text" id="expNom" placeholder="Nom Expéditeur" class="w-full p-4 bg-slate-50 rounded-2xl text-xs font-bold outline-none">
                        <div class="grid grid-cols-2 gap-2">
                            <input type="text" id="expQuartier" placeholder="Quartier" class="p-4 bg-slate-50 rounded-2xl text-xs font-bold outline-none">
                            <input type="tel" id="expTel" placeholder="Tél Exp" class="p-4 bg-slate-50 rounded-2xl text-xs font-bold outline-none">
                        </div>
                        <hr class="border-slate-100">
                        <input type="text" id="destNom" placeholder="Nom Destinataire" class="w-full p-4 bg-slate-50 rounded-2xl text-xs font-bold outline-none">
                        <div class="grid grid-cols-2 gap-2">
                            <input type="text" id="destQuartier" placeholder="Quartier Livraison" class="p-4 bg-slate-50 rounded-2xl text-xs font-bold outline-none">
                            <input type="tel" id="destTel" placeholder="Tél Dest" class="p-4 bg-slate-50 rounded-2xl text-xs font-bold outline-none">
                        </div>
                        <div class="grid grid-cols-2 gap-2">
                            <select id="natureColis" class="p-4 bg-slate-50 rounded-2xl text-xs font-bold outline-none">
                                <option value="colis">Colis</option><option value="repas">Repas</option>
                            </select>
                            <select id="modeReglement" class="p-4 bg-slate-50 rounded-2xl text-xs font-bold outline-none">
                                <option value="cash">Cash</option><option value="mobile">Mobile Money</option>
                            </select>
                        </div>
                        <div class="grid grid-cols-2 gap-2">
                            <input type="number" id="valeurDeclaree" placeholder="Valeur" class="p-4 bg-slate-50 rounded-2xl text-xs font-bold outline-none">
                            <input type="number" id="fraisLivraison" placeholder="FRAIS" class="p-4 bg-slate-100 rounded-2xl text-xs font-black border-2 border-emerald-600 outline-none">
                        </div>
                        <button onclick="genererMission()" class="w-full bg-emerald-600 text-white font-black py-5 rounded-3xl shadow-lg uppercase text-xs">Créer le Bon</button>
                    </div>
                </div>
            </section>

            <!-- LISTE DISPATCH (DISPATCH/ADMIN) -->
            <section id="rubrique2" class="role-section hidden">
                <h3 class="font-black text-xs uppercase text-slate-400 mb-3">Missions à dispatcher</h3>
                <div id="containerDispatch"></div>
            </section>

            <!-- LISTE LIVREUR (LIVREUR/ADMIN) -->
            <section id="rubrique3" class="role-section hidden">
                <h3 class="font-black text-xs uppercase text-slate-400 mb-3">Missions Disponibles</h3>
                <div id="containerLivreur"></div>
            </section>

            <!-- ARCHIVES -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-slate-900 rounded-[2.5rem] overflow-hidden shadow-2xl">
                    <div class="p-6 border-b border-white/5">
                        <h2 class="text-white font-black text-xs uppercase">Historique des Livraisons</h2>
                    </div>
                    <table class="w-full text-left">
                        <tbody id="archiveBody"></tbody>
                    </table>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL DETAIL -->
    <div id="detailModal" class="fixed inset-0 bg-black/80 z-[400] hidden flex items-center justify-center p-4">
        <div class="bg-white w-full max-w-lg rounded-3xl overflow-hidden">
            <div id="detailContent"></div>
            <div class="p-4 flex gap-2 no-print">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-4 rounded-xl text-xs">IMPRIMER</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-6 bg-slate-200 text-slate-600 font-bold py-4 rounded-xl text-xs">FERMER</button>
            </div>
        </div>
    </div>

    <!-- MODAL CAMERA -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[500] hidden flex flex-col items-center justify-center p-6 text-white">
        <h2 class="text-xl font-black mb-8">Preuve de Livraison</h2>
        <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
        <button onclick="document.getElementById('fileInput').click()" class="w-full max-w-xs bg-white text-black font-black py-5 rounded-3xl shadow-xl uppercase text-xs mb-4">Ouvrir l'appareil photo</button>
        <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-400 underline text-xs">Annuler</button>
    </div>
</body>
</html>
