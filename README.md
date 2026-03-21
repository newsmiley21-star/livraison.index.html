<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>CT241 - Logistique & Performance</title>
    
    <!-- PWA & Mobile Optimization -->
    <meta name="theme-color" content="#0f172a">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="apple-mobile-web-app-title" content="CT241 Log">
    <link rel="apple-touch-icon" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">

    <script src="https://cdn.tailwindcss.com"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot, query } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Configuration Firebase
        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'ct241-logistics-master';

        // État Global
        let currentUser = null;
        let activeRole = 'admin'; // Par défaut pour la démo
        let missions = [];
        let currentMissionId = null;

        // --- AUTHENTIFICATION ---
        const startAuth = async () => {
            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (err) {
                console.error("Auth Error:", err);
            }
        };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                document.getElementById('cloudStatus').innerText = "Serveur Cloud: Actif";
                initRealtimeData();
                generateNextId();
            }
        });

        startAuth();

        // --- SYNCHRONISATION TEMPS RÉEL ---
        function initRealtimeData() {
            if (!currentUser) return;
            const missionsRef = collection(db, 'artifacts', appId, 'public', 'data', 'missions');
            
            onSnapshot(missionsRef, (snapshot) => {
                missions = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
                renderAll();
            }, (error) => {
                console.error("Sync Error:", error);
            });
        }

        // --- ACTIONS MÉTIER ---
        window.switchRole = (role) => {
            activeRole = role;
            document.querySelectorAll('.role-tab').forEach(btn => {
                const isMatch = btn.getAttribute('onclick').includes(role);
                btn.className = isMatch 
                    ? "role-tab px-4 py-2 bg-slate-900 text-white rounded-full text-[10px] font-black shadow-lg transition-all"
                    : "role-tab px-4 py-2 bg-white text-slate-400 border border-slate-200 rounded-full text-[10px] font-black transition-all";
            });
            renderAll();
        };

        const generateNextId = () => {
            const id = "CT-" + Math.floor(100000 + Math.random() * 900000);
            window.nextMissionId = id;
            document.getElementById('idPreview').innerText = id;
        };

        window.submitMission = async () => {
            const btn = document.getElementById('submitBtn');
            const data = {
                expNom: document.getElementById('expNom').value,
                expQuartier: document.getElementById('expQuartier').value,
                expTel: document.getElementById('expTel').value,
                destNom: document.getElementById('destNom').value,
                destQuartier: document.getElementById('destQuartier').value,
                destTel: document.getElementById('destTel').value,
                prix: parseFloat(document.getElementById('prix').value) || 0,
                nature: parseInt(document.getElementById('nature').value),
                paiement: document.getElementById('paiement').value,
                status: 0, // 0: Attente, 1: En cours, 2: Livré
                timestamp: Date.now(),
                owner: currentUser.uid
            };

            if (!data.destNom || !data.prix) {
                notify("Champs obligatoires manquants", "error");
                return;
            }

            btn.disabled = true;
            btn.innerHTML = `<span class="animate-pulse">SYNCHRONISATION...</span>`;

            try {
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextMissionId), data);
                notify("Mission enregistrée au Cloud");
                ['expNom', 'expQuartier', 'expTel', 'destNom', 'destQuartier', 'destTel', 'prix'].forEach(id => document.getElementById(id).value = "");
                generateNextId();
            } catch (e) {
                notify("Erreur de sauvegarde", "error");
            } finally {
                btn.disabled = false;
                btn.innerText = "ENREGISTRER AU CLOUD";
            }
        };

        window.dispatchMission = async (id) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { status: 1 });
            notify("Mission assignée au livreur");
        };

        window.openCamera = (id) => {
            currentMissionId = id;
            document.getElementById('cameraModal').classList.remove('hidden');
        };

        window.handleCapture = (file) => {
            if (!file) return;
            const reader = new FileReader();
            reader.onload = async (e) => {
                const base64 = e.target.result;
                await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId), {
                    status: 2,
                    proofPhoto: base64,
                    deliveredAt: new Date().toLocaleString('fr-FR')
                });
                document.getElementById('cameraModal').classList.add('hidden');
                notify("Livraison confirmée avec succès");
            };
            reader.readAsDataURL(file);
        };

        // --- RENDU UI ---
        function renderAll() {
            const sections = {
                form: document.getElementById('sectionForm'),
                dispatch: document.getElementById('sectionDispatch'),
                livreur: document.getElementById('sectionLivreur'),
                archives: document.getElementById('sectionArchives')
            };

            // Visibilité selon le rôle
            sections.form.classList.toggle('hidden', activeRole === 'livreur');
            sections.dispatch.classList.toggle('hidden', activeRole === 'livreur' || activeRole === 'relais');
            sections.livreur.classList.toggle('hidden', activeRole === 'relais' || activeRole === 'dispatch');
            sections.archives.classList.toggle('hidden', activeRole === 'livreur');

            // Nettoyage
            const listDispatch = document.getElementById('listDispatch');
            const listLivreur = document.getElementById('listLivreur');
            const listArchives = document.getElementById('listArchives');
            
            listDispatch.innerHTML = "";
            listLivreur.innerHTML = "";
            listArchives.innerHTML = "";

            let totalCA = 0;
            let countLiv = 0;

            missions.sort((a,b) => b.timestamp - a.timestamp).forEach(m => {
                if (m.status === 0) {
                    listDispatch.innerHTML += createDispatchCard(m);
                } else if (m.status === 1) {
                    listLivreur.innerHTML += createLivreurCard(m);
                } else if (m.status === 2) {
                    listArchives.innerHTML += createArchiveRow(m);
                    totalCA += m.prix;
                    countLiv++;
                }
            });

            document.getElementById('statCA').innerText = totalCA.toLocaleString() + " CFA";
            document.getElementById('statCount').innerText = countLiv;
        }

        function createDispatchCard(m) {
            return `
                <div class="bg-white p-5 rounded-[2.5rem] border border-slate-100 shadow-sm flex justify-between items-center mb-4 animate-in fade-in slide-in-from-bottom-2">
                    <div class="space-y-1">
                        <span class="text-[9px] font-black text-indigo-600 uppercase tracking-tighter">${m.id}</span>
                        <p class="text-xs font-black text-slate-900">${m.destNom}</p>
                        <p class="text-[9px] font-bold text-slate-400 uppercase">📍 ${m.destQuartier}</p>
                    </div>
                    <div class="flex gap-2">
                        <button onclick="viewReceipt('${m.id}')" class="bg-slate-100 text-slate-500 px-4 py-3 rounded-2xl text-[10px] font-black">VOIR</button>
                        <button onclick="dispatchMission('${m.id}')" class="bg-indigo-600 text-white px-5 py-3 rounded-2xl text-[10px] font-black shadow-lg shadow-indigo-100">DISPATCH</button>
                    </div>
                </div>`;
        }

        function createLivreurCard(m) {
            return `
                <div class="bg-amber-50 p-6 rounded-[3rem] border border-amber-100 space-y-4 mb-5 animate-in slide-in-from-right">
                    <div class="flex justify-between items-center">
                        <span class="text-[10px] font-black text-amber-700 italic">${m.id}</span>
                        <span class="bg-amber-500 text-white text-[8px] font-black px-3 py-1 rounded-full uppercase">En Transit</span>
                    </div>
                    <div class="space-y-1">
                        <h3 class="text-sm font-black text-amber-900">${m.destNom}</h3>
                        <p class="text-xs font-bold text-amber-800/60">${m.destQuartier} • ${m.destTel}</p>
                    </div>
                    <div class="flex gap-2 pt-2">
                        <button onclick="viewReceipt('${m.id}')" class="flex-1 bg-white border border-amber-200 text-amber-700 font-black py-4 rounded-2xl text-[10px]">DÉTAILS</button>
                        <button onclick="openCamera('${m.id}')" class="flex-[2] bg-amber-600 text-white font-black py-4 rounded-2xl shadow-xl shadow-amber-200 text-[10px] tracking-widest">LIVRER MAINTENANT</button>
                    </div>
                </div>`;
        }

        function createArchiveRow(m) {
            return `
                <tr class="border-b border-slate-800 text-[10px] hover:bg-slate-800 transition-colors cursor-pointer" onclick="viewReceipt('${m.id}')">
                    <td class="p-5 font-black text-white italic">${m.id}</td>
                    <td class="p-5 text-slate-400 font-bold">${m.destNom}</td>
                    <td class="p-5 text-right font-black text-emerald-400">${m.prix.toLocaleString()} CFA</td>
                </tr>`;
        }

        window.viewReceipt = (id) => {
            const m = missions.find(x => x.id === id);
            const natures = ["📦 Colis Standard", "🍟 Repas Chaud", "📁 Documents", "💊 Pharmacie"];
            document.getElementById('modalContent').innerHTML = `
                <div class="p-10 text-slate-900 bg-white">
                    <div class="flex justify-between items-start border-b-4 border-slate-900 pb-8 mb-10">
                        <div>
                            <h1 class="text-4xl font-black italic tracking-tighter">CT241</h1>
                            <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Performance Logistique</p>
                        </div>
                        <div class="text-right">
                            <span class="text-[10px] font-black bg-slate-900 text-white px-4 py-1.5 rounded-full">BORDEREAU</span>
                            <p class="text-2xl font-black mt-2 text-slate-400">${m.id}</p>
                        </div>
                    </div>
                    
                    <div class="grid grid-cols-2 gap-8 mb-10">
                        <div class="bg-slate-50 p-6 rounded-[2.5rem]">
                            <p class="text-[9px] font-black text-slate-400 uppercase mb-3 border-b pb-2">Expéditeur</p>
                            <p class="text-lg font-black">${m.expNom || 'Anonyme'}</p>
                            <p class="text-xs font-bold text-slate-500">${m.expQuartier || 'N/A'}</p>
                            <p class="text-sm font-black text-indigo-600 mt-2">${m.expTel || ''}</p>
                        </div>
                        <div class="bg-slate-900 text-white p-6 rounded-[2.5rem] shadow-2xl">
                            <p class="text-[9px] font-black text-white/30 uppercase mb-3 border-b border-white/10 pb-2">Destinataire</p>
                            <p class="text-lg font-black">${m.destNom}</p>
                            <p class="text-xs font-bold text-white/60">${m.destQuartier}</p>
                            <p class="text-sm font-black text-indigo-400 mt-2">${m.destTel}</p>
                        </div>
                    </div>

                    <div class="bg-slate-50 p-8 rounded-[3rem] flex justify-between items-center mb-8 border border-slate-100">
                        <div>
                            <p class="text-[9px] font-black text-slate-400 uppercase">Nature</p>
                            <p class="text-sm font-black">${natures[m.nature]}</p>
                        </div>
                        <div class="text-right">
                            <p class="text-[9px] font-black text-slate-400 uppercase">Frais Livraison</p>
                            <p class="text-3xl font-black">${m.prix.toLocaleString()} <span class="text-xs">CFA</span></p>
                        </div>
                    </div>

                    ${m.proofPhoto ? `
                    <div class="mt-8">
                        <p class="text-[9px] font-black text-center text-slate-400 uppercase mb-4 tracking-widest">Preuve de réception</p>
                        <div class="relative">
                            <img src="${m.proofPhoto}" class="w-full rounded-[3rem] border-8 border-white shadow-xl">
                            <div class="absolute bottom-6 right-6 bg-white/90 backdrop-blur px-5 py-2 rounded-full text-[9px] font-black italic">✓ Livré le ${m.deliveredAt}</div>
                        </div>
                    </div>` : ''}
                </div>`;
            document.getElementById('receiptModal').classList.remove('hidden');
        };

        const notify = (msg, type = "success") => {
            const toast = document.getElementById('toast');
            toast.innerText = msg;
            toast.className = `fixed bottom-24 left-1/2 -translate-x-1/2 px-8 py-4 rounded-full text-white font-black text-[10px] uppercase tracking-widest shadow-2xl z-[999] animate-in slide-in-from-bottom duration-300 ${type === 'success' ? 'bg-emerald-600' : 'bg-red-600'}`;
            toast.classList.remove('hidden');
            setTimeout(() => toast.classList.add('hidden'), 3000);
        };
    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; color: #0f172a; overflow-x: hidden; }
        .hidden { display: none !important; }
        ::-webkit-scrollbar { display: none; }
        @media print { .no-print { display: none; } }
    </style>
</head>
<body class="pb-40">

    <div id="toast" class="hidden"></div>

    <!-- Header & Navigation -->
    <header class="sticky top-0 z-50 bg-white/80 backdrop-blur-2xl border-b p-5 flex justify-between items-center no-print">
        <div>
            <h1 class="text-2xl font-black italic tracking-tighter">CT241 <span class="text-indigo-600">LOG</span></h1>
            <p id="cloudStatus" class="text-[8px] font-black uppercase text-slate-400 tracking-widest">Serveur: Connexion...</p>
        </div>
        <div class="flex gap-2 p-1.5 bg-slate-100 rounded-full">
            <button onclick="switchRole('admin')" class="role-tab px-4 py-2 bg-slate-900 text-white rounded-full text-[10px] font-black shadow-lg">ADMIN</button>
            <button onclick="switchRole('relais')" class="role-tab px-4 py-2 bg-white text-slate-400 border border-slate-200 rounded-full text-[10px] font-black">RELAIS</button>
            <button onclick="switchRole('livreur')" class="role-tab px-4 py-2 bg-white text-slate-400 border border-slate-200 rounded-full text-[10px] font-black">LIVREUR</button>
        </div>
    </header>

    <main class="max-w-xl mx-auto p-6 space-y-8 no-print">
        
        <!-- Stats Dashboard -->
        <section class="grid grid-cols-2 gap-4">
            <div class="bg-slate-900 p-8 rounded-[3.5rem] text-white shadow-2xl relative overflow-hidden group">
                <p class="text-[10px] font-black text-slate-500 uppercase tracking-[0.2em] mb-2">Livrées</p>
                <h2 id="statCount" class="text-6xl font-black tracking-tighter">0</h2>
                <div class="absolute -right-6 -bottom-6 w-32 h-32 bg-white/5 rounded-full group-hover:scale-150 transition-transform duration-1000"></div>
            </div>
            <div class="bg-indigo-600 p-8 rounded-[3.5rem] text-white shadow-2xl shadow-indigo-200 relative overflow-hidden group">
                <p class="text-[10px] font-black text-indigo-200 uppercase tracking-[0.2em] mb-2">C.A Global</p>
                <h2 id="statCA" class="text-xl font-black tracking-tight leading-none mt-4">0 CFA</h2>
                <div class="absolute -right-6 -bottom-6 w-32 h-32 bg-black/10 rounded-full"></div>
            </div>
        </section>

        <!-- Création Formulaire -->
        <section id="sectionForm" class="bg-white p-8 rounded-[3.5rem] shadow-xl border border-slate-100 space-y-6">
            <div class="flex justify-between items-center px-2">
                <h2 class="text-[10px] font-black uppercase text-emerald-600 tracking-widest">Nouvelle Mission</h2>
                <span id="idPreview" class="text-[10px] font-black text-emerald-700 bg-emerald-50 px-4 py-1.5 rounded-full italic">...</span>
            </div>

            <div class="space-y-4">
                <div class="space-y-3">
                    <p class="text-[8px] font-black text-slate-400 uppercase ml-4">Origine (Boutique)</p>
                    <input type="text" id="expNom" placeholder="Nom Boutique" class="w-full p-5 bg-slate-50 rounded-[2rem] font-bold text-xs outline-none focus:ring-4 ring-indigo-500/10 transition-all">
                    <div class="grid grid-cols-2 gap-4">
                        <input type="text" id="expQuartier" placeholder="Quartier" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs outline-none">
                        <input type="tel" id="expTel" placeholder="Téléphone" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs outline-none">
                    </div>
                </div>

                <div class="h-px bg-slate-100 my-4"></div>

                <div class="space-y-3">
                    <p class="text-[8px] font-black text-slate-400 uppercase ml-4">Destination (Client)</p>
                    <input type="text" id="destNom" placeholder="Nom du Client" class="w-full p-5 bg-slate-50 rounded-[2rem] font-bold text-xs outline-none">
                    <div class="grid grid-cols-2 gap-4">
                        <input type="text" id="destQuartier" placeholder="Quartier" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs outline-none">
                        <input type="tel" id="destTel" placeholder="Téléphone" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs outline-none">
                    </div>
                </div>

                <div class="grid grid-cols-2 gap-4">
                    <select id="nature" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs outline-none appearance-none">
                        <option value="0">📦 Standard</option><option value="1">🍟 Repas</option><option value="2">📁 Documents</option><option value="3">💊 Pharma</option>
                    </select>
                    <select id="paiement" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs outline-none appearance-none">
                        <option>Espèces</option><option>Airtel Money</option><option>Moov Money</option>
                    </select>
                </div>

                <div class="relative">
                    <input type="number" id="prix" placeholder="Montant Livraison" class="w-full p-7 bg-indigo-600 text-white rounded-[2.5rem] font-black text-xl placeholder-indigo-300 outline-none shadow-xl">
                    <span class="absolute right-8 top-1/2 -translate-y-1/2 text-[10px] font-black text-indigo-200">CFA</span>
                </div>

                <button id="submitBtn" onclick="submitMission()" class="w-full bg-slate-900 text-white font-black py-6 rounded-[2.5rem] shadow-2xl hover:bg-black transition-all text-[11px] tracking-[0.3em]">
                    ENREGISTRER AU CLOUD
                </button>
            </div>
        </section>

        <!-- Listes de Dispatch & Livreur -->
        <section id="sectionDispatch" class="space-y-4">
            <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest ml-4">En attente de dispatch</h3>
            <div id="listDispatch"></div>
        </section>

        <section id="sectionLivreur" class="space-y-4">
            <h3 class="text-[10px] font-black text-amber-600 uppercase tracking-widest ml-4">Missions en cours</h3>
            <div id="listLivreur"></div>
        </section>

        <!-- Archive Section -->
        <section id="sectionArchives" class="bg-slate-900 rounded-[3.5rem] overflow-hidden shadow-2xl border border-slate-800">
            <div class="p-8 border-b border-slate-800 flex justify-between items-center">
                <h3 class="text-white font-black text-[10px] uppercase tracking-widest">Historique Récent</h3>
                <div class="w-2 h-2 bg-emerald-500 rounded-full animate-pulse shadow-glow shadow-emerald-500"></div>
            </div>
            <div class="overflow-x-auto">
                <table class="w-full">
                    <tbody id="listArchives"></tbody>
                </table>
            </div>
        </section>

    </main>

    <!-- Modal Receipt -->
    <div id="receiptModal" class="fixed inset-0 bg-slate-950/95 z-[900] hidden flex items-center justify-center p-4 backdrop-blur-xl animate-in fade-in duration-300">
        <div class="w-full max-w-2xl bg-white rounded-[4rem] overflow-hidden shadow-2xl relative">
            <div id="modalContent" class="max-h-[80vh] overflow-y-auto"></div>
            <div class="p-8 bg-slate-50 flex gap-4 no-print border-t">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-5 rounded-[2rem] text-xs tracking-widest shadow-xl">IMPRIMER LE BON</button>
                <button onclick="document.getElementById('receiptModal').classList.add('hidden')" class="px-10 bg-white border-2 border-slate-200 text-slate-500 font-black py-5 rounded-[2rem] text-xs uppercase">Fermer</button>
            </div>
        </div>
    </div>

    <!-- Modal Photo Capture -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[1000] hidden flex flex-col items-center justify-center p-12 text-white">
        <div class="text-center space-y-10 max-w-xs animate-in zoom-in duration-300">
            <div class="w-36 h-36 bg-white/5 rounded-full flex items-center justify-center text-6xl mx-auto border border-white/10 backdrop-blur-3xl shadow-2xl">📸</div>
            <div class="space-y-3">
                <h2 class="text-3xl font-black italic tracking-tighter">Confirmation</h2>
                <p class="text-slate-500 text-xs font-bold leading-relaxed">Capturez une preuve photo claire de la réception par le client.</p>
            </div>
            <input type="file" id="cameraInput" accept="image/*" capture="camera" class="hidden" onchange="handleCapture(this.files[0])">
            <button onclick="document.getElementById('cameraInput').click()" class="w-full bg-white text-black font-black py-7 rounded-[3rem] shadow-2xl uppercase tracking-[0.2em] text-[11px]">OUVRIR L'APPAREIL</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 font-black uppercase text-[10px] tracking-widest">Annuler</button>
        </div>
    </div>

</body>
</html>
