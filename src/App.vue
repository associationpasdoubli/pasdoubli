<script setup>
import { ref, onMounted } from 'vue';

// Import de l'image de fond depuis src/assets/
import fondUrl from './assets/fond.png';

// Composants de mise en page
import TopBanner from './components/layout/TopBanner.vue';
import AppHeader from './components/layout/AppHeader.vue';
import AppFooter from './components/layout/AppFooter.vue';

// Sections du site
import HeroSection from './components/sections/HeroSection.vue';
import AboutSection from './components/sections/AboutSection.vue';
import CauseSection from './components/sections/CauseSection.vue';
import RacesSection from './components/sections/RacesSection.vue';
import ScheduleSection from './components/sections/ScheduleSection.vue';
import VillageSection from './components/sections/VillageSection.vue';
import PracticalSection from './components/sections/PracticalSection.vue';
import PartnersSection from './components/sections/PartnersSection.vue';
import ContactSection from './components/sections/ContactSection.vue';
import MoneySection from './components/sections/MoneySection.vue';

// Espace d'administration
import AdminDashboard from './components/admin/AdminDashboard.vue';
import FaqSection from './components/sections/FaqSection.vue';

// Modales & Notifications
import RegistrationModal from './components/modals/RegistrationModal.vue';
import DonationModal from './components/modals/DonationModal.vue';
import VolunteerModal from './components/modals/VolunteerModal.vue';
import DetailsModal from './components/modals/DetailsModal.vue';
import ToastNotification from './components/ui/ToastNotification.vue';

const isAdmin = ref(window.location.hash === '#admin');

onMounted(() => {
  window.addEventListener('hashchange', () => {
    isAdmin.value = window.location.hash === '#admin';
  });
});
</script>

<template>
<!-- Bandeau d'information : Ouverture des inscriptions -->
<div class="bg-gradient-to-r from-emerald-800 via-emerald-700 to-teal-700 text-white py-2.5 px-4 text-center text-xs sm:text-sm font-semibold tracking-wide shadow-sm flex items-center justify-center gap-2">
  <span class="inline-flex items-center justify-center w-5 h-5 rounded-full bg-white/20 text-white text-[11px]">
    <i class="fa-solid fa-bell"></i>
  </span>
  <span>
    À vos baskets ! Les inscriptions ouvrent officiellement le <strong class="underline decoration-white/60 font-black">14 décembre 2026</strong>.
  </span>
</div>
  <div v-if="isAdmin">
    <AdminDashboard />
  </div>

  <div v-else class="min-h-screen w-full flex flex-col font-sans text-slate-800 bg-white">
    <AppHeader />

    <main class="w-full flex-1">
      <!-- 1. Le Hero conserve son propre fond sombre -->
      <HeroSection />

      <!-- 2. Zone avec l'image fond.png fixe et répétition désactivée -->
      <div 
  class="relative w-full bg-scroll md:bg-fixed bg-top md:bg-center bg-no-repeat bg-[length:100%_auto] md:bg-cover"
  :style="{ backgroundImage: `url(${fondUrl})` }"
>
        <!-- Voile léger pour maintenir le contraste et la lisibilité du texte -->
        <div class="absolute inset-0 bg-white/40 pointer-events-none"></div>

        <!-- Sections superposées à l'image -->
        <div class="relative z-10">
          
          <AboutSection />

          <CauseSection />
          <RacesSection />
          
          <ScheduleSection />
          <!-- <VillageSection /> -->
          <PracticalSection />
          <MoneySection />
          <PartnersSection />
          <ContactSection />
          <FaqSection />
        </div>
      </div>
    </main>

    <AppFooter />

    <!-- Modales & Notifications globales -->
    <RegistrationModal />
    <DonationModal />
    <VolunteerModal />
    <DetailsModal />
    <ToastNotification />
  </div>
</template>