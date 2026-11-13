<script setup>
import { ref } from 'vue';

// Ligne 1 : Les 3 courses
const coursesRow = [
  {
    id: 'course-10km',
    title: 'COURSE 10KM',
    category: 'Course chronométrée',
    strokeColor: '#006d32', // Vert forêt
    dividerColor: 'rgba(0, 109, 50, 0.35)',
    badgeClass: 'bg-emerald-100 text-emerald-900 border-emerald-200',
    startTime: '09h45',
    terrain: 'Chemin et route',
    condition: '+16 ans | PPS / FFA requis',
    price: '10 €',
    minAge: 'À partir de 16 ans',
    elevation: '+85 m D+',
    description: 'Le parcours phare de "Pas d\'oubli". Un tracé roulant mêlant chemins naturels et passages urbains avec ravitaillement à mi-parcours et à l\'arrivée.'
  },
  {
    id: 'course-6km',
    title: 'COURSE 6KM',
    category: 'Course populaire',
    strokeColor: '#006d32',
    dividerColor: 'rgba(0, 109, 50, 0.35)',
    badgeClass: 'bg-emerald-100 text-emerald-900 border-emerald-200',
    startTime: '10h15',
    terrain: 'Route',
    condition: '+16 ans | PPS / FFA requis',
    price: '6 €',
    minAge: 'À partir de 16 ans',
    elevation: '+45 m D+',
    description: 'Accessible à tous, idéal pour se challenger sur une distance dynamique ou partager un moment sportif entre collègues et amis.'
  },
  {
    id: 'course-3km',
    title: 'COURSE 3KM',
    category: 'Course découverte',
    strokeColor: '#006d32',
    dividerColor: 'rgba(0, 109, 50, 0.35)',
    badgeClass: 'bg-emerald-100 text-emerald-900 border-emerald-200',
    startTime: '11h15',
    terrain: 'Route',
    condition: '10 - 16 ans | PPS / FFA requis',
    price: '3 €',
    minAge: 'Entre 10 et 16 ans',
    elevation: '+20 m D+',
    description: 'Un format court et rapide, parfait pour une première expérience en compétition ou pour courir en duo parent-ado.'
  }
];

// Ligne 2 : Les 2 marches + la course enfant
const walksAndKidsRow = [
  {
    id: 'marche-6km',
    title: 'MARCHE 6KM',
    category: 'Marche solidaire',
    strokeColor: '#14b8a6', // Turquoise
    dividerColor: 'rgba(20, 184, 166, 0.35)',
    badgeClass: 'bg-teal-100 text-teal-900 border-teal-200',
    startTime: '10h20',
    terrain: 'Chemin et route',
    condition: 'Sans certificat',
    price: '6 €',
    minAge: 'Ouvert à tous',
    elevation: '+40 m D+',
    description: 'Une boucle conviviale sans chronomètre pour soutenir France Alzheimer 37 à votre rythme, en famille ou entre amis.'
  },
  {
    id: 'marche-3km',
    title: 'MARCHE 3KM',
    category: 'Marche accessible & familles',
    strokeColor: '#14b8a6',
    dividerColor: 'rgba(20, 184, 166, 0.35)',
    badgeClass: 'bg-teal-100 text-teal-900 border-teal-200',
    startTime: '11h20',
    terrain: 'Plat & accessible',
    condition: 'Sans certificat',
    price: '3 €',
    minAge: 'Ouvert à tous (familles & seniors)',
    elevation: '+15 m D+',
    description: 'Parcours plat et sécurisé permettant aux familles, enfants et personnes âgées de participer ensemble à la mobilisation.'
  },
  {
    id: 'course-1-2km',
    title: 'COURSE 1,2KM',
    category: 'Course Enfants',
    strokeColor: '#f59e0b', // Orange / Ambre
    dividerColor: 'rgba(245, 158, 11, 0.4)',
    badgeClass: 'bg-amber-100 text-amber-900 border-amber-200',
    startTime: '11h45',
    terrain: 'Parc & ligne droite',
    condition: 'Accord parental',
    price: 'Don libre',
    minAge: 'De 7 à 11 ans (Éveil & Poussins)',
    elevation: 'Plat',
    description: 'La course des jeunes champions ! Un tour encadré sous les applaudissements du public avec médaille et goûter offerts à chaque arrivant.'
  }
];

const selectedRace = ref(null);

const openRaceModal = (race) => {
  selectedRace.value = race;
};
</script>

<template>
  <section id="epreuves" class="py-16 sm:py-24 bg-white">
    <div class="max-w-6xl mx-auto px-4 sm:px-6 lg:px-8">
      
      <!-- En-tête de section -->
      <div class="text-center max-w-2xl mx-auto mb-14">
    
        <h2 class="text-3xl sm:text-4xl font-black text-slate-900 tracking-tight">
          Les Épreuves & Parcours
        </h2>
        <p class="text-slate-600 text-sm sm:text-base mt-2">
          Cliquez sur une épreuve pour afficher le règlement détaillé et le tracé.
        </p>

        <!-- Légende rapide des couleurs -->
        <div class="flex items-center justify-center gap-6 flex-wrap mt-5 text-xs font-bold text-slate-600">
          <span class="flex items-center gap-1.5">
            <span class="w-3 h-3 rounded-full bg-[#006d32]"></span> Courses
          </span>
          <span class="flex items-center gap-1.5">
            <span class="w-3 h-3 rounded-full bg-[#14b8a6]"></span> Marches
          </span>
          <span class="flex items-center gap-1.5">
            <span class="w-3 h-3 rounded-full bg-[#f59e0b]"></span> Enfants
          </span>
        </div>
      </div>

      <!-- ================= GRILLE DES 6 ÉPREUVES ================= -->
      <div class="space-y-6 select-none">
        
        <!-- LIGNE 1 : LES 3 COURSES -->
        <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
          <div
            v-for="race in coursesRow"
            :key="race.id"
            @click="openRaceModal(race)"
            class="group bg-white rounded-2xl p-6 border-[3px] shadow-xs hover:shadow-xl transition-all duration-300 hover:-translate-y-1 cursor-pointer flex flex-col justify-between"
            :style="{ borderColor: race.strokeColor }"
          >
            <!-- Titre condensé gras + filet fin horizontal -->
            <div>
              <h3
                class="text-center font-black uppercase text-[28px] leading-none tracking-tighter scale-y-110"
                :style="{ color: race.strokeColor }"
              >
                {{ race.title }}
              </h3>
              <div
                class="w-full h-[1px] mt-4"
                :style="{ backgroundColor: race.dividerColor }"
              ></div>
            </div>

            <!-- Liste des caractéristiques -->
            <div class="divide-y divide-slate-100 text-[13px] my-4">
              <div class="py-3 flex items-center justify-between">
                <span class="text-slate-500 font-medium">Départ</span>
                <span class="font-extrabold text-slate-900 font-mono">{{ race.startTime }}</span>
              </div>

              <div class="py-3 flex items-center justify-between">
                <span class="text-slate-500 font-medium">Parcours</span>
                <span class="font-extrabold text-slate-900">{{ race.terrain }}</span>
              </div>

              <div class="py-3 flex items-center justify-between gap-2">
                <span class="text-slate-500 font-medium">Condition</span>
                <span class="px-2.5 py-0.5 rounded-md bg-amber-100/90 text-amber-900 font-bold text-[11px] whitespace-nowrap">
                  {{ race.condition }}
                </span>
              </div>
            </div>

            <!-- Bas de carte : Prix à gauche, DÉTAILS à droite -->
            <div class="flex items-center justify-between pt-3 border-t border-slate-100">
              <span class="font-mono font-black text-base text-slate-900">
                {{ race.price }}
              </span>

              <span class="relative font-extrabold text-[11px] tracking-wider uppercase text-slate-900 pb-1">
                DÉTAILS
                <span class="absolute left-0 bottom-0 w-3/5 h-[2.5px] bg-slate-900 group-hover:w-full transition-all duration-300"></span>
              </span>
            </div>
          </div>
        </div>

        <!-- LIGNE 2 : LES 2 MARCHES + LA COURSE ENFANT -->
        <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
          <div
            v-for="race in walksAndKidsRow"
            :key="race.id"
            @click="openRaceModal(race)"
            class="group bg-white rounded-2xl p-6 border-[3px] shadow-xs hover:shadow-xl transition-all duration-300 hover:-translate-y-1 cursor-pointer flex flex-col justify-between"
            :style="{ borderColor: race.strokeColor }"
          >
            <!-- Titre condensé gras + filet fin horizontal -->
            <div>
              <h3
                class="text-center font-black uppercase text-[28px] leading-none tracking-tighter scale-y-110"
                :style="{ color: race.strokeColor }"
              >
                {{ race.title }}
              </h3>
              <div
                class="w-full h-[1px] mt-4"
                :style="{ backgroundColor: race.dividerColor }"
              ></div>
            </div>

            <!-- Liste des caractéristiques -->
            <div class="divide-y divide-slate-100 text-[13px] my-4">
              <div class="py-3 flex items-center justify-between">
                <span class="text-slate-500 font-medium">Départ</span>
                <span class="font-extrabold text-slate-900 font-mono">{{ race.startTime }}</span>
              </div>

              <div class="py-3 flex items-center justify-between">
                <span class="text-slate-500 font-medium">Parcours</span>
                <span class="font-extrabold text-slate-900">{{ race.terrain }}</span>
              </div>

              <div class="py-3 flex items-center justify-between gap-2">
                <span class="text-slate-500 font-medium">Condition</span>
                <span class="px-2.5 py-0.5 rounded-md bg-amber-100/90 text-amber-900 font-bold text-[11px] whitespace-nowrap">
                  {{ race.condition }}
                </span>
              </div>
            </div>

            <!-- Bas de carte : Prix à gauche, DÉTAILS à droite -->
            <div class="flex items-center justify-between pt-3 border-t border-slate-100">
              <span class="font-mono font-black text-base text-slate-900">
                {{ race.price }}
              </span>

              <span class="relative font-extrabold text-[11px] tracking-wider uppercase text-slate-900 pb-1">
                DÉTAILS
                <span class="absolute left-0 bottom-0 w-3/5 h-[2.5px] bg-slate-900 group-hover:w-full transition-all duration-300"></span>
              </span>
            </div>
          </div>
        </div>

      </div>

      <!-- ================= BOUTON D'INSCRIPTION GLOBAL ================= -->
      <div class="mt-12 text-center">
        <a
          href="#inscription"
          class="inline-flex items-center justify-center gap-3 px-8 py-4 bg-emerald-700 hover:bg-emerald-800 text-white font-black text-sm uppercase tracking-wider rounded-2xl shadow-lg shadow-emerald-700/25 transition-all duration-200 hover:-translate-y-0.5"
        >
          <i class="fa-solid fa-ticket text-base"></i>
          <span>S'inscrire à une épreuve</span>
        </a>
        <p class="text-xs text-slate-500 font-medium mt-2.5">
          Ouverture officielle de la billetterie le <strong class="text-slate-800">14 décembre</strong>.
        </p>
      </div>

    </div>

    <!-- ================= MODALE DE DÉTAIL DE L'ÉPREUVE ================= -->
    <div
      v-if="selectedRace"
      @click.self="selectedRace = null"
      class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4"
    >
      <div class="bg-white rounded-3xl max-w-lg w-full p-6 sm:p-8 shadow-2xl relative border-t-8" :style="{ borderColor: selectedRace.strokeColor }">
        <button
          @click="selectedRace = null"
          class="absolute top-5 right-5 text-slate-400 hover:text-slate-600 cursor-pointer text-lg"
        >
          <i class="fa-solid fa-xmark"></i>
        </button>

        <span :class="[selectedRace.badgeClass, 'text-[10px] uppercase font-extrabold tracking-wider px-2.5 py-1 rounded-full border inline-block']">
          {{ selectedRace.category }}
        </span>

        <h3 class="text-3xl font-black mt-3 uppercase" :style="{ color: selectedRace.strokeColor }">
          {{ selectedRace.title }}
        </h3>

        <p class="text-sm text-slate-600 leading-relaxed mt-3">
          {{ selectedRace.description }}
        </p>

        <div class="grid grid-cols-2 gap-3 my-6">
          <div class="p-3 rounded-2xl bg-slate-50 border border-slate-100">
            <div class="text-[10px] font-bold uppercase text-slate-400">Heure de départ</div>
            <div class="text-sm font-black text-slate-900 font-mono mt-0.5">{{ selectedRace.startTime }}</div>
          </div>

          <div class="p-3 rounded-2xl bg-slate-50 border border-slate-100">
            <div class="text-[10px] font-bold uppercase text-slate-400">Tarif</div>
            <div class="text-sm font-black font-mono mt-0.5" :style="{ color: selectedRace.strokeColor }">
              {{ selectedRace.price }}
            </div>
          </div>

          <div class="p-3 rounded-2xl bg-slate-50 border border-slate-100">
            <div class="text-[10px] font-bold uppercase text-slate-400">Dénivelé & Terrain</div>
            <div class="text-xs font-bold text-slate-900 mt-0.5">{{ selectedRace.terrain }} ({{ selectedRace.elevation }})</div>
          </div>

          <div class="p-3 rounded-2xl bg-slate-50 border border-slate-100">
            <div class="text-[10px] font-bold uppercase text-slate-400">Condition / Âge</div>
            <div class="text-xs font-bold text-slate-800 mt-0.5">{{ selectedRace.condition }} — {{ selectedRace.minAge }}</div>
          </div>
        </div>

        <div class="flex items-center justify-between gap-3 pt-2">
          <button
            @click="selectedRace = null"
            class="px-4 py-3 rounded-xl border border-slate-200 text-slate-600 hover:bg-slate-50 font-bold text-xs cursor-pointer transition"
          >
            Fermer
          </button>

          <a
            href="#inscription"
            @click="selectedRace = null"
            class="flex-1 py-3 text-white font-bold text-xs uppercase tracking-wider rounded-xl text-center shadow-md transition hover:opacity-90 flex items-center justify-center gap-2"
            :style="{ backgroundColor: selectedRace.strokeColor }"
          >
            <i class="fa-solid fa-ticket"></i>
            <span>S'inscrire (Ouverture le 14 déc.)</span>
          </a>
        </div>
      </div>
    </div>

  </section>
</template>