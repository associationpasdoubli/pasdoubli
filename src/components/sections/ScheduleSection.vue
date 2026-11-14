<script setup>
import { ref } from 'vue';

const activeTab = ref('matin');

const morningSchedule = [
  {
    time: '08h00',
    title: 'Ouverture du site aux bénévoles',
    desc: 'Briefing des signaleurs, poste de la Protection Civile et balisages finaux.'
  },
  {
    time: '08h30',
    title: 'Accueil des participants & Retrait des dossards',
    desc: "Café d'accueil et inauguration de l'espace France Alzheimer 37."
  },
  {
    time: 'à définir',
    tag: 'Départ',
    title: 'Marche 6 km',
    desc: 'Départ groupé pour les marcheurs, familles et poussettes.'
  },
  {
    time: 'à définir',
    tag: 'Départ',
    title: 'Course 10 km',
    desc: 'Épreuve chronométrée'
  },
  {
    time: 'à définir',
    tag: 'Départ',
    title: 'Course 6 km',
    desc: "Épreuve chronométrée"
  },
  {
    time: 'à définir',
    tag: 'Départs',
    title: 'Courses 3 km ',
    desc: 'Parcours ado dans la ville.'
  },
  {
    time: 'à définir',
    tag: 'Départs',
    title: 'Courses 1,5 km Enfants',
    desc: 'Parcours découvertes dans la ville.'
  }
];

const afternoonSchedule = [
  {
    time: '12h15',
    tag: 'Convivialité',
    title: 'Déjeuner Partagé, Buvette & Crêpes',
    desc: 'Restauration sur place et stands.'
  },
  {
    time: '13h45',
    tag: 'Conférence',
    title: '« Sport, Mémoire & Bien Vieillir »',
    desc: 'Animée par France Alzheimer 37 avec échanges et questions/réponses.'
  },
  {
    time: '14h30',
    title: 'Ateliers Mémoire ',
    desc: 'Activités intergénérationnelles sur l\'esplanade du village.'
  },
  {
    time: '16h15',
    title: 'Remise du Chèque Symbolique & Clôture',
    desc: 'Annonce officielle des fonds récoltés pour France Alzheimer 37.'
  }
];
</script>

<template>
  <section id="programme" class="py-20 bg-transparent">
    <div class="max-w-4xl mx-auto px-4 sm:px-6 lg:px-8">
      <!-- En-tête -->
      <div class="text-center mb-10">
        <h2 class="text-3xl sm:text-4xl font-extrabold text-slate-900 mt-3">Le Programme de la Journée</h2>

        <!-- Sélecteur d'onglets -->
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
          :key="idx"
          class="p-4 rounded-2xl bg-slate-50 border-2 border-slate-300 flex items-start justify-between gap-4 shadow-xs"
        >
          <div class="flex gap-4 items-start">
            <span class="font-mono font-extrabold text-slate-600 w-16 pt-0.5">
              {{ item.time }}
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
              <p class="text-xs text-slate-500 mt-0.5">
                {{ item.desc }}
              </p>
            </div>
          </div>
        </div>
      </div>

      <!-- Programme Après-midi -->
      <div v-show="activeTab === 'aprem'" class="space-y-3.5">
        <div 
          v-for="(item, idx) in afternoonSchedule" 
          :key="idx"
          class="p-4 rounded-2xl bg-slate-50 border-2 border-slate-300 flex items-start justify-between gap-4 shadow-xs"
        >
          <div class="flex gap-4 items-start">
            <span class="font-mono font-extrabold text-slate-600 w-16 pt-0.5">
              {{ item.time }}
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
              <p class="text-xs text-slate-500 mt-0.5">
                {{ item.desc }}
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>