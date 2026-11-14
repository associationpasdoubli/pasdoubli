<script setup>
import { useModal } from '../../composables/useModal';
import Countdown from '../sections/Countdown.vue';

const { openModal } = useModal();
</script>

<template>
  <section id="hero" class="relative bg-poster-checkers text-slate-900 pt-12 pb-20 sm:pt-16 sm:pb-24 border-b-4 border-[#1f7a4c] overflow-hidden">
    <!-- Voile blanc doux pour garder le damier subtil -->
    <div class="absolute inset-0 bg-gradient-to-b from-white/50 via-white/90 to-white/85 pointer-events-none"></div>

    <div class="relative max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
      <div class="grid grid-cols-1 lg:grid-cols-12 gap-10 items-center">
        
        <!-- Colonne Gauche : Titres & Infos -->
        <div class="lg:col-span-7 space-y-5 text-center lg:text-left">
          
          <div>
            <!-- Phrase d'accroche (catchphrase) -->
            <p class="text-xs sm:text-sm font-black uppercase tracking-[0.2em] text-[#14b8a6] mb-2 flex items-center justify-center lg:justify-start gap-2">
              <span class="w-6 h-[2.5px] bg-[#14b8a6] rounded-full inline-block"></span>
              <span>Ensemble, créons nos souvenirs</span>
            </p>

            <!-- Titre principal avec dégradé de vert -->
            <h1 class="text-4xl sm:text-6xl lg:text-[62px] font-black uppercase tracking-tight leading-[0.95] text-transparent bg-clip-text bg-gradient-to-r from-[#14532d] via-[#1f7a4c] to-[#10b981] pb-1">
              COURSE ET MARCHE<br />
              PAS D’OUBLI
            </h1>

            <!-- Sous-titre association -->
            <div class="mt-3">
              <span class=" bg-sky-100  text-[#1e6091] font-black text-sm sm:text-lg uppercase tracking-wider px-4 py-1 rounded-lg shadow-2xs">
                AU PROFIT DE FRANCE ALZHEIMER 37
              </span>
            </div>
          </div>

          <!-- Ruban vert incliné : Date officielle -->
          <div class="flex justify-center lg:justify-start pt-1">
            <div class="bg-[#1f7a4c] text-white font-black text-lg sm:text-2xl uppercase tracking-wider px-7 py-3 ribbon-slant shadow-md inline-block">
              DIMANCHE 23 MAI 2027
            </div>
          </div>

          <!-- Bloc Lieu -->
          <div class="flex items-center justify-center lg:justify-start gap-3.5 pt-1">
            <div class="w-11 h-11 rounded-full bg-[#1f7a4c] text-white flex items-center justify-center text-xl shadow-sm flex-shrink-0">
              <i class="fa-solid fa-location-dot"></i>
            </div>
            <div class="text-left leading-tight">
              <span class="block text-xs sm:text-sm font-black uppercase tracking-wider text-slate-800">
                SQUARE DES ANCIENS COMBATTANTS
              </span>
              <span class="block text-xl sm:text-2xl font-bold text-[#1f7a4c]">
                Athée-sur-Cher
              </span>
            </div>
          </div>

          

          <!-- Boutons d'action -->
          <div class="pt-3 flex flex-col sm:flex-row items-center justify-center lg:justify-start gap-4">
            <a
              href="#epreuves"
              class="w-full sm:w-auto px-8 py-4 bg-[#1f7a4c] hover:bg-[#166534] text-white font-black text-sm uppercase tracking-wider ribbon-slant-both shadow-lg transition-transform duration-200 hover:-translate-y-0.5 text-center flex items-center justify-center gap-2.5"
            >
              <i class="fa-solid fa-person-running text-base"></i>
              <span>Inscription en ligne</span>
            </a>

            <button
              @click="openModal('don')"
              class="w-full sm:w-auto px-7 py-3.5 bg-white hover:bg-sky-50 text-[#1e6091] border-[3px] border-[#1e6091] font-black text-sm uppercase tracking-wider rounded-xl shadow-sm transition-transform duration-200 hover:-translate-y-0.5 flex items-center justify-center gap-2 cursor-pointer"
            >
              <i class="fa-solid fa-heart text-rose-500"></i>
              <span>Faire un don</span>
            </button>
          </div>
        </div>

        <!-- Colonne Droite : Le Countdown hexagonal -->
        <div class="lg:col-span-5 flex justify-center lg:justify-end">
          <Countdown />
        </div>

      </div>
    </div>
  </section>
</template>