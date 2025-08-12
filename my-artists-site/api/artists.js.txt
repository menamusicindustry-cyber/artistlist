// /api/artists.js
const path = require('path');
const fs = require('fs');
const XLSX = require('xlsx');

// Your Excel headers (row 1) — change only if your headers differ
const COL = {
  firstName: 'First Name',
  lastName:  'Last Name',
  igExtract: 'Instagram Extract',
  link:      'Link',
  type:      'Artist Type',
  country:   'Country'
};

// Only send these to the browser
const PUBLIC_FIELDS = ['name','type','country','igExtract','link'];

let CACHE = null, MTIME = 0;

function loadExcel() {
  const filePath = path.join(process.cwd(), 'data', 'ArtistList.xlsx');
  const st = fs.statSync(filePath);
  if (CACHE && st.mtimeMs === MTIME) return CACHE; // reuse if file unchanged

  const wb = XLSX.read(fs.readFileSync(filePath));
  const ws = wb.Sheets[wb.SheetNames[0]];
  const json = XLSX.utils.sheet_to_json(ws, { defval: '' });

  const s = v => (v == null ? '' : String(v)).trim();
  CACHE = json.map(r => ({
    name: `${s(r[COL.firstName])} ${s(r[COL.lastName])}`.trim(),
    type: s(r[COL.type]),
    country: s(r[COL.country]),
    igExtract: s(r[COL.igExtract]),
    link: s(r[COL.link])
  })).filter(r => r.name);
  MTIME = st.mtimeMs;
  return CACHE;
}

module.exports = async (req, res) => {
  // Allow your page to call this (you can set a specific domain later)
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, OPTIONS');
  if (req.method === 'OPTIONS') return res.status(204).end();

  const all = loadExcel();
  const q = String(req.query.q || '').toLowerCase();
  const types = [].concat(req.query.type || []).map(x => String(x).toLowerCase());
  const countries = [].concat(req.query.country || []).map(x => String(x).toLowerCase());

  let filtered = all.filter(r =>
    (!q || r.name.toLowerCase().includes(q)) &&
    (!types.length || types.includes((r.type||'').toLowerCase())) &&
    (!countries.length || countries.includes((r.country||'').toLowerCase()))
  );

  // keep it snappy
  filtered = filtered.slice(0, 300);

  // send only public fields
  const results = filtered.map(r => {
    const out = {};
    for (const k of PUBLIC_FIELDS) out[k] = r[k] || '';
    return out;
  });

  res.status(200).json({ count: results.length, results });
};
