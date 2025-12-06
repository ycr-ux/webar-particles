# webar-particles
<!doctype html><html lang="zh"><head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1"/>
<title>WebAR GPU Fluid Particles</title>
<style>
html,body{margin:0;overflow:hidden;background:#000;font-family:system-ui}
#ui{position:fixed;top:10px;left:10px;color:#9ff;font-size:12px;z-index:10;background:rgba(0,0,0,.4);padding:10px;border-radius:10px}
#conf{margin-top:6px}
video{display:none}
</style>
</head>
<body>
<div id="ui">
状态：<span id="state">初始化</span><br>
手势置信度：<span id="conf">0</span>
</div>
<video id="video" autoplay playsinline></video>

<script type="module">
import * as THREE from "https://unpkg.com/three@0.152.2/build/three.module.js";
import {GPUComputationRenderer} from "https://unpkg.com/three@0.152.2/examples/jsm/misc/GPUComputationRenderer.js";
import {Hands} from "https://unpkg.com/@mediapipe/hands@0.5.3?module";
import {Camera} from "https://unpkg.com/@mediapipe/camera_utils@0.5.3?module";

/* ------------------ 基础 ------------------ */
const W = 128, H = 128, COUNT = W * H;
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(45, innerWidth/innerHeight, .1, 500);
camera.position.z = 80;

const renderer = new THREE.WebGLRenderer({antialias:true, alpha:true});
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(devicePixelRatio);
document.body.appendChild(renderer.domElement);

/* ----------- 摄像头 AR 背景 ----------- */
const video = document.getElementById("video");
const videoTex = new THREE.VideoTexture(video);
scene.background = videoTex;

/* ------------------ GPU 粒子 ------------------ */
const gpu = new GPUComputationRenderer(W, H, renderer);

const dtPos = gpu.createTexture();
const dtVel = gpu.createTexture();

for (let i = 0; i < dtPos.image.data.length; i += 4) {
  dtPos.image.data[i] = (Math.random() - .5) * 40;
  dtPos.image.data[i+1] = (Math.random() - .5) * 40;
  dtPos.image.data[i+2] = (Math.random() - .5) * 40;
  dtPos.image.data[i+3] = 1;
}

const velVar = gpu.addVariable("textureVelocity", `
uniform float explode;
void main(){
  vec2 uv=gl_FragCoord.xy/${W}.;
  vec4 vel = texture2D(textureVelocity, uv);
  vec4 pos = texture2D(texturePosition, uv);
  vel.xyz *= .985;
  if(explode>0.) vel.xyz += normalize(pos.xyz)*explode;
  gl_FragColor = vel;
}`, dtVel);

const posVar = gpu.addVariable("texturePosition", `
void main(){
  vec2 uv=gl_FragCoord.xy/${W}.;
  vec4 pos = texture2D(texturePosition, uv);
  vec4 vel = texture2D(textureVelocity, uv);
  pos.xyz += vel.xyz;
  gl_FragColor = pos;
}`, dtPos);

gpu.setVariableDependencies(velVar,[velVar,posVar]);
gpu.setVariableDependencies(posVar,[velVar,posVar]);

velVar.material.uniforms.explode={value:0};

gpu.init();

/* ------------------ 显示粒子 ------------------ */
const geo = new THREE.BufferGeometry();
const pos = new Float32Array(COUNT*3);
for(let i=0;i<COUNT;i++){
  pos[i*3]=(i%W)/W;
  pos[i*3+1]=~~(i/W)/H;
}
geo.setAttribute("position",new THREE.BufferAttribute(pos,3));

const mat = new THREE.ShaderMaterial({
  transparent:true, depthWrite:false, blending:THREE.AdditiveBlending,
  uniforms:{posTex:{value:null}},
  vertexShader:`
  uniform sampler2D posTex;
  void main(){
    vec3 p = texture2D(posTex,position.xy).xyz;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(p,1.);
    gl_PointSize = 2.0;
  }`,
  fragmentShader:`void main(){gl_FragColor=vec4(0.,1.,.8,1.);}
`
});
const particles = new THREE.Points(geo,mat);
scene.add(particles);

/* ------------------ MediaPipe ------------------ */
let gestureHistory=[];
const hands = new Hands({
  locateFile:f=>`https://unpkg.com/@mediapipe/hands@0.5.3/${f}`
});
hands.setOptions({maxNumHands:1});
hands.onResults(r=>{
  if(!r.multiHandLandmarks) return;
  const lm=r.multiHandLandmarks[0];
  const ext=i=>lm[i].y<lm[i-2].y;
  const g = ext(8)&&ext(12)&&ext(16)&&ext(20)?"open":
            ext(8)&&ext(12)?"scissors":
            ext(4)?"thumb":"fist";
  gestureHistory.push(g);
  if(gestureHistory.length>15) gestureHistory.shift();
});
new Camera(video,{onFrame:()=>hands.send({image:video})}).start();

/* ------------------ 动画 ------------------ */
function animate(){
  gpu.compute();
  mat.uniforms.posTex.value = gpu.getCurrentRenderTarget(posVar).texture;
  renderer.render(scene,camera);
  requestAnimationFrame(animate);
}
animate();
</script>
</body></html>
