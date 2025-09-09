# n8n-parasedata1
const formData = $('On form submission1').first().json ?? {};
const defaults = $('Get Article Metadata1').first().json ?? {};
const countryItems = $('Load Country Codes').all();              // 238 items
const countryTable = countryItems.map(i => i.json);              // [{country_code, country_name}, ...]

// --------- config ----------
const allowedKeys = ['url','focus_keyword','country','period_length','target_keywords','language'];

const keyMapping = {
  'URL': 'url',
  'Focus Keyword': 'focus_keyword',
  'Country': 'country',
  'Period Length': 'period_length',
  'Target Keywords': 'target_keywords',
  'Language': 'language',
};

// --------- helpers ----------
const toArray = (v) => {
  if (Array.isArray(v)) return v.filter(Boolean).map(s => String(s).trim()).filter(Boolean);
  if (v === undefined || v === null) return [];
  return String(v)
    .split(',')
    .map(s => s.trim())
    .filter(Boolean);
};

function normalizeCountry(input, table) {
  if (!input) return '';
  const s = String(input).trim();

  // if already a 2-letter code, trust it if present in table
  if (/^[A-Za-z]{2}$/.test(s)) {
    const hit = table.find(r => r.country_code.toLowerCase() === s.toLowerCase());
    return hit ? hit.country_code : s.toLowerCase();
  }

  // --- START: ALIAS MAPPING LOGIC (NOW IN VALID JAVASCRIPT) ---
  const aliasMap = {
    'us': 'united states',
    'usa': 'united states',
    'united states of america': 'united states',
    'uk': 'united kingdom',
    'gb': 'united kingdom',
    'uae': 'united arab emirates',
    'bharat': 'india'
  };

  const normalizedInput = s.toLowerCase();
  const canonicalName = aliasMap[normalizedInput] || normalizedInput;
  // --- END: ALIAS MAPPING LOGIC ---

  // try name match (case-insensitive) --- MODIFIED TO USE ALIAS ---
  const hitByName = table.find(r => r.country_name.toLowerCase() === canonicalName);
  if (hitByName) return hitByName.country_code;

  // try partial name match
  const hitPartial = table.find(r => r.country_name.toLowerCase().includes(s.toLowerCase()));
  if (hitPartial) return hitPartial.country_code;

  return s; // fallback (leave as-is)
}

function getFormValue(stdKey) {
  const formKey = Object.keys(keyMapping).find(k => keyMapping[k] === stdKey);
  return formKey ? formData[formKey] : undefined;
}

// --------- merge logic ----------
const mergedData = allowedKeys.reduce((acc, key) => {
  const formVal = getFormValue(key);
  const defaultVal = defaults[key];

  if (key === 'target_keywords') {
    const arr = toArray(formVal ?? defaultVal);
    acc[key] = arr;
  } else if (key === 'country') {
    const countryIn = (formVal ?? defaultVal ?? '').toString().trim();
    acc[key] = normalizeCountry(countryIn, countryTable);
  } 
  else if (key === 'url') {
    let urlValue = (formVal ?? defaultVal ?? '').toString().trim();
    if (urlValue && !urlValue.startsWith('http')) {
      urlValue = 'https://' + urlValue;
    }
    acc[key] = urlValue;
  }
  else {
    acc[key] = (formVal !== undefined && formVal !== null && String(formVal) !== '')
      ? formVal
      : (defaultVal ?? '');
  }

  return acc;
}, {});

// --------- validation ----------
const required = ['url','focus_keyword','country','period_length','language']; // target_keywords may be []
const emptyFields = required.filter(k => {
  const v = mergedData[k];
  return v === undefined || v === null || String(v).trim() === '';
});

if (emptyFields.length > 0) {
  throw new Error(`Required fields are missing: ${emptyFields.join(', ')}`);
}

// --------- output ----------
return [{ json: { article_data: mergedData } }];
