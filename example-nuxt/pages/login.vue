<template>
  <v-container>
    <v-layout row>
      <v-flex xs12 sm6 offset-sm3>
        <v-card>
          <v-card-text>
            <!-- <v-container v-if="!(user && !user.verified)"> -->
            <v-container v-if="otpCount <= 0">
              <!-- <v-img src="/static/logo-main.png" /> -->
              <form @keydown.enter="login">
                <v-layout row>
                  <v-flex xs12>
                    <v-text-field label="Email" v-model="email" type="text" required></v-text-field>
                  </v-flex>
                </v-layout>
                <v-layout row>
                  <v-flex xs12>
                    <v-text-field label="Password" v-model="password" type="password" required></v-text-field>
                  </v-flex>
                </v-layout>
                <v-layout row>
                  <v-flex xs12 class="text-xs-center">
                    <v-btn @click="login" :disabled="loading" :loading="loading">
                      Sign In
                      <span slot="loader" class="custom-loader"><v-icon light>cached</v-icon></span>
                    </v-btn>
                    <v-btn @click="loginGithub">Login with Github</v-btn>
                  </v-flex>
                </v-layout>
                <v-alert v-if="!!error" :value="!!error" type="error">{{ error }}</v-alert>
                <v-alert show v-if="$auth.$state.redirect">
                  You have to login before accessing to <strong>{{ $auth.$state.redirect }}</strong>
                </v-alert>
              </form>
              <!-- <div v-for="s in strategies" :key="s.key" class="mb-2">
                <v-btn @click="$auth.loginWith(s.key)" block :style="{background: s.color}" class="login-button">Login with {{ s.name }}</v-btn>
              </div> -->
            </v-container>
            <v-container v-else>
              <form @submit.prevent="otp">
                <v-layout row>
                  <v-flex xs12>
                    <v-text-field label="Enter Your 6-digit OTP" v-model="pin" type="number" maxlength="6" required clearable></v-text-field>
                  </v-flex>
                </v-layout>
                <v-layout row>
                  <v-flex xs12 class="text-xs-center">
                    <v-btn type="submit" :disabled="loading" :loading="loading">
                      Verify OTP
                      <span slot="loader" class="custom-loader"><v-icon light>cached</v-icon></span>
                    </v-btn>
                  </v-flex>
                </v-layout>
                <v-alert v-if="!!error" :value="!!error" type="error">{{ error }}</v-alert>
                <v-alert show v-if="$auth.$state.redirect">
                  You have to login before accessing to <strong>{{ $auth.$state.redirect }}</strong>
                </v-alert>
              </form>
            </v-container>
          </v-card-text>
        </v-card>
      </v-flex>
    </v-layout>
  </v-container>
</template>

<style scoped>
.login-button {
  border: 0;
}
</style>

<script>
import busyOverlay from '../../src/BusyOverlay'
export default {
  middleware: ['auth'],
  // components: { busyOverlay },
  data() {
    return {
      otpCount: 0,
      pin: '',
      loading: false,
      disabled: false,
      email: '',
      username: '',
      password: '',
      error: null
    }
  },
  computed: {
    // strategies: () => [
    //   { key: 'auth0', name: 'Auth0', color: '#ec5425' },
    //   { key: 'google', name: 'Google', color: '#4284f4' },
    //   { key: 'facebook', name: 'Facebook', color: '#3c65c4' },
    //   { key: 'github', name: 'GitHub', color: '#202326' }
    // ],
    redirect() {
      return this.$route.query.redirect && decodeURIComponent(this.$route.query.redirect)
    },
    isCallback() {
      return Boolean(this.$route.query.callback)
    }
  },
  created() {
    // console.log(process.env.USE_OTP)
  },
  methods: {
    async loginGithub() {
      if (!process.env.GITHUB_CLIENT_ID) {
        return alert(
          'Github Client ID required in environment file...\n' +
            'Please also set GITHUB_CLIENT_ID and GITHUB_CLIENT_SECRET in backend environment file'
        )
      }
      await this.$auth.logout()
      await this.$auth.loginWith('social')
    },
    async login() {
      this.error = null
      try {
        const { data } = await this.$axios.post('/api/auth/login', {
          email: this.email, // username
          password: this.password
        })
        this.$auth.setToken('local', `Bearer ${data.token}`)
        this.$auth.strategy._setToken(`Bearer ${data.token}`) // this.$axios.defaults.headers.common['Authorization']
        console.log('sss', process.env.USE_OTP)
        if (process.env.USE_OTP) {
          this.otpCount = 3 // 3 tries
        } else {
          const rv = await this.$axios.get('/api/auth/me')
          this.$auth.setUser(rv.data)
          this.$router.push('/secure')
        }
      } catch (e) {
        this.error = e + ''
      }
    },
    async otp() {
      if (this.otpCount <= 0) return
      this.error = null
      try {
        const { data } = await this.$axios.post('/api/auth/otp', {
          pin: this.pin
        })
        this.$auth.setToken('local', `Bearer ${data.token}`)
        this.$auth.strategy._setToken(`Bearer ${data.token}`) // this.$axios.defaults.headers.common['Authorization']
        const rv = await this.$axios.get('/api/auth/me')
        this.$auth.setUser(rv.data)
        this.$router.push('/secure')
      } catch (e) {
        // console.log('otp err', this.otpCount)
        this.otpCount--
        this.error = e + ''
      }
    }
  }
}
</script>
