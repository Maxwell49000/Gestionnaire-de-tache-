<template>
  <fieldset>
    <!-- Affiche un message si la liste des tâches est vide: -->
    <h1 v-if="tasks.length === 0">Vous n'avez pas de tâche à faire :</h1>
   <!-- Si la condition précédente est fausse (c'est-à-dire s'il y a des tâches), affiche un autre message: -->
    <h1 v-else>Liste des tâches</h1>

    <!-- Formulaire d'ajout -->

    <!-- Il contient deux champs : un pour le nom de la tâche (taskName) et un pour la date limite de la tâche (taskDateLimit),
     et un bouton pour ajouter la tâche: -->
     <!-- Empêche l'envoi du formulaire par défaut et appelle la méthode addTask pour ajouter une tâche: -->
    <form @submit.prevent="addTask">
      <input type="text" placeholder="Nouvelle tâche" v-model="taskName" />
      <input type="date" v-model="taskDateLimit" />
      <!-- Le bouton "Ajouter" est désactivé si taskName est vide (taskName.length === 0): -->
      <button :disabled="taskName.length === 0">Ajouter</button>
    </form>

    <!-- Liste des tâches -->
    <ul>
      <!-- v-for="task in sortedTasks" : Utilise la directive v-for pour boucler à travers la liste des tâches triées (sortedTasks): -->
      <!-- Le :key="task.date" assure une clé unique pour chaque élément de la liste, ici basée sur la date de la tâche: -->
      <li v-for="task in sortedTasks" :key="task.date">
        <!-- v-model="task.completed" lie la case à cocher à la propriété completed de la tâche (marquer la tâche comme terminée ou non): -->
        <!-- @change="saveTasks" : Lorsque l'état de la case change, la méthode saveTasks est appelée pour sauvegarder l'état des tâches dans localStorage: -->
        <input type="checkbox" v-model="task.completed" @change="saveTasks" />
        <span :class="getTaskClass(task)">
          {{ task.name }} (Échéance : {{ formatDate(task.dateLimit) }})
        </span>
        <!-- Le bouton "Supprimer" supprime la tâche de la liste en appelant la méthode deleteTask(task.date): -->
        <button @click="deleteTask(task.date)">Supprimer</button>
      </li>
    </ul>

    <!-- Options -->

    <!-- Il y a une option pour masquer les tâches terminées, basée sur le modèle hideCompleted: -->
    <label>
      <input type="checkbox" v-model="hideCompleted"> Masquer les tâches terminées
    </label>
    <!-- Le nombre de tâches restantes est affiché si des tâches non terminées existent: -->
    <p v-if="remainingTasks > 0">
      {{ remainingTasks }} tâche{{ remainingTasks > 1 ? 's' : "" }} restante{{ remainingTasks > 1 ? 's' : "" }}
    </p>

    <!-- Alerte des tâches en retard ou à échéance -->

    <!-- Si des alertes existent (par exemple, tâches en retard ou proches de l'échéance), 
    elles sont affichées dans une boîte d'alerte avec une liste des messages: -->
    <div v-if="alerts.length" class="alerts">
      <ul>
        <li v-for="alert in alerts" :key="alert.message" class="alert">
          <strong>{{ alert.type }}:</strong> {{ alert.message }}
        </li>
      </ul>
    </div>
  </fieldset>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
// tasks : Contient la liste des tâches. Initialisé avec les données récupérées de localStorage:
const tasks = ref([]);
// taskName, taskDateLimit : Représentent le nom de la tâche et la date limite dans le formulaire:
const taskName = ref('');
const taskDateLimit = ref('');
// hideCompleted : Indique si les tâches terminées doivent être affichées ou non:
const hideCompleted = ref(false);
// alerts : Contient les alertes à afficher pour les tâches en retard ou urgentes:
const alerts = ref([]); // Tableau pour les alertes

// Charger les tâches sauvegardées
tasks.value = JSON.parse(localStorage.getItem('tasks')) || [];

// saveTasks : Sauvegarde la liste des tâches dans le localStorage:
const saveTasks = () => {
  localStorage.setItem('tasks', JSON.stringify(tasks.value));
};

// addTask : Ajoute une nouvelle tâche à la liste. Elle vérifie si le nom de la tâche n'est pas vide, 
// puis ajoute une nouvelle tâche avec un nom, un statut completed à false, et une date limite si elle est définie. 
// Ensuite, elle appelle saveTasks et checkDeadlines:
const addTask = () => {
  if (taskName.value.trim() !== '') {
    tasks.value.push({
      name: taskName.value,
      completed: false,
      date: Date.now(),
      dateLimit: taskDateLimit.value || null
    });
    taskName.value = '';
    taskDateLimit.value = '';
    saveTasks();
    checkDeadlines(); // Vérifier les échéances après ajout
  }
};

// deleteTask : Supprime une tâche de la liste en filtrant les tâches dont la date ne correspond pas à celle passée en paramètre:
const deleteTask = (date) => {
  tasks.value = tasks.value.filter(t => t.date !== date);
  saveTasks();
};

// Trier les tâches par priorité (complétées en bas, puis par date limite)
const sortedTasks = computed(() => {
  let sorted = [...tasks.value].sort((a, b) => {
    // Si hideCompleted est activé, seules les tâches non terminées sont affichées:
    if (a.completed !== b.completed) {
      return a.completed - b.completed;
    }
    if (!a.dateLimit) return 1; // Les tâches sans date limite en bas
    if (!b.dateLimit) return -1;
    return new Date(a.dateLimit) - new Date(b.dateLimit);
  });
  return hideCompleted.value ? sorted.filter(t => !t.completed) : sorted;
});

// Compte le nombre de tâches non terminées:
const remainingTasks = computed(() => tasks.value.filter(t => !t.completed).length);

// formatDate : Formate une date en format DD/MM/YYYY. Si aucune date n'est spécifiée, retourne "Aucune échéance":
const formatDate = (dateStr) => {
  if (!dateStr) return 'Aucune échéance';
  const date = new Date(dateStr);
  return date.toLocaleDateString('fr-FR');
};

// Déterminer la classe CSS en fonction de la date limite
const getTaskClass = (task) => {
  if (task.completed) return 'completed'; // Si la tâche est terminée, applique la classe 'completed'
  if (!task.dateLimit) return 'normal'; // Si la tâche n'a pas de date limite, applique 'normal'

  const today = new Date();
  const taskDate = new Date(task.dateLimit);

  if (taskDate < today) return 'late'; // Date dépassée : classe 'late'
  if ((taskDate - today) / (1000 * 60 * 60 * 24) <= 3) return 'warning'; // Moins de 3 jours restants : classe 'warning'

  return 'normal'; // Tâches normales, qui ne sont ni en retard ni urgentes
};
// Parcourt les tâches et vérifie leur date limite. Si une tâche est en retard ou arrive à échéance dans les 3 jours, 
// une alerte est ajoutée à la liste des alertes:
const checkDeadlines = () => {
  const today = new Date();
  alerts.value = []; // Réinitialiser les alertes à chaque vérification

  tasks.value.forEach(task => {
    if (task.completed || !task.dateLimit) return;

    const taskDate = new Date(task.dateLimit);
    const diffDays = (taskDate - today) / (1000 * 60 * 60 * 24);

    if (diffDays < 0) {
      alerts.value.push({ type: 'Retard', message: `La tâche "${task.name}" est en retard !` });
    } else if (diffDays <= 3) {
      alerts.value.push({ type: 'Alerte', message: `La tâche "${task.name}" arrive bientôt à échéance !` });
    }
  });
};

// Lors du montage du composant, la méthode checkDeadlines est appelée pour vérifier l'état des tâches en termes d'échéance:
onMounted(() => {
  checkDeadlines();
});
</script>


<style>

/* Style pour les tâches terminées */
.completed {
  text-decoration: line-through;
  opacity: 0.5;
}

/* Style pour les tâches en retard */
.late {
  color: red;
  font-weight: bold;
}

/* Style pour les tâches proches de l'échéance */
.warning {
  color: orange;
  font-weight: bold;
}

/* Style pour les tâches normales (sans échéance ou non urgentes) */
.normal {
  color: black; /* Ou toute autre couleur de ton choix pour les tâches normales */
}

/* Styles généraux */
body {
  font-family: 'Arial', sans-serif;
  background-color: #f4f4f4;
  color: #333;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  margin: 0;
}

/* Conteneur principal */
fieldset {
  background: white;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0px 4px 10px rgba(0, 0, 0, 0.1);
  max-width: 400px;
  width: 100%;
}

/* Titres */
h1 {
  font-size: 1.5em;
  text-align: center;
  color: #222;
}

/* Formulaires */
form {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 10px;
}

input[type="text"] {
  flex: 1;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 5px;
}

/* Boutons */
button {
  background: #007bff;
  color: white;
  border: none;
  padding: 8px 12px;
  border-radius: 5px;
  cursor: pointer;
  transition: 0.3s;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

button:hover {
  background: #0056b3;
}

/* Liste des tâches */
ul {
  list-style: none;
  padding: 0;
}

li {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px;
  border-bottom: 1px solid #ddd;
}

/* Checkbox */
input[type="checkbox"] {
  margin-right: 10px;
}

/* Style pour les tâches terminées */
.completed {
  text-decoration: line-through;
  color: #888;
  opacity: 0.7;
}

/* Bouton Supprimer */
button.delete {
  background: #dc3545;
}

button.delete:hover {
  background: #a71d2a;
}

/* Checkbox pour masquer les tâches */
label {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 0.9em;
  margin-top: 10px;
}

/* Message pour les alertes de tâches en retard ou urgentes */
.alerts {
  margin-top: 20px;
  background-color: #ffcc00;
  padding: 10px;
  border-radius: 5px;
  border: 1px solid #ff9900;
}

.alert {
  background-color: #f9f3c9;
  padding: 5px;
  margin-bottom: 10px;
}

.alert strong {
  color: #cc0000;
}



</style>

