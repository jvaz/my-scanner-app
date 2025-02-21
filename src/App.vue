<template>
  <div id="app">
    <CameraScanner />
  </div>
</template>

<script>
import CameraScanner from './components/CameraScanner.vue';

export default {
  name: 'App',
  components: {
    CameraScanner
  }
};
</script>

<style>
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
