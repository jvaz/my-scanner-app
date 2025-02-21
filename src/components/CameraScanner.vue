<template>
  <div id="app">
    <header>
      <img
        src="https://cdn.prod.website-files.com/65363d385688281ce5b016ec/65363d385688281ce5b016f5_Vectors-Wrapper.svg"
        alt="Celfocus Logo"
      />
    </header>
    <!-- Clicking the container toggles the camera -->
    <div id="camera-container" @click="toggleScanner">
      <!-- Use a ref to target the scanner container -->
      <div id="scanner" ref="scannerContainer"></div>
    </div>
    <div id="output">
      Scanned Result: <span>{{ scannedResult }}</span>
    </div>
    <!-- Display error message if any -->
    <div v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </div>
    <button @click="startScanner" :disabled="isScannerRunning">
      Start Scanner
    </button>
    <button @click="stopScanner" :disabled="!isScannerRunning">
      Stop Scanner
    </button>
  </div>
</template>

<script>
import Quagga from 'quagga';

export default {
  name: 'CameraScanner',
  data() {
    return {
      isScannerRunning: false,
      scannedResult: 'None',
      cameraStream: null,
      errorMessage: '', // Holds error info if camera is unavailable
      errorTimer: null  // Timer to auto-clear error message
    };
  },
  methods: {
    // Helper to show an error message for 5 seconds
    showError(message) {
      if (this.errorTimer) {
        clearTimeout(this.errorTimer);
      }
      this.errorMessage = message;
      this.errorTimer = setTimeout(() => {
        this.errorMessage = '';
        this.errorTimer = null;
      }, 5000);
    },
    startScanner() {
      if (this.isScannerRunning) return;

      // Clear any previous error
      if (this.errorTimer) {
        clearTimeout(this.errorTimer);
        this.errorTimer = null;
      }
      this.errorMessage = '';

      Quagga.init(
        {
          inputStream: {
            type: 'LiveStream',
            // Use the ref for the scanner container
            target: this.$refs.scannerContainer,
            constraints: {
              facingMode: 'environment' // Use the back camera
            }
          },
          decoder: {
            readers: [
              'code_128_reader',
              'ean_reader',
              'ean_8_reader',
              'code_39_reader'
            ]
          }
        },
        (err) => {
          if (err) {
            console.error('QuaggaJS Initialization Error:', err);
            if (err.name === 'NotAllowedError' || err.name === 'PermissionDeniedError') {
              this.showError('Camera access was denied. Please allow camera permissions.');
            } else if (err.name === 'NotReadableError' || err.name === 'TrackStartError') {
              this.showError('The camera is currently in use by another application. Please close it and try again.');
            } else {
              this.showError('Unable to access the camera. Please try again later.');
            }
            return;
          }
          Quagga.start();
          this.isScannerRunning = true;
          console.log('QuaggaJS started');

          // Save the active camera stream
          const video = this.$refs.scannerContainer.querySelector('video');
          this.cameraStream = video ? video.srcObject : null;
        }
      );

      Quagga.onDetected((data) => {
        this.scannedResult = data.codeResult.code;
        console.log('Detected code:', data.codeResult.code);
      });
    },
    stopScanner() {
      if (!this.isScannerRunning) return;

      Quagga.stop();
      // Clear the video element from the scanner container
      this.$refs.scannerContainer.innerHTML = '';
      if (this.cameraStream) {
        this.cameraStream.getTracks().forEach((track) => track.stop());
      }
      Quagga.offDetected();
      this.isScannerRunning = false;
      console.log('QuaggaJS stopped');

      if (this.errorTimer) {
        clearTimeout(this.errorTimer);
        this.errorTimer = null;
      }
      this.errorMessage = '';
    },
    // Toggle scanner when clicking the camera area
    toggleScanner() {
      if (this.isScannerRunning) {
        this.stopScanner();
      } else {
        this.startScanner();
      }
    }
  },
  beforeUnmount() {
    if (this.isScannerRunning) {
      this.stopScanner();
    }
    if (this.errorTimer) {
      clearTimeout(this.errorTimer);
      this.errorTimer = null;
    }
  }
};
</script>

<style scoped>
/* General Reset */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Arial', sans-serif;
  text-align: center;
  background-color: #ffffff;
  color: #333333;
}

header {
  margin: 20px 0;
}

header img {
  width: 150px;
  height: auto;
}

h1 {
  font-size: 36px;
  margin: 20px 0;
  color: #1e1e1e;
  font-weight: bold;
}

#camera-container {
  position: relative;
  width: 100%;
  max-width: 600px;
  height: 350px;
  margin: 20px auto;
  border: 1px solid #e0e0e0;
  background-color: #f4f4f4;
  overflow: hidden;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  cursor: pointer; /* Indicate clickable area */
}

#scanner video {
  width: 100%;
  height: 100%;
  object-fit: cover;
  background-color: #000;
}

#output {
  margin-top: 20px;
  font-size: 1.2em;
  color: #1e1e1e;
  font-weight: bold;
}

.error-message {
  color: red;
  margin: 10px 0;
  font-weight: bold;
}

button {
  padding: 10px 20px;
  margin: 15px 10px;
  font-size: 1em;
  font-weight: bold;
  color: #ffffff;
  background-color: #1e1e1e;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  transition: all 0.3s ease;
}

button:hover {
  background-color: #ff5a5f;
}

button:disabled {
  background-color: #b3b3b3;
  cursor: not-allowed;
}
</style>
