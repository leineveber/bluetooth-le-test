<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Home</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">Home</ion-title>
        </ion-toolbar>
      </ion-header>

      <div>
        <ion-button
          v-if="!isScanning"
          @click="
            BleClient.requestLEScan(
              {},
              (result) => result.localName && newDevices.push(result)
            );
            isScanning = true;
          "
          >Scan...</ion-button
        >
        <ion-button
          v-else
          @click="
            BleClient.stopLEScan();
            isScanning = false;
          "
          >Stop scanning...</ion-button
        >
      </div>

      <div>
        <p>Found devices:</p>
        <ion-button
          v-for="device in newDevices"
          :key="device.device.deviceId"
          @click="connect(device)"
        >
          <ion-spinner
            v-if="connectingTo === device.device.deviceId"
            slot="start"
          />
          {{ device.localName }}
        </ion-button>
      </div>

      <div v-if="myDevice">
        Connected to:
        <ion-button @click="disconnect(myDevice!)">{{
          myDevice.localName
        }}</ion-button>
      </div>

      <div v-if="myDevice">
        <ion-button
          :disabled="isProcessing"
          @click="startNotifications(myDevice!)"
          >START NOTIFICATIONS</ion-button
        >
        <ion-button
          :disabled="isProcessing"
          @click="stopNotifications(myDevice!)"
          >STOP NOTIFICATIONS</ion-button
        >
        <ion-button :disabled="isProcessing" @click="getVersion(myDevice!)"
          >GET VERSION</ion-button
        >

        <p>buffer: {{ buffer }}</p>
      </div>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  IonContent,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
  IonButton,
  IonSpinner,
} from "@ionic/vue";
import {
  ScanResult,
  BleClient,
  dataViewToText,
  textToDataView,
} from "@capacitor-community/bluetooth-le";
import { ref, onMounted } from "vue";
import { alertController } from "@ionic/core";

const SERVICE = "331a36f5-2459-45ea-9d95-6142f0c4b307";
const TX_CHAR = "a73e9a10-628f-4494-a099-12efaf72258f";
const RX_CHAR = "a9da6040-0823-4995-94ec-9ce41ca28833";

const myDevice = ref<ScanResult | null>(null);
const newDevices = ref<ScanResult[]>([]);
const isScanning = ref(false);
const connectingTo = ref("");
const buffer = ref("");
const isProcessing = ref(false);

const connect = (device: ScanResult) => {
  connectingTo.value = device.device.deviceId;
  BleClient.connect(device.device.deviceId)
    .then(async () => {
      myDevice.value = device;
      connectingTo.value = "";
      BleClient.stopLEScan();
      isScanning.value = false;
      newDevices.value = newDevices.value.filter(
        (newDevice) => newDevice.device.deviceId !== device.device.deviceId
      );

      const isBonded = await BleClient.isBonded(device.device.deviceId);

      if (!isBonded) {
        await BleClient.createBond(device.device.deviceId);
      }
    })
    .catch((error) => {
      connectingTo.value = "";
      alertController
        .create({ message: error.message, buttons: ["Ok"] })
        .then((a) => a.present());
    });
};

const disconnect = (device: ScanResult) => {
  BleClient.disconnect(device.device.deviceId).then(() => {
    myDevice.value = null;
    newDevices.value = newDevices.value.concat(device);
  });
};

const startNotifications = (device: ScanResult) => {
  isProcessing.value = true;
  BleClient.startNotifications(
    device.device.deviceId,
    SERVICE,
    TX_CHAR,
    (result) => {
      console.log(result);
      return (buffer.value = buffer.value.concat(dataViewToText(result)));
    }
  )
    .then(() => {
      console.log("STARTED NOTIFICATIONS");
    })
    .catch(() =>
      alertController
        .create({
          message: "Failed to start notifications",
          buttons: ["Ok"],
        })
        .then((a) => a.present())
    )
    .finally(() => {
      isProcessing.value = false;
    });
};

const stopNotifications = (device: ScanResult) => {
  isProcessing.value = true;
  BleClient.stopNotifications(device.device.deviceId, SERVICE, TX_CHAR).finally(
    () => (isProcessing.value = false)
  );
};

const getVersion = (device: ScanResult) => {
  isProcessing.value = true;
  BleClient.write(
    device.device.deviceId,
    SERVICE,
    RX_CHAR,
    textToDataView('{"cmd":1}')
  ).finally(() => (isProcessing.value = false));
};

onMounted(() => {
  BleClient.initialize();
});
</script>

<style scoped></style>
