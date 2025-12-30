<script setup>
import { ref, watch, onMounted, nextTick } from 'vue'
import { useRouter } from 'vue-router'
import { useAuthStore } from '@/stores/auth'
import { useRoomStore } from '@/stores/room'
import { useToast } from '@/composables/useToast'
import Button from '@/components/ui/Button.vue'
import Input from '@/components/ui/Input.vue'
import Badge from '@/components/ui/Badge.vue'
import Avatar from '@/components/ui/Avatar.vue'
import AvatarImage from '@/components/ui/AvatarImage.vue'
import AvatarFallback from '@/components/ui/AvatarFallback.vue'
import ScrollArea from '@/components/ui/ScrollArea.vue'
import { Crown, Send, LogOut, Users, Copy, Loader2, Home, X, User, Mail, Briefcase, Code } from 'lucide-vue-next'

const router = useRouter()
const authStore = useAuthStore()
const roomStore = useRoomStore()
const { toast } = useToast()

const messageInput = ref('')
const messagesEndRef = ref(null)
const isInitialLoad = ref(true)

// Profile Modal State
const showProfileModal = ref(false)
const selectedMember = ref(null)

const openProfileModal = (member) => {
  if (member.id === authStore.user?.id) return // Don't show modal for self
  selectedMember.value = member
  showProfileModal.value = true
}

const closeProfileModal = () => {
  showProfileModal.value = false
  selectedMember.value = null
}

const getMemberById = (userId) => {
  return roomStore.activeRoom?.members?.find(m => m.id === userId) || null
}

const openProfileFromChat = (userId) => {
  if (userId === authStore.user?.id) return
  const member = getMemberById(userId)
  if (member) {
    openProfileModal(member)
  }
}

const scrollToBottom = () => {
  nextTick(() => {
    messagesEndRef.value?.scrollIntoView({ behavior: 'smooth' })
  })
}

watch(
  () => roomStore.messages,
  () => {
    scrollToBottom()
  },
  { deep: true }
)

onMounted(() => {
  isInitialLoad.value = false
})

// Redirect logic
watch(
  () => [roomStore.activeRoom, roomStore.isReconnecting],
  ([activeRoom, isReconnecting]) => {
    if (isInitialLoad.value) return
    if (isReconnecting) return

    const hasSavedRoom = localStorage.getItem('activeRoom')
    if (!activeRoom && !hasSavedRoom) {
      router.push('/dashboard')
    }
  }
)

const isLeader = () => {
  return roomStore.activeRoom?.leaderId === authStore.user?.id
}

const handleSendMessage = () => {
  if (!messageInput.value.trim()) return
  roomStore.sendMessage(messageInput.value)
  messageInput.value = ''
}

const handleEndSession = async () => {
  await roomStore.endSession()
  toast({
    title: 'Sesi selesai',
    description: 'Room telah ditutup oleh leader.'
  })
  router.push('/dashboard')
}

const handleLeaveRoom = async () => {
  await roomStore.leaveRoom()
  toast({
    title: 'Keluar dari room',
    description: 'Anda telah meninggalkan tim.'
  })
  router.push('/dashboard')
}

const handleBackToDashboard = () => {
  roomStore.clearAutoNavigate()
  toast({
    title: 'Kembali ke Dashboard',
    description: 'Anda tetap terhubung di room ini.'
  })
  router.push('/dashboard')
}

const handleCopyRoomId = () => {
  navigator.clipboard.writeText(roomStore.activeRoom?.id || '')
  toast({
    title: 'Disalin',
    description: 'ID Room disalin ke clipboard.'
  })
}
</script>

<template>
  <!-- Loading: Reconnecting -->
  <div v-if="roomStore.isReconnecting" class="h-screen flex flex-col items-center justify-center bg-gradient-to-br from-cyan-50 via-white to-teal-50">
    <div class="w-20 h-20 bg-gradient-to-br from-cyan-500 to-teal-500 rounded-2xl flex items-center justify-center shadow-lg mb-6">
      <Loader2 class="h-10 w-10 text-white animate-spin" />
    </div>
    <h2 class="text-xl font-bold bg-gradient-to-r from-cyan-600 to-teal-500 bg-clip-text text-transparent mb-2">
      Menghubungkan...
    </h2>
    <p class="text-slate-400 text-sm">
      Menghubungkan kembali ke room
    </p>
  </div>

  <!-- Loading: Has saved room but not loaded yet -->
  <div v-else-if="!roomStore.activeRoom && localStorage.getItem('activeRoom')" class="h-screen flex flex-col items-center justify-center bg-gradient-to-br from-cyan-50 via-white to-teal-50">
    <div class="w-20 h-20 bg-gradient-to-br from-cyan-500 to-teal-500 rounded-2xl flex items-center justify-center shadow-lg mb-6">
      <Loader2 class="h-10 w-10 text-white animate-spin" />
    </div>
    <h2 class="text-xl font-bold bg-gradient-to-r from-cyan-600 to-teal-500 bg-clip-text text-transparent mb-2">
      Memuat Room
    </h2>
    <p class="text-slate-400 text-sm">Mohon tunggu sebentar...</p>
  </div>

  <!-- No active room -->
  <div v-else-if="!roomStore.activeRoom"></div>

  <!-- Room Content -->
  <div v-else class="h-screen flex flex-col bg-gradient-to-br from-slate-50 via-white to-cyan-50">
    <!-- Header -->
    <div class="bg-white/80 backdrop-blur-md border-b border-slate-100 px-4 py-3">
      <div class="max-w-7xl mx-auto flex justify-between items-center">
        <div class="flex items-center gap-4">
          <div class="flex items-center gap-2">
            <div class="w-8 h-8 bg-gradient-to-br from-cyan-500 to-teal-500 rounded-lg flex items-center justify-center">
              <Users class="h-4 w-4 text-white" />
            </div>
            <span class="font-bold bg-gradient-to-r from-cyan-600 to-teal-500 bg-clip-text text-transparent hidden sm:inline">GroupMatch</span>
          </div>
          <div class="h-6 w-px bg-slate-200 hidden sm:block"></div>
          <div class="flex items-center gap-2 bg-slate-100 px-3 py-1.5 rounded-lg">
            <span class="text-xs text-slate-500">Room:</span>
            <code class="text-sm font-mono font-medium text-slate-700">
              {{ roomStore.activeRoom?.id ? String(roomStore.activeRoom.id).slice(-8) : '...' }}
            </code>
            <Button size="icon" variant="ghost" class="h-6 w-6" @click="handleCopyRoomId">
              <Copy class="h-3 w-3" />
            </Button>
          </div>
        </div>
        <div class="flex items-center gap-2">
          <Badge class="bg-gradient-to-r from-cyan-500 to-teal-500 text-white border-0 shadow-sm">
            <Users class="h-3 w-3 mr-1" />
            {{ roomStore.activeRoom?.members?.length || 0 }} Anggota
          </Badge>
        </div>
      </div>
    </div>

    <!-- Navbar -->
    <div class="bg-white border-b border-slate-100 px-4 py-3">
      <div class="max-w-7xl mx-auto flex justify-between items-center">
        <h1 class="text-lg font-bold text-slate-800 flex items-center gap-2">
          <Crown class="h-5 w-5 text-amber-500" />
          Team Workspace
        </h1>
        <div class="flex gap-2">
          <Button variant="ghost" size="sm" @click="handleBackToDashboard" class="text-slate-600 hover:text-slate-800">
            <Home class="mr-2 h-4 w-4" /> Dashboard
          </Button>
          <Button v-if="isLeader()" variant="destructive" size="sm" @click="handleEndSession" class="shadow-sm">
            <Crown class="mr-2 h-4 w-4" /> End Session
          </Button>
          <Button variant="outline" size="sm" @click="handleLeaveRoom" class="border-slate-200">
            <LogOut class="mr-2 h-4 w-4" /> Keluar
          </Button>
        </div>
      </div>
    </div>

    <div class="flex-1 flex max-w-7xl mx-auto w-full p-4 gap-4 overflow-hidden">
      <!-- Sidebar - Members -->
      <div class="w-72 hidden md:flex bg-white/80 backdrop-blur-sm rounded-2xl border border-slate-100 shadow-lg flex-col overflow-hidden">
        <div class="p-4 border-b border-slate-100 bg-gradient-to-r from-cyan-50 to-teal-50">
          <h3 class="font-semibold text-slate-700 flex items-center gap-2">
            <Users class="h-4 w-4 text-cyan-600" /> Anggota Tim ({{ roomStore.activeRoom?.members?.length || 0 }})
          </h3>
        </div>
        <div class="flex-1 p-3 space-y-2 overflow-y-auto">
          <div v-if="!roomStore.activeRoom?.members || roomStore.activeRoom.members.length === 0" class="text-center py-4">
            <p class="text-sm text-slate-400">Memuat anggota...</p>
          </div>
          <div
            v-for="m in roomStore.activeRoom?.members || []"
            :key="m.id"
            :class="[
              'flex gap-3 p-3 rounded-xl transition-all',
              m.id === authStore.user?.id
                ? 'bg-gradient-to-r from-cyan-50 to-teal-50 border border-cyan-100'
                : 'hover:bg-slate-50 cursor-pointer hover:shadow-sm'
            ]"
            @click="openProfileModal(m)"
          >
            <Avatar class="h-10 w-10 ring-2 ring-white shadow-sm">
              <AvatarImage :src="m.avatar || ''" />
              <AvatarFallback class="bg-gradient-to-br from-cyan-500 to-teal-500 text-white text-sm">
                {{ m.name?.[0] || 'U' }}
              </AvatarFallback>
            </Avatar>
            <div class="flex-1 min-w-0">
              <p class="text-sm font-medium text-slate-700 truncate flex items-center gap-1">
                <template v-if="m.id === authStore.user?.id">
                  <span class="text-cyan-600">(Kamu)</span>
                </template>
                <template v-else>
                  {{ m.name || m.username || `User ${String(m.id).slice(-4)}` }}
                </template>
                <Crown v-if="m.id === roomStore.activeRoom?.leaderId" class="h-3.5 w-3.5 text-amber-500" />
              </p>
              <p class="text-xs text-slate-500 truncate">
                {{ m.role || 'Member' }}<template v-if="m.username"> • @{{ m.username }}</template>
              </p>
            </div>
          </div>
        </div>
      </div>

      <!-- Chat Area -->
      <div class="flex-1 bg-white/80 backdrop-blur-sm rounded-2xl border border-slate-100 shadow-lg flex flex-col overflow-hidden">
        <div class="p-4 border-b border-slate-100 bg-gradient-to-r from-slate-50 to-white">
          <h3 class="font-semibold text-slate-700 flex items-center gap-2">
            <Send class="h-4 w-4 text-cyan-600" /> Chat Room
          </h3>
        </div>

        <ScrollArea class="flex-1 p-4">
          <div class="space-y-4">
            <div v-if="roomStore.messages.length === 0" class="text-center py-12">
              <div class="w-16 h-16 bg-slate-100 rounded-full flex items-center justify-center mx-auto mb-4">
                <Send class="h-8 w-8 text-slate-300" />
              </div>
              <p class="text-slate-400 text-sm">Belum ada pesan</p>
              <p class="text-slate-300 text-xs mt-1">Mulai percakapan dengan timmu!</p>
            </div>
            <div
              v-for="msg in roomStore.messages"
              :key="msg.id"
              :class="['flex gap-3', msg.userId === authStore.user?.id ? 'flex-row-reverse' : '']"
            >
              <Avatar 
                :class="[
                  'h-8 w-8 ring-2 ring-white shadow-sm flex-shrink-0',
                  msg.userId !== authStore.user?.id ? 'cursor-pointer hover:ring-cyan-300 transition-all' : ''
                ]"
                @click="openProfileFromChat(msg.userId)"
              >
                <AvatarFallback :class="[
                  'text-xs',
                  msg.userId === authStore.user?.id
                    ? 'bg-gradient-to-br from-cyan-500 to-teal-500 text-white'
                    : 'bg-slate-200'
                ]">
                  {{ msg.username?.[0] || 'U' }}
                </AvatarFallback>
              </Avatar>
              <div :class="['max-w-[70%]', msg.userId === authStore.user?.id ? 'text-right' : '']">
                <p class="text-xs text-slate-400 mb-1 px-1">
                  {{ msg.userId === authStore.user?.id ? 'Kamu' : msg.username }}
                </p>
                <div :class="[
                  'px-4 py-2.5 rounded-2xl text-sm shadow-sm',
                  msg.userId === authStore.user?.id
                    ? 'bg-gradient-to-r from-cyan-500 to-teal-500 text-white rounded-br-md'
                    : 'bg-slate-100 text-slate-700 rounded-bl-md'
                ]">
                  {{ msg.text }}
                </div>
              </div>
            </div>
            <div ref="messagesEndRef" />
          </div>
        </ScrollArea>

        <form @submit.prevent="handleSendMessage" class="p-4 border-t border-slate-100 bg-slate-50/80 flex gap-3">
          <Input
            v-model="messageInput"
            placeholder="Ketik pesan..."
            class="flex-1 h-11 bg-white border-slate-200 focus:border-cyan-300 rounded-xl"
          />
          <Button
            type="submit"
            :disabled="!messageInput.trim()"
            class="h-11 px-5 bg-gradient-to-r from-cyan-600 to-teal-500 hover:from-cyan-700 hover:to-teal-600 rounded-xl shadow-md"
          >
            <Send class="h-4 w-4" />
          </Button>
        </form>
      </div>
    </div>

    <!-- Profile Modal -->
    <Teleport to="body">
      <Transition name="fade">
        <div 
          v-if="showProfileModal && selectedMember" 
          class="fixed inset-0 z-50 flex items-center justify-center p-4"
        >
          <!-- Backdrop -->
          <div 
            class="absolute inset-0 bg-black/50 backdrop-blur-sm" 
            @click="closeProfileModal"
          ></div>
          
          <!-- Modal Content -->
          <div class="relative bg-white rounded-2xl shadow-2xl max-w-md w-full overflow-hidden animate-scale-in">
            <!-- Header with gradient -->
            <div class="bg-gradient-to-br from-cyan-500 to-teal-500 px-6 py-8 text-center relative">
              <Button 
                variant="ghost" 
                size="icon" 
                class="absolute top-3 right-3 text-white/80 hover:text-white hover:bg-white/20"
                @click="closeProfileModal"
              >
                <X class="h-5 w-5" />
              </Button>
              
              <Avatar class="h-24 w-24 mx-auto ring-4 ring-white shadow-lg">
                <AvatarImage :src="selectedMember.avatar || ''" />
                <AvatarFallback class="bg-white text-cyan-600 text-2xl font-bold">
                  {{ selectedMember.name?.[0] || selectedMember.username?.[0] || 'U' }}
                </AvatarFallback>
              </Avatar>
              
              <h2 class="mt-4 text-xl font-bold text-white flex items-center justify-center gap-2">
                {{ selectedMember.name || selectedMember.username || `User ${String(selectedMember.id).slice(-4)}` }}
                <Crown v-if="selectedMember.id === roomStore.activeRoom?.leaderId" class="h-5 w-5 text-amber-300" />
              </h2>
              
              <p v-if="selectedMember.username" class="text-white/80 text-sm mt-1">
                @{{ selectedMember.username }}
              </p>
            </div>
            
            <!-- Profile Info -->
            <div class="p-6 space-y-4">
              <!-- Role -->
              <div class="flex items-center gap-3 p-3 bg-slate-50 rounded-xl">
                <div class="w-10 h-10 bg-gradient-to-br from-cyan-100 to-teal-100 rounded-lg flex items-center justify-center">
                  <Briefcase class="h-5 w-5 text-cyan-600" />
                </div>
                <div>
                  <p class="text-xs text-slate-400 uppercase tracking-wider">Role</p>
                  <p class="text-sm font-medium text-slate-700">{{ selectedMember.role || 'Member' }}</p>
                </div>
              </div>
              
              <!-- Skills -->
              <div class="flex items-start gap-3 p-3 bg-slate-50 rounded-xl">
                <div class="w-10 h-10 bg-gradient-to-br from-cyan-100 to-teal-100 rounded-lg flex items-center justify-center flex-shrink-0">
                  <Code class="h-5 w-5 text-cyan-600" />
                </div>
                <div class="flex-1">
                  <p class="text-xs text-slate-400 uppercase tracking-wider mb-2">Skills</p>
                  <div v-if="selectedMember.skills && selectedMember.skills.length > 0" class="flex flex-wrap gap-2">
                    <Badge 
                      v-for="skill in selectedMember.skills" 
                      :key="skill"
                      class="bg-gradient-to-r from-cyan-500 to-teal-500 text-white text-xs border-0"
                    >
                      {{ skill }}
                    </Badge>
                  </div>
                  <p v-else class="text-sm text-slate-400 italic">Belum ada skills</p>
                </div>
              </div>
              
              <!-- Email (if available) -->
              <div v-if="selectedMember.email" class="flex items-center gap-3 p-3 bg-slate-50 rounded-xl">
                <div class="w-10 h-10 bg-gradient-to-br from-cyan-100 to-teal-100 rounded-lg flex items-center justify-center">
                  <Mail class="h-5 w-5 text-cyan-600" />
                </div>
                <div>
                  <p class="text-xs text-slate-400 uppercase tracking-wider">Email</p>
                  <p class="text-sm font-medium text-slate-700">{{ selectedMember.email }}</p>
                </div>
              </div>
              
              <!-- User ID -->
              <div class="flex items-center gap-3 p-3 bg-slate-50 rounded-xl">
                <div class="w-10 h-10 bg-gradient-to-br from-cyan-100 to-teal-100 rounded-lg flex items-center justify-center">
                  <User class="h-5 w-5 text-cyan-600" />
                </div>
                <div>
                  <p class="text-xs text-slate-400 uppercase tracking-wider">User ID</p>
                  <p class="text-sm font-medium text-slate-700 font-mono">{{ String(selectedMember.id).slice(-8) }}</p>
                </div>
              </div>
              
              <!-- Leader Badge -->
              <div v-if="selectedMember.id === roomStore.activeRoom?.leaderId" class="flex items-center gap-2 p-3 bg-amber-50 rounded-xl border border-amber-200">
                <Crown class="h-5 w-5 text-amber-500" />
                <span class="text-sm font-medium text-amber-700">Team Leader</span>
              </div>
            </div>
            
            <!-- Footer -->
            <div class="px-6 pb-6">
              <Button 
                class="w-full bg-gradient-to-r from-cyan-600 to-teal-500 hover:from-cyan-700 hover:to-teal-600"
                @click="closeProfileModal"
              >
                Tutup
              </Button>
            </div>
          </div>
        </div>
      </Transition>
    </Teleport>
  </div>
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.animate-scale-in {
  animation: scaleIn 0.2s ease-out;
}

@keyframes scaleIn {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}
</style>
