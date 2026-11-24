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
  'lucierdg@outlook.fr'
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
  if (email === 'lucierdg@outlook.fr') return 'Vice-Présidente';
  const roleMap = {
    communication: 'Pôle Communication',
    sponsor: 'Pôle Sponsors',
    tresorerie: 'Trésorerie',
    animation: 'Pôle Animation',
    logistique: 'Pôle Logistique',
    parcours: 'Pôle Parcours',
    direction: 'Pôle Direction'
  };
  return roleMap[currentRole.value] || 'Membre Association';
});

// ================= PÔLES & GESTION DES ACCÈS ================= //
const allPoles = [
  { id: 'direction', label: 'Direction', icon: 'fa-crown' },
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

const selectedPole = ref('direction');

const canAccessTab = (tab) => {
  // 'gestion_site' est strictement réservé au Président et à la Vice-Présidente
  if (tab === 'gestion_site') return isDirection.value;

  if (isDirection.value) return true;
  const r = currentRole.value;
  if (tab === 'poles') return accessiblePoles.value.length > 0;
  if (tab === 'materiel') return r !== 'tresorerie';
  if (tab === 'budget_tresorerie') return r === 'tresorerie';
  if (tab === 'messages') return r === 'communication';
  if (tab === 'benevoles') return r === 'communication';
  if (tab === 'faq') return r === 'communication';
  if (tab === 'annuaire_sponsors') return r === 'sponsor';
  if (tab === 'besoins_sponsors') return r === 'sponsor';
  if (tab === 'cagnotte') return r === 'tresorerie';
  return false;
};

const currentTab = ref('poles');
const showMobileMenu = ref(false);

const selectAdminTab = (tab) => {
  currentTab.value = tab;
  showMobileMenu.value = false;
};

// Droits spécifiques
const isLogistiqueOrDirection = computed(() => isDirection.value || currentRole.value === 'logistique');
const isTresorerieOrDirection = computed(() => isDirection.value || currentRole.value === 'tresorerie');
const canDelegateToSponsors = computed(() => isDirection.value || currentRole.value === 'logistique' || currentRole.value === 'tresorerie');

// ================= DONNÉES DU DASHBOARD ================= //
const messages = ref([]);
const selectedMessage = ref(null);
const volunteers = ref([]);
const faqs = ref([]);
const sponsors = ref([]);
const fundraisingCurrent = ref(0);
const fundraisingGoal = ref(5000);
const fundraisingParticipants = ref(0);
const updatingFundraising = ref(false);

// ================= KANBAN ================= //
const tasks = ref([]);
const steps = ref([]);
const comments = ref([]);
const taskSaving = ref(false);

const columns = [
  { id: 'sujet', label: 'Sujet / Idée', bgHeader: 'text-emerald-900 border-emerald-900' },
  { id: 'a_faire', label: 'À faire', bgHeader: 'text-emerald-900 border-emerald-900' },
  { id: 'en_cours', label: 'En cours', bgHeader: 'text-emerald-900 border-emerald-900' },
  { id: 'fait', label: 'Fait', bgHeader: 'text-emerald-900 border-emerald-900' }
];

const postItColorOptions = [
  { id: 'border-slate-200', label: 'Standard', colorClass: 'border-slate-300 bg-slate-100' },
  { id: 'border-emerald-500', label: 'Émeraude', colorClass: 'border-emerald-500 bg-emerald-100' },
  { id: 'border-amber-500', label: 'Ambre', colorClass: 'border-amber-500 bg-amber-100' },
  { id: 'border-rose-500', label: 'Rose / Rouge', colorClass: 'border-rose-500 bg-rose-100' },
  { id: 'border-blue-500', label: 'Bleu', colorClass: 'border-blue-500 bg-blue-100' },
  { id: 'border-purple-500', label: 'Violet', colorClass: 'border-purple-500 bg-purple-100' },
  { id: 'border-orange-500', label: 'Orange', colorClass: 'border-orange-500 bg-orange-100' }
];

const showTaskModal = ref(false);
const editingTask = ref(null);
const taskForm = ref({
  titre: '',
  description: '',
  assigne_a: '',
  status: 'sujet',
  border_color: 'border-slate-200'
});

const showDetailModal = ref(false);
const currentTaskDetail = ref(null);
const newStepTitle = ref('');
const newComment = ref('');

// ================= MATÉRIEL, ARBITRAGE & FILTRES PAR COLONNE ================= //
const materialRequests = ref([]);
const showMaterialModal = ref(false);
const materialForm = ref({
  article: '',
  quantite: 1,
  estimated_price: null,
  details: '',
  pole: ''
});

const showEditMaterialModal = ref(false);
const editingMaterialId = ref(null);
const editMaterialForm = ref({
  article: '',
  quantite: 1,
  estimated_price: null,
  details: '',
  pole: ''
});

const showDecisionModal = ref(false);
const decisionTargetItem = ref(null);
const decisionType = ref('logistique');
const decisionForm = ref({
  status: 'valide',
  comment: ''
});

const decisionStatusList = [
  { id: 'en_attente', label: 'En attente', badge: 'bg-amber-100 text-amber-900 border-amber-200' },
  { id: 'valide', label: 'Validé', badge: 'bg-emerald-100 text-emerald-900 border-emerald-200' },
  { id: 'refuse', label: 'Refusé', badge: 'bg-rose-100 text-rose-900 border-rose-200' }
];

// Filtres par colonne — Tableau Logistique
const matColFilters = ref({
  pole: '',
  article: '',
  prix: '',
  avis_logistique: '',
  avis_tresorerie: '',
  statut_sponsor: ''
});

const resetMatFilters = () => {
  matColFilters.value = { pole: '', article: '', prix: '', avis_logistique: '', avis_tresorerie: '', statut_sponsor: '' };
};

const filteredMaterialRequests = computed(() => {
  const base = isLogistiqueOrDirection.value
    ? materialRequests.value
    : materialRequests.value.filter(m => m.pole === currentRole.value);

  const f = matColFilters.value;
  return base.filter(item => {
    if (f.pole && !(item.pole || '').toLowerCase().includes(f.pole.toLowerCase())) return false;
    if (f.article) {
      const q = f.article.toLowerCase();
      const matchArt = (item.article || '').toLowerCase().includes(q) ||
                       (item.details || '').toLowerCase().includes(q) ||
                       (item.demandeur || '').toLowerCase().includes(q);
      if (!matchArt) return false;
    }
    if (f.prix) {
      const priceStr = item.statut_sponsor === 'sponsor_trouve' ? '0' : String(item.estimated_price ?? '');
      if (!priceStr.includes(f.prix)) return false;
    }
    if (f.avis_logistique) {
      const st = item.statut_sponsor === 'sponsor_trouve' ? 'sponsorise' : (item.avis_logistique || item.status || 'en_attente');
      if (st !== f.avis_logistique) return false;
    }
    if (f.avis_tresorerie) {
      const st = item.statut_sponsor === 'sponsor_trouve' ? 'sponsorise' : (item.avis_tresorerie || 'en_attente');
      if (st !== f.avis_tresorerie) return false;
    }
    if (f.statut_sponsor) {
      const st = item.demande_sponsor ? (item.statut_sponsor || 'en_attente') : 'non_delegue';
      if (st !== f.statut_sponsor) return false;
    }
    return true;
  });
});

// Filtres par colonne — Tableau Trésorerie
const tresorColFilters = ref({
  pole: '',
  article: '',
  prix: '',
  avis_logistique: '',
  avis_tresorerie: '',
  statut_sponsor: ''
});

const resetTresorFilters = () => {
  tresorColFilters.value = { pole: '', article: '', prix: '', avis_logistique: '', avis_tresorerie: '', statut_sponsor: '' };
};

const filteredTresorerieRequests = computed(() => {
  const f = tresorColFilters.value;
  return materialRequests.value.filter(item => {
    if (f.pole && !(item.pole || '').toLowerCase().includes(f.pole.toLowerCase())) return false;
    if (f.article) {
      const q = f.article.toLowerCase();
      const matchArt = (item.article || '').toLowerCase().includes(q) ||
                       (item.details || '').toLowerCase().includes(q) ||
                       (item.demandeur || '').toLowerCase().includes(q);
      if (!matchArt) return false;
    }
    if (f.prix) {
      const priceStr = item.statut_sponsor === 'sponsor_trouve' ? '0' : String(item.estimated_price ?? '');
      if (!priceStr.includes(f.prix)) return false;
    }
    if (f.avis_logistique) {
      const st = item.statut_sponsor === 'sponsor_trouve' ? 'sponsorise' : (item.avis_logistique || item.status || 'en_attente');
      if (st !== f.avis_logistique) return false;
    }
    if (f.avis_tresorerie) {
      const st = item.statut_sponsor === 'sponsor_trouve' ? 'sponsorise' : (item.avis_tresorerie || 'en_attente');
      if (st !== f.avis_tresorerie) return false;
    }
    if (f.statut_sponsor) {
      const st = item.demande_sponsor ? (item.statut_sponsor || 'en_attente') : 'non_delegue';
      if (st !== f.statut_sponsor) return false;
    }
    return true;
  });
});

// Besoins délégués aux sponsors + Filtres par colonne
const sponsorMaterialRequests = computed(() => {
  return materialRequests.value.filter(m => m.demande_sponsor === true);
});

const sponsorReqColFilters = ref({
  pole: '',
  article: '',
  details: '',
  cout: '',
  statut: ''
});

const resetSponsorReqFilters = () => {
  sponsorReqColFilters.value = { pole: '', article: '', details: '', cout: '', statut: '' };
};

const filteredSponsorMaterialRequests = computed(() => {
  const f = sponsorReqColFilters.value;
  return sponsorMaterialRequests.value.filter(req => {
    if (f.pole && !(req.pole || '').toLowerCase().includes(f.pole.toLowerCase())) return false;
    if (f.article && !(req.article || '').toLowerCase().includes(f.article.toLowerCase())) return false;
    if (f.details && !(req.details || '').toLowerCase().includes(f.details.toLowerCase())) return false;
    if (f.cout) {
      const total = req.estimated_price ? (req.estimated_price * req.quantite).toFixed(2) : '';
      if (!total.includes(f.cout)) return false;
    }
    if (f.statut && req.statut_sponsor !== f.statut) return false;
    return true;
  });
});

const totalEstimatedBudget = computed(() => {
  return materialRequests.value.reduce((acc, item) => {
    if (item.statut_sponsor === 'sponsor_trouve') return acc;
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
    const isTrouve = item.statut_sponsor === 'sponsor_trouve';
    const cost = (item.estimated_price && !isTrouve) ? Number(item.estimated_price) * (item.quantite || 1) : 0;
    summary[pId].total += cost;
    summary[pId].count += 1;
    if (item.avis_tresorerie === 'valide' && !isTrouve) {
      summary[pId].validatedTotal += cost;
    }
  });

  return Object.values(summary).filter(s => s.count > 0);
});

// ================= ANNUAIRE PROSPECTION SPONSORS & FILTRES PAR COLONNE ================= //
const sponsorProspects = ref([]);
const showProspectModal = ref(false);
const editingProspect = ref(null);

const prospectForm = ref({
  nom_entreprise: '',
  nom_contact: '',
  contact_asso: '',
  telephone: '',
  email: '',
  adresse_postale: '',
  responsable: '',
  statut: 'a_contacter',
  notes: ''
});

const prospectStatusList = [
  { id: 'a_contacter', label: 'À contacter', badge: 'bg-amber-100 text-amber-900 border-amber-200' },
  { id: 'contacte', label: 'Contactée', badge: 'bg-blue-100 text-blue-900 border-blue-200' },
  { id: 'en_negociation', label: 'En négociation', badge: 'bg-purple-100 text-purple-900 border-purple-200' },
  { id: 'signe', label: 'Partenaire confirmé', badge: 'bg-emerald-100 text-emerald-900 border-emerald-200' },
  { id: 'refuse', label: 'Refusé', badge: 'bg-rose-100 text-rose-900 border-rose-200' }
];

const prospectColFilters = ref({
  entreprise: '',
  contact: '',
  contact_asso: '',
  coordonnees: '',
  responsable: '',
  statut: ''
});

const resetProspectFilters = () => {
  prospectColFilters.value = { entreprise: '', contact: '', contact_asso: '', coordonnees: '', responsable: '', statut: '' };
};

const filteredProspects = computed(() => {
  const f = prospectColFilters.value;
  return sponsorProspects.value.filter(item => {
    if (f.entreprise) {
      const q = f.entreprise.toLowerCase();
      if (!(item.nom_entreprise || '').toLowerCase().includes(q) &&
          !(item.adresse_postale || '').toLowerCase().includes(q) &&
          !(item.notes || '').toLowerCase().includes(q)) return false;
    }
    if (f.contact && !(item.nom_contact || '').toLowerCase().includes(f.contact.toLowerCase())) return false;
    if (f.contact_asso && !(item.contact_asso || '').toLowerCase().includes(f.contact_asso.toLowerCase())) return false;
    if (f.coordonnees) {
      const q = f.coordonnees.toLowerCase();
      if (!(item.telephone || '').toLowerCase().includes(q) && !(item.email || '').toLowerCase().includes(q)) return false;
    }
    if (f.responsable && !(item.responsable || '').toLowerCase().includes(f.responsable.toLowerCase())) return false;
    if (f.statut && item.statut !== f.statut) return false;
    return true;
  });
});

// ================= MODALES FAQ & SPONSORS OFFICIELS ================= //
const showFaqModal = ref(false);
const faqForm = ref({ question: '', reponse: '' });

const showSponsorModal = ref(false);
const editingSponsor = ref(null);
const sponsorForm = ref({ nom: '', description: '', logo_url: '' });

const availabilityLabels = {
  jour_j: 'Le jour J (dimanche)',
  veille: 'La veille (samedi)',
  les_deux: 'Tout le week-end',
  avant: 'En amont / Préparation'
};

const poleLabels = {
  direction: 'Direction',
  communication: 'Communication',
  animation: 'Animation',
  parcours: 'Parcours',
  infrastructures: 'Infrastructures',
  dossards: 'Retrait dossards',
  sponsor: 'Sponsors',
  logistique: 'Logistique'
};

// ================= AUTHENTIFICATION & DONNÉES ================= //
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

    if (prof) {
      userProfile.value = prof;
    } else {
      const em = user.value.email.toLowerCase();
      if (em.includes('com')) userProfile.value = { role: 'communication' };
      else if (em.includes('sponsor')) userProfile.value = { role: 'sponsor' };
      else if (em.includes('tresor')) userProfile.value = { role: 'tresorerie' };
      else if (em.includes('logist')) userProfile.value = { role: 'logistique' };
      else if (em.includes('anim')) userProfile.value = { role: 'animation' };
      else if (em.includes('parcours')) userProfile.value = { role: 'parcours' };
    }
  }

  if (accessiblePoles.value.length > 0) {
    selectedPole.value = accessiblePoles.value[0].id;
  }

  currentTab.value = 'poles';
  await loadPermittedData();
};

const loadPermittedData = async () => {
  loadPoleData();
  fetchMaterialRequests();

  // Données exclusives Président / Vice-Présidente (Gestion Site)
  if (isDirection.value) {
    fetchProgramAndSettings();
    fetchVillageItems();
    const { data: spData } = await supabase.from('sponsors').select('*').order('id', { ascending: true });
    if (spData) sponsors.value = spData;
  }

  if (canAccessTab('annuaire_sponsors') || canAccessTab('besoins_sponsors')) {
    fetchSponsorProspects();
  }

  if (canAccessTab('messages')) {
    const { data } = await supabase.from('messages').select('*').order('id', { ascending: false });
    if (data) messages.value = data;
  }

  if (canAccessTab('benevoles')) {
    const { data } = await supabase.from('volunteers').select('*').order('id', { ascending: false });
    if (data) volunteers.value = data;
  }

  if (canAccessTab('faq')) {
    const { data } = await supabase.from('faqs').select('*').order('id', { ascending: false });
    if (data) faqs.value = data;
  }

  if (canAccessTab('cagnotte')) {
    const { data } = await supabase.from('fundraising').select('*').eq('id', 1).maybeSingle();
    if (data) {
      fundraisingCurrent.value = data.current_amount;
      fundraisingGoal.value = data.goal_amount;
      fundraisingParticipants.value = data.participants_count || 0;
    }
  }
};

const loadPoleData = async () => {
  const { data: tasksData } = await supabase.from('pole_tasks').select('*').order('id', { ascending: true });
  if (tasksData) tasks.value = tasksData;

  const { data: stepsData } = await supabase.from('pole_task_steps').select('*').order('id', { ascending: true });
  if (stepsData) steps.value = stepsData;

  const { data: commentsData } = await supabase.from('pole_task_comments').select('*').order('created_at', { ascending: true });
  if (commentsData) comments.value = commentsData;
};

// ================= GESTION DU KANBAN ================= //
const tasksByStatus = (status) => {
  return tasks.value.filter(t => t.pole === selectedPole.value && t.status === status);
};

const getTaskProgress = (taskId) => {
  const taskSteps = steps.value.filter(s => s.task_id === taskId);
  if (taskSteps.length === 0) return null;
  const done = taskSteps.filter(s => s.completed).length;
  return {
    done,
    total: taskSteps.length,
    percent: Math.round((done / taskSteps.length) * 100)
  };
};

const openCreateTask = (status = 'sujet') => {
  editingTask.value = null;
  taskForm.value = {
    titre: '',
    description: '',
    assigne_a: '',
    status,
    border_color: 'border-slate-200'
  };
  showTaskModal.value = true;
};

const openEditTask = (task) => {
  editingTask.value = task;
  taskForm.value = {
    titre: task.titre,
    description: task.description || '',
    assigne_a: task.assigne_a || '',
    status: task.status,
    border_color: task.border_color || 'border-slate-200'
  };
  showTaskModal.value = true;
};

const saveTask = async () => {
  if (!taskForm.value.titre || !taskForm.value.titre.trim()) {
    triggerToast('Le titre du post-it est requis.');
    return;
  }

  taskSaving.value = true;
  try {
    const payload = {
      titre: taskForm.value.titre.trim(),
      description: taskForm.value.description ? taskForm.value.description.trim() : null,
      assigne_a: taskForm.value.assigne_a ? taskForm.value.assigne_a.trim() : null,
      status: taskForm.value.status,
      border_color: taskForm.value.border_color || 'border-slate-200'
    };

    if (editingTask.value) {
      const { data, error } = await supabase
        .from('pole_tasks')
        .update(payload)
        .eq('id', editingTask.value.id)
        .select();

      if (error) throw error;

      const idx = tasks.value.findIndex(t => t.id === editingTask.value.id);
      if (idx !== -1 && data && data.length > 0) {
        tasks.value[idx] = data[0];
      }
      if (currentTaskDetail.value?.id === editingTask.value.id && data && data.length > 0) {
        currentTaskDetail.value = data[0];
      }
      showTaskModal.value = false;
      triggerToast('Post-it mis à jour.');
    } else {
      const insertPayload = {
        ...payload,
        pole: selectedPole.value
      };

      const { data, error } = await supabase
        .from('pole_tasks')
        .insert([insertPayload])
        .select();

      if (error) throw error;

      if (data && data.length > 0) {
        tasks.value.push(data[0]);
      }
      showTaskModal.value = false;
      triggerToast('Post-it épinglé avec succès !');
    }
  } catch (err) {
    console.error('Erreur saveTask :', err);
    triggerToast('Erreur lors de l’enregistrement du post-it.');
  } finally {
    taskSaving.value = false;
  }
};

const deleteTask = async (taskId) => {
  const { error } = await supabase.from('pole_tasks').delete().eq('id', taskId);
  if (!error) {
    tasks.value = tasks.value.filter(t => t.id !== taskId);
    steps.value = steps.value.filter(s => s.task_id !== taskId);
    comments.value = comments.value.filter(c => c.task_id !== taskId);
    if (currentTaskDetail.value?.id === taskId) showDetailModal.value = false;
    triggerToast('Post-it supprimé.');
  }
};

const onDragStart = (event, task) => {
  event.dataTransfer.setData('taskId', task.id.toString());
};

const onDrop = async (event, newStatus) => {
  const taskId = event.dataTransfer.getData('taskId');
  if (!taskId) return;

  const task = tasks.value.find(t => t.id == taskId);
  if (task && task.status !== newStatus) {
    task.status = newStatus;
    await supabase.from('pole_tasks').update({ status: newStatus }).eq('id', taskId);
    triggerToast(`Post-it déplacé vers "${columns.find(c => c.id === newStatus).label}"`);
  }
};

// ================= ÉTAPES POST-IT ================= //
const openDetail = (task) => {
  currentTaskDetail.value = task;
  newStepTitle.value = '';
  newComment.value = '';
  showDetailModal.value = true;
};

const currentTaskSteps = computed(() => {
  if (!currentTaskDetail.value) return [];
  return steps.value.filter(s => s.task_id === currentTaskDetail.value.id);
});

const currentTaskComments = computed(() => {
  if (!currentTaskDetail.value) return [];
  return comments.value.filter(c => c.task_id === currentTaskDetail.value.id);
});

const addStep = async () => {
  if (!newStepTitle.value.trim() || !currentTaskDetail.value) return;

  const payload = {
    task_id: currentTaskDetail.value.id,
    titre: newStepTitle.value.trim(),
    completed: false
  };

  const { data, error } = await supabase.from('pole_task_steps').insert([payload]).select();
  if (!error && data) {
    steps.value.push(data[0]);
    newStepTitle.value = '';
    triggerToast('Étape ajoutée au post-it !');
  }
};

const toggleStep = async (step) => {
  step.completed = !step.completed;
  await supabase.from('pole_task_steps').update({ completed: step.completed }).eq('id', step.id);
};

const deleteStep = async (stepId) => {
  await supabase.from('pole_task_steps').delete().eq('id', stepId);
  steps.value = steps.value.filter(s => s.id !== stepId);
};

const addComment = async () => {
  if (!newComment.value.trim() || !currentTaskDetail.value) return;

  const payload = {
    task_id: currentTaskDetail.value.id,
    auteur: userRoleTitle.value,
    commentaire: newComment.value.trim()
  };

  const { data, error } = await supabase.from('pole_task_comments').insert([payload]).select();
  if (!error && data) {
    comments.value.push(data[0]);
    newComment.value = '';
    triggerToast('Commentaire ajouté.');
  }
};

// ================= GESTION MATÉRIEL, ARBITRAGE & DÉLÉGATION ================= //
const fetchMaterialRequests = async () => {
  const { data } = await supabase
    .from('material_requests')
    .select('*')
    .order('created_at', { ascending: false });
  if (data) materialRequests.value = data;
};

const openMaterialRequestModal = () => {
  materialForm.value = {
    article: '',
    quantite: 1,
    estimated_price: null,
    details: '',
    pole: isDirection.value ? selectedPole.value : currentRole.value
  };
  showMaterialModal.value = true;
};

const submitMaterialRequest = async () => {
  if (!materialForm.value.article || materialForm.value.quantite < 1) {
    triggerToast('Veuillez renseigner un article et une quantité valide.');
    return;
  }

  if (!materialForm.value.details || !materialForm.value.details.trim()) {
    triggerToast('La justification du besoin est obligatoire !');
    return;
  }

  const payload = {
    pole: materialForm.value.pole || currentRole.value,
    demandeur: userRoleTitle.value,
    article: materialForm.value.article.trim(),
    quantite: materialForm.value.quantite,
    estimated_price: (materialForm.value.estimated_price !== null && materialForm.value.estimated_price !== '') 
      ? Number(materialForm.value.estimated_price) 
      : null,
    details: materialForm.value.details.trim(),
    status: 'en_attente',
    avis_logistique: 'en_attente',
    avis_tresorerie: 'en_attente',
    demande_sponsor: false,
    statut_sponsor: 'non_demande'
  };

  const { data, error } = await supabase.from('material_requests').insert([payload]).select();
  if (!error && data) {
    materialRequests.value.unshift(data[0]);
    showMaterialModal.value = false;
    triggerToast('Demande enregistrée avec succès !');
  } else if (error) {
    triggerToast('Erreur lors de l’envoi de la demande.');
    console.error(error);
  }
};

const askSponsorsForMaterial = async (item) => {
  if (!canDelegateToSponsors.value) {
    triggerToast('Seule la logistique, la trésorerie ou la direction peuvent déléguer.');
    return;
  }

  if (item.demande_sponsor && item.statut_sponsor !== 'non_finance' && item.statut_sponsor !== 'refuse') {
    triggerToast('Déjà transmis au pôle sponsors.');
    return;
  }

  const { error } = await supabase
    .from('material_requests')
    .update({ demande_sponsor: true, statut_sponsor: 'en_attente' })
    .eq('id', item.id);

  if (!error) {
    item.demande_sponsor = true;
    item.statut_sponsor = 'en_attente';
    triggerToast('Besoin transmis au Pôle Sponsors !');
  } else {
    triggerToast('Erreur lors de la transmission.');
  }
};

const handleSponsorResponse = async (item, decision) => {
  if (decision === 'refuse') {
    const { error } = await supabase
      .from('material_requests')
      .update({ statut_sponsor: 'refuse' })
      .eq('id', item.id);

    if (!error) {
      item.statut_sponsor = 'refuse';
      triggerToast('Demande refusée par le pôle sponsors.');
    }
    return;
  }

  if (decision === 'accepte_recherche') {
    const taskTitle = `Recherche sponsor : ${item.article} (x${item.quantite})`;
    const taskDesc = `Matériel : ${item.article} (x${item.quantite}) demandé par ${item.pole}.
Justification : ${item.details}
Prix estimé : ${item.estimated_price ? `${item.estimated_price} €/u` : 'Inconnu'}`;

    const taskPayload = {
      pole: 'sponsor',
      titre: taskTitle,
      description: taskDesc,
      status: 'a_faire',
      assigne_a: 'Équipe Sponsors',
      border_color: 'border-amber-500'
    };

    const { data: createdTask, error: taskError } = await supabase
      .from('pole_tasks')
      .insert([taskPayload])
      .select();

    if (taskError) {
      triggerToast('Erreur lors de la création du post-it.');
      return;
    }

    const { error: reqError } = await supabase
      .from('material_requests')
      .update({ statut_sponsor: 'en_recherche' })
      .eq('id', item.id);

    if (!reqError) {
      item.statut_sponsor = 'en_recherche';
      if (createdTask && createdTask.length > 0) {
        tasks.value.push(createdTask[0]);
      }
      triggerToast('Recherche acceptée ! Post-it créé dans "À faire".');
    }
    return;
  }

  if (decision === 'trouve') {
    const { error } = await supabase
      .from('material_requests')
      .update({
        statut_sponsor: 'sponsor_trouve',
        avis_logistique: 'pris_en_charge',
        avis_tresorerie: 'pris_en_charge'
      })
      .eq('id', item.id);

    if (!error) {
      item.statut_sponsor = 'sponsor_trouve';
      item.avis_logistique = 'pris_en_charge';
      item.avis_tresorerie = 'pris_en_charge';
      triggerToast('Sponsor validé ! Matériel financé à 0 €.');
    } else {
      triggerToast('Erreur lors de la validation.');
    }
    return;
  }

  if (decision === 'abandon') {
    if (!confirm('Déclarer que ce matériel n\'a pas pu être sponsorisé ? La logistique et la trésorerie devront reprendre la main et valider l\'achat.')) {
      return;
    }

    const { error } = await supabase
      .from('material_requests')
      .update({
        statut_sponsor: 'non_finance',
        avis_logistique: 'en_attente',
        avis_tresorerie: 'en_attente'
      })
      .eq('id', item.id);

    if (!error) {
      item.statut_sponsor = 'non_finance';
      item.avis_logistique = 'en_attente';
      item.avis_tresorerie = 'en_attente';
      triggerToast('Marqué non sponsorisé. Renvoi à la logistique et trésorerie.');
    } else {
      triggerToast('Erreur lors de la mise à jour.');
    }
  }
};

const openEditMaterialModal = (item) => {
  editingMaterialId.value = item.id;
  editMaterialForm.value = {
    pole: item.pole,
    article: item.article,
    quantite: item.quantite || 1,
    estimated_price: item.estimated_price !== null && item.estimated_price !== undefined ? item.estimated_price : null,
    details: item.details || ''
  };
  showEditMaterialModal.value = true;
};

const saveEditedMaterial = async () => {
  if (!editMaterialForm.value.article || editMaterialForm.value.quantite < 1) {
    triggerToast('Article et quantité valides requis.');
    return;
  }

  if (!editMaterialForm.value.details || !editMaterialForm.value.details.trim()) {
    triggerToast('La justification du besoin est obligatoire !');
    return;
  }

  const payload = {
    pole: editMaterialForm.value.pole,
    article: editMaterialForm.value.article.trim(),
    quantite: Number(editMaterialForm.value.quantite),
    estimated_price: (editMaterialForm.value.estimated_price !== null && editMaterialForm.value.estimated_price !== '')
      ? Number(editMaterialForm.value.estimated_price)
      : null,
    details: editMaterialForm.value.details.trim()
  };

  const { error } = await supabase
    .from('material_requests')
    .update(payload)
    .eq('id', editingMaterialId.value);

  if (!error) {
    const idx = materialRequests.value.findIndex(m => m.id === editingMaterialId.value);
    if (idx !== -1) {
      materialRequests.value[idx] = { ...materialRequests.value[idx], ...payload };
    }
    showEditMaterialModal.value = false;
    triggerToast('Demande mise à jour avec succès !');
  } else {
    triggerToast('Erreur lors de la mise à jour.');
    console.error(error);
  }
};

const openDecisionModal = (item, type) => {
  if (item.statut_sponsor === 'sponsor_trouve') {
    triggerToast('Ce besoin est pris en charge par un sponsor (aucun arbitrage requis).');
    return;
  }

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
    triggerToast('Veuillez justifier obligatoirement le refus.');
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
    triggerToast(`Avis ${decisionType.value === 'logistique' ? 'logistique' : 'trésorerie'} enregistré !`);
  } else {
    triggerToast('Erreur lors de la mise à jour.');
    console.error(error);
  }
};

const deleteMaterialRequest = async (id) => {
  if (!confirm('Supprimer définitivement cette demande ?')) return;
  const { error } = await supabase
    .from('material_requests')
    .delete()
    .eq('id', id);

  if (!error) {
    materialRequests.value = materialRequests.value.filter(m => m.id !== id);
    triggerToast('Demande supprimée.');
  }
};

// ================= GESTION ANNUAIRE PROSPECTION ================= //
const fetchSponsorProspects = async () => {
  const { data, error } = await supabase
    .from('sponsor_prospects')
    .select('*')
    .order('created_at', { ascending: false });
  if (data && !error) sponsorProspects.value = data;
};

const openCreateProspect = () => {
  editingProspect.value = null;
  prospectForm.value = {
    nom_entreprise: '',
    nom_contact: '',
    contact_asso: '',
    telephone: '',
    email: '',
    adresse_postale: '',
    responsable: userRoleTitle.value,
    statut: 'a_contacter',
    notes: ''
  };
  showProspectModal.value = true;
};

const openEditProspect = (item) => {
  editingProspect.value = item;
  prospectForm.value = {
    nom_entreprise: item.nom_entreprise || '',
    nom_contact: item.nom_contact || '',
    contact_asso: item.contact_asso || '',
    telephone: item.telephone || '',
    email: item.email || '',
    adresse_postale: item.adresse_postale || '',
    responsable: item.responsable || '',
    statut: item.statut || 'a_contacter',
    notes: item.notes || ''
  };
  showProspectModal.value = true;
};

const saveProspect = async () => {
  if (!prospectForm.value.nom_entreprise || !prospectForm.value.nom_entreprise.trim()) {
    triggerToast("Le nom de l'entreprise est obligatoire.");
    return;
  }

  const payload = {
    nom_entreprise: prospectForm.value.nom_entreprise.trim(),
    nom_contact: prospectForm.value.nom_contact?.trim() || null,
    contact_asso: prospectForm.value.contact_asso?.trim() || null,
    telephone: prospectForm.value.telephone?.trim() || null,
    email: prospectForm.value.email?.trim() || null,
    adresse_postale: prospectForm.value.adresse_postale?.trim() || null,
    responsable: prospectForm.value.responsable?.trim() || null,
    statut: prospectForm.value.statut,
    notes: prospectForm.value.notes?.trim() || null
  };

  if (editingProspect.value) {
    const { error } = await supabase
      .from('sponsor_prospects')
      .update(payload)
      .eq('id', editingProspect.value.id);

    if (!error) {
      const idx = sponsorProspects.value.findIndex(p => p.id === editingProspect.value.id);
      if (idx !== -1) sponsorProspects.value[idx] = { ...sponsorProspects.value[idx], ...payload };
      showProspectModal.value = false;
      triggerToast('Entreprise mise à jour !');
    } else {
      triggerToast('Erreur lors de la mise à jour.');
    }
  } else {
    const { data, error } = await supabase
      .from('sponsor_prospects')
      .insert([payload])
      .select();

    if (!error && data) {
      sponsorProspects.value.unshift(data[0]);
      showProspectModal.value = false;
      triggerToast('Entreprise ajoutée à l’annuaire !');
    } else {
      triggerToast('Erreur lors de l’ajout.');
    }
  }
};

const deleteProspect = async (id) => {
  if (!confirm('Supprimer définitivement ce contact d’entreprise ?')) return;
  const { error } = await supabase.from('sponsor_prospects').delete().eq('id', id);
  if (!error) {
    sponsorProspects.value = sponsorProspects.value.filter(p => p.id !== id);
    triggerToast('Contact supprimé.');
  }
};

// ================= EXPORTS EXCEL ================= //
const exportToExcel = (type = 'logistique') => {
  let csvContent = '\uFEFF';
  let filename = '';

  if (type === 'logistique') {
    filename = `demandes_materiel_logistique_${new Date().toISOString().slice(0, 10)}.csv`;
    csvContent += 'Pôle;Article / Commande;Quantité;Justification du Besoin;Demandeur;Prix Estimé Unitaire (€);Total Estimé (€);Avis Logistique;Motif Logistique;Avis Trésorerie;Statut Sponsor;Date\n';

    filteredMaterialRequests.value.forEach(req => {
      const pole = `"${(req.pole || '').replace(/"/g, '""')}"`;
      const article = `"${(req.article || '').replace(/"/g, '""')}"`;
      const qte = req.quantite || 1;
      const details = `"${(req.details || '').replace(/"/g, '""')}"`;
      const demandeur = `"${(req.demandeur || '').replace(/"/g, '""')}"`;
      const prix = req.estimated_price ? Number(req.estimated_price).toFixed(2).replace('.', ',') : '';
      const total = req.estimated_price ? (Number(req.estimated_price) * qte).toFixed(2).replace('.', ',') : '';
      const avisLog = `"${(req.statut_sponsor === 'sponsor_trouve' ? 'Sponsorisé (0 €)' : (decisionStatusList.find(s => s.id === (req.avis_logistique || req.status))?.label || '')).replace(/"/g, '""')}"`;
      const motifLog = `"${(req.reponse_logistique || '').replace(/"/g, '""')}"`;
      const avisTresor = `"${(req.statut_sponsor === 'sponsor_trouve' ? 'Sponsorisé (0 €)' : (decisionStatusList.find(s => s.id === req.avis_tresorerie)?.label || 'En attente')).replace(/"/g, '""')}"`;
      const statutSponsor = req.demande_sponsor ? (req.statut_sponsor === 'sponsor_trouve' ? 'Sponsorisé (Validé)' : (req.statut_sponsor === 'en_recherche' ? 'Recherche en cours' : (req.statut_sponsor === 'non_finance' ? 'Non sponsorisé' : (req.statut_sponsor === 'refuse' ? 'Refusé Sponsor' : 'En attente')))) : 'Non délégué';
      const date = req.created_at ? new Date(req.created_at).toLocaleDateString('fr-FR') : '';

      csvContent += `${pole};${article};${qte};${details};${demandeur};${prix};${total};${avisLog};${motifLog};${avisTresor};${statutSponsor};${date}\n`;
    });
  } else if (type === 'tresorerie') {
    filename = `budget_materiel_tresorerie_${new Date().toISOString().slice(0, 10)}.csv`;
    csvContent += 'Pôle;Article;Quantité;Prix Unitaire (€);Montant Total Estimé (€);Avis Trésorerie;Justification Trésorerie;Avis Logistique;Statut Sponsor;Demandeur;Date\n';

    filteredTresorerieRequests.value.forEach(req => {
      const pole = `"${(req.pole || '').replace(/"/g, '""')}"`;
      const article = `"${(req.article || '').replace(/"/g, '""')}"`;
      const qte = req.quantite || 1;
      const isSponsored = req.statut_sponsor === 'sponsor_trouve';
      const prix = isSponsored ? '0,00' : (req.estimated_price ? Number(req.estimated_price).toFixed(2).replace('.', ',') : '0,00');
      const total = isSponsored ? '0,00' : (req.estimated_price ? (Number(req.estimated_price) * qte).toFixed(2).replace('.', ',') : '0,00');
      const avisTresor = `"${(isSponsored ? 'Sponsorisé (0 €)' : (decisionStatusList.find(s => s.id === req.avis_tresorerie)?.label || 'En attente')).replace(/"/g, '""')}"`;
      const motifTresor = `"${(req.reponse_tresorerie || '').replace(/"/g, '""')}"`;
      const avisLog = `"${(isSponsored ? 'Sponsorisé (0 €)' : (decisionStatusList.find(s => s.id === (req.avis_logistique || req.status))?.label || '')).replace(/"/g, '""')}"`;
      const statutSponsor = req.demande_sponsor ? (req.statut_sponsor === 'sponsor_trouve' ? 'Sponsorisé' : (req.statut_sponsor === 'en_recherche' ? 'En recherche' : (req.statut_sponsor === 'non_finance' ? 'Non sponsorisé' : req.statut_sponsor))) : 'Non';
      const demandeur = `"${(req.demandeur || '').replace(/"/g, '""')}"`;
      const date = req.created_at ? new Date(req.created_at).toLocaleDateString('fr-FR') : '';

      csvContent += `${pole};${article};${qte};${prix};${total};${avisTresor};${motifTresor};${avisLog};${statutSponsor};${demandeur};${date}\n`;
    });
  } else if (type === 'sponsor_besoins') {
    filename = `suivi_materiel_sponsoring_${new Date().toISOString().slice(0, 10)}.csv`;
    csvContent += 'Pôle Demandeur;Matériel;Quantité;Coût Indicatif Unitaire (€);Coût Total Économisé (€);Justification;Statut Sponsoring;Date Demande\n';

    filteredSponsorMaterialRequests.value.forEach(req => {
      const pole = `"${(req.pole || '').replace(/"/g, '""')}"`;
      const article = `"${(req.article || '').replace(/"/g, '""')}"`;
      const qte = req.quantite || 1;
      const prix = req.estimated_price ? Number(req.estimated_price).toFixed(2).replace('.', ',') : '';
      const total = req.estimated_price ? (Number(req.estimated_price) * qte).toFixed(2).replace('.', ',') : '';
      const details = `"${(req.details || '').replace(/"/g, '""')}"`;
      const statut = req.statut_sponsor === 'sponsor_trouve' ? 'Sponsor trouvé (Validé)' : (req.statut_sponsor === 'en_recherche' ? 'Recherche en cours' : (req.statut_sponsor === 'non_finance' ? 'Non sponsorisé (Abandonné)' : (req.statut_sponsor === 'refuse' ? 'Refusé' : 'Nouveau besoin')));
      const date = req.created_at ? new Date(req.created_at).toLocaleDateString('fr-FR') : '';

      csvContent += `${pole};${article};${qte};${prix};${total};${details};"${statut}";${date}\n`;
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

// ================= ACTIONS MESSAGES / BÉNÉVOLES / FAQ / SPONSORS / CAGNOTTE ================= //
const selectMsg = async (msg) => {
  selectedMessage.value = msg;
  if (!msg.read) {
    msg.read = true;
    await supabase.from('messages').update({ read: true }).eq('id', msg.id);
  }
};

const deleteMsg = async (id) => {
  await supabase.from('messages').delete().eq('id', id);
  messages.value = messages.value.filter(m => m.id !== id);
  if (selectedMessage.value?.id === id) selectedMessage.value = null;
  triggerToast('Message supprimé.');
};

const openFaqModalFromMessage = (msg) => {
  faqForm.value = { question: msg.message, reponse: '' };
  showFaqModal.value = true;
};

const openManualFaqModal = () => {
  faqForm.value = { question: '', reponse: '' };
  showFaqModal.value = true;
};

const handleSaveFaq = async () => {
  if (!faqForm.value.question || !faqForm.value.reponse) return;
  const { data, error } = await supabase.from('faqs').insert([faqForm.value]).select();
  if (!error && data) {
    faqs.value.unshift(data[0]);
    showFaqModal.value = false;
    faqForm.value = { question: '', reponse: '' };
    triggerToast('Question publiée sur la FAQ !');
  }
};

const deleteFaq = async (id) => {
  await supabase.from('faqs').delete().eq('id', id);
  faqs.value = faqs.value.filter(f => f.id !== id);
  triggerToast('Question supprimée.');
};

const deleteVolunteer = async (id) => {
  await supabase.from('volunteers').delete().eq('id', id);
  volunteers.value = volunteers.value.filter(v => v.id !== id);
  triggerToast('Candidature retirée.');
};

const openCreateSponsor = () => {
  editingSponsor.value = null;
  sponsorForm.value = { nom: '', description: '', logo_url: '' };
  showSponsorModal.value = true;
};

const openEditSponsor = (s) => {
  editingSponsor.value = s;
  sponsorForm.value = { nom: s.nom, description: s.description || '', logo_url: s.logo_url || '' };
  showSponsorModal.value = true;
};

const handleSaveSponsor = async () => {
  if (!sponsorForm.value.nom || !sponsorForm.value.nom.trim()) {
    triggerToast('Le nom du partenaire est obligatoire.');
    return;
  }

  const payload = {
    nom: sponsorForm.value.nom.trim(),
    description: sponsorForm.value.description ? sponsorForm.value.description.trim() : null,
    logo_url: sponsorForm.value.logo_url ? sponsorForm.value.logo_url.trim() : null
  };

  try {
    if (editingSponsor.value) {
      const { data, error } = await supabase
        .from('sponsors')
        .update(payload)
        .eq('id', editingSponsor.value.id)
        .select();

      if (error) throw error;

      const idx = sponsors.value.findIndex(s => s.id === editingSponsor.value.id);
      if (idx !== -1 && data && data.length > 0) {
        sponsors.value[idx] = data[0];
      }
      showSponsorModal.value = false;
      triggerToast('Sponsor mis à jour.');
    } else {
      const { data, error } = await supabase
        .from('sponsors')
        .insert([payload])
        .select();

      if (error) throw error;

      if (data && data.length > 0) {
        sponsors.value.push(data[0]);
      }
      showSponsorModal.value = false;
      triggerToast('Sponsor officiel ajouté.');
    }
  } catch (err) {
    console.error('Erreur Supabase sponsors :', err);
    triggerToast(`Erreur : ${err.message || 'Impossible d\'enregistrer le sponsor'}`);
  }
};

const deleteSponsor = async (id) => {
  const { error } = await supabase.from('sponsors').delete().eq('id', id);
  if (!error) {
    sponsors.value = sponsors.value.filter(s => s.id !== id);
    triggerToast('Sponsor supprimé.');
  }
};

const handleSaveFundraising = async () => {
  updatingFundraising.value = true;
  const { error } = await supabase
    .from('fundraising')
    .update({
      current_amount: Number(fundraisingCurrent.value) || 0,
      goal_amount: Number(fundraisingGoal.value) || 5000,
      participants_count: Number(fundraisingParticipants.value) || 0,
      updated_at: new Date().toISOString()
    })
    .eq('id', 1);

  updatingFundraising.value = false;
  if (!error) triggerToast('Cagnotte et inscrits mis à jour en direct !');
};

// ================= GESTION PROGRAMME & LIEN HELLOASSO (GESTION SITE) ================= //
const programItems = ref([]);
const programPeriodFilter = ref('matin');
const showProgramModal = ref(false);
const editingProgramItem = ref(null);
const helloAssoSetting = ref('https://www.helloasso.com/associations/pas-d-oubli');

const programForm = ref({
  periode: 'matin',
  time_label: '',
  tag: '',
  title: '',
  description: ''
});

const fetchProgramAndSettings = async () => {
  const { data: progData } = await supabase
    .from('program_schedule')
    .select('*')
    .order('sort_order', { ascending: true })
    .order('id', { ascending: true });
  if (progData) programItems.value = progData;

  const { data: settingData } = await supabase
    .from('site_settings')
    .select('value')
    .eq('key', 'helloasso_url')
    .maybeSingle();
  if (settingData?.value) helloAssoSetting.value = settingData.value;
};

const filteredProgramItems = computed(() => {
  return programItems.value
    .filter(p => p.periode === programPeriodFilter.value)
    .sort((a, b) => (a.sort_order ?? 0) - (b.sort_order ?? 0));
});

const saveHelloAssoUrl = async () => {
  const { error } = await supabase
    .from('site_settings')
    .upsert({ key: 'helloasso_url', value: helloAssoSetting.value.trim() });
  if (!error) triggerToast('Lien HelloAsso mis à jour sur le site !');
};

const openCreateProgram = () => {
  editingProgramItem.value = null;
  programForm.value = {
    periode: programPeriodFilter.value,
    time_label: '',
    tag: '',
    title: '',
    description: ''
  };
  showProgramModal.value = true;
};

const openEditProgram = (item) => {
  editingProgramItem.value = item;
  programForm.value = {
    periode: item.periode,
    time_label: item.time_label,
    tag: item.tag || '',
    title: item.title,
    description: item.description || ''
  };
  showProgramModal.value = true;
};

const saveProgramItem = async () => {
  if (!programForm.value.time_label.trim() || !programForm.value.title.trim()) {
    triggerToast("L'horaire et l'intitulé sont obligatoires.");
    return;
  }

  const payload = {
    periode: programForm.value.periode,
    time_label: programForm.value.time_label.trim(),
    tag: programForm.value.tag?.trim() || null,
    title: programForm.value.title.trim(),
    description: programForm.value.description?.trim() || null
  };

  if (editingProgramItem.value) {
    const { error } = await supabase
      .from('program_schedule')
      .update(payload)
      .eq('id', editingProgramItem.value.id);

    if (!error) {
      Object.assign(editingProgramItem.value, payload);
      showProgramModal.value = false;
      triggerToast('Horaire mis à jour.');
    }
  } else {
    const maxOrder = filteredProgramItems.value.reduce((max, i) => Math.max(max, i.sort_order || 0), 0);
    const { data, error } = await supabase
      .from('program_schedule')
      .insert([{ ...payload, sort_order: maxOrder + 1 }])
      .select();

    if (!error && data) {
      programItems.value.push(data[0]);
      showProgramModal.value = false;
      triggerToast('Moment ajouté au programme !');
    }
  }
};

const deleteProgramItem = async (id) => {
  if (!confirm('Supprimer cet horaire du programme ?')) return;
  const { error } = await supabase.from('program_schedule').delete().eq('id', id);
  if (!error) {
    programItems.value = programItems.value.filter(p => p.id !== id);
    triggerToast('Horaire supprimé.');
  }
};

const moveProgramItem = async (index, direction) => {
  const list = [...filteredProgramItems.value];
  const targetIndex = index + direction;
  if (targetIndex < 0 || targetIndex >= list.length) return;

  const temp = list[index];
  list[index] = list[targetIndex];
  list[targetIndex] = temp;

  for (let i = 0; i < list.length; i++) {
    list[i].sort_order = i;
    await supabase.from('program_schedule').update({ sort_order: i }).eq('id', list[i].id);
  }
  triggerToast('Ordre du programme mis à jour !');
};

// ================= GESTION VILLAGE SOLIDAIRE (GESTION SITE) ================= //
const villageItems = ref([]);
const showVillageModal = ref(false);
const editingVillageItem = ref(null);
const uploadingVillageImg = ref(false);

const villageLayoutConfig = ref({
  split: '50-50',
  rightCols: '2',
  spacing: 'normal',
  gapSize: 'normal',
  cardStyle: 'glass',
  theme: 'dark'
});

const villageForm = ref({
  title: '',
  tag: '',
  description: '',
  icon: 'fa-tent',
  image_url: '',
  logo_fit: 'original',
  logo_size: 'md',
  logo_bg: 'white',
  color_theme: 'amber',
  layout_zone: 'auto',
  is_wide: false
});

const villageIconOptions = [
  { id: 'fa-tent', label: 'Stand / Village' },
  { id: 'fa-hand-holding-heart', label: 'Association / Solidarité' },
  { id: 'fa-kit-medical', label: 'Poste de Secours' },
  { id: 'fa-mug-hot', label: 'Buvette / Café' },
  { id: 'fa-utensils', label: 'Restauration' },
  { id: 'fa-puzzle-piece', label: 'Atelier Mémoire / Jeux' },
  { id: 'fa-music', label: 'Animation Musicale' },
  { id: 'fa-ticket', label: 'Tombola / Billetterie' },
  { id: 'fa-camera', label: 'Photocall Souvenir' },
  { id: 'fa-medal', label: 'Podium / Récompenses' },
  { id: 'fa-child-reaching', label: 'Espace Enfants' }
];

const fetchVillageItems = async () => {
  const [{ data, error }, { data: cfgData }] = await Promise.all([
    supabase
      .from('village_activities')
      .select('*')
      .order('sort_order', { ascending: true })
      .order('id', { ascending: true }),
    supabase
      .from('site_settings')
      .select('value')
      .eq('key', 'village_layout_config')
      .maybeSingle()
  ]);

  if (!error && data) {
    villageItems.value = data;
  }
  if (cfgData?.value) {
    try {
      villageLayoutConfig.value = { ...villageLayoutConfig.value, ...JSON.parse(cfgData.value) };
    } catch (e) {}
  }
};

const saveVillageLayoutConfig = async () => {
  const { error } = await supabase
    .from('site_settings')
    .upsert({
      key: 'village_layout_config',
      value: JSON.stringify(villageLayoutConfig.value)
    });

  if (!error) {
    triggerToast('Paramètres visuels du Village Solidaire enregistrés !');
  }
};

const sortedVillageItems = computed(() => {
  return [...villageItems.value].sort((a, b) => (a.sort_order ?? 0) - (b.sort_order ?? 0));
});

const openCreateVillage = () => {
  editingVillageItem.value = null;
  villageForm.value = {
    title: '',
    tag: '',
    description: '',
    icon: 'fa-tent',
    image_url: '',
    logo_fit: 'original',
    logo_size: 'md',
    logo_bg: 'white',
    color_theme: 'amber',
    layout_zone: 'auto',
    is_wide: false
  };
  showVillageModal.value = true;
};

const openEditVillage = (item) => {
  editingVillageItem.value = item;
  villageForm.value = {
    title: item.title,
    tag: item.tag || '',
    description: item.description || '',
    icon: item.icon || 'fa-tent',
    image_url: item.image_url || '',
    logo_fit: item.logo_fit || 'original',
    logo_size: item.logo_size || 'md',
    logo_bg: item.logo_bg || 'white',
    color_theme: item.color_theme || 'amber',
    layout_zone: item.layout_zone || 'auto',
    is_wide: Boolean(item.is_wide)
  };
  showVillageModal.value = true;
};

const handleVillageImageUpload = async (event) => {
  const file = event.target.files?.[0];
  if (!file) return;

  uploadingVillageImg.value = true;
  try {
    const fileExt = file.name.split('.').pop();
    const fileName = `${Date.now()}_${Math.random().toString(36).substring(2, 8)}.${fileExt}`;

    const { error: uploadError } = await supabase.storage
      .from('village-icons')
      .upload(fileName, file);

    if (uploadError) throw uploadError;

    const { data: publicUrlData } = supabase.storage
      .from('village-icons')
      .getPublicUrl(fileName);

    villageForm.value.image_url = publicUrlData.publicUrl;
    triggerToast('Logo / image importé !');
  } catch (err) {
    console.error('Erreur upload image:', err);
    triggerToast("Impossible d'importer le fichier (vérifiez le bucket 'village-icons' ou collez une URL).");
  } finally {
    uploadingVillageImg.value = false;
  }
};

const saveVillageItem = async () => {
  if (!villageForm.value.title.trim()) {
    triggerToast('Le titre du stand ou de l’animation est obligatoire.');
    return;
  }

  const payload = {
    title: villageForm.value.title.trim(),
    tag: villageForm.value.tag?.trim() || null,
    description: villageForm.value.description?.trim() || null,
    icon: villageForm.value.icon || 'fa-tent',
    image_url: villageForm.value.image_url?.trim() || null,
    logo_fit: villageForm.value.logo_fit || 'original',
    logo_size: villageForm.value.logo_size || 'md',
    logo_bg: villageForm.value.logo_bg || 'white',
    color_theme: villageForm.value.color_theme || 'amber',
    layout_zone: villageForm.value.layout_zone || 'auto',
    is_wide: Boolean(villageForm.value.is_wide)
  };

  if (editingVillageItem.value) {
    const { data, error } = await supabase
      .from('village_activities')
      .update(payload)
      .eq('id', editingVillageItem.value.id)
      .select();

    if (error) {
      triggerToast(`Erreur : ${error.message}`);
      return;
    }

    if (data && data.length > 0) {
      Object.assign(editingVillageItem.value, data[0]);
    }
    showVillageModal.value = false;
    triggerToast('Stand / animation mis à jour !');
  } else {
    const maxOrder = sortedVillageItems.value.reduce((max, i) => Math.max(max, i.sort_order || 0), 0);
    const { data, error } = await supabase
      .from('village_activities')
      .insert([{ ...payload, sort_order: maxOrder + 1 }])
      .select();

    if (error) {
      triggerToast(`Erreur lors de l'enregistrement : ${error.message}`);
      return;
    }

    if (data && data.length > 0) {
      villageItems.value.push(data[0]);
      showVillageModal.value = false;
      triggerToast('Ajouté au Village Solidaire !');
    }
  }
};

const deleteVillageItem = async (id) => {
  if (!confirm('Supprimer cet élément du Village Solidaire ?')) return;
  const { error } = await supabase.from('village_activities').delete().eq('id', id);
  if (!error) {
    villageItems.value = villageItems.value.filter(v => v.id !== id);
    triggerToast('Élément supprimé.');
  }
};

const moveVillageItem = async (index, direction) => {
  const list = [...sortedVillageItems.value];
  const targetIndex = index + direction;
  if (targetIndex < 0 || targetIndex >= list.length) return;

  const temp = list[index];
  list[index] = list[targetIndex];
  list[targetIndex] = temp;

  for (let i = 0; i < list.length; i++) {
    list[i].sort_order = i;
    await supabase.from('village_activities').update({ sort_order: i }).eq('id', list[i].id);
  }
  triggerToast('Ordre du Village Solidaire mis à jour !');
};

onMounted(async () => {
  // Sécurité : débloque l'affichage au bout de 3 secondes si Supabase met trop de temps
  const fallbackTimer = setTimeout(() => {
    authLoading.value = false;
  }, 3000);

  try {
    const { data, error } = await supabase.auth.getSession();
    if (error) {
      console.warn('Session invalide ou expirée :', error.message);
      await supabase.auth.signOut();
      user.value = null;
    } else {
      user.value = data?.session?.user || null;
    }

    if (user.value) {
      await fetchProfileAndData();
    }
  } catch (err) {
    console.error('Erreur initialisation auth admin :', err);
    user.value = null;
  } finally {
    clearTimeout(fallbackTimer);
    authLoading.value = false;
  }
});
</script>

<template>
  <div v-if="authLoading" class="min-h-screen flex items-center justify-center bg-slate-100">
    <i class="fa-solid fa-circle-notch fa-spin text-3xl text-emerald-600"></i>
  </div>

  <!-- CONNEXION INITIALE -->
  <div v-else-if="!user" class="min-h-screen bg-slate-900 flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-md w-full p-8 shadow-2xl">
      <div class="text-center mb-6">
        <div class="w-14 h-14 bg-emerald-100 text-emerald-800 rounded-2xl flex items-center justify-center mx-auto text-2xl mb-3 shadow-xs">
          <i class="fa-solid fa-shield-halved"></i>
        </div>
        <h2 class="text-2xl font-black text-slate-900">Espace Privé</h2>
        <p class="text-xs text-slate-500 mt-1">Connexion par rôle & pôle — Pas d'oubli 2027</p>
      </div>

      <form @submit.prevent="handleLogin" class="space-y-4">
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">E-mail</label>
          <input v-model="loginEmail" required type="email" placeholder="votre-pole@pasdoubli.fr" class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm focus:border-emerald-600 outline-none" />
        </div>
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Mot de passe</label>
          <input v-model="loginPassword" required type="password" placeholder="••••••••" class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm focus:border-emerald-600 outline-none" />
        </div>
        
        <p v-if="loginError" class="text-xs text-rose-600 font-semibold text-center">{{ loginError }}</p>

        <button type="submit" class="w-full py-3 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-xl text-sm transition cursor-pointer shadow-md">
          Se connecter
        </button>
      </form>

      <div class="mt-6 text-center">
        <a href="/" class="text-xs font-semibold text-slate-400 hover:text-slate-600">
          <i class="fa-solid fa-arrow-left mr-1"></i>Retour au site public
        </a>
      </div>
    </div>
  </div>

  <!-- TABLEAU DE BORD CONNECTÉ -->
  <div v-else class="min-h-screen bg-slate-100 p-4 sm:p-8">
    <div class="max-w-7xl mx-auto">
      
      <!-- EN-TÊTE -->
      <div class="flex items-center justify-between mb-6 pb-4 border-b border-slate-200 flex-wrap gap-4">
        <div class="flex items-center gap-3">
          <!-- BOUTON HAMBURGER UNIQUEMENT SUR MOBILE (md:hidden) -->
          <button
            @click="showMobileMenu = true"
            class="md:hidden w-10 h-10 rounded-xl bg-emerald-700 hover:bg-emerald-800 text-white flex items-center justify-center shadow-xs transition cursor-pointer flex-shrink-0"
            title="Ouvrir le menu"
          >
            <i class="fa-solid fa-bars text-base"></i>
          </button>

          <div>
            <div class="flex items-center gap-2.5">
              <h1 class="text-xl sm:text-2xl font-black text-slate-900">Espace {{ userRoleTitle }}</h1>
              <span 
                v-if="isDirection" 
                class="px-2.5 py-0.5 bg-gradient-to-r from-amber-500 to-amber-600 text-white font-black text-[10px] uppercase tracking-wider rounded-full shadow-xs flex items-center gap-1"
              >
                <i class="fa-solid fa-crown text-[9px]"></i> Direction
              </span>
            </div>
            <p class="text-xs text-slate-500 mt-0.5">Connecté sous {{ user.email }}</p>
          </div>
        </div>

        <div class="flex items-center gap-2 flex-wrap">
          <button 
            @click="openSwitchAccount" 
            class="text-xs font-bold text-slate-700 bg-white border border-slate-300 hover:bg-slate-50 px-3 py-2 rounded-xl transition cursor-pointer flex items-center gap-1.5 shadow-2xs"
            title="Changer de compte"
          >
            <i class="fa-solid fa-users-gear text-emerald-600"></i>
            <span class="hidden sm:inline">Changer de compte</span>
          </button>

          <a href="/" class="text-xs font-bold text-slate-700 bg-white border border-slate-300 hover:bg-slate-50 px-3 py-2 rounded-xl transition cursor-pointer flex items-center gap-1.5 shadow-2xs" title="Voir le site">
            <i class="fa-solid fa-globe text-emerald-600"></i>
            <span class="hidden sm:inline">Voir le site</span>
          </a>

          <button @click="handleLogout" class="text-xs font-bold text-slate-700 bg-white border border-slate-300 hover:bg-slate-50 px-3 py-2 rounded-xl transition cursor-pointer flex items-center gap-1.5 shadow-2xs" title="Déconnexion">
            <i class="fa-solid fa-right-from-bracket"></i>
            <span class="hidden sm:inline">Déconnexion</span>
          </button>
        </div>
      </div>

      <!-- BARRE D'ONGLETS CLASSIQUE SUR ORDINATEUR UNIQUEMENT (hidden md:block) -->
      <nav class="hidden md:block border-b border-slate-300/80 mb-8 overflow-hidden">
        <div class="flex items-center gap-6 -mb-px flex-wrap">
          <button
            v-if="canAccessTab('poles')"
            @click="currentTab = 'poles'"
            :class="currentTab === 'poles' ? 'text-emerald-700 border-b-2 border-emerald-600 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-solid fa-table-columns text-xs"></i>
            <span>Tableau Kanban</span>
          </button>

          <button
            v-if="canAccessTab('materiel')"
            @click="currentTab = 'materiel'"
            :class="currentTab === 'materiel' ? 'text-emerald-700 border-b-2 border-emerald-600 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-solid fa-boxes-stacked text-xs"></i>
            <span>Besoins Matériel ({{ filteredMaterialRequests.length }})</span>
          </button>

          <button
            v-if="canAccessTab('budget_tresorerie')"
            @click="currentTab = 'budget_tresorerie'"
            :class="currentTab === 'budget_tresorerie' ? 'text-emerald-700 border-b-2 border-emerald-600 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-solid fa-file-invoice-dollar text-xs"></i>
            <span>Budget & Pôles</span>
          </button>

          <button
            v-if="canAccessTab('annuaire_sponsors')"
            @click="currentTab = 'annuaire_sponsors'"
            :class="currentTab === 'annuaire_sponsors' ? 'text-emerald-700 border-b-2 border-emerald-600 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-solid fa-address-book text-xs"></i>
            <span>Annuaire Sponsors ({{ sponsorProspects.length }})</span>
          </button>

          <button
            v-if="canAccessTab('besoins_sponsors')"
            @click="currentTab = 'besoins_sponsors'"
            :class="currentTab === 'besoins_sponsors' ? 'text-emerald-700 border-b-2 border-emerald-600 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-solid fa-handshake text-xs"></i>
            <span>Besoins à sponsoriser ({{ sponsorMaterialRequests.length }})</span>
            <span v-if="sponsorMaterialRequests.filter(s => s.statut_sponsor === 'en_attente').length > 0" class="w-2 h-2 rounded-full bg-amber-500"></span>
          </button>

          <button
            v-if="canAccessTab('messages')"
            @click="currentTab = 'messages'"
            :class="currentTab === 'messages' ? 'text-emerald-700 border-b-2 border-emerald-600 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-regular fa-envelope text-xs"></i>
            <span>Messages ({{ messages.length }})</span>
          </button>

          <button
            v-if="canAccessTab('benevoles')"
            @click="currentTab = 'benevoles'"
            :class="currentTab === 'benevoles' ? 'text-emerald-700 border-b-2 border-emerald-600 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-solid fa-users text-xs"></i>
            <span>Bénévoles ({{ volunteers.length }})</span>
          </button>

          <button
            v-if="canAccessTab('faq')"
            @click="currentTab = 'faq'"
            :class="currentTab === 'faq' ? 'text-emerald-700 border-b-2 border-emerald-600 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-regular fa-circle-question text-xs"></i>
            <span>FAQ ({{ faqs.length }})</span>
          </button>

          <button
            v-if="canAccessTab('cagnotte')"
            @click="currentTab = 'cagnotte'"
            :class="currentTab === 'cagnotte' ? 'text-emerald-700 border-b-2 border-emerald-600 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-solid fa-hand-holding-dollar text-xs"></i>
            <span>Actualiser Cagnotte</span>
          </button>

          <!-- ONGLET EXCLUSIF PRÉSIDENT / VICE-PRÉSIDENTE : GESTION SITE -->
          <button
            v-if="canAccessTab('gestion_site')"
            @click="currentTab = 'gestion_site'"
            :class="currentTab === 'gestion_site' ? 'text-amber-700 border-b-2 border-amber-500 font-extrabold' : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
            class="pb-3.5 text-sm transition-all cursor-pointer flex items-center gap-2 select-none"
          >
            <i class="fa-solid fa-sliders text-xs text-amber-500"></i>
            <span>Gestion Site</span>
          </button>
        </div>
      </nav>

      <!-- TIROIR HAMBURGER UNIQUEMENT SUR MOBILE (md:hidden) -->
      <div v-if="showMobileMenu" @click.self="showMobileMenu = false" class="fixed inset-0 z-50 md:hidden bg-black/50 backdrop-blur-xs flex">
        <div class="bg-white w-72 max-w-[85vw] h-full shadow-2xl p-6 flex flex-col justify-between overflow-y-auto">
          <div>
            <div class="flex items-center justify-between pb-4 mb-5 border-b border-slate-100">
              <div>
                <span class="text-[10px] font-black uppercase tracking-widest text-emerald-700">Pas d'oubli 2027</span>
                <h2 class="text-lg font-black text-slate-900">Menu</h2>
              </div>
              <button @click="showMobileMenu = false" class="w-8 h-8 rounded-lg bg-slate-100 hover:bg-slate-200 text-slate-600 flex items-center justify-center cursor-pointer">
                <i class="fa-solid fa-xmark"></i>
              </button>
            </div>

            <nav class="space-y-1.5">
              <button
                v-if="canAccessTab('poles')"
                @click="selectAdminTab('poles')"
                :class="currentTab === 'poles' ? 'bg-emerald-700 text-white font-bold shadow-xs' : 'text-slate-700 hover:bg-slate-100 font-medium'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center gap-3 transition cursor-pointer text-left"
              >
                <i class="fa-solid fa-table-columns w-4 text-center"></i>
                <span>Tableau Kanban</span>
              </button>

              <button
                v-if="canAccessTab('materiel')"
                @click="selectAdminTab('materiel')"
                :class="currentTab === 'materiel' ? 'bg-emerald-700 text-white font-bold shadow-xs' : 'text-slate-700 hover:bg-slate-100 font-medium'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center justify-between transition cursor-pointer text-left"
              >
                <span class="flex items-center gap-3">
                  <i class="fa-solid fa-boxes-stacked w-4 text-center"></i>
                  <span>Besoins Matériel</span>
                </span>
                <span class="px-2 py-0.5 rounded-full text-[10px] font-mono" :class="currentTab === 'materiel' ? 'bg-white/20 text-white' : 'bg-slate-200 text-slate-700'">
                  {{ filteredMaterialRequests.length }}
                </span>
              </button>

              <button
                v-if="canAccessTab('budget_tresorerie')"
                @click="selectAdminTab('budget_tresorerie')"
                :class="currentTab === 'budget_tresorerie' ? 'bg-emerald-700 text-white font-bold shadow-xs' : 'text-slate-700 hover:bg-slate-100 font-medium'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center gap-3 transition cursor-pointer text-left"
              >
                <i class="fa-solid fa-file-invoice-dollar w-4 text-center"></i>
                <span>Budget & Pôles</span>
              </button>

              <button
                v-if="canAccessTab('annuaire_sponsors')"
                @click="selectAdminTab('annuaire_sponsors')"
                :class="currentTab === 'annuaire_sponsors' ? 'bg-emerald-700 text-white font-bold shadow-xs' : 'text-slate-700 hover:bg-slate-100 font-medium'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center justify-between transition cursor-pointer text-left"
              >
                <span class="flex items-center gap-3">
                  <i class="fa-solid fa-address-book w-4 text-center"></i>
                  <span>Annuaire Sponsors</span>
                </span>
                <span class="px-2 py-0.5 rounded-full text-[10px] font-mono" :class="currentTab === 'annuaire_sponsors' ? 'bg-white/20 text-white' : 'bg-slate-200 text-slate-700'">
                  {{ sponsorProspects.length }}
                </span>
              </button>

              <button
                v-if="canAccessTab('besoins_sponsors')"
                @click="selectAdminTab('besoins_sponsors')"
                :class="currentTab === 'besoins_sponsors' ? 'bg-emerald-700 text-white font-bold shadow-xs' : 'text-slate-700 hover:bg-slate-100 font-medium'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center justify-between transition cursor-pointer text-left"
              >
                <span class="flex items-center gap-3">
                  <i class="fa-solid fa-handshake w-4 text-center"></i>
                  <span>Besoins à sponsoriser</span>
                </span>
                <span class="px-2 py-0.5 rounded-full text-[10px] font-mono" :class="currentTab === 'besoins_sponsors' ? 'bg-white/20 text-white' : 'bg-slate-200 text-slate-700'">
                  {{ sponsorMaterialRequests.length }}
                </span>
              </button>

              <button
                v-if="canAccessTab('messages')"
                @click="selectAdminTab('messages')"
                :class="currentTab === 'messages' ? 'bg-emerald-700 text-white font-bold shadow-xs' : 'text-slate-700 hover:bg-slate-100 font-medium'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center justify-between transition cursor-pointer text-left"
              >
                <span class="flex items-center gap-3">
                  <i class="fa-regular fa-envelope w-4 text-center"></i>
                  <span>Messages</span>
                </span>
                <span class="px-2 py-0.5 rounded-full text-[10px] font-mono" :class="currentTab === 'messages' ? 'bg-white/20 text-white' : 'bg-slate-200 text-slate-700'">
                  {{ messages.length }}
                </span>
              </button>

              <button
                v-if="canAccessTab('benevoles')"
                @click="selectAdminTab('benevoles')"
                :class="currentTab === 'benevoles' ? 'bg-emerald-700 text-white font-bold shadow-xs' : 'text-slate-700 hover:bg-slate-100 font-medium'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center justify-between transition cursor-pointer text-left"
              >
                <span class="flex items-center gap-3">
                  <i class="fa-solid fa-users w-4 text-center"></i>
                  <span>Bénévoles</span>
                </span>
                <span class="px-2 py-0.5 rounded-full text-[10px] font-mono" :class="currentTab === 'benevoles' ? 'bg-white/20 text-white' : 'bg-slate-200 text-slate-700'">
                  {{ volunteers.length }}
                </span>
              </button>

              <button
                v-if="canAccessTab('faq')"
                @click="selectAdminTab('faq')"
                :class="currentTab === 'faq' ? 'bg-emerald-700 text-white font-bold shadow-xs' : 'text-slate-700 hover:bg-slate-100 font-medium'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center gap-3 transition cursor-pointer text-left"
              >
                <i class="fa-regular fa-circle-question w-4 text-center"></i>
                <span>FAQ ({{ faqs.length }})</span>
              </button>

              <button
                v-if="canAccessTab('cagnotte')"
                @click="selectAdminTab('cagnotte')"
                :class="currentTab === 'cagnotte' ? 'bg-emerald-700 text-white font-bold shadow-xs' : 'text-slate-700 hover:bg-slate-100 font-medium'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center gap-3 transition cursor-pointer text-left"
              >
                <i class="fa-solid fa-hand-holding-dollar w-4 text-center"></i>
                <span>Actualiser Cagnotte</span>
              </button>

              <!-- ONGLET EXCLUSIF PRÉSIDENT / VICE-PRÉSIDENTE : GESTION SITE -->
              <button
                v-if="canAccessTab('gestion_site')"
                @click="selectAdminTab('gestion_site')"
                :class="currentTab === 'gestion_site' ? 'bg-amber-600 text-white font-bold shadow-xs' : 'text-amber-900 bg-amber-50/70 hover:bg-amber-100 font-bold'"
                class="w-full px-4 py-3 rounded-xl text-xs flex items-center gap-3 transition cursor-pointer text-left mt-3 border border-amber-200"
              >
                <i class="fa-solid fa-sliders w-4 text-center"></i>
                <span>Gestion Site</span>
              </button>
            </nav>
          </div>

          <div class="pt-4 border-t border-slate-100 text-[11px] text-slate-400">
            Connecté en tant que <strong class="text-slate-700">{{ userRoleTitle }}</strong>
          </div>
        </div>
      </div>

      <!-- ================= VUE KANBAN ================= -->
      <div v-if="currentTab === 'poles'" class="space-y-6">
        <div v-if="accessiblePoles.length > 1" class="border-b border-slate-200 mb-6 overflow-hidden">
          <div class="flex items-center gap-6 -mb-px flex-wrap">
            <button 
              v-for="p in accessiblePoles" 
              :key="p.id"
              @click="selectedPole = p.id"
              :class="selectedPole === p.id 
                ? 'text-emerald-700 border-b-2 border-emerald-600 font-bold' 
                : 'text-slate-500 hover:text-slate-800 border-b-2 border-transparent font-medium'"
              class="pb-3 text-xs flex items-center gap-2 transition cursor-pointer select-none"
            >
              <i :class="['fa-solid', p.icon]"></i>
              <span>{{ p.label }}</span>
              <span class="text-[10px] px-1.5 py-0.2 rounded-full" :class="selectedPole === p.id ? 'bg-emerald-100 text-emerald-800' : 'bg-slate-200 text-slate-600'">
                {{ tasks.filter(t => t.pole === p.id).length }}
              </span>
            </button>
          </div>
        </div>

        <div class="flex items-center justify-between">
          <div>
            <h2 class="text-xl font-black text-slate-900 capitalize flex items-center gap-2">
              <span>Pôle {{ allPoles.find(p => p.id === selectedPole)?.label }}</span>
            </h2>
            <p class="text-xs text-slate-500">Cliquez sur un post-it pour ajouter ou cocher ses étapes.</p>
          </div>
          <button 
            @click="openCreateTask('sujet')"
            class="px-4 py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white text-xs font-bold rounded-xl shadow-xs transition cursor-pointer flex items-center gap-2"
          >
            <i class="fa-solid fa-plus"></i>
            <span>Nouveau Post-it</span>
          </button>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 items-start">
          <div 
            v-for="col in columns" 
            :key="col.id"
            @dragover.prevent
            @drop="onDrop($event, col.id)"
            class="bg-slate-200/60 rounded-2xl p-3 border border-slate-300/80 min-h-[480px] flex flex-col"
          >
            <div :class="['px-3 py-2 rounded-xl font-black text-xs flex items-center justify-between mb-3 border shadow-2xs', col.bgHeader]">
              <span>{{ col.label }}</span>
              <span class="bg-white/80 px-2 py-0.5 rounded-full text-[10px] font-mono">
                {{ tasksByStatus(col.id).length }}
              </span>
            </div>

            <div class="space-y-3 flex-1">
              <div 
                v-for="task in tasksByStatus(col.id)" 
                :key="task.id"
                draggable="true"
                @dragstart="onDragStart($event, task)"
                @click="openDetail(task)"
                :class="['bg-white rounded-xl p-4 border-2 shadow-xs hover:shadow-md transition cursor-grab active:cursor-grabbing relative group', task.border_color || 'border-slate-200']"
              >
                <div class="flex items-start justify-between gap-2 mb-2">
                  <span 
                    v-if="task.assigne_a" 
                    class="text-[10px] font-bold px-2 py-0.5 bg-slate-100 text-slate-700 rounded-md border border-slate-200 truncate"
                  >
                    <i class="fa-solid fa-user text-[9px] text-emerald-600 mr-1"></i>{{ task.assigne_a }}
                  </span>
                  <span v-else class="text-[10px] italic text-slate-400">Non assigné</span>

                  <div class="flex items-center gap-1 opacity-80 group-hover:opacity-100">
                    <button @click.stop="openEditTask(task)" class="p-1 text-slate-400 hover:text-emerald-700 cursor-pointer">
                      <i class="fa-solid fa-pen text-[11px]"></i>
                    </button>
                    <button @click.stop="deleteTask(task.id)" class="p-1 text-slate-400 hover:text-rose-600 cursor-pointer">
                      <i class="fa-solid fa-trash text-[11px]"></i>
                    </button>
                  </div>
                </div>

                <h4 class="font-extrabold text-slate-900 text-sm leading-snug mb-1">{{ task.titre }}</h4>
                <p v-if="task.description" class="text-xs text-slate-600 line-clamp-2 leading-relaxed mb-3">
                  {{ task.description }}
                </p>

                <div v-if="getTaskProgress(task.id)" class="mb-3 bg-slate-50 border border-slate-100 rounded-lg p-2">
                  <div class="flex justify-between items-center text-[10px] font-bold text-slate-600 mb-1">
                    <span class="flex items-center gap-1">
                      <i class="fa-solid fa-list-check text-emerald-600 text-[11px]"></i>
                      Étapes
                    </span>
                    <span>{{ getTaskProgress(task.id).done }} / {{ getTaskProgress(task.id).total }}</span>
                  </div>
                  <div class="w-full bg-slate-200 rounded-full h-1.5 overflow-hidden">
                    <div 
                      class="bg-emerald-600 h-full rounded-full transition-all duration-300"
                      :style="{ width: `${getTaskProgress(task.id).percent}%` }"
                    ></div>
                  </div>
                </div>

                <div class="pt-2 border-t border-slate-100 flex items-center justify-between text-[11px] text-slate-400">
                  <span class="hover:text-emerald-700 font-semibold flex items-center gap-1">
                    <i class="fa-regular fa-comment-dots text-xs"></i>
                    <span>{{ comments.filter(c => c.task_id === task.id).length }} avis</span>
                  </span>
                  <span class="text-[10px]">{{ new Date(task.created_at).toLocaleDateString('fr-FR', { day: '2-digit', month: '2-digit' }) }}</span>
                </div>
              </div>

              <button 
                @click="openCreateTask(col.id)"
                class="w-full py-2 border-2 border-dashed border-slate-300 hover:border-emerald-600 hover:text-emerald-700 text-slate-400 text-xs font-bold rounded-xl transition cursor-pointer flex items-center justify-center gap-1.5"
              >
                <i class="fa-solid fa-plus text-[10px]"></i>
                <span>Ajouter</span>
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- ================= VUE BESOINS EN MATÉRIEL (LOGISTIQUE) ================= -->
      <div v-if="currentTab === 'materiel' && canAccessTab('materiel')" class="space-y-6">
        <div class="flex items-center justify-between flex-wrap gap-4">
          <div>
            <h2 class="text-xl font-black text-slate-900 flex items-center gap-2">
              <i class="fa-solid fa-boxes-packing text-emerald-600"></i>
              <span>{{ isLogistiqueOrDirection ? 'Tableau Central du Matériel (Pôle Logistique)' : 'Besoins Matériel de votre Pôle' }}</span>
            </h2>
            <p class="text-xs text-slate-500 mt-0.5">
              Filtrez par colonne, modifiez vos demandes et suivez les arbitrages logistiques, financiers et sponsors.
            </p>
          </div>

          <div class="flex items-center gap-2.5 flex-wrap">
            <button 
              @click="exportToExcel('logistique')" 
              class="px-3.5 py-2.5 bg-emerald-800 hover:bg-emerald-900 text-white font-bold text-xs rounded-xl shadow-2xs transition cursor-pointer flex items-center gap-2"
            >
              <i class="fa-solid fa-file-excel text-sm"></i>
              <span>Exporter Excel (.csv)</span>
            </button>

            <button 
              @click="openMaterialRequestModal"
              class="px-4 py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold text-xs rounded-xl shadow-xs transition cursor-pointer flex items-center gap-2"
            >
              <i class="fa-solid fa-plus"></i>
              <span>Demander du matériel</span>
            </button>
          </div>
        </div>

        <div class="bg-white rounded-2xl border border-slate-200 shadow-xs overflow-hidden">
          <div class="overflow-x-auto">
            <table class="w-full text-left border-collapse text-xs">
              <thead>
                <tr class="bg-slate-50 border-b border-slate-200 text-slate-500 uppercase tracking-wider font-extrabold text-[10px]">
                  <th class="p-3 w-32">
                    <div class="mb-1.5">Pôle</div>
                    <select v-model="matColFilters.pole" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                      <option value="">Tous</option>
                      <option v-for="p in allPoles" :key="p.id" :value="p.id">{{ p.label }}</option>
                    </select>
                  </th>
                  <th class="p-3">
                    <div class="mb-1.5">Commande & Détails</div>
                    <input v-model="matColFilters.article" type="text" placeholder="Rechercher article, justification..." class="w-full px-2.5 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                  </th>
                  <th class="p-3 w-32">
                    <div class="mb-1.5">Prix indicatif</div>
                    <input v-model="matColFilters.prix" type="text" placeholder="Montant..." class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                  </th>
                  <th class="p-3 w-40">
                    <div class="mb-1.5">Avis Logistique</div>
                    <select v-model="matColFilters.avis_logistique" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                      <option value="">Tous</option>
                      <option value="en_attente">En attente</option>
                      <option value="valide">Validé</option>
                      <option value="refuse">Refusé</option>
                      <option value="sponsorise">Sponsorisé</option>
                    </select>
                  </th>
                  <th class="p-3 w-40">
                    <div class="mb-1.5">Avis Trésorerie</div>
                    <select v-model="matColFilters.avis_tresorerie" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                      <option value="">Tous</option>
                      <option value="en_attente">En attente</option>
                      <option value="valide">Validé</option>
                      <option value="refuse">Refusé</option>
                      <option value="sponsorise">Sponsorisé</option>
                    </select>
                  </th>
                  <th class="p-3 w-44">
                    <div class="mb-1.5">Sponsors</div>
                    <select v-model="matColFilters.statut_sponsor" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                      <option value="">Tous</option>
                      <option value="non_delegue">Non délégué</option>
                      <option value="en_attente">En attente</option>
                      <option value="en_recherche">En recherche</option>
                      <option value="sponsor_trouve">Sponsor trouvé</option>
                      <option value="non_finance">Non sponsorisé</option>
                      <option value="refuse">Refusé</option>
                    </select>
                  </th>
                  <th class="p-3 text-right w-24 align-top">
                    <div class="mb-1.5">Actions</div>
                    <button @click="resetMatFilters" class="text-[10px] font-bold text-emerald-700 hover:underline normal-case cursor-pointer" title="Réinitialiser les filtres">
                      Réinitialiser
                    </button>
                  </th>
                </tr>
              </thead>
              <tbody class="divide-y divide-slate-100">
                <tr v-for="item in filteredMaterialRequests" :key="item.id" class="hover:bg-slate-50/70 transition">
                  <td class="p-4 whitespace-nowrap align-top">
                    <span class="inline-block px-2.5 py-1 rounded-md text-[10px] font-black uppercase tracking-wider bg-slate-100 text-slate-700 border border-slate-200">
                      {{ item.pole }}
                    </span>
                    <div class="text-[10px] text-slate-400 mt-1">Par {{ item.demandeur }}</div>
                  </td>

                  <td class="p-4 align-top">
                    <div class="flex items-center gap-2 mb-1.5">
                      <span class="font-black text-slate-900 text-sm">{{ item.article }}</span>
                      <span class="px-2 py-0.5 bg-emerald-100 text-emerald-800 text-xs font-mono font-bold rounded-full">
                        x{{ item.quantite }}
                      </span>
                    </div>

                    <div class="p-2.5 bg-slate-50 border border-slate-200/80 rounded-xl text-slate-700 text-xs">
                      <div class="flex items-center gap-1 text-[10px] font-bold uppercase text-slate-500 mb-0.5">
                        <i class="fa-solid fa-circle-info text-emerald-600"></i>
                        <span>Justification du besoin :</span>
                      </div>
                      <p class="leading-relaxed whitespace-pre-wrap">{{ item.details }}</p>
                    </div>
                  </td>

                  <td class="p-4 whitespace-nowrap align-top">
                    <div v-if="item.statut_sponsor === 'sponsor_trouve'">
                      <span class="px-2 py-0.5 bg-emerald-50 text-emerald-800 border border-emerald-200 rounded text-[11px] font-bold">
                        0.00 € (Sponsorisé)
                      </span>
                    </div>
                    <div v-else-if="item.estimated_price !== null && item.estimated_price !== undefined">
                      <span class="font-mono font-bold text-emerald-800 bg-emerald-50 border border-emerald-200 px-2 py-0.5 rounded-md text-[11px]">
                        {{ Number(item.estimated_price).toFixed(2) }} € <span class="text-[9px] font-sans font-normal text-slate-500">/u</span>
                      </span>
                      <div class="text-[10px] text-slate-500 font-semibold mt-1">
                        Total : <strong class="text-slate-800 font-mono">{{ (Number(item.estimated_price) * item.quantite).toFixed(2) }} €</strong>
                      </div>
                    </div>
                    <span v-else class="text-slate-400 italic text-[11px]">Non renseigné</span>
                  </td>

                  <!-- AVIS LOGISTIQUE -->
                  <td class="p-4 align-top">
                    <div v-if="item.statut_sponsor === 'sponsor_trouve'" class="text-emerald-700 font-bold text-xs flex items-center gap-1.5">
                      <i class="fa-solid fa-circle-check"></i>
                      <span>Pris en charge par sponsor</span>
                    </div>
                    <div v-else class="space-y-1.5">
                      <span :class="[decisionStatusList.find(s => s.id === (item.avis_logistique || item.status))?.badge, 'px-2.5 py-1 rounded-md text-[10px] font-bold border inline-block']">
                        {{ decisionStatusList.find(s => s.id === (item.avis_logistique || item.status))?.label }}
                      </span>
                      <p v-if="item.reponse_logistique" class="text-xs text-slate-600 bg-slate-50 p-2 rounded-lg border border-slate-200/60 italic leading-snug">
                        « {{ item.reponse_logistique }} »
                      </p>
                      <button 
                        v-if="isLogistiqueOrDirection"
                        @click="openDecisionModal(item, 'logistique')"
                        class="text-[11px] font-bold text-emerald-700 hover:underline cursor-pointer flex items-center gap-1 mt-1"
                      >
                        <i class="fa-solid fa-stamp text-[10px]"></i>
                        <span>Arbitrer (Logistique)</span>
                      </button>
                    </div>
                  </td>

                  <!-- AVIS TRÉSORERIE -->
                  <td class="p-4 align-top">
                    <div v-if="item.statut_sponsor === 'sponsor_trouve'" class="text-emerald-700 font-bold text-xs flex items-center gap-1.5">
                      <i class="fa-solid fa-circle-check"></i>
                      <span>Don sponsor (0 € asso)</span>
                    </div>
                    <div v-else class="space-y-1.5">
                      <span :class="[decisionStatusList.find(s => s.id === (item.avis_tresorerie || 'en_attente'))?.badge, 'px-2.5 py-1 rounded-md text-[10px] font-bold border inline-block']">
                        {{ decisionStatusList.find(s => s.id === (item.avis_tresorerie || 'en_attente'))?.label }}
                      </span>
                      <p v-if="item.reponse_tresorerie" class="text-xs text-slate-600 bg-slate-50 p-2 rounded-lg border border-slate-200/60 italic leading-snug">
                        « {{ item.reponse_tresorerie }} »
                      </p>
                      <span v-else class="text-[11px] text-slate-400 italic block">En attente</span>
                    </div>
                  </td>

                  <!-- STATUT SPONSOR -->
                  <td class="p-4 align-top">
                    <div v-if="item.demande_sponsor">
                      <span v-if="item.statut_sponsor === 'sponsor_trouve'" class="px-2.5 py-1 rounded-md text-[10px] font-black bg-emerald-100 text-emerald-900 border border-emerald-300 inline-flex items-center gap-1 shadow-2xs">
                        <i class="fa-solid fa-circle-check text-emerald-600"></i>
                        <span>Sponsor trouvé</span>
                      </span>

                      <span v-else-if="item.statut_sponsor === 'en_recherche'" class="px-2.5 py-1 rounded-md text-[10px] font-bold bg-amber-50 text-amber-900 border border-amber-300 inline-flex items-center gap-1">
                        <i class="fa-solid fa-magnifying-glass text-amber-600 animate-pulse"></i>
                        <span>Recherche en cours</span>
                      </span>

                      <span v-else-if="item.statut_sponsor === 'non_finance'" class="px-2.5 py-1 rounded-md text-[10px] font-bold bg-slate-100 text-slate-700 border border-slate-300 inline-flex items-center gap-1">
                        <i class="fa-solid fa-triangle-exclamation text-amber-600"></i>
                        <span>Non sponsorisé</span>
                      </span>

                      <span v-else-if="item.statut_sponsor === 'refuse'" class="px-2.5 py-1 rounded-md text-[10px] font-bold bg-rose-50 text-rose-800 border border-rose-200 inline-flex items-center gap-1">
                        <i class="fa-solid fa-ban text-rose-500"></i>
                        <span>Refusé Sponsors</span>
                      </span>

                      <span v-else class="px-2.5 py-1 rounded-md text-[10px] font-bold bg-blue-50 text-blue-800 border border-blue-200 inline-flex items-center gap-1">
                        <i class="fa-regular fa-clock text-blue-500"></i>
                        <span>En attente de prise en charge</span>
                      </span>
                    </div>

                    <button 
                      v-else-if="canDelegateToSponsors"
                      @click="askSponsorsForMaterial(item)"
                      class="px-2.5 py-1 text-[11px] font-bold text-amber-800 bg-amber-50 hover:bg-amber-100 border border-amber-300 rounded-lg transition cursor-pointer flex items-center gap-1.5 shadow-2xs"
                      title="Déléguer ce besoin aux sponsors"
                    >
                      <i class="fa-solid fa-handshake"></i>
                      <span>Déléguer</span>
                    </button>
                    <span v-else class="text-slate-400 italic text-[11px]">Non délégué</span>
                  </td>

                  <td class="p-4 text-right whitespace-nowrap align-top">
                    <div class="flex items-center justify-end gap-1.5">
                      <button 
                        @click="openEditMaterialModal(item)"
                        class="text-slate-400 hover:text-emerald-700 p-1.5 rounded-lg hover:bg-emerald-50 transition cursor-pointer"
                        title="Modifier cette ligne"
                      >
                        <i class="fa-solid fa-pen text-xs"></i>
                      </button>
                      <button 
                        @click="deleteMaterialRequest(item.id)" 
                        class="text-slate-400 hover:text-rose-600 p-1.5 rounded-lg hover:bg-rose-50 transition cursor-pointer"
                        title="Supprimer la demande"
                      >
                        <i class="fa-solid fa-trash text-xs"></i>
                      </button>
                    </div>
                  </td>
                </tr>

                <tr v-if="filteredMaterialRequests.length === 0">
                  <td colspan="7" class="p-10 text-center text-slate-400 italic text-xs">
                    Aucune demande de matériel ne correspond aux filtres sélectionnés.
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <!-- ================= VUE BUDGET & TRÉSORERIE PAR PÔLE ================= -->
      <div v-if="currentTab === 'budget_tresorerie' && canAccessTab('budget_tresorerie')" class="space-y-6">
        <div class="flex items-center justify-between flex-wrap gap-4">
          <div>
            <h2 class="text-xl font-black text-slate-900 flex items-center gap-2">
              <i class="fa-solid fa-file-invoice-dollar text-emerald-600"></i>
              <span>Budget Matériel & Arbitrage Trésorerie</span>
            </h2>
            <p class="text-xs text-slate-500 mt-0.5">
              Filtrez par colonne et arbitrez les dépenses demandées. Les matériels sponsorisés sont automatiquement exclus du coût.
            </p>
          </div>

          <div class="flex items-center gap-2.5">
            <button 
              @click="exportToExcel('tresorerie')" 
              class="px-4 py-2.5 bg-emerald-800 hover:bg-emerald-900 text-white font-bold text-xs rounded-xl shadow-2xs transition cursor-pointer flex items-center gap-2"
            >
              <i class="fa-solid fa-file-excel text-sm"></i>
              <span>Exporter Rapport Budgétaire (.csv)</span>
            </button>
          </div>
        </div>

        <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
          <div 
            v-for="b in budgetSummaryByPole" 
            :key="b.label"
            class="bg-white rounded-2xl border border-slate-200 p-5 shadow-xs flex flex-col justify-between"
          >
            <div>
              <div class="flex items-center justify-between mb-2">
                <span class="font-black text-sm uppercase tracking-wider text-slate-800">Pôle {{ b.label }}</span>
                <span class="text-xs bg-slate-100 px-2 py-0.5 rounded-full font-bold text-slate-600">
                  {{ b.count }} article(s)
                </span>
              </div>
              <div class="text-2xl font-black text-emerald-700 font-mono mt-1">
                {{ b.total.toFixed(2) }} €
              </div>
            </div>

            <div class="mt-4 pt-3 border-t border-slate-100 flex items-center justify-between text-xs text-slate-500">
              <span>Budget validé (hors sponsors) :</span>
              <strong class="font-mono text-emerald-800 font-bold">{{ b.validatedTotal.toFixed(2) }} €</strong>
            </div>
          </div>
        </div>

        <div class="bg-white rounded-2xl border border-slate-200 shadow-xs overflow-hidden">
          <div class="p-4 border-b border-slate-200 bg-slate-50 flex justify-between items-center flex-wrap gap-2">
            <span class="font-bold text-xs text-slate-700 uppercase tracking-wider">Demandes de dépenses et arbitrage financier ({{ filteredTresorerieRequests.length }})</span>
            <span class="text-xs font-semibold text-slate-500">Total estimé à payer : <strong class="text-emerald-700 font-mono">{{ totalEstimatedBudget.toFixed(2) }} €</strong></span>
          </div>
          <div class="overflow-x-auto">
            <table class="w-full text-left border-collapse text-xs">
              <thead>
                <tr class="bg-slate-50 border-b border-slate-200 text-slate-500 uppercase tracking-wider font-extrabold text-[10px]">
                  <th class="p-3 w-32">
                    <div class="mb-1.5">Pôle</div>
                    <select v-model="tresorColFilters.pole" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                      <option value="">Tous</option>
                      <option v-for="p in allPoles" :key="p.id" :value="p.id">{{ p.label }}</option>
                    </select>
                  </th>
                  <th class="p-3">
                    <div class="mb-1.5">Commande & Détails</div>
                    <input v-model="tresorColFilters.article" type="text" placeholder="Rechercher article, justification..." class="w-full px-2.5 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                  </th>
                  <th class="p-3 w-32">
                    <div class="mb-1.5">Prix indicatif</div>
                    <input v-model="tresorColFilters.prix" type="text" placeholder="Montant..." class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                  </th>
                  <th class="p-3 w-40">
                    <div class="mb-1.5">Avis Logistique</div>
                    <select v-model="tresorColFilters.avis_logistique" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                      <option value="">Tous</option>
                      <option value="en_attente">En attente</option>
                      <option value="valide">Validé</option>
                      <option value="refuse">Refusé</option>
                      <option value="sponsorise">Sponsorisé</option>
                    </select>
                  </th>
                  <th class="p-3 w-40">
                    <div class="mb-1.5">Avis Trésorerie</div>
                    <select v-model="tresorColFilters.avis_tresorerie" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                      <option value="">Tous</option>
                      <option value="en_attente">En attente</option>
                      <option value="valide">Validé</option>
                      <option value="refuse">Refusé</option>
                      <option value="sponsorise">Sponsorisé</option>
                    </select>
                  </th>
                  <th class="p-3 w-44">
                    <div class="mb-1.5">Sponsors</div>
                    <select v-model="tresorColFilters.statut_sponsor" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                      <option value="">Tous</option>
                      <option value="non_delegue">Non délégué</option>
                      <option value="en_attente">En attente</option>
                      <option value="en_recherche">En recherche</option>
                      <option value="sponsor_trouve">Sponsor trouvé</option>
                      <option value="non_finance">Non sponsorisé</option>
                      <option value="refuse">Refusé</option>
                    </select>
                  </th>
                  <th class="p-3 text-right w-24 align-top">
                    <div class="mb-1.5">Actions</div>
                    <button @click="resetTresorFilters" class="text-[10px] font-bold text-emerald-700 hover:underline normal-case cursor-pointer">
                      Réinitialiser
                    </button>
                  </th>
                </tr>
              </thead>
              <tbody class="divide-y divide-slate-100">
                <tr v-for="item in filteredTresorerieRequests" :key="item.id" class="hover:bg-slate-50/70 transition">
                  <td class="p-4 whitespace-nowrap align-top">
                    <span class="inline-block px-2.5 py-1 rounded-md text-[10px] font-black uppercase tracking-wider bg-slate-100 text-slate-700 border border-slate-200">
                      {{ item.pole }}
                    </span>
                    <div class="text-[10px] text-slate-400 mt-1">Par {{ item.demandeur }}</div>
                  </td>

                  <td class="p-4 align-top">
                    <div class="flex items-center gap-2 mb-1.5">
                      <span class="font-black text-slate-900 text-sm">{{ item.article }}</span>
                      <span class="px-2 py-0.5 bg-emerald-100 text-emerald-800 text-xs font-mono font-bold rounded-full">
                        x{{ item.quantite }}
                      </span>
                    </div>

                    <div class="p-2.5 bg-slate-50 border border-slate-200/80 rounded-xl text-slate-700 text-xs">
                      <div class="flex items-center gap-1 text-[10px] font-bold uppercase text-slate-500 mb-0.5">
                        <i class="fa-solid fa-circle-info text-emerald-600"></i>
                        <span>Justification du besoin :</span>
                      </div>
                      <p class="leading-relaxed whitespace-pre-wrap">{{ item.details }}</p>
                    </div>
                  </td>

                  <td class="p-4 whitespace-nowrap align-top">
                    <div v-if="item.statut_sponsor === 'sponsor_trouve'">
                      <span class="px-2 py-0.5 bg-emerald-50 text-emerald-800 border border-emerald-200 rounded text-[11px] font-bold">
                        0.00 € (Sponsorisé)
                      </span>
                    </div>
                    <div v-else-if="item.estimated_price !== null && item.estimated_price !== undefined">
                      <span class="font-mono font-bold text-emerald-800 bg-emerald-50 border border-emerald-200 px-2 py-0.5 rounded-md text-[11px]">
                        {{ Number(item.estimated_price).toFixed(2) }} € <span class="text-[9px] font-sans font-normal text-slate-500">/u</span>
                      </span>
                      <div class="text-[10px] text-slate-500 font-semibold mt-1">
                        Total : <strong class="text-slate-800 font-mono">{{ (Number(item.estimated_price) * item.quantite).toFixed(2) }} €</strong>
                      </div>
                    </div>
                    <span v-else class="text-slate-400 italic text-[11px]">Non renseigné</span>
                  </td>

                  <td class="p-4 align-top">
                    <div v-if="item.statut_sponsor === 'sponsor_trouve'" class="text-emerald-700 font-bold text-xs flex items-center gap-1.5">
                      <i class="fa-solid fa-circle-check"></i>
                      <span>Pris en charge sponsor</span>
                    </div>
                    <div v-else class="space-y-1.5">
                      <span :class="[decisionStatusList.find(s => s.id === (item.avis_logistique || item.status))?.badge, 'px-2.5 py-1 rounded-md text-[10px] font-bold border inline-block']">
                        {{ decisionStatusList.find(s => s.id === (item.avis_logistique || item.status))?.label }}
                      </span>
                      <p v-if="item.reponse_logistique" class="text-xs text-slate-600 bg-slate-50 p-2 rounded-lg border border-slate-200/60 italic leading-snug">
                        « {{ item.reponse_logistique }} »
                      </p>
                    </div>
                  </td>

                  <td class="p-4 align-top">
                    <div v-if="item.statut_sponsor === 'sponsor_trouve'" class="text-emerald-700 font-bold text-xs flex items-center gap-1.5">
                      <i class="fa-solid fa-circle-check"></i>
                      <span>0 € à décaisser</span>
                    </div>
                    <div v-else class="space-y-1.5">
                      <span :class="[decisionStatusList.find(s => s.id === (item.avis_tresorerie || 'en_attente'))?.badge, 'px-2.5 py-1 rounded-md text-[10px] font-bold border inline-block']">
                        {{ decisionStatusList.find(s => s.id === (item.avis_tresorerie || 'en_attente'))?.label }}
                      </span>
                      <p v-if="item.reponse_tresorerie" class="text-xs text-slate-600 bg-slate-50 p-2 rounded-lg border border-slate-200/60 italic leading-snug">
                        « {{ item.reponse_tresorerie }} »
                      </p>
                      <button 
                        @click="openDecisionModal(item, 'tresorerie')"
                        class="text-[11px] font-bold text-emerald-700 hover:underline cursor-pointer flex items-center gap-1 mt-1"
                      >
                        <i class="fa-solid fa-stamp text-[10px]"></i>
                        <span>Arbitrer (Trésorerie)</span>
                      </button>
                    </div>
                  </td>

                  <td class="p-4 align-top">
                    <div v-if="item.demande_sponsor">
                      <span v-if="item.statut_sponsor === 'sponsor_trouve'" class="px-2.5 py-1 rounded-md text-[10px] font-black bg-emerald-100 text-emerald-900 border border-emerald-300 inline-flex items-center gap-1 shadow-2xs">
                        <i class="fa-solid fa-circle-check text-emerald-600"></i>
                        <span>Sponsor trouvé</span>
                      </span>

                      <span v-else-if="item.statut_sponsor === 'en_recherche'" class="px-2.5 py-1 rounded-md text-[10px] font-bold bg-amber-50 text-amber-900 border border-amber-300 inline-flex items-center gap-1">
                        <i class="fa-solid fa-magnifying-glass text-amber-600 animate-pulse"></i>
                        <span>Recherche en cours</span>
                      </span>

                      <span v-else-if="item.statut_sponsor === 'non_finance'" class="px-2.5 py-1 rounded-md text-[10px] font-bold bg-slate-100 text-slate-700 border border-slate-300 inline-flex items-center gap-1">
                        <i class="fa-solid fa-triangle-exclamation text-amber-600"></i>
                        <span>Non sponsorisé</span>
                      </span>

                      <span v-else-if="item.statut_sponsor === 'refuse'" class="px-2.5 py-1 rounded-md text-[10px] font-bold bg-rose-50 text-rose-800 border border-rose-200 inline-flex items-center gap-1">
                        <i class="fa-solid fa-ban text-rose-500"></i>
                        <span>Refusé Sponsor</span>
                      </span>

                      <span v-else class="px-2.5 py-1 rounded-md text-[10px] font-bold bg-blue-50 text-blue-800 border border-blue-200 inline-flex items-center gap-1">
                        <i class="fa-regular fa-clock text-blue-500"></i>
                        <span>En attente</span>
                      </span>
                    </div>
                    <button 
                      v-else-if="canDelegateToSponsors"
                      @click="askSponsorsForMaterial(item)"
                      class="px-2.5 py-1 text-[11px] font-bold text-amber-800 bg-amber-50 hover:bg-amber-100 border border-amber-300 rounded-lg transition cursor-pointer flex items-center gap-1.5 shadow-2xs"
                      title="Déléguer ce besoin aux sponsors"
                    >
                      <i class="fa-solid fa-handshake"></i>
                      <span>Déléguer</span>
                    </button>
                    <span v-else class="text-slate-400 italic text-[11px]">Non délégué</span>
                  </td>

                  <td class="p-4 text-right whitespace-nowrap align-top">
                    <div class="flex items-center justify-end gap-1.5">
                      <button 
                        @click="openEditMaterialModal(item)"
                        class="text-slate-400 hover:text-emerald-700 p-1.5 rounded-lg hover:bg-emerald-50 transition cursor-pointer"
                        title="Modifier cette ligne"
                      >
                        <i class="fa-solid fa-pen text-xs"></i>
                      </button>
                      <button 
                        @click="deleteMaterialRequest(item.id)" 
                        class="text-slate-400 hover:text-rose-600 p-1.5 rounded-lg hover:bg-rose-50 transition cursor-pointer"
                        title="Supprimer la demande"
                      >
                        <i class="fa-solid fa-trash text-xs"></i>
                      </button>
                    </div>
                  </td>
                </tr>

                <tr v-if="filteredTresorerieRequests.length === 0">
                  <td colspan="7" class="p-10 text-center text-slate-400 italic text-xs">
                    Aucune ligne ne correspond aux filtres sélectionnés.
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <!-- ================= VUE ANNUAIRE PROSPECTION SPONSORS (ONGLET PRINCIPAL) ================= -->
      <div v-if="currentTab === 'annuaire_sponsors' && canAccessTab('annuaire_sponsors')" class="bg-white rounded-2xl border border-slate-200 shadow-xs p-6 space-y-4">
        <div class="flex items-center justify-between flex-wrap gap-4">
          <div>
            <h2 class="text-xl font-black text-slate-900 flex items-center gap-2">
              <i class="fa-solid fa-address-book text-emerald-600"></i>
              <span>Annuaire Prospection Entreprises ({{ filteredProspects.length }})</span>
            </h2>
            <p class="text-xs text-slate-500 mt-0.5">
              Suivi des démarches, contacts de l'association et statut des négociations.
            </p>
          </div>

          <div class="flex items-center gap-2.5 flex-wrap">
            <button
              @click="resetProspectFilters"
              class="px-3.5 py-2.5 bg-slate-100 hover:bg-slate-200 text-slate-700 font-bold text-xs rounded-xl transition cursor-pointer"
            >
              Réinitialiser les filtres
            </button>
            <button 
              @click="openCreateProspect" 
              class="px-4 py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold text-xs rounded-xl shadow-xs transition cursor-pointer flex items-center gap-2"
            >
              <i class="fa-solid fa-plus"></i>
              <span>Ajouter une entreprise</span>
            </button>
          </div>
        </div>

        <div class="overflow-x-auto">
          <table class="w-full text-left border-collapse text-xs">
            <thead>
              <tr class="bg-slate-50 border-b border-slate-200 text-slate-500 uppercase tracking-wider font-extrabold text-[10px]">
                <th class="p-3">
                  <div class="mb-1.5">Entreprise & Adresse</div>
                  <input v-model="prospectColFilters.entreprise" type="text" placeholder="Nom, ville, note..." class="w-full px-2.5 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                </th>
                <th class="p-3">
                  <div class="mb-1.5">Interlocuteur</div>
                  <input v-model="prospectColFilters.contact" type="text" placeholder="Filtrer nom..." class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                </th>
                <th class="p-3">
                  <div class="mb-1.5">Contact Asso</div>
                  <input v-model="prospectColFilters.contact_asso" type="text" placeholder="Membre asso..." class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                </th>
                <th class="p-3">
                  <div class="mb-1.5">Coordonnées</div>
                  <input v-model="prospectColFilters.coordonnees" type="text" placeholder="Tél ou email..." class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                </th>
                <th class="p-3">
                  <div class="mb-1.5">Responsable</div>
                  <input v-model="prospectColFilters.responsable" type="text" placeholder="Qui contacte..." class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                </th>
                <th class="p-3 w-36">
                  <div class="mb-1.5">Statut</div>
                  <select v-model="prospectColFilters.statut" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                    <option value="">Tous</option>
                    <option v-for="st in prospectStatusList" :key="st.id" :value="st.id">{{ st.label }}</option>
                  </select>
                </th>
                <th class="p-3 text-right w-20 align-top">Actions</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-slate-100">
              <tr v-for="item in filteredProspects" :key="item.id" class="hover:bg-slate-50/70 transition">
                <td class="p-3.5 align-top">
                  <div class="font-extrabold text-slate-900 text-sm">{{ item.nom_entreprise }}</div>
                  <div v-if="item.adresse_postale" class="text-[11px] text-slate-500 flex items-start gap-1 mt-0.5 max-w-xs">
                    <i class="fa-solid fa-location-dot text-slate-400 mt-0.5 text-[10px]"></i>
                    <span>{{ item.adresse_postale }}</span>
                  </div>
                  <p v-if="item.notes" class="text-[10px] text-slate-500 italic mt-1 bg-slate-50 p-1.5 rounded border border-slate-200/50">
                    Note : {{ item.notes }}
                  </p>
                </td>

                <td class="p-3.5 align-top">
                  <span v-if="item.nom_contact" class="font-bold text-slate-800">{{ item.nom_contact }}</span>
                  <span v-else class="text-slate-400 italic">Non spécifié</span>
                </td>

                <td class="p-3.5 align-top">
                  <span v-if="item.contact_asso" class="px-2 py-0.5 rounded-md bg-emerald-50 text-emerald-800 border border-emerald-200 font-bold text-[10px] inline-flex items-center gap-1">
                    <i class="fa-solid fa-user-tag text-[9px]"></i>
                    {{ item.contact_asso }}
                  </span>
                  <span v-else class="text-slate-400 italic">Aucun</span>
                </td>

                <td class="p-3.5 align-top space-y-1">
                  <div v-if="item.telephone">
                    <a :href="`tel:${item.telephone}`" class="text-slate-700 hover:text-emerald-700 font-medium flex items-center gap-1.5">
                      <i class="fa-solid fa-phone text-[10px] text-slate-400"></i>
                      <span>{{ item.telephone }}</span>
                    </a>
                  </div>
                  <div v-if="item.email">
                    <a :href="`mailto:${item.email}`" class="text-emerald-700 hover:underline font-medium flex items-center gap-1.5">
                      <i class="fa-regular fa-envelope text-[10px]"></i>
                      <span class="truncate max-w-[160px]">{{ item.email }}</span>
                    </a>
                  </div>
                  <div v-if="!item.telephone && !item.email" class="text-slate-400 italic">Non renseigné</div>
                </td>

                <td class="p-3.5 align-top">
                  <span v-if="item.responsable" class="font-semibold text-slate-800 text-[11px] bg-slate-100 px-2 py-0.5 rounded">
                    {{ item.responsable }}
                  </span>
                  <span v-else class="text-slate-400 italic">Non assigné</span>
                </td>

                <td class="p-3.5 align-top whitespace-nowrap">
                  <span :class="[prospectStatusList.find(s => s.id === item.statut)?.badge, 'px-2 py-0.5 rounded text-[10px] font-bold border inline-block']">
                    {{ prospectStatusList.find(s => s.id === item.statut)?.label || item.statut }}
                  </span>
                </td>

                <td class="p-3.5 text-right align-top whitespace-nowrap">
                  <div class="flex items-center justify-end gap-1.5">
                    <button @click="openEditProspect(item)" class="text-slate-400 hover:text-emerald-700 p-1.5 rounded-lg hover:bg-emerald-50 transition cursor-pointer" title="Modifier">
                      <i class="fa-solid fa-pen text-xs"></i>
                    </button>
                    <button @click="deleteProspect(item.id)" class="text-slate-400 hover:text-rose-600 p-1.5 rounded-lg hover:bg-rose-50 transition cursor-pointer" title="Supprimer">
                      <i class="fa-solid fa-trash text-xs"></i>
                    </button>
                  </div>
                </td>
              </tr>

              <tr v-if="filteredProspects.length === 0">
                <td colspan="7" class="p-8 text-center text-slate-400 italic text-xs">
                  Aucune entreprise ne correspond aux filtres sélectionnés.
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- ================= VUE BESOINS À SPONSORISER (ONGLET PRINCIPAL) ================= -->
      <div v-if="currentTab === 'besoins_sponsors' && canAccessTab('besoins_sponsors')" class="bg-white rounded-2xl border border-slate-200 p-6 shadow-xs space-y-4">
        <div class="flex items-center justify-between flex-wrap gap-4">
          <div>
            <h2 class="text-xl font-black text-slate-900 flex items-center gap-2">
              <i class="fa-solid fa-handshake text-emerald-600"></i>
              <span>Besoins matériels à faire sponsoriser ({{ filteredSponsorMaterialRequests.length }})</span>
            </h2>
            <p class="text-xs text-slate-500 mt-0.5">
              Acceptez la mission de prospection, puis confirmez dès qu'une entreprise finance le matériel.
            </p>
          </div>

          <div class="flex items-center gap-2.5 flex-wrap">
            <button
              @click="resetSponsorReqFilters"
              class="px-3.5 py-2.5 bg-slate-100 hover:bg-slate-200 text-slate-700 font-bold text-xs rounded-xl transition cursor-pointer"
            >
              Réinitialiser les filtres
            </button>

            <button 
              @click="exportToExcel('sponsor_besoins')" 
              class="px-3.5 py-2.5 bg-emerald-800 hover:bg-emerald-900 text-white font-bold text-xs rounded-xl shadow-2xs transition cursor-pointer flex items-center gap-2"
            >
              <i class="fa-solid fa-file-excel text-sm"></i>
              <span>Exporter Suivi Sponsoring (.csv)</span>
            </button>
          </div>
        </div>

        <div class="overflow-x-auto">
          <table class="w-full text-left border-collapse text-xs">
            <thead>
              <tr class="bg-slate-50 border-b border-slate-200 text-slate-500 uppercase tracking-wider font-extrabold text-[10px]">
                <th class="p-3 w-32">
                  <div class="mb-1.5">Pôle</div>
                  <select v-model="sponsorReqColFilters.pole" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                    <option value="">Tous</option>
                    <option v-for="p in allPoles" :key="p.id" :value="p.id">{{ p.label }}</option>
                  </select>
                </th>
                <th class="p-3">
                  <div class="mb-1.5">Matériel</div>
                  <input v-model="sponsorReqColFilters.article" type="text" placeholder="Rechercher matériel..." class="w-full px-2.5 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                </th>
                <th class="p-3">
                  <div class="mb-1.5">Justification</div>
                  <input v-model="sponsorReqColFilters.details" type="text" placeholder="Filtrer détails..." class="w-full px-2.5 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                </th>
                <th class="p-3 w-32">
                  <div class="mb-1.5">Coût estimé</div>
                  <input v-model="sponsorReqColFilters.cout" type="text" placeholder="Montant..." class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-normal text-slate-700 normal-case" />
                </th>
                <th class="p-3 w-44">
                  <div class="mb-1.5">Étape Sponsoring</div>
                  <select v-model="sponsorReqColFilters.statut" class="w-full px-2 py-1 rounded-lg border border-slate-300 bg-white text-[11px] font-medium text-slate-700 normal-case">
                    <option value="">Toutes</option>
                    <option value="en_attente">Nouveau besoin</option>
                    <option value="en_recherche">Recherche en cours</option>
                    <option value="sponsor_trouve">Sponsor trouvé</option>
                    <option value="non_finance">Non sponsorisé</option>
                    <option value="refuse">Refusé</option>
                  </select>
                </th>
                <th class="p-3 text-right align-top">Actions Sponsors</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-slate-100">
              <tr v-for="req in filteredSponsorMaterialRequests" :key="req.id" class="hover:bg-slate-50/70 transition">
                <td class="p-3 font-bold uppercase text-[11px] text-slate-700">{{ req.pole }}</td>
                <td class="p-3 font-extrabold text-slate-900">
                  {{ req.article }} <span class="text-slate-500 font-mono">(x{{ req.quantite }})</span>
                </td>
                <td class="p-3 text-slate-600 max-w-xs leading-relaxed">{{ req.details }}</td>
                <td class="p-3 font-mono font-bold text-emerald-800">
                  {{ req.estimated_price ? `${(req.estimated_price * req.quantite).toFixed(2)} €` : '—' }}
                </td>

                <td class="p-3">
                  <span v-if="req.statut_sponsor === 'sponsor_trouve'" class="px-2 py-0.5 rounded text-[10px] font-black bg-emerald-100 text-emerald-900 border border-emerald-300 inline-flex items-center gap-1">
                    <i class="fa-solid fa-circle-check text-emerald-600"></i>
                    <span>Sponsor trouvé (Validé)</span>
                  </span>

                  <span v-else-if="req.statut_sponsor === 'en_recherche'" class="px-2 py-0.5 rounded text-[10px] font-bold bg-amber-50 text-amber-900 border border-amber-300 inline-flex items-center gap-1">
                    <i class="fa-solid fa-magnifying-glass text-amber-600"></i>
                    <span>Recherche en cours</span>
                  </span>

                  <span v-else-if="req.statut_sponsor === 'non_finance'" class="px-2 py-0.5 rounded text-[10px] font-bold bg-slate-100 text-slate-700 border border-slate-300 inline-flex items-center gap-1">
                    <i class="fa-solid fa-triangle-exclamation text-amber-600"></i>
                    <span>Non sponsorisé</span>
                  </span>

                  <span v-else-if="req.statut_sponsor === 'refuse'" class="px-2 py-0.5 rounded text-[10px] font-bold bg-rose-50 text-rose-800 border border-rose-200 inline-flex items-center gap-1">
                    <i class="fa-solid fa-ban text-rose-500"></i>
                    <span>Refusé</span>
                  </span>

                  <span v-else class="px-2 py-0.5 rounded text-[10px] font-bold bg-blue-50 text-blue-800 border border-blue-200">
                    Nouveau besoin
                  </span>
                </td>

                <td class="p-3 text-right whitespace-nowrap">
                  <div v-if="req.statut_sponsor === 'en_attente'" class="flex items-center justify-end gap-1.5">
                    <button 
                      @click="handleSponsorResponse(req, 'accepte_recherche')"
                      class="px-2.5 py-1 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-lg transition cursor-pointer flex items-center gap-1"
                    >
                      <i class="fa-solid fa-check"></i>
                      <span>Chercher un sponsor</span>
                    </button>
                    <button 
                      @click="handleSponsorResponse(req, 'refuse')"
                      class="px-2.5 py-1 bg-rose-600 hover:bg-rose-700 text-white font-bold rounded-lg transition cursor-pointer flex items-center gap-1"
                    >
                      <i class="fa-solid fa-xmark"></i>
                      <span>Refuser</span>
                    </button>
                  </div>

                  <div v-else-if="req.statut_sponsor === 'en_recherche'" class="flex items-center justify-end gap-1.5">
                    <button 
                      @click="handleSponsorResponse(req, 'trouve')"
                      class="px-2.5 py-1 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-lg transition cursor-pointer flex items-center gap-1 shadow-2xs"
                    >
                      <i class="fa-solid fa-award"></i>
                      <span>Sponsor trouvé !</span>
                    </button>
                    <button 
                      @click="handleSponsorResponse(req, 'abandon')"
                      class="px-2 py-1 text-[11px] font-bold text-slate-600 bg-slate-100 hover:bg-slate-200 rounded-lg transition cursor-pointer"
                    >
                      <span>Abandonner</span>
                    </button>
                  </div>

                  <div v-else-if="req.statut_sponsor === 'sponsor_trouve'" class="flex items-center justify-end gap-2">
                    <span class="text-[11px] font-black text-emerald-700">Validé</span>
                    <button 
                      @click="handleSponsorResponse(req, 'abandon')"
                      class="text-[10px] text-slate-400 hover:text-rose-600 hover:underline cursor-pointer"
                    >
                      Désistement ?
                    </button>
                  </div>

                  <div v-else class="flex items-center justify-end gap-1.5">
                    <button 
                      @click="handleSponsorResponse(req, 'accepte_recherche')"
                      class="px-2 py-1 text-[11px] font-bold text-emerald-700 bg-emerald-50 hover:bg-emerald-100 rounded-lg transition cursor-pointer"
                    >
                      Retenter la recherche
                    </button>
                  </div>
                </td>
              </tr>

              <tr v-if="filteredSponsorMaterialRequests.length === 0">
                <td colspan="6" class="p-10 text-center text-slate-400 italic text-xs">
                  Aucun besoin matériel ne correspond aux filtres sélectionnés.
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- ================= VUE GESTION SITE (PRÉSIDENT / VICE-PRÉSIDENTE UNIQUEMENT) ================= -->
      <div v-if="currentTab === 'gestion_site' && canAccessTab('gestion_site')" class="space-y-8">
        
        <!-- Bloc 1 : Lien HelloAsso -->
        <div class="bg-white rounded-2xl border border-slate-200 p-6 shadow-xs">
          <h3 class="text-base font-black text-slate-900 flex items-center gap-2 mb-1">
            <i class="fa-solid fa-link text-emerald-600"></i>
            <span>Lien d'inscription HelloAsso</span>
          </h3>
          <p class="text-xs text-slate-500 mb-4">Ce lien est utilisé par les boutons « S'inscrire » de la section Épreuves.</p>

          <div class="flex flex-col sm:flex-row gap-3">
            <input
              v-model="helloAssoSetting"
              type="url"
              placeholder="https://www.helloasso.com/..."
              class="flex-1 px-4 py-2.5 rounded-xl border border-slate-300 text-xs font-mono outline-none focus:border-emerald-600"
            />
            <button
              @click="saveHelloAssoUrl"
              class="px-5 py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold text-xs rounded-xl transition cursor-pointer"
            >
              Enregistrer le lien
            </button>
          </div>
        </div>

        <!-- Bloc 2 : Organisation du Programme de la Journée -->
        <div class="bg-white rounded-2xl border border-slate-200 p-6 shadow-xs space-y-5">
          <div class="flex items-center justify-between flex-wrap gap-4">
            <div>
              <h3 class="text-lg font-black text-slate-900 flex items-center gap-2">
                <i class="fa-solid fa-calendar-day text-amber-500"></i>
                <span>Organisation du Programme de la Journée</span>
              </h3>
              <p class="text-xs text-slate-500 mt-0.5">
                Modifiez les horaires, les intitulés et réorganisez l'ordre d'apparition sur le site.
              </p>
            </div>

            <div class="flex items-center gap-3 flex-wrap">
              <div class="inline-flex p-1 bg-slate-100 rounded-xl border border-slate-200">
                <button
                  @click="programPeriodFilter = 'matin'"
                  :class="programPeriodFilter === 'matin' ? 'bg-white text-emerald-700 font-bold shadow-2xs' : 'text-slate-600'"
                  class="px-3.5 py-1.5 rounded-lg text-xs transition cursor-pointer"
                >
                  Matinée
                </button>
                <button
                  @click="programPeriodFilter = 'aprem'"
                  :class="programPeriodFilter === 'aprem' ? 'bg-white text-emerald-700 font-bold shadow-2xs' : 'text-slate-600'"
                  class="px-3.5 py-1.5 rounded-lg text-xs transition cursor-pointer"
                >
                  Après-midi
                </button>
              </div>

              <button
                @click="openCreateProgram"
                class="px-4 py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold text-xs rounded-xl shadow-xs transition cursor-pointer flex items-center gap-1.5"
              >
                <i class="fa-solid fa-plus"></i>
                <span>Ajouter un moment</span>
              </button>
            </div>
          </div>

          <div class="space-y-2.5">
            <div
              v-for="(item, index) in filteredProgramItems"
              :key="item.id"
              class="p-4 rounded-xl bg-slate-50 border border-slate-200 flex items-center justify-between gap-4 hover:bg-white transition"
            >
              <div class="flex items-center gap-3 sm:gap-4">
                <div class="flex flex-col gap-1">
                  <button
                    @click="moveProgramItem(index, -1)"
                    :disabled="index === 0"
                    class="w-6 h-6 rounded bg-white border border-slate-300 text-slate-600 hover:text-emerald-700 disabled:opacity-30 flex items-center justify-center cursor-pointer text-[10px]"
                    title="Monter"
                  >
                    <i class="fa-solid fa-chevron-up"></i>
                  </button>
                  <button
                    @click="moveProgramItem(index, 1)"
                    :disabled="index === filteredProgramItems.length - 1"
                    class="w-6 h-6 rounded bg-white border border-slate-300 text-slate-600 hover:text-emerald-700 disabled:opacity-30 flex items-center justify-center cursor-pointer text-[10px]"
                    title="Descendre"
                  >
                    <i class="fa-solid fa-chevron-down"></i>
                  </button>
                </div>

                <span class="font-mono font-black text-xs text-emerald-800 bg-emerald-100 px-2.5 py-1 rounded-lg w-20 sm:w-24 text-center flex-shrink-0">
                  {{ item.time_label }}
                </span>

                <div>
                  <div class="flex items-center gap-2 flex-wrap">
                    <span v-if="item.tag" class="text-[10px] uppercase font-bold bg-slate-700 text-white px-2 py-0.5 rounded">
                      {{ item.tag }}
                    </span>
                    <h4 class="font-bold text-slate-900 text-sm">{{ item.title }}</h4>
                  </div>
                  <p v-if="item.description" class="text-xs text-slate-500 mt-0.5">{{ item.description }}</p>
                </div>
              </div>

              <div class="flex items-center gap-1.5 flex-shrink-0">
                <button
                  @click="openEditProgram(item)"
                  class="p-2 text-slate-400 hover:text-emerald-700 rounded-lg hover:bg-emerald-50 cursor-pointer"
                  title="Modifier"
                >
                  <i class="fa-solid fa-pen text-xs"></i>
                </button>
                <button
                  @click="deleteProgramItem(item.id)"
                  class="p-2 text-slate-400 hover:text-rose-600 rounded-lg hover:bg-rose-50 cursor-pointer"
                  title="Supprimer"
                >
                  <i class="fa-solid fa-trash text-xs"></i>
                </button>
              </div>
            </div>

            <div v-if="filteredProgramItems.length === 0" class="text-center py-8 text-slate-400 text-xs italic">
              Aucun créneau enregistré dans la base pour cette période (les horaires par défaut s'affichent sur le site).
            </div>
          </div>
        </div>

        <!-- Bloc 3 : Organisation du Village Solidaire -->
        <div class="bg-white rounded-2xl border border-slate-200 p-6 shadow-xs space-y-6">
          <div class="flex items-center justify-between flex-wrap gap-4">
            <div>
              <h3 class="text-lg font-black text-slate-900 flex items-center gap-2">
                <i class="fa-solid fa-tent text-emerald-600"></i>
                <span>Organisation du Village Solidaire ({{ sortedVillageItems.length }})</span>
              </h3>
              <p class="text-xs text-slate-500 mt-0.5">
                Personnalisez la mise en page globale, ajoutez des logos en proportions réelles et réordonnez les stands.
              </p>
            </div>

            <button
              @click="openCreateVillage"
              class="px-4 py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold text-xs rounded-xl shadow-xs transition cursor-pointer flex items-center gap-1.5"
            >
              <i class="fa-solid fa-plus"></i>
              <span>Ajouter un stand / activité</span>
            </button>
          </div>

          <!-- PANNEAU DE CONTRÔLE DE L'ESPACE & DU VISUEL -->
          <div class="p-4 rounded-2xl bg-slate-50 border border-slate-200 space-y-4">
            <div class="flex items-center justify-between">
              <span class="text-xs font-black uppercase tracking-wider text-slate-700 flex items-center gap-1.5">
                <i class="fa-solid fa-sliders text-emerald-600"></i>
                <span>Paramètres d'espace & de mise en page</span>
              </span>
              <button
                @click="saveVillageLayoutConfig"
                class="px-3.5 py-1.5 bg-slate-900 hover:bg-slate-800 text-white font-bold text-[11px] rounded-lg transition cursor-pointer"
              >
                Appliquer le visuel
              </button>
            </div>

            <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-3">
              <div>
                <label class="block text-[10px] font-bold uppercase text-slate-500 mb-1">Répartition des colonnes</label>
                <select v-model="villageLayoutConfig.split" class="w-full px-3 py-2 rounded-xl border border-slate-300 text-xs font-bold bg-white">
                  <option value="50-50">50% Gauche / 50% Droite</option>
                  <option value="40-60">40% Gauche / 60% Droite</option>
                  <option value="stacked">Superposé (Pleine largeur)</option>
                </select>
              </div>

              <div>
                <label class="block text-[10px] font-bold uppercase text-slate-500 mb-1">Colonnes grille droite</label>
                <select v-model="villageLayoutConfig.rightCols" class="w-full px-3 py-2 rounded-xl border border-slate-300 text-xs font-bold bg-white">
                  <option value="2">2 colonnes (2×2)</option>
                  <option value="3">3 colonnes (3×2)</option>
                </select>
              </div>

              <div>
                <label class="block text-[10px] font-bold uppercase text-slate-500 mb-1">Espacement vertical</label>
                <select v-model="villageLayoutConfig.spacing" class="w-full px-3 py-2 rounded-xl border border-slate-300 text-xs font-bold bg-white">
                  <option value="compact">Compact</option>
                  <option value="normal">Équilibré (Défaut)</option>
                  <option value="relaxed">Aéré (Grand espace)</option>
                </select>
              </div>

              <div>
                <label class="block text-[10px] font-bold uppercase text-slate-500 mb-1">Écart entre les cartes</label>
                <select v-model="villageLayoutConfig.gapSize" class="w-full px-3 py-2 rounded-xl border border-slate-300 text-xs font-bold bg-white">
                  <option value="tight">Serré</option>
                  <option value="normal">Standard</option>
                  <option value="wide">Espacé</option>
                </select>
              </div>

              <div>
                <label class="block text-[10px] font-bold uppercase text-slate-500 mb-1">Style des cartes</label>
                <select v-model="villageLayoutConfig.cardStyle" class="w-full px-3 py-2 rounded-xl border border-slate-300 text-xs font-bold bg-white">
                  <option value="glass">Translucide (Verre)</option>
                  <option value="solid">Opaque contrasté</option>
                  <option value="bordered">Bordure émeraude marquée</option>
                </select>
              </div>

              <div>
                <label class="block text-[10px] font-bold uppercase text-slate-500 mb-1">Ambiance couleur</label>
                <select v-model="villageLayoutConfig.theme" class="w-full px-3 py-2 rounded-xl border border-slate-300 text-xs font-bold bg-white">
                  <option value="dark">Vert Nuit Sombre (#07291b)</option>
                  <option value="forest">Vert Forêt Dégradé</option>
                  <option value="light">Fond Clair Damier</option>
                </select>
              </div>
            </div>
          </div>

          <div class="space-y-2.5">
            <div
              v-for="(item, index) in sortedVillageItems"
              :key="item.id"
              class="p-4 rounded-xl bg-slate-50 border border-slate-200 flex items-center justify-between gap-4 hover:bg-white transition"
            >
              <div class="flex items-center gap-3 sm:gap-4">
                <div class="flex flex-col gap-1">
                  <button
                    @click="moveVillageItem(index, -1)"
                    :disabled="index === 0"
                    class="w-6 h-6 rounded bg-white border border-slate-300 text-slate-600 hover:text-emerald-700 disabled:opacity-30 flex items-center justify-center cursor-pointer text-[10px]"
                    title="Monter"
                  >
                    <i class="fa-solid fa-chevron-up"></i>
                  </button>
                  <button
                    @click="moveVillageItem(index, 1)"
                    :disabled="index === sortedVillageItems.length - 1"
                    class="w-6 h-6 rounded bg-white border border-slate-300 text-slate-600 hover:text-emerald-700 disabled:opacity-30 flex items-center justify-center cursor-pointer text-[10px]"
                    title="Descendre"
                  >
                    <i class="fa-solid fa-chevron-down"></i>
                  </button>
                </div>

                <div class="h-12 min-w-[48px] max-w-[96px] px-1.5 rounded-xl bg-white text-emerald-800 flex items-center justify-center flex-shrink-0 overflow-hidden border border-slate-200 shadow-2xs">
                  <img
                    v-if="item.image_url"
                    :src="item.image_url"
                    :alt="item.title"
                    :class="item.logo_fit === 'cover' ? 'w-12 h-12 object-cover rounded-lg' : 'max-h-10 max-w-full w-auto h-auto object-contain'"
                  />
                  <i v-else :class="['fa-solid', item.icon || 'fa-tent', 'text-lg']"></i>
                </div>

                <div>
                  <div class="flex items-center gap-2 flex-wrap">
                    <h4 class="font-bold text-slate-900 text-sm">{{ item.title }}</h4>
                    <span
                      class="text-[10px] uppercase font-extrabold px-2 py-0.5 rounded border"
                      :class="item.layout_zone === 'left'
                        ? 'bg-amber-50 text-amber-800 border-amber-200'
                        : item.layout_zone === 'right'
                          ? 'bg-sky-50 text-sky-800 border-sky-200'
                          : 'bg-slate-200 text-slate-700 border-slate-300'"
                    >
                      {{ item.layout_zone === 'left' ? 'Bandeau Gauche' : item.layout_zone === 'right' ? 'Grille Droite' : 'Auto' }}
                    </span>
                    <span v-if="item.is_wide" class="text-[10px] uppercase font-bold px-2 py-0.5 rounded bg-purple-50 text-purple-800 border border-purple-200">
                      Carte Large (2 col.)
                    </span>
                  </div>
                  <p v-if="item.description" class="text-xs text-slate-500 mt-0.5 line-clamp-2">{{ item.description }}</p>
                </div>
              </div>

              <div class="flex items-center gap-1.5 flex-shrink-0">
                <button
                  @click="openEditVillage(item)"
                  class="p-2 text-slate-400 hover:text-emerald-700 rounded-lg hover:bg-emerald-50 cursor-pointer"
                  title="Modifier"
                >
                  <i class="fa-solid fa-pen text-xs"></i>
                </button>
                <button
                  @click="deleteVillageItem(item.id)"
                  class="p-2 text-slate-400 hover:text-rose-600 rounded-lg hover:bg-rose-50 cursor-pointer"
                  title="Supprimer"
                >
                  <i class="fa-solid fa-trash text-xs"></i>
                </button>
              </div>
            </div>

            <div v-if="sortedVillageItems.length === 0" class="text-center py-8 text-slate-400 text-xs italic">
              Aucun stand enregistré en base (les éléments par défaut s'affichent sur le site).
            </div>
          </div>
        </div>

        <!-- Bloc 4 : Sponsors officiels affichés sur le site -->
        <div class="bg-white rounded-2xl border border-slate-200 p-6 shadow-xs space-y-5">
          <div class="flex justify-between items-center flex-wrap gap-4">
            <div>
              <h3 class="text-lg font-black text-slate-900 flex items-center gap-2">
                <i class="fa-solid fa-award text-amber-500"></i>
                <span>Sponsors officiels affichés sur le site ({{ sponsors.length }})</span>
              </h3>
              <p class="text-xs text-slate-500 mt-0.5">Ces logos et descriptions apparaissent directement sur la page d'accueil.</p>
            </div>
            <button 
              @click="openCreateSponsor" 
              class="px-4 py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold text-xs rounded-xl shadow-xs transition cursor-pointer flex items-center gap-1.5"
            >
              <i class="fa-solid fa-plus"></i>
              <span>Ajouter un sponsor</span>
            </button>
          </div>

          <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
            <div 
              v-for="s in sponsors" 
              :key="s.id" 
              class="bg-slate-50 rounded-2xl border border-slate-200 p-5 flex flex-col justify-between"
            >
              <div class="flex items-start gap-4 mb-4">
                <div class="w-16 h-16 rounded-xl bg-white border border-slate-200 flex items-center justify-center p-2 flex-shrink-0">
                  <img v-if="s.logo_url" :src="s.logo_url" :alt="s.nom" class="max-h-full max-w-full object-contain" />
                  <i v-else class="fa-solid fa-handshake text-xl text-slate-300"></i>
                </div>
                <div class="overflow-hidden">
                  <h4 class="font-bold text-slate-900 text-sm truncate">{{ s.nom }}</h4>
                  <p class="text-xs text-slate-500 line-clamp-2 mt-0.5">{{ s.description || 'Aucune description' }}</p>
                </div>
              </div>

              <div class="flex items-center justify-end gap-2 pt-3 border-t border-slate-200/70">
                <button 
                  @click="openEditSponsor(s)" 
                  class="text-xs font-semibold text-slate-700 hover:text-emerald-700 bg-white border border-slate-200 hover:bg-emerald-50 px-3 py-1.5 rounded-lg transition cursor-pointer flex items-center gap-1"
                >
                  <i class="fa-solid fa-pen text-[10px]"></i>
                  <span>Modifier</span>
                </button>
                <button 
                  @click="deleteSponsor(s.id)" 
                  class="text-xs font-semibold text-rose-600 hover:bg-rose-50 px-3 py-1.5 rounded-lg transition border border-rose-200 bg-white cursor-pointer flex items-center gap-1"
                >
                  <i class="fa-solid fa-trash text-[10px]"></i>
                  <span>Supprimer</span>
                </button>
              </div>
            </div>
          </div>

          <div v-if="sponsors.length === 0" class="text-center py-8 text-slate-400 text-xs italic">
            Aucun sponsor officiel enregistré sur le site pour l'instant.
          </div>
        </div>

      </div>

      <!-- ================= VUE MESSAGES ================= -->
      <div v-if="currentTab === 'messages' && canAccessTab('messages')" class="grid grid-cols-1 lg:grid-cols-12 gap-6 items-start">
        <div class="lg:col-span-5 bg-white rounded-2xl border border-slate-200 divide-y divide-slate-100 max-h-[620px] overflow-y-auto shadow-xs">
          <div 
            v-for="msg in messages" 
            :key="msg.id"
            @click="selectMsg(msg)"
            :class="[
              selectedMessage?.id === msg.id ? 'bg-emerald-50/70 border-l-4 border-emerald-600' : 'hover:bg-slate-50',
              !msg.read ? 'font-bold bg-white' : 'text-slate-600'
            ]"
            class="p-4 cursor-pointer transition flex items-start justify-between gap-3"
          >
            <div class="overflow-hidden space-y-1">
              <div class="flex items-center gap-2">
                <span v-if="!msg.read" class="w-2 h-2 rounded-full bg-emerald-500 flex-shrink-0"></span>
                <span class="text-sm truncate text-slate-900 font-bold">{{ msg.nom }}</span>
              </div>
              <div class="text-xs text-emerald-700 font-medium truncate">{{ msg.email }}</div>
              <div class="text-xs text-slate-500 truncate">{{ msg.message }}</div>
            </div>
            <span class="text-[10px] text-slate-400 flex-shrink-0">
              {{ new Date(msg.created_at).toLocaleDateString('fr-FR', { day: '2-digit', month: '2-digit' }) }}
            </span>
          </div>
          <div v-if="messages.length === 0" class="p-8 text-center text-slate-400 text-xs">
            Aucun message reçu.
          </div>
        </div>

        <div class="lg:col-span-7 bg-white rounded-2xl border border-slate-200 p-6 shadow-xs min-h-[380px]">
          <template v-if="selectedMessage">
            <div class="flex flex-col sm:flex-row sm:items-start justify-between border-b border-slate-100 pb-4 mb-4 gap-4">
              <div>
                <span class="text-[10px] uppercase font-bold tracking-wider px-2 py-0.5 bg-emerald-100 text-emerald-800 rounded">
                  {{ selectedMessage.sujet || 'Message' }}
                </span>
                <h3 class="text-xl font-black text-slate-900 mt-2">{{ selectedMessage.nom }}</h3>
                <p class="text-sm text-slate-600 mt-1">
                  <a :href="`mailto:${selectedMessage.email}`" class="text-emerald-700 hover:underline font-semibold">
                    {{ selectedMessage.email }}
                  </a>
                </p>
              </div>
              <div class="flex gap-2">
                <button 
                  @click="openFaqModalFromMessage(selectedMessage)"
                  class="text-xs font-bold text-emerald-800 bg-emerald-50 border border-emerald-300 hover:bg-emerald-100 px-3 py-2 rounded-xl transition cursor-pointer flex items-center gap-1.5"
                >
                  <i class="fa-solid fa-circle-question"></i>
                  <span>FAQ</span>
                </button>
                <button 
                  @click="deleteMsg(selectedMessage.id)"
                  class="text-xs font-semibold text-rose-600 hover:bg-rose-50 px-3 py-2 rounded-xl transition border border-rose-200 cursor-pointer"
                >
                  <i class="fa-solid fa-trash"></i>
                </button>
              </div>
            </div>
            <div class="text-sm text-slate-700 whitespace-pre-wrap leading-relaxed bg-slate-50 p-5 rounded-2xl border border-slate-100">
              {{ selectedMessage.message }}
            </div>
            <div class="mt-6 pt-4 border-t border-slate-100 flex justify-end">
              <a 
                :href="`mailto:${selectedMessage.email}?subject=Re: Course Pas d'oubli`"
                class="px-5 py-2.5 bg-emerald-600 hover:bg-emerald-700 text-white font-bold text-xs rounded-xl transition flex items-center gap-2"
              >
                <i class="fa-solid fa-reply"></i>
                <span>Répondre par e-mail</span>
              </a>
            </div>
          </template>
          <div v-else class="h-64 flex flex-col items-center justify-center text-slate-400 text-center">
            <i class="fa-regular fa-envelope-open text-4xl mb-2 text-slate-300"></i>
            <p class="text-xs">Sélectionnez un message dans la colonne de gauche.</p>
          </div>
        </div>
      </div>

      <!-- ================= VUE BÉNÉVOLES ================= -->
      <div v-if="currentTab === 'benevoles' && canAccessTab('benevoles')" class="bg-white rounded-2xl border border-slate-200 divide-y divide-slate-100 shadow-xs overflow-hidden">
        <div 
          v-for="b in volunteers" 
          :key="b.id" 
          class="p-5 flex flex-col lg:flex-row lg:items-center justify-between gap-4 hover:bg-slate-50 transition"
        >
          <div class="space-y-2">
            <div class="flex items-center gap-3 flex-wrap">
              <span class="font-bold text-slate-900 text-base">{{ b.prenom }} {{ b.nom }}</span>
              <span class="text-[10px] uppercase font-bold tracking-wider px-2.5 py-0.5 bg-emerald-50 text-emerald-800 border border-emerald-200 rounded-full">
                {{ availabilityLabels[b.disponibilites] || b.disponibilites }}
              </span>
              <span class="text-xs text-slate-400">
                {{ new Date(b.created_at).toLocaleDateString('fr-FR') }}
              </span>
            </div>

            <div class="flex flex-wrap gap-1.5 pt-0.5">
              <span 
                v-for="pole in b.poles" 
                :key="pole" 
                class="text-xs font-semibold px-2.5 py-1 rounded-lg bg-slate-100 text-slate-700 border border-slate-200"
              >
                <i class="fa-solid fa-check text-emerald-600 mr-1 text-[10px]"></i>
                {{ poleLabels[pole] || pole }}
              </span>
            </div>

            <div class="flex items-center gap-4 text-xs text-slate-600 pt-1 flex-wrap">
              <a :href="`mailto:${b.email}`" class="text-emerald-700 hover:underline flex items-center gap-1.5 font-medium">
                <i class="fa-regular fa-envelope"></i>{{ b.email }}
              </a>
              <a :href="`tel:${b.telephone}`" class="text-slate-700 hover:underline flex items-center gap-1.5 font-medium">
                <i class="fa-solid fa-phone"></i>{{ b.telephone }}
              </a>
            </div>

            <p v-if="b.remarques" class="text-xs text-slate-600 italic bg-slate-50 border border-slate-100 p-2.5 rounded-xl max-w-2xl mt-1">
              « {{ b.remarques }} »
            </p>
          </div>

          <div class="flex items-center gap-2 self-end lg:self-center">
            <a 
              :href="`mailto:${b.email}?subject=Association Pas d'oubli - Candidature bénévole`"
              class="px-3.5 py-2 bg-emerald-600 hover:bg-emerald-700 text-white font-bold text-xs rounded-xl transition flex items-center gap-1.5"
            >
              <i class="fa-solid fa-reply"></i>
              <span>Contacter</span>
            </a>
            <button 
              @click="deleteVolunteer(b.id)" 
              class="p-2 text-slate-400 hover:text-rose-600 rounded-xl hover:bg-rose-50 transition cursor-pointer"
            >
              <i class="fa-solid fa-trash text-sm"></i>
            </button>
          </div>
        </div>

        <div v-if="volunteers.length === 0" class="p-12 text-center text-slate-400 text-xs">
          Aucune candidature bénévole enregistrée.
        </div>
      </div>

      <!-- ================= VUE CAGNOTTE ================= -->
      <div v-if="currentTab === 'cagnotte' && canAccessTab('cagnotte')" class="bg-white rounded-2xl border border-slate-200 p-6 sm:p-8 shadow-xs max-w-xl">
        <div class="mb-6">
          <h3 class="text-lg font-black text-slate-900 flex items-center gap-2">
            <i class="fa-solid fa-hand-holding-dollar text-emerald-600"></i>
            Actualisation de la cagnotte en direct
          </h3>
          <p class="text-xs text-slate-500 mt-1">
            Les changements modifient directement la jauge affichée aux visiteurs sur le site.
          </p>
        </div>

        <div class="space-y-4">
          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Montant récolté (€)</label>
            <input 
              v-model.number="fundraisingCurrent" 
              type="number" 
              min="0"
              class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm font-mono font-bold text-slate-900 outline-none focus:border-emerald-600"
            />
          </div>

          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Objectif fixé (€)</label>
            <input 
              v-model.number="fundraisingGoal" 
              type="number" 
              min="1"
              class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm font-mono font-bold text-slate-900 outline-none focus:border-emerald-600"
            />
          </div>

          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Nombre de participants inscrits</label>
            <input 
              v-model.number="fundraisingParticipants" 
              type="number" 
              min="0"
              placeholder="Ex. 142"
              class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm font-mono font-bold text-slate-900 outline-none focus:border-emerald-600"
            />
          </div>

          <button 
            @click="handleSaveFundraising" 
            :disabled="updatingFundraising"
            class="w-full py-3 bg-emerald-600 hover:bg-emerald-700 text-white font-bold rounded-xl text-sm transition cursor-pointer flex items-center justify-center gap-2"
          >
            <i v-if="updatingFundraising" class="fa-solid fa-circle-notch fa-spin"></i>
            <span>Mettre à jour la cagnotte</span>
          </button>
        </div>
      </div>

      <!-- ================= VUE FAQ ================= -->
      <div v-if="currentTab === 'faq' && canAccessTab('faq')" class="space-y-4">
        <div class="flex justify-between items-center">
          <p class="text-xs text-slate-500">Gérez les questions fréquentes affichées sur la page d'accueil.</p>
          <button 
            @click="openManualFaqModal" 
            class="px-4 py-2.5 bg-emerald-700 hover:bg-emerald-800 text-white font-bold text-xs rounded-xl shadow-xs transition cursor-pointer flex items-center gap-1.5"
          >
            <i class="fa-solid fa-plus"></i>
            <span>Nouvelle question</span>
          </button>
        </div>

        <div class="bg-white rounded-2xl border border-slate-200 divide-y divide-slate-100 shadow-xs">
          <div v-for="item in faqs" :key="item.id" class="p-5 flex items-start justify-between gap-4">
            <div class="space-y-1.5">
              <h4 class="font-bold text-slate-900 text-sm flex items-center gap-2">
                <span class="w-2 h-2 rounded-full bg-emerald-500"></span>
                {{ item.question }}
              </h4>
              <p class="text-xs text-slate-600 leading-relaxed pl-4">{{ item.reponse }}</p>
            </div>
            <button @click="deleteFaq(item.id)" class="text-slate-400 hover:text-rose-600 p-2 text-sm transition cursor-pointer">
              <i class="fa-solid fa-trash"></i>
            </button>
          </div>
          <div v-if="faqs.length === 0" class="p-8 text-center text-slate-400 text-xs">
            La FAQ est vide.
          </div>
        </div>
      </div>

    </div>
  </div>

  <!-- MODALE AJOUT / MODIFICATION VILLAGE SOLIDAIRE -->
  <div v-if="showVillageModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-lg w-full p-6 shadow-2xl relative max-h-[90vh] overflow-y-auto">
      <button @click="showVillageModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 cursor-pointer">
        <i class="fa-solid fa-xmark text-lg"></i>
      </button>

      <h3 class="text-lg font-black text-slate-900 mb-4">
        {{ editingVillageItem ? 'Modifier ce stand / activité' : 'Ajouter au Village Solidaire' }}
      </h3>

      <form @submit.prevent="saveVillageItem" class="space-y-4">
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Titre *</label>
          <input v-model="villageForm.title" required type="text" placeholder="Ex. Stand France Alzheimer 37" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs" />
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Emplacement dans la section</label>
          <div class="grid grid-cols-3 gap-2">
            <button
              type="button"
              @click="villageForm.layout_zone = 'left'"
              :class="villageForm.layout_zone === 'left' ? 'bg-emerald-700 text-white font-bold border-emerald-700' : 'bg-slate-50 text-slate-700 border-slate-200'"
              class="py-2 px-2 text-[11px] rounded-xl border transition cursor-pointer text-center"
            >
              Bandeau Gauche
            </button>
            <button
              type="button"
              @click="villageForm.layout_zone = 'right'"
              :class="villageForm.layout_zone === 'right' ? 'bg-emerald-700 text-white font-bold border-emerald-700' : 'bg-slate-50 text-slate-700 border-slate-200'"
              class="py-2 px-2 text-[11px] rounded-xl border transition cursor-pointer text-center"
            >
              Grille Droite
            </button>
            <button
              type="button"
              @click="villageForm.layout_zone = 'auto'"
              :class="villageForm.layout_zone === 'auto' ? 'bg-emerald-700 text-white font-bold border-emerald-700' : 'bg-slate-50 text-slate-700 border-slate-200'"
              class="py-2 px-2 text-[11px] rounded-xl border transition cursor-pointer text-center"
            >
              Automatique
            </button>
          </div>

          <label v-if="villageForm.layout_zone === 'right'" class="flex items-center gap-2 mt-2 text-xs text-slate-700 cursor-pointer select-none">
            <input type="checkbox" v-model="villageForm.is_wide" class="rounded text-emerald-600 focus:ring-emerald-500" />
            <span>Étendre cette carte sur toute la largeur de la grille (2 colonnes)</span>
          </label>
        </div>

        <div class="p-3.5 rounded-2xl bg-slate-50 border border-slate-200 space-y-3">
          <label class="block text-xs font-bold uppercase text-slate-700">Logo ou Image (optionnel)</label>
          
          <div class="flex items-center gap-3">
            <div class="h-14 min-w-[56px] max-w-[110px] px-2 rounded-xl bg-white border border-slate-300 flex items-center justify-center overflow-hidden flex-shrink-0">
              <img
                v-if="villageForm.image_url"
                :src="villageForm.image_url"
                alt="Aperçu"
                :class="villageForm.logo_fit === 'cover' ? 'w-14 h-14 object-cover' : 'max-h-12 max-w-full w-auto h-auto object-contain'"
              />
              <i v-else :class="['fa-solid', villageForm.icon, 'text-slate-400 text-lg']"></i>
            </div>

            <div class="flex-1 space-y-1.5">
              <input
                v-model="villageForm.image_url"
                type="url"
                placeholder="Coller le lien d'un logo (https://...)"
                class="w-full px-3 py-1.5 rounded-lg border border-slate-300 text-xs bg-white"
              />
              <label class="inline-flex items-center gap-1.5 text-[11px] font-bold text-emerald-700 hover:underline cursor-pointer">
                <i class="fa-solid fa-upload"></i>
                <span>{{ uploadingVillageImg ? 'Importation...' : 'Importer un fichier logo (PNG, SVG, JPG)' }}</span>
                <input type="file" accept="image/*" @change="handleVillageImageUpload" class="hidden" />
              </label>
            </div>

            <button
              v-if="villageForm.image_url"
              type="button"
              @click="villageForm.image_url = ''"
              class="text-xs text-rose-600 hover:underline cursor-pointer"
              title="Retirer le logo"
            >
              <i class="fa-solid fa-xmark"></i>
            </button>
          </div>

          <div v-if="villageForm.image_url" class="grid grid-cols-3 gap-2 pt-2 border-t border-slate-200">
            <div>
              <label class="block text-[10px] font-bold uppercase text-slate-500 mb-1">Proportions</label>
              <select v-model="villageForm.logo_fit" class="w-full px-2 py-1.5 rounded-lg border border-slate-300 text-xs font-bold bg-white">
                <option value="original">Taille d'origine</option>
                <option value="contain">Ajusté (non rogné)</option>
                <option value="cover">Carré rogné</option>
              </select>
            </div>

            <div>
              <label class="block text-[10px] font-bold uppercase text-slate-500 mb-1">Taille max</label>
              <select v-model="villageForm.logo_size" class="w-full px-2 py-1.5 rounded-lg border border-slate-300 text-xs font-bold bg-white">
                <option value="sm">Petit</option>
                <option value="md">Moyen</option>
                <option value="lg">Grand</option>
                <option value="xl">Très grand</option>
              </select>
            </div>

            <div>
              <label class="block text-[10px] font-bold uppercase text-slate-500 mb-1">Fond du logo</label>
              <select v-model="villageForm.logo_bg" class="w-full px-2 py-1.5 rounded-lg border border-slate-300 text-xs font-bold bg-white">
                <option value="white">Cadre blanc</option>
                <option value="transparent">Transparent</option>
              </select>
            </div>
          </div>
        </div>

        <div class="grid grid-cols-2 gap-3">
          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Couleur du carré icône</label>
            <select v-model="villageForm.color_theme" class="w-full px-3 py-2.5 rounded-xl border border-slate-300 text-xs font-bold bg-white">
              <option value="amber">Jaune / Ambre</option>
              <option value="teal">Vert Menthe / Turquoise</option>
              <option value="emerald">Vert Émeraude</option>
              <option value="sky">Bleu Ciel</option>
            </select>
          </div>

          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Icône (si pas de logo)</label>
            <select v-model="villageForm.icon" class="w-full px-3 py-2.5 rounded-xl border border-slate-300 text-xs font-bold bg-white">
              <option v-for="ic in villageIconOptions" :key="ic.id" :value="ic.id">{{ ic.label }}</option>
            </select>
          </div>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Description</label>
          <textarea v-model="villageForm.description" rows="3" placeholder="Présentez brièvement l'activité ou le stand..." class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs"></textarea>
        </div>

        <button type="submit" class="w-full py-3 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-xl text-xs transition cursor-pointer mt-2">
          {{ editingVillageItem ? 'Enregistrer' : 'Ajouter au village' }}
        </button>
      </form>
    </div>
  </div>

  <!-- MODALE AJOUT / MODIFICATION MOMENT PROGRAMME -->
  <div v-if="showProgramModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-md w-full p-6 shadow-2xl relative">
      <button @click="showProgramModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 cursor-pointer">
        <i class="fa-solid fa-xmark text-lg"></i>
      </button>

      <h3 class="text-lg font-black text-slate-900 mb-4">
        {{ editingProgramItem ? 'Modifier ce moment' : 'Ajouter un moment au programme' }}
      </h3>

      <form @submit.prevent="saveProgramItem" class="space-y-3.5">
        <div class="grid grid-cols-2 gap-3">
          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Période</label>
            <select v-model="programForm.periode" class="w-full px-3 py-2.5 rounded-xl border border-slate-300 text-xs font-bold bg-white">
              <option value="matin">Matinée</option>
              <option value="aprem">Après-midi</option>
            </select>
          </div>
          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Horaire *</label>
            <input v-model="programForm.time_label" required type="text" placeholder="Ex. 09h45 ou à définir" class="w-full px-3 py-2.5 rounded-xl border border-slate-300 text-xs font-mono" />
          </div>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Badge / Tag (optionnel)</label>
          <input v-model="programForm.tag" type="text" placeholder="Ex. Départ, Convivialité, Conférence..." class="w-full px-3 py-2.5 rounded-xl border border-slate-300 text-xs" />
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Titre du moment *</label>
          <input v-model="programForm.title" required type="text" placeholder="Ex. Départ Course 10 km" class="w-full px-3 py-2.5 rounded-xl border border-slate-300 text-xs" />
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Sous-titre court (optionnel)</label>
          <input v-model="programForm.description" type="text" placeholder="Ex. Épreuve chronométrée" class="w-full px-3 py-2.5 rounded-xl border border-slate-300 text-xs" />
        </div>

        <button type="submit" class="w-full py-3 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-xl text-xs transition cursor-pointer mt-2">
          {{ editingProgramItem ? 'Enregistrer' : 'Ajouter au programme' }}
        </button>
      </form>
    </div>
  </div>

  <!-- MODALE : CHANGER DE COMPTE -->
  <div v-if="showSwitchAccountModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-md w-full p-6 sm:p-8 shadow-2xl relative">
      <button @click="showSwitchAccountModal = false" class="absolute top-5 right-5 text-slate-400 hover:text-slate-600 cursor-pointer text-lg">
        <i class="fa-solid fa-xmark"></i>
      </button>

      <div class="text-center mb-6">
        <div class="w-12 h-12 bg-emerald-100 text-emerald-800 rounded-2xl flex items-center justify-center mx-auto text-xl mb-2">
          <i class="fa-solid fa-users-gear"></i>
        </div>
        <h3 class="text-xl font-black text-slate-900">Changer de Compte</h3>
        <p class="text-xs text-slate-500 mt-1">Connectez-vous à un autre pôle ou à la direction.</p>
      </div>

      <form @submit.prevent="handleLogin" class="space-y-4">
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Nouvel e-mail</label>
          <input 
            v-model="loginEmail" 
            required 
            type="email" 
            placeholder="ex: logistique@pasdoubli.fr" 
            class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm focus:border-emerald-600 outline-none" 
          />
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Mot de passe</label>
          <input 
            v-model="loginPassword" 
            required 
            type="password" 
            placeholder="••••••••" 
            class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm focus:border-emerald-600 outline-none" 
          />
        </div>

        <p v-if="loginError" class="text-xs text-rose-600 font-semibold text-center">{{ loginError }}</p>

        <button 
          type="submit" 
          class="w-full py-3 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-xl text-sm transition cursor-pointer shadow-md"
        >
          Valider le changement
        </button>
      </form>
    </div>
  </div>

  <!-- MODALE : CRÉATION / ÉDITION DE POST-IT -->
  <div v-if="showTaskModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-md w-full p-6 shadow-2xl relative">
      <button @click="showTaskModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 cursor-pointer">
        <i class="fa-solid fa-xmark text-lg"></i>
      </button>

      <h3 class="text-lg font-black text-slate-900 mb-1">
        {{ editingTask ? 'Modifier le Post-it' : 'Nouveau Post-it' }}
      </h3>
      <p class="text-xs text-slate-500 mb-4">Pôle : <strong class="capitalize text-emerald-700">{{ selectedPole }}</strong></p>

      <div class="space-y-3.5">
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Titre de l'action *</label>
          <input v-model="taskForm.titre" required type="text" placeholder="Ex. Relancer la mairie pour le tracé" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-sm outline-none focus:border-emerald-600" />
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">En charge</label>
          <input v-model="taskForm.assigne_a" type="text" placeholder="Ex. Thomas, Lucie..." class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-sm outline-none focus:border-emerald-600" />
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Colonne</label>
          <select v-model="taskForm.status" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-sm outline-none focus:border-emerald-600 bg-white">
            <option v-for="c in columns" :key="c.id" :value="c.id">{{ c.label }}</option>
          </select>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1.5">Couleur du contour</label>
          <div class="flex items-center gap-2 flex-wrap">
            <button
              v-for="color in postItColorOptions"
              :key="color.id"
              type="button"
              @click="taskForm.border_color = color.id"
              :class="[
                color.colorClass,
                taskForm.border_color === color.id ? 'ring-2 ring-emerald-600 ring-offset-2 scale-110' : 'opacity-70 hover:opacity-100',
                'w-7 h-7 rounded-full border-2 transition transform cursor-pointer'
              ]"
              :title="color.label"
            ></button>
          </div>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Description</label>
          <textarea v-model="taskForm.description" rows="3" placeholder="Détails du sujet..." class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-sm outline-none focus:border-emerald-600"></textarea>
        </div>

        <button 
          @click="saveTask" 
          :disabled="taskSaving"
          class="w-full py-3 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-xl text-sm transition cursor-pointer mt-2 flex items-center justify-center gap-2 shadow-sm"
        >
          <i v-if="taskSaving" class="fa-solid fa-circle-notch fa-spin"></i>
          <span>{{ editingTask ? 'Mettre à jour le post-it' : 'Épingler le post-it' }}</span>
        </button>
      </div>
    </div>
  </div>

  <!-- MODALE DÉTAIL POST-IT -->
  <div v-if="showDetailModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-xl w-full p-6 sm:p-7 shadow-2xl relative flex flex-col max-h-[90vh]">
      <button @click="showDetailModal = false" class="absolute top-5 right-5 text-slate-400 hover:text-slate-600 cursor-pointer">
        <i class="fa-solid fa-xmark text-lg"></i>
      </button>

      <div class="border-b border-slate-100 pb-3 mb-4 pr-6">
        <span class="text-[10px] font-black uppercase tracking-wider px-2 py-0.5 bg-emerald-100 text-emerald-800 rounded">
          Pôle {{ currentTaskDetail?.pole }}
        </span>
        <h3 class="text-xl font-black text-slate-900 mt-2">{{ currentTaskDetail?.titre }}</h3>
        <p v-if="currentTaskDetail?.assigne_a" class="text-xs text-slate-500 mt-0.5">
          En charge : <strong class="text-slate-800">{{ currentTaskDetail?.assigne_a }}</strong>
        </p>
        <p v-if="currentTaskDetail?.description" class="text-xs text-slate-600 bg-slate-50 p-2.5 rounded-xl mt-2 border border-slate-100">
          {{ currentTaskDetail?.description }}
        </p>
      </div>

      <div class="flex-1 overflow-y-auto space-y-6 pr-1">
        <div>
          <div class="flex items-center justify-between mb-2">
            <h4 class="text-xs font-black uppercase text-slate-700 tracking-wider flex items-center gap-1.5">
              <i class="fa-solid fa-list-check text-emerald-600"></i>
              Étapes d'avancement ({{ currentTaskSteps.filter(s => s.completed).length }}/{{ currentTaskSteps.length }})
            </h4>
          </div>

          <div class="flex gap-2 mb-3">
            <input 
              v-model="newStepTitle" 
              @keyup.enter="addStep"
              type="text" 
              placeholder="Ajouter une étape au post-it (ex: Appel validé)..." 
              class="flex-1 px-3 py-2 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600"
            />
            <button 
              @click="addStep" 
              class="px-3.5 py-2 bg-slate-900 hover:bg-slate-800 text-white font-bold text-xs rounded-xl transition cursor-pointer"
            >
              Ajouter
            </button>
          </div>

          <div class="space-y-1.5">
            <div 
              v-for="st in currentTaskSteps" 
              :key="st.id" 
              class="flex items-center justify-between p-2.5 rounded-xl border transition group"
              :class="st.completed ? 'bg-slate-50 border-slate-200 text-slate-400 line-through' : 'bg-white border-slate-200 text-slate-800'"
            >
              <label class="flex items-center gap-2.5 flex-1 cursor-pointer select-none text-xs">
                <input 
                  type="checkbox" 
                  :checked="st.completed" 
                  @change="toggleStep(st)"
                  class="w-4 h-4 text-emerald-600 rounded border-slate-300 focus:ring-emerald-500 cursor-pointer"
                />
                <span>{{ st.titre }}</span>
              </label>
              <button 
                @click="deleteStep(st.id)" 
                class="text-slate-300 hover:text-rose-600 opacity-0 group-hover:opacity-100 transition p-1 cursor-pointer"
              >
                <i class="fa-solid fa-trash text-[11px]"></i>
              </button>
            </div>

            <div v-if="currentTaskSteps.length === 0" class="text-center py-3 text-slate-400 text-xs italic">
              Aucune étape définie pour ce post-it.
            </div>
          </div>
        </div>

        <div class="pt-4 border-t border-slate-100">
          <h4 class="text-xs font-black uppercase text-slate-700 tracking-wider mb-2 flex items-center gap-1.5">
            <i class="fa-regular fa-comments text-emerald-600"></i>
            Commentaires & avis ({{ currentTaskComments.length }})
          </h4>

          <div class="space-y-2 mb-3">
            <div v-for="c in currentTaskComments" :key="c.id" class="p-3 rounded-xl bg-slate-50 border border-slate-200 text-xs">
              <div class="flex items-center justify-between font-bold text-slate-800 mb-1">
                <span class="text-emerald-800 font-extrabold">{{ c.auteur }}</span>
                <span class="text-[10px] text-slate-400 font-normal">
                  {{ new Date(c.created_at).toLocaleDateString('fr-FR', { day: '2-digit', month: '2-digit' }) }}
                </span>
              </div>
              <p class="text-slate-600 whitespace-pre-wrap leading-relaxed">{{ c.commentaire }}</p>
            </div>
            <div v-if="currentTaskComments.length === 0" class="text-center py-4 text-slate-400 text-xs italic">
              Aucun avis pour l'instant.
            </div>
          </div>

          <div class="flex gap-2">
            <input 
              v-model="newComment" 
              @keyup.enter="addComment"
              type="text" 
              placeholder="Écrire une remarque..." 
              class="flex-1 px-3.5 py-2 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600"
            />
            <button 
              @click="addComment" 
              class="px-4 py-2 bg-emerald-700 hover:bg-emerald-800 text-white font-bold text-xs rounded-xl cursor-pointer transition"
            >
              Envoyer
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- MODALE MODIFICATION D'UNE DEMANDE EXISTANTE -->
  <div v-if="showEditMaterialModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-md w-full p-6 sm:p-7 shadow-2xl relative">
      <button @click="showEditMaterialModal = false" class="absolute top-5 right-5 text-slate-400 hover:text-slate-600 cursor-pointer text-lg">
        <i class="fa-solid fa-xmark"></i>
      </button>

      <h3 class="text-xl font-black text-slate-900 mb-1">Modifier la demande</h3>
      <p class="text-xs text-slate-500 mb-4">Corrigez l'article, la quantité, le prix estimé ou la justification.</p>

      <div class="space-y-3.5">
        <div v-if="isDirection">
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Pôle concerné</label>
          <select v-model="editMaterialForm.pole" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs font-bold outline-none focus:border-emerald-600 bg-white">
            <option v-for="p in allPoles" :key="p.id" :value="p.id">{{ p.label }}</option>
          </select>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Article / Matériel *</label>
          <input 
            v-model="editMaterialForm.article" 
            type="text" 
            required
            class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600"
          />
        </div>

        <div class="grid grid-cols-2 gap-3">
          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Quantité *</label>
            <input 
              v-model.number="editMaterialForm.quantite" 
              type="number" 
              min="1" 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600 font-mono font-bold"
            />
          </div>

          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">
              Prix unit. (€) <span class="text-[9px] text-slate-400 font-normal lowercase">(option)</span>
            </label>
            <input 
              v-model.number="editMaterialForm.estimated_price" 
              type="number" 
              step="0.01" 
              min="0" 
              placeholder="0.00" 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600 font-mono font-bold"
            />
          </div>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-700 mb-1 flex items-center justify-between">
            <span>Justification du besoin *</span>
            <span class="text-[10px] text-rose-600 font-bold lowercase">Obligatoire</span>
          </label>
          <textarea 
            v-model="editMaterialForm.details" 
            rows="3" 
            required
            class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600"
          ></textarea>
        </div>

        <button 
          @click="saveEditedMaterial" 
          class="w-full py-3 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-xl text-xs transition cursor-pointer mt-2"
        >
          Enregistrer les modifications
        </button>
      </div>
    </div>
  </div>

  <!-- MODALE D'ARBITRAGE ET JUSTIFICATION -->
  <div v-if="showDecisionModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-md w-full p-6 sm:p-7 shadow-2xl relative">
      <button @click="showDecisionModal = false" class="absolute top-5 right-5 text-slate-400 hover:text-slate-600 cursor-pointer text-lg">
        <i class="fa-solid fa-xmark"></i>
      </button>

      <h3 class="text-xl font-black text-slate-900 mb-1">
        Avis {{ decisionType === 'logistique' ? 'Logistique' : 'Trésorerie' }}
      </h3>
      <p class="text-xs text-slate-500 mb-4">
        Article : <strong class="text-slate-800">{{ decisionTargetItem?.article }} (x{{ decisionTargetItem?.quantite }})</strong> — Pôle {{ decisionTargetItem?.pole }}
      </p>

      <div class="space-y-4">
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Décision</label>
          <div class="grid grid-cols-3 gap-2">
            <button 
              type="button" 
              @click="decisionForm.status = 'valide'"
              :class="decisionForm.status === 'valide' ? 'bg-emerald-700 text-white font-bold border-emerald-700' : 'bg-slate-50 text-slate-700 border-slate-200'"
              class="py-2 text-xs rounded-xl border transition cursor-pointer"
            >
              Validé
            </button>
            <button 
              type="button" 
              @click="decisionForm.status = 'refuse'"
              :class="decisionForm.status === 'refuse' ? 'bg-rose-700 text-white font-bold border-rose-700' : 'bg-slate-50 text-slate-700 border-slate-200'"
              class="py-2 text-xs rounded-xl border transition cursor-pointer"
            >
              Refusé
            </button>
            <button 
              type="button" 
              @click="decisionForm.status = 'en_attente'"
              :class="decisionForm.status === 'en_attente' ? 'bg-amber-600 text-white font-bold border-amber-600' : 'bg-slate-50 text-slate-700 border-slate-200'"
              class="py-2 text-xs rounded-xl border transition cursor-pointer"
            >
              En attente
            </button>
          </div>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-700 mb-1 flex items-center justify-between">
            <span>Motif / Commentaire</span>
            <span v-if="decisionForm.status === 'refuse'" class="text-[10px] text-rose-600 font-bold lowercase">Obligatoire pour un refus</span>
            <span v-else class="text-[10px] text-slate-400 font-normal lowercase">Facultatif</span>
          </label>
          <textarea 
            v-model="decisionForm.comment" 
            rows="4" 
            :placeholder="decisionForm.status === 'refuse' ? 'Indiquez impérativement la raison du refus...' : 'Ajoutez une précision ou une condition éventuelle...'" 
            class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600"
          ></textarea>
        </div>

        <button 
          @click="saveDecision" 
          class="w-full py-3 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-xl text-xs transition cursor-pointer"
        >
          Valider la décision
        </button>
      </div>
    </div>
  </div>

  <!-- MODALE DEMANDE MATÉRIEL INITIALE -->
  <div v-if="showMaterialModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-md w-full p-6 sm:p-7 shadow-2xl relative">
      <button @click="showMaterialModal = false" class="absolute top-5 right-5 text-slate-400 hover:text-slate-600 cursor-pointer text-lg">
        <i class="fa-solid fa-xmark"></i>
      </button>

      <h3 class="text-xl font-black text-slate-900 mb-1">Demande de Matériel</h3>
      <p class="text-xs text-slate-500 mb-4">La justification est exigée pour instruction conjointe de la logistique et de la trésorerie.</p>

      <div class="space-y-3.5">
        <div v-if="isDirection">
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Pour le pôle</label>
          <select v-model="materialForm.pole" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs font-bold outline-none focus:border-emerald-600 bg-white">
            <option v-for="p in allPoles" :key="p.id" :value="p.id">{{ p.label }}</option>
          </select>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Article / Matériel souhaité *</label>
          <input 
            v-model="materialForm.article" 
            type="text" 
            placeholder="Ex. 10 barrières Vauban, 2 barnums 3x3..." 
            class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600"
          />
        </div>

        <div class="grid grid-cols-2 gap-3">
          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Quantité *</label>
            <input 
              v-model.number="materialForm.quantite" 
              type="number" 
              min="1" 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600 font-mono font-bold"
            />
          </div>

          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">
              Prix unit. indicatif (€) <span class="text-[9px] text-slate-400 font-normal lowercase">(option)</span>
            </label>
            <input 
              v-model.number="materialForm.estimated_price" 
              type="number" 
              step="0.01" 
              min="0" 
              placeholder="Ex. 15.50" 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600 font-mono font-bold"
            />
          </div>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-700 mb-1 flex items-center justify-between">
            <span>Pourquoi demandez-vous cela ? *</span>
            <span class="text-[10px] text-rose-600 font-bold lowercase">Obligatoire</span>
          </label>
          <textarea 
            v-model="materialForm.details" 
            rows="3" 
            required
            placeholder="Précisez impérativement l'utilité, l'emplacement exact et l'horaire requis..." 
            class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600"
          ></textarea>
        </div>

        <button 
          @click="submitMaterialRequest" 
          class="w-full py-3 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-xl text-xs transition cursor-pointer mt-2"
        >
          Transmettre la demande
        </button>
      </div>
    </div>
  </div>

  <!-- MODALE CRÉATION SPONSOR DE LA PAGE PRINCIPALE -->
  <div v-if="showSponsorModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-lg w-full p-6 shadow-2xl relative">
      <button @click="showSponsorModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 cursor-pointer">
        <i class="fa-solid fa-xmark text-lg"></i>
      </button>
      <h3 class="text-lg font-black text-slate-900 mb-1">
        {{ editingSponsor ? 'Modifier le sponsor officiel' : 'Nouveau sponsor officiel' }}
      </h3>
      <p class="text-xs text-slate-500 mb-4">Ce logo apparaîtra sur la page d'accueil du site.</p>
      
      <div class="space-y-4">
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Nom</label>
          <input v-model="sponsorForm.nom" type="text" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-sm outline-none focus:border-emerald-600" />
        </div>
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Description</label>
          <input v-model="sponsorForm.description" type="text" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-sm outline-none focus:border-emerald-600" />
        </div>
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">URL du logo</label>
          <input v-model="sponsorForm.logo_url" type="url" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-sm outline-none focus:border-emerald-600" />
        </div>
        <button @click="handleSaveSponsor" class="w-full py-3 bg-emerald-600 hover:bg-emerald-700 text-white font-bold rounded-xl text-sm transition cursor-pointer">
          {{ editingSponsor ? 'Enregistrer' : 'Créer' }}
        </button>
      </div>
    </div>
  </div>

  <!-- MODALE : AJOUT / MODIFICATION D'ENTREPRISE PROSPECTION -->
  <div v-if="showProspectModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-lg w-full p-6 sm:p-7 shadow-2xl relative max-h-[90vh] overflow-y-auto">
      <button @click="showProspectModal = false" class="absolute top-5 right-5 text-slate-400 hover:text-slate-600 cursor-pointer text-lg">
        <i class="fa-solid fa-xmark"></i>
      </button>

      <h3 class="text-xl font-black text-slate-900 mb-1">
        {{ editingProspect ? 'Modifier l’entreprise' : 'Nouvelle entreprise à démarcher' }}
      </h3>
      <p class="text-xs text-slate-500 mb-4">Renseignez les coordonnées et le contact interne à l'association.</p>

      <form @submit.prevent="saveProspect" class="space-y-3.5">
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Nom de l'entreprise *</label>
          <input 
            v-model="prospectForm.nom_entreprise" 
            required 
            type="text" 
            placeholder="Ex. Super U, Décathlon, Entreprise Bâtiment..." 
            class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600" 
          />
        </div>

        <div class="grid grid-cols-1 sm:grid-cols-2 gap-3">
          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Personne contact</label>
            <input 
              v-model="prospectForm.nom_contact" 
              type="text" 
              placeholder="Ex. M. Dupont (Directeur)" 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600" 
            />
          </div>

          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Connu par quelqu'un de l'asso ?</label>
            <input 
              v-model="prospectForm.contact_asso" 
              type="text" 
              placeholder="Ex. Connaissance de Lucie, oncle de Thomas..." 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600" 
            />
          </div>
        </div>

        <div class="grid grid-cols-1 sm:grid-cols-2 gap-3">
          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Numéro de téléphone</label>
            <input 
              v-model="prospectForm.telephone" 
              type="tel" 
              placeholder="06 12 34 56 78" 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600" 
            />
          </div>

          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Adresse e-mail</label>
            <input 
              v-model="prospectForm.email" 
              type="email" 
              placeholder="contact@entreprise.fr" 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600" 
            />
          </div>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Adresse postale</label>
          <input 
            v-model="prospectForm.adresse_postale" 
            type="text" 
            placeholder="Ex. 12 rue du Commerce, 37000 Tours" 
            class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600" 
          />
        </div>

        <div class="grid grid-cols-1 sm:grid-cols-2 gap-3">
          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Qui la contacte / responsable</label>
            <input 
              v-model="prospectForm.responsable" 
              type="text" 
              placeholder="Ex. Thomas, Lucie..." 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600" 
            />
          </div>

          <div>
            <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Statut démarche</label>
            <select 
              v-model="prospectForm.statut" 
              class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs font-bold outline-none focus:border-emerald-600 bg-white"
            >
              <option v-for="st in prospectStatusList" :key="st.id" :value="st.id">{{ st.label }}</option>
            </select>
          </div>
        </div>

        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Remarques / Historique d'échanges</label>
          <textarea 
            v-model="prospectForm.notes" 
            rows="3" 
            placeholder="Ex. Appel passé le 10/11, envoyer plaquette mécénat par mail avant vendredi..." 
            class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-xs outline-none focus:border-emerald-600"
          ></textarea>
        </div>

        <button 
          type="submit" 
          class="w-full py-3 bg-emerald-700 hover:bg-emerald-800 text-white font-bold rounded-xl text-xs transition cursor-pointer mt-2"
        >
          {{ editingProspect ? 'Enregistrer les modifications' : 'Ajouter à l’annuaire' }}
        </button>
      </form>
    </div>
  </div>

  <!-- MODALE FAQ -->
  <div v-if="showFaqModal" class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4">
    <div class="bg-white rounded-3xl max-w-lg w-full p-6 shadow-2xl relative">
      <button @click="showFaqModal = false" class="absolute top-4 right-4 text-slate-400 hover:text-slate-600 cursor-pointer">
        <i class="fa-solid fa-xmark text-lg"></i>
      </button>
      <h3 class="text-lg font-black text-slate-900 mb-1">Publier une question FAQ</h3>
      <div class="space-y-4 mt-4">
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Question</label>
          <input v-model="faqForm.question" type="text" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-sm outline-none focus:border-emerald-600" />
        </div>
        <div>
          <label class="block text-xs font-bold uppercase text-slate-600 mb-1">Réponse</label>
          <textarea v-model="faqForm.reponse" rows="4" class="w-full px-3.5 py-2.5 rounded-xl border border-slate-300 text-sm outline-none focus:border-emerald-600"></textarea>
        </div>
        <button @click="handleSaveFaq" class="w-full py-3 bg-emerald-600 hover:bg-emerald-700 text-white font-bold rounded-xl text-sm transition cursor-pointer">
          Enregistrer dans la FAQ
        </button>
      </div>
    </div>
  </div>
</template>