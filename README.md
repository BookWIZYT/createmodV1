

addEventListener("fetch", event => {
  event.respondWith(handleRequest(event.request));
});

/**
 * worker.js - serves index.html and create.js to the browser.
 * - GET /           -> returns simple index.html (loads /create.js)
 * - GET /create.js  -> returns your simulator JS (as application/javascript)
 *
 * Important: Put your browser-side simulator code into the CREATE_SIM_CODE string below.
 * If your simulator includes backticks (`) or ${...} sequences, you must escape them.
 */

async function handleRequest(request) {
  const url = new URL(request.url);
  if (request.method !== "GET") {
    return new Response("Only GET supported", { status: 405 });
  }

  if (url.pathname === "/" || url.pathname === "/index.html") {
    return new Response(INDEX_HTML, {
      headers: { "content-type": "text/html; charset=utf-8" },
    });
  }

  if (url.pathname === "/create.js") {
    return new Response(CREATE_SIM_CODE, {
      headers: { "content-type": "application/javascript; charset=utf-8" },
    });
  }

  // optional blank proxy page if you need it (served as HTML)
  if (url.pathname === "/blank") {
    return new Response(BLANK_PAGE, {
      headers: { "content-type": "text/html; charset=utf-8" },
    });
  }

  // default 404
  return new Response("Not found", { status: 404 });
}

/* ---------------------------------------------------------------------------
   INDEX_HTML - small loader page that will load /create.js
   You can edit this to include wrappers or to provide fake getBlock/getInventory APIs
   for testing in the browser.
   --------------------------------------------------------------------------- */
const INDEX_HTML = `<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Create Simulator Test</title>
  <style>
    body { font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial; background:#0b1222; color:#fff; padding:24px; }
    header { display:flex; gap:12px; align-items:center; }
    pre { background: rgba(255,255,255,0.04); padding:12px; border-radius:8px; overflow:auto; max-height:300px; }
    .panel { margin-top:16px; }
    button { background:#06d6a0; border:none; padding:8px 12px; border-radius:6px; cursor:pointer; color:#002; }
  </style>
</head>
<body>
  <header>
    <h1>CREATE Simulator (client)</h1>
    <div style="margin-left:auto">
      <button id="showState">Show Sim State</button>
    </div>
  </header>

  <section class="panel">
    <p>This page loads <code>/create.js</code>. The simulator expects some host functions to exist:
      <code>window.getBlock(x,y,z)</code>, <code>window.getInventoryItem(...)</code>, <code>window.chat(msg)</code> etc.
      For a quick test we provide simple mock implementations below.</p>
  </section>

  <section class="panel">
    <h3>Console / Logs</h3>
    <pre id="logArea">Loading create.js…</pre>
  </section>

  <script>
    // Small logger to show messages on the page (simulator will use window.chat)
    (function(){
      const logArea = document.getElementById('logArea');
      window._simLog = function (msg) {
        const t = new Date().toLocaleTimeString();
        logArea.textContent = t + ' ' + String(msg) + '\\n' + logArea.textContent;
      };
      // Provide placeholders for required game APIs the simulator expects.
      // Replace these with your real engine hooks when available.
      window.getBlock = function(x,y,z) {
        // For testing, return null / no machine. You can place test "blocks" by setting window._blocks
        const key = [Math.floor(x),Math.floor(y),Math.floor(z)].join(',');
        return (window._blocks && window._blocks[key]) || null;
      };
      window.getInventoryItem = function(x,y,z,slot) {
        // simple mock: return stored inventory if present
        const key = [Math.floor(x),Math.floor(y),Math.floor(z)].join(',');
        if(window._inventories && window._inventories[key] && window._inventories[key][slot]) {
          return window._inventories[key][slot];
        }
        return null;
      };
      window.setInventoryItem = function(x,y,z,slot,itemId,count) {
        const key = [Math.floor(x),Math.floor(y),Math.floor(z)].join(',');
        window._inventories = window._inventories || {};
        window._inventories[key] = window._inventories[key] || {};
        window._inventories[key][slot] = itemId ? { id: itemId, count: count } : null;
        window._simLog('inventory['+key+']['+slot+'] = ' + JSON.stringify(window._inventories[key][slot]));
      };
      window.chat = function(msg){ window._simLog(msg); };
      // expose a small helper to place machine-mapped test blocks in the world
      window.placeTestBlock = function(x,y,z,id) {
        window._blocks = window._blocks || {};
        window._blocks[[x,y,z].join(',')] = id;
        window._simLog('placed block ' + id + ' at ' + [x,y,z].join(','));
      };
    })();
  </script>

  <!-- load the simulator script from the worker -->
  <script src="/create.js"></script>

  <script>
    // UI: show some state from simulator if available
    document.getElementById('showState').addEventListener('click', () => {
      try {
        const ns = window.CreateSim && window.CreateSim.debugSnapshot ? window.CreateSim.debugSnapshot() : {msg:'no snapshot available'};
        document.getElementById('logArea').textContent = JSON.stringify(ns, null, 2) + '\\n' + document.getElementById('logArea').textContent;
      } catch (e) {
        document.getElementById('logArea').textContent = 'Error getting snapshot: ' + e.message;
      }
    });
  </script>
</body>
</html>`;

/* ---------------------------------------------------------------------------
   BLANK_PAGE - optional page that serves an iframe to your worker proxy (if you use /blank)
   --------------------------------------------------------------------------- */
const BLANK_PAGE = `<!doctype html><html><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>Blank Proxy</title></head><body style="margin:0"><iframe src="/p" style="width:100vw;height:100vh;border:none"></iframe></body></html>`;

/* ---------------------------------------------------------------------------
   CREATE_SIM_CODE - place your browser-side simulator code here (as a JS string).
   IMPORTANT:
   - Do not include raw template backticks (`) or ${...} inside the code unless escaped.
   - Remove or rewrite any block-comments that contain angle brackets (< or >).
   - The code below is an example small shim derived from your script; replace it with
     your full simulator (sanitized if it contains ` or ${}).
   --------------------------------------------------------------------------- */
const CREATE_SIM_CODE = `
// Create Simulator (client-side) - paste your simulator here.
// NOTE: This file runs in the browser. It expects helper functions like:
//   window.getBlock(x,y,z), window.getInventoryItem(x,y,z,slot), window.setInventoryItem(...), and window.chat(msg).
// For debugging, this script exposes window.CreateSim.debugSnapshot().

(function(){
  // Small wrapper so user can access simulator state for debugging
  const CreateSim = {
    CONFIG: {
      TICK_RATE_MS: 100,
      SCAN_RADIUS: 6,
      BASE_STRESS_CAPACITY: 256,
      MOTOR_SPEED: 128
    },
    WorldState: { network: new Map(), currentStress:0, stressCapacity:0 },
    PROCESSING_QUEUE: new Map(),
    init() {
      if (this._interval) clearInterval(this._interval);
      this._interval = setInterval(()=>this.tick(), this.CONFIG.TICK_RATE_MS);
      if(window.chat) window.chat('CreateSim started (client-side).');
    },
    stop() {
      if (this._interval) clearInterval(this._interval);
      this._interval = null;
      if(window.chat) window.chat('CreateSim stopped.');
    },
    debugSnapshot() {
      const net = {};
      this.WorldState.network.forEach((v,k)=>{ net[k]=Object.assign({},v); });
      return { network: net, stress: this.WorldState.currentStress, capacity: this.WorldState.stressCapacity };
    },
    tick() {
      // very small demo: scan for a 'motor' at player's position and propagate
      const p = window.player || (window.game && window.game.player) || {x:0,y:64,z:0};
      // simple scan: look only for diamond_block within scan radius
      this.WorldState.network.clear();
      this.WorldState.currentStress = 0;
      this.WorldState.stressCapacity = 0;
      for(let x = Math.floor(p.x)-this.CONFIG.SCAN_RADIUS; x <= Math.floor(p.x)+this.CONFIG.SCAN_RADIUS; x++){
        for(let y = Math.floor(p.y)-1; y <= Math.floor(p.y)+1; y++){
          for(let z = Math.floor(p.z)-this.CONFIG.SCAN_RADIUS; z <= Math.floor(p.z)+this.CONFIG.SCAN_RADIUS; z++){
            const id = window.getBlock ? window.getBlock(x,y,z) : null;
            if(id === 'minecraft:diamond_block') {
              const posId = [x,y,z].join(',');
              this.WorldState.network.set(posId, { x,y,z, type:'MOTOR', speed:this.CONFIG.MOTOR_SPEED, isPowered:true });
              this.WorldState.stressCapacity += this.CONFIG.BASE_STRESS_CAPACITY * 5;
            }
            // add other machines or test anchors as you need...
          }
        }
      }
      // Mock propagation: each non-motor becomes a shaft with same speed if adjacent
      this.WorldState.network.forEach((block, id)=>{
        for(const v of [[1,0,0],[-1,0,0],[0,0,1],[0,0,-1],[0,1,0],[0,-1,0]]) {
          const nx=block.x+v[0], ny=block.y+v[1], nz=block.z+v[2];
          const nid=[nx,ny,nz].join(',');
          const b = window.getBlock ? window.getBlock(nx,ny,nz) : null;
          if(b === 'minecraft:stick') {
            this.WorldState.network.set(nid, { x:nx,y:ny,z:nz, type:'SHAFT', speed:block.speed, isPowered:true });
          }
        }
      });
      // Demo processing: run one tick of processing queue (no recipes implemented in this demo)
      // Expose snapshot to window for debug
      window.CreateSim = window.CreateSim || {};
      window.CreateSim.debugSnapshot = () => this.debugSnapshot();
    }
  };
  CreateSim.init();
  // Keep a reference globally
  window.CreateSim = CreateSim;
})();
`;



