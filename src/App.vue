<template>
  <h1>You did it!</h1>
  <p>
    Visit <a href="https://vuejs.org/" target="_blank" rel="noopener">vuejs.org</a> to read the
    documentation
  </p>

  <div class="p-4 space-y-4">
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
// webrtc-adapter как сайд-эффект (без default экспорта)
import adapter from 'webrtc-adapter';

const gridItems = Array.from({ length: 10 }, (_, i) => i + 1)

// Janus поднимем динамически (устойчиво к разным типам экспорта)
let Janus;

// 1) URL до Janus и ICE
const JANUS_URL = import.meta.env.VITE_JANUS_URL ?? "wss://janus.tulister.com/"; // если проксируешь корень
// если у тебя наружу /janus, то используй:
// const JANUS_URL = import.meta.env.VITE_JANUS_URL ?? "wss://janus.tulister.com/janus";

const ICE_SERVERS = [
  { urls: "stun:stun.l.google.com:19302" },
  {
    urls: [
      "turn:janus.tulister.com:3478?transport=udp",
      "turn:janus.tulister.com:3478?transport=tcp"
    ],
    username: "janus",
    credential: "supersecret"
  }
];

// 2) Константы видеокомнаты
const ROOM_ID = 1234;

// Refs для <video>/<audio>
const localVideo = ref(null);
const remoteVideo = ref(null);
const remoteAudio = ref(null);

// Состояния/хендлы
const state = reactive({
  janus: null,
  plugin: null,        // publisher
  sub: null,           // subscriber (мультистрим)
  myId: null,
  myPrivateId: null,
});
const connected = ref(false);

// Safari?
const isSafari = () =>
  !!Janus?.webRTCAdapter && Janus.webRTCAdapter.browserDetails?.browser === "safari";

// Подключение
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
    // adapter уже подключён сайд-эффектом; дефолтные зависимости достаточно
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
        publishMyMedia();
      } else if (event === "event") {
        if (msg.publishers?.length) subscribeToPublishers(msg.publishers);
        if (msg.leaving || msg.unpublished) {
          // подчистка UI при уходе/отписке
        }
      }

      if (jsep) {
        state.plugin.handleRemoteJsep({ jsep });
      }
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
  const body = {
    request: "join",
    ptype: "publisher",
    room: ROOM_ID,
    display: "vue-user"
  };
  state.plugin.send({ message: body });
}

function publishMyMedia() {
  const videoConstraints = true; // или { facingMode: { exact: "user" | "environment" } }

  const tracks = [
    { type: "audio", capture: true, recv: false },
    {
      type: "video",
      capture: videoConstraints,
      recv: false,
      simulcast: isSafari() ? false : true
    }
  ];

  state.plugin.createOffer({
    iceRestart: true,
    tracks,
    ...(isSafari() ? {} : { video: "lowres" }),
    success: (jsep) => {
      const body = { request: "configure", audio: true, video: true };
      state.plugin.send({ message: body, jsep });
    },
    error: (err) => {
      console.error("createOffer error:", err);
      alert("Доступ к камере/микрофону не получен или ошибка WebRTC");
    }
  });
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
  pubs.forEach((p) => {
    (p.streams || []).forEach((st) => {
      subscribe.push({ feed: p.id, mid: st.mid });
    });
  });

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

onMounted(() => { /* ждём кнопку */ });
onBeforeUnmount(leave);
</script>

<style lang="scss" scoped>
@use "@/resources/styles/participants-grid";
@use "@/resources/styles/participants-local-grid";
</style>
