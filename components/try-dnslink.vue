<template>
  <div class="container">
    <input
      v-model="domain"
      placeholder="eg. dnslink.dev"
      @keydown="onKeyDown"
    >
    <details v-if="doh && endpoints[doh].docs" class="provider">
      <summary>
        <label for="doh">Provider:</label> <select name="doh" v-model="doh">
          <option
            v-for="endpoint in endpointNames"
            :key="endpoint"
            :value="endpoint.name"
          >{{ endpoint.name }}</option>
        </select>
      </summary>

      <div class="provider-content">
        To resolve the DNS TXT entries this site, and many libraries use: <a href="https://en.wikipedia.org/wiki/DNS_over_HTTPS" target="_blank">DNS over HTTPS</a> (DoH).
        There are many, publically offered DoH providers, we use this select list of providers we know to work in the browser.

        <dl>
          <dt>Information about "{{ doh }}":</dt>
          <dd><a :href="endpoints[doh].docs" target="_blank">{{ endpoints[doh].docs }}</a></dd>
          <dt>URL for the lookup:</dt>
          <dd><a :href="dohLocation" target="_blank">{{ dohLocation }}</a></dd>
          <dt v-if="endpoints[doh].location">Location</dt>
          <dd v-if="endpoints[doh].location">{{ endpoints[doh].location }}</dd>
        </dl>
      </div>
    </details>
    <div v-if="running">...running</div>
    <pre v-if="error">{{ error.message }}</pre>
    <div v-if="result">
      <ul>
        <!-- eslint-disable-next-line vue/require-v-for-key -->
        <li v-for="(value, key) in result.links">
          {{ key }} → {{ value }}
        </li>
      </ul>
      <div v-if="result.path.length > 0">
        <h5>Deep Linking</h5>
        <ul>
          <!-- eslint-disable-next-line vue/require-v-for-key -->
          <li v-for="path in result.path">
            {{ pathToString(path) }}
          </li>
        </ul>
      </div>
      <details class="log">
        <summary>Log</summary>
        <table >
          <tbody>
            <tr v-for="entry in result.log">
              <td><LogCode :code="entry.code" :reason="entry.reason" /></td>
              <td>{{ entry.domain }}{{ pathToString(entry) }}{{ entry.entry }}</td>
            </tr>
          </tbody>
        </table>
      </details>
    </div>
  </div>
</template>
<style lang="scss" scoped>
input {
  font-size: 1.2em;
  margin-bottom: 0.5em;
  width: 60%;
}
.container {
  padding: 1em;
  background: #f0f0f0;
}
.container {
  color: black;
}
.provider, .log {
  font-size: 0.9em;
}
.provider {
  padding-bottom: 0.5em;
  border-bottom: 1px solid #b0b0b0;
}
.provider-content {
  margin-top: 0.5em;
}
table {
  width: 100%;
}
</style>
<script lang="ts" setup>
import { ref, watch, computed } from 'vue'
import { resolveN, Result, PathEntry } from '@dnslink/js'
import { endpoints }  from 'dns-query'
import LogCode from './log-code.vue'

const result = ref<Result>(null)
const error = ref<Error>(null)
const running = ref(false)
const dohLocation = computed(() => {
  const endpoint = endpoints[doh.value]
  if (!endpoint) return ''
  return `https://${endpoint.host}${endpoint.port ? `:${endpoint.port}` : '' }${ endpoint.path || '/dns-query' }`
})
// eslint-disable-next-line no-unused-vars
const endpointNames = Object
  .entries(endpoints)
  .filter(([name, endpoint]) => endpoint.cors || name === 'google')
  .map(([name, endpoint]) => ({ name, endpoint }))
  .sort((a, b) => {
    if (a.name < b.name) return -1
    if (a.name > b.name) return 1
    return 0
  })
const domain = ref()
const doh = ref('cloudflare')
const reactAfter = 200
const restartAfter = 1000
let controller
let timer

function onKeyDown (e) {
  if (e.code === "Enter") {
    restart()
  }
}

function start () {
  if (!domain.value) {
    return
  }
  const mine = new AbortController()
  controller = mine
  error.value = null
  running.value = true
  resolveN(domain.value, {
    signal: controller.signal,
    doh: doh.value
  }).then(
    next(_result => { result.value = _result }),
    next(_error => { error.value = _error })
  )

  function next (op) {
    return function () {
      if (controller === mine) {
        op.apply(null, arguments)
        controller = undefined
        running.value = false
      }
    }
  }
}

function pathToString (path: PathEntry): string {
  let result = path.pathname !== undefined ? path.pathname : ''
  let sep = '?'
  if (path.search) {
    for (const [key, values] of Object.entries(path.search)) {
      for (const value of values) {
        result += sep + encodeURIComponent(key) + '=' + encodeURIComponent(value)
        sep = '&'
      }
    }
  }
  return result
}

function restart () {
  if (timer !== undefined) {
    clearTimeout(timer)
    timer = undefined
  }
  if (controller) {
    controller.abort()
  }
  start()
}

watch([domain, doh], () => {
  if (controller === undefined) {
    if (timer !== undefined) {
      clearTimeout(timer)
    }
    timer = setTimeout(() => {
      start() 
      timer = undefined
    }, reactAfter)
  } else {
    if (timer === undefined) {
      timer = setTimeout(() => {
        restart()
        timer = undefined
      }, restartAfter)
    }
  }
})
</script>
