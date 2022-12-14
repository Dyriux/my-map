<template>
  <div id="map-container">
    <client-only>
      <div v-if="rdvError === true" class="overlay">Sélectionner le lieu de RDV</div>
      <div v-if="rdvSet === true" class="rdvInfo">
        <p>Le rdv est à <input id="heureDuRdv" type="time" v-model="heureRdv" @change="calcTempsTrajet" />.&nbsp;</p>
        <p>Il faut partir à {{ heureDepart }}</p>
      </div>

      <l-map :zoom="16" :center="userPosition" @click="ajoutRdv">
        <l-tile-layer url="http://{s}.tile.osm.org/{z}/{x}/{y}.png"></l-tile-layer>
        <l-marker :lat-lng="userPosition" :icon="iconPerson"></l-marker>
        <l-marker v-if="rdv" :lat-lng="rdv" :icon="iconRdv"></l-marker>
        <l-itinerary v-if="itinerary" :lat-lngs="itinerary"></l-itinerary>
        <l-marker
          v-for="(resto, index) in restaurants"
          :key="index"
          :lat-lng="resto.loc"
          :icon="iconResto"
          @click="stepResto"></l-marker>
      </l-map>
    </client-only>
  </div>
</template>

<script>
/******************************* IMPORTS *******************************/
import { icon } from 'leaflet'
import socket from '@/services/socket-client.js'
import moment from 'moment'

/******************************* VALEURS PAR DEFAUT *******************************/
export default {
  data() {
    return {
      userPosition: [48.83374, 2.24352],
      iconPerson: icon({
        iconUrl: '/images/green-marker.png',
        iconSize: [30, 35],
        iconAnchor: [15, 35]
      }),
      iconRdv: icon({
        iconUrl: '/images/red-marker.png',
        iconSize: [30, 35],
        iconAnchor: [15, 35]
      }),
      iconResto: icon({
        iconUrl: '/images/blue-marker.png',
        iconSize: [30, 35],
        iconAnchor: [15, 35]
      }),
      rdv: null,
      heureRdv: null,
      itinerary: [],
      distance: 0,
      temps: 0,
      heureDepart: null,
      restoSelec: null,
      currentRoom: null,
      rdvError: false,
      rdvSet: false
    }
  },
  props: {
    restaurants: [],
    restoMap: null,
    user: null,
    rooms: null
  },

/******************************* FONCTIONS *******************************/
  mounted() {
    this.getUserLocation()
  },
  watch: {
    restoMap: function (restoMap) {
      this.stepResto(restoMap)
    },
    userPosition() {
      this.onDataChange()
    },
    rdv() {
      this.onDataChange()
    },
    restoSelec() {
      this.onDataChange()
    },
    rooms(newValue, oldValue) {
      this.currentRoom = newValue?.find(room => room.users.find(user => user.id === this.user.id))
    }
  },

  methods: {
    onDataChange() {
      // Renvoie les nouvelles données lors d'un changement
      socket.emit('user-map-data', {
        user: this.user,
        userPosition: this.userPosition,
        rdv: this.rdv,
        restoSelec: this.restoSelec
      })
    },
    getUserLocation() {
      // Récupère les coordonnées de l'utilisateur
      navigator.geolocation.getCurrentPosition((position) => {
        this.userPosition = [position.coords.latitude, position.coords.longitude]
      })
    },
    ajoutRdv(event) {
      // Création du RDV
      this.rdv = [event.latlng.lat, event.latlng.lng]
      this.itinerary = []
      this.itinerary.push(this.userPosition)
      this.itinerary.push(this.rdv)
      this.getDistance(this.itinerary[0], this.itinerary[1])
      this.rdvError = false
      this.rdvSet = false
    },

    stepResto(event) {
      // Tracage de l'itinéraire de l'utilisateur
      if (this.rdv) {
        let t1 = 0
        let t2 = 0
        this.restoSelec = this.restoMap
        if (event.latlng) {
          this.itinerary = []
          this.itinerary.push(this.userPosition)
          this.itinerary.push(this.rdv)
          this.itinerary.splice(1, 0, [event.latlng.lat, event.latlng.lng])
        } else {
          this.itinerary = []
          this.itinerary.push(this.userPosition)
          this.itinerary.push(this.rdv)
          this.itinerary.splice(1, 0, [this.restoSelec.loc[0], this.restoSelec.loc[1]])
        }
        this.getDistance(this.itinerary[0], this.itinerary[1])
        t1 = this.temps
        this.getDistance(this.itinerary[1], this.itinerary[2])
        t2 = this.temps
        this.temps = t1 + t2
        this.rdvSet = true
        if (this.heureRdv) {
          this.calcTempsTrajet()
        }
      } else {
        this.rdvError = true
      }
    },

    getDistance(pos1, pos2) {
      // Calcul de distance
      let vitesse = 5
      let lat1 = pos1[0] * (Math.PI / 180)
      let lat2 = pos2[0] * (Math.PI / 180)
      let long1 = pos1[1] * (Math.PI / 180)
      let long2 = pos2[1] * (Math.PI / 180)

      // ACOS(SIN(RADIANS(A2))*SIN(RADIANS(A3))+COS(RADIANS(A2))*COS(RADIANS(A3))*COS(RADIANS(B2-B3)))*6371
      this.distance =
        Math.acos(Math.sin(lat1) * Math.sin(lat2) + Math.cos(lat1) * Math.cos(lat2) * Math.cos(long2 - long1)) * 6371
      this.temps = Math.trunc((this.distance / vitesse) * 60) + 1
    },

    formatTime(time) {
      // On formate le temps en hh:mm
      time = moment.utc(moment.duration(this.heureDepart, 'minutes').asMilliseconds())
      const minutes = time.minutes()
      const hours = time.hours()
      const hourFormatStr = hours === 1 ? 'h' : 'h'
      const minuteFormatStr = minutes === 1 ? 'm' : 'm'
      if (!time.minutes()) {
        this.heureDepart = time.format(`HH [${hourFormatStr}]`)
      }
      this.heureDepart = time.format(`HH [${hourFormatStr}] mm [${minuteFormatStr}]`)
    },

    calcTempsTrajet(heureRdv) {
      // Calcul du temps de trajet
      this.heureDepart = moment.utc(moment.duration(this.temps, 'minutes').asMilliseconds()).format('HH:mm')
      this.heureDepart = moment.duration(this.heureRdv).asMinutes() - moment.duration(this.heureDepart).asMinutes()
      this.formatTime(this.heureDepart)
    }
  }
}
</script>


<!---------- On met en forme la page ---------->

<style scoped>
#map-container {
  height: 100vh
}

.overlay {
  position: absolute;
  padding: 1rem 2rem;
  z-index: 5;
  background-color: #CE2525;
  width: 50%;
  left: 50%;
  transform: translate(-50%);
  margin-top: 1rem;
  display: flex;
  justify-content: center;
  font-size: 2rem;
  border-radius: 10px;
}

.rdvInfo {
  position: absolute;
  padding: 1rem 2rem;
  z-index: 6;
  background-color: #25A1CE;
  width: 50%;
  left: 50%;
  transform: translate(-50%);
  margin-top: 1rem;
  display: flex;
  justify-content: center;
  font-size: 2rem;
  border-radius: 10px;
}
</style>
