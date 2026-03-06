<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>886E — Particle Filter Simulator</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    background: #060910;
    color: #e2e8f0;
    font-family: 'Courier New', monospace;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    padding: 24px;
  }
  h1  { font-size: 22px; letter-spacing: 0.05em; color: #f1f5f9; margin-bottom: 4px; }
  .subtitle { font-size: 11px; letter-spacing: 0.3em; color: #64748b; text-transform: uppercase; margin-bottom: 6px; }
  .hint     { font-size: 11px; color: #38bdf8; margin-bottom: 18px; min-height: 16px; }
  .layout   { display: flex; gap: 20px; align-items: flex-start; flex-wrap: wrap; justify-content: center; }

  .canvas-wrap { position: relative; }
  canvas { border: 1px solid rgba(255,255,255,0.08); border-radius: 8px; display: block; }

  .legend { position: absolute; bottom: 48px; left: 8px; display: flex; flex-direction: column; gap: 4px; }
  .legend-item { display: flex; align-items: center; gap: 6px; font-size: 10px; color: #94a3b8; }
  .legend-dot  { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }

  .beam-warn { position: absolute; top: 8px; left: 50%; transform: translateX(-50%);
    background: rgba(251,146,60,0.15); border: 1px solid rgba(251,146,60,0.4);
    color: #fb923c; font-size: 10px; padding: 3px 10px; border-radius: 20px;
    white-space: nowrap; opacity: 0; transition: opacity 0.3s; pointer-events: none; }
  .beam-warn.show { opacity: 1; }

  .sidebar    { display: flex; flex-direction: column; gap: 12px; width: 210px; }
  .panel      { background: rgba(255,255,255,0.03); border: 1px solid rgba(255,255,255,0.07); border-radius: 8px; padding: 14px; }
  .panel-title{ font-size: 10px; color: #64748b; letter-spacing: 0.2em; margin-bottom: 10px; }
  .stat-row   { display: flex; justify-content: space-between; margin-bottom: 5px; font-size: 12px; }
  .stat-label { color: #64748b; }
  .stat-val   { color: #e2e8f0; font-weight: 600; }
  .stat-val.warn { color: #fb923c; }

  .btn      { padding: 10px; border-radius: 6px; border: none; cursor: pointer; font-size: 12px;
    font-family: 'Courier New', monospace; font-weight: 700; letter-spacing: 0.1em;
    width: 100%; transition: all 0.2s; }
  .btn-grey { background: rgba(255,255,255,0.04); color: #94a3b8; border: 1px solid rgba(255,255,255,0.1); }

  input[type=range] { width: 100%; accent-color: #38bdf8; margin-bottom: 10px; cursor: pointer; }
</style>
</head>
<body>

<div class="subtitle">Monte Carlo Localization — 4-Beam Wall Distance Sensors</div>
<h1>886E — Particle Filter Simulator</h1>

<div class="layout">
  <div class="canvas-wrap">
    <canvas id="canvas" width="440" height="440"></canvas>
    <div class="beam-warn" id="beam-warn">⚠ All beams out of range — particles drifting</div>

    <div class="legend">
      <div class="legend-item"><div class="legend-dot" style="background:#f87171"></div>True robot</div>
      <div class="legend-item"><div class="legend-dot" style="background:#e2e8f0;opacity:0.5"></div>Particles</div>
      <div class="legend-item"><div class="legend-dot" style="background:#34d399"></div>Estimate</div>
      <div class="legend-item"><div class="legend-dot" style="background:#38bdf8"></div>Front beam (F)</div>
      <div class="legend-item"><div class="legend-dot" style="background:#a78bfa"></div>Right beam (R)</div>
      <div class="legend-item"><div class="legend-dot" style="background:#fb923c"></div>Back beam (B)</div>
      <div class="legend-item"><div class="legend-dot" style="background:#34d399"></div>Left beam (L)</div>
    </div>
  </div>

  <div class="sidebar">
    <div class="panel">
      <div class="panel-title">STATUS</div>
      <div class="stat-row"><span class="stat-label">Step</span><span class="stat-val" id="s-step">0</span></div>
      <div class="stat-row"><span class="stat-label">Error</span><span class="stat-val" id="s-error">—</span></div>
      <div class="stat-row"><span class="stat-label">Beams reading</span><span class="stat-val" id="s-beams">0 / 4</span></div>
      <div class="stat-row"><span class="stat-label">Beam width</span><span class="stat-val" id="s-fov">20°</span></div>
      <div class="stat-row"><span class="stat-label">Sensor range</span><span class="stat-val" id="s-range">60 in</span></div>
      <div class="stat-row"><span class="stat-label">Heading (gyro)</span><span class="stat-val" id="s-heading">0.0°</span></div>
      <div class="stat-row"><span class="stat-label">Robot X</span><span class="stat-val" id="s-rx">—</span></div>
      <div class="stat-row"><span class="stat-label">Robot Y</span><span class="stat-val" id="s-ry">—</span></div>
      <div class="stat-row"><span class="stat-label">Particles</span><span class="stat-val">500</span></div>
    </div>

    <div class="panel">
      <div class="panel-title">BEAM READINGS (in)</div>
      <div class="stat-row"><span class="stat-label" style="color:#38bdf8">▶ Front</span><span class="stat-val" id="b-f">—</span></div>
      <div class="stat-row"><span class="stat-label" style="color:#a78bfa">▶ Right</span><span class="stat-val" id="b-r">—</span></div>
      <div class="stat-row"><span class="stat-label" style="color:#fb923c">▶ Back</span> <span class="stat-val" id="b-b">—</span></div>
      <div class="stat-row"><span class="stat-label" style="color:#34d399">▶ Left</span> <span class="stat-val" id="b-l">—</span></div>
    </div>

    <div class="panel">
      <div class="panel-title">ERROR HISTORY</div>
      <svg id="chart" width="182" height="55" style="display:block"></svg>
    </div>

    <div class="panel">
      <div class="panel-title">SENSOR SETTINGS</div>
      <div class="stat-row" style="margin-bottom:2px;">
        <span class="stat-label" style="font-size:11px;">Beam width</span>
        <span class="stat-val"   style="font-size:11px;" id="lbl-fov">20°</span>
      </div>
      <input type="range" id="sl-fov" min="2" max="45" value="20" step="1">
      <div class="stat-row" style="margin-bottom:2px;">
        <span class="stat-label" style="font-size:11px;">Sensor range (in)</span>
        <span class="stat-val"   style="font-size:11px;" id="lbl-range">60</span>
      </div>
      <input type="range" id="sl-range" min="1" max="144" value="60" step="1">
      <div class="stat-row" style="margin-bottom:2px;">
        <span class="stat-label" style="font-size:11px;">Sensor noise (in)</span>
        <span class="stat-val"   style="font-size:11px;" id="lbl-noise">0.2</span>
      </div>
      <input type="range" id="sl-noise" min="0.1" max="6" value="0.2" step="0.1" style="margin-bottom:10px;">
      <div class="stat-row" style="margin-bottom:2px;">
        <span class="stat-label" style="font-size:11px;">Gyro drift (rad/step)</span>
        <span class="stat-val"   style="font-size:11px;" id="lbl-gyro">0.01</span>
      </div>
      <input type="range" id="sl-gyro" min="0.001" max="0.3" value="0.01" step="0.001" style="margin-bottom:0;">
    </div>

    <div style="display:flex;flex-direction:column;gap:8px;">
      <button class="btn btn-grey" id="btn-reset">↺ RESET</button>
    </div>

    <div class="panel">
      <div class="panel-title">MCL PHASES</div>
      <div class="phase"><div class="phase-num">1</div><div><div class="phase-name">Init</div><div class="phase-desc">Scatter particles in room</div></div></div>
      <div class="phase"><div class="phase-num">2</div><div><div class="phase-name">Sense</div><div class="phase-desc">4 beams measure wall dist</div></div></div>
      <div class="phase"><div class="phase-num">3</div><div><div class="phase-name">Predict</div><div class="phase-desc">Move particles + noise</div></div></div>
      <div class="phase"><div class="phase-num">4</div><div><div class="phase-name">Weight</div><div class="phase-desc">Score against wall dists</div></div></div>
      <div class="phase"><div class="phase-num">5</div><div><div class="phase-name">Resample</div><div class="phase-desc">Survive or die by weight</div></div></div>
    </div>
  </div>
</div>

<script>
// ─── World setup ──────────────────────────────────────────────────────────────
//
// The room is a 12ft × 12ft square.
// 1 foot = 304.8mm, so 12ft = 3657.6mm — we'll use 3600mm for simplicity.
// On canvas, the room is 400×400px with a 20px margin on each side (440px canvas).
//
// Scale: 400px = 3600mm → 1px = 9mm

const ROOM_MM      = 3600;  // Room is 3600mm × 3600mm (≈ 12ft × 12ft = 144in × 144in)
const CANVAS_SIZE  = 440;
const MARGIN_PX    = 20;
const ROOM_PX      = 400;
const MM_PER_PX    = ROOM_MM / ROOM_PX;   // 9 mm per pixel
const PX_PER_MM    = ROOM_PX / ROOM_MM;

// Unit conversions
function mmToPx(mm)     { return mm * PX_PER_MM; }
function pxToMm(px)     { return px * MM_PER_PX; }
function inToMm(inches) { return inches * 25.4; }
function mmToIn(mm)     { return mm / 25.4; }
function pxToIn(px)     { return mmToIn(pxToMm(px)); }

// Room walls in canvas-pixel coords (top-left corner of room = MARGIN_PX, MARGIN_PX)
// The robot lives inside this box
const WALL = {
  left:   MARGIN_PX,
  right:  MARGIN_PX + ROOM_PX,
  top:    MARGIN_PX,
  bottom: MARGIN_PX + ROOM_PX,
};

// ─── Constants ────────────────────────────────────────────────────────────────
const NUM_PARTICLES  = 500;
const MOTION_NOISE   = 0.3;   // Position noise on particle movement (px)
const RESAMPLE_NOISE = 0.6;   // Jitter added on resample (px)

// Controlled by sliders
let SENSOR_NOISE_IN  = 0.2;
let BEAM_WIDTH_DEG   = 20;
let SENSOR_RANGE_IN  = 60;
// Gyro drift: how much the gyro heading reading deviates from truth (radians std dev per step)
// 0 = perfect gyro, 0.01 = very good, 0.05 = mediocre, 0.2 = bad
let GYRO_NOISE       = 0.01;

// Internal mm values used for geometry
function sensorRangeMm()  { return inToMm(SENSOR_RANGE_IN); }
function sensorRangePx()  { return mmToPx(sensorRangeMm()); }
function sensorNoiseMm()  { return inToMm(SENSOR_NOISE_IN); }

// Four beams: front (0°), right (90°), back (180°), left (270°) relative to heading
const BEAM_OFFSETS = [0, Math.PI / 2, Math.PI, -Math.PI / 2];
const BEAM_LABELS  = ["F", "R", "B", "L"];
const BEAM_COLORS  = ["#38bdf8", "#a78bfa", "#fb923c", "#34d399"];
const BEAM_IDS     = ["b-f", "b-r", "b-b", "b-l"];

// ─── Math helpers ─────────────────────────────────────────────────────────────

function gauss(mean, std) {
  let u = 0, v = 0;
  while (!u) u = Math.random();
  while (!v) v = Math.random();
  return mean + std * Math.sqrt(-2 * Math.log(u)) * Math.cos(2 * Math.PI * v);
}
function dist(x1,y1,x2,y2) { return Math.sqrt((x1-x2)**2+(y1-y2)**2); }
function clamp(v,lo,hi)     { return Math.max(lo,Math.min(hi,v)); }
function normAngle(a) {
  while (a >  Math.PI) a -= 2*Math.PI;
  while (a < -Math.PI) a += 2*Math.PI;
  return a;
}
function hexAlpha(hex, alpha) {
  const r=parseInt(hex.slice(1,3),16), g=parseInt(hex.slice(3,5),16), b=parseInt(hex.slice(5,7),16);
  return `rgba(${r},${g},${b},${alpha})`;
}

// ─── Wall ray casting ─────────────────────────────────────────────────────────
//
// Cast a ray from (rx, ry) in direction `angle` (radians).
// Returns the distance in PIXELS to the first wall hit, or Infinity if no wall in range.
// The room has 4 axis-aligned walls — we solve for intersection with each and take the min.
//
function rayToWall(rx, ry, angle) {
  const cos = Math.cos(angle);
  const sin = Math.sin(angle);
  let minDist = Infinity;

  // Each wall is a line at a fixed x or y. Solve: point + t*(cos,sin) = wall
  // t must be > 0 (forward along ray), and the intersection must be on the wall segment.

  // Left wall (x = WALL.left)
  if (cos < -1e-9) {
    const t = (WALL.left - rx) / cos;
    const y = ry + t * sin;
    if (t > 0 && y >= WALL.top && y <= WALL.bottom) minDist = Math.min(minDist, t);
  }
  // Right wall (x = WALL.right)
  if (cos > 1e-9) {
    const t = (WALL.right - rx) / cos;
    const y = ry + t * sin;
    if (t > 0 && y >= WALL.top && y <= WALL.bottom) minDist = Math.min(minDist, t);
  }
  // Top wall (y = WALL.top)
  if (sin < -1e-9) {
    const t = (WALL.top - ry) / sin;
    const x = rx + t * cos;
    if (t > 0 && x >= WALL.left && x <= WALL.right) minDist = Math.min(minDist, t);
  }
  // Bottom wall (y = WALL.bottom)
  if (sin > 1e-9) {
    const t = (WALL.bottom - ry) / sin;
    const x = rx + t * cos;
    if (t > 0 && x >= WALL.left && x <= WALL.right) minDist = Math.min(minDist, t);
  }

  return minDist; // in pixels
}

// ─── Simulation state ─────────────────────────────────────────────────────────
let particles  = [];
let robot      = { x: WALL.left + ROOM_PX/2, y: WALL.top + ROOM_PX/2, theta: 0 };
let estimate   = { x: robot.x, y: robot.y };
let stepCount  = 0;
let errorVal   = 0;
let history    = [];

// Latest beam readings in inches (null = out of range)
let beamReadings = [null, null, null, null];

// ─── Phase 1: Initialize particles ───────────────────────────────────────────
// Scatter particles uniformly in the room.
// With a gyro, we initialise all particles to the robot's known heading + tiny drift.
// This eliminates rotational ambiguity from the very start.
function initParticles() {
  particles = [];
  for (let i = 0; i < NUM_PARTICLES; i++) {
    particles.push({
      x:     WALL.left + Math.random() * ROOM_PX,
      y:     WALL.top  + Math.random() * ROOM_PX,
      // Gyro gives us the heading — each particle starts near the true heading
      theta: robot.theta + gauss(0, GYRO_NOISE),
      w:     1 / NUM_PARTICLES,
    });
  }
}

// ─── Phase 2: Sense — 4-beam wall distance ────────────────────────────────────
// For each of the 4 beams, cast a ray along the beam's center direction and
// measure the distance to the nearest wall. If the wall is within sensor range,
// return the noisy distance (in mm). Otherwise return null (out of range).
//
// Note: each beam fires a single center ray. The beam width (BEAM_WIDTH_DEG) only
// affects how the beam is visualized as a cone — the measurement is from the center ray.
function senseBeams(rx, ry, theta) {
  return BEAM_OFFSETS.map(offset => {
    const beamDir = theta + offset;
    const distPx  = rayToWall(rx, ry, beamDir);
    if (distPx > sensorRangePx()) return null;
    const distIn  = pxToIn(distPx);                   // convert to inches for output
    return distIn + gauss(0, SENSOR_NOISE_IN);         // noise in inches
  });
}

// ─── Phase 3: Move robot ──────────────────────────────────────────────────────
function moveRobot(fwdPx, turn) {
  robot.theta += turn;
  robot.x = clamp(robot.x + fwdPx * Math.cos(robot.theta), WALL.left + 5,  WALL.right  - 5);
  robot.y = clamp(robot.y + fwdPx * Math.sin(robot.theta), WALL.top  + 5,  WALL.bottom - 5);
}

// ─── Phase 3: Move particles (gyro-assisted) ─────────────────────────────────
// With a gyro, each particle's heading is set to the robot's measured heading
// plus a small gyro drift noise — NOT a free random walk.
// This keeps all particles rotationally aligned, eliminating the biggest source
// of divergence in a symmetric room.
function moveParticles(fwdPx, turn) {
  particles = particles.map(p => {
    // Gyro gives the true heading — particle theta tracks it with tiny drift
    const theta = robot.theta + gauss(0, GYRO_NOISE);
    return {
      ...p, theta,
      x: clamp(p.x + fwdPx*Math.cos(theta) + gauss(0, MOTION_NOISE*0.5), WALL.left, WALL.right),
      y: clamp(p.y + fwdPx*Math.sin(theta) + gauss(0, MOTION_NOISE*0.5), WALL.top,  WALL.bottom),
    };
  });
}

// ─── Phase 4: Update weights ──────────────────────────────────────────────────
// For each particle, simulate what its 4 beams would read if it were the robot.
// Compare against the real robot's beam readings using Gaussian log-likelihood.
// Only beams that returned a valid reading are used for scoring.
function updateWeights(measurement) {
  let total    = 0;
  let seenCount = measurement.filter(m => m !== null).length;

  particles = particles.map(p => {
    let logW     = 0;
    let compared = 0;

    BEAM_OFFSETS.forEach((offset, b) => {
      if (measurement[b] === null) return; // This beam was out of range — skip

      const beamDir      = p.theta + offset;
      const predictedPx  = rayToWall(p.x, p.y, beamDir);
      const predictedIn  = pxToIn(predictedPx);
      const err          = (predictedIn - measurement[b]) ** 2;

      // Gaussian log-likelihood in inch space
      logW += -err / (2 * SENSOR_NOISE_IN * SENSOR_NOISE_IN);
      compared++;
    });

    const w = compared > 0 ? Math.exp(logW) : 0.0001;
    total  += w;
    return { ...p, w };
  });

  particles = particles.map(p => ({ ...p, w: p.w / (total || 1) }));
  return seenCount;
}

// ─── Phase 5: Resample with jitter ───────────────────────────────────────────
function resample() {
  const next = [];
  for (let i = 0; i < NUM_PARTICLES; i++) {
    const r = Math.random();
    let cum = 0, chosen = particles[particles.length-1];
    for (const p of particles) {
      cum += p.w;
      if (r <= cum) { chosen = p; break; }
    }
    next.push({
      ...chosen,
      x:     clamp(chosen.x + gauss(0, RESAMPLE_NOISE), WALL.left, WALL.right),
      y:     clamp(chosen.y + gauss(0, RESAMPLE_NOISE), WALL.top,  WALL.bottom),
      theta: chosen.theta + gauss(0, 0.01),
      w:     1 / NUM_PARTICLES,
    });
  }
  particles = next;
}

// ─── Estimate + convergence ───────────────────────────────────────────────────
function estimatePos() {
  let x=0, y=0;
  particles.forEach(p => { x+=p.x; y+=p.y; });
  return { x: x/NUM_PARTICLES, y: y/NUM_PARTICLES };
}

function particleSpread() {
  let mx=0, my=0;
  particles.forEach(p => { mx+=p.x; my+=p.y; });
  mx/=NUM_PARTICLES; my/=NUM_PARTICLES;
  let v=0;
  particles.forEach(p => { v+=(p.x-mx)**2+(p.y-my)**2; });
  return Math.sqrt(v/NUM_PARTICLES);
}

// ─── Full MCL step ────────────────────────────────────────────────────────────
function runStep(fwdPx, turn) {
  moveRobot(fwdPx, turn);
  beamReadings = senseBeams(robot.x, robot.y, robot.theta);
  moveParticles(fwdPx, turn);
  const seen = updateWeights(beamReadings);
  resample();
  estimate  = estimatePos();
  // Error in inches
  errorVal  = pxToIn(dist(robot.x, robot.y, estimate.x, estimate.y));
  stepCount++;
  history = [...history.slice(-40), errorVal];

  updateUI(seen);
  draw();

  document.getElementById("beam-warn").classList.toggle("show", seen === 0);
}

// ─── UI ───────────────────────────────────────────────────────────────────────
function updateUI(seen) {
  document.getElementById("s-step").textContent    = stepCount;
  document.getElementById("s-error").textContent   = errorVal.toFixed(2) + " in";
  document.getElementById("s-fov").textContent     = BEAM_WIDTH_DEG + "°";
  document.getElementById("s-range").textContent   = SENSOR_RANGE_IN + " in";
  document.getElementById("s-rx").textContent      = pxToIn(robot.x - WALL.left).toFixed(1) + " in";
  document.getElementById("s-ry").textContent      = pxToIn(robot.y - WALL.top).toFixed(1)  + " in";
  document.getElementById("s-heading").textContent = (robot.theta * 180 / Math.PI).toFixed(1) + "°";
  const beamsEl = document.getElementById("s-beams");
  beamsEl.textContent = `${seen} / 4`;
  beamsEl.className   = "stat-val" + (seen === 0 ? " warn" : "");

  BEAM_IDS.forEach((id, b) => {
    const el = document.getElementById(id);
    el.textContent = beamReadings[b] !== null ? beamReadings[b].toFixed(2) + " in" : "—";
  });

  drawChart();
}

function drawChart() {
  const svg = document.getElementById("chart");
  if (history.length < 2) return;
  const maxE = Math.max(...history, 1);
  const pts  = history.map((e,i) => `${(i/(history.length-1))*182},${55-(e/maxE)*50}`).join(" ");
  const last = history[history.length-1];
  svg.innerHTML = `
    <polyline points="${pts}" fill="none" stroke="#38bdf8" stroke-width="1.5" stroke-linejoin="round"/>
    <circle cx="182" cy="${55-(last/maxE)*50}" r="3" fill="#38bdf8"/>`;
}

// ─── Canvas drawing ───────────────────────────────────────────────────────────
const canvas = document.getElementById("canvas");
const ctx    = canvas.getContext("2d");

function draw() {
  ctx.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

  // Dark background
  ctx.fillStyle = "#0a0e1a";
  ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

  // ── Room walls ──────────────────────────────────────────────────────────────
  // Draw the 12ft × 12ft square room
  ctx.strokeStyle = "#475569";
  ctx.lineWidth   = 3;
  ctx.strokeRect(WALL.left, WALL.top, ROOM_PX, ROOM_PX);

  // Wall fill (very subtle)
  ctx.fillStyle = "rgba(71,85,105,0.06)";
  ctx.fillRect(WALL.left, WALL.top, ROOM_PX, ROOM_PX);

  // Wall dimension labels
  ctx.font      = "10px 'Courier New'";
  ctx.fillStyle = "#475569";
  ctx.textAlign = "center";
  ctx.fillText("12 ft / 3600 mm", WALL.left + ROOM_PX/2, WALL.top - 6);
  ctx.save();
  ctx.translate(WALL.left - 6, WALL.top + ROOM_PX/2);
  ctx.rotate(-Math.PI/2);
  ctx.fillText("12 ft / 3600 mm", 0, 0);
  ctx.restore();
  ctx.textAlign = "left";

  // Grid lines inside room (every 300mm = 1ft ≈ 33px)
  const gridStepPx = mmToPx(300);
  ctx.strokeStyle = "rgba(255,255,255,0.04)";
  ctx.lineWidth   = 1;
  for (let i = 1; i < 12; i++) {
    const x = WALL.left + i * gridStepPx;
    const y = WALL.top  + i * gridStepPx;
    ctx.beginPath(); ctx.moveTo(x, WALL.top);    ctx.lineTo(x, WALL.bottom); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(WALL.left, y);   ctx.lineTo(WALL.right, y);  ctx.stroke();
  }

  // ── Particles ──────────────────────────────────────────────────────────────
  particles.forEach(p => {
    const alpha = Math.min(1, p.w * NUM_PARTICLES * 2);
    ctx.beginPath();
    ctx.arc(p.x, p.y, 2, 0, Math.PI * 2);
    ctx.fillStyle = `rgba(200,210,230,${0.08 + alpha * 0.6})`;
    ctx.fill();
  });

  // ── Sensor beams ───────────────────────────────────────────────────────────
  // Draw each beam: a narrow cone + a ray line to the wall hit point
  const halfAngle   = BEAM_WIDTH_DEG * Math.PI / 180;
  const rangePx     = sensorRangePx();

  BEAM_OFFSETS.forEach((offset, b) => {
    const beamDir  = robot.theta + offset;
    const wallDist = rayToWall(robot.x, robot.y, beamDir); // px
    const inRange  = wallDist <= rangePx;
    const drawDist = Math.min(wallDist, rangePx);          // clip to range
    const color    = BEAM_COLORS[b];

    // Beam cone (narrow wedge)
    ctx.beginPath();
    ctx.moveTo(robot.x, robot.y);
    ctx.arc(robot.x, robot.y, drawDist, beamDir - halfAngle, beamDir + halfAngle);
    ctx.closePath();
    ctx.fillStyle   = hexAlpha(color, inRange ? 0.07 : 0.03);
    ctx.strokeStyle = hexAlpha(color, inRange ? 0.4  : 0.12);
    ctx.lineWidth   = 1;
    ctx.fill();
    ctx.stroke();

    // Center ray line — solid if hitting wall, dashed if at range limit
    const hitX = robot.x + wallDist * Math.cos(beamDir);
    const hitY = robot.y + wallDist * Math.sin(beamDir);

    if (inRange) {
      // Solid ray to wall hit
      ctx.beginPath();
      ctx.moveTo(robot.x, robot.y);
      ctx.lineTo(hitX, hitY);
      ctx.strokeStyle = hexAlpha(color, 0.7);
      ctx.lineWidth   = 1.5;
      ctx.setLineDash([]);
      ctx.stroke();

      // Hit point dot on the wall
      ctx.beginPath();
      ctx.arc(hitX, hitY, 4, 0, Math.PI * 2);
      ctx.fillStyle = hexAlpha(color, 0.9);
      ctx.fill();

      // Distance label near hit point (in mm)
      const labelDist = beamReadings[b];
      if (labelDist !== null) {
        ctx.font      = "9px 'Courier New'";
        ctx.fillStyle = hexAlpha(color, 0.85);
        ctx.textAlign = "center";
        const lx = robot.x + (wallDist * 0.6) * Math.cos(beamDir);
        const ly = robot.y + (wallDist * 0.6) * Math.sin(beamDir);
        ctx.fillText(labelDist.toFixed(2) + "in", lx, ly - 5);
        ctx.textAlign = "left";
      }
    } else {
      // Dashed ray stopping at range limit
      ctx.beginPath();
      ctx.moveTo(robot.x, robot.y);
      ctx.lineTo(robot.x + rangePx*Math.cos(beamDir), robot.y + rangePx*Math.sin(beamDir));
      ctx.strokeStyle = hexAlpha(color, 0.2);
      ctx.lineWidth   = 1;
      ctx.setLineDash([4, 6]);
      ctx.stroke();
      ctx.setLineDash([]);
    }

    // Beam label (F/R/B/L) at tip of range arc
    const labelR = Math.min(drawDist, rangePx) + 12;
    ctx.font      = "bold 10px 'Courier New'";
    ctx.fillStyle = hexAlpha(color, inRange ? 0.8 : 0.3);
    ctx.textAlign = "center";
    ctx.fillText(BEAM_LABELS[b],
      robot.x + labelR * Math.cos(beamDir),
      robot.y + labelR * Math.sin(beamDir) + 4);
    ctx.textAlign = "left";
  });

  ctx.setLineDash([]);

  // ── Estimated position ─────────────────────────────────────────────────────
  ctx.beginPath();
  ctx.arc(estimate.x, estimate.y, 10, 0, Math.PI * 2);
  ctx.strokeStyle = "rgba(52,211,153,0.7)"; ctx.lineWidth = 2; ctx.stroke();
  ctx.beginPath();
  ctx.arc(estimate.x, estimate.y, 4, 0, Math.PI * 2);
  ctx.fillStyle = "#34d399"; ctx.fill();

  // ── True robot ─────────────────────────────────────────────────────────────
  ctx.save();
  ctx.translate(robot.x, robot.y);
  ctx.rotate(robot.theta);
  ctx.beginPath();
  ctx.moveTo(12,0); ctx.lineTo(-8,7); ctx.lineTo(-8,-7); ctx.closePath();
  ctx.fillStyle = "#f87171"; ctx.strokeStyle = "#fca5a5"; ctx.lineWidth = 1.5;
  ctx.fill(); ctx.stroke();
  ctx.restore();
}

// ─── WASD input ───────────────────────────────────────────────────────────────
const keys = {};
document.addEventListener("keydown", e => {
  const k = e.key.toLowerCase();
  if (["w","a","s","d"].includes(k)) { e.preventDefault(); keys[k] = true; }
});
document.addEventListener("keyup", e => {
  keys[e.key.toLowerCase()] = false;
});

let lastTime = 0;
function gameLoop(ts) {
  requestAnimationFrame(gameLoop);
  if (ts - lastTime < 80) return;
  lastTime = ts;
  const fwdPx = ((keys["w"] ? 1 : 0) - (keys["s"] ? 1 : 0)) * mmToPx(100);
  const turn  = (keys["d"] ? 0.13 : 0) - (keys["a"] ? 0.13 : 0);
  if (fwdPx !== 0 || turn !== 0) runStep(fwdPx, turn);
}
requestAnimationFrame(gameLoop);

// ─── Reset ────────────────────────────────────────────────────────────────────
document.getElementById("btn-reset").addEventListener("click", () => {
  robot = { x: WALL.left + ROOM_PX/2, y: WALL.top + ROOM_PX/2, theta: 0 };
  estimate = { x: robot.x, y: robot.y };
  stepCount=0; errorVal=0; history=[]; beamReadings=[null,null,null,null];
  initParticles(); updateUI(0); draw();
});

// ─── Sliders ──────────────────────────────────────────────────────────────────
document.getElementById("sl-fov").addEventListener("input", e => {
  BEAM_WIDTH_DEG = +e.target.value;
  document.getElementById("lbl-fov").textContent = BEAM_WIDTH_DEG + "°";
  document.getElementById("s-fov").textContent   = BEAM_WIDTH_DEG + "°";
  draw();
});
document.getElementById("sl-range").addEventListener("input", e => {
  SENSOR_RANGE_IN = +e.target.value;
  document.getElementById("lbl-range").textContent = SENSOR_RANGE_IN;
  document.getElementById("s-range").textContent   = SENSOR_RANGE_IN + " in";
  draw();
});
document.getElementById("sl-noise").addEventListener("input", e => {
  SENSOR_NOISE_IN = +e.target.value;
  document.getElementById("lbl-noise").textContent = (+e.target.value).toFixed(1);
});

document.getElementById("sl-gyro").addEventListener("input", e => {
  GYRO_NOISE = +e.target.value;
  document.getElementById("lbl-gyro").textContent = (+e.target.value).toFixed(3);
});

// ─── Boot ─────────────────────────────────────────────────────────────────────
initParticles();
draw();
updateUI(0, 0);
</script>
</body>
</html>
