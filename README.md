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
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, onSnapshot } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Configuration Firebase
        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'ct241-log-v3';

        // État de l'application
        let user = null;
        let userRole = 'admin'; 
        let allMissions = [];
        let currentMissionId = null;

        // --- AUTHENTIFICATION ---
        const initAuth = async () => {
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                await signInWithCustomToken(auth, __initial_auth_token);
            } else {
                await signInAnonymously(auth);
            }
        };

        onAuthStateChanged(auth, (u) => {
            if (u) {
                user = u;
                document.getElementById('userStatus').innerText = "Cloud Connecté";
                setupRealtimeSync();
                prepareNextId();
            }
        });

        initAuth();

        // --- SYNCHRONISATION CLOUD ---
        function setupRealtimeSync() {
            if (!user) return;
            const colRef = collection(db, 'artifacts', appId, 'public', 'data', 'missions');
            
            onSnapshot(colRef, (snapshot) => {
                allMissions = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                renderUI();
                renderStats();
            }, (error) => {
                console.error("Erreur de synchronisation:", error);
            });
        }

        // --- LOGIQUE MÉTIER ---
        window.changeRole = (role) => {
            userRole = role;
            document.querySelectorAll('.role-btn').forEach(btn => {
                const isSelected = btn.getAttribute('onclick').includes(role);
                btn.className = isSelected 
                    ? "text-[9px] font-black bg-slate-900 text-white px-3 py-1.5 rounded-full role-btn transition-all shadow-lg"
                    : "text-[9px] font-black bg-white border border-slate-200 text-slate-400 px-3 py-1.5 rounded-full role-btn hover:bg-slate-50";
            });
            renderUI();
        };

        const prepareNextId = () => {
            window.nextId = "CT-" + Math.floor(Math.random() * 900000 + 100000);
            document.getElementById('displayNextId').innerText = window.nextId;
        };

        window.creerMission = async () => {
            const btn = document.getElementById('btnSubmit');
            const fields = ['expNom', 'expQuartier', 'expTel', 'destNom', 'destQuartier', 'destTel', 'prix'];
            
            const data = {};
            fields.forEach(id => data[id] = document.getElementById(id).value);
            
            if (!data.destNom || !data.prix) {
                showToast("Veuillez remplir les informations clés", "error");
                return;
            }

            btn.disabled = true;
            btn.innerText = "SAUVEGARDE CLOUD...";

            const missionData = {
                en: data.expNom, eq: data.expQuartier, et: data.expTel,
                dn: data.destNom, dq: data.destQuartier, dt: data.destTel,
                n: parseInt(document.getElementById('nature').value),
                p: parseFloat(data.prix) || 0,
                r: parseInt(document.getElementById('reglement').value),
                status: 0,
                createdAt: Date.now(),
                creator: user.uid
            };

            try {
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId), missionData);
                showToast("Mission enregistrée avec succès", "success");
                fields.forEach(id => document.getElementById(id).value = "");
                prepareNextId();
            } catch (e) {
                showToast("Erreur de connexion Cloud", "error");
            } finally {
                btn.disabled = false;
                btn.innerText = "ENREGISTRER AU CLOUD";
            }
        };

        window.validerDispatch = async (id) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { status: 1 });
            showToast("Mission assignée au livreur");
        };

        window.openCamera = (id) => {
            currentMissionId = id;
            document.getElementById('cameraModal').classList.remove('hidden');
        };

        window.processImage = (file) => {
            if (!file) return;
            const reader = new FileReader();
            reader.onload = async (e) => {
                const photo = e.target.result;
                await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId), {
                    status: 2,
                    photo: photo,
                    deliveredAt: new Date().toLocaleString(),
                    deliveredBy: user.uid
                });
                document.getElementById('cameraModal').classList.add('hidden');
                showToast("Livraison confirmée et archivée", "success");
            };
            reader.readAsDataURL(file);
        };

        // --- RENDU UI ---
        function renderUI() {
            const containers = {
                creer: document.getElementById('creationSection'),
                dispatch: document.getElementById('containerDispatch'),
                livreur: document.getElementById('containerLivreur'),
                archives: document.getElementById('archiveSection'),
                archiveBody: document.getElementById('archiveBody')
            };

            // Visibilité par rôle
            containers.creer.classList.toggle('hidden', userRole === 'livreur' || userRole === 'dispatch');
            containers.dispatch.parentElement.classList.toggle('hidden', userRole === 'relais' || userRole === 'livreur');
            containers.livreur.parentElement.classList.toggle('hidden', userRole === 'relais' || userRole === 'dispatch');
            containers.archives.classList.toggle('hidden', userRole === 'livreur');

            containers.dispatch.innerHTML = "";
            containers.livreur.innerHTML = "";
            containers.archiveBody.innerHTML = "";

            allMissions.sort((a,b) => b.createdAt - a.createdAt).forEach(m => {
                if (m.status === 0) {
                    containers.dispatch.innerHTML += `
                        <div class="p-5 bg-white border border-slate-100 rounded-[2rem] mb-4 shadow-sm flex justify-between items-center animate-in fade-in duration-500">
                            <div class="space-y-1">
                                <span class="text-[9px] font-black text-blue-600 uppercase tracking-tighter">${m.id}</span>
                                <p class="text-[11px] font-black">${m.dn}</p>
                                <p class="text-[9px] text-slate-400 font-bold uppercase">${m.dq}</p>
                            </div>
                            <div class="flex gap-2">
                                <button onclick="voirBon('${m.id}')" class="bg-slate-100 text-slate-500 text-[10px] px-4 py-2.5 rounded-2xl font-black">BON</button>
                                <button onclick="validerDispatch('${m.id}')" class="bg-blue-600 text-white text-[10px] px-5 py-2.5 rounded-2xl font-black shadow-lg shadow-blue-200">DISPATCH</button>
                            </div>
                        </div>`;
                } else if (m.status === 1) {
                    containers.livreur.innerHTML += `
                        <div class="p-6 bg-amber-50 border border-amber-100 rounded-[2.5rem] mb-5 space-y-4 animate-in slide-in-from-right duration-300">
                            <div class="flex justify-between items-center">
                                <span class="text-[10px] font-black text-amber-600 italic">${m.id}</span>
                                <span class="bg-amber-500 text-white text-[8px] font-black px-3 py-1 rounded-full uppercase">En cours</span>
                            </div>
                            <div class="space-y-1">
                                <h3 class="text-sm font-black text-amber-900">${m.dn}</h3>
                                <p class="text-xs font-bold text-amber-700/70">📍 ${m.dq} • 📞 ${m.dt}</p>
                            </div>
                            <div class="flex gap-2">
                                <button onclick="voirBon('${m.id}')" class="flex-1 bg-white border border-amber-200 text-amber-700 font-black py-4 rounded-2xl text-[10px] uppercase">Détails</button>
                                <button onclick="openCamera('${m.id}')" class="flex-[2] bg-amber-500 text-white font-black py-4 rounded-2xl shadow-xl shadow-amber-200 uppercase text-[10px] tracking-widest">Valider Livraison</button>
                            </div>
                        </div>`;
                } else if (m.status === 2) {
                    containers.archiveBody.innerHTML += `
                        <tr class="border-b border-slate-800 text-[10px] hover:bg-slate-800 transition-colors cursor-pointer" onclick="voirBon('${m.id}')">
                            <td class="p-5 font-black text-white italic">${m.id}</td>
                            <td class="p-5 text-slate-400 font-bold">${m.dn}</td>
                            <td class="p-5 text-center"><span class="bg-emerald-500/10 text-emerald-500 px-3 py-1 rounded-full text-[8px] font-black">LIVRÉ</span></td>
                            <td class="p-5 text-right text-white font-black">${m.p.toLocaleString()} CFA</td>
                        </tr>`;
                }
            });
        }

        function renderStats() {
            let totalCA = 0;
            let countLiv = 0;
            allMissions.forEach(m => {
                if (m.status === 2) {
                    totalCA += m.p;
                    countLiv++;
                }
            });
            document.getElementById('caTotal').innerText = totalCA.toLocaleString() + " CFA";
            document.getElementById('countTotal').innerText = countLiv;
        }

        window.voirBon = (id) => {
            const m = allMissions.find(x => x.id === id);
            const natureMap = ["📦 Colis Standard", "🍟 Repas Chaud", "📁 Documents", "💊 Pharmacie"];
            document.getElementById('detailContent').innerHTML = `
                <div class="p-10 text-slate-900 bg-white min-h-[500px]">
                    <div class="flex justify-between items-start border-b-4 border-slate-900 pb-8 mb-10">
                        <div>
                            <h1 class="text-4xl font-black italic tracking-tighter">CT241</h1>
                            <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Logistique & Performance</p>
                        </div>
                        <div class="text-right">
                            <h2 class="text-xs font-black uppercase tracking-widest bg-slate-900 text-white px-4 py-2 rounded-full mb-2 inline-block">Bordereau</h2>
                            <p class="text-xl font-black text-slate-500">${m.id}</p>
                        </div>
                    </div>
                    <div class="grid grid-cols-2 gap-10 mb-10">
                        <div class="p-6 bg-slate-50 rounded-[2rem] space-y-3">
                            <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest border-b pb-2">Expéditeur</p>
                            <p class="text-lg font-black text-slate-900">${m.en || '---'}</p>
                            <p class="text-xs font-bold text-slate-500">${m.eq || 'N/A'}</p>
                            <p class="text-sm font-black text-blue-600">${m.et || ''}</p>
                        </div>
                        <div class="p-6 bg-slate-900 text-white rounded-[2.5rem] space-y-3 shadow-2xl">
                            <p class="text-[10px] font-black text-white/30 uppercase tracking-widest border-b border-white/10 pb-2">Destinataire</p>
                            <p class="text-lg font-black text-white">${m.dn}</p>
                            <p class="text-xs font-bold text-white/70">${m.dq}</p>
                            <p class="text-sm font-black text-blue-400">${m.dt}</p>
                        </div>
                    </div>
                    <div class="p-8 bg-slate-50 rounded-[2.5rem] flex justify-between items-center mb-8 border border-slate-200">
                        <div class="space-y-1">
                            <p class="text-[10px] font-black text-slate-400 uppercase">Type de contenu</p>
                            <p class="text-sm font-black text-slate-900">${natureMap[m.n]}</p>
                        </div>
                        <div class="text-right">
                            <p class="text-[10px] font-black text-slate-400 uppercase">Frais de livraison</p>
                            <p class="text-3xl font-black text-slate-900">${m.p.toLocaleString()} <small class="text-xs">CFA</small></p>
                        </div>
                    </div>
                    ${m.photo ? `
                    <div class="mt-10 animate-in zoom-in duration-500">
                        <p class="text-[10px] font-black text-slate-400 uppercase mb-4 text-center tracking-[0.2em]">Preuve de réception</p>
                        <div class="relative group">
                            <img src="${m.photo}" class="w-full rounded-[3rem] border-8 border-white shadow-2xl">
                            <div class="absolute bottom-4 right-8 bg-white/80 backdrop-blur px-4 py-2 rounded-full text-[9px] font-black italic">✓ Livré le ${m.deliveredAt}</div>
                        </div>
                    </div>` : ''}
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        const showToast = (msg, type = "success") => {
            const t = document.getElementById('toast');
            t.innerText = msg;
            t.className = `fixed bottom-24 left-1/2 -translate-x-1/2 px-8 py-4 rounded-full text-white font-black text-[10px] uppercase tracking-widest shadow-2xl z-[1000] animate-in slide-in-from-bottom duration-300 ${type==='success'?'bg-emerald-600':'bg-red-600'}`;
            t.classList.remove('hidden');
            setTimeout(() => t.classList.add('hidden'), 3000);
        };
    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; letter-spacing: -0.02em; }
        .hidden { display: none; }
        @media print { .no-print { display: none; } }
    </style>
</head>
<body class="pb-32">

    <div id="toast" class="hidden"></div>

    <!-- Navigation -->
    <nav class="bg-white/80 backdrop-blur-xl p-5 sticky top-0 z-50 border-b flex justify-between items-center no-print">
        <div>
            <h1 class="text-xl font-black italic tracking-tighter">CT241 <span class="text-blue-600">LOG</span></h1>
            <span id="userStatus" class="text-[8px] font-black uppercase text-slate-400 tracking-widest">Connexion...</span>
        </div>
        <div class="flex gap-1.5 p-1 bg-slate-100 rounded-full">
            <button onclick="changeRole('admin')" class="text-[9px] font-black bg-slate-900 text-white px-3 py-1.5 rounded-full role-btn transition-all shadow-lg">ADMIN</button>
            <button onclick="changeRole('relais')" class="text-[9px] font-black bg-white border border-slate-200 text-slate-400 px-3 py-1.5 rounded-full role-btn">RELAIS</button>
            <button onclick="changeRole('livreur')" class="text-[9px] font-black bg-white border border-slate-200 text-slate-400 px-3 py-1.5 rounded-full role-btn">LIVREUR</button>
        </div>
    </nav>

    <main class="max-w-xl mx-auto p-5 space-y-8 no-print">
        
        <!-- Stats -->
        <section class="grid grid-cols-2 gap-4">
            <div class="bg-slate-900 p-8 rounded-[3rem] text-white shadow-2xl space-y-2 group overflow-hidden relative">
                <p class="text-[9px] font-black text-slate-500 uppercase tracking-widest">Livraisons</p>
                <h2 id="countTotal" class="text-5xl font-black tracking-tighter">0</h2>
                <div class="absolute -right-4 -bottom-4 w-24 h-24 bg-white/5 rounded-full group-hover:scale-150 transition-transform duration-700"></div>
            </div>
            <div class="bg-blue-600 p-8 rounded-[3rem] text-white shadow-2xl shadow-blue-200 space-y-2 relative overflow-hidden">
                <p class="text-[9px] font-black text-blue-200 uppercase tracking-widest">C.A Global</p>
                <h2 id="caTotal" class="text-xl font-black tracking-tighter leading-none">0 CFA</h2>
                <div class="absolute -right-4 -bottom-4 w-24 h-24 bg-black/10 rounded-full"></div>
            </div>
        </section>

        <!-- Création -->
        <section id="creationSection" class="bg-white rounded-[3rem] shadow-xl p-8 space-y-6 border border-slate-100">
            <div class="flex justify-between items-center">
                <h2 class="font-black text-[10px] uppercase text-emerald-600 tracking-[0.2em]">Nouveau Bordereau</h2>
                <span id="displayNextId" class="text-[10px] font-black bg-emerald-50 text-emerald-600 px-4 py-1.5 rounded-full italic tracking-widest">...</span>
            </div>
            
            <div class="space-y-4">
                <div class="group">
                    <p class="text-[8px] font-black text-slate-400 uppercase mb-2 ml-4">Informations Expéditeur</p>
                    <input type="text" id="expNom" placeholder="Nom de la Boutique" class="w-full p-5 bg-slate-50 rounded-[2rem] font-bold text-xs border-none focus:ring-4 ring-emerald-500/20 transition-all outline-none">
                </div>
                <div class="grid grid-cols-2 gap-4">
                    <input type="text" id="expQuartier" placeholder="Quartier Origine" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs border-none outline-none">
                    <input type="tel" id="expTel" placeholder="Tél. Boutique" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs border-none outline-none">
                </div>
                
                <div class="h-px bg-slate-100 my-4"></div>
                
                <div class="group">
                    <p class="text-[8px] font-black text-slate-400 uppercase mb-2 ml-4">Informations Destinataire</p>
                    <input type="text" id="destNom" placeholder="Nom Complet du Client" class="w-full p-5 bg-slate-50 rounded-[2rem] font-bold text-xs border-none focus:ring-4 ring-emerald-500/20 transition-all outline-none">
                </div>
                <div class="grid grid-cols-2 gap-4">
                    <input type="text" id="destQuartier" placeholder="Destination Exacte" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs border-none outline-none">
                    <input type="tel" id="destTel" placeholder="Contact Client" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs border-none outline-none">
                </div>

                <div class="grid grid-cols-2 gap-4">
                    <select id="nature" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs border-none outline-none appearance-none">
                        <option value="0">📦 Standard</option><option value="1">🍟 Repas</option><option value="2">📁 Documents</option><option value="3">💊 Pharma</option>
                    </select>
                    <select id="reglement" class="p-5 bg-slate-50 rounded-[2rem] font-bold text-xs border-none outline-none appearance-none">
                        <option value="0">Espèces</option><option value="1">Airtel Money</option><option value="2">Moov Money</option>
                    </select>
                </div>

                <div class="relative">
                    <input type="number" id="prix" placeholder="MONTANT LIVRAISON" class="w-full p-6 bg-emerald-600 text-white rounded-[2rem] font-black text-lg border-none placeholder-emerald-300 outline-none">
                    <span class="absolute right-6 top-1/2 -translate-y-1/2 text-[10px] font-black text-white/50">CFA</span>
                </div>
            </div>
            
            <button id="btnSubmit" onclick="creerMission()" class="w-full bg-slate-900 text-white font-black py-6 rounded-[2.5rem] shadow-2xl hover:bg-black transition-all uppercase text-[11px] tracking-[0.3em]">
                Enregistrer au Cloud
            </button>
        </section>

        <!-- Listes -->
        <section class="space-y-4">
            <h2 class="font-black text-[10px] uppercase text-slate-400 tracking-widest ml-4">Missions en attente</h2>
            <div id="containerDispatch"></div>
        </section>

        <section class="space-y-4">
            <h2 class="font-black text-[10px] uppercase text-amber-600 tracking-widest ml-4">Vos Livraisons Actives</h2>
            <div id="containerLivreur"></div>
        </section>

        <!-- Archives -->
        <section id="archiveSection" class="bg-slate-900 rounded-[3rem] overflow-hidden shadow-2xl">
            <div class="p-8 border-b border-slate-800 flex justify-between items-center">
                <h2 class="text-white font-black text-[10px] uppercase tracking-[0.2em]">Historique Cloud</h2>
                <div class="w-2 h-2 bg-emerald-500 rounded-full animate-pulse"></div>
            </div>
            <div class="overflow-x-auto">
                <table class="w-full">
                    <tbody id="archiveBody"></tbody>
                </table>
            </div>
        </section>

    </main>

    <!-- Modal Détail -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/95 z-[400] hidden flex items-center justify-center p-4 backdrop-blur-xl animate-in fade-in duration-300">
        <div class="w-full max-w-2xl bg-white rounded-[3.5rem] overflow-hidden shadow-2xl relative">
            <div id="detailContent" class="max-h-[80vh] overflow-y-auto"></div>
            <div class="p-8 bg-slate-50 flex gap-4 no-print">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-5 rounded-[2rem] text-xs uppercase tracking-widest shadow-xl">Imprimer le Bon</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-10 bg-white border-2 border-slate-200 text-slate-500 font-black py-5 rounded-[2rem] text-xs uppercase tracking-widest">Fermer</button>
            </div>
        </div>
    </div>

    <!-- Modal Photo -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[500] hidden flex flex-col items-center justify-center p-10 text-white">
        <div class="text-center space-y-8 max-w-xs animate-in zoom-in duration-300">
            <div class="w-32 h-32 bg-white/10 rounded-full flex items-center justify-center text-6xl mx-auto backdrop-blur-3xl border border-white/20">📸</div>
            <div class="space-y-2">
                <h2 class="text-3xl font-black italic tracking-tighter">Confirmation</h2>
                <p class="text-slate-500 text-xs font-bold leading-relaxed">Capturez une preuve claire de la réception du colis par le client.</p>
            </div>
            <input type="file" id="camInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('camInput').click()" class="w-full bg-white text-black font-black py-6 rounded-[2.5rem] shadow-2xl uppercase tracking-widest text-[11px] hover:scale-105 transition-transform">Prendre la Photo</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-500 font-black uppercase text-[10px] tracking-widest pt-4">Annuler</button>
        </div>
    </div>

</body>
</html>
