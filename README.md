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

    <link rel="manifest" id="pwa-manifest">
    <link rel="icon" type="image/png" href="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png">
    <script src="https://cdn.tailwindcss.com"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInWithEmailAndPassword, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, updateDoc, deleteDoc, onSnapshot, query, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

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
        let currentMissionId = null;
        let searchQuery = "";
        let filterToday = false;

        // --- AUTH ---
        window.handleLogin = async (e) => {
            e.preventDefault();
            const email = document.getElementById('loginEmail').value;
            const pass = document.getElementById('loginPass').value;
            const btn = document.getElementById('loginBtn');
            btn.disabled = true;
            btn.innerText = "Connexion...";
            try { await signInWithEmailAndPassword(auth, email, pass); } 
            catch (error) { 
                document.getElementById('loginError').classList.remove('hidden');
                btn.disabled = false; btn.innerText = "ACCÉDER AU PORTAIL";
            }
        };

        window.handleLogout = async () => { await signOut(auth); location.reload(); };

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
            }
        });

        const assignRole = (email) => {
            const e = email.toLowerCase();
            if (e.includes('admin')) userRole = 'admin';
            else if (e.includes('relais') || e.includes('boutique')) userRole = 'relais';
            else if (e.includes('dispatch')) userRole = 'dispatch';
            else if (e.includes('livreur')) userRole = 'livreur';
            
            document.querySelectorAll('.role-section').forEach(s => s.classList.add('hidden'));
            const badge = document.getElementById('badgeDisplay');
            const colors = { admin: 'bg-red-600', dispatch: 'bg-blue-600', relais: 'bg-emerald-600', livreur: 'bg-amber-600' };
            badge.innerHTML = `<span class="px-3 py-1 rounded-full text-[10px] font-black uppercase text-white ${colors[userRole] || 'bg-slate-500'}">${userRole}</span>`;

            if (userRole === 'admin') document.querySelectorAll('.role-section').forEach(s => s.classList.remove('hidden'));
            else if (userRole === 'relais') { document.getElementById('rubrique1').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'dispatch') { document.getElementById('rubrique2').classList.remove('hidden'); document.getElementById('rubrique4').classList.remove('hidden'); }
            else if (userRole === 'livreur') { document.getElementById('rubrique3').classList.remove('hidden'); document.getElementById('statLivreurSection').classList.remove('hidden'); }
        };

        // --- MISSIONS ---
        const prepareNextId = () => {
            window.nextId = "2026-" + Math.floor(Math.random() * 900000 + 100000);
            document.getElementById('displayNextId').innerText = window.nextId;
        };

        window.genererMission = async () => {
            const fields = {
                en: document.getElementById('expNom').value,
                eq: document.getElementById('expQuartier').value,
                et: document.getElementById('expTel').value,
                dn: document.getElementById('destNom').value,
                dq: document.getElementById('destQuartier').value,
                dt: document.getElementById('destTel').value,
                n: parseInt(document.getElementById('natureColis').value),
                v: parseFloat(document.getElementById('valeurDeclaree').value) || 0,
                p: parseFloat(document.getElementById('fraisLivraison').value) || 0,
                r: parseInt(document.getElementById('modeReglement').value),
            };
            if (!fields.en || !fields.dt || !fields.p) return showToast("Champs obligatoires !", "error");

            try {
                const now = Date.now();
                await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', window.nextId), {
                    ...fields, id: window.nextId, s: 0, ca: now, cad: new Date(now).toISOString().split('T')[0], ce: currentUser.email
                });
                showToast("Mission créée !", "success");
                prepareNextId();
                renderUI();
            } catch (e) { showToast("Erreur création", "error"); }
        };

        window.supprimerMission = async (id) => {
            if (confirm("Supprimer cette mission ?")) {
                await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id));
                showToast("Mission supprimée", "success");
            }
        };

        window.publierMission = async (id) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', id), { s: 1 });
            showToast("Mission publiée", "success");
        };

        window.toggleFilterToday = () => {
            filterToday = !filterToday;
            document.getElementById('btnFilterToday').className = filterToday ? "bg-yellow-500 text-slate-900 px-4 py-2 rounded-xl font-black text-[10px]" : "bg-slate-800 text-slate-400 px-4 py-2 rounded-xl font-black text-[10px]";
            renderUI();
        };

        const calculateStats = () => {
            const today = new Date().toISOString().split('T')[0];
            const stats = { caTotal: 0, livraisonsTotal: 0, caJour: 0, livreurs: {} };
            allMissions.forEach(m => {
                if (m.s === 2) {
                    stats.caTotal += m.p || 0;
                    stats.livraisonsTotal++;
                    if (m.lk === today) stats.caJour += m.p || 0;
                    const l = m.le || 'Inconnu';
                    if (!stats.livreurs[l]) stats.livreurs[l] = { count: 0, ca: 0, bonus: 0 };
                    stats.livreurs[l].count++;
                    stats.livreurs[l].ca += m.p || 0;
                    if (stats.livreurs[l].count > 17) stats.livreurs[l].bonus += 700;
                }
            });
            return stats;
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
            const stats = calculateStats();
            if (userRole === 'admin') {
                document.getElementById('dashCaTotal').innerText = stats.caTotal.toLocaleString() + ' CFA';
                document.getElementById('dashCaJour').innerText = stats.caJour.toLocaleString() + ' CFA';
                document.getElementById('dashLivTotal').innerText = stats.livraisonsTotal;
                const list = document.getElementById('dashLivreursList');
                list.innerHTML = "";
                Object.entries(stats.livreurs).forEach(([email, data]) => {
                    list.innerHTML += `<div class="flex justify-between p-3 bg-slate-800 rounded-xl mb-2 text-[10px]"><span class="text-white truncate w-32">${email}</span><span class="text-yellow-500 font-black">${data.count} missions</span><span class="text-emerald-500 font-black">+${data.bonus}</span></div>`;
                });
            }
            if (userRole === 'livreur') {
                const my = stats.livreurs[currentUser.email] || { count: 0, bonus: 0 };
                document.getElementById('myCount').innerText = my.count;
                document.getElementById('myBonus').innerText = my.bonus + ' CFA';
                document.getElementById('progBar').style.width = Math.min((my.count / 17) * 100, 100) + '%';
            }
        };

        window.renderUI = () => {
            const containers = { dispatch: document.getElementById('containerDispatch'), livreur: document.getElementById('containerLivreur'), archives: document.getElementById('archiveBody') };
            Object.values(containers).forEach(c => { if(c) c.innerHTML = "" });
            const today = new Date().toISOString().split('T')[0];

            allMissions.sort((a,b) => (b.ca || 0) - (a.ca || 0)).forEach(m => {
                if (filterToday && (m.lk !== today && m.cad !== today)) return;
                const isMyColis = m.ce === currentUser.email;
                const delBtn = userRole === 'admin' ? `<button onclick="supprimerMission('${m.id}')" class="p-2 text-red-400 bg-red-500/10 rounded-lg"><svg class="w-3 h-3" fill="currentColor" viewBox="0 0 24 24"><path d="M3 6h18v2H3V6m2 3h14v13c0 1.1-.9 2-2 2H7c-1.1 0-2-.9-2-2V9m3 3v8h2v-8H8m4 0v8h2v-8h-2m4 0v8h2v-8h-2Z"/></svg></button>` : '';

                if (m.s === 0 && (userRole === 'admin' || userRole === 'dispatch')) {
                    containers.dispatch.innerHTML += `
                    <div class="p-4 bg-white border border-slate-100 rounded-2xl mb-3 shadow-sm flex justify-between items-center">
                        <div class="text-[11px]"><span class="font-black text-blue-600 block">${m.id}</span><b>${m.dn}</b><br><span class="text-slate-400">${m.dq}</span></div>
                        <div class="flex gap-2">
                            ${delBtn}
                            <button onclick="openBonImpression('${m.id}')" class="bg-slate-100 text-slate-600 text-[9px] px-3 py-2 rounded-xl font-bold uppercase">Bon d'impression</button>
                            <button onclick="publierMission('${m.id}')" class="bg-blue-600 text-white text-[10px] px-5 py-2 rounded-xl font-bold uppercase">Expédier</button>
                        </div>
                    </div>`;
                } else if (m.s === 1 && (userRole === 'admin' || userRole === 'livreur')) {
                    containers.livreur.innerHTML += `
                    <div class="p-5 bg-amber-50 border border-amber-200 rounded-3xl mb-4 space-y-3">
                        <div class="flex justify-between font-black text-amber-700"><span>${m.id}</span><span class="text-[9px] uppercase">En cours</span></div>
                        <p class="text-[11px] text-amber-900"><b>Lieu:</b> ${m.dq}<br><b>Contact:</b> ${m.dn} (${m.dt})</p>
                        <div class="flex gap-2">
                             <button onclick="openBonImpression('${m.id}')" class="flex-1 bg-white border border-amber-200 text-amber-700 font-black py-4 rounded-2xl text-[10px]">VOIR BON</button>
                             <button onclick="openCamera('${m.id}')" class="flex-[2] bg-amber-500 text-white font-black py-4 rounded-2xl shadow-lg">LIVRÉ</button>
                        </div>
                    </div>`;
                }

                if ((userRole === 'admin' || userRole === 'dispatch' || (userRole === 'relais' && isMyColis)) && m.s === 2) {
                    if (!searchQuery || m.id.toLowerCase().includes(searchQuery) || m.dn.toLowerCase().includes(searchQuery)) {
                        containers.archives.innerHTML += `
                        <tr class="border-b border-slate-800 text-[10px] hover:bg-slate-800 transition">
                            <td onclick="openArchiveDetail('${m.id}')" class="p-4 font-bold text-white">${m.id}</td>
                            <td onclick="openArchiveDetail('${m.id}')" class="p-4 text-slate-300"><b>${m.dn}</b><br><span class="text-[8px] text-slate-500">${m.lk}</span></td>
                            <td class="p-4 text-center">${delBtn}</td>
                            <td class="p-4 text-right text-emerald-400 font-black">${(m.p || 0).toLocaleString()}</td>
                        </tr>`;
                    }
                }
            });
            document.getElementById('archiveCount').innerText = containers.archives.children.length;
        };

        // --- CAMERA & PHOTO ---
        window.openCamera = (id) => { currentMissionId = id; document.getElementById('cameraModal').classList.remove('hidden'); };
        window.processImage = (file) => {
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    canvas.width = 600; canvas.height = img.height * (600 / img.width);
                    canvas.getContext('2d').drawImage(img, 0, 0, canvas.width, canvas.height);
                    finalizeDelivery(canvas.toDataURL('image/jpeg', 0.6));
                };
                img.src = e.target.result;
            };
            reader.readAsDataURL(file);
        };
        const finalizeDelivery = async (photo) => {
            await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'missions', currentMissionId), { s: 2, img: photo, le: currentUser.email, lk: new Date().toISOString().split('T')[0] });
            document.getElementById('cameraModal').classList.add('hidden');
            showToast("Livraison Validée !", "success");
        };

        window.handleSearch = (v) => { searchQuery = v.toLowerCase(); renderUI(); };
        const showToast = (m, t) => {
            const el = document.getElementById('toast');
            el.innerText = m; el.className = `fixed bottom-10 left-1/2 -translate-x-1/2 px-6 py-3 rounded-full text-white font-bold text-xs shadow-2xl z-[500] ${t==='success'?'bg-emerald-600':'bg-red-600'}`;
            el.classList.remove('hidden'); setTimeout(() => el.classList.add('hidden'), 3000);
        };

        // --- DÉTAILS & BON D'IMPRESSION PROFESSIONNEL ---
        window.openBonImpression = (id) => {
            const m = allMissions.find(x => x.id === id);
            const natureMap = ["Standard", "Repas", "Documents", "Pharma"];
            const modeMap = ["Espèces", "Airtel Money", "Moov Money"];
            
            document.getElementById('detailContent').innerHTML = `
                <div class="print-area bg-white p-6 md:p-12 text-slate-900 min-h-screen flex flex-col">
                    <!-- HEADER LOGISTICIEN -->
                    <div class="flex justify-between items-start mb-8 pb-8 border-b-2 border-slate-100">
                        <div class="flex items-center gap-4">
                            <div class="w-16 h-16 bg-slate-900 rounded-2xl flex items-center justify-center">
                                <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="w-12 filter invert">
                            </div>
                            <div>
                                <h1 class="text-3xl font-black tracking-tighter text-slate-900 leading-none">CT241</h1>
                                <p class="text-[10px] font-bold uppercase tracking-[0.2em] text-slate-400 mt-1">Logistics & Performance</p>
                                <div class="mt-2 space-y-0.5 text-[9px] font-semibold text-slate-500">
                                    <p>BP 12450 • Zone Industrielle Oloumi</p>
                                    <p>Libreville, Gabon • Tél: +241 077 00 00 00</p>
                                </div>
                            </div>
                        </div>
                        <div class="text-right">
                            <h2 class="text-xl font-black uppercase text-slate-900">Bon de Livraison</h2>
                            <div class="inline-block bg-slate-100 px-3 py-1 rounded-lg mt-2">
                                <span class="text-[10px] font-black text-slate-500 uppercase">Référence :</span>
                                <span class="text-[11px] font-black text-slate-900">${m.id}</span>
                            </div>
                            <p class="text-[10px] text-slate-400 font-bold mt-2">Date : ${new Date(m.ca).toLocaleDateString('fr-FR')}</p>
                        </div>
                    </div>

                    <!-- BLOC EXPÉDITEUR / DESTINATAIRE -->
                    <div class="grid grid-cols-2 gap-12 mb-12">
                        <div class="relative">
                            <div class="absolute -left-4 top-0 bottom-0 w-1 bg-emerald-500 rounded-full"></div>
                            <h3 class="text-[10px] font-black uppercase tracking-widest text-slate-400 mb-4">Expéditeur / Origine</h3>
                            <div class="space-y-1">
                                <p class="text-sm font-black text-slate-900 uppercase">${m.en}</p>
                                <p class="text-[11px] font-medium text-slate-600">${m.eq}</p>
                                <p class="text-[11px] font-bold text-slate-900 mt-2">Contact: ${m.et}</p>
                            </div>
                        </div>
                        <div class="relative">
                            <div class="absolute -left-4 top-0 bottom-0 w-1 bg-blue-600 rounded-full"></div>
                            <h3 class="text-[10px] font-black uppercase tracking-widest text-slate-400 mb-4">Destinataire / Destination</h3>
                            <div class="space-y-1">
                                <p class="text-sm font-black text-slate-900 uppercase">${m.dn}</p>
                                <p class="text-[11px] font-medium text-slate-600">${m.dq}</p>
                                <p class="text-[11px] font-bold text-slate-900 mt-2">Contact: ${m.dt}</p>
                            </div>
                        </div>
                    </div>

                    <!-- TABLEAU DES DÉTAILS -->
                    <div class="flex-grow">
                        <table class="w-full text-left mb-8 border-collapse">
                            <thead>
                                <tr class="bg-slate-50 border-y border-slate-100">
                                    <th class="py-4 px-4 text-[10px] font-black uppercase text-slate-400 tracking-widest">Description de la marchandise</th>
                                    <th class="py-4 px-4 text-[10px] font-black uppercase text-slate-400 tracking-widest">Nature</th>
                                    <th class="py-4 px-4 text-[10px] font-black uppercase text-slate-400 tracking-widest text-right">Valeur Déclarée</th>
                                </tr>
                            </thead>
                            <tbody class="divide-y divide-slate-50">
                                <tr>
                                    <td class="py-6 px-4">
                                        <p class="text-sm font-black text-slate-900">Envoi Standard CT241</p>
                                        <p class="text-[10px] text-slate-400 mt-1 font-medium italic">Vérifié et scellé par nos agents de dispatch</p>
                                    </td>
                                    <td class="py-6 px-4">
                                        <span class="inline-block px-3 py-1 bg-slate-100 rounded-full text-[10px] font-black text-slate-700 uppercase">${natureMap[m.n]}</span>
                                    </td>
                                    <td class="py-6 px-4 text-right">
                                        <p class="text-sm font-black text-slate-900">${m.v.toLocaleString()} CFA</p>
                                    </td>
                                </tr>
                            </tbody>
                        </table>

                        <!-- RÉCAPITULATIF FINANCIER -->
                        <div class="flex justify-end mb-12">
                            <div class="w-72 space-y-3">
                                <div class="flex justify-between items-center text-[10px] font-bold text-slate-500 uppercase">
                                    <span>Frais de service</span>
                                    <span>${m.p.toLocaleString()} CFA</span>
                                </div>
                                <div class="flex justify-between items-center text-[10px] font-bold text-slate-500 uppercase">
                                    <span>Mode de paiement</span>
                                    <span class="text-slate-900">${modeMap[m.r]}</span>
                                </div>
                                <div class="h-px bg-slate-200 my-4"></div>
                                <div class="flex justify-between items-center">
                                    <span class="text-xs font-black uppercase text-slate-900">Total net à payer</span>
                                    <span class="text-2xl font-black text-slate-900">${m.p.toLocaleString()} CFA</span>
                                </div>
                            </div>
                        </div>
                    </div>

                    <!-- BLOC DE VALIDATION (SIGNATURES) -->
                    <div class="grid grid-cols-3 gap-6 mb-12 pt-12 border-t border-slate-100">
                        <div class="space-y-4">
                            <h4 class="text-[9px] font-black uppercase tracking-tighter text-slate-400">Signature Expéditeur</h4>
                            <div class="h-24 border border-dashed border-slate-200 rounded-2xl flex items-center justify-center italic text-[9px] text-slate-300">Cachet & Signature</div>
                        </div>
                        <div class="space-y-4">
                            <h4 class="text-[9px] font-black uppercase tracking-tighter text-slate-400">Prise en charge Livreur</h4>
                            <div class="h-24 border border-dashed border-slate-200 rounded-2xl p-3 flex flex-col justify-end">
                                <p class="text-[8px] font-bold text-slate-400 border-t border-slate-50 pt-2">Nom du chauffeur</p>
                            </div>
                        </div>
                        <div class="space-y-4">
                            <h4 class="text-[9px] font-black uppercase tracking-tighter text-slate-900">Accusé de Réception Client</h4>
                            <div class="h-24 bg-slate-50 border-2 border-slate-900 rounded-2xl p-3 flex flex-col justify-between">
                                <p class="text-[7px] font-bold text-slate-400 italic">"Je reconnais avoir reçu mon colis en bon état"</p>
                                <p class="text-[8px] font-black text-slate-900 border-t border-slate-200 pt-2">Date: ___/___/2026 à ___h___</p>
                            </div>
                        </div>
                    </div>

                    <!-- QR CODE & MENTIONS -->
                    <div class="flex justify-between items-end border-t border-slate-100 pt-6">
                        <div class="max-w-md space-y-2">
                            <p class="text-[8px] font-black uppercase text-slate-900">Conditions Générales de Transport</p>
                            <p class="text-[7px] leading-relaxed text-slate-400">
                                La responsabilité du transporteur est engagée uniquement sur la valeur déclarée. En cas de non-déclaration, le forfait minimum s'applique.
                                Toute réclamation doit être formulée par écrit dans les 24 heures suivant la réception. CT241 n'est pas responsable des retards liés à la circulation ou aux contrôles routiers.
                            </p>
                        </div>
                        <div class="text-right">
                           <img src="https://api.qrserver.com/v1/create-qr-code/?size=100x100&data=tel:%2B24177736065" 
     alt="QR Code Appel CT241" 
     style="width: 100px; height: 100px; display: block; margin: 0 auto;">
                            <p class="text-[7px] font-black uppercase tracking-widest text-slate-400">Authentification CT241 Express</p>
                        </div>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

        window.openArchiveDetail = (id) => {
            const m = allMissions.find(x => x.id === id);
            document.getElementById('detailContent').innerHTML = `
                <div class="bg-white p-6 space-y-4">
                    <div class="flex justify-between items-center pb-4 border-b border-slate-100">
                        <h2 class="text-xl font-black">Preuve Numérique : ${m.id}</h2>
                        <span class="bg-emerald-100 text-emerald-700 px-3 py-1 rounded-full font-black text-[10px]">LIVRÉ</span>
                    </div>
                    ${m.img ? `<img src="${m.img}" class="w-full rounded-3xl shadow-lg border-4 border-white">` : '<div class="h-40 bg-slate-100 flex items-center justify-center italic text-slate-400 rounded-3xl text-sm">Pas de photo de preuve</div>'}
                    <div class="grid grid-cols-2 gap-4 text-[11px]">
                        <div class="bg-slate-50 p-4 rounded-2xl">
                            <p class="text-slate-500 font-bold uppercase mb-1">Destinataire</p>
                            <p class="font-black text-slate-900">${m.dn}</p>
                            <p>${m.dq}</p>
                        </div>
                        <div class="bg-slate-50 p-4 rounded-2xl">
                            <p class="text-slate-500 font-bold uppercase mb-1">Agent Livreur</p>
                            <p class="font-black text-slate-900">${m.le || 'Inconnu'}</p>
                            <p>Livré le : ${m.lk || 'Date inconnue'}</p>
                        </div>
                    </div>
                </div>`;
            document.getElementById('detailModal').classList.remove('hidden');
        };

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; background-color: #f8fafc; }
        .hidden { display: none; }
        @media print { 
            body * { visibility: hidden; } 
            .print-area, .print-area * { visibility: visible; } 
            .print-area { position: absolute; left: 0; top: 0; width: 100%; border: none !important; } 
            .no-print { display: none !important; }
        }
    </style>
</head>
<body class="min-h-screen">

    <div id="toast" class="hidden"></div>

    <!-- AUTHENTICATION -->
    <section id="authSection" class="fixed inset-0 bg-slate-900 flex items-center justify-center p-6 z-[200]">
        <div class="w-full max-w-sm bg-white rounded-[3rem] p-10 space-y-8 shadow-2xl text-center">
            <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-24 mx-auto">
            <h1 class="text-2xl font-black text-slate-900 uppercase">Portail CT241</h1>
            <form onsubmit="handleLogin(event)" class="space-y-4">
                <input type="email" id="loginEmail" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="E-mail">
                <input type="password" id="loginPass" required class="w-full p-4 bg-slate-50 border border-slate-100 rounded-2xl font-bold text-xs" placeholder="Mot de passe">
                <p id="loginError" class="text-[10px] text-red-500 font-bold hidden">Identifiants incorrects.</p>
                <button id="loginBtn" type="submit" class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl hover:bg-black transition-colors">ACCÉDER AU PORTAIL</button>
            </form>
        </div>
    </section>

    <!-- APP MAIN CONTENT -->
    <main id="appContent" class="hidden pb-24">
        <nav class="bg-white/90 backdrop-blur-md p-4 sticky top-0 z-50 border-b border-slate-100">
            <div class="max-w-xl mx-auto flex justify-between items-center">
                <div class="flex items-center gap-3">
                    <img src="https://i.ibb.co/q3t8t3Rj/Gemini-Generated-Image-1pvtp31pvtp31pvt.png" class="h-8">
                    <div class="flex flex-col"><span class="text-[9px] font-black text-slate-900" id="userDisplayEmail">...</span><div id="badgeDisplay"></div></div>
                </div>
                <div class="flex gap-2">
                    <button onclick="toggleFilterToday()" id="btnFilterToday" class="bg-slate-100 text-slate-400 px-4 py-2 rounded-xl font-black text-[10px] uppercase">Aujourd'hui</button>
                    <button onclick="handleLogout()" class="p-2 bg-slate-100 rounded-full text-slate-400 hover:text-red-500"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m0 0l-4-4m4 4H7" /></svg></button>
                </div>
            </div>
        </nav>

        <div class="max-w-xl mx-auto p-4 space-y-6">
            <!-- DASHBOARD ADMIN / ANALYTICS -->
            <section id="adminDash" class="role-section hidden space-y-4">
                <div class="flex justify-between items-center">
                    <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest">Performance Flotte</h3>
                </div>
                <div class="grid grid-cols-3 gap-3">
                    <div class="bg-white p-4 rounded-3xl shadow-sm border border-slate-100 text-center"><span class="text-[8px] font-black text-slate-400 uppercase">CA Total</span><div id="dashCaTotal" class="text-[11px] font-black text-slate-900">0</div></div>
                    <div class="bg-white p-4 rounded-3xl shadow-sm border border-slate-100 text-center"><span class="text-[8px] font-black text-slate-400 uppercase">CA Jour</span><div id="dashCaJour" class="text-[11px] font-black text-emerald-600">0</div></div>
                    <div class="bg-white p-4 rounded-3xl shadow-sm border border-slate-100 text-center"><span class="text-[8px] font-black text-slate-400 uppercase">Livrées</span><div id="dashLivTotal" class="text-[11px] font-black text-blue-600">0</div></div>
                </div>
                <div class="bg-slate-900 p-5 rounded-[2rem] space-y-2 shadow-2xl" id="dashLivreursList"></div>
            </section>

            <!-- BOUTIQUE / RELAIS : CRÉATION -->
            <section id="rubrique1" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] shadow-xl border border-slate-100 overflow-hidden">
                    <div class="bg-slate-900 p-6 text-white flex justify-between items-center">
                        <div><p class="text-[9px] font-black uppercase opacity-50 tracking-widest">Nouveau Bordereau</p><h2 class="font-black text-sm">Réf : <span id="displayNextId">...</span></h2></div>
                        <div class="w-10 h-10 bg-white/10 rounded-xl flex items-center justify-center text-xl">✍️</div>
                    </div>
                    <div class="p-6 space-y-4">
                        <div class="space-y-4">
                            <label class="text-[9px] font-black uppercase text-slate-400 ml-2">Infos Expéditeur</label>
                            <input type="text" id="expNom" placeholder="Nom Boutique / Expéditeur" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none border-2 border-transparent focus:border-slate-900 transition-all">
                            <div class="grid grid-cols-2 gap-3">
                                <input type="text" id="expQuartier" placeholder="Quartier" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                                <input type="tel" id="expTel" placeholder="Tél Expéditeur" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                            </div>
                        </div>
                        <div class="h-px bg-slate-100 my-4"></div>
                        <div class="space-y-4">
                            <label class="text-[9px] font-black uppercase text-slate-400 ml-2">Infos Destinataire</label>
                            <input type="text" id="destNom" placeholder="Nom complet du client" class="w-full p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none border-2 border-transparent focus:border-blue-600 transition-all">
                            <div class="grid grid-cols-2 gap-3">
                                <select id="destQuartier" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                                    <option value="" disabled selected>Choisir la Zone</option>
                                    <option>Angondjé</option><option>Nzeng-Ayong</option><option>Oloumi</option><option>Akanda</option><option>Owendo</option><option>Pk0-12s</option><option>Glass</option>
                                </select>
                                <input type="tel" id="destTel" placeholder="Tél du client" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                            </div>
                        </div>
                        <div class="grid grid-cols-2 gap-3 pt-2">
                            <select id="natureColis" class="p-4 bg-slate-900 text-white rounded-2xl font-black text-[10px] outline-none">
                                <option value="0">📦 Envoi Standard</option><option value="1">🍟 Repas / Frais</option><option value="2">📁 Courrier / Doc</option><option value="3">💊 Pharmacie</option>
                            </select>
                            <select id="modeReglement" class="p-4 bg-slate-100 rounded-2xl font-black text-[10px] outline-none">
                                <option value="0">Espèces</option><option value="1">Airtel Money</option><option value="2">Moov Money</option>
                            </select>
                        </div>
                        <div class="grid grid-cols-2 gap-3">
                            <input type="number" id="valeurDeclaree" placeholder="Valeur Marchandise" class="p-4 bg-slate-50 rounded-2xl font-bold text-[11px] outline-none">
                            <input type="number" id="fraisLivraison" placeholder="PRIX LIVRAISON" class="p-4 bg-blue-600 text-white rounded-2xl font-black text-[11px] outline-none placeholder:text-blue-200">
                        </div>
                        <button onclick="genererMission()" class="w-full bg-slate-900 text-white font-black py-5 rounded-3xl shadow-xl hover:bg-black transition-all text-[11px] uppercase tracking-[0.2em]">Générer le Bon</button>
                    </div>
                </div>
            </section>

            <!-- DISPATCH / ADMIN : MISSIONS EN ATTENTE -->
            <section id="rubrique2" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-4 ml-2">Dispatching (En attente)</h3>
                <div id="containerDispatch"></div>
            </section>

            <!-- LIVREUR : MISSIONS ACTIVES -->
            <section id="rubrique3" class="role-section hidden">
                <h3 class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-4 ml-2">Mes Livraisons Actives</h3>
                <div id="containerLivreur"></div>
            </section>

            <!-- HISTORIQUE / ARCHIVES -->
            <section id="rubrique4" class="role-section hidden">
                <div class="bg-white rounded-[2.5rem] overflow-hidden shadow-xl border border-slate-100">
                    <div class="p-6 border-b border-slate-50">
                        <div class="flex justify-between items-center mb-6">
                            <h2 class="text-slate-900 font-black text-xs uppercase tracking-tight">Archives Numériques</h2>
                            <div id="archiveCount" class="bg-slate-900 text-white font-black text-[9px] px-3 py-1 rounded-full tracking-tighter">0</div>
                        </div>
                        <div class="relative">
                             <input type="text" oninput="handleSearch(this.value)" placeholder="Rechercher par ID ou Nom..." class="w-full p-4 bg-slate-50 rounded-2xl text-[11px] text-slate-900 font-bold outline-none border-2 border-transparent focus:border-slate-900 transition-all">
                             <span class="absolute right-4 top-1/2 -translate-y-1/2 opacity-20">🔍</span>
                        </div>
                    </div>
                    <div class="overflow-x-auto">
                        <table class="w-full text-left">
                            <tbody id="archiveBody"></tbody>
                        </table>
                    </div>
                </div>
            </section>
        </div>
    </main>

    <!-- MODAL DÉTAILS / PRINT -->
    <div id="detailModal" class="fixed inset-0 bg-slate-950/95 z-[400] hidden flex items-center justify-center p-0 md:p-8">
        <div class="w-full max-w-4xl bg-white md:rounded-[3rem] overflow-y-auto max-h-screen md:max-h-[95vh] animate-in slide-in-from-bottom duration-500 shadow-2xl">
            <div id="detailContent"></div>
            <div class="p-8 bg-slate-50 flex gap-3 no-print border-t border-slate-100 sticky bottom-0">
                <button onclick="window.print()" class="flex-1 bg-slate-900 text-white font-black py-5 rounded-3xl text-[11px] uppercase tracking-widest shadow-xl hover:bg-black transition-colors">🖨️ Lancer l'impression</button>
                <button onclick="document.getElementById('detailModal').classList.add('hidden')" class="px-10 bg-white border border-slate-200 text-slate-500 font-black py-5 rounded-3xl text-[11px] uppercase tracking-widest">Quitter</button>
            </div>
        </div>
    </div>

    <!-- MODAL CAMERA -->
    <div id="cameraModal" class="fixed inset-0 bg-black z-[300] hidden flex flex-col items-center justify-center p-8 text-white">
        <div class="text-center space-y-8 max-w-xs">
            <div class="w-24 h-24 bg-white/10 rounded-full flex items-center justify-center text-5xl mx-auto">📸</div>
            <h2 class="text-2xl font-black">Preuve de Livraison</h2>
            <p class="text-slate-500 text-xs">Prenez une photo claire du colis ou du bon de livraison dûment signé par le client.</p>
            <input type="file" id="fileInput" accept="image/*" capture="camera" class="hidden" onchange="processImage(this.files[0])">
            <button onclick="document.getElementById('fileInput').click()" class="w-full bg-white text-black font-black py-5 rounded-3xl shadow-2xl uppercase text-[11px] tracking-widest">Ouvrir l'appareil</button>
            <button onclick="document.getElementById('cameraModal').classList.add('hidden')" class="text-slate-600 text-[10px] font-bold underline uppercase tracking-widest">Abandonner</button>
        </div>
    </div>

</body>
</html>
