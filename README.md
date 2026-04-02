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
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">

    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
    <script src="https://unpkg.com/html5-qrcode"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, deleteDoc, onSnapshot, query, orderBy } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

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

        const getMissionDoc = (id) => doc(db, 'artifacts', appId, 'public', 'data', 'missions', id);
        const getMissionsColl = () => collection(db, 'artifacts', appId, 'public', 'data', 'missions');

        let currentUser = null;
        let userRole = null;
        let allMissions = [];
        let html5QrCode = null;

        // --- AUTHENTIFICATION ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value.trim().toLowerCase();
            const pass = document.getElementById('loginPass').value;
            const btn = e.target.querySelector('button');
            btn.disabled = true;

            try { 
                await signInWithEmailAndPassword(auth, email, pass); 
            } catch (error) { 
                document.getElementById('loginError').classList.remove('hidden');
                btn.disabled = false;
            }
        };

        window.handleLogout = async () => { await signOut(auth); location.reload(); };

        onAuthStateChanged(auth, (user) => {
            if (user) {
                currentUser = user;
                const email = user.email.toLowerCase();
                if (email.includes('admin')) userRole = 'admin';
                else if (email.includes('relais')) userRole = 'relais';
                else if (email.includes('dispatch')) userRole = 'dispatch';
                else userRole = 'livreur';

                document.getElementById('authSection').classList.add('hidden');
                document.getElementById('appContent').classList.remove('hidden');
                document.getElementById('userMail').innerText = email;
                
                setupRoleUI();
                startListeners();
                generateNewID();
            }
        });

        const setupRoleUI = () => {
            document.querySelectorAll('.role-view').forEach(el => el.classList.add('hidden'));
            if (userRole === 'admin') {
                document.querySelectorAll('.role-view').forEach(el => el.classList.remove('hidden'));
            } else if (userRole === 'relais') {
                document.getElementById('createSection').classList.remove('hidden');
                document.getElementById('archiveSection').classList.remove('hidden');
            } else if (userRole === 'dispatch') {
                document.getElementById('dispatchSection').classList.remove('hidden');
                document.getElementById('archiveSection').classList.remove('hidden');
            } else {
                document.getElementById('livreurSection').classList.remove('hidden');
                document.getElementById('archiveSection').classList.remove('hidden');
            }
        };

        // --- ACTIONS SUR LES MISSIONS ---
        window.generateNewID = () => {
            window.currentMissionId = "241-" + Math.floor(100000 + Math.random() * 900000);
            document.getElementById('nextIdDisplay').innerText = window.currentMissionId;
        };

        window.saveMission = async () => {
            const data = {
                id: window.currentMissionId,
                en: document.getElementById('expNom').value.trim(),
                eq: document.getElementById('expQuartier').value.trim(),
                et: document.getElementById('expTel').value.trim(),
                dn: document.getElementById('destNom').value.trim(),
                dq: document.getElementById('dq').value.trim(),
                dt: document.getElementById('dt').value.trim(),
                p: parseFloat(document.getElementById('price').value) || 0,
                s: 0, 
                ts: Date.now(),
                date: new Date().toLocaleDateString('fr-FR'),
                author: currentUser.email,
                history: [{status: 0, date: new Date().toLocaleString('fr-FR'), label: "Créé au relais"}]
            };

            if (!data.dn || !data.dt) return showToast("Destinataire requis", "error");

            try {
                await setDoc(getMissionDoc(data.id), data);
                showToast("Bordereau créé !", "success");
                generateNewID();
                document.getElementById('destNom').value = "";
                document.getElementById('dt').value = "";
            } catch (e) { showToast("Erreur lors de la création", "error"); }
        };

        window.updateStatus = async (id, newStatus) => {
            try {
                const m = allMissions.find(x => x.id === id);
                const history = m.history || [];
                const labels = ["Mis en attente", "En cours de livraison", "Livré avec succès"];
                
                history.push({
                    status: newStatus,
                    date: new Date().toLocaleString('fr-FR'),
                    label: labels[newStatus]
                });

                const updateData = { s: newStatus, history: history };
                if (newStatus === 2) {
                    updateData.deliveryDate = new Date().toLocaleDateString('fr-FR');
                    updateData.deliveredBy = currentUser.email;
                }
                
                await updateDoc(getMissionDoc(id), updateData);
                showToast("Statut mis à jour", "success");
                if(newStatus === 2) document.getElementById('detailModal').classList.add('hidden');
            } catch (e) { showToast("Erreur de mise à jour", "error"); }
        };

        window.deleteMission = async (id) => {
            if (!confirm(`Attention : Voulez-vous vraiment supprimer définitivement la mission #${id} ?`)) return;
            try {
                await deleteDoc(getMissionDoc(id));
                showToast("Mission supprimée", "success");
                document.getElementById('detailModal').classList.add('hidden');
            } catch (e) { showToast("Erreur lors de la suppression", "error"); }
        };

        // --- AFFICHAGE DÉTAILLÉ & IMPRESSION ---
        window.openDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            if (!m) return;

            const isAdmin = userRole === 'admin';
            
            let historyHtml = (m.history || []).map(h => `
                <div class="flex gap-4 items-start">
                    <div class="w-2 h-2 rounded-full mt-1.5 ${h.status === 2 ? 'bg-emerald-500' : 'bg-blue-500'}"></div>
                    <div>
                        <p class="text-[10px] font-bold text-slate-800">${h.label}</p>
                        <p class="text-[8px] text-slate-400 font-medium">${h.date}</p>
                    </div>
                </div>
            `).join('');

            let html = `
                <div class="p-8 space-y-6 print-container">
                    <div class="flex justify-between items-start">
                        <div>
                            <p class="text-[10px] font-black text-blue-600 uppercase">Bordereau de livraison</p>
                            <h2 class="text-3xl font-black text-slate-900">#${m.id}</h2>
                            <p class="text-[10px] text-slate-400 font-bold">${m.date}</p>
                        </div>
                        <div id="qrcode" class="bg-white p-2 rounded-xl shadow-sm border border-slate-100"></div>
                    </div>

                    <div class="grid grid-cols-2 gap-6 bg-slate-50 p-6 rounded-[2rem]">
                        <div>
                            <p class="text-[9px] font-black text-slate-400 uppercase mb-2">Expéditeur</p>
                            <p class="font-bold text-sm">${m.en || 'Relais CT241'}</p>
                            <p class="text-xs text-slate-500">${m.eq || ''}</p>
                        </div>
                        <div>
                            <p class="text-[9px] font-black text-slate-400 uppercase mb-2">Destinataire</p>
                            <p class="font-bold text-sm text-blue-600">${m.dn}</p>
                            <p class="text-xs text-slate-500">${m.dq} • ${m.dt}</p>
                        </div>
                    </div>

                    <div class="border-y border-dashed border-slate-200 py-6 flex justify-between items-center">
                        <p class="text-[10px] font-black text-slate-400 uppercase">Montant à collecter</p>
                        <p class="text-2xl font-black text-slate-900">${m.p} CFA</p>
                    </div>

                    <div class="space-y-4 no-print">
                        <p class="text-[9px] font-black text-slate-400 uppercase">Progression de la mission</p>
                        <div class="space-y-4 border-l-2 border-slate-100 ml-1 pl-4">
                            ${historyHtml}
                        </div>
                    </div>

                    <div class="no-print pt-6 space-y-3">
                        ${m.s === 1 ? `<button onclick="updateStatus('${m.id}', 2)" class="w-full bg-emerald-500 text-white py-5 rounded-2xl font-black text-xs uppercase tracking-widest">Valider la livraison</button>` : ''}
                        ${isAdmin ? `<button onclick="deleteMission('${m.id}')" class="w-full bg-red-50 text-red-500 py-4 rounded-2xl font-black text-[9px] uppercase tracking-widest">Supprimer l'archive</button>` : ''}
                    </div>
                </div>
            `;

            document.getElementById('detailContent').innerHTML = html;
            document.getElementById('detailModal').classList.remove('hidden');
            
            setTimeout(() => {
                new QRCode(document.getElementById("qrcode"), { 
                    text: m.id, 
                    width: 70, 
                    height: 70,
                    colorDark : "#0f172a",
                    colorLight : "#ffffff",
                    correctLevel : QRCode.CorrectLevel.H
                });
            }, 100);
        };

        // --- SCANNER QR ---
        window.startScan = () => {
            document.getElementById('qrScannerModal').classList.remove('hidden');
            html5QrCode = new Html5Qrcode("reader");
            html5QrCode.start(
                { facingMode: "environment" }, 
                { fps: 15, qrbox: { width: 250, height: 250 } },
                (decodedText) => {
                    html5QrCode.stop();
                    document.getElementById('qrScannerModal').classList.add('hidden');
                    openDetail(decodedText);
                }
            ).catch(err => {
                showToast("Veuillez autoriser la caméra", "error");
                document.getElementById('qrScannerModal').classList.add('hidden');
            });
        };

        // --- ÉCOUTE DES DONNÉES ---
        const startListeners = () => {
            const q = query(getMissionsColl(), orderBy("ts", "desc"));
            onSnapshot(q, (snap) => {
                allMissions = [];
                snap.forEach(d => allMissions.push(d.data()));
                refreshTables();
            }, (error) => console.error("Erreur Firestore:", error));
        };

        const refreshTables = () => {
            const disp = document.getElementById('dispatchList');
            const liv = document.getElementById('livreurList');
            const arch = document.getElementById('archiveList');
            
            if (disp) disp.innerHTML = "";
            if (liv) liv.innerHTML = "";
            if (arch) arch.innerHTML = "";

            allMissions.forEach(m => {
                const isAdminOrDispatch = userRole === 'admin' || userRole === 'dispatch';
                const isAdminOrLivreur = userRole === 'admin' || userRole === 'livreur';

                if (m.s === 0 && isAdminOrDispatch) {
                    disp.innerHTML += `
                        <div onclick="openDetail('${m.id}')" class="bg-white p-5 rounded-[2rem] shadow-sm border border-slate-100 flex justify-between items-center mb-3 active:scale-95 transition-transform">
                            <div>
                                <p class="text-[9px] font-black text-blue-500">#${m.id}</p>
                                <p class="text-sm font-bold text-slate-800">${m.dn}</p>
                                <p class="text-[9px] text-slate-400 font-bold uppercase">${m.dq}</p>
                            </div>
                            <button onclick="event.stopPropagation(); updateStatus('${m.id}', 1)" class="bg-blue-600 text-white px-5 py-3 rounded-xl text-[9px] font-black uppercase">Assigner</button>
                        </div>`;
                } else if (m.s === 1 && isAdminOrLivreur) {
                    liv.innerHTML += `
                        <div onclick="openDetail('${m.id}')" class="bg-white p-6 rounded-[2.5rem] shadow-lg border-l-8 border-amber-400 mb-4 active:scale-95 transition-transform">
                            <div class="mb-4">
                                <span class="bg-amber-50 text-amber-600 text-[8px] font-black px-2 py-1 rounded-full uppercase">En cours</span>
                                <h4 class="font-black text-lg text-slate-900 mt-2">${m.dn}</h4>
                                <p class="text-[10px] text-slate-500 font-bold uppercase">${m.dq} • ${m.dt}</p>
                            </div>
                            <div class="flex gap-2">
                                <a href="tel:${m.dt}" class="flex-1 bg-slate-100 text-slate-600 py-3 rounded-xl font-black text-[9px] uppercase text-center">Appeler</a>
                                <button onclick="event.stopPropagation(); updateStatus('${m.id}', 2)" class="flex-[2] bg-slate-900 text-white py-3 rounded-xl font-black text-[9px] uppercase tracking-widest">Terminer</button>
                            </div>
                        </div>`;
                } else if (m.s === 2) {
                    arch.innerHTML += `
                        <tr onclick="openDetail('${m.id}')" class="border-b border-slate-800/50 text-[10px] active:bg-white/5 transition-colors cursor-pointer">
                            <td class="py-4 text-white/70 font-bold">#${m.id}</td>
                            <td class="py-4">
                                <div class="font-bold text-white">${m.dn}</div>
                                <div class="text-[8px] text-slate-500 uppercase">${m.dq}</div>
                            </td>
                            <td class="py-4 text-emerald-400 font-black text-right">${m.p} CFA</td>
                        </tr>`;
                }
            });
        };

        window.showToast = (msg, type) => {
            const t = document.getElementById('toast');
            t.innerText = msg;
            t.className = `fixed bottom-8 left-1/2 -translate-x-1/2 px-8 py-4 rounded-full text-white font-black text-[10px] uppercase tracking-widest z-[1000] shadow-2xl ${type==='success'?'bg-emerald-500':'bg-red-500'}`;
            t.classList.remove('hidden');
            setTimeout(() => t.classList.add('hidden'), 3000);
        };

    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; color: #0f172a; -webkit-tap-highlight-color: transparent; }
        .hidden { display: none !important; }
        
        @media print {
            .no-print { display: none !important; }
            body { background: white; }
            .print-container { padding: 0 !important; width: 100% !important; border: none !important; box-shadow: none !important; }
            #detailModal { position: absolute; top: 0; left: 0; width: 100%; height: auto; background: white; padding: 0; margin: 0; }
            #detailModal > div { box-shadow: none !important; border: none !important; width: 100% !important; max-width: 100% !important; }
        }
    </style>
</head>
<body class="antialiased select-none">

    <div id="toast" class="hidden"></div>

    <!-- CONNEXION -->
    <section id="authSection" class="fixed inset-0 bg-[#0f172a] flex items-center justify-center p-6 z-[2000]">
        <div class="bg-white w-full max-w-sm rounded-[3.5rem] p-10 shadow-2xl text-center">
            <div class="w-24 h-24 rounded-[2rem] bg-slate-50 mx-auto mb-8 flex items-center justify-center shadow-inner overflow-hidden border-2 border-slate-100">
                 <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="w-full h-full object-cover scale-110">
            </div>
            <h1 class="text-3xl font-black mb-2 tracking-tighter">CT241 Log</h1>
            <p class="text-slate-400 text-[10px] font-black uppercase tracking-[0.3em] mb-10">Interface Sécurisée</p>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" placeholder="votre@email.com" required class="w-full p-5 bg-slate-50 rounded-2xl font-bold outline-none text-sm border-2 border-transparent focus:border-blue-100 transition-all">
                <input type="password" id="loginPass" placeholder="••••••••" required class="w-full p-5 bg-slate-50 rounded-2xl font-bold outline-none text-sm border-2 border-transparent focus:border-blue-100 transition-all">
                <p id="loginError" class="text-[9px] text-red-500 font-black uppercase hidden">Identifiants incorrects</p>
                <button type="submit" class="w-full bg-slate-900 text-white py-6 rounded-2xl font-black uppercase text-[11px] tracking-widest active:scale-95 transition-all shadow-xl">Se connecter</button>
            </form>
        </div>
    </section>

    <!-- APPLICATION -->
    <main id="appContent" class="hidden min-h-screen pb-24">
        <nav class="bg-white/80 backdrop-blur-2xl p-5 border-b sticky top-0 z-50 flex justify-between items-center no-print">
            <div class="flex items-center gap-4">
                <div class="w-10 h-10 rounded-xl overflow-hidden border border-slate-100">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="w-full h-full object-cover">
                </div>
                <div>
                    <p id="userMail" class="text-[8px] font-black text-slate-400 uppercase tracking-widest">Utilisateur</p>
                    <p class="text-[10px] font-black text-slate-900 uppercase">Panel Logistique</p>
                </div>
            </div>
            <div class="flex gap-2">
                <button onclick="startScan()" class="bg-slate-100 w-11 h-11 rounded-xl flex items-center justify-center hover:bg-slate-200 transition-colors">📸</button>
                <button onclick="handleLogout()" class="bg-red-50 text-red-500 px-4 h-11 rounded-xl font-black text-[9px] uppercase tracking-wider">OFF</button>
            </div>
        </nav>

        <div class="max-w-md mx-auto p-6 space-y-10">
            
            <!-- SECTION CRÉATION (Visible Relais/Admin) -->
            <section id="createSection" class="role-view hidden bg-white p-10 rounded-[3.5rem] shadow-xl border border-slate-100 no-print">
                <div class="flex justify-between items-center mb-8">
                    <h2 class="font-black text-xl tracking-tighter">Nouveau Colis</h2>
                    <span id="nextIdDisplay" class="bg-blue-50 text-blue-600 px-4 py-2 rounded-xl font-black text-[10px] border border-blue-100">Génération...</span>
                </div>
                <div class="space-y-5">
                    <input type="text" id="expNom" placeholder="Nom Boutique / Expéditeur" class="w-full p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                    <div class="grid grid-cols-2 gap-3">
                        <input type="text" id="expQuartier" placeholder="Ville/Quartier" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold outline-none border-2 border-transparent focus:border-blue-100">
                        <input type="tel" id="expTel" placeholder="Contact" class="p-5 bg-slate-50 rounded-2xl text-sm font-bold outline-none border-2 border-transparent focus:border-blue-100">
                    </div>
                    <div class="h-px bg-slate-100 my-2"></div>
                    <input type="text" id="destNom" placeholder="Destinataire (Nom complet)" class="w-full p-5 bg-slate-50 rounded-2xl text-sm font-bold border-2 border-transparent focus:border-blue-100 outline-none transition-all">
                    <div class="grid grid-cols-2 gap-3">
                        <input type="text" id="dq" placeholder="Quartier Dest." class="p-5 bg-slate-50 rounded-2xl text-sm font-bold outline-none border-2 border-transparent focus:border-blue-100">
                        <input type="tel" id="dt" placeholder="Téléphone Dest." class="p-5 bg-slate-50 rounded-2xl text-sm font-bold outline-none border-2 border-transparent focus:border-blue-100">
                    </div>
                    <div class="pt-4">
                        <p class="text-[9px] font-black text-slate-400 uppercase mb-2 ml-1">Frais de livraison (CFA)</p>
                        <input type="number" id="price" placeholder="0" class="w-full p-6 bg-blue-50 text-blue-700 rounded-[2rem] text-center text-2xl font-black outline-none border-2 border-blue-100">
                    </div>
                    <button onclick="saveMission()" class="w-full bg-slate-900 text-white py-6 rounded-[2rem] font-black text-[10px] uppercase tracking-widest shadow-xl active:scale-95 transition-all">Valider l'enregistrement</button>
                </div>
            </section>

            <!-- SECTION DISPATCH (Visible Dispatch/Admin) -->
            <section id="dispatchSection" class="role-view hidden no-print">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-6 ml-2 flex items-center gap-2">
                    <span class="w-1.5 h-1.5 rounded-full bg-blue-500"></span>
                    Missions à assigner
                </h3>
                <div id="dispatchList" class="space-y-3"></div>
            </section>

            <!-- SECTION LIVREUR (Visible Livreur/Admin) -->
            <section id="livreurSection" class="role-view hidden no-print">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-6 ml-2 flex items-center gap-2">
                    <span class="w-1.5 h-1.5 rounded-full bg-amber-500"></span>
                    Livraisons en cours
                </h3>
                <div id="livreurList" class="space-y-4"></div>
            </section>

            <!-- SECTION ARCHIVES (Visible par tous) -->
            <section id="archiveSection" class="role-view hidden bg-[#0f172a] rounded-[3.5rem] p-10 shadow-2xl no-print">
                <div class="flex justify-between items-center mb-10">
                    <h3 class="text-white text-[11px] font-black uppercase tracking-widest">Archives & Recettes</h3>
                    <div class="w-8 h-8 bg-white/5 rounded-full flex items-center justify-center text-xs">📦</div>
                </div>
                <div class="overflow-x-auto">
                    <table class="w-full">
                        <tbody id="archiveList"></tbody>
                    </table>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL DÉTAIL & IMPRESSION -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/90 z-[500] hidden flex items-center justify-center p-4 backdrop-blur-sm">
        <div class="w-full max-w-md bg-white rounded-[3.5rem] overflow-hidden shadow-2xl transition-all scale-100">
            <div id="detailContent" class="max-h-[80vh] overflow-y-auto">
                <!-- Rempli par JS -->
            </div>
            <div class="p-8 bg-slate-50 flex gap-4 no-print border-t border-slate-100">
                <button onclick="window.print()" class="flex-[2] bg-slate-900 text-white font-black py-5 rounded-2xl text-[10px] uppercase tracking-widest shadow-lg hover:bg-black transition-colors">🖨️ Imprimer</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="flex-1 bg-white border border-slate-200 text-slate-500 font-black py-5 rounded-2xl text-[10px] uppercase tracking-widest">Fermer</button>
            </div>
        </div>
    </div>

    <!-- MODAL SCANNER QR -->
    <div id="qrScannerModal" class="fixed inset-0 bg-black/95 z-[900] hidden flex flex-col items-center justify-center p-8">
        <div class="w-full max-w-sm space-y-8">
            <div id="reader" class="rounded-[3rem] overflow-hidden bg-slate-800 border-4 border-white/5 shadow-2xl"></div>
            <button onclick="document.getElementById('qrScannerModal').classList.add('hidden'); if(html5QrCode) html5QrCode.stop();" class="w-full bg-red-600 text-white py-6 rounded-[2rem] font-black text-xs uppercase tracking-widest">Annuler le scan</button>
        </div>
    </div>

</body>
</html>
