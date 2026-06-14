// ═══════════════════════════════════════════════════════════════
// Sync openfootball → Firebase (Quiniela Chida Mundial 2026)
// Corre cada 1 hr vía GitHub Actions y actualiza los marcadores
// oficiales en TODAS las comunidades activas.
// ═══════════════════════════════════════════════════════════════

import admin from 'firebase-admin';

// ─── Config ────────────────────────────────────────────────────
const OPENFOOTBALL_URL = 'https://raw.githubusercontent.com/openfootball/worldcup.json/master/2026/worldcup.json';
const DRY_RUN = process.env.DRY_RUN === 'true';

// ─── Validar variables de entorno ──────────────────────────────
const serviceAccountStr = process.env.FIREBASE_SERVICE_ACCOUNT;
const databaseURL = process.env.FIREBASE_DATABASE_URL;

if (!serviceAccountStr) {
  console.error('❌ FIREBASE_SERVICE_ACCOUNT no está configurado');
  process.exit(1);
}
if (!databaseURL) {
  console.error('❌ FIREBASE_DATABASE_URL no está configurado');
  process.exit(1);
}

let serviceAccount;
try {
  serviceAccount = JSON.parse(serviceAccountStr);
} catch (e) {
  console.error('❌ FIREBASE_SERVICE_ACCOUNT no es JSON válido:', e.message);
  process.exit(1);
}

// ─── Datos del Mundial (mismo que index.html) ──────────────────
const GRP = {
  A: ['México', 'Corea del Sur', 'Sudáfrica', 'Rep. Checa'],
  B: ['Canadá', 'Suiza', 'Qatar', 'Bosnia y Herz.'],
  C: ['Brasil', 'Marruecos', 'Escocia', 'Haití'],
  D: ['EE. UU.', 'Paraguay', 'Australia', 'Turquía'],
  E: ['Alemania', 'Ecuador', 'Costa de Marfil', 'Curazao'],
  F: ['Países Bajos', 'Japón', 'Túnez', 'Suecia'],
  G: ['Bélgica', 'Irán', 'Egipto', 'Nueva Zelanda'],
  H: ['España', 'Uruguay', 'Arabia Saudita', 'Cabo Verde'],
  I: ['Francia', 'Senegal', 'Noruega', 'Irak'],
  J: ['Argentina', 'Austria', 'Argelia', 'Jordania'],
  K: ['Portugal', 'Colombia', 'Uzbekistán', 'R.D. Congo'],
  L: ['Inglaterra', 'Croacia', 'Panamá', 'Ghana'],
};
const GKEYS = Object.keys(GRP);
const GROUP_MATCH_PAIRS = [[0,1],[2,3],[0,2],[1,3],[0,3],[1,2]];

const OF_TEAM_MAP = {
  'México':'Mexico','Corea del Sur':'South Korea','Sudáfrica':'South Africa','Rep. Checa':'Czech Republic',
  'Canadá':'Canada','Suiza':'Switzerland','Qatar':'Qatar','Bosnia y Herz.':'Bosnia & Herzegovina',
  'Brasil':'Brazil','Marruecos':'Morocco','Escocia':'Scotland','Haití':'Haiti',
  'EE. UU.':'USA','Paraguay':'Paraguay','Australia':'Australia','Turquía':'Turkey',
  'Alemania':'Germany','Ecuador':'Ecuador','Costa de Marfil':'Ivory Coast','Curazao':'Curaçao',
  'Países Bajos':'Netherlands','Japón':'Japan','Túnez':'Tunisia','Suecia':'Sweden',
  'Bélgica':'Belgium','Irán':'Iran','Egipto':'Egypt','Nueva Zelanda':'New Zealand',
  'España':'Spain','Uruguay':'Uruguay','Arabia Saudita':'Saudi Arabia','Cabo Verde':'Cape Verde',
  'Francia':'France','Senegal':'Senegal','Noruega':'Norway','Irak':'Iraq',
  'Argentina':'Argentina','Austria':'Austria','Argelia':'Algeria','Jordania':'Jordan',
  'Portugal':'Portugal','Colombia':'Colombia','Uzbekistán':'Uzbekistan','R.D. Congo':'DR Congo',
  'Inglaterra':'England','Croacia':'Croatia','Panamá':'Panama','Ghana':'Ghana',
};
const OF_TEAM_MAP_REV = {};
for (const es in OF_TEAM_MAP) {
  OF_TEAM_MAP_REV[OF_TEAM_MAP[es].toLowerCase()] = es;
}

function ofToSpanish(name) {
  if (!name) return null;
  if (/winner|loser|path|tbd|undecided|runner|place|group\s/i.test(name)) return null;
  return OF_TEAM_MAP_REV[name.trim().toLowerCase()] || null;
}

function ofMatchScore(m) {
  if (!m) return null;
  if (m.score) {
    if (Array.isArray(m.score.ft) && m.score.ft.length === 2) return { a: m.score.ft[0], b: m.score.ft[1] };
    if (Array.isArray(m.score) && m.score.length === 2) return { a: m.score[0], b: m.score[1] };
  }
  if (typeof m.score1 === 'number' && typeof m.score2 === 'number') return { a: m.score1, b: m.score2 };
  return null;
}

// ─── Parsear openfootball ──────────────────────────────────────
function ofParseGroupMatches(data) {
  const out = [];
  const matches = data.matches || [];
  for (const m of matches) {
    if (!m.group) continue;
    const gMatch = /Group\s+([A-L])/i.exec(m.group);
    if (!gMatch) continue;
    const g = gMatch[1].toUpperCase();
    const t1es = ofToSpanish(m.team1);
    const t2es = ofToSpanish(m.team2);
    if (!t1es || !t2es) continue;
    const groupTeams = GRP[g];
    if (!groupTeams) continue;
    const idx1 = groupTeams.indexOf(t1es);
    const idx2 = groupTeams.indexOf(t2es);
    if (idx1 < 0 || idx2 < 0) continue;
    let mi = -1, swapped = false;
    for (let k = 0; k < GROUP_MATCH_PAIRS.length; k++) {
      const [a, b] = GROUP_MATCH_PAIRS[k];
      if (a === idx1 && b === idx2) { mi = k; swapped = false; break; }
      if (a === idx2 && b === idx1) { mi = k; swapped = true; break; }
    }
    if (mi < 0) continue;
    const sc = ofMatchScore(m);
    if (!sc || sc.a == null || sc.b == null) continue;
    const score = swapped ? { a: sc.b, b: sc.a } : sc;
    out.push({ group: g, mi, score, t1es, t2es });
  }
  return out;
}

// ─── Helper: iterar respetando índices (mismo que en index.html) ─
function iterIndexed(v, maxLen, cb) {
  if (!v) return;
  if (Array.isArray(v)) {
    v.forEach((item, i) => { if (i < maxLen) cb(i, item); });
  } else {
    for (const k in v) {
      const i = parseInt(k, 10);
      if (!isNaN(i) && i >= 0 && i < maxLen) cb(i, v[k]);
    }
  }
}

// ─── MAIN ──────────────────────────────────────────────────────
async function main() {
  console.log('═══════════════════════════════════════════════');
  console.log('  Sync Mundial 2026 — ' + new Date().toISOString());
  console.log('  DRY_RUN:', DRY_RUN ? 'YES (no escribe)' : 'NO (escribirá a Firebase)');
  console.log('═══════════════════════════════════════════════');

  // 1) Init Firebase Admin
  admin.initializeApp({
    credential: admin.credential.cert(serviceAccount),
    databaseURL,
  });
  const db = admin.database();

  // 2) Fetch openfootball
  console.log('\n→ Fetching openfootball...');
  const res = await fetch(OPENFOOTBALL_URL, { cache: 'no-store' });
  if (!res.ok) throw new Error('openfootball respondió ' + res.status);
  const data = await res.json();
  const parsed = ofParseGroupMatches(data);
  const withScores = parsed.filter(m => m.score && m.score.a != null && m.score.b != null);
  console.log(`  ✓ ${withScores.length} marcadores oficiales disponibles`);

  if (withScores.length === 0) {
    console.log('\n📭 No hay resultados oficiales disponibles. Nada que sincronizar.');
    process.exit(0);
  }

  // 3) Listar comunidades
  console.log('\n→ Listando comunidades...');
  const commsSnap = await db.ref('communities').once('value');
  const communities = commsSnap.val() || {};
  const cids = Object.keys(communities);
  console.log(`  ✓ ${cids.length} comunidades encontradas`);

  // 4) Sincronizar cada comunidad
  let totalChanges = 0;
  let totalCommUpdated = 0;
  const changesByComm = {};

  for (const cid of cids) {
    const commName = communities[cid].name || cid;
    const resultsSnap = await db.ref('community_results/' + cid).once('value');
    const current = resultsSnap.val() || {};
    const grupos = current.grupos || {};

    const updates = {};
    let commChanges = 0;

    for (const m of withScores) {
      // Leer marcador actual respetando sparse arrays
      const groupData = grupos[m.group];
      const matches = groupData?.matches;
      let cur = { a: null, b: null };
      if (matches) {
        if (Array.isArray(matches)) {
          cur = matches[m.mi] || { a: null, b: null };
        } else {
          cur = matches[String(m.mi)] || matches[m.mi] || { a: null, b: null };
        }
      }
      const curA = cur && cur.a != null && cur.a !== '' ? Number(cur.a) : null;
      const curB = cur && cur.b != null && cur.b !== '' ? Number(cur.b) : null;
      const newA = Number(m.score.a);
      const newB = Number(m.score.b);

      const isNew = curA === null || curB === null;
      const isDifferent = !isNew && (curA !== newA || curB !== newB);

      if (isNew || isDifferent) {
        const path = `community_results/${cid}/grupos/${m.group}/matches/${m.mi}`;
        updates[path] = { a: newA, b: newB };
        commChanges++;
        if (!changesByComm[commName]) changesByComm[commName] = [];
        changesByComm[commName].push(
          `Grupo ${m.group}: ${m.t1es} ${newA}-${newB} ${m.t2es} ${isNew ? '(NUEVO)' : `(CAMBIO de ${curA}-${curB})`}`
        );
      }
    }

    if (commChanges > 0) {
      console.log(`\n  📍 ${commName}: ${commChanges} cambio(s)`);
      if (!DRY_RUN) {
        await db.ref().update(updates);
        // También actualiza timestamp del último sync
        await db.ref(`community_results/${cid}/_lastSyncedAt`).set(Date.now());
        console.log(`    ✓ Guardado en Firebase`);
      } else {
        console.log(`    ⚠️  DRY_RUN: no se guardó`);
      }
      totalChanges += commChanges;
      totalCommUpdated++;
    }
  }

  // 5) También actualiza el path legacy 'admin' (compat con super-admin)
  if (!DRY_RUN) {
    const adminUpdates = {};
    const adminSnap = await db.ref('admin').once('value');
    const adminCur = adminSnap.val() || {};
    const adminGrupos = adminCur.grupos || {};
    for (const m of withScores) {
      const groupData = adminGrupos[m.group];
      const matches = groupData?.matches;
      let cur = { a: null, b: null };
      if (matches) {
        if (Array.isArray(matches)) {
          cur = matches[m.mi] || { a: null, b: null };
        } else {
          cur = matches[String(m.mi)] || matches[m.mi] || { a: null, b: null };
        }
      }
      const curA = cur && cur.a != null && cur.a !== '' ? Number(cur.a) : null;
      const curB = cur && cur.b != null && cur.b !== '' ? Number(cur.b) : null;
      const newA = Number(m.score.a);
      const newB = Number(m.score.b);
      if (curA === null || curB === null || curA !== newA || curB !== newB) {
        adminUpdates[`admin/grupos/${m.group}/matches/${m.mi}`] = { a: newA, b: newB };
      }
    }
    if (Object.keys(adminUpdates).length > 0) {
      await db.ref().update(adminUpdates);
      console.log(`\n  📍 Path legacy 'admin': ${Object.keys(adminUpdates).length} cambio(s) guardado(s)`);
    }
  }

  // 6) Resumen
  console.log('\n═══════════════════════════════════════════════');
  if (totalChanges === 0) {
    console.log('  ✓ Todo está al día. Nada que sincronizar.');
  } else {
    console.log(`  ✓ ${totalChanges} cambio(s) en ${totalCommUpdated} comunidad(es)`);
    console.log('\n  Detalle:');
    for (const [comm, changes] of Object.entries(changesByComm)) {
      console.log(`    ▸ ${comm}:`);
      changes.forEach(c => console.log(`        • ${c}`));
    }
  }
  console.log('═══════════════════════════════════════════════');

  process.exit(0);
}

main().catch(err => {
  console.error('\n❌ Error fatal:', err.message);
  console.error(err.stack);
  process.exit(1);
});
