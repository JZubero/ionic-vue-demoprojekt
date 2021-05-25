<template>
  <ion-page>
    <ion-header>
      <ion-toolbar color="primary">
        <ion-title>My Schwammerl</ion-title>
      </ion-toolbar>
    </ion-header>
    
    <ion-content class="ion-padding-top">
      <p v-show="noEntries" class="ion-margin-start">Jetzt ersten Schwammerl fotografieren!</p>

      <ion-item v-show="!noEntries" v-for="entry of entries" :key="entry.id" @click="goToMaps(entry)">
        <ion-thumbnail slot="start">
          <img :src="entry.imageSrc" />
        </ion-thumbnail>
        <ion-label>
          <h2>{{ entry.title }}</h2>
          <p>Aufgenommen am {{ entry.timestamp }}</p>
        </ion-label>
      </ion-item>

      <ion-fab vertical="bottom" horizontal="end" slot="fixed">
        <ion-fab-button @click="addEntry" color="success">
          <ion-icon :icon="add"></ion-icon>
        </ion-fab-button>
      </ion-fab>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonItem, IonLabel, IonThumbnail, IonFab, IonFabButton, IonIcon, alertController } from '@ionic/vue';
import { add } from 'ionicons/icons';
import { defineComponent, ref, computed } from 'vue';
import { Camera, CameraResultType } from '@capacitor/camera';
import { Geolocation } from '@capacitor/geolocation';
import { Storage } from '@capacitor/storage';
import { Browser } from '@capacitor/browser';

interface MyEntry {
  id: number;
  title: string;
  coords: {
    lat: number;
    lng: number;
  };
  imageSrc: string;
  timestamp: string;
}

const STORAGE_KEY = 'my-entries';

export default defineComponent({
  name: 'Home',
  components: {
    IonContent,
    IonHeader,
    IonPage,
    IonTitle,
    IonToolbar,
    IonThumbnail,
    IonItem,
    IonLabel,
    IonFab,
    IonFabButton,
    IonIcon
  },
  setup() {
    const entries = ref<MyEntry[]>([]);
    const noEntries = computed(() => {
      return entries.value.length === 0;
    });

    // Fetch entries from local storage using destructuring in returned Promise
    Storage.get({
      key: STORAGE_KEY
    }).then(({ value }) => {
      if (value) {
        // We need to use the spread operator (...) to convert array result of JSON.parse() to comma-separated list of single items to comply with Array.push() syntax
        entries.value.push(...JSON.parse(value) as MyEntry[]);
      }
    });

    async function addEntry() {
      // Take photo with camera and use "DataUrl" (<-- Base64 with Image Type Prefix)
      const image = await Camera.getPhoto({
        quality: 90,
        allowEditing: true,
        resultType: CameraResultType.DataUrl
      });

      // Grab current geolocation and use location.coords.(latitude|longitude)
      const location = await Geolocation.getCurrentPosition({
        enableHighAccuracy: true
      });

      // Create an alert for entering a title
      const alert = await alertController.create({
        header: 'Name your Schwammerl',
        message: 'Welchen Schwammerl hast du gefunden?',
        inputs: [
          {
            type: 'text',
            name: 'title'
          }
        ],
        buttons: [
          {
            text: 'Anlegen',
            handler({ title }) {
              // Add to beginning of entries collection
              entries.value.unshift({
                id: Date.now(),
                title,
                coords: {
                  lat: location.coords.latitude,
                  lng: location.coords.longitude
                },
                imageSrc: String(image.dataUrl),
                timestamp: new Date().toLocaleString()
              });

              // Persist whole entries collection using JSON.stringify() since Storage.set() only accepts string values
              Storage.set({
                key: STORAGE_KEY,
                value: JSON.stringify(entries.value)
              });
            }
          }
        ]
      });

      // Show the alert
      await alert.present();
    }

    function goToMaps(entry: MyEntry) {
      const url = `https://maps.google.com/?q=${entry.coords.lat},${entry.coords.lng}`;

      Browser.open({ url });
    }

    return { entries, add, addEntry, goToMaps, noEntries };
  }
});
</script>