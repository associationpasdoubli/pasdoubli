<script setup>
import { ref, computed, onMounted } from 'vue';
import { supabase } from '../../services/supabase';

const activeTab = ref('matin');
const dbSchedule = ref([]);

const defaultMorning = [
  { time_label: '08h00', title: 'Ouverture du site aux bénévoles', description: 'Briefing des signaleurs, poste de la Protection Civile et balisages finaux.' },
  { time_label: '08h30', title: 'Accueil des participants & Retrait des dossards', description: "Café d'accueil et inauguration de l'espace France Alzheimer 37." },
  { time_label: 'à définir', tag: 'Départ', title: 'Marche 6 km', description: 'Départ groupé pour les marcheurs, familles et poussettes.' },
  { time_label: 'à définir', tag: 'Départ', title: 'Course 10 km', description: 'Épreuve chronométrée' },
  { time_label: 'à définir', tag: 'Départ', title: 'Course 6 km', description: 'Épreuve chronométrée' },
  { time_label: 'à définir', tag: 'Départs', title: 'Courses 3 km', description: 'Parcours ado dans la ville.' },
  { time_label: 'à définir', tag: 'Départs', title: 'Courses 1,5 km Enfants', description: 'Parcours découvertes dans la ville.' }
];

const defaultAfternoon = [
  { time_label: '12h15', tag: 'Convivialité', title: 'Déjeuner Partagé, Buvette & Crêpes', description: 'Restauration sur place et stands.' },
  { time_label: '13h45', tag: 'Conférence', title: '« Sport, Mémoire & Bien Vieillir »', description: 'Animée par France Alzheimer 37 avec échanges et questions/réponses.' },
  { time_label: '14h30', title: 'Ateliers Mémoire', description: "Activités intergénérationnelles sur l'esplanade du village." },
  { time_label: '16h15', title: 'Remise du Chèque Symbolique & Clôture', description: 'Annonce officielle des fonds récoltés pour France Alzheimer 37.' }
];

const morningSchedule = computed(() => {
  const list = dbSchedule.value.filter(i => i.periode === 'matin');
  return list.length > 0 ? list : defaultMorning;
});

const afternoonSchedule = computed(() => {
  const list = dbSchedule.value.filter(i => i.periode === 'aprem');
  return list.length > 0 ? list : defaultAfternoon;
});

onMounted(async () => {
  const { data } = await supabase
    .from('program_schedule')
    .select('*')
    .order('sort_order', { ascending: true })
    .order('id', { ascending: true });
  if (data) dbSchedule.value = data;
});
</script>

<template>
  <section id="programme" class="py-20 bg-transparent">
    <div class="max-w-4xl mx-auto px-4 sm:px-6 lg:px-8">
      <div class="text-center mb-10">
        <h2 class="text-3xl sm:text-4xl font-extrabold text-slate-900 mt-3">Le Programme de la Journée</h2>

        <div class="inline-flex p-1 bg-slate-100 rounded-2xl border border-slate-200 mt-6">
          <button 
            @click="activeTab = 'matin'" 
            :class="activeTab === 'matin' ? 'bg-white text-emerald-700 shadow-sm' : 'text-slate-600'" 
            class="px-5 py-2.5 rounded-xl font-bold text-sm transition flex items-center gap-2 cursor-pointer"
          >
            <i class="fa-solid text-amber-500"></i>Matinée Sportive (8h - 12h15)
          </button>
          <button 
            @click="activeTab = 'aprem'" 
            :class="activeTab === 'aprem' ? 'bg-white text-emerald-700 shadow-sm' : 'text-slate-600'" 
            class="px-5 py-2.5 rounded-xl font-bold text-sm transition flex items-center gap-2 cursor-pointer"
          >
            <i class="fa-solid text-emerald-600"></i>Après-midi Village (12h15 - 17h)
          </button>
        </div>
      </div>

      <!-- Programme Matin -->
      <div v-show="activeTab === 'matin'" class="space-y-3.5">
        <div 
          v-for="(item, idx) in morningSchedule" 
          :key="item.id || idx"
          class="p-4 rounded-2xl bg-slate-50 border-2 border-slate-300 flex items-start justify-between gap-4 shadow-xs"
        >
          <div class="flex gap-4 items-start">
            <span class="font-mono font-extrabold text-slate-600 w-20 pt-0.5">
              {{ item.time_label }}
            </span>
            <div>
              <span 
                v-if="item.tag" 
                class="text-[10px] uppercase font-bold bg-slate-600 text-white px-2 py-0.5 rounded"
              >
                {{ item.tag }}
              </span>
              <h4 class="font-bold text-slate-600 mt-1">
                {{ item.title }}
              </h4>
              <p v-if="item.description" class="text-xs text-slate-500 mt-0.5">
                {{ item.description }}
              </p>
            </div>
          </div>
        </div>
      </div>

      <!-- Programme Après-midi -->
      <div v-show="activeTab === 'aprem'" class="space-y-3.5">
        <div 
          v-for="(item, idx) in afternoonSchedule" 
          :key="item.id || idx"
          class="p-4 rounded-2xl bg-slate-50 border-2 border-slate-300 flex items-start justify-between gap-4 shadow-xs"
        >
          <div class="flex gap-4 items-start">
            <span class="font-mono font-extrabold text-slate-600 w-20 pt-0.5">
              {{ item.time_label }}
            </span>
            <div>
              <span 
                v-if="item.tag" 
                class="text-[10px] uppercase font-bold bg-slate-600 text-white px-2 py-0.5 rounded"
              >
                {{ item.tag }}
              </span>
              <h4 class="font-bold text-slate-600 mt-1">
                {{ item.title }}
              </h4>
              <p v-if="item.description" class="text-xs text-slate-500 mt-0.5">
                {{ item.description }}
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>