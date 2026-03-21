<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>CT241 - Logistique & Cloud</title>
    
    <meta name="theme-color" content="#0f172a">
    <script src="https://cdn.tailwindcss.com"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, collection, onSnapshot, query, addDoc, updateDoc, deleteDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Configuration Firebase
        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'ct241-log-v1';

        // État de l'application
        let user = null;
        let deliveries = [];

        // 1. Authentification (Règle 3)
        const initAuth = async () => {
            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Erreur Auth:", error);
            }
        };

        initAuth();

        onAuthStateChanged(auth, (u) => {
            user = u;
            if (user) {
                console.log("Connecté en tant que:", user.uid);
                setupRealtimeSync();
            }
        });

        // 2. Synchronisation Temps Réel (Règle 1 & 2)
        function setupRealtimeSync() {
            if (!user) return;

            // Chemin strict : /artifacts/{appId}/public/data/deliveries
            const deliveriesRef = collection(db, 'artifacts', appId, 'public', 'data', 'deliveries');
            
            onSnapshot(deliveriesRef, (snapshot) => {
                deliveries = snapshot.docs.map(doc => ({
                    id: doc.id,
                    ...doc.data()
                }));
                renderDeliveries();
                updateStats();
            }, (error) => {
                console.error("Erreur Firestore:", error);
            });
        }

        // 3. Fonctions de base de données
        window.addDelivery = async (e) => {
            e.preventDefault();
            if (!user) return;

            const formData = new FormData(e.target);
            const newDelivery = {
                client: formData.get('client'),
                destination: formData.get('destination'),
                status: 'En attente',
                timestamp: Date.now(),
                createdBy: user.uid
            };

            try {
                const deliveriesRef = collection(db, 'artifacts', appId, 'public', 'data', 'deliveries');
                await addDoc(deliveriesRef, newDelivery);
                e.target.reset();
                toggleModal('addModal');
            } catch (error) {
                alert("Erreur lors de l'ajout");
            }
        };

        window.updateStatus = async (id, newStatus) => {
            if (!user) return;
            const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'deliveries', id);
            await updateDoc(docRef, { status: newStatus });
        };

        window.deleteDelivery = async (id) => {
            if (!user || !confirm("Supprimer cette livraison ?")) return;
            const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'deliveries', id);
            await deleteDoc(docRef);
        };

        // UI - Rendu
        function renderDeliveries() {
            const list = document.getElementById('deliveryList');
            if (!list) return;

            list.innerHTML = deliveries.length === 0 
                ? `<div class="p-10 text-center text-slate-400">Aucune livraison enregistrée</div>`
                : deliveries.map(d => `
                    <div class="bg-white p-4 rounded-2xl shadow-sm border border-slate-100 mb-3 flex justify-between items-center animate-in fade-in slide-in-from-bottom-2">
                        <div>
                            <h3 class="font-bold text-slate-800">${d.client}</h3>
                            <p class="text-xs text-slate-500">${d.destination}</p>
                            <span class="inline-block mt-2 px-2 py-1 rounded-lg text-[10px] font-black uppercase ${getStatusClass(d.status)}">
                                ${d.status}
                            </span>
                        </div>
                        <div class="flex gap-2">
                            <button onclick="updateStatus('${d.id}', 'Livré')" class="p-2 bg-green-50 text-green-600 rounded-xl">✓</button>
                            <button onclick="deleteDelivery('${d.id}')" class="p-2 bg-red-50 text-red-600 rounded-xl">✕</button>
                        </div>
                    </div>
                `).join('');
        }

        function updateStats() {
            document.getElementById('totalCount').innerText = deliveries.length;
            document.getElementById('pendingCount').innerText = deliveries.filter(d => d.status !== 'Livré').length;
        }

        function getStatusClass(status) {
            return status === 'Livré' ? 'bg-green-100 text-green-700' : 'bg-amber-100 text-amber-700';
        }

        window.toggleModal = (id) => {
            document.getElementById(id).classList.toggle('hidden');
            document.getElementById(id).classList.toggle('flex');
        };

    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700;900&display=swap');
        body { font-family: 'Inter', sans-serif; background-color: #f8fafc; }
    </style>
</head>
<body class="pb-20">

    <!-- Header -->
    <header class="bg-slate-900 text-white p-6 rounded-b-[40px] shadow-2xl">
        <div class="flex justify-between items-center mb-6">
            <div>
                <p class="text-slate-400 text-xs font-bold uppercase tracking-widest">Tableau de bord</p>
                <h1 class="text-2xl font-black">CT241 LOGISTIQUE</h1>
            </div>
            <div class="w-12 h-12 bg-white/10 rounded-2xl flex items-center justify-center text-xl">📦</div>
        </div>
        
        <div class="grid grid-cols-2 gap-4">
            <div class="bg-white/10 p-4 rounded-3xl backdrop-blur-md">
                <p class="text-[10px] uppercase font-bold text-slate-400">Total</p>
                <p id="totalCount" class="text-2xl font-black">0</p>
            </div>
            <div class="bg-amber-500 p-4 rounded-3xl shadow-lg shadow-amber-500/20">
                <p class="text-[10px] uppercase font-bold text-amber-900/60">En cours</p>
                <p id="pendingCount" class="text-2xl font-black text-amber-950">0</p>
            </div>
        </div>
    </header>

    <!-- Main Content -->
    <main class="p-6">
        <div class="flex justify-between items-center mb-4">
            <h2 class="font-black text-slate-800 uppercase text-sm tracking-tighter">Livraisons Récentes</h2>
            <button onclick="toggleModal('addModal')" class="bg-slate-900 text-white px-4 py-2 rounded-xl text-xs font-bold">+ AJOUTER</button>
        </div>

        <div id="deliveryList">
            <!-- Injecté par JS -->
            <div class="p-10 text-center text-slate-400">Chargement des données...</div>
        </div>
    </main>

    <!-- Modal Ajout -->
    <div id="addModal" class="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-50 hidden items-center justify-center p-6">
        <div class="bg-white w-full max-w-md rounded-[32px] overflow-hidden shadow-2xl">
            <div class="p-8">
                <h2 class="text-xl font-black mb-6">Nouvelle Livraison</h2>
                <form onsubmit="addDelivery(event)" class="space-y-4">
                    <div>
                        <label class="block text-[10px] font-black uppercase text-slate-400 mb-1 ml-1">Client</label>
                        <input name="client" required class="w-full bg-slate-50 border-none p-4 rounded-2xl focus:ring-2 ring-slate-900" placeholder="Nom du client">
                    </div>
                    <div>
                        <label class="block text-[10px] font-black uppercase text-slate-400 mb-1 ml-1">Destination</label>
                        <input name="destination" required class="w-full bg-slate-50 border-none p-4 rounded-2xl focus:ring-2 ring-slate-900" placeholder="Quartier / Ville">
                    </div>
                    <div class="flex gap-2 pt-4">
                        <button type="button" onclick="toggleModal('addModal')" class="flex-1 py-4 font-bold text-slate-400">Annuler</button>
                        <button type="submit" class="flex-[2] bg-slate-900 text-white py-4 rounded-2xl font-black shadow-lg shadow-slate-900/20">CONFIRMER</button>
                    </div>
                </form>
            </div>
        </div>
    </div>

</body>
</html>
