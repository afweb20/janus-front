<template>
  <h1>You did it!</h1>
  <p>
    Visit <a href="https://vuejs.org/" target="_blank" rel="noopener">vuejs.org</a> to read the
    documentation
  </p>

  <div class="p-4 space-y-4">
    <!-- Выбор устройств -->
    <div class="space-y-2 rounded-lg p-3" style="background:#f5f7fb;">
      <div class="flex flex-col md:flex-row gap-3 items-start md:items-end">
        <div class="flex-1">
          <label class="block text-sm mb-1">Микрофон</label>
          <select v-model="selectedMicId" class="w-full border rounded px-2 py-1">
            <option v-if="mics.length===0" disabled>Нет микрофонов</option>
            <option v-for="(d, i) in mics" :key="d.deviceId || i" :value="d.deviceId">
              {{ d.label || `Микрофон #${i+1}` }}
            </option>
          </select>
        </div>

        <div class="flex-1">
          <label class="block text-sm mb-1">Камера</label>
          <select v-model="selectedCamId" class="w-full border rounded px-2 py-1">
            <!-- <option value="__USER__">Фронтальная (facingMode)</option>
            <option value="__ENV__">Задняя (facingMode)</option>
            <option v-if="cams.length" disabled>──────────</option> -->
            <option v-for="(d, i) in cams" :key="d.deviceId || i" :value="d.deviceId">
              {{ d.label || `Камера #${i+1}` }}
            </option>
          </select>
        </div>

        <div class="flex gap-2">
          <button @click="refreshDevices" class="border rounded px-3 py-1">Обновить</button>
          <button @click="applyDevices" class="border rounded px-3 py-1" :disabled="!connected">Применить</button>
        </div>
      </div>
      <div v-if="deviceHint" class="text-xs text-gray-500">
        {{ deviceHint }}
      </div>
    </div>

    <div class="flex gap-3">
      <button @click="connect" :disabled="connected">Подключиться</button>
      <button @click="leave" :disabled="!connected">Выйти</button>
    </div>

    <div class="grid gap-4 md:grid-cols-2">
      <div>
        <div>Моё видео</div>
        <video ref="localVideo" autoplay playsinline muted style="width:100%;border-radius:12px;"></video>
      </div>
      <div>
        <div>Удалённое видео</div>
        <video ref="remoteVideo" autoplay playsinline style="width:100%;border-radius:12px;"></video>
        <audio ref="remoteAudio" autoplay playsinline></audio>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref, reactive } from "vue";
// Подключаем adapter как сайд-эффект
import adapter from 'webrtc-adapter';

let Janus;

// === Настройки ===
const JANUS_URL = import.meta.env.VITE_JANUS_URL ?? "wss://janus.tulister.com/";
const ICE_SERVERS = [
  { urls: "stun:stun.l.google.com:19302" },
  { urls: ["turn:janus.tulister.com:3478?transport=udp","turn:janus.tulister.com:3478?transport=tcp"], username: "janus", credential: "supersecret" }
];
const ROOM_ID = 1234;

// === Refs/State ===
const localVideo = ref(null);
const remoteVideo = ref(null);
const remoteAudio = ref(null);

const state = reactive({
  janus: null,
  plugin: null,
  sub: null,
  myId: null,
  myPrivateId: null,
});
const connected = ref(false);

// Списки устройств и выбранные id
const mics = ref([]);
const cams = ref([]);
const selectedMicId = ref("");
const selectedCamId = ref("__USER__"); // по умолчанию фронталка через facingMode
const deviceHint = ref("Подсказка: для отображения названий устройств дайте доступ к камере/микрофону.");

// === Utils ===
const isSafari = () =>
  !!Janus?.webRTCAdapter && Janus.webRTCAdapter.browserDetails?.browser === "safari";

// Загрузка устройств
async function ensureDeviceAccessForLabels() {
  try {
    const tmp = await navigator.mediaDevices.getUserMedia({ audio: true, video: true });
    tmp.getTracks().forEach(t => t.stop());
    deviceHint.value = "";
  } catch (e) {
    // Без разрешения labels будут пустыми — это ок
    deviceHint.value = "Названия устройств скрыты — браузер не дал доступ. Это не мешает выбору.";
  }
}

async function refreshDevices() {
  try {
    const list = await navigator.mediaDevices.enumerateDevices();
    mics.value = list.filter(d => d.kind === "audioinput");
    cams.value = list.filter(d => d.kind === "videoinput");

    // LOG
    console.log("LOG list", list);
    console.log("LOG cams", cams);

    if (!selectedMicId.value && mics.value[0]) selectedMicId.value = mics.value[0].deviceId;
    // selectedCamId по умолчанию оставляем __USER__
  } catch (e) {
    console.error("enumerateDevices error:", e);
  }
}

// === Janus connect ===
const connect = async () => {
  if (connected.value) return;

  // динамический импорт janus-gateway
  const mod = await import('janus-gateway');
  Janus = mod.default || mod.Janus || window.Janus;
  if (!Janus) {
    console.error('Janus library failed to load:', { mod, windowJanus: window.Janus });
    alert('Не удалось загрузить Janus библиотеку');
    return;
  }

  console.log('JANUS_URL:', JANUS_URL, 'ROOM_ID:', ROOM_ID, 'origin:', location.origin);

  Janus.init({
    debug: true,
    dependencies: Janus.useDefaultDependencies({adapter}),
    callback: () => {
      state.janus = new Janus({
        server: JANUS_URL,
        iceServers: ICE_SERVERS,
        success: attachPublisher,
        error: (err) => {
          console.error("Janus error:", err);
          alert("Не удалось подключиться к Janus");
        },
        destroyed: () => {
          connected.value = false;
        }
      });
    }
  });
};

function attachPublisher() {
  state.janus.attach({
    plugin: "janus.plugin.videoroom",
    success: (pluginHandle) => {
      state.plugin = pluginHandle;
      joinRoomAsPublisher();
    },
    error: console.error,
    onmessage: (msg, jsep) => {
      const event = msg["videoroom"];
      if (event === "joined") {
        state.myId = msg.id;
        state.myPrivateId = msg.private_id;
        connected.value = true;

        if (msg.publishers?.length) subscribeToPublishers(msg.publishers);
        publishMyMedia(); // используем выбранные устройства
      } else if (event === "event") {
        if (msg.publishers?.length) subscribeToPublishers(msg.publishers);
      }

      if (jsep) state.plugin.handleRemoteJsep({ jsep });
    },
    onlocaltrack: (track, on /*, mid*/) => {
      if (!on) return;
      const stream = new MediaStream([track]);
      if (track.kind === "video" && localVideo.value) {
        localVideo.value.muted = true;
        Janus.attachMediaStream(localVideo.value, stream);
      }
    }
  });
}

function joinRoomAsPublisher() {
  state.plugin.send({
    message: { request: "join", ptype: "publisher", room: ROOM_ID, display: "vue-user" }
  });
}

function currentAudioCapture() {
  return selectedMicId.value
    ? { deviceId: { exact: selectedMicId.value } }
    : true;
}

function currentVideoCapture() {
  if (selectedCamId.value === "__ENV__") {
    return { facingMode: { exact: "environment" } };
  }
  if (selectedCamId.value === "__USER__") {
    return { facingMode: { exact: "user" } };
  }
  return selectedCamId.value
    ? { deviceId: { exact: selectedCamId.value } }
    : true;
}

function publishMyMedia() {
  const tracks = [
    { type: "audio", capture: currentAudioCapture(), recv: false },
    {
      type: "video",
      capture: currentVideoCapture(),
      recv: false,
      simulcast: isSafari() ? false : true
    }
  ];

  state.plugin.createOffer({
    iceRestart: true,
    tracks,
    ...(isSafari() ? {} : { video: "lowres" }),
    success: (jsep) => {
      state.plugin.send({ message: { request: "configure", audio: true, video: true }, jsep });
    },
    error: (err) => {
      console.error("createOffer error:", err);
      alert("Доступ к камере/микрофону не получен или ошибка WebRTC");
    }
  });
}

// Применить выбранные устройства во время звонка (переоффер)
function applyDevices() {
  if (!connected.value || !state.plugin) return;
  publishMyMedia();
}

// Подписка на других участников
function subscribeToPublishers(pubs) {
  if (!state.sub) {
    state.janus.attach({
      plugin: "janus.plugin.videoroom",
      success: (ph) => {
        state.sub = ph;
        doSubscribe(pubs);
      },
      error: console.error,
      onmessage: (msg, jsep) => {
        if (jsep) {
          state.sub.createAnswer({
            jsep,
            tracks: [{ type: "data" }],
            success: (ans) => {
              state.sub.send({ message: { request: "start", room: ROOM_ID }, jsep: ans });
            },
            error: console.error
          });
        }
      },
      onremotetrack: (track, mid, on /*, metadata*/) => {
        if (!on) return;
        const s = new MediaStream([track]);
        if (track.kind === "video" && remoteVideo.value) {
          Janus.attachMediaStream(remoteVideo.value, s);
        }
        if (track.kind === "audio" && remoteAudio.value) {
          Janus.attachMediaStream(remoteAudio.value, s);
        }
      }
    });
  } else {
    doSubscribe(pubs);
  }
}

function doSubscribe(pubs) {
  const subscribe = [];
  pubs.forEach((p) => (p.streams || []).forEach((st) => subscribe.push({ feed: p.id, mid: st.mid })));

  const body = state.sub.webrtcUp
    ? { request: "update", subscribe }
    : { request: "join", ptype: "subscriber", room: ROOM_ID, streams: subscribe, private_id: state.myPrivateId };

  state.sub.send({ message: body });
}

const leave = () => {
  try { state.plugin?.hangup?.(); } catch {}
  try { state.sub?.hangup?.(); } catch {}
  try { state.janus?.destroy?.(); } catch {}
  connected.value = false;
};

// Инициализация списков устройств
onMounted(async () => {
  // Слушаем изменения устройств (подключение USB-гарнитуры и т.п.)
  if (navigator.mediaDevices?.addEventListener) {
    navigator.mediaDevices.addEventListener('devicechange', refreshDevices);
  }
  await ensureDeviceAccessForLabels();
  await refreshDevices();
});

onBeforeUnmount(() => {
  if (navigator.mediaDevices?.removeEventListener) {
    navigator.mediaDevices.removeEventListener('devicechange', refreshDevices);
  }
  leave();
});
</script>

<style lang="scss" scoped>
@use "@/resources/styles/participants-grid";
@use "@/resources/styles/participants-local-grid";
</style>
