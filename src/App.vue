<template>
  <div class="app">
    <!-- Auth Screen -->
    <div v-if="!isAuthenticated" class="auth-container">
      <div class="auth-background">
        <div class="auth-pattern"></div>
      </div>
      <div class="auth-form">
        <div class="auth-header">
          <div class="app-logo">
            <div class="logo-icon">📹</div>
            <h1>VideoCall</h1>
          </div>
          <p class="app-subtitle">Connect with anyone, anywhere</p>
        </div>

        <div class="tabs">
          <button
            @click="authMode = 'login'"
            :class="{ active: authMode === 'login' }"
            class="tab-btn"
          >
            <span class="tab-icon">🔑</span>
            Login
          </button>
          <button
            @click="authMode = 'register'"
            :class="{ active: authMode === 'register' }"
            class="tab-btn"
          >
            <span class="tab-icon">👤</span>
            Register
          </button>
        </div>

        <form @submit.prevent="authenticate" class="auth-form-content">
          <div class="input-group">
            <div class="input-icon">👤</div>
            <input
              v-model="username"
              placeholder="Enter your username"
              required
              class="auth-input"
              :disabled="loading"
            />
          </div>

          <button type="submit" :disabled="loading" class="auth-submit-btn">
            <span v-if="loading" class="loading-spinner"></span>
            <span v-else>{{
              authMode === "login" ? "Sign In" : "Create Account"
            }}</span>
          </button>
        </form>

        <div v-if="error" class="error-message">
          <span class="error-icon">⚠️</span>
          {{ error }}
        </div>

        <div class="auth-footer">
          <p>Secure video calling powered by WebRTC</p>
        </div>
      </div>
    </div>

    <!-- Main App -->
    <div v-else class="main-container">
      <!-- Call Interface -->
      <div v-if="currentView === 'call-interface'" class="call-interface">
        <!-- Header -->
        <div class="interface-header">
          <div class="user-profile">
            <div class="user-avatar">
              {{ currentUser.username.charAt(0).toUpperCase() }}
            </div>
            <div class="user-details">
              <h2>Welcome back, {{ currentUser.username }}!</h2>
              <p class="user-id">ID: {{ currentUser.userId }}</p>
            </div>
          </div>
          <button @click="logout" class="logout-btn">
            <span class="logout-icon">🚪</span>
            Logout
          </button>
        </div>

        <!-- Quick Call Section -->
        <div class="quick-call-section">
          <div class="section-header">
            <h3>📞 Quick Call</h3>
            <p>Start a video call instantly</p>
          </div>
          <div class="call-input-container">
            <div class="input-wrapper">
              <span class="input-prefix">@</span>
              <input
                v-model="calleeId"
                placeholder="Enter user ID to call"
                class="call-input"
                :disabled="calling"
              />
            </div>
            <button
              @click="initiateCall"
              :disabled="!calleeId || calling"
              class="call-btn"
            >
              <span v-if="calling" class="loading-spinner"></span>
              <span v-else>📞 Call</span>
            </button>
          </div>
        </div>

        <!-- Contacts Section -->
        <div class="contacts-section">
          <div class="section-header">
            <h3>👥 Online Users</h3>
            <p>Connect with available users</p>
          </div>

          <div class="search-container">
            <div class="search-icon">🔍</div>
            <input
              v-model="searchQuery"
              placeholder="Search users..."
              class="search-input"
            />
          </div>

          <div class="contacts-list">
            <div
              v-for="user in filteredUsers"
              :key="user.id"
              class="contact-item"
              @click="callUser(user.id)"
            >
              <div class="contact-avatar">
                {{ user.username.charAt(0).toUpperCase() }}
              </div>
              <div class="contact-info">
                <div class="contact-name">{{ user.username }}</div>
                <div class="contact-id">ID: {{ user.id }}</div>
              </div>
              <div class="contact-actions">
                <div
                  :class="[
                    'status-indicator',
                    user.online ? 'online' : 'offline',
                  ]"
                ></div>
                <span class="status-text">{{
                  user.online ? "Online" : "Offline"
                }}</span>
              </div>
            </div>

            <div v-if="filteredUsers.length === 0" class="no-contacts">
              <div class="no-contacts-icon">👥</div>
              <p>No users found</p>
              <small>Try adjusting your search</small>
            </div>
          </div>
        </div>
      </div>

      <!-- Outgoing Call -->
      <div v-else-if="currentView === 'outgoing-call'" class="call-screen">
        <div class="call-background">
          <div class="call-pattern"></div>
        </div>

        <div class="call-info">
          <div class="caller-avatar">
            {{ (calleeInfo.username || calleeId).charAt(0).toUpperCase() }}
          </div>
          <h3>Calling...</h3>
          <p>{{ calleeInfo.username || calleeId }}</p>
          <div class="call-status">
            <div class="pulse-dot"></div>
            <span>Connecting...</span>
          </div>
        </div>

        <!-- Show local video during outgoing call -->
        <div class="outgoing-video-container">
          <video
            ref="localVideo"
            autoplay
            muted
            playsinline
            webkit-playsinline
            class="outgoing-local-video"
          ></video>
          <div class="video-overlay">
            <div class="video-label">You</div>
          </div>
        </div>

        <button @click="endCall" class="end-call-btn">
          <span class="btn-icon">📞</span>
          End Call
        </button>
      </div>

      <!-- Incoming Call -->
      <div v-else-if="currentView === 'incoming-call'" class="call-screen">
        <div class="call-background">
          <div class="call-pattern"></div>
        </div>

        <div class="call-info">
          <div class="caller-avatar">
            {{ incomingCall.callerName.charAt(0).toUpperCase() }}
          </div>
          <h3>Incoming Call</h3>
          <p>{{ incomingCall.callerName }}</p>
          <div class="call-status">
            <div class="pulse-dot"></div>
            <span>Incoming...</span>
          </div>
        </div>

        <div class="call-actions">
          <button @click="acceptCall" class="accept-btn">
            <span class="btn-icon">✅</span>
            Accept
          </button>
          <button @click="rejectCall" class="reject-btn">
            <span class="btn-icon">❌</span>
            Reject
          </button>
        </div>
      </div>

      <!-- Video Call -->
      <div v-else-if="currentView === 'video-call'" class="video-call">
        <!-- Call Header -->
        <div class="call-header">
          <div class="call-info">
            <h3>{{ calleeInfo.username || "Video Call" }}</h3>
            <div class="call-duration">00:00</div>
          </div>
        </div>

        <div class="video-container">
          <!-- Remote video (big) -->
          <div class="remote-wrap">
            <video
              ref="remoteVideo"
              autoplay
              playsinline
              webkit-playsinline
              class="remote-video"
            ></video>

            <!-- Local preview (small) -->
            <video
              ref="localVideo"
              autoplay
              muted
              playsinline
              webkit-playsinline
              class="local-video"
            ></video>

            <!-- Play button overlay if browser blocked autoplay -->
            <div v-if="showRemotePlayButton" class="play-overlay">
              <button @click="playRemote" class="play-btn">
                <span class="play-icon">▶️</span>
                Click to play
              </button>
            </div>
          </div>
        </div>

        <!-- Call Controls -->
        <div class="call-controls">
          <button class="control-btn mute-btn" title="Mute">
            <span class="btn-icon">🎤</span>
          </button>

          <button class="control-btn video-btn" title="Turn off video">
            <span class="btn-icon">📷</span>
          </button>

          <button @click="endCall" class="end-call-btn">
            <span class="btn-icon">📞</span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted, watch } from "vue";
import { io } from "socket.io-client";

/* ---------- Reactive state ---------- */
const isAuthenticated = ref(false);
const authMode = ref("login");
const username = ref("");
const loading = ref(false);
const error = ref("");
const currentUser = reactive({ username: "", userId: "" });
const currentView = ref("call-interface");
const calleeId = ref("");
const searchQuery = ref("");
const users = ref([]);
const calling = ref(false);
const incomingCall = ref(null);
const calleeInfo = ref({});
const currentCallId = ref(null);

/* ---------- DOM refs ---------- */
const localVideo = ref(null);
const remoteVideo = ref(null);
const showRemotePlayButton = ref(false);

/* ---------- WebRTC / Socket state ---------- */
let localStream = null;
let peerConnection = null;
let socket = null;
let pendingRemoteCandidates = []; // queue ICE until remoteDescription exists
let remoteMediaStream = null; // persist remote stream even if element not mounted yet
let callTimeout = null; // timeout for call establishment

/* ---------- helpers ---------- */
const filteredUsers = computed(() =>
  users.value.filter(
    (u) =>
      u.id !== currentUser.userId &&
      u.username.toLowerCase().includes(searchQuery.value.toLowerCase())
  )
);

/* ---------- Create peer connection and handlers ---------- */
const createPeerConnectionIfNeeded = () => {
  if (peerConnection) return;

  peerConnection = new RTCPeerConnection({
    iceServers: [{ urls: "stun:stun.l.google.com:19302" }],
  });

  peerConnection.onicecandidate = (ev) => {
    if (ev.candidate && socket) {
      console.log("Sending ICE candidate", ev.candidate);
      socket.emit("ice-candidate", {
        callId: currentCallId.value,
        candidate: ev.candidate,
      });
    }
  };

  peerConnection.ontrack = (ev) => {
    console.log("ontrack event streams:", ev.streams);
    console.log("ontrack event track:", ev.track);
    const stream = ev.streams && ev.streams[0] ? ev.streams[0] : null;
    if (stream) {
      // persist and attach when possible
      remoteMediaStream = stream;
      console.log("Remote stream received:", stream);
      if (remoteVideo.value && remoteVideo.value.srcObject !== stream) {
        remoteVideo.value.srcObject = stream;
        remoteVideo.value.playsInline = true;
        remoteVideo.value.play().catch((err) => {
          console.warn("remote.play() blocked:", err);
          showRemotePlayButton.value = true;
        });
        console.log("Remote video attached and playing");
      } else {
        console.log(
          "Remote video element not ready, stream will be attached via watcher"
        );
      }
    } else {
      // fallback: create MediaStream from track
      const fallbackStream = new MediaStream([ev.track]);
      remoteMediaStream = fallbackStream;
      console.log("Fallback remote stream created:", fallbackStream);
      if (remoteVideo.value && remoteVideo.value.srcObject !== fallbackStream) {
        remoteVideo.value.srcObject = fallbackStream;
        remoteVideo.value.playsInline = true;
        remoteVideo.value
          .play()
          .catch(() => (showRemotePlayButton.value = true));
        console.log("Fallback remote video attached and playing");
      } else {
        console.log(
          "Remote video element not ready, fallback stream will be attached via watcher"
        );
      }
    }
  };

  peerConnection.onconnectionstatechange = () => {
    console.log("PC state:", peerConnection.connectionState);
    if (
      ["disconnected", "failed", "closed"].includes(
        peerConnection.connectionState || ""
      )
    ) {
      console.log("Connection lost, cleaning up call");
      cleanupCall();
    }
  };

  // Monitor ICE connection state for better debugging
  peerConnection.oniceconnectionstatechange = () => {
    console.log("ICE connection state:", peerConnection.iceConnectionState);
    if (peerConnection.iceConnectionState === "failed") {
      console.log("ICE connection failed, attempting to restart ICE");
      // Try to restart ICE
      peerConnection.restartIce();
    }
  };

  // Monitor ICE gathering state
  peerConnection.onicegatheringstatechange = () => {
    console.log("ICE gathering state:", peerConnection.iceGatheringState);
  };
};

/* ---------- getUserMedia + attach to local video ---------- */
const startLocalStream = async () => {
  if (localStream) return localStream;
  try {
    // Mobile-optimized constraints
    const constraints = {
      video: {
        width: { ideal: 640, max: 1280 },
        height: { ideal: 480, max: 720 },
        facingMode: "user", // Front camera for mobile
        frameRate: { ideal: 15, max: 30 }, // Lower frame rate for mobile
      },
      audio: {
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: true,
      },
    };

    localStream = await navigator.mediaDevices.getUserMedia(constraints);
    console.log("Got local stream", localStream);
    console.log("Local stream tracks:", localStream.getTracks());

    // Attach to local video element if it exists
    if (localVideo.value) {
      console.log("Attaching local stream to video element");
      localVideo.value.srcObject = localStream;
      // local is muted to avoid echo
      localVideo.value.muted = true;
      localVideo.value.playsInline = true; // Important for iOS
      localVideo.value
        .play()
        .catch((e) => console.warn("localVideo.play() blocked:", e));
    } else {
      console.log(
        "Local video element not available yet, will attach via watcher"
      );
    }

    return localStream;
  } catch (err) {
    console.error("getUserMedia error:", err);
    // Show user-friendly error for mobile
    if (err.name === "NotAllowedError") {
      alert(
        "Camera and microphone access is required. Please allow permissions and try again."
      );
    } else if (err.name === "NotFoundError") {
      alert("No camera or microphone found. Please check your device.");
    } else {
      alert("Failed to access camera/microphone. Please try again.");
    }
    throw err;
  }
};

/* ---------- Drain queued ICE candidates ---------- */
const drainPendingCandidates = async () => {
  if (!peerConnection || !peerConnection.remoteDescription) return;
  for (const c of pendingRemoteCandidates) {
    try {
      await peerConnection.addIceCandidate(new RTCIceCandidate(c));
    } catch (err) {
      console.warn("failed to add queued candidate", err);
    }
  }
  pendingRemoteCandidates = [];
};

/* ---------- Socket initialization ---------- */
const initializeSocket = () => {
  socket = io("https://5d1f56feeee4.ngrok-free.app");

  socket.on("connect", () => {
    console.log("socket connected", socket.id);
    if (currentUser.userId) socket.emit("join", currentUser.userId);
  });

  socket.on("user-online", () => loadUsers());
  socket.on("user-offline", () => loadUsers());

  socket.on("call-initiated", ({ callId }) => {
    console.log("call-initiated -> callId", callId);
    currentCallId.value = callId;
  });

  socket.on("incoming-call", (data) => {
    console.log("incoming-call", data);
    incomingCall.value = data;
    currentCallId.value = data.callId;
    currentView.value = "incoming-call";
    // stow callee info for UI
    calleeInfo.value = { username: data.callerName, id: data.callerId };
  });

  socket.on("call-accepted", async (data) => {
    console.log("call-accepted", data);
    currentCallId.value = data.callId;

    // Clear call timeout
    if (callTimeout) {
      clearTimeout(callTimeout);
      callTimeout = null;
    }

    // On caller side: set remote description (answer) and drain queued ICE
    try {
      createPeerConnectionIfNeeded();
      await peerConnection.setRemoteDescription(
        new RTCSessionDescription(data.answer)
      );
      await drainPendingCandidates();

      // Switch to video call view
      currentView.value = "video-call";

      // Ensure videos play after view change (important for mobile)
      setTimeout(() => {
        if (localVideo.value && localVideo.value.srcObject) {
          localVideo.value.play().catch(console.warn);
        }
        if (remoteVideo.value && remoteVideo.value.srcObject) {
          remoteVideo.value.play().catch(console.warn);
        }
      }, 100);

      console.log("Call accepted, switched to video call view");
    } catch (err) {
      console.error("Error applying answer on caller:", err);
      alert("Failed to establish video call. Please try again.");
      cleanupCall();
    }
  });

  socket.on("call-rejected", () => {
    console.log("call rejected");
    calling.value = false;
    currentView.value = "call-interface";
    alert("Call was rejected or callee unavailable");
  });

  socket.on("call-ended", () => {
    console.log("call-ended received");
    cleanupCall();
  });

  socket.on("ice-candidate", async (data) => {
    // data: { callId, candidate } or candidate
    const candidate = data?.candidate ?? data;
    console.log("received ice-candidate", candidate);
    if (!peerConnection || !peerConnection.remoteDescription) {
      pendingRemoteCandidates.push(candidate);
      return;
    }
    try {
      await peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
    } catch (err) {
      console.warn("Error adding ICE candidate:", err);
      // still okay to queue
      pendingRemoteCandidates.push(candidate);
    }
  });
};

/* ---------- Call flow: caller initiates ---------- */
const initiateCall = async () => {
  if (!calleeId.value) return;

  try {
    calling.value = true;
    currentView.value = "outgoing-call";

    // Start local stream first (important for mobile)
    const stream = await startLocalStream();

    // Create peer connection after getting stream
    createPeerConnectionIfNeeded();

    // Add tracks to peer connection
    stream.getTracks().forEach((t) => peerConnection.addTrack(t, stream));

    // Create offer with proper constraints for mobile
    const offer = await peerConnection.createOffer({
      offerToReceiveAudio: true,
      offerToReceiveVideo: true,
      voiceActivityDetection: true,
    });

    await peerConnection.setLocalDescription(offer);

    // Send offer to server (server will forward to callee)
    socket.emit("initiate-call", {
      callerId: currentUser.userId,
      calleeId: calleeId.value,
      offer: peerConnection.localDescription,
    });

    // Set timeout for call establishment (30 seconds)
    callTimeout = setTimeout(() => {
      if (currentView.value === "outgoing-call") {
        console.log("Call timeout - no answer received");
        alert("Call timed out. The other person may not be available.");
        cleanupCall();
      }
    }, 30000);

    console.log("Call initiated successfully");
  } catch (err) {
    console.error("initiateCall error:", err);
    alert("Failed to start call. Please try again.");
    cleanupCall();
  }
};

/* ---------- When user clicks contact ---------- */
const callUser = (userId) => {
  calleeId.value = userId;
  initiateCall();
};

/* ---------- Callee accepts the call ---------- */
const acceptCall = async () => {
  if (!incomingCall.value) return;

  // mark current call id
  currentCallId.value = incomingCall.value.callId;

  // IMPORTANT ORDER:
  // 1. create pc (to receive tracks)
  // 2. setRemoteDescription(offer)
  // 3. getUserMedia and add local tracks
  // 4. createAnswer & setLocalDescription
  // 5. send answer

  try {
    createPeerConnectionIfNeeded();

    // Step 2 - set remote (the caller's offer)
    await peerConnection.setRemoteDescription(
      new RTCSessionDescription(incomingCall.value.offer)
    );

    // Step 3 - get local stream and add tracks
    // For mobile, we need to ensure this happens after user interaction
    const stream = await startLocalStream();
    stream.getTracks().forEach((t) => peerConnection.addTrack(t, stream));

    // Step 4 - create answer and set local
    const answer = await peerConnection.createAnswer();
    await peerConnection.setLocalDescription(answer);

    // Step 5 - send answer back to caller
    socket.emit("accept-call", {
      callId: incomingCall.value.callId,
      answer: peerConnection.localDescription,
    });

    // Drain queued candidates (if any)
    await drainPendingCandidates();

    // Switch to video call view FIRST
    currentView.value = "video-call";

    // Wait for DOM update, then ensure videos play
    await new Promise((resolve) => setTimeout(resolve, 200));

    // Ensure local video is attached and playing
    if (localVideo.value && localStream) {
      localVideo.value.srcObject = localStream;
      localVideo.value.muted = true;
      localVideo.value.playsInline = true;
      try {
        await localVideo.value.play();
        console.log("Local video playing successfully");
      } catch (err) {
        console.warn("Local video play failed:", err);
      }
    }

    // Ensure remote video plays if available
    if (remoteVideo.value && remoteVideo.value.srcObject) {
      try {
        await remoteVideo.value.play();
        console.log("Remote video playing successfully");
      } catch (err) {
        console.warn("Remote video play failed:", err);
        showRemotePlayButton.value = true;
      }
    }
  } catch (err) {
    console.error("acceptCall error", err);
    // Show user-friendly error
    alert("Failed to accept call. Please try again.");
    cleanupCall();
  }
};

/* ---------- Reject call ---------- */
const rejectCall = () => {
  if (!incomingCall.value) return;
  socket.emit("reject-call", { callId: incomingCall.value.callId });
  cleanupCall();
};

/* ---------- End/cleanup ---------- */
const cleanupCall = () => {
  try {
    if (localStream) {
      localStream.getTracks().forEach((t) => t.stop());
      localStream = null;
    }
  } catch (e) {}

  try {
    if (peerConnection) {
      peerConnection.close();
      peerConnection = null;
    }
  } catch (e) {}

  // Clear call timeout
  if (callTimeout) {
    clearTimeout(callTimeout);
    callTimeout = null;
  }

  pendingRemoteCandidates = [];
  calling.value = false;
  currentCallId.value = null;
  calleeId.value = "";
  calleeInfo.value = {};
  incomingCall.value = null;
  currentView.value = "call-interface";
  showRemotePlayButton.value = false;
  if (remoteVideo.value) remoteVideo.value.srcObject = null;
  if (localVideo.value) localVideo.value.srcObject = null;
  remoteMediaStream = null;
};

const endCall = () => {
  if (currentCallId.value && socket) {
    socket.emit("end-call", { callId: currentCallId.value });
  }
  cleanupCall();
};

/* ---------- Play remote (user click if autoplay blocked) ---------- */
const playRemote = async () => {
  if (!remoteVideo.value) return;
  try {
    await remoteVideo.value.play();
    showRemotePlayButton.value = false;
  } catch (err) {
    console.warn("playRemote failed", err);
  }
};

/* ---------- Auth + users ---------- */
const authenticate = async () => {
  loading.value = true;
  error.value = "";
  try {
    const res = await fetch(
      `https://5d1f56feeee4.ngrok-free.app/api/auth/${authMode.value}`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ username: username.value }),
      }
    );
    const data = await res.json();
    if (!res.ok) {
      error.value = data.error || "Auth failed";
      loading.value = false;
      return;
    }

    currentUser.username = data.username;
    currentUser.userId = data.userId;
    isAuthenticated.value = true;
    initializeSocket();
    loadUsers();
  } catch (err) {
    console.error(err);
    error.value = "Network error";
  } finally {
    loading.value = false;
  }
};

const loadUsers = async () => {
  try {
    const res = await fetch("https://5d1f56feeee4.ngrok-free.app/api/users");
    const data = await res.json();
    users.value = data;
  } catch (err) {
    console.error("loadUsers error", err);
  }
};

const logout = () => {
  if (socket) {
    socket.disconnect();
    socket = null;
  }
  cleanupCall();
  isAuthenticated.value = false;
  currentUser.username = "";
  currentUser.userId = "";
  username.value = "";
  error.value = "";
};

/* ---------- Lifecycle ---------- */
onUnmounted(() => {
  if (socket) socket.disconnect();
  cleanupCall();
});

/* ---------- For convenience during dev: auto open socket if already logged in ---------- */
onMounted(() => {
  // nothing to do. user clicks login/register.
});

// Ensure remote stream attaches once the video element exists and view is active
watch([currentView, remoteVideo], async () => {
  if (
    currentView.value === "video-call" &&
    remoteVideo.value &&
    remoteMediaStream
  ) {
    console.log("Watcher: Attaching remote stream to video element");
    if (remoteVideo.value.srcObject !== remoteMediaStream) {
      remoteVideo.value.srcObject = remoteMediaStream;
      remoteVideo.value.playsInline = true;
    }
    try {
      await remoteVideo.value.play();
      showRemotePlayButton.value = false;
      console.log("Watcher: Remote video playing successfully");
    } catch (err) {
      console.warn("Watcher: remoteVideo play after mount blocked:", err);
      showRemotePlayButton.value = true;
    }
  }
});

// Ensure local stream attaches once the video element exists and view is active
watch([currentView, localVideo], async () => {
  if (currentView.value === "video-call" && localVideo.value && localStream) {
    console.log("Watcher: Attaching local stream to video element");
    if (localVideo.value.srcObject !== localStream) {
      localVideo.value.srcObject = localStream;
      localVideo.value.muted = true; // Ensure muted to avoid echo
      localVideo.value.playsInline = true;
    }
    try {
      await localVideo.value.play();
      console.log("Watcher: Local video playing successfully");
    } catch (err) {
      console.warn("Watcher: localVideo play after mount blocked:", err);
    }
  }
});
</script>

<style scoped>
/* ===== GLOBAL STYLES ===== */
* {
  box-sizing: border-box;
}

.app {
  min-height: 100vh;
  background: #f5f5f5;
  padding: 0;
  box-sizing: border-box;
  font-family: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    sans-serif;
}

/* ===== AUTH CONTAINER ===== */
.auth-container {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  overflow: hidden;
}

.auth-background {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.auth-pattern {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: radial-gradient(
      circle at 25% 25%,
      rgba(255, 255, 255, 0.1) 0%,
      transparent 50%
    ),
    radial-gradient(
      circle at 75% 75%,
      rgba(255, 255, 255, 0.1) 0%,
      transparent 50%
    );
  animation: float 20s ease-in-out infinite;
}

@keyframes float {
  0%,
  100% {
    transform: translateY(0px) rotate(0deg);
  }
  50% {
    transform: translateY(-20px) rotate(180deg);
  }
}

.auth-form {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(20px);
  padding: 40px;
  border-radius: 20px;
  width: 100%;
  max-width: 400px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  position: relative;
  z-index: 10;
  animation: slideUp 0.6s ease-out;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.auth-header {
  text-align: center;
  margin-bottom: 30px;
}

.app-logo {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  margin-bottom: 8px;
}

.logo-icon {
  font-size: 32px;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%,
  100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.1);
  }
}

.app-logo h1 {
  margin: 0;
  font-size: 28px;
  font-weight: 700;
  background: linear-gradient(135deg, #667eea, #764ba2);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.app-subtitle {
  color: #666;
  margin: 0;
  font-size: 14px;
}

.tabs {
  display: flex;
  gap: 8px;
  margin-bottom: 30px;
  background: #f8f9fa;
  padding: 4px;
  border-radius: 12px;
}

.tab-btn {
  flex: 1;
  padding: 12px 16px;
  border: none;
  background: transparent;
  cursor: pointer;
  border-radius: 8px;
  font-weight: 500;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.tab-btn:hover {
  background: rgba(102, 126, 234, 0.1);
}

.tab-btn.active {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
}

.tab-icon {
  font-size: 16px;
}

.auth-form-content {
  margin-bottom: 20px;
}

.input-group {
  position: relative;
  margin-bottom: 20px;
}

.input-icon {
  position: absolute;
  left: 16px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 18px;
  color: #999;
  z-index: 2;
}

.auth-input {
  width: 100%;
  padding: 16px 16px 16px 50px;
  border: 2px solid #e9ecef;
  border-radius: 12px;
  font-size: 16px;
  transition: all 0.3s ease;
  background: white;
  box-sizing: border-box;
}

.auth-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.auth-input:disabled {
  background: #f8f9fa;
  cursor: not-allowed;
}

.auth-submit-btn {
  width: 100%;
  padding: 16px;
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  border: none;
  border-radius: 12px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.auth-submit-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(102, 126, 234, 0.3);
}

.auth-submit-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
  transform: none;
}

.loading-spinner {
  width: 20px;
  height: 20px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top: 2px solid white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

.error-message {
  background: #fee;
  color: #c33;
  padding: 12px 16px;
  border-radius: 8px;
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  gap: 8px;
  border: 1px solid #fcc;
  animation: shake 0.5s ease-in-out;
}

@keyframes shake {
  0%,
  100% {
    transform: translateX(0);
  }
  25% {
    transform: translateX(-5px);
  }
  75% {
    transform: translateX(5px);
  }
}

.error-icon {
  font-size: 16px;
}

.auth-footer {
  text-align: center;
  color: #999;
  font-size: 12px;
}

.auth-footer p {
  margin: 0;
}

/* ===== MAIN CONTAINER ===== */
.main-container {
  min-height: 100vh;
  background: #f8f9fa;
  padding: 0;
}

/* ===== CALL INTERFACE ===== */
.call-interface {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
}

.interface-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: white;
  padding: 24px 32px;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  margin-bottom: 24px;
  border: 1px solid #e9ecef;
}

.user-profile {
  display: flex;
  align-items: center;
  gap: 16px;
}

.user-avatar {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea, #764ba2);
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 24px;
  font-weight: 700;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
}

.user-details h2 {
  margin: 0;
  font-size: 24px;
  font-weight: 700;
  color: #2c3e50;
}

.user-id {
  margin: 4px 0 0 0;
  color: #6c757d;
  font-size: 14px;
  font-family: "Courier New", monospace;
}

.logout-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 20px;
  background: #f8f9fa;
  border: 1px solid #e9ecef;
  border-radius: 12px;
  color: #6c757d;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
}

.logout-btn:hover {
  background: #e9ecef;
  color: #495057;
  transform: translateY(-1px);
}

.logout-icon {
  font-size: 16px;
}

.quick-call-section,
.contacts-section {
  background: white;
  border-radius: 16px;
  padding: 24px 32px;
  margin-bottom: 24px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  border: 1px solid #e9ecef;
}

.section-header {
  margin-bottom: 24px;
}

.section-header h3 {
  margin: 0 0 8px 0;
  font-size: 20px;
  font-weight: 700;
  color: #2c3e50;
}

.section-header p {
  margin: 0;
  color: #6c757d;
  font-size: 14px;
}

.call-input-container {
  display: flex;
  gap: 12px;
  align-items: center;
}

.input-wrapper {
  flex: 1;
  position: relative;
  display: flex;
  align-items: center;
}

.input-prefix {
  position: absolute;
  left: 16px;
  color: #6c757d;
  font-weight: 600;
  font-size: 16px;
  z-index: 2;
}

.call-input {
  width: 100%;
  padding: 16px 16px 16px 40px;
  border: 2px solid #e9ecef;
  border-radius: 12px;
  font-size: 16px;
  transition: all 0.3s ease;
  background: white;
  box-sizing: border-box;
}

.call-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.call-input:disabled {
  background: #f8f9fa;
  cursor: not-allowed;
}

.call-btn {
  padding: 16px 24px;
  background: linear-gradient(135deg, #28a745, #20c997);
  color: white;
  border: none;
  border-radius: 12px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 8px;
  white-space: nowrap;
}

.call-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(40, 167, 69, 0.3);
}

.call-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
  transform: none;
}

.search-container {
  position: relative;
  margin-bottom: 20px;
}

.search-icon {
  position: absolute;
  left: 16px;
  top: 50%;
  transform: translateY(-50%);
  color: #6c757d;
  font-size: 16px;
  z-index: 2;
}

.search-input {
  width: 100%;
  padding: 16px 16px 16px 50px;
  border: 2px solid #e9ecef;
  border-radius: 12px;
  font-size: 16px;
  transition: all 0.3s ease;
  background: white;
  box-sizing: border-box;
}

.search-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.contacts-list {
  max-height: 400px;
  overflow-y: auto;
  border-radius: 12px;
}

.contact-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 1px solid transparent;
  margin-bottom: 8px;
}

.contact-item:hover {
  background: #f8f9fa;
  border-color: #e9ecef;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.contact-avatar {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea, #764ba2);
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 18px;
  font-weight: 700;
  flex-shrink: 0;
}

.contact-info {
  flex: 1;
  min-width: 0;
}

.contact-name {
  font-weight: 600;
  color: #2c3e50;
  margin-bottom: 4px;
  font-size: 16px;
}

.contact-id {
  color: #6c757d;
  font-size: 12px;
  font-family: "Courier New", monospace;
}

.contact-actions {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 4px;
}

.status-indicator {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  flex-shrink: 0;
}

.status-indicator.online {
  background: #28a745;
  box-shadow: 0 0 8px rgba(40, 167, 69, 0.4);
}

.status-indicator.offline {
  background: #6c757d;
}

.status-text {
  font-size: 12px;
  color: #6c757d;
  font-weight: 500;
}

.no-contacts {
  text-align: center;
  padding: 40px 20px;
  color: #6c757d;
}

.no-contacts-icon {
  font-size: 48px;
  margin-bottom: 16px;
  opacity: 0.5;
}

.no-contacts p {
  margin: 0 0 8px 0;
  font-size: 16px;
  font-weight: 500;
}

.no-contacts small {
  font-size: 14px;
  opacity: 0.7;
}

/* ===== CALL SCREENS ===== */
.call-screen {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  position: relative;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 20px;
}

.call-background {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.call-pattern {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: radial-gradient(
      circle at 25% 25%,
      rgba(255, 255, 255, 0.1) 0%,
      transparent 50%
    ),
    radial-gradient(
      circle at 75% 75%,
      rgba(255, 255, 255, 0.1) 0%,
      transparent 50%
    );
  animation: float 20s ease-in-out infinite;
}

.call-info {
  text-align: center;
  margin-bottom: 40px;
  z-index: 10;
}

.caller-avatar {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 48px;
  font-weight: 700;
  margin: 0 auto 20px;
  border: 3px solid rgba(255, 255, 255, 0.3);
  animation: pulse 2s infinite;
}

.call-info h3 {
  font-size: 32px;
  font-weight: 700;
  margin: 0 0 8px 0;
}

.call-info p {
  font-size: 18px;
  margin: 0 0 16px 0;
  opacity: 0.9;
}

.call-status {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  font-size: 16px;
  opacity: 0.8;
}

.pulse-dot {
  width: 8px;
  height: 8px;
  background: #28a745;
  border-radius: 50%;
  animation: pulse-dot 1.5s ease-in-out infinite;
}

@keyframes pulse-dot {
  0%,
  100% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.2);
    opacity: 0.7;
  }
}

.outgoing-video-container {
  width: 100%;
  max-width: 400px;
  height: 300px;
  background: #000;
  border-radius: 20px;
  overflow: hidden;
  margin: 20px 0;
  position: relative;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.3);
}

.outgoing-local-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transform: scaleX(-1);
}

.video-overlay {
  position: absolute;
  top: 16px;
  left: 16px;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(10px);
  padding: 8px 16px;
  border-radius: 20px;
  color: white;
  font-size: 14px;
  font-weight: 600;
}

.call-actions {
  display: flex;
  gap: 20px;
  margin-top: 40px;
  z-index: 10;
}

.accept-btn,
.reject-btn {
  padding: 20px 40px;
  border: none;
  border-radius: 50px;
  font-size: 18px;
  font-weight: 700;
  cursor: pointer;
  min-width: 140px;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.2);
}

.accept-btn {
  background: linear-gradient(135deg, #28a745, #20c997);
  color: white;
}

.accept-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 12px 35px rgba(40, 167, 69, 0.4);
}

.reject-btn {
  background: linear-gradient(135deg, #dc3545, #c82333);
  color: white;
}

.reject-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 12px 35px rgba(220, 53, 69, 0.4);
}

.end-call-btn {
  padding: 20px 40px;
  border-radius: 50px;
  background: linear-gradient(135deg, #dc3545, #c82333);
  color: white;
  border: none;
  cursor: pointer;
  font-size: 18px;
  font-weight: 700;
  min-width: 160px;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.2);
  z-index: 10;
}

.end-call-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 12px 35px rgba(220, 53, 69, 0.4);
}

.btn-icon {
  font-size: 20px;
}

/* ===== VIDEO CALL ===== */
.video-call {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  height: 100vh;
  background: #000;
  color: white;
}

.call-header {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 20;
  background: rgba(0, 0, 0, 0.7);
  padding: 15px 20px;
  border-radius: 12px;
  backdrop-filter: blur(10px);
}

.call-header h3 {
  margin: 0;
  font-size: 18px;
  font-weight: 600;
}

.call-duration {
  font-size: 14px;
  color: #ccc;
  margin-top: 5px;
  font-family: "Courier New", monospace;
}

.video-container {
  position: relative;
  width: 100%;
  max-width: 900px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 12px;
  flex: 1;
}

.remote-wrap {
  position: relative;
  width: 100%;
  max-width: 640px;
  height: 480px;
  background: #000;
  border-radius: 8px;
  overflow: hidden;
}

.remote-video {
  width: 100%;
  height: 100%;
  background: #000;
  border-radius: 8px;
  object-fit: cover;
  transform: scaleX(-1);
}

.local-video {
  position: absolute;
  bottom: 12px;
  right: 12px;
  width: 200px;
  height: 150px;
  border: 2px solid #fff;
  border-radius: 8px;
  object-fit: cover;
  background: #000;
  z-index: 10;
  transform: scaleX(-1);
  transition: opacity 0.3s ease;
}

.play-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(0, 0, 0, 0.25);
  border-radius: 8px;
}

.play-btn {
  padding: 16px 32px;
  border-radius: 12px;
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  border: none;
  cursor: pointer;
  font-size: 16px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 8px;
  transition: all 0.3s ease;
}

.play-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(102, 126, 234, 0.3);
}

.play-icon {
  font-size: 18px;
}

.call-controls {
  position: absolute;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 20px;
  align-items: center;
  background: rgba(0, 0, 0, 0.8);
  padding: 15px 25px;
  border-radius: 50px;
  backdrop-filter: blur(10px);
  z-index: 20;
}

.control-btn {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  border: none;
  background: rgba(255, 255, 255, 0.2);
  color: white;
  font-size: 24px;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

.control-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: scale(1.1);
}

.control-btn.active {
  background: #dc3545;
  transform: scale(1.1);
}

.control-btn.active:hover {
  background: #c82333;
}

.end-call-btn {
  width: 30px;
  height: 60px;
  border-radius: 10px;
  background: #dc3545;
  color: white;
  border: none;
  font-size: 24px;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

.end-call-btn:hover {
  background: #c82333;
  transform: scale(1.1);
}

/* ===== MOBILE RESPONSIVE ===== */
@media (max-width: 768px) {
  .app {
    padding: 0;
  }

  .auth-form {
    margin: 20px;
    padding: 30px 20px;
  }

  .call-interface {
    padding: 16px;
  }

  .interface-header {
    flex-direction: column;
    gap: 16px;
    align-items: flex-start;
    padding: 20px;
  }

  .user-profile {
    width: 100%;
  }

  .logout-btn {
    width: 100%;
    justify-content: center;
  }

  .quick-call-section,
  .contacts-section {
    padding: 20px;
    margin-bottom: 16px;
  }

  .call-input-container {
    flex-direction: column;
    gap: 12px;
  }

  .call-btn {
    width: 100%;
    justify-content: center;
  }

  .contact-item {
    padding: 12px;
  }

  .contact-avatar {
    width: 40px;
    height: 40px;
    font-size: 16px;
  }

  .contact-name {
    font-size: 14px;
  }

  .contact-id {
    font-size: 11px;
  }

  .call-screen {
    padding: 16px;
  }

  .caller-avatar {
    width: 100px;
    height: 100px;
    font-size: 40px;
  }

  .call-info h3 {
    font-size: 24px;
  }

  .call-info p {
    font-size: 16px;
  }

  .outgoing-video-container {
    max-width: 100%;
    height: 250px;
  }

  .call-actions {
    flex-direction: column;
    gap: 15px;
    width: 100%;
    max-width: 300px;
  }

  .accept-btn,
  .reject-btn,
  .end-call-btn {
    width: 100%;
    padding: 20px;
    font-size: 20px;
  }

  .video-call {
    height: 100vh;
    padding: 0;
  }

  .call-header {
    top: 10px;
    left: 10px;
    right: 10px;
    padding: 10px 15px;
  }

  .call-header h3 {
    font-size: 16px;
  }

  .call-duration {
    font-size: 12px;
  }

  .remote-wrap {
    width: 100%;
    height: 60vh;
    max-height: none;
  }

  .local-video {
    width: 120px;
    height: 90px;
    bottom: 100px;
    right: 10px;
  }

  .video-container {
    padding: 0;
  }

  .call-controls {
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    gap: 15px;
    padding: 12px 20px;
  }

  .control-btn,
  .end-call-btn {
    width: 50px;
    height: 50px;
    font-size: 20px;
  }
}

@media (max-width: 480px) {
  .auth-form {
    margin: 10px;
    padding: 20px 15px;
  }

  .app-logo h1 {
    font-size: 24px;
  }

  .caller-avatar {
    width: 80px;
    height: 80px;
    font-size: 32px;
  }

  .call-info h3 {
    font-size: 20px;
  }

  .outgoing-video-container {
    height: 200px;
  }

  .local-video {
    width: 100px;
    height: 75px;
    bottom: 80px;
    right: 8px;
  }
}
</style>
