<template>
  <v-app class="h-100">
    <router-view />
  </v-app>
</template>

<script setup>
import 'spacegt/dist/style.css'
import { TokenApi } from '@/api/accounts/token'
import { onMounted } from 'vue'
import { useRouter } from 'vue-router'
import spacegt from 'spacegt'

const authorizationStore = spacegt.stores.useAuthorizationStore()
const accountsStore = spacegt.stores.useAccountsStore()
const router = useRouter()

onMounted(async () => {
  if (authorizationStore.token) {
    const res = await TokenApi.validate(authorizationStore.token)

    if (!res) {
      accountsStore.account = null
      accountsStore.authorities = []

      authorizationStore.token = ''

      router.push('/login')
    }
  } else {
    accountsStore.account = null
    accountsStore.authorities = []

    authorizationStore.token = ''

    router.push('/login')
  }
}) 
</script>
<style>
html,
body,
#app {
  height: 100%;
}
</style>