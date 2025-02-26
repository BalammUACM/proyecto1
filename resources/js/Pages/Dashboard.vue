<script setup>
///////////////////////////////LIBRERIAS//////////////////////////////////////////////////////////////////
import AppLayout from '@/Layouts/AppLayout.vue';
import Welcome from '@/Components/Welcome.vue';
import { ref, onMounted,watch,computed } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
/////////////////////////////////////////Escena///////////////////////////////////////////////////////////
function closeOverlay() {
    const overlay = document.getElementById('overlayPanel');
    overlay.style.transform = 'translateX(100%)'; // Oculta el panel deslizándolo fuera de la vista
}
function abrirOverlay(){
    const overlay = document.getElementById('overlayPanel');
    overlay.style.transform = 'translateX(0%)'; // aparece el panel deslizándolo a la vista
}
//////////////////////////////////////////////////////////////////////////////////////////////////////////
const target = ref();
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xabcdef);
// luz ambiental
const ambientLight = new THREE.AmbientLight(0xffffff, 0.5); // Luz blanca con intensidad 0.5
scene.add(ambientLight);
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.z = 0.00;
camera.position.y = -0.57;
camera.position.x = 0.00;
//////////preparacion de modelo 3d ////////////////////////////////////////////////////////////////////////
const renderer = new THREE.WebGLRenderer();
renderer.setSize(1350,520);//520
document.body.appendChild(renderer.domElement);
// Inicializar controles de órbita después de la cámara y el renderizador
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true; // Habilita amortiguación para un movimiento más fluido
controls.dampingFactor = 0.05; // Ajusta la velocidad de amortiguación
controls.rotateSpeed = 0.5; // Ajusta la sensibilidad de rotación
controls.enableRotate = false;  // Desactiva la rotación con el mouse
//////////////////////////// Parte numerica Denavit Hartemberg /////////////////////////////////////////////
// Parámetros del brazo robótico (longitudes de los eslabones)
// Evento para mostrar las coordenadas de la cámara cuando los controles cambian
/* controls.addEventListener('change', () => {
    console.log(`Posición de la cámara: x=${camera.position.x.toFixed(2)}, y=${camera.position.y.toFixed(2)}, z=${camera.position.z.toFixed(2)}`);
});  */
const l1 = 0.12*0;
const l2 = 0.24;
const l3 = 0.03;
const l4 = 0.28;

// Variables para los sliders
const q1Angle = ref(0);
const q2Angle = ref(0);
const q3Angle = ref(0);

// Cargar el modelo GLB
let robot;
const loader = new GLTFLoader();
loader.load('models/glb/brazo_v5.glb', function (gltf) {
    robot = gltf.scene;
    scene.add(robot);
}, undefined, function (error) {
    console.error(error);
});
// Crear una matriz de transformación usando los parámetros DH
// Función para crear una matriz de transformación DH
function dhMatrix(alpha, a, d, theta) {
    const cosTheta = Math.cos(theta);
    const sinTheta = Math.sin(theta);
    const cosAlpha = Math.cos(alpha);
    const sinAlpha = Math.sin(alpha);

    return new THREE.Matrix4().set(
        cosTheta, -sinTheta, 0, a,
        sinTheta * cosAlpha, cosTheta * cosAlpha, -sinAlpha, -sinAlpha * d,
        sinTheta * sinAlpha, cosTheta * sinAlpha, cosAlpha, cosAlpha * d,
        0, 0, 0, 1
    );
}
// Función para aplicar la matriz DH a un eslabón
function applyDHTransform(eslabon, dhMatrix,origen) {
    const x=dhMatrix.elements[0+4*3]+origen.x;
    const y=dhMatrix.elements[1+4*3]+origen.y;
    const z=dhMatrix.elements[2+4*3]+origen.z;
    eslabon.setRotationFromMatrix(dhMatrix);
    eslabon.position.set(x,y,z);
}
// Función para actualizar el brazo robótico según los ángulos de los sliders
function updateRobot() {
    if (!robot) return;  // Verificar si el modelo ha sido cargado

    // Leer los ángulos de los sliders (convertir a radianes)
    const q1 = THREE.MathUtils.degToRad(q1Angle.value);
    const q2 = THREE.MathUtils.degToRad(q2Angle.value);
    const q3 = THREE.MathUtils.degToRad(q3Angle.value);

    // Matrices de transformación DH
    const H_0_1 = dhMatrix(0, 0, l1, q1);
    const H_1_2 = dhMatrix(Math.PI / 2, 0, 0, q2 + Math.PI / 2);
    const H_2_3 = dhMatrix(0, l2, 0, -Math.PI / 2);
    const H_3_4 = dhMatrix(0, l3, 0, q3);
    const H_4_5 = dhMatrix(0, l4, 0, 0);

    // Matrices acumulativas
    const H_0_2 = new THREE.Matrix4().multiplyMatrices(H_0_1, H_1_2);
    const H_0_3 = new THREE.Matrix4().multiplyMatrices(H_0_2, H_2_3);
    const H_0_4 = new THREE.Matrix4().multiplyMatrices(H_0_3, H_3_4);
    const H_0_5 = new THREE.Matrix4().multiplyMatrices(H_0_4, H_4_5);

    // Aplicar las transformaciones a los eslabones del robot
    const base = robot.getObjectByName('Base');
    const eslabon0 = robot.getObjectByName('Eslabon0');
    const eslabon1 = robot.getObjectByName('Eslabon1');
    const eslabon2 = robot.getObjectByName('Eslabon2');
    const origen=eslabon0.position.clone();// nuevo origen del robot

    if (eslabon0) applyDHTransform(eslabon0, H_0_1,origen);
    if (eslabon1) applyDHTransform(eslabon1, H_0_2,origen);
    if (eslabon2) applyDHTransform(eslabon2, H_0_4,origen);

    // Asegurarse de que se aplique la transformación
    robot.updateMatrixWorld(true);
}
// Función de animación
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}
// Inicializar la escena y el robot al montar el componente
onMounted(() => {
    target.value.appendChild(renderer.domElement);
    animate();
    updateRobot();
});
// Observadores para los sliders
watch([q1Angle, q2Angle, q3Angle], () => {
    updateRobot();  // Llamar a la función para actualizar el robot

});

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
import axios from "axios";

const ip = "192.168.4.1"; // IP del robot
const responseText = ref("");
const command = ref(""); // Entrada para el usuario
const updateCommand = () => {
  command.value = JSON.stringify({
    T: 112,
    b: q1Angle.value, // Aquí se usa el valor de q1Angle
    s: q2Angle.value*(-1), // Puedes agregar más valores según lo necesites/////calibrada
    e: q3Angle.value
  }, null, 2);
  console.log("JSON actualizado:", command.value); // Ver en consola
};
// Observar cambios en los sliders y actualizar `command`
watch([q1Angle, q2Angle, q3Angle], updateCommand, { immediate: true });

// Función para enviar comandos JSON al robot
const sendCommand = async () => {
  if (!command.value) return; // Evita enviar si el campo está vacío

  const url = `http://${ip}/js?json=${encodeURIComponent(command.value)}`;

  try {
    const response = await axios.get(url);
    responseText.value = response.data; // Muestra la respuesta del robot
  } catch (error) {
    responseText.value = "Error en la comunicación.";
    console.error(error);
  }
};

</script>
<template>
    <AppLayout title="Dashboard">
        <nav class="bg-[#15803d] shadow-lg p-4 flex justify-between items-center">
            <div class="text-xl text-gray-100">Proyecto Brazo Dragón</div>
            <div class="flex items-center space-x-4">
                <button id="openButton" @click.left="()=>abrirOverlay()" class="text-gray-300 hover:text-gray-100 focus:outline-none">
                    <!-- Hamburger Icon (SVG) -->
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"
                        xmlns="http://www.w3.org/2000/svg">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16m-7 6h7"></path>
                    </svg>
                </button>
            </div>
        </nav>
        <div class="flex-grow h-full flex items-center justify-center bg-white">
            <div id="scene-container" class="w-full h-full">
                <div ref="target"></div>
            </div>
        </div>
        <div id="overlayPanel" class="fixed top-0 right-0 w-1/4 h-full overflow-y-auto bg-white shadow-lg transform transition-transform duration-300">
            <button id="closeButton" @click.left="()=>closeOverlay()"
                class="text-xl absolute top-4 right-4 text-gray-500 hover:text-gray-700">×
                
            </button>
            <form class="bg-white rounded px-8 pt-6 pb-8 mb-4">
                <h2 class="text-2xl font-bold mb-6 text-center">Proyecto Brazo de Dragon</h2>
                <!--
                    <div class="mb-4">
                        <label for="nombre" class="block text-gray-700 text-sm font-bold mb-2">Nombre:</label>
                        <input type="text" id="nombre" name="nombre" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" required>
                    </div>
                    <div class="mb-4">
                        <label for="email" class="block text-gray-700 text-sm font-bold mb-2">Correo Electrónico:</label>
                        <input type="email" id="email" name="email" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" required>
                    </div>
                    
                    <div class="flex items-center justify-between">
                        <button type="submit" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline">Registrar</button>
                    </div>-->
                    <div class="mb-4">
                        <div class="control-panel">
                            <label>
                                Articulación 1 (q1):
                                <input type="range" v-model="q1Angle" min="-180" max="180" step="1" />
                                {{ q1Angle }}°
                            </label>
                            <br />
                            <label>
                                Articulación 2 (q2):
                                <input type="range" v-model="q2Angle" min="-90" max="90" step="1" />
                                {{ q2Angle }}°
                            </label>
                            <br />
                            <label>
                                Articulación 3 (q3):
                                <input type="range" v-model="q3Angle" min="-90" max="90" step="1" />
                                {{ q3Angle }}°
                            </label>
                        </div>
                    </div>
                <!---->
                    <div class="mb-4">
                        <h1>Control del Robot</h1>

                        <label>Comando JSON:</label>
                        <textarea v-model="command" type="text" readonly />

                        <button @click="sendCommand">Enviar Comando</button>

                        <h3>Respuesta del Robot:</h3>
                        <pre>{{ responseText }}</pre>
                    </div>
                    
            </form>
        </div>    
    </AppLayout>  
</template>
<style>
.control-panel {
    
    top: 10px;
    left: 10px;
    background-color: rgba(255, 255, 255, 0.8);
    padding: 10px;
    border-radius: 4px;
}
</style>

