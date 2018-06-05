<template>
  <div class="container" id="app">
    <h3 v-if="googleData.error !== ''" class="error">{{ googleData.error }}</h3>
    <div v-if="googleData.auth.access_token">
      <h1>Hi {{ googleData.profile.ig }}</h1>
      <p><button class="btn btn-primary btn-lg" @click="logout">Logout</button></p>
      <div class="row">
        <div class="col-12">
          <div class="alert alert-primary">
            <pre>{{ googleData }}</pre>
          </div>
        </div>
        <div class="col-6">
          <ul>
            <li v-for="f in googleResponse.drive.files" v-bind:key="f.id">{{ f.name }}</li>
          </ul>
        </div>
        <div class="col-6">
          <ul>
            <li v-for="f in googleResponse.calendar.items" v-bind:key="f.id">{{ f.summary }}</li>
          </ul>
        </div>
      </div>
    </div>
    <button class="btn btn-primary btn-lg" v-if="!googleData.auth.access_token" @click="auth">Auth</button>
  </div>
</template>

<script>
import Vue from 'vue'
import GoogleAuth from '../static/modules/google-auth.js'

Vue.use(GoogleAuth, {
  client_id: 'xXXxXXXxxXX@CLIENTIDOVERHERE',
  scope: [ 'openid', 'email', 'https://www.googleapis.com/auth/calendar', 'https://www.googleapis.com/auth/drive.readonly' ].join(' ')
})
Vue.googleAuth().directAccess()
Vue.googleAuth().load()

export default {
  name: 'App',
  components: {},
  data () {
    return {
      googleData: {
        profile: {},
        auth: {},
        error: ''
      },
      googleResponse: {
        drive: {},
        calendar: {}
      }
    }
  },
  mounted () {
    if (typeof window.localStorage['googleauth'] !== 'undefined') {
      try {
        let auth = JSON.parse(window.localStorage['googleauth'])
        Object.keys(auth).forEach(k => {
          this.googleData[k] = auth[k]
        })

        this.request()
      } catch (e) {
      }
    }
  },
  methods: {
    callGoogle (uri) {
      return new Promise((resolve, reject) => {
        const headers = new Headers()
        headers.append('Authorization', `Bearer ${this.googleData.auth.access_token}`)

        window.fetch(uri, {
          method: 'GET',
          headers
        }).then(d => { return d.json() }).then(d => {
          console.log(d)

          if (typeof d.error !== 'undefined' && typeof d.error.message !== 'undefined') {
            this.googleData.error = d.error.message
            reject(new Error(d.error.message))
          } else {
            resolve(d)
          }
        }).catch(e => {
          this.googleData.error = e.message
          reject(e)
        })
      })
    },
    request () {
      let qDrive = 'https://www.googleapis.com/drive/v3/files?orderBy=viewedByMeTime&q='
      qDrive += encodeURI(`mimeType = 'application/vnd.google-apps.folder'`)

      let qCalendar = 'https://www.googleapis.com/calendar/v3/users/me/calendarList'

      this.callGoogle(qDrive).then(d => {
        this.googleResponse.drive = d
      }).catch(e => {})
      this.callGoogle(qCalendar).then(d => {
        this.googleResponse.calendar = d
      }).catch(e => {})
    },
    logout () {
      delete window.localStorage['googleauth']
      this.googleData.profile = {}
      this.googleData.auth = {}
    },
    auth () {
      Vue.googleAuth().signIn(googleUser => {
        // things to do when sign-in succeeds
        this.googleData.profile = googleUser.getBasicProfile()
        this.googleData.auth = googleUser.getAuthResponse(true)
        window.localStorage['googleauth'] = JSON.stringify(this.googleData)
        this.request()
      }, error => {
        // things to do when sign-in fails
        console.log(error)
      })
    }
  }
}
</script>

<style lang="scss">
  #app {
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
  }
  .error { color: #ca0000; }
</style>
