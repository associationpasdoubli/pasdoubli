<script setup>
import { ref } from 'vue';
import runnerImg from '../../assets/affiche.png';

// 1. LES COURSES (4 formats chronométrés / enfants)
const courses = [
  {
    id: 'course-10km',
    distance: '10 KM',
    title: 'COURSE + DE 16 ANS',
    subtitle: 'Course chronométrée officielle',
    priceLabel: '10 €',
    isFreeDonation: false,
    ribbonBg: 'bg-[#1f7a4c] group-hover:bg-[#166534]',
    priceBorder: 'border-[#1f7a4c] text-[#1f7a4c] group-hover:bg-emerald-50',
    startTime: '',
    terrain: 'Chemin et route',
    condition: 'PPS ou licence FFA (+16 ans)',
    description: 'Le parcours phare de 10 km au départ du Square des Anciens Combattants d\'Athée-sur-Cher. Chronométrage officiel, médaille finisher et t-shirt offert.'
  },
  {
    id: 'course-6km',
    distance: '6 KM',
    title: 'COURSE + DE 16 ANS',
    subtitle: 'Course populaire',
    priceLabel: '6 €',
    isFreeDonation: false,
    ribbonBg: 'bg-[#1f7a4c] group-hover:bg-[#166534]',
    priceBorder: 'border-[#1f7a4c] text-[#1f7a4c] group-hover:bg-emerald-50',
    startTime: '',
    terrain: 'Chemin et route',
    condition: 'PPS ou licence FFA (+16 ans)',
    description: 'Une distance dynamique accessible à tous les coureurs dès 16 ans pour se dépasser dans une ambiance conviviale.'
  },
  {
    id: 'course-3km',
    distance: '3 KM',
    title: 'COURSE 10 / 16 ANS',
    subtitle: 'Course jeunes & découverte',
    priceLabel: '3 €',
    isFreeDonation: false,
    ribbonBg: 'bg-[#1f7a4c] group-hover:bg-[#166534]',
    priceBorder: 'border-[#1f7a4c] text-[#1f7a4c] group-hover:bg-emerald-50',
    startTime: '',
    terrain: 'Chemin et route',
    condition: 'Jeunes de 10 à 16 ans (accord parental)',
    description: 'Un tracé court et rapide spécialement calibré pour les jeunes coureurs de 10 à 16 ans.'
  },
  {
    id: 'course-1-2km',
    distance: '1,2 KM',
    title: 'COURSE - DE 10 ANS',
    subtitle: 'Course enfants',
    priceLabel: 'DON LIBRE',
    isFreeDonation: true,
    ribbonBg: 'bg-[#0284c7] group-hover:bg-[#0369a1]',
    priceBorder: 'border-[#1e6091] text-[#1e6091] bg-sky-50 group-hover:bg-sky-100',
    startTime: '',
    terrain: 'Boucle sécurisée au village',
    condition: 'Enfants de moins de 10 ans',
    description: 'La course dédiée aux enfants de moins de 10 ans ! Participation sur don libre au profit de France Alzheimer 37, médaille finisher pour chaque enfant.'
  }
];

// 2. LES MARCHES (2 formats solidaires ouverts à tous sans certificat)
const marches = [
  {
    id: 'marche-6km',
    distance: '6 KM',
    title: 'MARCHE TOUT PUBLIC',
    subtitle: 'Grande boucle solidaire',
    priceLabel: '6 €',
    isFreeDonation: false,
    ribbonBg: 'bg-[#0d9488] group-hover:bg-[#0f766e]',
    priceBorder: 'border-[#0d9488] text-[#0d9488] group-hover:bg-teal-50',
    startTime: '',
    terrain: 'Chemin et route',
    condition: 'Ouvert à tous • Sans certificat médical',
    description: 'Une randonnée solidaire de 6 km sans chronomètre pour soutenir France Alzheimer 37 à votre rythme, en famille ou entre amis.'
  },
  {
    id: 'marche-3km',
    distance: '3 KM',
    title: 'MARCHE TOUT PUBLIC',
    subtitle: 'Boucle découverte & familles',
    priceLabel: '3 €',
    isFreeDonation: false,
    ribbonBg: 'bg-[#0d9488] group-hover:bg-[#0f766e]',
    priceBorder: 'border-[#0d9488] text-[#0d9488] group-hover:bg-teal-50',
    startTime: '',
    terrain: 'Plat & accessible',
    condition: 'Ouvert à tous • Sans certificat médical',
    description: 'Un parcours plat et sécurisé permettant aux familles, enfants et seniors de participer ensemble à la mobilisation.'
  }
];

const selectedRace = ref(null);
</script>

<template>
  <section id="epreuves" class="relative py-16 sm:py-24 bg-white overflow-hidden">
    
    <!-- ILLUSTRATION COUREUSE COLLÉE AU BORD DROIT DE L'ÉCRAN -->
    <div class="hidden xl:block pointer-events-none select-none absolute right-0 top-1/2 -translate-y-1/2 z-0">
      <img
        :src="runnerImg"
        alt="Coureuse Pas d'oubli"
        class="h-[620px] 2xl:h-[700px] w-auto object-contain object-right mix-blend-multiply"
      />
    </div>

    <div class="relative z-10 max-w-6xl mx-auto px-4 sm:px-6 lg:px-8">
      <div class="xl:pr-56">
        
        <!-- En-tête de section -->
        <div class="text-center lg:text-left max-w-2xl mx-auto lg:mx-0 mb-12">
          <h2 class="text-3xl sm:text-5xl font-black uppercase text-[#1f7a4c] tracking-tight">
            Les Épreuves & Tarifs
          </h2>
          <p class="text-slate-600 text-sm sm:text-base mt-2">
            Chacun peut agir à petite échelle. Chaque participation, chaque échange et chaque geste compte
          </p>
        </div>

        <div class="space-y-12 select-none">
          
          <!-- ================= BLOC 1 : LES COURSES ================= -->
          <div>
            <div class="flex items-center gap-3 mb-5">
              <h3 class="text-xl sm:text-2xl font-black uppercase tracking-tight text-slate-900">
                Les Courses
              </h3>
              <div class="flex-1 h-[2px] bg-[#1f7a4c]/20 rounded-full"></div>
            </div>

            <div class="space-y-4">
              <div
                v-for="race in courses"
                :key="race.id"
                @click="selectedRace = race"
                class="group flex flex-col sm:flex-row items-stretch sm:items-center gap-3 sm:gap-5 cursor-pointer transition-transform duration-200 hover:translate-x-1"
              >
                <!-- Ruban incliné -->
                <div :class="[race.ribbonBg, 'flex-1 text-white px-5 sm:px-7 py-4 ribbon-slant shadow-md flex items-center justify-between gap-4 transition-colors']">
                  <div class="flex items-center gap-4 sm:gap-6">
                    <span class="text-3xl sm:text-5xl font-black tracking-tighter leading-none whitespace-nowrap">
                      {{ race.distance }}
                    </span>

                    <div class="flex flex-col justify-center border-l-2 border-white/30 pl-4">
                      <span class="text-sm sm:text-lg font-black uppercase tracking-tight leading-tight">
                        {{ race.title }}
                      </span>
                      <span class="text-[11px] sm:text-xs font-semibold text-white/85">
                        {{ race.subtitle }} • Départ {{ race.startTime }}
                      </span>
                    </div>
                  </div>

                  <span class="hidden md:inline-flex items-center gap-1.5 text-[11px] font-extrabold uppercase tracking-wider bg-white/15 px-3 py-1.5 rounded-lg">
                    <span>Détails</span>
                    <i class="fa-solid fa-chevron-right text-[9px]"></i>
                  </span>
                </div>

                <!-- Tarif encadré net -->
                <div class="sm:w-44 flex items-center justify-between sm:justify-center flex-shrink-0">
                  <span class="text-xs font-bold uppercase text-slate-500 sm:hidden">Tarif :</span>
                  <span
                    :class="[
                      race.priceBorder,
                      race.isFreeDonation ? 'text-base sm:text-lg font-sans' : 'text-2xl sm:text-3xl font-mono',
                      'w-full text-center px-4 py-2 bg-white border-[3px] font-black tracking-tight rounded-xl shadow-xs transition-colors'
                    ]"
                  >
                    {{ race.priceLabel }}
                  </span>
                </div>
              </div>
            </div>
          </div>

          <!-- ================= BLOC 2 : LES MARCHES ================= -->
          <div>
            <div class="flex items-center gap-3 mb-5">
              <h3 class="text-xl sm:text-2xl font-black uppercase tracking-tight text-slate-900">
                Les Marches Solidaires
              </h3>
              <span class="px-2.5 py-0.5 bg-teal-50 text-[#0d9488] border border-teal-200 rounded-md text-[10px] font-extrabold uppercase">
                Sans certificat
              </span>
              <div class="flex-1 h-[2px] bg-[#0d9488]/20 rounded-full"></div>
            </div>

            <div class="space-y-4">
              <div
                v-for="marche in marches"
                :key="marche.id"
                @click="selectedRace = marche"
                class="group flex flex-col sm:flex-row items-stretch sm:items-center gap-3 sm:gap-5 cursor-pointer transition-transform duration-200 hover:translate-x-1"
              >
                <!-- Ruban incliné turquoise -->
                <div :class="[marche.ribbonBg, 'flex-1 text-white px-5 sm:px-7 py-4 ribbon-slant shadow-md flex items-center justify-between gap-4 transition-colors']">
                  <div class="flex items-center gap-4 sm:gap-6">
                    <span class="text-3xl sm:text-5xl font-black tracking-tighter leading-none whitespace-nowrap">
                      {{ marche.distance }}
                    </span>

                    <div class="flex flex-col justify-center border-l-2 border-white/30 pl-4">
                      <span class="text-sm sm:text-lg font-black uppercase tracking-tight leading-tight">
                        {{ marche.title }}
                      </span>
                      <span class="text-[11px] sm:text-xs font-semibold text-white/85">
                        {{ marche.subtitle }} • Départ {{ marche.startTime }}
                      </span>
                    </div>
                  </div>

                  <span class="hidden md:inline-flex items-center gap-1.5 text-[11px] font-extrabold uppercase tracking-wider bg-white/15 px-3 py-1.5 rounded-lg">
                    <span>Détails</span>
                    <i class="fa-solid fa-chevron-right text-[9px]"></i>
                  </span>
                </div>

                <!-- Tarif encadré turquoise -->
                <div class="sm:w-44 flex items-center justify-between sm:justify-center flex-shrink-0">
                  <span class="text-xs font-bold uppercase text-slate-500 sm:hidden">Tarif :</span>
                  <span
                    :class="[
                      marche.priceBorder,
                      'w-full text-center px-4 py-2 bg-white border-[3px] font-mono font-black text-2xl sm:text-3xl tracking-tight rounded-xl shadow-xs transition-colors'
                    ]"
                  >
                    {{ marche.priceLabel }}
                  </span>
                </div>
              </div>
            </div>
          </div>

        </div>

        <!-- Encart d'inscription global -->
        <div class="mt-12 bg-sky-50/90 border-[3px] border-[#1f7a4c] rounded-2xl p-6 flex flex-col sm:flex-row items-center justify-between gap-5 shadow-xs">
          <div class="text-center sm:text-left">
            <p class="text-xs sm:text-sm text-slate-800 font-bold">
              Ouverture officielle des inscriptions en ligne le <span class="text-[#1f7a4c] font-black underline">14 décembre</span>.
            </p>
            <p class="text-xs text-slate-500 mt-0.5">
              Médaille finisher & t-shirt offerts • 100% des bénéfices reversés à France Alzheimer 37.
            </p>
          </div>

          <a
            href="#inscription"
            class="w-full sm:w-auto px-7 py-3.5 bg-[#1f7a4c] hover:bg-[#166534] text-white font-black text-xs uppercase tracking-wider ribbon-slant-both shadow-md transition hover:-translate-y-0.5 text-center whitespace-nowrap"
          >
            S'inscrire en ligne
          </a>
        </div>

      </div>
    </div>

    <!-- MODALE DÉTAIL ÉPREUVE -->
    <div
      v-if="selectedRace"
      @click.self="selectedRace = null"
      class="fixed inset-0 z-50 bg-black/60 backdrop-blur-xs flex items-center justify-center p-4"
    >
      <div class="bg-white rounded-2xl max-w-lg w-full p-6 sm:p-8 shadow-2xl relative border-[3px] border-[#1f7a4c]">
        <button
          @click="selectedRace = null"
          class="absolute top-5 right-5 text-slate-400 hover:text-slate-700 cursor-pointer text-lg"
        >
          <i class="fa-solid fa-xmark"></i>
        </button>

        <div class="bg-[#1f7a4c] text-white font-black text-2xl uppercase px-4 py-1.5 ribbon-slant inline-block">
          {{ selectedRace.distance }}
        </div>

        <h4 class="text-lg font-black uppercase text-slate-900 mt-3">
          {{ selectedRace.title }}
        </h4>

        <p class="text-sm text-slate-600 leading-relaxed mt-3">
          {{ selectedRace.description }}
        </p>

        <div class="grid grid-cols-2 gap-3 my-6">
          <div class="p-3 rounded-xl bg-slate-50 border border-slate-200">
            <div class="text-[10px] font-bold uppercase text-slate-400">Départ</div>
            <div class="text-sm font-black text-slate-900 font-mono mt-0.5">{{ selectedRace.startTime }}</div>
          </div>

          <div class="p-3 rounded-xl bg-slate-50 border border-slate-200">
            <div class="text-[10px] font-bold uppercase text-slate-400">Tarif</div>
            <div class="text-sm font-black text-[#1f7a4c] mt-0.5">{{ selectedRace.priceLabel }}</div>
          </div>

          <div class="p-3 rounded-xl bg-slate-50 border border-slate-200 col-span-2">
            <div class="text-[10px] font-bold uppercase text-slate-400">Conditions d'accès</div>
            <div class="text-xs font-bold text-slate-800 mt-0.5">{{ selectedRace.condition }}</div>
          </div>
        </div>

        <div class="flex items-center justify-end gap-3">
          <button
            @click="selectedRace = null"
            class="px-4 py-2.5 rounded-xl border border-slate-300 text-slate-700 font-bold text-xs cursor-pointer hover:bg-slate-50"
          >
            Fermer
          </button>
          <a
            href="#inscription"
            @click="selectedRace = null"
            class="px-6 py-2.5 bg-[#1f7a4c] hover:bg-[#166534] text-white font-black text-xs uppercase tracking-wider rounded-xl shadow-sm"
          >
            Inscription (14 déc.)
          </a>
        </div>
      </div>
    </div>
  </section>
</template>