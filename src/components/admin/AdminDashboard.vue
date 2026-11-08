<script setup>
import { ref, computed, onMounted } from 'vue';
import { supabase } from '../../services/supabase';
import { useToast } from '../../composables/useToast';

const { triggerToast } = useToast();

// ================= AUTHENTIFICATION & RÔLES ================= //
const user = ref(null);
const userProfile = ref(null);
const authLoading = ref(true);
const loginEmail = ref('');
const loginPassword = ref('');
const loginError = ref('');

const showSwitchAccountModal = ref(false);

const directionEmails = [
  'rodrigues.thomas2005@gmail.com',
  'lucierdg@outlook.com'
];

const currentRole = computed(() => {
  if (!user.value) return null;
  const email = user.value.email?.toLowerCase();
  if (directionEmails.includes(email)) return 'direction';
  return userProfile.value?.role || 'visiteur';
});

const isDirection = computed(() => currentRole.value === 'direction');

const userRoleTitle = computed(() => {
  const email = user.value?.email?.toLowerCase();
  if (email === 'rodrigues.thomas2005@gmail.com') return 'Président';
  if (email === 'lucierdg@outlook.com') return 'Vice-Présidente';
  const roleMap = {
    communication: 'Pôle Communication',
    sponsor: 'Pôle Sponsors',
    tresorerie: 'Trésorerie',
    animation: 'Pôle Animation',
    logistique: 'Pôle Logistique',
    parcours: 'Pôle Parcours'
  };
  return roleMap[currentRole.value] || 'Membre Association';
});

// ================= PÔLES & GESTION KANBAN ================= //
const allPoles = [
  { id: 'animation', label: 'Animation', icon: 'fa-bullhorn' },
  { id: 'sponsor', label: 'Sponsors', icon: 'fa-handshake' },
  { id: 'logistique', label: 'Logistique', icon: 'fa-boxes-packing' },
  { id: 'tresorerie', label: 'Trésorerie', icon: 'fa-vault' },
  { id: 'communication', label: 'Communication', icon: 'fa-hashtag' },
  { id: 'parcours', label: 'Parcours', icon: 'fa-route' }
];

const accessiblePoles = computed(() => {
  if (isDirection.value) return allPoles;
  return allPoles.filter(p => p.id === currentRole.value);
});

const selectedPole = ref('communication');

const canAccessTab = (tab) => {
  if (isDirection.value) return true;
  const r = currentRole.value;
  if (tab === 'poles') return accessiblePoles.value.length > 0;
  if (tab === 'materiel') return r !== 'tresorerie';
  if (tab === 'budget_tresorerie') return r === 'tresorerie';
  if (tab === 'messages') return r === 'communication';
  if (tab === 'benevoles') return r === 'communication';
  if (tab === 'faq') return r === 'communication';
  if (tab === 'sponsors') return r === 'sponsor';
  if (tab === 'cagnotte') return r === 'tresorerie';
  return false;
};

const currentTab = ref('poles');

// ================= DONNÉES DU DASHBOARD ================= //
const messages = ref([]);
const selectedMessage = ref(null);
const volunteers = ref([]);
const faqs = ref([]);
const sponsors = ref([]);
const fundraisingCurrent = ref(0);
const fundraisingGoal = ref(5000);
const updatingFundraising = ref(false);

// Kanban
const tasks = ref([]);
const steps = ref([]);
const comments = ref([]);

const columns = [
  { id: 'sujet', label: 'Sujet / Idée', textCol: 'text-slate-800' },
  { id: 'a_faire', label: 'À faire', textCol: 'text-amber-800' },
  { id: 'en_cours', label: 'En cours', textCol: 'text-blue-800' },
  { id: 'fait', label: 'Fait', textCol: 'text-emerald-800' }
];

const showTaskModal = ref(false);
const editingTask = ref(null);
const taskForm = ref({ titre: '', description: '', assigne_a: '', status: 'sujet' });

const showDetailModal = ref(false);
const currentTaskDetail = ref(null);
const newStepTitle = ref('');
const newComment = ref('');

// ================= GESTION DU MATÉRIEL & MODALES ================= //
const materialRequests = ref([]);
const showMaterialModal = ref(false);
const editingMaterialRequest = ref(null);
const materialForm = ref({
  article: '',
  quantite: 1,
  estimated_price: null,
  details: '',
  pole: ''
});

// Modale Décision (Logistique ou Trésorerie)
const showDecisionModal = ref(false);
const decisionTargetItem = ref(null);
const decisionType = ref('logistique');
const decisionForm = ref({ status: 'valide', comment: '' });

// Modale Suppression avec justification pour Trésorerie
const showDeleteReasonModal = ref(false);
const itemToDelete = ref(null);
const deleteReasonInput = ref('');

const isLogistiqueOrDirection = computed(() => isDirection.value || currentRole.value === 'logistique');
const isTresorerieOrDirection = computed(() => isDirection.value || currentRole.value === 'tresorerie');

const filteredMaterialRequests = computed(() => {
  if (isLogistiqueOrDirection.value) return materialRequests.value;
  return materialRequests.value.filter(m => m.pole === currentRole.value);
});

const totalEstimatedBudget = computed(() => {
  return materialRequests.value.reduce((acc, item) => {
    if (item.estimated_price !== null && item.estimated_price !== undefined) {
      return acc + (Number(item.estimated_price) * (item.quantite || 1));
    }
    return acc;
  }, 0);
});

const budgetSummaryByPole = computed(() => {
  const summary = {};
  allPoles.forEach(p => {
    summary[p.id] = { label: p.label, total: 0, count: 0, validatedTotal: 0 };
  });

  materialRequests.value.forEach(item => {
    const pId = item.pole || 'autre';
    if (!summary[pId]) {
      summary[pId] = { label: pId, total: 0, count: 0, validatedTotal: 0 };
    }
    const cost = item.estimated_price ? Number(item.estimated_price) * (item.quantite || 1) : 0;
    summary[pId].total += cost;
    summary[pId].count += 1;
    if (item.avis_tresorerie === 'valide') {
      summary[pId].validatedTotal += cost;
    }
  });

  return Object.values(summary).filter(s => s.count > 0);
});

const getStatusColorClass = (status) => {
  if (status === 'valide') return 'text-emerald-700 font-bold';
  if (status === 'refuse') return 'text-rose-600 font-bold';
  return 'text-amber-600 font-medium';
};

const getStatusLabel = (status) => {
  if (status === 'valide') return 'Validé';
  if (status === 'refuse') return 'Refusé';
  return 'En attente';
};

// ================= EXPORTS EXCEL SANS DOUBLONS ================= //
const exportToExcel = (type = 'logistique') => {
  let csvContent = '\uFEFF';
  let filename = '';

  if (type === 'logistique') {
    filename = `demandes_materiel_logistique_${new Date().toISOString().slice(0, 10)}.csv`;
    csvContent += 'Pôle;Article;Quantité;Justification du Besoin;Demandeur;Prix Estimé Unitaire (€);Total Estimé (€);Avis Logistique;Motif Logistique;Avis Trésorerie;Date\n';

    filteredMaterialRequests.value.forEach(req => {
      const pole = `"${(req.pole || '').replace(/"/g, '""')}"`;
      const article = `"${(req.article || '').replace(/"/g, '""')}"`;
      const qte = req.quantite || 1;
      const details = `"${(req.details || '').replace(/"/g, '""')}"`;
      const demandeur = `"${(req.demandeur || '').replace(/"/g, '""')}"`;
      const prix = req.estimated_price ? Number(req.estimated_price).toFixed(2).replace('.', ',') : '';
      const total = req.estimated_price ? (Number(req.estimated_price) * qte).toFixed(2).replace('.', ',') : '';
      const avisLog = `"${getStatusLabel(req.avis_logistique || req.status)}"`;
      const motifLog = `"${(req.reponse_logistique || '').replace(/"/g, '""')}"`;
      const avisTresor = `"${getStatusLabel(req.avis_tresorerie)}"`;
      const date = req.created_at ? new Date(req.created_at).toLocaleDateString('fr-FR') : '';

      csvContent += `${pole};${article};${qte};${details};${demandeur};${prix};${total};${avisLog};${motifLog};${avisTresor};${date}\n`;
    });
  } else if (type === 'tresorerie') {
    filename = `budget_materiel_tresorerie_${new Date().toISOString().slice(0, 10)}.csv`;
    csvContent += 'Pôle;Article;Quantité;Prix Unitaire (€);Montant Total Estimé (€);Avis Trésorerie;Justification Trésorerie;Avis Logistique;Demandeur;Date\n';

    materialRequests.value.forEach(req => {
      const pole = `"${(req.pole || '').replace(/"/g, '""')}"`;
      const article = `"${(req.article || '').replace(/"/g, '""')}"`;
      const qte = req.quantite || 1;
      const prix = req.estimated_price ? Number(req.estimated_price).toFixed(2).replace('.', ',') : '0,00';
      const total = req.estimated_price ? (Number(req.estimated_price) * qte).toFixed(2).replace('.', ',') : '0,00';
      const avisTresor = `"${getStatusLabel(req.avis_tresorerie)}"`;
      const motifTresor = `"${(req.reponse_tresorerie || '').replace(/"/g, '""')}"`;
      const avisLog = `"${getStatusLabel(req.avis_logistique || req.status)}"`;
      const demandeur = `"${(req.demandeur || '').replace(/"/g, '""')}"`;
      const date = req.created_at ? new Date(req.created_at).toLocaleDateString('fr-FR') : '';

      csvContent += `${pole};${article};${qte};${prix};${total};${avisTresor};${motifTresor};${avisLog};${demandeur};${date}\n`;
    });
  }

  const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.setAttribute('download', filename);
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  triggerToast('Export Excel téléchargé !');
};

// ================= ACTIONS MATÉRIEL & ÉDITION ================= //
const fetchMaterialRequests = async () => {
  const { data } = await supabase
    .from('material_requests')
    .select('*')
    .order('created_at', { ascending: false });
  if (data) materialRequests.value = data;
};

const openMaterialRequestModal = (item = null) => {
  editingMaterialRequest.value = item;
  if (item) {
    materialForm.value = {
      article: item.article,
      quantite: item.quantite,
      estimated_price: item.estimated_price,
      details: item.details,
      pole: item.pole
    };
  } else {
    materialForm.value = {
      article: '',
      quantite: 1,
      estimated_price: null,
      details: '',
      pole: isDirection.value ? selectedPole.value : currentRole.value
    };
  }
  showMaterialModal.value = true;
};

const submitMaterialRequest = async () => {
  if (!materialForm.value.article || materialForm.value.quantite < 1) {
    triggerToast('Veuillez préciser l\'article et une quantité valide.');
    return;
  }

  if (!materialForm.value.details || !materialForm.value.details.trim()) {
    triggerToast('La justification du besoin est obligatoire !');
    return;
  }

  const payload = {
    pole: materialForm.value.pole || currentRole.value,
    article: materialForm.value.article.trim(),
    quantite: materialForm.value.quantite,
    estimated_price: (materialForm.value.estimated_price !== null && materialForm.value.estimated_price !== '') 
      ? Number(materialForm.value.estimated_price) 
      : null,
    details: materialForm.value.details.trim()
  };

  if (editingMaterialRequest.value) {
    const { error } = await supabase
      .from('material_requests')
      .update(payload)
      .eq('id', editingMaterialRequest.value.id);

    if (!error) {
      Object.assign(editingMaterialRequest.value, payload);
      showMaterialModal.value = false;
      triggerToast('Demande modifiée avec succès.');
    }
  } else {
    payload.demandeur = userRoleTitle.value;
    payload.status = 'en_attente';
    payload.avis_logistique = 'en_attente';
    payload.avis_tresorerie = 'en_attente';

    const { data, error } = await supabase.from('material_requests').insert([payload]).select();
    if (!error && data) {
      materialRequests.value.unshift(data[0]);
      showMaterialModal.value = false;
      triggerToast('Demande enregistrée.');
    }
  }
};

const openDecisionModal = (item, type) => {
  decisionTargetItem.value = item;
  decisionType.value = type;

  if (type === 'logistique') {
    decisionForm.value = {
      status: item.avis_logistique || item.status || 'valide',
      comment: item.reponse_logistique || ''
    };
  } else {
    decisionForm.value = {
      status: item.avis_tresorerie || 'valide',
      comment: item.reponse_tresorerie || ''
    };
  }

  showDecisionModal.value = true;
};

const saveDecision = async () => {
  if (!decisionTargetItem.value) return;

  if (decisionForm.value.status === 'refuse' && (!decisionForm.value.comment || !decisionForm.value.comment.trim())) {
    triggerToast('Justification obligatoire pour un refus.');
    return;
  }

  const updatePayload = {};
  if (decisionType.value === 'logistique') {
    updatePayload.avis_logistique = decisionForm.value.status;
    updatePayload.status = decisionForm.value.status;
    updatePayload.reponse_logistique = decisionForm.value.comment.trim() || null;
  } else {
    updatePayload.avis_tresorerie = decisionForm.value.status;
    updatePayload.reponse_tresorerie = decisionForm.value.comment.trim() || null;
  }

  const { error } = await supabase
    .from('material_requests')
    .update(updatePayload)
    .eq('id', decisionTargetItem.value.id);

  if (!error) {
    Object.assign(decisionTargetItem.value, updatePayload);
    showDecisionModal.value = false;
    triggerToast(`Décision ${decisionType.value} enregistrée.`);
  }
};

// Suppression avec justification Trésorerie
const promptDeleteWithReason = (item) => {
  itemToDelete.value = item;
  deleteReasonInput.value = '';
  showDeleteReasonModal.value = true;
};

const confirmDeleteWithReason = async () => {
  if (!deleteReasonInput.value.trim()) {
    triggerToast('Veuillez indiquer la raison de la suppression pour le pôle.');
    return;
  }

  // Enregistre l'arbitrage refusé avec le motif avant de supprimer (ou suppression directe)
  const { error } = await supabase.from('material_requests').delete().eq('id', itemToDelete.value.id);
  if (!error) {
    materialRequests.value = materialRequests.value.filter(m => m.id !== itemToDelete.value.id);
    showDeleteReasonModal.value = false;
    triggerToast(`Ligne supprimée. Motif notifié : "${deleteReasonInput.value.trim()}"`);
  }
};

const deleteMaterialRequest = async (id) => {
  if (!confirm('Supprimer cette demande ?')) return;
  const { error } = await supabase.from('material_requests').delete().eq('id', id);
  if (!error) {
    materialRequests.value = materialRequests.value.filter(m => m.id !== id);
    triggerToast('Demande supprimée.');
  }
};

// ================= AUTHENTIFICATION & CYCLE DE VIE ================= //
const handleLogin = async () => {
  loginError.value = '';
  const { data, error } = await supabase.auth.signInWithPassword({
    email: loginEmail.value.trim(),
    password: loginPassword.value
  });

  if (error) {
    loginError.value = 'Identifiants incorrects ou compte inexistant.';
  } else {
    user.value = data.user;
    loginPassword.value = '';
    showSwitchAccountModal.value = false;
    await fetchProfileAndData();
    triggerToast(`Connecté sous : ${user.value.email}`);
  }
};

const handleLogout = async () => {
  await supabase.auth.signOut();
  user.value = null;
  userProfile.value = null;
  window.location.hash = '';
};

const openSwitchAccount = () => {
  loginEmail.value = '';
  loginPassword.value = '';
  loginError.value = '';
  showSwitchAccountModal.value = true;
};

const fetchProfileAndData = async () => {
  if (!user.value) return;

  if (!directionEmails.includes(user.value.email?.toLowerCase())) {
    const { data: prof } = await supabase
      .from('profiles')
      .select('role')
      .eq('id', user.value.id)
      .maybeSingle();

    if (prof) userProfile.value = prof;
  }

  if (accessiblePoles.value.length > 0) {
    selectedPole.value = accessiblePoles.value[0].id;
  }

  currentTab.value = 'poles';
  loadPermittedData();
};

const loadPermittedData = async () => {
  loadPoleData();
  fetchMaterialRequests();
};

const loadPoleData = async () => {
  const { data: tasksData } = await supabase.from('pole_tasks').select('*').order('id', { ascending: true });
  if (tasksData) tasks.value = tasksData;

  const { data: stepsData } = await supabase.from('pole_task_steps').select('*').order('id', { ascending: true });
  if (stepsData) steps.value = stepsData;

  const { data: commentsData } = await supabase.from('pole_task_comments').select('*').order('created_at', { ascending: true });
  if (commentsData) comments.value = commentsData;
};

// Kanban actions
const tasksByStatus = (status) => tasks.value.filter(t => t.pole === selectedPole.value && t.status === status);

const getTaskProgress = (taskId) => {
  const taskSteps = steps.value.filter(s => s.task_id === taskId);
  if (taskSteps.length === 0) return null;
  const done = taskSteps.filter(s => s.completed).length;
  return { done, total: taskSteps.length, percent: Math.round((done / taskSteps.length) * 100) };
};

const openCreateTask = (status = 'sujet') => {
  editingTask.value = null;
  taskForm.value = { titre: '', description: '', assigne_a: '', status };
  showTaskModal.value = true;
};

const openEditTask = (task) => {
  editingTask.value = task;
  taskForm.value = { titre: task.titre, description: task.description || '', assigne_a: task.assigne_a || '', status: task.status };
  showTaskModal.value = true;
};

const saveTask = async () => {
  if (!taskForm.value.titre) return;
  if (editingTask.value) {
    const { error } = await supabase.from('pole_tasks').update(taskForm.value).eq('id', editingTask.value.id);
    if (!error) {
      const idx = tasks.value.findIndex(t => t.id === editingTask.value.id);
      if (idx !== -1) tasks.value[idx] = { ...tasks.value[idx], ...taskForm.value };
      showTaskModal.value = false;
      triggerToast('Post-it mis à jour.');
    }
  } else {
    const { data, error } = await supabase.from('pole_tasks').insert([{ ...taskForm.value, pole: selectedPole.value }]).select();
    if (!error && data) {
      tasks.value.push(data[0]);
      showTaskModal.value = false;
      triggerToast('Post-it créé.');
    }
  }
};

const deleteTask = async (taskId) => {
  const { error } = await supabase.from('pole_tasks').delete().eq('id', taskId);
  if (!error) {
    tasks.value = tasks.value.filter(t => t.id !== taskId);
    triggerToast('Post-it supprimé.');
  }
};

const onDragStart = (event, task) => event.dataTransfer.setData('taskId', task.id.toString());

const onDrop = async (event, newStatus) => {
  const taskId = event.dataTransfer.getData('taskId');
  if (!taskId) return;
  const task = tasks.value.find(t => t.id == taskId);
  if (task && task.status !== newStatus) {
    task.status = newStatus;
    await supabase.from('pole_tasks').update({ status: newStatus }).eq('id', taskId);
    triggerToast('Post-it déplacé.');
  }
};

onMounted(async () => {
  const { data } = await supabase.auth.getSession();
  user.value = data.session?.user || null;
  authLoading.value = false;
  if (user.value) await fetchProfileAndData();
});
</script>
<style scoped>
nav {
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE et Edge */
}
nav::-webkit-scrollbar {
  display: none; /* Chrome, Safari, Opera */
}
</style>
<template>
  <div v-if="authLoading" class="min-h-screen flex items-center justify-center bg-white">
    <i class="fa-solid fa-circle-notch fa-spin text-2xl text-emerald-600"></i>
  </div>

  <!-- AUTHENTIFICATION -->
  <div v-else-if="!user" class="min-h-screen bg-slate-900 flex items-center justify-center p-4">
    <div class="bg-white rounded-2xl max-w-md w-full p-8 shadow-xl">
      <div class="text-center mb-6">
        <h2 class="text-2xl font-bold text-slate-900">Espace Privé</h2>
        <p class="text-xs text-slate-500 mt-1">Connexion par rôle & pôle</p>
      </div>
      <form @submit.prevent="handleLogin" class="space-y-4">
        <div>
          <label class="block text-xs font-semibold uppercase text-slate-500 mb-1">E-mail</label>
          <input v-model="loginEmail" required type="email" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-sm outline-none focus:border-emerald-600" />
        </div>
        <div>
          <label class="block text-xs font-semibold uppercase text-slate-500 mb-1">Mot de passe</label>
          <input v-model="loginPassword" required type="password" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-sm outline-none focus:border-emerald-600" />
        </div>
        <p v-if="loginError" class="text-xs text-rose-600">{{ loginError }}</p>
        <button type="submit" class="w-full py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-lg text-sm transition">
          Se connecter
        </button>
      </form>
    </div>
  </div>

  <!-- APPLICATION CONNECTÉE -->
  <div v-else class="min-h-screen bg-white p-6 sm:p-10 text-slate-800">
    <div class="max-w-7xl mx-auto">
      
      <!-- En-tête simplifié -->
      <div class="flex items-center justify-between pb-6 border-b border-slate-200 mb-6 flex-wrap gap-4">
        <div>
          <div class="flex items-center gap-3">
            <h1 class="text-2xl font-black text-slate-900">Espace {{ userRoleTitle }}</h1>
            <span v-if="isDirection" class="text-xs uppercase font-extrabold tracking-wider text-emerald-700">Direction</span>
          </div>
          <p class="text-xs text-slate-500">{{ user.email }}</p>
        </div>

        <div class="flex items-center gap-4 text-xs font-semibold">
          <button @click="openSwitchAccount" class="text-slate-600 hover:text-emerald-700 transition cursor-pointer">
            Changer de compte
          </button>
          <a href="/" class="text-slate-600 hover:text-emerald-700 transition">Voir le site</a>
          <button @click="handleLogout" class="text-rose-600 hover:underline transition cursor-pointer">Déconnexion</button>
        </div>
      </div>

      <!-- BARRE D'ONGLETS ÉPURÉE (Sans flèches ni barre de défilement) -->
<nav class="border-b border-slate-200 mb-8 overflow-hidden">
  <div class="flex items-center gap-8 -mb-px flex-wrap">
    
    <button
      v-if="canAccessTab('poles')"
      @click="currentTab = 'poles'"
      :class="currentTab === 'poles'
        ? 'text-emerald-700 border-b-2 border-emerald-600 font-bold'
        : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
      class="pb-3 text-sm transition-colors cursor-pointer"
    >
      Tableau Kanban
    </button>

    <!-- Onglet Logistique (exclu Trésorerie) -->
    <button
      v-if="canAccessTab('materiel')"
      @click="currentTab = 'materiel'"
      :class="currentTab === 'materiel'
        ? 'text-emerald-700 border-b-2 border-emerald-600 font-bold'
        : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
      class="pb-3 text-sm transition-colors cursor-pointer"
    >
      Besoins Matériel ({{ filteredMaterialRequests.length }})
    </button>

    <!-- Onglet Trésorerie -->
    <button
      v-if="canAccessTab('budget_tresorerie')"
      @click="currentTab = 'budget_tresorerie'"
      :class="currentTab === 'budget_tresorerie'
        ? 'text-emerald-700 border-b-2 border-emerald-600 font-bold'
        : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
      class="pb-3 text-sm transition-colors cursor-pointer"
    >
      Budget & Pôles
    </button>

  </div>
</nav>

      <!-- ================= 1. VUE KANBAN ================= -->
      <div v-if="currentTab === 'poles'" class="space-y-6">
        <div v-if="accessiblePoles.length > 1" class="border-b border-slate-100 pb-3 flex items-center gap-6 overflow-x-auto text-xs">
          <button 
            v-for="p in accessiblePoles" 
            :key="p.id"
            @click="selectedPole = p.id"
            :class="selectedPole === p.id ? 'text-emerald-700 font-bold' : 'text-slate-400 hover:text-slate-700'"
            class="transition cursor-pointer"
          >
            {{ p.label }} ({{ tasks.filter(t => t.pole === p.id).length }})
          </button>
        </div>

        <div class="flex items-center justify-between">
          <h2 class="text-lg font-bold text-slate-800">Pôle {{ allPoles.find(p => p.id === selectedPole)?.label }}</h2>
          <button @click="openCreateTask('sujet')" class="text-xs font-bold text-emerald-700 hover:text-emerald-800 transition cursor-pointer">
            + Nouveau Post-it
          </button>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 items-start">
          <div 
            v-for="col in columns" 
            :key="col.id"
            @dragover.prevent
            @drop="onDrop($event, col.id)"
            class="bg-slate-50 p-4 rounded-xl min-h-[460px] flex flex-col"
          >
            <div class="flex items-center justify-between pb-3 mb-3 border-b border-slate-200">
              <span :class="['text-xs font-bold uppercase tracking-wider', col.textCol]">{{ col.label }}</span>
              <span class="text-xs text-slate-400 font-mono">{{ tasksByStatus(col.id).length }}</span>
            </div>

            <div class="space-y-3 flex-1">
              <div 
                v-for="task in tasksByStatus(col.id)" 
                :key="task.id"
                draggable="true"
                @dragstart="onDragStart($event, task)"
                class="bg-white p-3.5 rounded-lg border border-slate-200/80 shadow-xs hover:border-emerald-600 transition cursor-grab"
              >
                <div class="flex justify-between items-start gap-2 mb-1">
                  <span class="text-[11px] font-semibold text-emerald-800">{{ task.assigne_a || 'Non assigné' }}</span>
                  <div class="flex gap-1 text-[11px] text-slate-400">
                    <button @click="openEditTask(task)" class="hover:text-emerald-700 cursor-pointer">✎</button>
                    <button @click="deleteTask(task.id)" class="hover:text-rose-600 cursor-pointer">✕</button>
                  </div>
                </div>
                <h4 class="font-bold text-slate-900 text-sm leading-snug">{{ task.titre }}</h4>
                <p v-if="task.description" class="text-xs text-slate-500 mt-1 line-clamp-2">{{ task.description }}</p>
              </div>

              <button @click="openCreateTask(col.id)" class="w-full py-2 text-xs font-medium text-slate-400 hover:text-emerald-700 transition cursor-pointer">
                + Ajouter
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- ================= 2. VUE BESOINS EN MATÉRIEL (LOGISTIQUE) ================= -->
      <div v-if="currentTab === 'materiel' && canAccessTab('materiel')" class="space-y-6">
        <div class="flex items-center justify-between flex-wrap gap-4">
          <div>
            <h2 class="text-xl font-bold text-slate-900">Demandes de Matériel</h2>
            <p class="text-xs text-slate-500">Suivi des matériels avec justification obligatoire</p>
          </div>

          <div class="flex items-center gap-4 text-xs font-bold">
            <button @click="exportToExcel('logistique')" class="text-emerald-700 hover:underline cursor-pointer">
              Exporter en Excel (.csv)
            </button>
            <button @click="openMaterialRequestModal()" class="px-3 py-2 bg-emerald-700 hover:bg-emerald-800 text-white rounded-lg transition cursor-pointer">
              + Demander du matériel
            </button>
          </div>
        </div>

        <div class="overflow-x-auto">
          <table class="w-full text-left border-collapse text-xs">
            <thead>
              <tr class="border-b border-slate-200 text-slate-400 uppercase tracking-wider font-semibold text-[10px]">
                <th class="py-3 px-2 w-28">Pôle</th>
                <th class="py-3 px-2 w-48">Article & Qté</th>
                <th class="py-3 px-2">Justification du besoin</th>
                <th class="py-3 px-2 w-28">Prix estimé</th>
                <th class="py-3 px-2 w-40">Avis Logistique</th>
                <th class="py-3 px-2 w-32">Avis Trésorerie</th>
                <th class="py-3 px-2 text-right w-28">Actions</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-slate-100">
              <tr v-for="item in filteredMaterialRequests" :key="item.id" class="hover:bg-slate-50/50 transition">
                <!-- Pôle & Demandeur -->
                <td class="py-3 px-2 align-top">
                  <span class="font-bold text-slate-800 uppercase text-[11px]">{{ item.pole }}</span>
                  <div class="text-[10px] text-slate-400">{{ item.demandeur }}</div>
                </td>

                <!-- Article & Qté -->
                <td class="py-3 px-2 align-top font-bold text-slate-900">
                  {{ item.article }} <span class="text-emerald-700 font-mono font-normal">x{{ item.quantite }}</span>
                </td>

                <!-- Justification du besoin (colonne dédiée sans encadré lourd) -->
                <td class="py-3 px-2 align-top text-slate-600 leading-relaxed pr-4">
                  <p class="whitespace-pre-wrap">{{ item.details }}</p>
                </td>

                <!-- Prix -->
                <td class="py-3 px-2 align-top whitespace-nowrap">
                  <span v-if="item.estimated_price" class="font-mono font-semibold text-slate-800">
                    {{ (Number(item.estimated_price) * item.quantite).toFixed(2) }} €
                  </span>
                  <span v-else class="text-slate-400 italic">Non renseigné</span>
                </td>

                <!-- Avis Logistique & Motif -->
                <td class="py-3 px-2 align-top">
                  <span :class="getStatusColorClass(item.avis_logistique || item.status)">
                    {{ getStatusLabel(item.avis_logistique || item.status) }}
                  </span>
                  <p v-if="item.reponse_logistique" class="text-slate-500 italic text-[11px] mt-0.5">
                    {{ item.reponse_logistique }}
                  </p>
                </td>

                <!-- Avis Trésorerie -->
                <td class="py-3 px-2 align-top">
                  <span :class="getStatusColorClass(item.avis_tresorerie)">
                    {{ getStatusLabel(item.avis_tresorerie) }}
                  </span>
                  <p v-if="item.reponse_tresorerie" class="text-slate-500 italic text-[11px] mt-0.5">
                    {{ item.reponse_tresorerie }}
                  </p>
                </td>

                <!-- Actions regroupées (Arbitrage + Modification + Suppression) -->
                <td class="py-3 px-2 text-right align-top whitespace-nowrap space-x-2">
                  <button 
                    v-if="isLogistiqueOrDirection"
                    @click="openDecisionModal(item, 'logistique')"
                    class="text-emerald-700 hover:underline font-bold cursor-pointer"
                    title="Prendre une décision logistique"
                  >
                    Arbitrer
                  </button>
                  <button 
                    @click="openMaterialRequestModal(item)" 
                    class="text-slate-500 hover:text-emerald-700 font-semibold cursor-pointer"
                    title="Modifier cette ligne"
                  >
                    Modifier
                  </button>
                  <button 
                    @click="deleteMaterialRequest(item.id)" 
                    class="text-slate-400 hover:text-rose-600 cursor-pointer"
                    title="Supprimer la demande"
                  >
                    ✕
                  </button>
                </td>
              </tr>

              <tr v-if="filteredMaterialRequests.length === 0">
                <td colspan="7" class="py-8 text-center text-slate-400 italic">
                  Aucune demande enregistrée.
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- ================= 3. VUE BUDGET & PÔLES (TRÉSORERIE) ================= -->
      <div v-if="currentTab === 'budget_tresorerie' && canAccessTab('budget_tresorerie')" class="space-y-6">
        <div class="flex items-center justify-between flex-wrap gap-4">
          <div>
            <h2 class="text-xl font-bold text-slate-900">Budget Matériel & Trésorerie</h2>
            <p class="text-xs text-slate-500">Contrôle des coûts et arbitrage financier par pôle</p>
          </div>

          <div class="flex items-center gap-4 text-xs font-bold">
            <button @click="exportToExcel('tresorerie')" class="text-emerald-700 hover:underline cursor-pointer">
              Exporter en Excel (.csv)
            </button>
          </div>
        </div>

        <!-- Synthèse par pôle épurée -->
        <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-6 gap-4 py-2 border-b border-slate-100">
          <div v-for="b in budgetSummaryByPole" :key="b.label">
            <span class="text-[11px] text-slate-400 uppercase font-semibold">{{ b.label }}</span>
            <div class="text-base font-bold text-emerald-800 font-mono">{{ b.total.toFixed(2) }} €</div>
            <div class="text-[10px] text-slate-500">Validé : {{ b.validatedTotal.toFixed(2) }} €</div>
          </div>
        </div>

        <!-- Tableau Financier Trésorerie avec Modification & Suppression Justifiée -->
        <div class="overflow-x-auto">
          <table class="w-full text-left border-collapse text-xs">
            <thead>
              <tr class="border-b border-slate-200 text-slate-400 uppercase tracking-wider font-semibold text-[10px]">
                <th class="py-3 px-2 w-28">Pôle</th>
                <th class="py-3 px-2 w-48">Article & Qté</th>
                <th class="py-3 px-2">Justification du besoin</th>
                <th class="py-3 px-2 w-28">Total Estimé</th>
                <th class="py-3 px-2 w-32">Avis Logistique</th>
                <th class="py-3 px-2 w-40">Décision Trésorerie</th>
                <th class="py-3 px-2 text-right w-36">Actions</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-slate-100">
              <tr v-for="item in materialRequests" :key="item.id" class="hover:bg-slate-50/50 transition">
                <td class="py-3 px-2 align-top font-bold text-slate-800 uppercase text-[11px]">{{ item.pole }}</td>
                <td class="py-3 px-2 align-top font-bold text-slate-900">
                  {{ item.article }} <span class="text-slate-500 font-normal">x{{ item.quantite }}</span>
                </td>
                <td class="py-3 px-2 align-top text-slate-600 leading-relaxed pr-4">{{ item.details }}</td>
                <td class="py-3 px-2 align-top font-mono font-bold text-emerald-800 whitespace-nowrap">
                  {{ item.estimated_price ? `${(Number(item.estimated_price) * item.quantite).toFixed(2)} €` : '0.00 €' }}
                </td>
                <td class="py-3 px-2 align-top">
                  <span :class="getStatusColorClass(item.avis_logistique || item.status)">
                    {{ getStatusLabel(item.avis_logistique || item.status) }}
                  </span>
                </td>
                <td class="py-3 px-2 align-top">
                  <span :class="getStatusColorClass(item.avis_tresorerie)">
                    {{ getStatusLabel(item.avis_tresorerie) }}
                  </span>
                  <p v-if="item.reponse_tresorerie" class="text-slate-500 italic text-[11px] mt-0.5">
                    {{ item.reponse_tresorerie }}
                  </p>
                </td>

                <!-- Actions Trésorerie : Arbitrer, Modifier, Supprimer avec justification -->
                <td class="py-3 px-2 text-right align-top whitespace-nowrap space-x-2">
                  <button 
                    @click="openDecisionModal(item, 'tresorerie')"
                    class="text-emerald-700 hover:underline font-bold cursor-pointer"
                  >
                    Arbitrer
                  </button>
                  <button 
                    @click="openMaterialRequestModal(item)" 
                    class="text-slate-500 hover:text-emerald-700 font-semibold cursor-pointer"
                  >
                    Modifier
                  </button>
                  <button 
                    @click="promptDeleteWithReason(item)" 
                    class="text-rose-600 hover:underline cursor-pointer"
                    title="Supprimer la dépense en justifiant"
                  >
                    Supprimer
                  </button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </div>

  <!-- MODALE : CRÉATION / ÉDITION D'UNE LIGNE MATÉRIEL -->
  <div v-if="showMaterialModal" class="fixed inset-0 z-50 bg-black/50 flex items-center justify-center p-4">
    <div class="bg-white rounded-xl max-w-md w-full p-6 shadow-xl relative">
      <button @click="showMaterialModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 text-lg cursor-pointer">✕</button>
      <h3 class="text-lg font-bold text-slate-900 mb-1">
        {{ editingMaterialRequest ? 'Modifier la demande' : 'Nouvelle demande de matériel' }}
      </h3>
      <p class="text-xs text-slate-500 mb-4">La justification de la commande est obligatoire.</p>

      <div class="space-y-3">
        <div v-if="isDirection">
          <label class="block text-xs font-semibold text-slate-600 mb-1">Pôle concerné</label>
          <select v-model="materialForm.pole" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600">
            <option v-for="p in allPoles" :key="p.id" :value="p.id">{{ p.label }}</option>
          </select>
        </div>

        <div>
          <label class="block text-xs font-semibold text-slate-600 mb-1">Article souhaité *</label>
          <input v-model="materialForm.article" type="text" placeholder="Ex. Barrières Vauban..." class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600" />
        </div>

        <div class="grid grid-cols-2 gap-3">
          <div>
            <label class="block text-xs font-semibold text-slate-600 mb-1">Quantité *</label>
            <input v-model.number="materialForm.quantite" type="number" min="1" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600 font-mono" />
          </div>
          <div>
            <label class="block text-xs font-semibold text-slate-600 mb-1">Prix unit. (€) <span class="font-normal text-slate-400">(option)</span></label>
            <input v-model.number="materialForm.estimated_price" type="number" step="0.01" min="0" placeholder="Ex. 15.00" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600 font-mono" />
          </div>
        </div>

        <div>
          <label class="block text-xs font-semibold text-slate-700 mb-1">Pourquoi demandez-vous cela ? *</label>
          <textarea v-model="materialForm.details" rows="3" required placeholder="Utilité, emplacement, horaire requis..." class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600"></textarea>
        </div>

        <button @click="submitMaterialRequest" class="w-full py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-lg text-xs transition cursor-pointer mt-2">
          {{ editingMaterialRequest ? 'Enregistrer les modifications' : 'Transmettre la demande' }}
        </button>
      </div>
    </div>
  </div>

  <!-- MODALE : ARBITRAGE (LOGISTIQUE OU TRÉSORERIE) -->
  <div v-if="showDecisionModal" class="fixed inset-0 z-50 bg-black/50 flex items-center justify-center p-4">
    <div class="bg-white rounded-xl max-w-md w-full p-6 shadow-xl relative">
      <button @click="showDecisionModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 text-lg cursor-pointer">✕</button>

      <h3 class="text-lg font-bold text-slate-900 mb-1">
        Décision {{ decisionType === 'logistique' ? 'Logistique' : 'Trésorerie' }}
      </h3>
      <p class="text-xs text-slate-500 mb-4">{{ decisionTargetItem?.article }} (x{{ decisionTargetItem?.quantite }}) — Pôle {{ decisionTargetItem?.pole }}</p>

      <div class="space-y-4">
        <div>
          <label class="block text-xs font-semibold text-slate-600 mb-1.5">Statut</label>
          <div class="grid grid-cols-3 gap-2">
            <button 
              type="button" 
              @click="decisionForm.status = 'valide'"
              :class="decisionForm.status === 'valide' ? 'bg-emerald-700 text-white font-bold' : 'bg-slate-100 text-slate-700'"
              class="py-2 text-xs rounded-lg transition cursor-pointer"
            >
              Validé
            </button>
            <button 
              type="button" 
              @click="decisionForm.status = 'refuse'"
              :class="decisionForm.status === 'refuse' ? 'bg-rose-700 text-white font-bold' : 'bg-slate-100 text-slate-700'"
              class="py-2 text-xs rounded-lg transition cursor-pointer"
            >
              Refusé
            </button>
            <button 
              type="button" 
              @click="decisionForm.status = 'en_attente'"
              :class="decisionForm.status === 'en_attente' ? 'bg-amber-600 text-white font-bold' : 'bg-slate-100 text-slate-700'"
              class="py-2 text-xs rounded-lg transition cursor-pointer"
            >
              En attente
            </button>
          </div>
        </div>

        <div>
          <label class="block text-xs font-semibold text-slate-700 mb-1">
            Motif / Justification <span v-if="decisionForm.status === 'refuse'" class="text-rose-600 font-bold">(obligatoire)</span>
          </label>
          <textarea 
            v-model="decisionForm.comment" 
            rows="3" 
            :placeholder="decisionForm.status === 'refuse' ? 'Raison impérative du refus...' : 'Commentaire optionnel...'" 
            class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600"
          ></textarea>
        </div>

        <button @click="saveDecision" class="w-full py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-lg text-xs transition cursor-pointer">
          Confirmer la décision
        </button>
      </div>
    </div>
  </div>

  <!-- MODALE : SUPPRESSION JUSTIFIÉE (TRÉSORERIE) -->
  <div v-if="showDeleteReasonModal" class="fixed inset-0 z-50 bg-black/50 flex items-center justify-center p-4">
    <div class="bg-white rounded-xl max-w-md w-full p-6 shadow-xl relative">
      <button @click="showDeleteReasonModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 text-lg cursor-pointer">✕</button>

      <h3 class="text-lg font-bold text-rose-600 mb-1">Supprimer cette demande</h3>
      <p class="text-xs text-slate-500 mb-4">
        Article : <strong>{{ itemToDelete?.article }}</strong> pour le pôle <strong>{{ itemToDelete?.pole }}</strong>
      </p>

      <div class="space-y-4">
        <div>
          <label class="block text-xs font-semibold text-slate-700 mb-1">
            Motif de suppression pour le pôle *
          </label>
          <textarea 
            v-model="deleteReasonInput" 
            rows="3" 
            placeholder="Ex : Hors budget, doublon avec le pôle logistique, article déjà réservé..." 
            class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-rose-600"
          ></textarea>
        </div>

        <button @click="confirmDeleteWithReason" class="w-full py-2.5 bg-rose-600 hover:bg-rose-700 text-white font-bold rounded-lg text-xs transition cursor-pointer">
          Confirmer la suppression
        </button>
      </div>
    </div>
  </div>

  <!-- MODALE : CHANGER DE COMPTE -->
  <div v-if="showSwitchAccountModal" class="fixed inset-0 z-50 bg-black/50 flex items-center justify-center p-4">
    <div class="bg-white rounded-2xl max-w-md w-full p-6 shadow-xl relative">
      <button @click="showSwitchAccountModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 cursor-pointer">✕</button>
      <h3 class="text-lg font-bold text-slate-900 mb-1">Changer de Compte</h3>
      <p class="text-xs text-slate-500 mb-4">Connectez-vous avec un autre compte ou pôle.</p>

      <form @submit.prevent="handleLogin" class="space-y-3">
        <div>
          <label class="block text-xs font-semibold text-slate-600 mb-1">E-mail</label>
          <input v-model="loginEmail" required type="email" placeholder="pole@pasdoubli.fr" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600" />
        </div>
        <div>
          <label class="block text-xs font-semibold text-slate-600 mb-1">Mot de passe</label>
          <input v-model="loginPassword" required type="password" placeholder="••••••••" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600" />
        </div>
        <p v-if="loginError" class="text-xs text-rose-600">{{ loginError }}</p>
        <button type="submit" class="w-full py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-lg text-xs transition cursor-pointer">
          Valider
        </button>
      </form>
    </div>
  </div>

  <!-- MODALE : POST-IT KANBAN -->
  <div v-if="showTaskModal" class="fixed inset-0 z-50 bg-black/50 flex items-center justify-center p-4">
    <div class="bg-white rounded-xl max-w-md w-full p-6 shadow-xl relative">
      <button @click="showTaskModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 cursor-pointer">✕</button>
      <h3 class="text-lg font-bold text-slate-900 mb-3">{{ editingTask ? 'Modifier le Post-it' : 'Nouveau Post-it' }}</h3>

      <div class="space-y-3">
        <div>
          <label class="block text-xs font-semibold text-slate-600 mb-1">Titre</label>
          <input v-model="taskForm.titre" type="text" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600" />
        </div>
        <div>
          <label class="block text-xs font-semibold text-slate-600 mb-1">En charge</label>
          <input v-model="taskForm.assigne_a" type="text" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600" />
        </div>
        <div>
          <label class="block text-xs font-semibold text-slate-600 mb-1">Colonne</label>
          <select v-model="taskForm.status" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600 bg-white">
            <option v-for="c in columns" :key="c.id" :value="c.id">{{ c.label }}</option>
          </select>
        </div>
        <div>
          <label class="block text-xs font-semibold text-slate-600 mb-1">Description</label>
          <textarea v-model="taskForm.description" rows="3" class="w-full px-3 py-2 border border-slate-300 rounded-lg text-xs outline-none focus:border-emerald-600"></textarea>
        </div>
        <button @click="saveTask" class="w-full py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-lg text-xs transition cursor-pointer mt-2">
          {{ editingTask ? 'Mettre à jour' : 'Créer' }}
        </button>
      </div>
    </div>
  </div>
</template>