<script setup>
const JANUS_URL = "wss://janus.tulister.com/janus";
const iceServers = [
  { urls: "stun:stun.l.google.com:19302" },
  {
    urls: [
      "turn:janus.website.com:3478?transport=udp",
      "turn:janus.website.com:3478?transport=tcp",
    ],
    username: "janus",
    credential: "supersecret",
  },
];
</script>

<template>
  <h1>You did it!</h1>
  <p>
    Visit <a href="https://vuejs.org/" target="_blank" rel="noopener">vuejs.org</a> to read the
    documentation
  </p>
</template>

<style scoped></style>
