<script setup>
///////////////////////////////LIBRERIAS//////////////////////////////////////////////////////////////////
// Importar math.js
import { pi , matrix , cos , sin , multiply, re, abs } from "mathjs";
//////////////////////////////////////////////////////////////////////////////
import AppLayout from '@/Layouts/AppLayout.vue';
import Welcome from '@/Components/Welcome.vue';
import { ref, onMounted } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { CCDIKSolver, CCDIKHelper } from 'three/addons/animation/CCDIKSolver.js';
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
camera.position.z = 0.5;
//////////preparacion de modelo 3d ////////////////////////////////////////////////////////////////////////
const renderer = new THREE.WebGLRenderer();
renderer.setSize(1350,520);//520
document.body.appendChild(renderer.domElement);
//////////////////////////////  Montage  /////////////////////////////////////////////////////////////////
onMounted(() => {
  target.value.appendChild(renderer.domElement);
  ///////////////// montar modelo 3D /////////////////////////////////////////////////////////////////////
  const loader=new GLTFLoader();
  const articulaciones = {}; // Guardará las referencias de las articulaciones
  loader.load('models/glb/brazo_v3.glb',(gltf)=>{
    const model=gltf.scene;
    scene.add(model);
    const axesHelper = new THREE.AxesHelper(5); // Tamaño del eje (5 unidades)
    scene.add(axesHelper);
    ////////////////rotar esenario
    camera.lookAt(scene.position)
    // Buscar las articulaciones
    model.traverse((node) => {
        //console.log("Nodo encontrado:", node.name); // Verifica la estructura completa
        if(node.name=="Base"||node.name=="Eslabon0"||node.name=="Eslabon1"||node.name=="Eslabon2"){
            console.log("Exito Encontrado: ",node.name,node);
            articulaciones[node.name] = node;
        }
        });
        if("Base"){
            articulaciones["Base"].position.x=O_0_num[1]; 
            articulaciones["Base"].position.y=O_0_num[2];
            articulaciones["Base"].position.z=O_0_num[0];
        }
        if("Eslabon0"){
            articulaciones["Eslabon0"].position.x=O_1_num[1];
            articulaciones["Eslabon0"].rotation.y=O_1_num[2];
            articulaciones["Eslabon0"].position.z=O_1_num[0];
        }
        if("Eslabon1"){
            articulaciones["Eslabon1"].position.x=O_2_num[1];
            articulaciones["Eslabon1"].rotation.y=O_2_num[2];
            articulaciones["Eslabon1"].rotation.z=O_2_num[0];
        }
        
    console.log("listo el modelo para trabajar");
  });
}, undefined, function ( error ) {
    console.error( error );
});
// Animación
// Inicializar controles de órbita después de la cámara y el renderizador
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true; // Habilita amortiguación para un movimiento más fluido
controls.dampingFactor = 0.05; // Ajusta la velocidad de amortiguación
controls.rotateSpeed = 0.5; // Ajusta la sensibilidad de rotación
const velocidad= 0.01;
function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
    if (controls) {
        controls.update(); 
    }
}
animate();
/////////////////////////////////////////////////////////////////////////////////////////////////////////////
//////////////////////////// Parte numerica Denavit Hartemberg /////////////////////////////////////////////
// Parámetros numéricos
const parametros = {
  l1: 0.12,
  l2: 0.24,
  l3: 0.03,
  l4: 0.28,
  q1: 180 * (pi / 180),
  q2: 45 * (pi / 180),
  q3: -55 * (pi / 180),
};
// Definir variables numéricas directamente
const q1 = parametros.q1;
const q2 = parametros.q2;
const q3 = parametros.q3;
const l1 = parametros.l1;
const l2 = parametros.l2;
const l3 = parametros.l3;
const l4 = parametros.l4;
// Crear una matriz de transformación usando los parámetros DH
function dhMatrix(alpha_i_1, a_i_1, d_i, theta_i) {
  return matrix([
    [redondear(cos(theta_i)), -redondear(sin(theta_i)), 0, a_i_1],
    [redondear(sin(theta_i)) * redondear(cos(alpha_i_1)), redondear(cos(theta_i)) * redondear(cos(alpha_i_1)), -redondear(sin(alpha_i_1)), -redondear(sin(alpha_i_1)) * d_i,],
    [redondear(sin(theta_i)) * redondear(sin(alpha_i_1)), redondear(cos(theta_i)) * redondear(sin(alpha_i_1)), redondear(cos(alpha_i_1)), redondear(cos(alpha_i_1)) * d_i,],
    [0, 0, 0, 1],
  ]);
}
//funcion para redondeo de numeros muy bajos 
function redondear(num){
  const tolerancia=1e-10;
  if(abs(num)<tolerancia){
    return 0;
  }else{
    return num;
  }
}
// Matrices de transformación
const H_0_1 = dhMatrix(0, 0, l1, q1);
//console.log("transformacion",H_0_1)// prueba  de la matriz H_0_1
const H_1_2 = dhMatrix((pi / 2), 0, 0, q2+ (pi/2));
const H_2_3 = dhMatrix(0, l2, 0, -(pi / 2));
const H_3_4 = dhMatrix(0, l3, 0, q3 );
const H_4_5 = dhMatrix(0, l4, 0, 0);
// Matrices acumulativas
const H_0_2 = multiply(H_0_1, H_1_2);
const H_0_3 = multiply(H_0_2, H_2_3);
const H_0_4 = multiply(H_0_3, H_3_4);
const H_0_5 = multiply(H_0_4, H_4_5);
// Orígenes de los sistemas
const O_0 = matrix([0, 0, 0, 1]);
const O_1 = multiply(H_0_1, O_0);
const O_2 = multiply(H_0_2, O_0);
const O_3 = multiply(H_0_3, O_0);
const O_4 = multiply(H_0_4, O_0);
const O_5 = multiply(H_0_5, O_0);
////////////////////////////////////////////////Duda
function convertirANumero(matrizSimbolica) {
  const arrayNumerico = matrizSimbolica.toArray(); // Convertir a array
  // Eliminar la última fila o el último elemento (correspondiente al valor homogéneo)
  return arrayNumerico.slice(0, 3).map((fila) =>
    Array.isArray(fila) ? fila.slice(0, 3) : fila
  );
}
//////////////////////////////////////////////////
// Orígenes numéricos
const O_0_num = convertirANumero(O_0);
const O_1_num = convertirANumero(O_1);
const O_2_num = convertirANumero(O_2);
const O_3_num = convertirANumero(O_3);
const O_4_num = convertirANumero(O_4);
const O_5_num = convertirANumero(O_5);
// Imprimir resultados
console.log('O_0_num:', O_0_num);
console.log('O_1_num:', O_1_num);
console.log('O_2_num:', O_2_num);
console.log('O_3_num:', O_3_num);
console.log('O_4_num:', O_4_num);
console.log('O_5_num:', O_5_num);
////////////////////////////////////////////////////////////////////////////

</script>

<template>
    <AppLayout title="Dashboard">
        <nav class="bg-[#15803d] shadow-lg p-4 flex justify-between items-center">
            <div class="text-xl text-gray-100">Proyecto Brazo de Dragon</div>
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
                <div class="mb-4">
                        <label for="nombre" class="block text-gray-700 text-sm font-bold mb-2">Nombre:</label>
                        <input type="text" id="nombre" name="nombre" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" required>
                    </div>
                    <div class="mb-4">
                        <label for="email" class="block text-gray-700 text-sm font-bold mb-2">Correo Electrónico:</label>
                        <input type="email" id="email" name="email" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" required>
                    </div>
                    <div class="mb-4">
                        <label for="password" class="block text-gray-700 text-sm font-bold mb-2">Contraseña:</label>
                        <input type="password" id="password" name="password" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" required>
                    </div>
                    <div class="mb-6">
                        <label for="password_confirmation" class="block text-gray-700 text-sm font-bold mb-2">Confirmar Contraseña:</label>
                        <input type="password" id="password_confirmation" name="password_confirmation" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" required>
                    </div>
                    <div class="flex items-center justify-between">
                        <button type="submit" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline">Registrar</button>
                    </div>
                    <div class="mb-4">
                        <!--<label for="nombre" class="block text-gray-700 text-sm font-bold mb-2">Eslabon 0:</label>
                        <input id="eslabon_0" type="range" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" v-model="inputValue" min="0" max="360" step="10" required>
                        <p id="value_0">Value:{{ inputValue }}</p>-->
                        <div v-for="(slider, index) in sliders" :key="index" class="slider">
                            <label :for="`slider_${index}`">Slider {{ index + 1 }}:</label>
                            <input
                                type="range"
                                :id="`slider_${index}`"
                                v-model="slider.value"
                                :min="slider.min"
                                :max="slider.max"
                                :step="slider.step"
                            />
                            <p>Valor: {{ slider.value }}</p>
                        </div>
                    </div>
                    <!--
                    <div class="mb-4">
                        <label for="email" class="block text-gray-700 text-sm font-bold mb-2">Eslabon 1:</label>
                        <input type="range" id="eslabon_1"  class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" min="0" max="180" step="10" required>
                        <p>Value: <output id="value_1"></output></p>
                    </div>
                    <div class="mb-4">
                        <label for="password" class="block text-gray-700 text-sm font-bold mb-2">Eslabon 2:</label>
                        <input type="range" id="eslabon_2"  class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" min="0" max="180" step="10" required>
                        <p>Value: <output id="value_2"></output></p>
                    </div>
                    <div class="mb-6">
                        <label for="password_confirmation" class="block text-gray-700 text-sm font-bold mb-2">Eslabon 3:</label>
                        <input type="range" id="eslabon_3"  class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline" min="0" max="180" step="10" required>
                        <p>Value: <output id="value_3"></output></p>
                    </div>
                    <div class="flex items-center justify-between">
                        <button type="submit" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline">Registrar</button>
                    </div>-->
            </form>
        </div>    
    </AppLayout>  
</template>
<script>/*
function closeOverlay() {
    const overlay = document.getElementById('overlayPanel');
    overlay.style.transform = 'translateX(100%)'; // Oculta el panel deslizándolo fuera de la vista
}
function abrirOverlay(){
    const overlay = document.getElementById('overlayPanel');
    overlay.style.transform = 'translateX(0%)'; // aparece el panel deslizándolo a la vista
}*/
export default {
  data() {
    return {
      sliders: [
        { value: 0, min: 0, max: 360, step: 10 }, // Slider 1
        { value: 90, min: 0, max: 180, step: 10 },  // Slider 2
        { value: 90, min: 0, max: 180, step: 10 }, // Slider 3
        { value: 90, min: 0, max: 180, step: 10 }, // Slider 4
      ],
    };
  },
};
</script>

