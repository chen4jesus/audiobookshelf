<template>
  <div class="min-h-screen bg-slate-900 text-white font-sans selection:bg-blue-500/30">
    <!-- Animated Background -->
    <div class="fixed inset-0 pointer-events-none overflow-hidden">
      <div class="absolute -top-1/4 -left-1/4 w-1/2 h-1/2 bg-blue-600/10 blur-[120px] rounded-full animate-pulse"></div>
      <div class="absolute -bottom-1/4 -right-1/4 w-1/2 h-1/2 bg-indigo-600/10 blur-[120px] rounded-full animate-pulse" style="animation-delay: 2s"></div>
    </div>

    <!-- Layout Wrapper -->
    <div class="relative z-10 max-w-5xl mx-auto px-4 py-8 md:py-16">
      <!-- Header -->
      <header class="text-center mb-8">
        <h1 class="text-4xl md:text-6xl font-black mb-4 bg-gradient-to-r from-blue-400 to-indigo-400 bg-clip-text text-transparent">{{ $strings.PageShareTitle }}</h1>
        <p class="text-slate-400 text-lg md:text-xl font-medium max-w-2xl mx-auto">{{ $strings.PageShareSubheader }}</p>
      </header>

      <!-- Advanced Search & Filter Bar -->
      <div class="mb-10 space-y-4">
        <div class="relative group/search">
          <span class="absolute left-5 top-1/2 -translate-y-1/2 material-symbols text-slate-500 group-focus-within/search:text-blue-400 transition-colors">search</span>
          <input v-model="searchQuery" type="text" :placeholder="$strings.PlaceholderSearch" class="w-full bg-white/5 border border-white/10 rounded-2xl py-4 pl-14 pr-6 text-white placeholder-slate-500 focus:outline-none focus:border-blue-500/50 focus:bg-white/[0.08] transition-all" @input="debouncedFetch" />
        </div>

        <!-- Alphabetical Navigation -->
        <div class="flex flex-wrap justify-center gap-1 sm:gap-2 px-2">
          <button
            v-for="char in alphabet"
            :key="char"
            @click="toggleInitial(char)"
            class="w-8 h-8 sm:w-10 sm:h-10 rounded-xl flex items-center justify-center font-bold text-xs sm:text-sm transition-all border"
            :class="selectedInitial === char ? 'bg-blue-600 border-blue-500 text-white shadow-lg shadow-blue-600/30' : 'bg-white/5 border-white/5 text-slate-400 hover:bg-white/10 hover:text-slate-200'"
          >
            {{ char }}
          </button>
          <button @click="clearFilters" class="px-4 h-8 sm:h-10 rounded-xl flex items-center justify-center font-bold text-xs sm:text-sm transition-all border bg-white/5 border-white/5 text-slate-400 hover:bg-white/10 hover:text-slate-200" v-if="searchQuery || selectedInitial">
            {{ $strings.ButtonClearFilter }}
          </button>
        </div>
      </div>

      <!-- Loading State -->
      <div v-if="loading" class="flex flex-col items-center justify-center py-24">
        <div class="relative w-16 h-16">
          <div class="absolute inset-0 border-4 border-blue-500/20 rounded-full"></div>
          <div class="absolute inset-0 border-4 border-blue-500 border-t-transparent rounded-full animate-spin"></div>
        </div>
        <p class="mt-4 text-slate-500 font-medium animate-pulse">{{ $strings.MessageLoadingSharedLibrary }}</p>
      </div>

      <!-- Empty State -->
      <div v-else-if="groups.length === 0" class="text-center py-32 glass rounded-3xl border border-white/5 mx-auto max-w-xl">
        <span class="material-symbols text-6xl text-slate-700 mb-4 block">folder_off</span>
        <h3 class="text-2xl font-bold text-slate-400 mb-2">{{ searchQuery || selectedInitial ? $strings.MessageBookshelfNoResultsForQuery : $strings.MessageNoActiveShares }}</h3>
        <p class="text-slate-500">{{ searchQuery || selectedInitial ? '' : $strings.MessageNoPublicSharesAvailable }}</p>
      </div>

      <!-- Tree View List -->
      <div v-else class="space-y-6">
        <div v-for="group in groups" :key="group.name" class="group/category">
          <div @click="toggleGroup(group.name)" class="flex items-center justify-between p-5 rounded-2xl glass border border-white/5 cursor-pointer hover:bg-white/5 transition-all duration-300">
            <div class="flex items-center space-x-4">
              <div class="w-10 h-10 flex items-center justify-center rounded-xl bg-blue-500/10 text-blue-400 group-hover/category:scale-110 transition-transform">
                <span v-if="isGroupLoading(group.name)" class="material-symbols text-2xl animate-spin">progress_activity</span>
                <span v-else class="material-symbols text-2xl transition-transform duration-300" :class="isGroupOpen(group.name) ? 'rotate-90' : ''">chevron_right</span>
              </div>
              <div>
                <h2 class="text-xl font-bold text-slate-200" v-html="highlightText(group.name)"></h2>
                <p class="text-sm text-slate-500 uppercase tracking-wider font-semibold">{{ group.count === 1 ? $getString('LabelSharedItemCount', [group.count]) : $getString('LabelSharedItemsCount', [group.count]) }}</p>
              </div>
            </div>
          </div>

          <transition enter-active-class="transition duration-300 ease-out" enter-from-class="transform -translate-y-4 opacity-0" enter-to-class="transform translate-y-0 opacity-100" leave-active-class="transition duration-200 ease-in" leave-from-class="transform translate-y-0 opacity-100" leave-to-class="transform -translate-y-4 opacity-0">
            <div v-show="isGroupOpen(group.name)" class="mt-3 ml-4 md:ml-8 space-y-3">
              <!-- Loading state for group items -->
              <div v-if="isGroupLoading(group.name)" class="flex items-center justify-center p-8">
                <div class="relative w-8 h-8">
                  <div class="absolute inset-0 border-2 border-blue-500/20 rounded-full"></div>
                  <div class="absolute inset-0 border-2 border-blue-500 border-t-transparent rounded-full animate-spin"></div>
                </div>
                <p class="ml-3 text-slate-500 text-sm">{{ $strings.MessageLoadingItems }}</p>
              </div>

              <!-- Group items -->
              <div v-else v-for="share in getGroupShares(group.name)" :key="share.id" class="flex items-center justify-between p-3 sm:p-4 pl-4 sm:pl-6 rounded-2xl bg-white/5 border border-white/5 hover:bg-white/[0.08] hover:border-blue-500/30 transition-all duration-300 group/item">
                <div class="flex-1 min-w-0 pr-2 sm:pr-4">
                  <h3 class="font-bold text-slate-100 text-base sm:text-lg line-clamp-2 leading-tight mb-1" v-html="highlightText(share.title)"></h3>
                  <div class="flex items-center space-x-3">
                    <span class="text-[10px] sm:text-xs px-2 py-0.5 rounded-full bg-slate-800 text-slate-400 font-bold uppercase tracking-tighter">{{ share.mediaItemType === 'book' ? $strings.LabelAudiobook : $strings.LabelPodcast }}</span>
                    <span v-if="share.expiresAt" class="text-[10px] sm:text-xs text-slate-500">Exp: {{ formatDate(share.expiresAt) }}</span>
                  </div>
                </div>

                <div class="flex items-center flex-shrink-0 ml-2">
                  <button @click="playShare(share)" class="group-hover/item:scale-105 transition-all duration-300 h-10 sm:h-12 px-3 sm:px-6 rounded-xl bg-blue-600 hover:bg-blue-500 flex items-center space-x-1 sm:space-x-2 font-bold shadow-lg shadow-blue-600/20 active:scale-95">
                    <span class="material-symbols text-lg sm:text-xl">play_circle</span>
                    <span class="text-sm sm:text-base">{{ $strings.ButtonListen }}</span>
                  </button>
                </div>
              </div>
            </div>
          </transition>
        </div>
      </div>
    </div>

    <!-- Footer -->
    <footer class="max-w-5xl mx-auto px-4 py-12 text-center text-slate-600 text-sm font-medium">&copy; {{ new Date().getFullYear() }} Listen Faithfully &bull; {{ $strings.LabelSharedLibraryAccess }}</footer>
  </div>
</template>

<script>
export default {
  layout: 'blank',
  head() {
    return {
      title: `${this.$strings.PageShareTitle} | Listen Faithfully`,
      meta: [{ hid: 'description', name: 'description', content: this.$strings.PageShareMetaDescription }],
      link: [{ rel: 'stylesheet', href: 'https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@20..48,100..700,0..1,-50..200' }]
    }
  },
  data() {
    return {
      groups: [], // Array of { name, count }
      groupShares: {}, // Map of groupName -> shares array (lazy loaded)
      loadingGroups: {}, // Map of groupName -> boolean (loading state)
      openGroups: {}, // Map of groupName -> boolean (open state)
      loading: true,
      searchQuery: '',
      selectedInitial: null,
      alphabet: 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.split(''),
      debounceTimeout: null
    }
  },
  methods: {
    async fetchGroups() {
      try {
        this.loading = true

        let url = '/public/shares/groups'
        const params = new URLSearchParams()
        if (this.searchQuery) params.append('search', this.searchQuery)
        if (this.selectedInitial) params.append('initial', this.selectedInitial)

        if (params.toString()) url += `?${params.toString()}`

        // Fetch only the group summaries (lightweight)
        const [data] = await Promise.all([this.$axios.$get(url), new Promise((resolve) => setTimeout(resolve, 400))])
        this.groups = data
      } catch (error) {
        console.error('Failed to fetch share groups', error)
        if (this.$toast) this.$toast.error('Failed to load shared media')
      } finally {
        this.loading = false
      }
    },
    debouncedFetch() {
      clearTimeout(this.debounceTimeout)
      this.debounceTimeout = setTimeout(() => {
        this.fetchGroups()
      }, 500)
    },
    toggleInitial(char) {
      if (this.selectedInitial === char) {
        this.selectedInitial = null
      } else {
        this.selectedInitial = char
      }
      this.fetchGroups()
    },
    clearFilters() {
      this.searchQuery = ''
      this.selectedInitial = null
      this.fetchGroups()
    },
    async fetchGroupShares(groupName) {
      if (this.groupShares[groupName] || this.loadingGroups[groupName]) {
        // Already loaded or currently loading
        return
      }

      try {
        this.$set(this.loadingGroups, groupName, true)
        const encodedName = encodeURIComponent(groupName)
        const data = await this.$axios.$get(`/public/shares/group/${encodedName}`)
        this.$set(this.groupShares, groupName, data)
      } catch (error) {
        console.error(`Failed to fetch shares for group "${groupName}"`, error)
        if (this.$toast) this.$toast.error(`Failed to load items for ${groupName}`)
        // Set empty array on error so we don't keep retrying
        this.$set(this.groupShares, groupName, [])
      } finally {
        this.$set(this.loadingGroups, groupName, false)
      }
    },
    async toggleGroup(groupName) {
      const isCurrentlyOpen = this.openGroups[groupName]

      if (!isCurrentlyOpen) {
        // Opening the group - fetch shares if not loaded
        this.$set(this.openGroups, groupName, true)
        await this.fetchGroupShares(groupName)
      } else {
        // Closing the group
        this.$set(this.openGroups, groupName, false)
      }
    },
    isGroupOpen(groupName) {
      return !!this.openGroups[groupName]
    },
    isGroupLoading(groupName) {
      return !!this.loadingGroups[groupName]
    },
    getGroupShares(groupName) {
      const shares = this.groupShares[groupName] || []
      if (!this.searchQuery) return shares

      const search = this.searchQuery.toLowerCase()
      const groupMatches = groupName.toLowerCase().includes(search)

      return shares.filter((share) => {
        // Show if group name matches OR item title matches
        return groupMatches || (share.title && share.title.toLowerCase().includes(search))
      })
    },
    playShare(share) {
      this.$router.push(`/share/${share.slug}`)
    },
    formatDate(date) {
      if (!date) return ''
      return new Date(date).toLocaleDateString([], { month: 'short', day: 'numeric', year: 'numeric' })
    },
    highlightText(text) {
      if (!this.searchQuery) return text
      const regex = new RegExp(`(${this.searchQuery.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi')
      return text.replace(regex, '<span class="text-blue-400 bg-blue-400/10 rounded px-0.5 shadow-sm">$1</span>')
    }
  },
  mounted() {
    this.fetchGroups()
  }
}
</script>

<style scoped>
.glass {
  background: rgba(255, 255, 255, 0.03);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
}

.material-symbols {
  font-family: 'Material Symbols Outlined';
  font-weight: normal;
  font-style: normal;
  font-size: 24px;
  line-height: 1;
  letter-spacing: normal;
  text-transform: none;
  display: inline-block;
  white-space: nowrap;
  word-wrap: normal;
  direction: ltr;
  -webkit-font-smoothing: antialiased;
}

/* Custom Scrollbar */
::-webkit-scrollbar {
  width: 8px;
}
::-webkit-scrollbar-track {
  background: transparent;
}
::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
}
::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.2);
}
</style>
