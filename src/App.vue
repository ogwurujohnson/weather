<template>
  <div id="app">
    <header>
      <span>Ably Geolocation PWA</span>
    </header>
    <main>
      <Loading :active.sync="fetchingWeather" 
        :is-full-page="true" />
      <img :src="iconUrl" alt="Vue.js PWA">
      <div class="details">
        <h1>{{temp}}<sup>o<span>C</span></sup></h1>
        <h3>{{location}}</h3>
        <h3>{{temp_description}}</h3>
      </div>
    </main>
  </div>
</template>

<script>
import axios from 'axios'
import * as Ably from 'ably'
import Loading from 'vue-loading-overlay'
import 'vue-loading-overlay/dist/vue-loading.css'

var ably = new Ably.Realtime('rc6yQA.s3Vueg:Ratu2ce1NI5Fvk1o')
var channel = ably.channels.get('test')
export default {
  name: 'app',
  data () {
    return {
      gettingLocation: false,
      iconUrl: 'https://openweathermap.org/img/wn/10d@2x.png',
      fetchingWeather: false,
      errorStr: '',
      temp: '00',
      temp_description: 'fetching...',
      location: 'unknown'
    }
  },
  components: {
    Loading
  },
  beforeDestroy () {
    clearInterval(this.polling)
  },
  async created () {
    let iconUrl
    let tempKelvin
    let tempDescription
    await channel.subscribe('update', function (message) {
      const icon = message.data.weather[0].icon
      tempKelvin = message.data.main.temp
      tempDescription = message.data.weather[0].main
      iconUrl = `https://openweathermap.org/img/wn/${icon}@2x.png`
      console.log(message.data)
    })
    this.fetchData()
    this.polling = setInterval(() => {
      this.fetchData()
      this.iconUrl = iconUrl
      this.temp = tempKelvin - 273.15
      this.temp_description = tempDescription
    }, 30000)
  },
  methods: {
    fetchData () {
      if (!('geolocation' in navigator)) {
        this.errorStr = 'Geolocation is not available.'
        return
      }
      this.gettingLocation = true
      navigator.geolocation.getCurrentPosition(pos => {
        this.gettingLocation = false
        this.getWeather(pos.coords.longitude, pos.coords.latitude)
      }, err => {
        this.gettingLocation = false
        this.errorStr = err.message
      })
    },

    updateInfo (url) {
      this.iconUrl = url
    },

    getWeather (lon, lat) {
      this.fetchingWeather = true
      axios.post(`https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=cbe869d50efffec20f3a18fe4e759bfb`)
        .then((res) => {
          const icon = res.data.weather[0].icon
          const city = res.data.name
          const country = res.data.sys.country
          const tempDescription = res.data.weather[0].main
          const tempKelvin = res.data.main.temp
          this.fetchingWeather = false
          this.iconUrl = `https://openweathermap.org/img/wn/${icon}@2x.png`
          this.temp = tempKelvin - 273.15
          this.temp_description = tempDescription
          this.location = `${city}, ${country}`
          this.sendToAbly(res.data)
        })
    },

    sendToAbly (data) {
      channel.publish('update', data)
    }
  },
  watch: {
    getUrl () {
      console.log(this.iconUrl)
    }
  }
}
</script>

<style>
body {
  margin: 0;
}

#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  background: #becbd8;
  height: 100vh;
}

main {
  text-align: center;
  margin-top: 40px;
  display: flex;
  flex-direction: column-reverse;
  justify-content: center;
  align-items: center;
  box-shadow: 0 19px 38px rgba(0,0,0,.3), 0 15px 12px rgba(0,0,0,.22);
  width: 25%;
  margin: 20px auto;
  height: 75%;
}
h1 {
  line-height: 0;
  font-size: 7em;
}

sup {
  font-size: 20px;
}

span {
  font-size: 20px;
}

img {
  width: 150px;
  vertical-align: top;
}

.details {
  color: #35495E;
  font-weight: bold;
  text-align: center;
}

header {
  margin: 0;
  height: 56px;
  padding: 0 16px 0 24px;
  background-color: #35495E;
  color: #ffffff;
}

header span {
  display: block;
  position: relative;
  font-size: 20px;
  line-height: 1;
  letter-spacing: .02em;
  font-weight: 400;
  box-sizing: border-box;
  padding-top: 16px;
}

@media screen and (max-width: 450px) {
  main {
    box-shadow: none;
  }
}
</style>
