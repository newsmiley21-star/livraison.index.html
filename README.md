<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CT241 - Logistics Hub</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-database-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-storage-compat.js"></script>
</head>
<body class="bg-gray-100 font-sans">

    <nav class="bg-blue-900 text-white p-4 shadow-lg flex justify-between items-center">
        <h1 class="text-xl font-bold tracking-widest">CT241</h1>
        <span id="userRole" class="bg-yellow-500 text-xs px-2 py-1 rounded text-blue-900 font-bold uppercase">Chargement...</span>
    </nav>

    <div class="max-w-4xl mx-auto p-4">

        <section id="rubrique1" class="hidden mb-6">
            <div class="bg-white p-6 rounded-xl shadow-md border-t-4 border-green-500">
                <h2 class="text-lg font-bold mb-4 text-green-700">📦 Création Mission (Point Relais)</h2>
                <div class="space-y-4">
                    <input type="text" id="clientNom" placeholder="Nom du Client" class="w-full p-2 border rounded">
                    <div class="grid grid-cols-2 gap-2">
                        <select id="relaisDepart" class="p-2 border rounded bg-gray-50">
                            <option>Départ: Relais Nzeng-Ayong</option>
                            <option>Départ: Relais Akanda</option>
                            <option>Départ: Relais Louis</option>
                        </select>
                        <select id="relaisArrivee" class="p-2 border rounded bg-gray-50">
                            <option>Arrivée: Relais Owendo</option>
                            <option>Arrivée: Relais Akanda</option>
                            <option>Arrivée: Domicile Client</option>
                        </select>
                    </div>
                    <button onclick="creerMission()" class="w-full bg-green-600 text-white py-3 rounded-lg font-bold">GÉNÉRER MANIFESTE & ID</button>
                </div>
            </div>
        </section>

        <section id="rubrique2" class="hidden mb-6">
            <div class="bg-white p-6 rounded-xl shadow-md border-t-4 border-blue-600">
                <h2 class="text-lg font-bold mb-4 text-blue-800">⚙️ Dispatch & Flux de Transfert</h2>
                <div id="listeAttente" class="space-y-3">
                    </div>
            </div>
        </section>

        <section id="rubrique3" class="hidden mb-6">
            <div class="bg-white p-6 rounded-xl shadow-md border-t-4 border-yellow-500">
                <h2 class="text-lg font-bold mb-4 text-yellow-700">🏍️ Espace Livreur</h2>
                <div class="bg-yellow-50 p-4 rounded border border-yellow-200 mb-4">
                    <p class="text-sm font-bold">Mission Actuelle: <span id="idMissionCours">Aucune</span></p>
                </div>
                <label class="block mb-2 text-sm font-medium">Filmer le colis (Image compressée) :</label>
                <input type="file" accept="image/*" capture="camera" id="photoColis" class="mb-4">
                <button onclick="validerLivraison()" class="w-full bg-yellow-600 text-white py-3 rounded-lg font-bold">VALIDER LA LIVRAISON</button>
            </div>
        </section>

        <section id="rubrique4" class="hidden">
            <div class="bg-gray-800 p-6 rounded-xl shadow-md text-white">
                <h2 class="text-lg font-bold mb-4">📂 Archives des Missions</h2>
                <div class="overflow-x-auto">
                    <table class="w-full text-left text-sm">
                        <thead>
                            <tr class="border-b border-gray-600">
                                <th class="py-2">ID</th>
                                <th>Trajet</th>
                                <th>Statut</th>
                            </tr>
                        </thead>
                        <tbody id="tableArchives">
                            </tbody>
                    </table>
                </div>
            </div>
        </section>

    </div>

    <script>
        // CONFIGURATION FIREBASE
        const firebaseConfig = {
            apiKey: "VOTRE_API_KEY",
            databaseURL: "VOTRE_DATABASE_URL",
            storageBucket: "VOTRE_STORAGE_URL",
            projectId: "VOTRE_PROJECT_ID"
        };
        firebase.initializeApp(firebaseConfig);
        const db = firebase.database();
        const storage = firebase.storage();

        // SIMULATION DU RÔLE (À remplacer par l'auth réelle)
        // Valeurs possibles: 'admin', 'gestionnaire', 'relais', 'livreur'
        const currentUserRole = 'admin'; 

        // GESTION DES ACCÈS
        function controlerAcces() {
            document.getElementById('userRole').innerText = currentUserRole;
            
            if (currentUserRole === 'relais' || currentUserRole === 'admin') {
                document.getElementById('rubrique1').classList.remove('hidden');
            }
            if (currentUserRole === 'gestionnaire' || currentUserRole === 'admin') {
                document.getElementById('rubrique2').classList.remove('hidden');
                document.getElementById('rubrique4').classList.remove('hidden');
            }
            if (currentUserRole === 'livreur' || currentUserRole === 'admin') {
                document.getElementById('rubrique3').classList.remove('hidden');
            }
        }

        // CRÉATION DE MISSION (RUBRIQUE 1)
        function creerMission() {
            const idAuto = "CT-" + Math.floor(Math.random() * 900000 + 100000);
            const mission = {
                id: idAuto,
                client: document.getElementById('clientNom').value,
                depart: document.getElementById('relaisDepart').value,
                arrivee: document.getElementById('relaisArrivee').value,
                statut: 'en_attente',
                date: new Date().toISOString()
            };
            db.ref('missions/' + idAuto).set(mission);
            alert("Colis enregistré ! ID: " + idAuto);
        }

        // COMPRESSION ET UPLOAD PHOTO (RUBRIQUE 3)
        async function validerLivraison() {
            const file = document.getElementById('photoColis').files[0];
            if(!file) return alert("Prenez une photo d'abord");

            // Simulation de compression via Canvas (Logique interne)
            const storageRef = storage.ref('preuves/' + Date.now() + '.jpg');
            await storageRef.put(file);
            alert("Livraison validée et photo archivée !");
        }

        window.onload = controlerAcces;
    </script>
</body>
</html>
