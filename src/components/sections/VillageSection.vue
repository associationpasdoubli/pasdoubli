<script setup>
import { ref, computed, onMounted } from 'vue';
import { supabase } from '../../services/supabase';

const dbActivities = ref([]);
const loading = ref(true);

const layoutConfig = ref({
  split: '50-50',      // '50-50', '40-60', 'stacked'
  rightCols: '2',      // '2' ou '3'
  spacing: 'normal',   // 'compact', 'normal', 'relaxed'
  gapSize: 'normal',   // 'tight', 'normal', 'wide'
  cardStyle: 'glass',  // 'glass' (translucide), 'solid' (opaque), 'bordered' (bordure marquée)
  theme: 'dark'        // 'dark' (#07291b), 'forest' (#14532d), 'light' (damier clair)
});

const defaultActivities = [
  {
    title: 'Stand France Alzheimer 37',
    description: 'Sensibilisation, conseils, accompagnement et table ronde avec les bénévoles.',
    icon: 'fa-hand-holding-heart',
    color_theme: 'amber',
    layout_zone: 'left',
    logo_fit: 'contain',
    logo_size: 'md',
    logo_bg: 'white'
  },
  {
    title: 'Poste de secours',
    description: 'Prévention et sécurité médicale assurées par la Protection Civile.',
    icon: 'fa-kit-medical',
    color_theme: 'teal',
    layout_zone: 'left',
    logo_fit: 'contain',
    logo_size: 'md',
    logo_bg: 'white'
  },
  {
    title: 'Animation Pas d’Oubli',
    description: 'Jeux intergénérationnels, quiz mémoire et animations conviviales.',
    icon: 'fa-puzzle-piece',
    color_theme: 'emerald',
    layout_zone: 'right',
    logo_fit: 'contain',
    logo_size: 'md',
    logo_bg: 'white'
  },
  {
    title: 'Spectacle des enfants',
    description: 'Spectacle présenté par les enfants du centre de loisirs d’Athée-sur-Cher.',
    icon: 'fa-music',
    color_theme: 'emerald',
    layout_zone: 'right',
    logo_fit: 'contain',
    logo_size: 'md',
    logo_bg: 'white'
  },
  {
    title: 'Stand de restauration',
    description: 'Gâteaux maison, crêpes, boissons fraîches et gourmandises locales.',
    icon: 'fa-mug-hot',
    color_theme: 'emerald',
    layout_zone: 'right',
    logo_fit: 'contain',
    logo_size: 'md',
    logo_bg: 'white'
  },
  {
    title: 'Tombola Solidaire',
    description: 'De nombreux lots offerts par nos commerçants et partenaires locaux.',
    icon: 'fa-ticket',
    color_theme: 'emerald',
    layout_zone: 'right',
    logo_fit: 'contain',
    logo_size: 'md',
    logo_bg: 'white'
  }
];

const allActivities = computed(() => {
  return dbActivities.value.length > 0 ? dbActivities.value : defaultActivities;
});

const leftActivities = computed(() => {
  const explicitLeft = allActivities.value.filter(i => i.layout_zone === 'left');
  const autoItems = allActivities.value.filter(i => !i.layout_zone || i.layout_zone === 'auto');
  if (explicitLeft.length > 0) return explicitLeft;
  return autoItems.slice(0, 2);
});

const rightActivities = computed(() => {
  const explicitLeft = allActivities.value.filter(i => i.layout_zone === 'left');
  const explicitRight = allActivities.value.filter(i => i.layout_zone === 'right');
  const autoItems = allActivities.value.filter(i => !i.layout_zone || i.layout_zone === 'auto');

  if (explicitLeft.length > 0) {
    return [...explicitRight, ...autoItems];
  }
  return [...explicitRight, ...autoItems.slice(2)];
});

const sectionThemeClasses = computed(() => {
  const style = layoutConfig.value.cardStyle || 'glass';

  if (layoutConfig.value.theme === 'forest') {
    return {
      section: 'bg-gradient-to-br from-[#14532d] via-[#166534] to-[#0f3f22] text-white',
      subtitle: 'text-emerald-100/85',
      leftCard: style === 'solid'
        ? 'bg-[#0b381e] hover:bg-[#0e4425] border-emerald-400/25 shadow-md'
        : style === 'bordered'
          ? 'bg-white/[0.06] hover:bg-white/[0.12] border-2 border-emerald-400/50'
          : 'bg-white/[0.08] hover:bg-white/[0.14] border-white/15 backdrop-blur-xs',
      rightCard: style === 'solid'
        ? 'bg-[#0b381e] hover:bg-[#0e4425] border-emerald-400/25 shadow-md'
        : style === 'bordered'
          ? 'bg-white/[0.06] hover:bg-white/[0.12] border-2 border-emerald-400/50'
          : 'bg-white/[0.10] hover:bg-white/[0.16] border-white/15 backdrop-blur-xs',
      cardTitle: 'text-white',
      cardDesc: 'text-emerald-100/80',
      iconPill: 'bg-emerald-400/20 text-emerald-200'
    };
  }

  if (layoutConfig.value.theme === 'light') {
    return {
      section: 'bg-poster-checkers text-slate-900 border-y-4 border-[#1f7a4c]',
      subtitle: 'text-slate-600',
      leftCard: 'bg-white hover:bg-emerald-50/40 border-2 border-[#1f7a4c]/30 hover:border-[#1f7a4c] shadow-sm',
      rightCard: 'bg-white hover:bg-emerald-50/40 border-2 border-[#1f7a4c]/30 hover:border-[#1f7a4c] shadow-sm',
      cardTitle: 'text-slate-900',
      cardDesc: 'text-slate-600',
      iconPill: 'bg-emerald-100 text-[#1f7a4c]'
    };
  }

  // Thème sombre profond (#07291b)
  return {
    section: 'bg-[#07291b] text-white',
    subtitle: 'text-emerald-100/80',
    leftCard: style === 'solid'
      ? 'bg-[#0d3b28] hover:bg-[#114730] border-emerald-500/20 shadow-lg'
      : style === 'bordered'
        ? 'bg-white/[0.03] hover:bg-white/[0.07] border-2 border-emerald-500/40'
        : 'bg-white/[0.04] hover:bg-white/[0.08] border-white/10 hover:border-emerald-400/35',
    rightCard: style === 'solid'
      ? 'bg-[#0d3b28] hover:bg-[#114730] border-emerald-500/20 shadow-lg'
      : style === 'bordered'
        ? 'bg-white/[0.05] hover:bg-white/[0.10] border-2 border-emerald-500/40'
        : 'bg-white/[0.07] hover:bg-white/[0.12] border-white/10 hover:border-emerald-400/35',
    cardTitle: 'text-white',
    cardDesc: 'text-emerald-100/75',
    iconPill: 'bg-emerald-400/15 text-emerald-300'
  };
});

const sectionPaddingClass = computed(() => {
  if (layoutConfig.value.spacing === 'compact') return 'py-12 sm:py-16';
  if (layoutConfig.value.spacing === 'relaxed') return 'py-24 sm:py-32';
  return 'py-16 sm:py-24';
});

const gridGapClass = computed(() => {
  if (layoutConfig.value.gapSize === 'tight') return 'gap-3 sm:gap-3.5';
  if (layoutConfig.value.gapSize === 'wide') return 'gap-6 sm:gap-8';
  return 'gap-4 sm:gap-5';
});

const columnGridClasses = computed(() => {
  if (layoutConfig.value.split === '40-60') {
    return { left: 'lg:col-span-5', right: 'lg:col-span-7' };
  }
  if (layoutConfig.value.split === 'stacked') {
    return { left: 'lg:col-span-12', right: 'lg:col-span-12' };
  }
  return { left: 'lg:col-span-6', right: 'lg:col-span-6' };
});

const rightGridColsClass = computed(() => {
  if (layoutConfig.value.rightCols === '3') return 'grid-cols-1 sm:grid-cols-2 lg:grid-cols-3';
  return 'grid-cols-1 sm:grid-cols-2';
});

const getSquareColor = (theme) => {
  if (theme === 'teal') return 'bg-[#2dd49f] text-[#07291b]';
  if (theme === 'emerald') return 'bg-[#22c55e] text-[#07291b]';
  if (theme === 'sky') return 'bg-[#38bdf8] text-[#07291b]';
  return 'bg-[#fbbf24] text-[#07291b]';
};

// Gestion complète de la taille et des proportions du logo
const getLogoWrapperClass = (item, zone) => {
  const size = item.logo_size || 'md';
  const fit = item.logo_fit || 'contain';
  const hasWhiteBg = item.logo_bg !== 'transparent';

  const bgClasses = hasWhiteBg
    ? 'bg-white px-3 py-1.5 rounded-xl shadow-xs'
    : 'bg-transparent p-0';

  if (fit === 'original') {
    if (size === 'sm') return `${bgClasses} max-h-12 max-w-[140px]`;
    if (size === 'lg') return `${bgClasses} max-h-24 max-w-[260px]`;
    if (size === 'xl') return `${bgClasses} max-h-32 max-w-[320px]`;
    return `${bgClasses} max-h-16 max-w-[200px]`;
  }

  if (fit === 'contain') {
    if (size === 'sm') return `${bgClasses} ${zone === 'left' ? 'h-10 max-w-[120px]' : 'h-10 max-w-[140px]'}`;
    if (size === 'lg') return `${bgClasses} ${zone === 'left' ? 'h-16 max-w-[190px]' : 'h-16 max-w-[220px]'}`;
    if (size === 'xl') return `${bgClasses} ${zone === 'left' ? 'h-20 max-w-[240px]' : 'h-20 max-w-[260px]'}`;
    return `${bgClasses} ${zone === 'left' ? 'h-13 max-w-[155px]' : 'h-13 max-w-[175px]'}`;
  }

  // Carré rogné ('cover')
  if (size === 'sm') return 'w-10 h-10 rounded-xl overflow-hidden shadow-sm';
  if (size === 'lg') return 'w-16 h-16 rounded-xl overflow-hidden shadow-sm';
  if (size === 'xl') return 'w-20 h-20 rounded-xl overflow-hidden shadow-sm';
  return 'w-12 h-12 rounded-xl overflow-hidden shadow-sm';
};

const getLogoImgClass = (item) => {
  const fit = item.logo_fit || 'contain';
  if (fit === 'cover') return 'w-full h-full object-cover';
  if (fit === 'original') return 'w-auto h-auto max-h-full max-w-full object-contain';
  return 'max-h-full max-w-full w-auto h-auto object-contain';
};

onMounted(async () => {
  const [{ data: actData }, { data: cfgData }] = await Promise.all([
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

  if (actData) dbActivities.value = actData;
  if (cfgData?.value) {
    try {
      layoutConfig.value = { ...layoutConfig.value, ...JSON.parse(cfgData.value) };
    } catch (e) {
      console.warn('Config village invalide, valeurs par défaut appliquées.');
    }
  }
  loading.value = false;
});
</script>

<template>
  <section id="village" :class="[sectionThemeClasses.section, sectionPaddingClass, 'relative overflow-hidden transition-colors']">
    <!-- Halos décoratifs subtils en arrière-plan -->
    <div v-if="layoutConfig.theme !== 'light'" class="pointer-events-none absolute -top-24 -right-24 w-96 h-96 rounded-full bg-emerald-500/10 blur-3xl"></div>
    <div v-if="layoutConfig.theme !== 'light'" class="pointer-events-none absolute -bottom-24 -left-24 w-80 h-80 rounded-full bg-teal-400/10 blur-3xl"></div>

    <div class="relative z-10 max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">

      <div v-if="loading" class="text-center py-12">
        <i class="fa-solid fa-circle-notch fa-spin text-3xl text-emerald-400"></i>
      </div>

      <div v-else class="grid grid-cols-1 lg:grid-cols-12 gap-10 lg:gap-12 items-center">
        
        <!-- COLONNE GAUCHE : Titre, sous-titre et bandeaux horizontaux -->
        <div :class="[columnGridClasses.left, 'space-y-6']">
          <div>
            <h2 class="text-3xl sm:text-4xl lg:text-[44px] font-black tracking-tight leading-tight">
              Un village solidaire
            </h2>
            <p :class="[sectionThemeClasses.subtitle, 'text-sm sm:text-base mt-2.5']">
              Parce que la solidarité unit toutes les générations.
            </p>
          </div>

          <div :class="layoutConfig.split === 'stacked' ? `grid grid-cols-1 md:grid-cols-2 ${gridGapClass} pt-2` : `space-y-4 pt-2`">
            <div
              v-for="(item, idx) in leftActivities"
              :key="item.id || idx"
              :class="[sectionThemeClasses.leftCard, 'border rounded-2xl p-4 sm:p-5 flex items-center gap-4 sm:gap-5 transition-all duration-200 hover:-translate-y-0.5']"
            >
              <!-- IMAGE / LOGO PERSONNALISÉ -->
              <div
                v-if="item.image_url"
                :class="[getLogoWrapperClass(item, 'left'), 'flex-shrink-0 flex items-center justify-center']"
              >
                <img :src="item.image_url" :alt="item.title" :class="getLogoImgClass(item)" />
              </div>

              <!-- CARRÉ COLORÉ AVEC ICÔNE -->
              <div
                v-else
                :class="[getSquareColor(item.color_theme), getLogoWrapperClass({ ...item, logo_fit: 'cover' }, 'left'), 'flex-shrink-0 flex items-center justify-center text-lg']"
              >
                <i :class="['fa-solid', item.icon || 'fa-tent']"></i>
              </div>

              <div class="min-w-0 flex-1">
                <div class="flex items-center gap-2 flex-wrap">
                  <h3 :class="[sectionThemeClasses.cardTitle, 'font-extrabold text-sm sm:text-base leading-snug']">
                    {{ item.title }}
                  </h3>
                  <span v-if="item.tag" class="px-2 py-0.5 rounded text-[10px] font-bold uppercase bg-emerald-500/20 text-emerald-300">
                    {{ item.tag }}
                  </span>
                </div>
                <p v-if="item.description" :class="[sectionThemeClasses.cardDesc, 'text-xs sm:text-[13px] mt-1 leading-relaxed']">
                  {{ item.description }}
                </p>
              </div>
            </div>
          </div>
        </div>

        <!-- COLONNE DROITE : Grille de cartes centrées -->
        <div :class="[columnGridClasses.right, rightGridColsClass, gridGapClass, 'grid']">
          <div
            v-for="(item, idx) in rightActivities"
            :key="item.id || idx"
            :class="[
              sectionThemeClasses.rightCard,
              item.is_wide ? 'sm:col-span-2' : '',
              'border rounded-3xl p-6 sm:p-7 text-center flex flex-col items-center justify-center min-h-[195px] transition-all duration-200 hover:-translate-y-0.5'
            ]"
          >
            <!-- LOGO / IMAGE PERSONNALISÉE -->
            <div
              v-if="item.image_url"
              :class="[getLogoWrapperClass(item, 'right'), 'mb-3.5 flex items-center justify-center']"
            >
              <img :src="item.image_url" :alt="item.title" :class="getLogoImgClass(item)" />
            </div>

            <!-- ICÔNE SUBTILE SI PAS D'IMAGE -->
            <div
              v-else-if="item.icon"
              :class="[sectionThemeClasses.iconPill, 'w-11 h-11 rounded-2xl flex items-center justify-center text-base mb-3.5']"
            >
              <i :class="['fa-solid', item.icon]"></i>
            </div>

            <h3 :class="[sectionThemeClasses.cardTitle, 'font-extrabold text-base sm:text-lg leading-snug']">
              {{ item.title }}
            </h3>
            <p v-if="item.description" :class="[sectionThemeClasses.cardDesc, 'text-xs sm:text-[13px] mt-2 leading-relaxed']">
              {{ item.description }}
            </p>
          </div>
        </div>

      </div>

    </div>
  </section>
</template>