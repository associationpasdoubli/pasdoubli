<script setup>
import { ref, computed, onMounted } from 'vue';
import { supabase } from '../../services/supabase';

const dbActivities = ref([]);
const loading = ref(true);

const defaultActivities = [
  {
    title: 'Stand Gourmand & Crêpes pour leur camp d’été',
    description: '100% des recettes financent le séjour d’été des jeunes du centre.',
    color_theme: 'amber',
    layout_zone: 'left'
  },
  {
    title: 'Encadrement des Courses Enfants & Jeux',
    description: 'Échauffement ludique, maquillage, jeux géants et remise des médailles.',
    color_theme: 'teal',
    layout_zone: 'left'
  },
  {
    title: 'Buvette & Restauration',
    description: 'Café d’accueil, sandwichs frais locaux, jus de fruits artisanaux et fouées locaux.',
    color_theme: 'emerald',
    layout_zone: 'right'
  },
  {
    title: 'Animation sur la maladie',
    description: 'Stands organisés pour sensibiliser et informer sur la maladie, animés par les bénévoles de France Alzheimer 37.',
    color_theme: 'emerald',
    layout_zone: 'right'
  },
  {
    title: 'Photocall Souvenir',
    description: 'Repartez avec votre cliché de la journée avec votre dossard ou votre médaille.',
    color_theme: 'emerald',
    layout_zone: 'right'
  },
  {
    title: 'Poste de Secours',
    description: 'Dispositif Prévisionnel de Secours assuré par la Croix-Rouge française.',
    color_theme: 'emerald',
    layout_zone: 'right'
  }
];

const allActivities = computed(() => {
  return dbActivities.value.length > 0 ? dbActivities.value : defaultActivities;
});

// Éléments placés explicitement à gauche ('left'), complétés par les premiers 'auto' si besoin
const leftActivities = computed(() => {
  const explicitLeft = allActivities.value.filter(i => i.layout_zone === 'left');
  const autoItems = allActivities.value.filter(i => !i.layout_zone || i.layout_zone === 'auto');
  if (explicitLeft.length > 0) return explicitLeft;
  return autoItems.slice(0, 2);
});

// Éléments placés explicitement à droite ('right'), plus le reste des 'auto'
const rightActivities = computed(() => {
  const explicitLeft = allActivities.value.filter(i => i.layout_zone === 'left');
  const explicitRight = allActivities.value.filter(i => i.layout_zone === 'right');
  const autoItems = allActivities.value.filter(i => !i.layout_zone || i.layout_zone === 'auto');

  if (explicitLeft.length > 0) {
    return [...explicitRight, ...autoItems];
  }
  return [...explicitRight, ...autoItems.slice(2)];
});

const getSquareColor = (theme) => {
  if (theme === 'teal') return 'bg-[#2dd49f]';
  if (theme === 'emerald') return 'bg-[#22c55e]';
  return 'bg-[#fbbf24]';
};

onMounted(async () => {
  const { data } = await supabase
    .from('village_activities')
    .select('*')
    .order('sort_order', { ascending: true })
    .order('id', { ascending: true });

  if (data) dbActivities.value = data;
  loading.value = false;
});
</script>

<template>
  <section id="village" class="py-16 sm:py-24 bg-[#07291b] text-white">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">

      <div v-if="loading" class="text-center py-12">
        <i class="fa-solid fa-circle-notch fa-spin text-3xl text-emerald-400"></i>
      </div>

      <div v-else class="grid grid-cols-1 lg:grid-cols-12 gap-10 lg:gap-12 items-center">
        
        <!-- COLONNE GAUCHE : Titre, sous-titre et bandeaux horizontaux -->
        <div class="lg:col-span-6 space-y-6">
          <div>
            <h2 class="text-3xl sm:text-4xl lg:text-[42px] font-black tracking-tight text-white leading-tight">
              Un village solidaire
            </h2>
            <p class="text-emerald-100/80 text-sm sm:text-base mt-2.5">
              Parce que la solidarité unit toutes les générations.
            </p>
          </div>

          <div class="space-y-4 pt-2">
            <div
              v-for="(item, idx) in leftActivities"
              :key="item.id || idx"
              class="bg-white/[0.04] hover:bg-white/[0.07] border border-white/10 rounded-2xl p-4 sm:p-5 flex items-center gap-4 sm:gap-5 transition-colors"
            >
              <!-- Image personnalisée si renseignée, sinon carré coloré -->
              <div
                v-if="item.image_url"
                class="w-12 h-12 rounded-xl overflow-hidden flex-shrink-0 shadow-sm bg-white/10"
              >
                <img :src="item.image_url" :alt="item.title" class="w-full h-full object-cover" />
              </div>
              <div
                v-else
                :class="[getSquareColor(item.color_theme), 'w-12 h-12 rounded-xl flex-shrink-0 shadow-sm flex items-center justify-center text-[#07291b] text-lg']"
              >
                <i v-if="item.icon && item.icon !== 'fa-tent'" :class="['fa-solid', item.icon]"></i>
              </div>

              <div>
                <h3 class="font-extrabold text-white text-sm sm:text-base leading-snug">
                  {{ item.title }}
                </h3>
                <p v-if="item.description" class="text-xs sm:text-[13px] text-emerald-100/70 mt-1 leading-relaxed">
                  {{ item.description }}
                </p>
              </div>
            </div>
          </div>
        </div>

        <!-- COLONNE DROITE : Grille de cartes centrées -->
        <div class="lg:col-span-6 grid grid-cols-1 sm:grid-cols-2 gap-4 sm:gap-5">
          <div
            v-for="(item, idx) in rightActivities"
            :key="item.id || idx"
            class="bg-white/[0.08] hover:bg-white/[0.12] border border-white/10 rounded-3xl p-6 sm:p-7 text-center flex flex-col items-center justify-center min-h-[185px] transition-colors"
          >
            <!-- Image ou icône optionnelle au-dessus du titre si ajoutée -->
            <div
              v-if="item.image_url"
              class="w-12 h-12 rounded-xl overflow-hidden mb-3 shadow-sm bg-white/10"
            >
              <img :src="item.image_url" :alt="item.title" class="w-full h-full object-cover" />
            </div>

            <h3 class="font-extrabold text-white text-base sm:text-lg leading-snug">
              {{ item.title }}
            </h3>
            <p v-if="item.description" class="text-xs sm:text-[13px] text-emerald-100/75 mt-2.5 leading-relaxed">
              {{ item.description }}
            </p>
          </div>
        </div>

      </div>

    </div>
  </section>
</template>