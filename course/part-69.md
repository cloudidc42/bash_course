# Part 69: AR/VR Platform, Real-Time Gaming Infrastructure และ WebRTC at Scale

## ขั้นตอนที่ 623: AR/VR Platform Development

### `ar-vr-platform.sh`

```bash
#!/bin/bash
# AR/VR Platform: WebXR, Three.js, Spatial Computing

cat > webxr-app/index.html << 'HTML'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>WebXR Payment Visualization</title>
  <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/webxr/VRButton.js"></script>
  <style>
    body { margin: 0; background: #000; overflow: hidden; }
    canvas { display: block; }
    #info { position: absolute; top: 10px; left: 10px; color: #fff; font-family: monospace; }
  </style>
</head>
<body>
  <div id="info">WebXR Payment Analytics — Move head to navigate</div>
  <script type="module">
    import * as THREE from 'three';
    import { VRButton } from 'three/addons/webxr/VRButton.js';
    import { XRControllerModelFactory } from 'three/addons/webxr/XRControllerModelFactory.js';
    import { Text } from 'troika-three-text';

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.xr.enabled = true;
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    document.body.appendChild(renderer.domElement);
    document.body.appendChild(VRButton.createButton(renderer));

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0a1a);
    scene.fog = new THREE.FogExp2(0x0a0a1a, 0.05);

    const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 100);
    camera.position.set(0, 1.6, 3);

    // Spatial audio setup
    const audioListener = new THREE.AudioListener();
    camera.add(audioListener);

    // Environment lighting
    const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
    scene.add(ambientLight);

    const dirLight = new THREE.DirectionalLight(0x6699ff, 1);
    dirLight.position.set(5, 10, 5);
    dirLight.castShadow = true;
    dirLight.shadow.mapSize.width = 2048;
    dirLight.shadow.mapSize.height = 2048;
    scene.add(dirLight);

    // Holographic grid floor
    const gridHelper = new THREE.GridHelper(20, 20, 0x1a1aff, 0x0d0d80);
    scene.add(gridHelper);

    // Payment data visualization — 3D bar chart
    const payments = [
      { region: 'SEA', amount: 4520000, color: 0x00ff88 },
      { region: 'US',  amount: 8930000, color: 0x0088ff },
      { region: 'EU',  amount: 6210000, color: 0xff8800 },
      { region: 'JP',  amount: 2340000, color: 0xff0088 },
      { region: 'AU',  amount: 1870000, color: 0x88ff00 },
    ];

    const maxAmount = Math.max(...payments.map(p => p.amount));

    payments.forEach((payment, i) => {
      const height = (payment.amount / maxAmount) * 3;
      const geometry = new THREE.BoxGeometry(0.4, height, 0.4);
      const material = new THREE.MeshPhongMaterial({
        color: payment.color,
        emissive: payment.color,
        emissiveIntensity: 0.3,
        transparent: true,
        opacity: 0.85,
      });
      const bar = new THREE.Mesh(geometry, material);
      bar.position.set((i - 2) * 0.8, height / 2, -1);
      bar.castShadow = true;
      bar.userData = { payment, originalY: height / 2 };
      scene.add(bar);

      // Point light inside each bar for glow effect
      const pointLight = new THREE.PointLight(payment.color, 0.5, 2);
      pointLight.position.copy(bar.position);
      scene.add(pointLight);

      // Region label using troika-three-text
      const label = new Text();
      label.text = `${payment.region}\n$${(payment.amount / 1e6).toFixed(1)}M`;
      label.fontSize = 0.1;
      label.color = payment.color;
      label.anchorX = 'center';
      label.anchorY = 'top';
      label.position.set((i - 2) * 0.8, height + 0.1, -1);
      label.sync();
      scene.add(label);
    });

    // Spatial transaction stream (particle system)
    const particleCount = 500;
    const positions = new Float32Array(particleCount * 3);
    const colors = new Float32Array(particleCount * 3);
    const velocities = [];

    for (let i = 0; i < particleCount; i++) {
      positions[i * 3]     = (Math.random() - 0.5) * 10;
      positions[i * 3 + 1] = Math.random() * 4;
      positions[i * 3 + 2] = (Math.random() - 0.5) * 10;
      colors[i * 3]     = Math.random();
      colors[i * 3 + 1] = Math.random() * 0.5 + 0.5;
      colors[i * 3 + 2] = 1;
      velocities.push({
        x: (Math.random() - 0.5) * 0.01,
        y: Math.random() * 0.02 + 0.01,
        z: (Math.random() - 0.5) * 0.01,
      });
    }

    const particleGeometry = new THREE.BufferGeometry();
    particleGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    particleGeometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

    const particleMaterial = new THREE.PointsMaterial({
      size: 0.03,
      vertexColors: true,
      transparent: true,
      opacity: 0.8,
      blending: THREE.AdditiveBlending,
    });

    const particles = new THREE.Points(particleGeometry, particleMaterial);
    scene.add(particles);

    // XR Controller setup
    const controllerModelFactory = new XRControllerModelFactory();

    function setupController(index) {
      const controller = renderer.xr.getController(index);
      controller.addEventListener('selectstart', onSelectStart);
      controller.addEventListener('selectend', onSelectEnd);
      scene.add(controller);

      const grip = renderer.xr.getControllerGrip(index);
      grip.add(controllerModelFactory.createControllerModel(grip));
      scene.add(grip);

      // Controller ray
      const rayGeometry = new THREE.BufferGeometry().setFromPoints([
        new THREE.Vector3(0, 0, 0),
        new THREE.Vector3(0, 0, -5),
      ]);
      const rayMaterial = new THREE.LineBasicMaterial({
        color: 0x00ffff,
        transparent: true,
        opacity: 0.5,
      });
      controller.add(new THREE.Line(rayGeometry, rayMaterial));

      return controller;
    }

    const controller1 = setupController(0);
    const controller2 = setupController(1);

    let selectedObject = null;

    function onSelectStart(event) {
      const controller = event.target;
      const intersections = getIntersections(controller);
      if (intersections.length > 0) {
        selectedObject = intersections[0].object;
        selectedObject.material.emissiveIntensity = 0.8;
        // Haptic feedback
        const session = renderer.xr.getSession();
        if (session) {
          const source = session.inputSources[controller === controller1 ? 0 : 1];
          if (source?.gamepad?.hapticActuators?.length > 0) {
            source.gamepad.hapticActuators[0].pulse(0.5, 100);
          }
        }
      }
    }

    function onSelectEnd() {
      if (selectedObject) {
        selectedObject.material.emissiveIntensity = 0.3;
        selectedObject = null;
      }
    }

    const raycaster = new THREE.Raycaster();
    const tempMatrix = new THREE.Matrix4();

    function getIntersections(controller) {
      tempMatrix.identity().extractRotation(controller.matrixWorld);
      raycaster.ray.origin.setFromMatrixPosition(controller.matrixWorld);
      raycaster.ray.direction.set(0, 0, -1).applyMatrix4(tempMatrix);
      return raycaster.intersectObjects(scene.children, false);
    }

    // Animation loop
    const clock = new THREE.Clock();
    let time = 0;

    renderer.setAnimationLoop(() => {
      const delta = clock.getDelta();
      time += delta;

      // Animate particles
      const pos = particleGeometry.attributes.position.array;
      for (let i = 0; i < particleCount; i++) {
        pos[i * 3]     += velocities[i].x;
        pos[i * 3 + 1] += velocities[i].y;
        pos[i * 3 + 2] += velocities[i].z;

        if (pos[i * 3 + 1] > 5) {
          pos[i * 3]     = (Math.random() - 0.5) * 10;
          pos[i * 3 + 1] = 0;
          pos[i * 3 + 2] = (Math.random() - 0.5) * 10;
        }
      }
      particleGeometry.attributes.position.needsUpdate = true;

      // Animate bars (breathing effect)
      scene.children.forEach(child => {
        if (child.userData?.payment) {
          child.position.y = child.userData.originalY + Math.sin(time * 2 + child.position.x) * 0.02;
        }
      });

      renderer.render(scene, camera);
    });

    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
  </script>
</body>
</html>
HTML

# Spatial Audio API implementation
cat > webxr-app/spatial-audio.js << 'JS'
export class SpatialAudioManager {
  constructor(audioListener) {
    this.listener = audioListener;
    this.sounds = new Map();
    this.audioContext = audioListener.context;
  }

  async loadAmbientSound(url, loop = true) {
    const sound = new THREE.Audio(this.listener);
    const audioLoader = new THREE.AudioLoader();
    
    return new Promise((resolve, reject) => {
      audioLoader.load(url, (buffer) => {
        sound.setBuffer(buffer);
        sound.setLoop(loop);
        sound.setVolume(0.3);
        resolve(sound);
      }, undefined, reject);
    });
  }

  createPositionalSound(position, url) {
    const sound = new THREE.PositionalAudio(this.listener);
    const audioLoader = new THREE.AudioLoader();
    
    audioLoader.load(url, (buffer) => {
      sound.setBuffer(buffer);
      sound.setRefDistance(1);
      sound.setRolloffFactor(2);
      sound.setDistanceModel('exponential');
      sound.setMaxDistance(10);
      sound.position.copy(position);
    });

    return sound;
  }

  // Sonify data — pitch maps to transaction amount
  sonifyTransaction(amount, maxAmount) {
    const oscillator = this.audioContext.createOscillator();
    const gainNode = this.audioContext.createGain();
    
    // Map amount to frequency (200Hz - 2000Hz)
    const frequency = 200 + (amount / maxAmount) * 1800;
    oscillator.frequency.setValueAtTime(frequency, this.audioContext.currentTime);
    oscillator.type = 'sine';
    
    gainNode.gain.setValueAtTime(0.1, this.audioContext.currentTime);
    gainNode.gain.exponentialRampToValueAtTime(0.001, this.audioContext.currentTime + 0.3);
    
    oscillator.connect(gainNode);
    gainNode.connect(this.audioContext.destination);
    
    oscillator.start();
    oscillator.stop(this.audioContext.currentTime + 0.3);
  }
}
JS

# AR.js integration for mobile AR
cat > webxr-app/ar-overlay.html << 'HTML'
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>AR Payment Overlay</title>
  <script src="https://aframe.io/releases/1.4.0/aframe.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/ar.js@2.3.4/aframe/build/aframe-ar.min.js"></script>
</head>
<body style="margin:0; overflow:hidden;">
  <a-scene
    embedded
    arjs="sourceType: webcam; debugUIEnabled: false; detectionMode: mono_and_matrix; matrixCodeType: 3x3;"
    renderer="logarithmicDepthBuffer: true; colorManagement: true;"
    vr-mode-ui="enabled: false"
  >
    <!-- Hiro marker triggers payment dashboard -->
    <a-marker type="hiro" id="payment-marker">
      <a-entity id="payment-dashboard" position="0 0.5 0">
        <!-- Holographic panel -->
        <a-plane
          color="#0a1628"
          opacity="0.85"
          width="2"
          height="1.2"
          position="0 0 0"
          rotation="-90 0 0"
        ></a-plane>

        <!-- Real-time metrics text -->
        <a-text
          id="revenue-text"
          value="Revenue: $4.52M"
          color="#00ff88"
          position="-0.8 0.01 0.4"
          rotation="-90 0 0"
          scale="0.8 0.8 0.8"
        ></a-text>

        <a-text
          id="txn-text"
          value="TXN/s: 1,247"
          color="#0088ff"
          position="-0.8 0.01 0.2"
          rotation="-90 0 0"
          scale="0.8 0.8 0.8"
        ></a-text>

        <a-text
          id="error-text"
          value="Error Rate: 0.02%"
          color="#ff8800"
          position="-0.8 0.01 0"
          rotation="-90 0 0"
          scale="0.8 0.8 0.8"
        ></a-text>

        <!-- Animated pulse ring for alerts -->
        <a-ring
          id="alert-ring"
          color="#ff0000"
          radius-inner="0.1"
          radius-outer="0.12"
          position="0.8 0.01 0.3"
          rotation="-90 0 0"
          animation="property: scale; to: 1.5 1.5 1.5; dur: 1000; easing: easeInOutQuad; loop: true; dir: alternate"
          visible="false"
        ></a-ring>
      </a-entity>
    </a-marker>

    <!-- Matrix barcode marker for specific payment details -->
    <a-marker type="barcode" value="6">
      <a-entity position="0 0.3 0">
        <a-box color="#1a1aff" opacity="0.8" depth="0.3" height="0.3" width="0.3"
          animation="property: rotation; to: 0 360 0; dur: 3000; loop: true; easing: linear">
        </a-box>
        <a-text value="Payment #TXN-6" color="#fff" position="0 0.3 0"
          align="center" scale="0.5 0.5 0.5">
        </a-text>
      </a-entity>
    </a-marker>

    <a-camera-static></a-camera-static>
  </a-scene>

  <script>
    // WebSocket for real-time data updates
    const ws = new WebSocket('wss://api.payment.internal/ws/metrics');
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      
      document.getElementById('revenue-text').setAttribute('value', 
        `Revenue: $${(data.revenue / 1e6).toFixed(2)}M`);
      document.getElementById('txn-text').setAttribute('value',
        `TXN/s: ${data.tps.toLocaleString()}`);
      document.getElementById('error-text').setAttribute('value',
        `Error Rate: ${(data.error_rate * 100).toFixed(3)}%`);
      
      // Show alert ring if error rate exceeds threshold
      const ring = document.getElementById('alert-ring');
      ring.setAttribute('visible', data.error_rate > 0.01);
    };
  </script>
</body>
</html>
HTML

echo "AR/VR Platform setup complete"
```

---

## ขั้นตอนที่ 624: Real-Time Gaming Infrastructure with Agones

### `gaming-infrastructure.sh`

```bash
#!/bin/bash
# Real-Time Gaming Infrastructure: Agones, Matchmaking, Game Server Fleet

# Install Agones
helm repo add agones https://agones.dev/chart/agones
helm repo update

helm install agones agones/agones \
  --namespace agones-system \
  --create-namespace \
  --set agones.featureGates="PlayerTracking=true,NodeExternalDNS=true" \
  --set gameservers.namespaces="{default,game-prod}" \
  --set controller.numWorkers=100 \
  --set controller.apiServerQPS=400 \
  --set controller.apiServerQPSBurst=500 \
  --set ping.replicas=3 \
  --set ping.http.expose=true \
  --set ping.udp.expose=true

# Game Server Fleet for Battle Royale
cat > fleet-battleroyale.yaml << 'EOF'
apiVersion: agones.dev/v1
kind: Fleet
metadata:
  name: battle-royale-fleet
  namespace: game-prod
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  scheduling: Packed
  template:
    metadata:
      labels:
        game-type: battle-royale
        version: v2.4.1
    spec:
      ports:
        - name: game
          containerPort: 7777
          protocol: UDP
        - name: relay
          containerPort: 7778
          protocol: UDP
      health:
        initialDelaySeconds: 30
        periodSeconds: 5
        failureThreshold: 3
      sdkServer:
        logLevel: Error
        grpcPort: 9357
        httpPort: 9358
      players:
        initialCapacity: 100
      template:
        spec:
          nodeSelector:
            agones.dev/role: gameserver
          tolerations:
            - key: "agones.dev/gameserver"
              operator: "Exists"
              effect: "NoSchedule"
          containers:
            - name: battle-royale
              image: gcr.io/myproject/battle-royale:v2.4.1
              imagePullPolicy: Always
              resources:
                requests:
                  cpu: "2"
                  memory: "4Gi"
                limits:
                  cpu: "4"
                  memory: "8Gi"
              env:
                - name: GAME_SERVER_MAX_PLAYERS
                  value: "100"
                - name: GAME_SERVER_TICK_RATE
                  value: "60"
                - name: GAME_SERVER_MAP
                  value: "island_01"
                - name: OTEL_EXPORTER_OTLP_ENDPOINT
                  value: "http://otel-collector:4317"
              volumeMounts:
                - name: game-data
                  mountPath: /data
          volumes:
            - name: game-data
              emptyDir:
                medium: Memory
                sizeLimit: "512Mi"
---
apiVersion: autoscaling.agones.dev/v1
kind: FleetAutoscaler
metadata:
  name: battle-royale-autoscaler
  namespace: game-prod
spec:
  fleetName: battle-royale-fleet
  policy:
    type: Buffer
    buffer:
      bufferSize: 5
      minReplicas: 5
      maxReplicas: 50
  sync:
    type: FixedInterval
    fixedInterval:
      seconds: 30
EOF
kubectl apply -f fleet-battleroyale.yaml

# Matchmaking Service
cat > matchmaker/matchmaker.py << 'PYTHON'
import asyncio
import json
import logging
import time
from dataclasses import dataclass, field
from collections import defaultdict
from typing import Optional
import aiohttp
import redis.asyncio as redis
from kubernetes import client as k8s_client, config as k8s_config

logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')
logger = logging.getLogger(__name__)


@dataclass
class Player:
    id: str
    skill_rating: float
    region: str
    game_mode: str
    queue_time: float = field(default_factory=time.time)
    connection_quality: str = "good"  # good, medium, poor


@dataclass
class Match:
    id: str
    players: list[Player]
    game_server_address: str
    game_server_port: int
    game_mode: str
    region: str
    created_at: float = field(default_factory=time.time)


class SkillBasedMatchmaker:
    SKILL_TOLERANCE_BASE = 100
    SKILL_TOLERANCE_GROWTH = 10    # per second waited
    MAX_WAIT_SECONDS = 120
    TEAM_SIZE = {"battle-royale": 100, "squad": 4, "duel": 2}
    REGION_LATENCY_THRESHOLD = 80  # ms

    def __init__(self, redis_url: str, agones_namespace: str):
        self.redis_url = redis_url
        self.agones_namespace = agones_namespace
        self.queues: dict[str, dict[str, list[Player]]] = defaultdict(
            lambda: defaultdict(list)
        )
        self._redis: Optional[redis.Redis] = None
        k8s_config.load_incluster_config()
        self._k8s_custom = k8s_client.CustomObjectsApi()

    async def start(self):
        self._redis = await redis.from_url(self.redis_url, encoding="utf-8", decode_responses=True)
        logger.info("Matchmaker started")
        await asyncio.gather(
            self._process_queue_loop(),
            self._cleanup_stale_players(),
            self._metrics_reporter(),
        )

    async def enqueue_player(self, player: Player):
        key = f"{player.game_mode}:{player.region}"
        self.queues[player.game_mode][player.region].append(player)
        await self._redis.hset(
            f"player:{player.id}",
            mapping={
                "skill": player.skill_rating,
                "region": player.region,
                "mode": player.game_mode,
                "queued_at": player.queue_time,
            }
        )
        await self._redis.expire(f"player:{player.id}", 300)
        logger.info(f"Player {player.id} queued for {key} (SR={player.skill_rating:.0f})")

    async def _process_queue_loop(self):
        while True:
            try:
                await self._run_matchmaking_pass()
            except Exception as e:
                logger.error(f"Matchmaking error: {e}")
            await asyncio.sleep(1)

    async def _run_matchmaking_pass(self):
        for game_mode, regions in self.queues.items():
            for region, players in regions.items():
                if not players:
                    continue

                required = self.TEAM_SIZE.get(game_mode, 4)
                formed_groups = self._form_skill_groups(players, required)

                for group in formed_groups:
                    server = await self._allocate_game_server(game_mode, region)
                    if server:
                        match = Match(
                            id=f"match-{int(time.time())}-{region}",
                            players=group,
                            game_server_address=server["address"],
                            game_server_port=server["port"],
                            game_mode=game_mode,
                            region=region,
                        )
                        await self._notify_players(match)
                        # Remove matched players from queue
                        for p in group:
                            players.remove(p)
                        logger.info(
                            f"Match {match.id} formed: {len(group)} players on {server['address']}"
                        )

    def _form_skill_groups(self, players: list[Player], required: int) -> list[list[Player]]:
        sorted_players = sorted(players, key=lambda p: p.skill_rating)
        groups = []

        while len(sorted_players) >= required:
            anchor = sorted_players[0]
            wait_time = time.time() - anchor.queue_time
            tolerance = self.SKILL_TOLERANCE_BASE + wait_time * self.SKILL_TOLERANCE_GROWTH

            # Collect players within skill tolerance
            group = [anchor]
            remaining = []
            for p in sorted_players[1:]:
                if len(group) < required and abs(p.skill_rating - anchor.skill_rating) <= tolerance:
                    group.append(p)
                else:
                    remaining.append(p)

            if len(group) == required:
                groups.append(group)
                sorted_players = remaining
            else:
                # Not enough players in tolerance — expand or give up for now
                if wait_time >= self.MAX_WAIT_SECONDS:
                    # Force-fill with closest available
                    sorted_players.sort(key=lambda p: abs(p.skill_rating - anchor.skill_rating))
                    group = sorted_players[:required]
                    groups.append(group)
                    sorted_players = sorted_players[required:]
                else:
                    break

        return groups

    async def _allocate_game_server(self, game_mode: str, region: str) -> Optional[dict]:
        try:
            # Create GameServerAllocation
            allocation = {
                "apiVersion": "allocation.agones.dev/v1",
                "kind": "GameServerAllocation",
                "spec": {
                    "selectors": [
                        {
                            "matchLabels": {
                                "game-type": game_mode.replace("_", "-"),
                                "agones.dev/fleet": f"{game_mode.replace('_', '-')}-fleet",
                            },
                            "matchExpressions": [
                                {
                                    "key": "agones.dev/sdk-node-name",
                                    "operator": "In",
                                    "values": [region],
                                }
                            ],
                        }
                    ],
                    "priorities": [
                        {
                            "type": "Counter",
                            "key": "players",
                            "order": "Ascending",
                        }
                    ],
                    "counters": {
                        "players": {
                            "action": "Increment",
                            "amount": self.TEAM_SIZE.get(game_mode, 4),
                        }
                    },
                    "metadata": {
                        "annotations": {
                            "region": region,
                            "game-mode": game_mode,
                        }
                    },
                },
            }

            result = self._k8s_custom.create_namespaced_custom_object(
                group="allocation.agones.dev",
                version="v1",
                namespace=self.agones_namespace,
                plural="gameserverallocations",
                body=allocation,
            )

            status = result.get("status", {})
            if status.get("state") == "Allocated":
                ports = status.get("ports", [{}])
                return {
                    "address": status.get("address"),
                    "node_name": status.get("nodeName"),
                    "port": next((p["port"] for p in ports if p.get("name") == "game"), 7777),
                }
        except Exception as e:
            logger.error(f"Allocation failed for {game_mode}/{region}: {e}")
        return None

    async def _notify_players(self, match: Match):
        payload = {
            "match_id": match.id,
            "server": f"{match.game_server_address}:{match.game_server_port}",
            "player_ids": [p.id for p in match.players],
        }
        await self._redis.publish(
            f"match_ready:{match.game_mode}",
            json.dumps(payload),
        )
        # Store match record
        await self._redis.setex(
            f"match:{match.id}",
            3600,
            json.dumps(payload),
        )

    async def _cleanup_stale_players(self):
        while True:
            await asyncio.sleep(30)
            now = time.time()
            for game_mode, regions in self.queues.items():
                for region, players in regions.items():
                    before = len(players)
                    regions[region] = [
                        p for p in players
                        if now - p.queue_time < self.MAX_WAIT_SECONDS * 2
                    ]
                    removed = before - len(regions[region])
                    if removed > 0:
                        logger.warning(f"Removed {removed} stale players from {game_mode}/{region}")

    async def _metrics_reporter(self):
        while True:
            await asyncio.sleep(15)
            total = sum(
                len(players)
                for regions in self.queues.values()
                for players in regions.values()
            )
            logger.info(f"Queue depth: {total} players total")
PYTHON

echo "Gaming infrastructure setup complete"
```

---

## ขั้นตอนที่ 625: WebRTC at Scale — SFU with LiveKit

### `webrtc-platform.sh`

```bash
#!/bin/bash
# WebRTC at Scale: LiveKit SFU, TURN/STUN, Recording Pipeline

# Deploy LiveKit SFU
helm repo add livekit https://helm.livekit.io
helm repo update

cat > livekit-values.yaml << 'EOF'
replicaCount: 3

image:
  repository: livekit/livekit-server
  tag: v1.6.0
  pullPolicy: IfNotPresent

livekit:
  port: 7880
  rtc:
    tcp_port: 7881
    port_range_start: 50000
    port_range_end: 60000
    use_external_ip: true
    enable_loopback_candidate: false
  redis:
    address: redis-cluster:6379
    username: ""
    password: ""
    use_tls: true
    db: 0
  turn:
    enabled: true
    domain: turn.payment.internal
    tls_port: 5349
    udp_port: 443
    external_tls: true
  webhook:
    api_key: "livekit_server_key"
    urls:
      - "https://api.payment.internal/livekit/webhook"
  log_level: info
  api_key: ${LIVEKIT_API_KEY}
  api_secret: ${LIVEKIT_API_SECRET}

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/upstream-hash-by: "$http_upgrade"
  hosts:
    - host: livekit.payment.internal
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: livekit-tls
      hosts:
        - livekit.payment.internal

resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "8"
    memory: "16Gi"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60

# Egress for recording/streaming
egress:
  enabled: true
  replicaCount: 2
  image:
    repository: livekit/egress
    tag: v1.8.0
  config:
    api_key: ${LIVEKIT_API_KEY}
    api_secret: ${LIVEKIT_API_SECRET}
    ws_url: wss://livekit.payment.internal
    health_port: 9090
    template_port: 7980
    s3:
      access_key: ${S3_ACCESS_KEY}
      secret: ${S3_SECRET_KEY}
      region: ap-southeast-1
      bucket: recordings-bucket
      endpoint: ""
    cpu_cost:
      room_composite_cpu_cost: 4
      track_composite_cpu_cost: 2
      track_cpu_cost: 1

# Ingress controller for WebRTC
ingress_controller:
  enabled: true
  replicaCount: 2
  image:
    repository: livekit/ingress
    tag: v1.3.0
EOF

helm install livekit livekit/livekit-server \
  --namespace livekit \
  --create-namespace \
  --values livekit-values.yaml

# TURN server (coturn) deployment
cat > coturn-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coturn
  namespace: livekit
spec:
  replicas: 2
  selector:
    matchLabels:
      app: coturn
  template:
    metadata:
      labels:
        app: coturn
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: coturn
          image: coturn/coturn:4.6.2
          args:
            - --log-file=stdout
            - --external-ip=$(EXTERNAL_IP)/$(INTERNAL_IP)
            - --listening-port=3478
            - --tls-listening-port=5349
            - --listening-ip=0.0.0.0
            - --relay-ip=$(INTERNAL_IP)
            - --min-port=49152
            - --max-port=65535
            - --lt-cred-mech
            - --no-auth
            - --use-auth-secret
            - --static-auth-secret=$(TURN_SECRET)
            - --realm=turn.payment.internal
            - --cert=/certs/tls.crt
            - --pkey=/certs/tls.key
            - --cipher-list=ECDHE-RSA-AES256-GCM-SHA512
            - --no-stdout-log
            - --syslog
            - --prometheus
            - --prometheus-port=9641
          env:
            - name: EXTERNAL_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            - name: INTERNAL_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: TURN_SECRET
              valueFrom:
                secretKeyRef:
                  name: turn-secret
                  key: secret
          ports:
            - containerPort: 3478
              protocol: UDP
            - containerPort: 3478
              protocol: TCP
            - containerPort: 5349
              protocol: TCP
          volumeMounts:
            - name: tls-certs
              mountPath: /certs
              readOnly: true
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"
      volumes:
        - name: tls-certs
          secret:
            secretName: coturn-tls
EOF
kubectl apply -f coturn-deployment.yaml

# LiveKit Room Management Service
cat > livekit-service/room_manager.py << 'PYTHON'
import asyncio
import logging
import time
from dataclasses import dataclass
from typing import Optional
from livekit import api, rtc
import jwt

logger = logging.getLogger(__name__)


@dataclass
class RoomConfig:
    name: str
    max_participants: int = 500
    empty_timeout: int = 300
    departure_timeout: int = 20
    enable_recording: bool = False
    enable_e2e_encryption: bool = False


class LiveKitRoomManager:
    def __init__(self, api_key: str, api_secret: str, ws_url: str):
        self.api_key = api_key
        self.api_secret = api_secret
        self.ws_url = ws_url
        self.lk_api = api.LiveKitAPI(
            url=ws_url,
            api_key=api_key,
            api_secret=api_secret,
        )

    def create_access_token(
        self,
        room_name: str,
        identity: str,
        name: str,
        can_publish: bool = True,
        can_subscribe: bool = True,
        can_publish_data: bool = True,
        metadata: str = "",
        ttl_seconds: int = 3600,
    ) -> str:
        at = api.AccessToken(self.api_key, self.api_secret)
        at.with_identity(identity)
        at.with_name(name)
        at.with_metadata(metadata)
        at.with_ttl(ttl_seconds)
        at.with_grants(
            api.VideoGrants(
                room_join=True,
                room=room_name,
                can_publish=can_publish,
                can_subscribe=can_subscribe,
                can_publish_data=can_publish_data,
                can_publish_sources=["camera", "microphone", "screen_share"],
            )
        )
        return at.to_jwt()

    async def create_room(self, config: RoomConfig) -> api.Room:
        room = await self.lk_api.room.create_room(
            api.CreateRoomRequest(
                name=config.name,
                empty_timeout=config.empty_timeout,
                max_participants=config.max_participants,
                departure_timeout=config.departure_timeout,
                metadata=f'{{"recording": {str(config.enable_recording).lower()}}}',
            )
        )
        logger.info(f"Room created: {config.name} (sid={room.sid})")
        return room

    async def start_egress_recording(
        self,
        room_name: str,
        output_path: str,
        video_width: int = 1920,
        video_height: int = 1080,
        video_bitrate: int = 4500,
        audio_bitrate: int = 128,
    ) -> str:
        request = api.RoomCompositeEgressRequest(
            room_name=room_name,
            layout="grid-dark",
            audio_only=False,
            video_only=False,
            custom_base_url="",
            file_outputs=[
                api.EncodedFileOutput(
                    file_type=api.EncodedFileType.MP4,
                    filepath=output_path,
                    s3=api.S3Upload(
                        access_key="${S3_ACCESS_KEY}",
                        secret="${S3_SECRET_KEY}",
                        region="ap-southeast-1",
                        bucket="recordings-bucket",
                    ),
                )
            ],
            options=api.EncodingOptions(
                width=video_width,
                height=video_height,
                depth=24,
                framerate=30,
                audio_codec=api.AudioCodec.OPUS,
                audio_bitrate=audio_bitrate,
                audio_frequency=48000,
                video_codec=api.VideoCodec.H264_HIGH,
                video_bitrate=video_bitrate,
            ),
        )

        egress = await self.lk_api.egress.start_room_composite_egress(request)
        logger.info(f"Recording started: egress_id={egress.egress_id}")
        return egress.egress_id

    async def start_hls_stream(self, room_name: str, hls_url: str) -> str:
        request = api.RoomCompositeEgressRequest(
            room_name=room_name,
            layout="speaker-dark",
            stream_outputs=[
                api.StreamOutput(
                    protocol=api.StreamProtocol.HLS,
                    urls=[hls_url],
                )
            ],
        )
        egress = await self.lk_api.egress.start_room_composite_egress(request)
        return egress.egress_id

    async def stop_egress(self, egress_id: str):
        await self.lk_api.egress.stop_egress(
            api.StopEgressRequest(egress_id=egress_id)
        )

    async def get_room_participants(self, room_name: str) -> list[api.ParticipantInfo]:
        response = await self.lk_api.room.list_participants(
            api.ListParticipantsRequest(room=room_name)
        )
        return response.participants

    async def send_data(
        self,
        room_name: str,
        data: bytes,
        kind: api.DataPacket_Kind = api.DataPacket_Kind.RELIABLE,
        destination_identities: Optional[list[str]] = None,
    ):
        await self.lk_api.room.send_data(
            api.SendDataRequest(
                room=room_name,
                data=data,
                kind=kind,
                destination_identities=destination_identities or [],
            )
        )

    async def update_participant_metadata(
        self,
        room_name: str,
        identity: str,
        metadata: str,
        attributes: Optional[dict] = None,
    ):
        await self.lk_api.room.update_participant(
            api.UpdateParticipantRequest(
                room=room_name,
                identity=identity,
                metadata=metadata,
                attributes=attributes or {},
            )
        )

    async def remove_participant(self, room_name: str, identity: str):
        await self.lk_api.room.remove_participant(
            api.RoomParticipantIdentity(
                room=room_name,
                identity=identity,
            )
        )

    async def delete_room(self, room_name: str):
        await self.lk_api.room.delete_room(
            api.DeleteRoomRequest(name=room_name)
        )
        logger.info(f"Room deleted: {room_name}")


# Adaptive Bitrate Controller
class ABRController:
    QUALITY_PROFILES = {
        "high":   {"width": 1920, "height": 1080, "fps": 30, "bitrate": 4000},
        "medium": {"width": 1280, "height": 720,  "fps": 30, "bitrate": 1500},
        "low":    {"width": 640,  "height": 360,  "fps": 15, "bitrate": 500},
        "audio":  {"width": 0,    "height": 0,    "fps": 0,  "bitrate": 64},
    }

    def select_quality(
        self,
        available_bandwidth_kbps: float,
        packet_loss_rate: float,
        rtt_ms: float,
    ) -> str:
        # Penalize available bandwidth based on network quality
        effective_bw = available_bandwidth_kbps
        if packet_loss_rate > 0.05:  # >5% packet loss
            effective_bw *= 0.5
        if rtt_ms > 200:
            effective_bw *= 0.7

        if effective_bw >= 5000:
            return "high"
        elif effective_bw >= 2000:
            return "medium"
        elif effective_bw >= 700:
            return "low"
        else:
            return "audio"
PYTHON

echo "WebRTC platform setup complete"
```

---

## ขั้นตอนที่ 626: Real-Time Multiplayer Game Server (Socket.io + Node.js)

### `game-server-realtime.sh`

```bash
#!/bin/bash
# Real-Time Multiplayer Game Server: Socket.io, Room Management, State Sync

cat > game-server/server.ts << 'TS'
import { createServer } from 'http';
import { Server, Socket } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import { instrument } from '@socket.io/admin-ui';
import { Tracer, trace, SpanStatusCode } from '@opentelemetry/api';

interface Player {
  id: string;
  name: string;
  position: { x: number; y: number; z: number };
  rotation: { yaw: number; pitch: number; roll: number };
  health: number;
  score: number;
  team: string;
  latency: number;
  lastUpdateAt: number;
}

interface GameRoom {
  id: string;
  mode: 'battle-royale' | 'squad' | 'duel';
  maxPlayers: number;
  players: Map<string, Player>;
  state: 'waiting' | 'countdown' | 'active' | 'ended';
  startTime?: number;
  tickRate: number;
  worldState: WorldState;
}

interface WorldState {
  safeZoneCenter: { x: number; y: number };
  safeZoneRadius: number;
  safeZoneNextRadius: number;
  nextZoneShrinkAt: number;
  pickups: Pickup[];
  projectiles: Projectile[];
}

interface Pickup {
  id: string;
  type: 'health' | 'ammo' | 'weapon' | 'shield';
  position: { x: number; y: number; z: number };
  value: number;
  spawnedAt: number;
}

interface Projectile {
  id: string;
  ownerId: string;
  weaponType: string;
  position: { x: number; y: number; z: number };
  velocity: { x: number; y: number; z: number };
  spawnedAt: number;
  ttl: number;
}

class GameServer {
  private io: Server;
  private rooms = new Map<string, GameRoom>();
  private playerRoomMap = new Map<string, string>();
  private tracer: Tracer;
  private TICK_RATE = 20; // 20Hz server tick
  private tickIntervals = new Map<string, NodeJS.Timeout>();

  constructor() {
    const httpServer = createServer();
    this.tracer = trace.getTracer('game-server', '1.0.0');

    this.io = new Server(httpServer, {
      cors: { origin: '*', methods: ['GET', 'POST'] },
      transports: ['websocket'],
      pingTimeout: 10000,
      pingInterval: 5000,
      maxHttpBufferSize: 1e6,
      perMessageDeflate: {
        threshold: 1024,
        zlibDeflateOptions: { level: 6 },
      },
    });

    instrument(this.io, { auth: false, mode: 'development' });

    this.setupRedisAdapter();
    this.setupNamespaces();
    httpServer.listen(3000, () => console.log('Game server running on :3000'));
  }

  private async setupRedisAdapter() {
    const pubClient = createClient({ url: process.env.REDIS_URL });
    const subClient = pubClient.duplicate();
    await Promise.all([pubClient.connect(), subClient.connect()]);
    this.io.adapter(createAdapter(pubClient, subClient));
    console.log('Redis adapter connected');
  }

  private setupNamespaces() {
    // Game namespace
    const game = this.io.of('/game');

    game.use(async (socket, next) => {
      const span = this.tracer.startSpan('socket.auth');
      try {
        const token = socket.handshake.auth.token as string;
        if (!token) throw new Error('No auth token');
        // Verify JWT (simplified)
        socket.data.playerId = this.verifyToken(token);
        socket.data.playerName = socket.handshake.auth.name;
        span.setStatus({ code: SpanStatusCode.OK });
        next();
      } catch (err) {
        span.setStatus({ code: SpanStatusCode.ERROR });
        next(new Error('Authentication failed'));
      } finally {
        span.end();
      }
    });

    game.on('connection', (socket) => this.handleConnection(socket));
  }

  private verifyToken(token: string): string {
    // In production: verify JWT signature + expiry
    return token.split(':')[1] || token;
  }

  private handleConnection(socket: Socket) {
    console.log(`Player connected: ${socket.data.playerId}`);

    socket.on('join_room', (data) => this.handleJoinRoom(socket, data));
    socket.on('player_update', (data) => this.handlePlayerUpdate(socket, data));
    socket.on('fire_weapon', (data) => this.handleFireWeapon(socket, data));
    socket.on('pickup_item', (data) => this.handlePickupItem(socket, data));
    socket.on('chat_message', (data) => this.handleChatMessage(socket, data));
    socket.on('disconnect', () => this.handleDisconnect(socket));

    // Measure latency
    socket.on('ping', (timestamp: number) => {
      socket.emit('pong', timestamp);
    });
  }

  private handleJoinRoom(socket: Socket, { roomId, team }: { roomId: string; team: string }) {
    const span = this.tracer.startSpan('game.join_room');
    span.setAttribute('room.id', roomId);
    span.setAttribute('player.id', socket.data.playerId);

    try {
      let room = this.rooms.get(roomId);
      if (!room) {
        room = this.createRoom(roomId, 'battle-royale');
        this.rooms.set(roomId, room);
      }

      if (room.players.size >= room.maxPlayers) {
        socket.emit('error', { code: 'ROOM_FULL', message: 'Room is full' });
        return;
      }

      const player: Player = {
        id: socket.data.playerId,
        name: socket.data.playerName,
        position: { x: Math.random() * 100, y: 0, z: Math.random() * 100 },
        rotation: { yaw: 0, pitch: 0, roll: 0 },
        health: 100,
        score: 0,
        team,
        latency: 0,
        lastUpdateAt: Date.now(),
      };

      room.players.set(socket.data.playerId, player);
      this.playerRoomMap.set(socket.id, roomId);

      socket.join(roomId);
      socket.emit('room_joined', {
        roomId,
        playerId: socket.data.playerId,
        worldState: this.serializeWorldState(room),
        players: Array.from(room.players.values()),
        serverTickRate: room.tickRate,
      });

      socket.to(roomId).emit('player_joined', player);

      if (room.players.size >= room.maxPlayers && room.state === 'waiting') {
        this.startCountdown(room);
      }

      span.setStatus({ code: SpanStatusCode.OK });
    } finally {
      span.end();
    }
  }

  private handlePlayerUpdate(
    socket: Socket,
    update: {
      position: Player['position'];
      rotation: Player['rotation'];
      health: number;
      latency: number;
      tick: number;
    }
  ) {
    const roomId = this.playerRoomMap.get(socket.id);
    if (!roomId) return;

    const room = this.rooms.get(roomId);
    if (!room || room.state !== 'active') return;

    const player = room.players.get(socket.data.playerId);
    if (!player) return;

    // Server-side validation
    const maxSpeedPerTick = 10 / this.TICK_RATE; // 10 units/sec max
    const dist = Math.sqrt(
      Math.pow(update.position.x - player.position.x, 2) +
      Math.pow(update.position.z - player.position.z, 2)
    );

    if (dist > maxSpeedPerTick * 3) { // 3-tick tolerance for lag
      // Potential speed hack — correct position
      socket.emit('position_corrected', player.position);
      return;
    }

    player.position = update.position;
    player.rotation = update.rotation;
    player.health = Math.min(update.health, 100);
    player.latency = update.latency;
    player.lastUpdateAt = Date.now();

    // Broadcast delta to others in room (not back to sender)
    socket.to(roomId).emit('player_state', {
      id: socket.data.playerId,
      position: update.position,
      rotation: update.rotation,
      health: update.health,
      tick: update.tick,
    });
  }

  private handleFireWeapon(
    socket: Socket,
    { weaponType, origin, direction }: { weaponType: string; origin: Player['position']; direction: { x: number; y: number; z: number } }
  ) {
    const roomId = this.playerRoomMap.get(socket.id);
    if (!roomId) return;

    const room = this.rooms.get(roomId);
    if (!room) return;

    const projectile: Projectile = {
      id: `${socket.data.playerId}-${Date.now()}`,
      ownerId: socket.data.playerId,
      weaponType,
      position: origin,
      velocity: direction,
      spawnedAt: Date.now(),
      ttl: 3000,
    };

    room.worldState.projectiles.push(projectile);

    this.io.of('/game').to(roomId).emit('projectile_fired', projectile);
  }

  private handlePickupItem(socket: Socket, { pickupId }: { pickupId: string }) {
    const roomId = this.playerRoomMap.get(socket.id);
    if (!roomId) return;

    const room = this.rooms.get(roomId);
    if (!room) return;

    const pickupIndex = room.worldState.pickups.findIndex(p => p.id === pickupId);
    if (pickupIndex === -1) return;

    const pickup = room.worldState.pickups[pickupIndex];
    const player = room.players.get(socket.data.playerId);
    if (!player) return;

    room.worldState.pickups.splice(pickupIndex, 1);

    if (pickup.type === 'health') {
      player.health = Math.min(100, player.health + pickup.value);
    }

    this.io.of('/game').to(roomId).emit('item_picked_up', {
      pickupId,
      playerId: socket.data.playerId,
      newHealth: player.health,
    });
  }

  private handleChatMessage(socket: Socket, { message }: { message: string }) {
    const roomId = this.playerRoomMap.get(socket.id);
    if (!roomId) return;

    // Sanitize message
    const sanitized = message.substring(0, 200).replace(/[<>]/g, '');
    this.io.of('/game').to(roomId).emit('chat', {
      playerId: socket.data.playerId,
      playerName: socket.data.playerName,
      message: sanitized,
      timestamp: Date.now(),
    });
  }

  private handleDisconnect(socket: Socket) {
    const roomId = this.playerRoomMap.get(socket.id);
    if (roomId) {
      const room = this.rooms.get(roomId);
      if (room) {
        room.players.delete(socket.data.playerId);
        socket.to(roomId).emit('player_left', { id: socket.data.playerId });

        if (room.players.size === 0) {
          this.cleanupRoom(roomId);
        }
      }
      this.playerRoomMap.delete(socket.id);
    }
    console.log(`Player disconnected: ${socket.data.playerId}`);
  }

  private createRoom(id: string, mode: GameRoom['mode']): GameRoom {
    return {
      id,
      mode,
      maxPlayers: mode === 'battle-royale' ? 100 : mode === 'squad' ? 16 : 2,
      players: new Map(),
      state: 'waiting',
      tickRate: this.TICK_RATE,
      worldState: {
        safeZoneCenter: { x: 0, y: 0 },
        safeZoneRadius: 500,
        safeZoneNextRadius: 400,
        nextZoneShrinkAt: Date.now() + 120000,
        pickups: this.generatePickups(50),
        projectiles: [],
      },
    };
  }

  private generatePickups(count: number): Pickup[] {
    const types: Pickup['type'][] = ['health', 'ammo', 'weapon', 'shield'];
    return Array.from({ length: count }, (_, i) => ({
      id: `pickup-${i}`,
      type: types[Math.floor(Math.random() * types.length)],
      position: {
        x: (Math.random() - 0.5) * 900,
        y: 0,
        z: (Math.random() - 0.5) * 900,
      },
      value: Math.floor(Math.random() * 50) + 10,
      spawnedAt: Date.now(),
    }));
  }

  private startCountdown(room: GameRoom) {
    room.state = 'countdown';
    let countdown = 10;
    const ns = this.io.of('/game');

    const interval = setInterval(() => {
      ns.to(room.id).emit('countdown', { seconds: countdown });
      countdown--;
      if (countdown < 0) {
        clearInterval(interval);
        this.startGame(room);
      }
    }, 1000);
  }

  private startGame(room: GameRoom) {
    room.state = 'active';
    room.startTime = Date.now();
    const ns = this.io.of('/game');
    ns.to(room.id).emit('game_start', { startTime: room.startTime });

    // Server-side game tick for authoritative state
    const tickInterval = setInterval(() => this.gameTick(room), 1000 / room.tickRate);
    this.tickIntervals.set(room.id, tickInterval);
  }

  private gameTick(room: GameRoom) {
    const now = Date.now();
    const ns = this.io.of('/game');

    // Clean up expired projectiles
    room.worldState.projectiles = room.worldState.projectiles.filter(
      p => now - p.spawnedAt < p.ttl
    );

    // Safe zone shrink
    if (now >= room.worldState.nextZoneShrinkAt) {
      room.worldState.safeZoneRadius = room.worldState.safeZoneNextRadius;
      room.worldState.safeZoneNextRadius = Math.max(50, room.worldState.safeZoneRadius * 0.7);
      room.worldState.nextZoneShrinkAt = now + 90000;

      ns.to(room.id).emit('zone_update', {
        center: room.worldState.safeZoneCenter,
        radius: room.worldState.safeZoneRadius,
        nextRadius: room.worldState.safeZoneNextRadius,
        shrinkAt: room.worldState.nextZoneShrinkAt,
      });
    }

    // Apply zone damage to players outside
    for (const [, player] of room.players) {
      const distToCenter = Math.sqrt(
        Math.pow(player.position.x - room.worldState.safeZoneCenter.x, 2) +
        Math.pow(player.position.z - room.worldState.safeZoneCenter.y, 2)
      );
      if (distToCenter > room.worldState.safeZoneRadius) {
        player.health -= 2; // 2 HP per tick outside zone
        if (player.health <= 0) {
          ns.to(room.id).emit('player_eliminated', { id: player.id, cause: 'zone' });
          room.players.delete(player.id);
        }
      }
    }

    // Check win condition
    if (room.players.size <= 1 && room.state === 'active') {
      const winner = room.players.values().next().value;
      room.state = 'ended';
      ns.to(room.id).emit('game_ended', { winner: winner?.id });
      this.cleanupRoom(room.id);
    }
  }

  private serializeWorldState(room: GameRoom) {
    return {
      safeZoneCenter: room.worldState.safeZoneCenter,
      safeZoneRadius: room.worldState.safeZoneRadius,
      nextZoneShrinkAt: room.worldState.nextZoneShrinkAt,
      pickupCount: room.worldState.pickups.length,
    };
  }

  private cleanupRoom(roomId: string) {
    const interval = this.tickIntervals.get(roomId);
    if (interval) {
      clearInterval(interval);
      this.tickIntervals.delete(roomId);
    }
    this.rooms.delete(roomId);
    console.log(`Room cleaned up: ${roomId}`);
  }
}

new GameServer();
TS

# Kubernetes deployment for game server
cat > game-server-k8s.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-server
  namespace: game-prod
spec:
  replicas: 5
  selector:
    matchLabels:
      app: game-server
  template:
    metadata:
      labels:
        app: game-server
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
        - name: game-server
          image: gcr.io/myproject/game-server:latest
          ports:
            - containerPort: 3000
              name: websocket
            - containerPort: 9090
              name: metrics
          env:
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: game-server-secrets
                  key: redis-url
            - name: NODE_ENV
              value: "production"
            - name: UV_THREADPOOL_SIZE
              value: "128"
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "4"
              memory: "8Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 9090
            initialDelaySeconds: 15
            periodSeconds: 10
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: game-server-pdb
  namespace: game-prod
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: game-server
EOF
kubectl apply -f game-server-k8s.yaml

echo "Real-time game server deployed"
```

---

## สรุป Part 69

Part 69 ครอบคลุมการพัฒนาระบบเรียลไทม์และอินเตอร์แอคทีฟขนาด production:

| Step | หัวข้อ | เทคโนโลยีหลัก |
|------|--------|--------------|
| 623 | AR/VR Platform | WebXR, Three.js VR, AR.js, Spatial Audio, Holo Panel |
| 624 | Gaming Infrastructure | Agones Fleet/FleetAutoscaler, Skill-based Matchmaking, K8s allocation |
| 625 | WebRTC at Scale | LiveKit SFU, coturn TURN/STUN, ABR Controller, Egress Recording |
| 626 | Real-Time Multiplayer | Socket.io TypeScript, Server-authoritative game loop, Anti-cheat, Zone system |

### Key Concepts ที่เรียนรู้:
- **WebXR API**: VRButton, XRController, Haptic Feedback, 3D Data Visualization
- **Agones**: GameServer CRD, FleetAutoscaler Buffer policy, GameServerAllocation
- **Matchmaking**: Skill-based grouping with wait-time tolerance expansion
- **LiveKit**: SFU architecture, Room management, Egress (Recording + HLS), Access tokens
- **TURN/STUN**: coturn deployment, external-ip resolution, static-auth-secret
- **Socket.io Cluster**: Redis adapter, namespace middleware, binary protocol
- **Game Server Authority**: Server-side validation, speed hack detection, position correction
- **Battle Royale Mechanics**: Safe zone shrinking, zone damage, win condition detection

ขั้นตอนต่อไป: **Part 70** — Blockchain & Web3 Infrastructure, Smart Contract Security Auditing และ DeFi Protocol Engineering
