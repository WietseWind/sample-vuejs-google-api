<template>
  <div :class="{ 'container-fluid': googleData.auth.access_token && googleData.error === '', 'container': !googleData.auth.access_token || googleData.error !== '' }" id="app">
    <div v-if="googleData.auth.access_token && googleData.error === ''">
      <!-- All fine -->
      <div class="row">
        <div class="col-6 tl">
          <div class="card">
            <div class="card-header">
              <b>Calendar</b>
            </div>
            <div class="card-body">
              <p v-if="Object.keys(googleResponse.calendar).length < 1" class="alert text-center text-primary">Loading...</p>
              <pre>{{ googleResponse.calendar }}</pre>
            </div>
          </div>
        </div>
        <div class="col-6 tr">
          <div class="card">
            <div class="card-header">
              <b>Read Ahead</b>
              <span v-if="googleData.readAheadFolder.name" class="text-primary float-right">
                Folder: <b>{{ googleData.readAheadFolder.name }}</b>
                <a @click="clearRaFolder" class="text-danger">&times;</a>
              </span>
            </div>
            <div class="card-body" v-if="googleData.readAheadFolder.name === ''">
              <p class="alert alert-warning text-center">Please select the Read Ahead folder first.</p>
              <p v-if="Object.keys(googleData.readAheadFolder.folders).length < 1" class="alert text-center text-primary">Loading...</p>
              <div class="select text-center mt-5" v-if="Object.keys(googleData.readAheadFolder.folders).length > 0">
                <select v-model="googleData.readAheadFolder.id" class="form-control">
                  <option v-for="f in googleData.readAheadFolder.folders.files" :value="f.id" v-bind:key="f.id">{{ f.name }}</option>
                </select>
                <br />
                <button @click="saveReadAheadFolder" :disabled="googleData.readAheadFolder.id === ''" class="btn btn-primary float-right btn-md">Save</button>
              </div>
            </div>
            <div class="card-body" v-if="googleData.readAheadFolder.name !== ''">
              <p v-if="Object.keys(googleResponse.drive).length < 1" class="alert text-center text-primary">Loading...</p>
              <div class="alert alert-warning text-center" v-if="googleResponse.drive.files && googleResponse.drive.files.length < 1">
                No files in this folder.
              </div>
              <ul class="list-unstyled" v-if="googleResponse.drive.files && googleResponse.drive.files.length > 0">
                <li v-for="f in googleResponse.drive.files" v-bind:key="f.id"><a target="_blank" :href="'https://drive.google.com/open?id=' + f.id">{{ f.name }}</a></li>
              </ul>
              <!-- <pre>{{ googleResponse.drive.files }}</pre> -->
            </div>
          </div>
        </div>
        <div class="col-6 bl">
          <div class="card">
            <div class="card-header">
              <b>Search files</b>
            </div>
            <div class="card-body">
              <p v-if="Object.keys(googleResponse.search.results).length < 1 && googleResponse.search.searching" class="alert text-center text-primary">Loading...</p>
              <pre>{{ googleResponse.search.results }}</pre>
            </div>
          </div>
        </div>
        <div class="col-6 br">
          <div class="card">
            <div class="card-header">
              <b>Recently accessed</b>
            </div>
            <div class="card-body">
              <p v-if="Object.keys(googleResponse.recent).length < 1" class="alert text-center text-primary">Loading...</p>
              <pre>{{ googleResponse.recent }}</pre>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div v-if="googleData.error !== ''">
      <!-- Error -->
      <br />
      <h1>Google Dashboard</h1>
      <p>Error</p>
      <p class="alert alert-danger">
        {{ googleData.error }}
      </p>
      <button class="btn btn-primary btn-lg" @click="logout();auth();">Re-Authorize your Google account</button>
    </div>
    <div v-if="!googleData.auth.access_token">
      <!-- No access token -->
      <br />
      <h1>Google Dashboard</h1>
      <p>Please login first</p>
      <button class="btn btn-primary btn-lg" @click="auth">Authorize your Google account</button>
    </div>
  </div>
</template>

<script>
import Vue from 'vue'
import GoogleAuth from '../static/modules/google-auth.js'
import Config from '../google-config.js'

Vue.use(GoogleAuth, {
  client_id: Config.google.client_id,
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
        error: '',
        readAheadFolder: {
          name: '',
          id: '',
          folders: {}
        }
      },
      googleResponse: {
        drive: {},
        recent: {},
        search: {
          query: '',
          searching: false,
          results: {}
        },
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
    saveReadAheadFolder () {
      this.googleData.readAheadFolder.name = this.googleData.readAheadFolder.folders.files.filter(f => {
        return f.id === this.googleData.readAheadFolder.id
      })[0].name
      window.localStorage['raFolder'] = JSON.stringify({
        name: this.googleData.readAheadFolder.name,
        id: this.googleData.readAheadFolder.id
      })
      this.googleResponse.drive = {}
      this.getRaFolderContents()
    },
    clearRaFolder () {
      delete window.localStorage['raFolder']
      this.googleData.readAheadFolder.name = ''
      this.loadRaFolders()
    },
    callGoogle (uri) {
      return new Promise((resolve, reject) => {
        if (this.googleData.error === '') {
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
        } else {
          resolve(new Error('Skip, existing error.'))
        }
      })
    },
    getRaFolderContents () {
      if (this.googleData.readAheadFolder.name !== '') {
        let qDrive = 'https://www.googleapis.com/drive/v3/files?orderBy=viewedByMeTime&q=' + encodeURIComponent(`'${this.googleData.readAheadFolder.id}' in parents and mimeType != 'application/vnd.google-apps.folder'`)
        // qDrive += encodeURI(`mimeType = 'application/vnd.google-apps.folder'`)
        this.callGoogle(qDrive).then(d => {
          this.googleResponse.drive = d
        }).catch(e => {})
      }
    },
    loadRaFolders () {
      if (this.googleData.readAheadFolder.name === '') {
        let qDrive = 'https://www.googleapis.com/drive/v3/files?orderBy=name&q='
        qDrive += encodeURIComponent(`parents in 'root' and mimeType = 'application/vnd.google-apps.folder'`)
        this.callGoogle(qDrive).then(d => {
          this.googleData.readAheadFolder.folders = d
        }).catch(e => {})
      }
    },
    request () {
      if (typeof window.localStorage['raFolder'] !== 'undefined') {
        try {
          let parsedRa = JSON.parse(window.localStorage['raFolder'])
          this.googleData.readAheadFolder.name = parsedRa.name
          this.googleData.readAheadFolder.id = parsedRa.id
          this.getRaFolderContents()
        } catch (e) {
          delete window.localStorage['raFolder']
        }
      } else {
        this.loadRaFolders()
      }

      // let qDrive = 'https://www.googleapis.com/drive/v3/files?orderBy=viewedByMeTime&q='
      // qDrive += encodeURI(`mimeType = 'application/vnd.google-apps.folder'`)
      // this.callGoogle(qDrive).then(d => {
      //   this.googleResponse.drive = d
      // }).catch(e => {})

      // let qCalendar = 'https://www.googleapis.com/calendar/v3/users/me/calendarList'
      // this.callGoogle(qCalendar).then(d => {
      //   this.googleResponse.calendar = d
      // }).catch(e => {})
    },
    logout () {
      this.googleData.error = ''
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
  html { height: 100vh; }
  body { min-height: 100vh; }
  html, body { overflow: hidden; }

  #app {
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
    height: 100vh;
    min-height: 100vh;

    a { cursor: pointer; }

    &.container-fluid {
      padding: 0;
      margin: 0;
      height: 100vh;

      div.row {
        padding: 0;
        margin: 0;
        height: 100vh;

        div.col-6 {
          height: 50vh;
          padding: 5px 5px;

          div.card {
            height: 100%;
            overflow: hidden;

            div.card-body {
              border: 1px solid red;
              max-height: 100%;
              overflow: auto;
              -webkit-overflow-scrolling: auto;
            }
          }

          &.tl { padding-right: 2px; padding-bottom: 2px; }
          &.bl { padding-right: 2px; padding-top: 3px; }
          &.tr { padding-left: 3px;; padding-bottom: 2px; }
          &.br { padding-left: 3px; padding-top: 3px; }
        }
      }
    }
  }
</style>
