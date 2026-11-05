<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Crear post</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">Crear post</ion-title>
        </ion-toolbar>
      </ion-header>

      <ion-list>
        <ion-item>
          <ion-input v-model="form.title" label="Título" label-placement="stacked" required />
        </ion-item>
        <ion-item>
          <ion-textarea v-model="form.body" label="Contenido" label-placement="stacked" :auto-grow="true" required />
        </ion-item>
        <ion-item>
          <ion-select
            v-model="form.userId"
            label="Usuario"
            label-placement="stacked"
            placeholder="Selecciona un usuario"
            :disabled="loadingUsers || usuarios.length === 0"
            required
          >
            <ion-select-option v-for="usuario in usuarios" :key="usuario.id" :value="String(usuario.id)">
              {{ usuario.name }} ({{ usuario.username }})
            </ion-select-option>
          </ion-select>
        </ion-item>
      </ion-list>

      <div class="ion-padding">
        <ion-button expand="block" :disabled="saving || loadingUsers" @click="guardarPost">
          {{ saving ? 'Guardando…' : 'Guardar post' }}
        </ion-button>
        <ion-button expand="block" fill="outline" :disabled="saving" @click="escanearQr"> Escanear QR </ion-button>
      </div>

      <ion-note v-if="errorMessage" color="danger" class="ion-padding">
        {{ errorMessage }}
      </ion-note>
      <ion-note v-if="successMessage" color="success" class="ion-padding">
        {{ successMessage }}
      </ion-note>

      <ion-card v-if="postGuardado">
        <ion-card-header>
          <ion-card-title>{{ postGuardado.title }}</ion-card-title>
          <ion-card-subtitle v-if="postGuardado.id">
            ID: {{ postGuardado.id }} · Usuario: {{ postGuardado.userId }}
          </ion-card-subtitle>
        </ion-card-header>
        <ion-card-content>
          {{ postGuardado.body }}
        </ion-card-content>
      </ion-card>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  CapacitorBarcodeScanner,
  CapacitorBarcodeScannerCameraDirection,
  CapacitorBarcodeScannerScanOrientation,
  CapacitorBarcodeScannerTypeHint,
} from '@capacitor/barcode-scanner';
import type { Usuarios } from '@/interfaces/usuarios';
import {
  IonButton,
  IonCard,
  IonCardContent,
  IonCardHeader,
  IonCardSubtitle,
  IonCardTitle,
  IonContent,
  IonHeader,
  IonInput,
  IonItem,
  IonList,
  IonNote,
  IonPage,
  IonSelect,
  IonSelectOption,
  IonTextarea,
  IonTitle,
  IonToolbar,
} from '@ionic/vue';
import { onMounted, reactive, ref } from 'vue';

type PostFormulario = {
  title: string;
  body: string;
  userId: string;
};

type PostGuardado = {
  id?: number;
  title: string;
  body: string;
  userId: number;
};

const usuarios = ref<Usuarios[]>([]);
const form = reactive<PostFormulario>({
  title: '',
  body: '',
  userId: '',
});
const postGuardado = ref<PostGuardado | null>(null);
const loadingUsers = ref(false);
const saving = ref(false);
const errorMessage = ref('');
const successMessage = ref('');

async function cargarUsuario() {
  loadingUsers.value = true;
  errorMessage.value = '';

  try {
    const response = await fetch('https://jsonplaceholder.typicode.com/users');
    if (!response.ok) {
      throw new Error(`No se pudieron cargar los usuarios (${response.status}).`);
    }

    usuarios.value = (await response.json()) as Usuarios[];
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : 'Ocurrió un error al cargar los usuarios.';
  } finally {
    loadingUsers.value = false;
  }
}

async function guardarPost() {
  errorMessage.value = '';
  successMessage.value = '';
  postGuardado.value = null;

  const title = form.title.trim();
  const body = form.body.trim();
  const userId = Number(form.userId);

  if (!title || !body || !form.userId) {
    errorMessage.value = 'Completa el título, el contenido y selecciona un usuario.';
    return;
  }

  if (!Number.isSafeInteger(userId) || !usuarios.value.some((usuario) => usuario.id === userId)) {
    errorMessage.value = 'Selecciona un usuario válido de la lista.';
    return;
  }

  saving.value = true;
  try {
    const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json; charset=UTF-8' },
      body: JSON.stringify({ title, body, userId }),
    });

    if (!response.ok) {
      throw new Error(`No se pudo guardar`);
    }

    const result = (await response.json()) as Partial<PostGuardado>;
    postGuardado.value = {
      title,
      body,
      userId,
      ...(typeof result.id === 'number' ? { id: result.id } : {}),
    };
    successMessage.value = 'Datos guardados correctamente';
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : 'Ocurrió un error al guardar';
  } finally {
    saving.value = false;
  }
}

async function escanearQr() {
  errorMessage.value = '';
  successMessage.value = '';

  try {
    const result = await CapacitorBarcodeScanner.scanBarcode({
      hint: CapacitorBarcodeScannerTypeHint.QR_CODE,
      cameraDirection: CapacitorBarcodeScannerCameraDirection.BACK,
      scanOrientation: CapacitorBarcodeScannerScanOrientation.ADAPTIVE,
      scanInstructions: 'Escanea un QR',
    });
    const scannedData: unknown = JSON.parse(result.ScanResult);

    if (typeof scannedData !== 'object' || scannedData === null || Array.isArray(scannedData)) {
      throw new Error('El QR no está correcto');
    }

    const data = scannedData as Record<string, unknown>;
    const userId = Number(data.userId);
    if (
      typeof data.title !== 'string' ||
      typeof data.body !== 'string' ||
      !Number.isSafeInteger(userId) ||
      !usuarios.value.some((usuario) => usuario.id === userId)
    ) {
      throw new Error('El QR debe incluir los datos correspondientes');
    }

    form.title = data.title;
    form.body = data.body;
    form.userId = String(userId);
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : 'No se pudo leer el código QR.';
  }
}

onMounted(cargarUsuario);
</script>
