// ==UserScript==
// @name         Cinemana Ultimate Auto Subtitle Engine – German & Arabic PRO
// @name:ar      سينمانا — محرك الترجمات التلقائي (ألماني وعربي) PRO
// @namespace    cinemana-ultimate-auto-subtitle-engine
// @version      1.4.3
// @description  ترجمتان متزامنتان مع خيار Deutsch للألمانية فقط، مع رفض عناوين التوصيات.
// @author       Cinemana Studio
// @match        *://cinemana.shabakaty.com/*
// @match        *://www.cinemana.shabakaty.com/*
// @match        *://*.cinemana.shabakaty.com/*
// @icon         data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHJlY3Qgd2lkdGg9IjI0IiBoZWlnaHQ9IjI0IiBmaWxsPSIjZGY4YTFjIi8+PHBhdGggZD0iTTMgMTZoMTh2M0gzem0wLTNoMTJ2M0gzem0wLThoMTZ2M0gzem0wIDNoMXYzSDN6IiBmaWxsPSIjMWYxYTExIi8+PC9zdmc+
// @grant        GM_xmlhttpRequest
// @grant        GM_getValue
// @grant        GM_setValue
// @grant        GM_registerMenuCommand
// @grant        GM_addStyle
// @grant        GM_notification
// @connect      api.opensubtitles.com
// @connect      *.opensubtitles.com
// @connect      *.opensubtitles.org
// @connect      api.subdl.com
// @connect      dl.subdl.com
// @connect      api.subsource.net
// @connect      subsource.net
// @connect      *.podnapisi.net
// @connect      www.subtitlecat.com
// @connect      subtitlecat.com
// @run-at       document-idle
// @noframes
// @license      MIT
// ==/UserScript==
(function () {
  'use strict';
  if (window.__CMS_AUTO_SUBS_LOADED__) return;
  window.__CMS_AUTO_SUBS_LOADED__ = true;
  /* ═══════════════════════════════════════════════════════════════
     0) الإعدادات — ضع مفاتيح API هنا (لن تظهر في الواجهة أو السجلات)
     ═══════════════════════════════════════════════════════════════ */
  const VERSION = '1.4.3';
  const NS = 'cms.ultimate.autosub.';
  const IS_IOS = /iPad|iPhone|iPod/.test(navigator.userAgent) ||
    (navigator.platform === 'MacIntel' && navigator.maxTouchPoints > 1);
  const API = {
    // مفتاح مجاني: opensubtitles.com → حسابك → تبويب API
    openSubtitles: {
      apiKey: 'خلي المفتاح الاول هنا وركز ع الاسم',
      email: 'apikey@localhost',
      username: '',             // اختياري: حساب OpenSubtitles، لا تضع مفتاح API هنا
      password: '',             // اختياري: كلمة مرور الحساب للحد الأعلى من التنزيلات
      apiVersion: '0.4.1',
      userAgent: 'CinemanaUltimateAutoSubs v' + VERSION,
      base: 'https://api.opensubtitles.com/api/v1',
    },
    // مفتاح مجاني: subdl.com → لوحة الحساب (2000 طلب/يوم)
    subdl: {
      apiKey: 'والمفتاح الثاني هنا نفس الشي',
      base: 'https://api.subdl.com/api/v1',
      dlBase: 'https://dl.subdl.com',
    },
    // مفتاح: subsource.net → الملف الشخصي → API
    subsource: {
      apiKey: 'والثالث هنا ',
      base: 'https://api.subsource.net/api/v1',
    },
    // إعداد قديم محفوظ للتوافق فقط؛ Podnapisi متوقف ولا تُرسل إليه طلبات.
    podnapisi: {
      searchUrl: 'https://www.podnapisi.net/ppodnapisi/search',
      dlBase: 'https://www.podnapisi.net/subtitles/',
    },
    subtitleCat: {
      base: 'https://www.subtitlecat.com/', // HTML public, not a stable API
    },
  };
  const DEFAULTS = {
    auto: true,                 // بحث تلقائي عند اكتشاف عمل جديد
    show: true,                 // إظهار الترجمة
    langs: { de: true, ar: true },
    primary: 'de',              // الألمانية هي الرئيسية
    secondary: 'ar',            // والعربية ثانوية
    engineOrder: ['openSubtitles', 'subdl', 'subsource', 'subtitleCat', 'podnapisi'],
    engineOn: { openSubtitles: true, subdl: true, subsource: true, subtitleCat: true, podnapisi: false },
    minScore: 70,
    offset: 0,                  // إزاحة الزمان (ثوانٍ)
    fontSize: 22,
    deColor: '#ffffff',
    arColor: '#ffffff',
    bgOpacity: 0.58,
    arabicBgOpacity: 0.94,
    warmColor: '#f4eb29',       // إعداد قديم محفوظ للتوافق فقط
    freshColor: '#83f54b',      // لا يُستخدم في النمط الأبيض الحالي
    subtitleLayout: 'cinematic',
    highlightVocabulary: false,
    subtitleGap: 6,            // تباعد صغير، لا تلاصق ولا فجوة كبيرة
    useOverlay: true,
    keepNativeTracks: false,
    nativeFullscreenFallback: true,
    maxRetries: 2,
    panelOpen: false,
    showLog: false,
  };
  const LANG_NAME = { de: 'الألمانية', ar: 'العربية', en: 'الإنجليزية' };
  const ISO3 = { deu: 'de', ara: 'ar', eng: 'en', ger: 'de', heb: 'he', fre: 'fr', fra: 'fr', ita: 'it', spa: 'es' };
  const ENGINE_TIE = { openSubtitles: 4, subdl: 3, subsource: 2, subtitleCat: 1, podnapisi: 0 };
  const BACKOFF = [5000, 15000];
  /* ───────────────────── أدوات عامة ───────────────────── */
  const $ = s => document.querySelector(s);
  const $$ = s => Array.from(document.querySelectorAll(s));
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const hostOf = u => { try { return new URL(u).host; } catch (e) { return u.slice(0, 40); } };
  const clamp = (v, a, b) => Math.min(b, Math.max(a, v));
  const log = {
    _open: false,
    _buf: [],
    push(level, args) {
      const line = args.map(a => (typeof a === 'string' ? a : tryFmt(a)));
      this._buf.push({ t: Date.now(), level, line: line.join(' ') });
      if (this._buf.length > 250) this._buf.shift();
      const fn = level === 'error' ? console.error : level === 'warn' ? console.warn : console.log;
      fn('[CineSubs ' + VERSION + ']', ...line);
    },
    info(...a) { this.push('info', a); },
    warn(...a) { this.push('warn', a); },
    error(...a) { this.push('error', a); },
    // لا نطبع المفاتيح ولا أحرفاً منها في السجل.
  };
  function tryFmt(a) { try { return a instanceof Error ? (a.message || String(a)) : JSON.stringify(a); } catch (e) { return String(a); } }
  function notify(title, msg) {
    try {
      if (typeof GM_notification === 'function') { GM_notification({ title, message: msg }); return; }
    } catch (e) { /* تجاهل */ }
    log.info('🔔', title, msg);
  }
  /* ───────────────────── التخزين (GM مع بديل localStorage) ───────────────────── */
  const store = {
    get(k, def) {
      try {
        if (typeof GM_getValue === 'function') { const v = GM_getValue(NS + k); return v === undefined ? def : v; }
        const r = localStorage.getItem(NS + k);
        return r == null ? def : JSON.parse(r);
      } catch (e) { return def; }
    },
    set(k, v) {
      try {
        if (typeof GM_setValue === 'function') { GM_setValue(NS + k, v); return; }
        localStorage.setItem(NS + k, JSON.stringify(v));
      } catch (e) { log.warn('تعذر الحفظ:', e.message); }
    },
  };
  const savedSettings = store.get('settings', {}) || {};
  const S = Object.assign({}, DEFAULTS, savedSettings);
  S.langs = Object.assign({}, DEFAULTS.langs, S.langs || {});
  S.engineOn = Object.assign({}, DEFAULTS.engineOn, S.engineOn || {});
  S.engineOrder = DEFAULTS.engineOrder.slice();
  S.engineOn.podnapisi = false; // بقي محرك التوافق، لكن نطاقه لا يعمل حالياً.
  // النمط الجديد مقصود: اللون أبيض لكلتا اللغتين حتى لو حُفظ لون سابق.
  S.deColor = '#ffffff'; S.arColor = '#ffffff';
  S.highlightVocabulary = false;
  S.subtitleLayout = 'cinematic';
  S.subtitleGap = clamp(Number(S.subtitleGap) || 6, 3, 14);
  S.minScore = Math.max(70, Number(S.minScore) || 0);
  function saveSettings() { store.set('settings', S); }
  /* ───────────────────── الكاش ─────────────────────
     نتائج البحث: 10 مفاتيح × حتى 60 دقيقة
     نصوص الترجمة: 6 ملفات × حتى 1.5MB               */
  const Cache = {
    search() { return store.get('cache.search.v3', { e: {} }); },
    searchKey(key, langs) { return key + '|langs:' + langs.slice().sort().join(','); },
    setSearch(key, engine, list, langs) {
      const c = this.search();
      const cacheKey = this.searchKey(key, langs);
      const t = Date.now();
      c.e[cacheKey] = c.e[cacheKey] || {};
      c.e[cacheKey][engine] = { t, list };
      c.e[cacheKey].t0 = t;
      if (Object.keys(c.e).length > 10) {
        const oldest = Object.keys(c.e).sort((a, b) => (c.e[a].t0 || 0) - (c.e[b].t0 || 0))[0];
        delete c.e[oldest];
      }
      store.set('cache.search.v3', c);
    },
    getSearch(key, engine, langs) {
      const c = this.search();
      const cacheKey = this.searchKey(key, langs);
      const e = c.e && c.e[cacheKey] && c.e[cacheKey][engine];
      return e && Date.now() - e.t < (e.list.length ? 60 : 10) * 60e3 ? e.list : null;
    },
    subs() { return store.get('cache.subs.v3', { e: {} }); },
    getSub(key, lang) {
      const c = this.subs(), entry = c.e[key + '|' + lang];
      return entry && Date.now() - entry.t < 14 * 86400e3 ? entry : null;
    },
    setSub(key, lang, data) {
      if (!data || !data.text || data.text.length > 1500 * 1024) return;
      const c = this.subs();
      c.e[key + '|' + lang] = { t: Date.now(), text: data.text, title: data.title,
        engine: data.engine, machine: !!data.machine, score: data.score,
        imdbId: data.imdbId || null, year: data.year || null,
        subtitleDuration: data.subtitleDuration || null, matchReason: data.matchReason || '' };
      const ks = Object.keys(c.e);
      if (ks.length > 6) delete c.e[ks.sort((a, b) => c.e[a].t - c.e[b].t)[0]];
      try { store.set('cache.subs.v3', c); } catch (e) { store.set('cache.subs.v3', { e: {} }); }
    },
    clearFor(key) {
      const s = this.search(), c = this.subs();
      for (const searchKey of Object.keys(s.e)) {
        if (searchKey === key || searchKey.startsWith(key + '|langs:')) delete s.e[searchKey];
      }
      for (const lang of ['de', 'ar']) delete c.e[key + '|' + lang];
      store.set('cache.search.v3', s); store.set('cache.subs.v3', c);
    },
    clear() {
      store.set('cache.subs.v3', { e: {} }); store.set('cache.search.v3', { e: {} });
      log.info('تم مسح الكاش');
    },
  };
  /* ───────────────────── HTTP (GM_xmlhttpRequest أو fetch) ───────────────────── */
  function binStrToBytes(str) {
    const n = str.length, b = new Uint8Array(n);
    for (let i = 0; i < n; i++) b[i] = str.charCodeAt(i) & 0xff;
    return b;
  }
  function http(opts) {
    const o = Object.assign({ method: 'GET', headers: {}, body: null, timeout: 25000 }, opts);
    return new Promise((resolve, reject) => {
      let finished = false, request = null;
      const done = (fn, v) => {
        if (finished) return;
        finished = true; clearTimeout(timer); fn(v);
      };
      const timer = setTimeout(() => {
        try { if (request && request.abort) request.abort(); } catch (e) { /* تجاهل */ }
        done(reject, new Error('انتهت مهلة الاتصال — ' + hostOf(o.url)));
      }, o.timeout + 1000);
      function statusError(status) {
        const reason = status === 429 ? ' (تجاوز حد الاستخدام)' :
          status === 406 ? ' (انتهت حصة التنزيل)' :
          (status === 401 || status === 403) ? ' (تصريح API غير صالح أو مفقود)' : '';
        return Object.assign(new Error('HTTP ' + status + reason + ' — ' + hostOf(o.url)), { status });
      }
      if (typeof GM_xmlhttpRequest === 'function') {
        try {
          request = GM_xmlhttpRequest({
            method: o.method, url: o.url,
            headers: o.headers,
            data: o.body == null ? undefined : o.body,
            responseType: 'arraybuffer',
            timeout: o.timeout,
            ontimeout: () => done(reject, new Error('انتهت المهلة — ' + hostOf(o.url))),
            onerror: () => done(reject, new Error('تعذر الوصول إلى ' + hostOf(o.url))),
            onabort: () => done(reject, new Error('أُلغي الاتصال — ' + hostOf(o.url))),
            onload: r => {
              if (!(r.status >= 200 && r.status < 300)) { done(reject, statusError(r.status)); return; }
              const raw = r.response;
              const bytes = raw instanceof ArrayBuffer ? new Uint8Array(raw) :
                ArrayBuffer.isView(raw) ? new Uint8Array(raw.buffer, raw.byteOffset, raw.byteLength) :
                typeof raw === 'string' ? binStrToBytes(raw) : null;
              if (!bytes || !bytes.length) { done(reject, new Error('استجابة فارغة من ' + hostOf(o.url))); return; }
              done(resolve, { status: r.status, bytes });
            },
          });
        } catch (e) { done(reject, new Error('تعذر إرسال الطلب إلى ' + hostOf(o.url))); }
        return;
      }
      // بديل عند غياب GM؛ طلبات الشبكة عبر الأصل الآخر ستحتاج CORS.
      const ctl = new AbortController();
      const t2 = setTimeout(() => ctl.abort(), o.timeout);
      fetch(o.url, {
        method: o.method, headers: o.headers, body: o.body,
        signal: ctl.signal, credentials: 'omit',
      }).then(async r => {
        clearTimeout(t2);
        if (!r.ok) { done(reject, statusError(r.status)); return; }
        const buf = new Uint8Array(await r.arrayBuffer());
        done(resolve, { status: r.status, bytes: buf });
      }).catch(e => {
        clearTimeout(t2);
        done(reject, new Error(e.name === 'AbortError' ? 'انتهت المهلة' : 'تعذر الوصول إلى ' + hostOf(o.url)));
      });
    });
  }
  async function httpJson(opts) {
    const r = await http(opts);
    const s = decodeSubText(r.bytes);
    try { return JSON.parse(s); } catch (e) {
      // بعض الخوادم ترمز بـ latin:
      throw new Error('استجابة غير صالحة من ' + hostOf(opts.url));
    }
  }
  /* ───────────────────── فك الترميز ───────────────────── */
  function cp1252(b) {
    let s = '';
    for (let i = 0; i < b.length; i++) {
      const x = b[i];
      if (x === 10 || x === 9 || x === 13) continue;
      if (x >= 32 && x < 127) s += String.fromCharCode(x);
      else if (x >= 160) s += String.fromCharCode(x); // تقريب لاتيني مقبول كنقطة أخيرة
    }
    return s;
  }
  function decodeSubText(bytes, lang) {
    if (bytes[0] === 0xff && bytes[1] === 0xfe) return new TextDecoder('utf-16le').decode(bytes);
    if (bytes[0] === 0xfe && bytes[1] === 0xff) return new TextDecoder('utf-16be').decode(bytes);
    try {
      const s = new TextDecoder('utf-8', { fatal: true }).decode(bytes);
      if (s.indexOf('\uFFFD') === -1) return s;
    } catch (e) { /* ليس UTF-8 صريحاً */ }
    try { return new TextDecoder(lang === 'ar' ? 'windows-1256' : 'windows-1252').decode(bytes); }
    catch (e) { return cp1252(bytes); }
  }
  /* ───────────────────── inflate (RFC1951) — قارئ ZIP كامل ───────────────────── */
  const Zip = (function () {
    function mkCtx(s, p, end) { return { s, p, end, bit: 0, val: 0 }; }
    function rb(ctx, n) {
      let v = ctx.val, b = ctx.bit;
      while (b < n) {
        if (ctx.p >= ctx.end) throw new Error('نهاية البيانات أثناء فك الضغط');
        v |= ctx.s[ctx.p++] << b; b += 8;
      }
      ctx.val = v >>> n; ctx.bit = b - n;
      return v & ((1 << n) - 1);
    }
    function buildHuff(lens) {
      const counts = new Array(16).fill(0);
      for (const l of lens) counts[l]++;
      const start = new Array(16).fill(0), symByLen = new Array(16);
      let code = 0;
      for (let len = 1; len <= 15; len++) {
        if (len > 1) code = (code + counts[len - 1]) << 1;
        start[len] = code; symByLen[len] = [];
      }
      for (let i = 0; i < lens.length; i++) { const l = lens[i]; if (l) symByLen[l].push(i); }
      return { start, counts, symByLen };
    }
    function readHuff(ctx, h) {
      let code = 0;
      for (let len = 1; len <= 15; len++) {
        code = (code << 1) | rb(ctx, 1);
        const c = h.counts[len];
        if (c && code >= h.start[len] && code < h.start[len] + c) return h.symByLen[len][code - h.start[len]];
      }
      throw new Error('رمز Huffman غير صالح');
    }
    const LEN_BASE = [3,4,5,6,7,8,9,10,11,13,15,17,19,23,27,31,35,43,51,59,67,83,99,115,131,163,195,227,258,0,0];
    const LEN_EXTRA = [0,0,0,0,1,1,2,2,3,3,4,4,5,5,6,6,7,7,8,8,9,9,10,10,11,11,12,12,13,0,0];
    const D_BASE = [1,2,3,4,5,7,9,13,17,25,33,49,65,97,129,193,257,385,513,769,1025,1537,2049,3073,4097,6145,8193,12289,16385,24577];
    const D_EXTRA = [0,0,0,0,1,1,2,2,3,3,4,4,5,5,6,6,7,7,8,8,9,9,10,10,11,11,12,12,13,13];
    function fixedLit() {
      const l = new Array(288);
      for (let i = 0; i < 144; i++) l[i] = 8;
      for (let i = 144; i < 256; i++) l[i] = 9;
      for (let i = 256; i < 280; i++) l[i] = 7;
      for (let i = 280; i < 288; i++) l[i] = 8;
      return buildHuff(l);
    }
    function fixedDist() { return buildHuff(new Array(30).fill(5)); }
    function block(ctx, lit, dist, out) {
      for (;;) {
        const sym = readHuff(ctx, lit);
        if (sym < 256) { out.push(sym); continue; }
        if (sym === 256) return;
        const li = sym - 257;
        if (li >= 29) throw new Error('رمز طول غير صالح');
        const len = LEN_BASE[li] + rb(ctx, LEN_EXTRA[li]);
        const ds = readHuff(ctx, dist);
        if (ds >= 30) throw new Error('رمز مسافة غير صالح');
        const d = D_BASE[ds] + rb(ctx, D_EXTRA[ds]);
        if (d > out.length) throw new Error('مرجع خلفي غير صالح');
        for (let i = 0; i < len; i++) out.push(out[out.length - d]);
        if (out.length > 50 * 1024 * 1024) throw new Error('الحجم كبير جداً');
      }
    }
    function dynTrees(ctx) {
      const hlit = rb(ctx, 5) + 257, hdist = rb(ctx, 5) + 1, hclen = rb(ctx, 4) + 4;
      const order = [16,17,18,0,8,7,9,6,10,5,11,4,12,3,13,2,14,1,15];
      const clLens = new Array(19).fill(0);
      for (let i = 0; i < hclen; i++) clLens[order[i]] = rb(ctx, 3);
      const cl = buildHuff(clLens);
      const total = hlit + hdist, lens = new Array(total).fill(0);
      let i = 0, prev = 0;
      while (i < total) {
        const sym = readHuff(ctx, cl);
        if (sym <= 15) { prev = sym; lens[i++] = sym; }
        else if (sym === 16) {
          const rep = 3 + rb(ctx, 2);
          if (!i) throw new Error('تكرار 16 بلا سابقة');
          for (let k = 0; k < rep; k++) lens[i++] = prev;
        } else if (sym === 17) { const rep = 3 + rb(ctx, 3); for (let k = 0; k < rep; k++) lens[i++] = 0; prev = 0; }
        else { const rep = 11 + rb(ctx, 4); for (let k = 0; k < rep; k++) lens[i++] = 0; prev = 0; }
        if (i > total) throw new Error('ZIP: عدد أطوال الرموز غير صالح');
      }
      return { lit: buildHuff(lens.slice(0, hlit)), dist: buildHuff(lens.slice(hlit)) };
    }
    function inflateRaw(src, start, size) {
      const ctx = mkCtx(src, start, start + size);
      const out = [];
      const lit = fixedLit(), dist = fixedDist();
      let fin = false;
      while (!fin) {
        fin = !!rb(ctx, 1);
        const bt = rb(ctx, 2);
        if (bt === 0) {
          if (ctx.bit) rb(ctx, ctx.bit % 8); // محاذاة حدود البايت
          ctx.bit = 0; ctx.val = 0;
          if (ctx.p + 4 > ctx.end) throw new Error('كتلة مخزنة معطوبة');
          const len = ctx.s[ctx.p] | (ctx.s[ctx.p + 1] << 8);
          if ((len ^ 0xffff) !== (ctx.s[ctx.p + 2] | (ctx.s[ctx.p + 3] << 8))) throw new Error('كتلة ZIP معطوبة');
          ctx.p += 4;
          if (ctx.p + len > ctx.end) throw new Error('كتلة مخزنة تتجاوز النهاية');
          for (let i = 0; i < len; i++) out.push(ctx.s[ctx.p++]);
        } else if (bt === 1) {
          block(ctx, lit, dist, out);
        } else if (bt === 2) {
          const t = dynTrees(ctx);
          block(ctx, t.lit, t.dist, out);
        } else throw new Error('نوع كتلة غير صالح');
      }
      return new Uint8Array(out);
    }
    function open(bytes, lang, ctx) {
      if (bytes.length < 22 || bytes[0] !== 0x50 || bytes[1] !== 0x4b) throw new Error('ليس ملف ZIP');
      if (bytes.length > 8 * 1024 * 1024) throw new Error('ZIP: أرشيف كبير جداً');
      // End of Central Directory
      let eocd = -1;
      const from = Math.max(0, bytes.length - 22 - 65536);
      for (let i = bytes.length - 22; i >= from; i--) {
        if (bytes[i] === 0x50 && bytes[i + 1] === 0x4b && bytes[i + 2] === 5 && bytes[i + 3] === 6) { eocd = i; break; }
      }
      if (eocd < 0) throw new Error('ZIP: فهرس مفقود');
      const dv = new DataView(bytes.buffer, bytes.byteOffset, bytes.byteLength);
      const count = dv.getUint16(eocd + 10, true);
      let p = dv.getUint32(eocd + 16, true);
      const entries = [];
      for (let i = 0; i < count; i++) {
        if (dv.getUint32(p, true) !== 0x02014b50) throw new Error('ZIP: فهرس معطب');
        const method = dv.getUint16(p + 10, true);
        const cSize = dv.getUint32(p + 20, true);
        const uSize = dv.getUint32(p + 24, true);
        const nLen = dv.getUint16(p + 28, true);
        const eLen = dv.getUint16(p + 30, true);
        const cLen = dv.getUint16(p + 32, true);
        const off = dv.getUint32(p + 42, true);
        let name = '';
        for (let k = 0; k < nLen; k++) name += String.fromCharCode(bytes[p + 46 + k]);
        try { name = new TextDecoder('utf-8').decode(bytes.subarray(p + 46, p + 46 + nLen)); } catch (e) { /* ابقِ الاسم الخام */ }
        p += 46 + nLen + eLen + cLen;
        entries.push({ name, method, cSize, uSize, off });
      }
      const subs = entries.filter(e => /\.(srt|vtt)$/i.test(e.name) && e.uSize <= 2 * 1024 * 1024);
      if (!subs.length) throw new Error('ZIP: لا توجد ترجمة SRT/VTT صالحة');
      let choices = subs;
      if (ctx && ctx.season && ctx.episode && subs.length > 1) {
        choices = subs.filter(e => {
          const m = e.name.match(/\bS(\d{1,2})\s*[Ee.×x-]\s*(\d{1,3})\b/i);
          return m && +m[1] === ctx.season && +m[2] === ctx.episode;
        });
        if (!choices.length) throw new Error('ZIP: الحلقة المطلوبة غير موجودة في الأرشيف');
      }
      const langTags = lang === 'ar' ? /(?:arabic|arab|ara|(?:^|[._ -])ar(?:[._ -]|$))/i :
        /(?:german|deutsch|ger|deu|(?:^|[._ -])de(?:[._ -]|$))/i;
      const otherTags = lang === 'ar' ? /(?:german|deutsch|ger|deu|(?:^|[._ -])de(?:[._ -]|$))/i :
        /(?:arabic|arab|ara|(?:^|[._ -])ar(?:[._ -]|$))/i;
      const preferred = choices.find(e => langTags.test(e.name));
      if (!preferred && choices.length > 1 && choices.some(e => otherTags.test(e.name))) {
        throw new Error('ZIP: لم يوجد ملف اللغة المطلوبة داخل الأرشيف');
      }
      const pick = preferred || choices[0];
      let lp = pick.off;
      if (dv.getUint32(lp, true) !== 0x04034b50) throw new Error('ZIP: ترويسة محلية معطوبة');
      const lN = dv.getUint16(lp + 26, true), lE = dv.getUint16(lp + 28, true);
      const ds = lp + 30 + lN + lE;
      if (ds + pick.cSize > bytes.length) throw new Error('ZIP: بيانات الترجمة معطوبة');
      const raw = bytes.subarray(ds, ds + pick.cSize);
      if (pick.method === 0) return raw;
      if (pick.method === 8) return inflateRaw(raw, 0, raw.length);
      throw new Error('ZIP: طريقة ضغط غير مدعومة (' + pick.method + ')');
    }
    return { open };
  })();
  function decodeSubtitleBytes(bytes, lang, ctx) {
    if (bytes.length && bytes[0] === 0x50 && bytes[1] === 0x4b) {
      return decodeSubText(Zip.open(bytes, lang, ctx), lang);
    }
    return decodeSubText(bytes, lang);
  }
  /* ───────────────────── تحليل النصوص (SRT / WebVTT) ───────────────────── */
  function parseTs(s) {
    let sec = 0;
    for (const part of String(s).trim().split(':')) sec = sec * 60 + Number(part.replace(',', '.'));
    return sec;
  }
  function parseCues(raw) {
    if (!raw) return [];
    let text = String(raw).replace(/^\uFEFF/, '').replace(/\r/g, '');
    text = text.replace(/\u00a0/gi, ' ').replace(/&amp;/g, '&').replace(/&lt;/g, '<')
               .replace(/&gt;/g, '>').replace(/&#39;|&apos;/g, "'").replace(/&quot;/g, '"').replace(/&hellip;/g, '…');
    const blocks = text.split(/\n{2,}/);
    const cues = [];
    for (const b of blocks) {
      const m = b.match(/(\d{1,2}:)?\d{1,2}:\d{2}[.,]\d{1,3}\s*-->\s*(\d{1,2}:)?\d{1,2}:\d{2}[.,]\d{1,3}/);
      if (!m) continue;
      const s = parseTs(m[1] ? m[0].slice(0, m[0].indexOf('-->')) : m[0].slice(0, m[0].indexOf('-->')));
      const eStr = m[0].slice(m[0].indexOf('-->') + 3);
      const e = parseTs(eStr.trim().split(/\s/)[0]);
      const idx = b.indexOf('-->');
      let body = b.slice(idx + 3).split('\n').slice(1)
        .map(l => l.replace(/^\s*(position|align|vertical|line|size)\s*:.*/i, ''))
        .join('\n');
      body = body.replace(/<[^>]*>/g, ' ').replace(/\s+/g, ' ').trim();
      if (body && e > s && isFinite(s) && isFinite(e)) cues.push({ s, e, text: body });
    }
    cues.sort((a, b) => a.s - b.s);
    return cues;
  }
  // بحث ثنائي عن الـ cue النشطة عند الزمن t (مع مؤشر متزايد)
  function makeSeeker(cues) {
    let ptr = 0;
    return t => {
      while (ptr > 0 && cues[ptr - 1].e > t) ptr--;
      while (ptr < cues.length && cues[ptr].s <= t) {
        if (t < cues[ptr].e) break;
        ptr++;
      }
      const c = cues[ptr];
      return (c && c.s <= t && t < c.e) ? c.text : null;
    };
  }
  /* ───────────────────── كشف محتوى الصفحة ───────────────────── */
  function normDigits(s) {
    return String(s).replace(/[٠-٩]/g, d => String('٠١٢٣٤٥٦٧٨٩'.indexOf(d)))
                    .replace(/[۰-۹]/g, d => String('۰۱۲۳۴۵۶۷۸۹'.indexOf(d)));
  }
  function stripArabicTashkeel(s) {
    return String(s).replace(/[\u064B-\u0652\u0670\u0640]/g, '');
  }
  function cleanTitle(t) {
    t = normDigits(stripArabicTashkeel(t)).trim();
    t = t.split(/\s*[|–-]\s*(سينمانا|Cinemana|شبكة شباك|shabakaty)/i)[0];
    t = t.replace(/(مشاهدة|متابعة|تحميل)\s+(فيلم|مسلسل|حلقة)?\s*(اونلاين|اون لاين|أونلاين|أون لاين|آن لاين|online)/gi, ' ');
    t = t.replace(/^(?:مشاهدة|شاهد|فيلم|مسلسل|watch|stream)\s+/i, '');
    t = t.replace(/(HD|4K|1080p|720p|WEBRip|HDTV|دبلج|ترجمة)/gi, ' ');
    return t.replace(/\s{2,}/g, ' ').replace(/[\s.!?،؛:]+$/, '').trim();
  }
  function isNonFilmHeading(value) {
    const s = stripArabicTashkeel(normDigits(String(value || '')))
      .replace(/[أإآٱ]/g, 'ا').replace(/ى/g, 'ي')
      .replace(/[؟?!:؛،.]+/g, ' ').replace(/\s+/g, ' ').trim().toLowerCase();
    return !s ||
      /^(?:قد ترغب(?:\s|$)|قد يعجبك(?:\s|$)|قد تحب(?:\s|$)|ربما يعجبك(?:\s|$)|شاهد ايضا(?:\s|$)|افلام مشابهة|افلام مشابهه|افلام ذات صله|اعمال مشابهة|مسلسلات مشابهة|مقترح لك|موصي به|توصيات|المزيد من الافلام|نتائج البحث|you may also like|you might also like|more like this|recommended(?:\s|$)|related (?:movies|shows)|similar (?:movies|shows))/i.test(s) ||
      /^(?:ابحث|بحث عن|سينمانا|cinemana|search for|loading)(?:\s|$)/i.test(s) ||
      /^(?:فيلم|افلام|مسلسل|مسلسلات|الفيلم|movie|movies|movie title|tv shows|series|video|player|مشغل|العنوان)$/i.test(s);
  }
  function isUsableMovieTitle(value) {
    const name = String(value || '').trim();
    return name.length >= 2 && name.length <= 120 &&
      (/[A-Za-z\u0600-\u06ff]/.test(name) || /^(?:19|20)\d{2}$/.test(name)) &&
      !isNonFilmHeading(name);
  }
  function parseTitle(t) {
    t = t.trim();
    const original = t;
    let year = null, season = null, episode = null;
    let m;
    const years = [...t.matchAll(/\b(?:19\d{2}|20\d{2})\b/g)];
    const yearMatch = years[0]?.index === 0 ? years[1] : years[0];
    if (yearMatch) {
      year = Number(yearMatch[0]);
      t = t.slice(0, yearMatch.index) + ' ' + t.slice(yearMatch.index + 4);
    }
    if ((m = t.match(/\bS(\d{1,2})\s*[Ee.×x-]\s*(\d{1,3})\b/i))) { season = +m[1]; episode = +m[2]; }
    else {
      let sM = t.match(/الموسم\s*(\d{1,2})/), eM = t.match(/(?:الحلقة|حلقة)\s*(\d{1,3})/);
      if (!sM) sM = t.match(/\bم\s*(\d{1,2})\b/);
      if (!eM) eM = t.match(/\bح\s*(\d{1,3})\b/);
      if (sM) season = +sM[1];
      if (eM) episode = +eM[1];
    }
    const name = t
      .replace(/\bS\d{1,2}\s*[Ee.×x-]\s*\d{1,3}\b/gi, ' ')
      .replace(/الموسم\s*\d{1,2}/g, ' ')
      .replace(/(?:الحلقة|حلقة)\s*\d{1,3}/g, ' ')
      .replace(/\b[مح]\s*\d{1,3}\b/g, ' ')
      .replace(/\s{2,}/g, ' ')
      .replace(/[\s.!?،؛:]+$/, '').trim();
    return { name, year, season, episode, raw: original };
  }
  function imdbId(value) {
    const m = String(value || '').match(/\btt(\d{5,10})\b/i);
    return m ? 'tt' + m[1] : null;
  }
  function imdbNumber(value) {
    const id = imdbId(value);
    return id ? String(Number(id.slice(2))) : null;
  }
  function yearFrom(value) {
    const match = normDigits(String(value || '')).match(/\b(19\d{2}|20\d{2})\b/);
    const year = match ? Number(match[1]) : null;
    return year && year <= new Date().getFullYear() + 2 ? year : null;
  }
  function movieMetadata(title) {
    const out = { year: null, imdbId: null, aliases: [] };
    const scripts = $$('script[type="application/ld+json"]').slice(0, 8);
    for (const script of scripts) {
      if (script.textContent.length > 80000) continue;
      let json;
      try { json = JSON.parse(script.textContent); } catch (e) { continue; }
      const entries = Array.isArray(json) ? json : [json];
      for (const root of entries) {
        const nodes = Array.isArray(root['@graph']) ? root['@graph'] : [root];
        for (const node of nodes) {
          if (!node || !/movie|tvseries|tvepisode/i.test(String(node['@type'] || ''))) continue;
          const names = [node.name, node.alternateName].flat().filter(x => typeof x === 'string');
          if (!names.some(name => nameSim(title, name) >= 0.87)) continue;
          out.year = yearFrom(node.datePublished || node.releaseDate) || out.year;
          const refs = [node.sameAs, node.url, node['@id']].flat();
          out.imdbId = refs.map(imdbId).find(Boolean) || out.imdbId;
          out.aliases.push(...names);
        }
      }
    }
    if (!out.imdbId) {
      const refs = $$('a[href*="imdb.com/title/tt"]')
        .filter(a => !a.closest('[class*="recommend"], [class*="related"], [class*="suggest"]'))
        .map(a => imdbId(a.href)).filter(Boolean);
      const unique = [...new Set(refs)];
      if (unique.length === 1) out.imdbId = unique[0];
    }
    if (!out.year) {
      for (const selector of ['[itemprop="datePublished"]', '.movie-year', '.release-year', '[data-release-year]']) {
        const node = $(selector);
        if (node) { out.year = yearFrom(node.getAttribute('datetime') || node.getAttribute('content') ||
          node.getAttribute('data-release-year') || node.textContent); if (out.year) break; }
      }
    }
    return out;
  }
  function videoReleaseName(video, ctx) {
    if (!video) return '';
    const hints = [video.getAttribute('data-filename'), video.getAttribute('data-file-name')];
    for (const src of [video.currentSrc, video.src, video.querySelector('source[src]')?.src]) {
      if (!src || src.startsWith('blob:')) continue;
      try { hints.push(decodeURIComponent(new URL(src, location.href).pathname.split('/').pop())); }
      catch (e) { /* اسم بث غير قابل للقراءة */ }
    }
    for (const hint of hints) {
      if (!hint || /trailer|teaser|preview/i.test(hint)) continue;
      if (!/WEB[-. ]?DL|WEBRip|BluRay|BRRip|BDRip|HDTV|DVDRip|2160p|1080p|720p/i.test(hint)) continue;
      const text = normKey(hint);
      if (text.includes(normKey(ctx.name)) || (ctx.year && text.includes(String(ctx.year)))) {
        return String(hint).slice(0, 200);
      }
    }
    return '';
  }
  function detectContent() {
    const cands = [];
    const video = findVideo();
    if (video) {
      let node = video;
      for (let depth = 0; depth < 5 && node; depth++, node = node.parentElement) {
        for (const attr of ['data-movie-title', 'data-series-title', 'data-original-title', 'data-title']) {
          const name = node.getAttribute(attr);
          if (isUsableMovieTitle(name)) cands.push(name);
        }
      }
    }
    ['.movie-title', '.series-title', '.video-title', '.entry-title', '.post-title',
      '[data-testid="movie-title"]', 'h1', '.page-title',
      'meta[property="og:title"]', 'meta[name="twitter:title"]', 'title'].forEach(sel => {
      $$(sel).forEach(node => {
        const t = (node.content || node.textContent || '').trim();
        if (node.closest?.('[class*="recommend" i], [class*="related" i], [class*="suggest" i], [class*="similar" i]')) return;
        if (t && t.length < 160 && isUsableMovieTitle(t)) cands.push(t);
      });
    });
    if (!cands.length) {
      const schemaTitles = [];
      for (const script of $$('script[type="application/ld+json"]').slice(0, 8)) {
        try {
          const data = JSON.parse(script.textContent);
          const roots = Array.isArray(data) ? data : [data];
          for (const root of roots) {
            const nodes = Array.isArray(root['@graph']) ? root['@graph'] : [root];
            for (const node of nodes) {
              if (!node || !/movie|tvseries|tvepisode/i.test(String(node['@type'] || '')) ||
                  !isUsableMovieTitle(node.name)) continue;
              const ref = node.url || node['@id'];
              if (!ref) continue;
              const schemaUrl = new URL(ref, location.href);
              if (schemaUrl.origin === location.origin && schemaUrl.pathname === location.pathname) {
                schemaTitles.push(node.name);
              }
            }
          }
        } catch (e) { /* بيانات صفحة لا تخص فيلماً */ }
      }
      if ([...new Set(schemaTitles)].length === 1) cands.push(schemaTitles[0]);
    }
    // لا نحول رابط البث أو عنوان قسم التوصيات إلى اسم فيلم.
    for (const raw of cands) {
      const t = cleanTitle(raw);
      if (!isUsableMovieTitle(t)) continue;
      const p = parseTitle(t);
      if (!isUsableMovieTitle(p.name)) continue;
      for (const other of cands) {
        const alternative = parseTitle(cleanTitle(other));
        if (!p.year && isUsableMovieTitle(alternative.name) && alternative.year &&
            nameSim(p.name, alternative.name) >= 0.92) p.year = alternative.year;
      }
      const metadata = movieMetadata(p.name);
      p.year = p.year || metadata.year;
      p.imdbId = metadata.imdbId;
      p.aliases = [...new Set([p.name, ...metadata.aliases])];
      p.releaseName = videoReleaseName(video, p);
      return p;
    }
    return null;
  }
  function keyOf(c) {
    return normKey(c.name) + '|' + (c.imdbId || '') + '|' + (c.year || 0) + '|' +
      (c.season || 0) + 'x' + (c.episode || 0) + '|' + (c.releaseName ? normKey(c.releaseName) : '');
  }
  /* ───────────────────── المشغل ───────────────────── */
  function videoArea(v) { const r = v.getBoundingClientRect(); return Math.max(0, r.width) * Math.max(0, r.height); }
  function findVideo() {
    const vs = $$('video').filter(v => videoArea(v) > 240 * 120 && v.getClientRects().length &&
      (v.readyState >= 1 || v.currentSrc || v.src || v.querySelector('source[src]')));
    if (!vs.length) return null;
    vs.sort((a, b) => videoArea(b) - videoArea(a));
    return vs[0];
  }
  function videoDuration(v) {
    if (!v) return null;
    try { return isFinite(v.duration) && v.duration > 0 ? v.duration : null; } catch (e) { return null; }
  }
  /* ───────────────────── المطابقة والتقييم ───────────────────── */
  function normKey(s) {
    return String(s || '')
      .replace(/ä/g, 'ae').replace(/ö/g, 'oe').replace(/ü/g, 'ue').replace(/ß/g, 'ss')
      .replace(/Ä/g, 'Ae').replace(/Ö/g, 'Oe').replace(/Ü/g, 'Ue')
      .normalize('NFD').replace(/[\u0300-\u036f]/g, '')
      .toLowerCase().replace(/[^a-z0-9\u0600-\u06FF]+/g, ' ')
      .replace(/\s+/g, ' ').trim();
  }
  function lev(a, b) {
    const m = a.length, n = b.length;
    if (!m) return n; if (!n) return m;
    let prev = new Array(n + 1);
    for (let j = 0; j <= n; j++) prev[j] = j;
    for (let i = 1; i <= m; i++) {
      let cur = [i];
      for (let j = 1; j <= n; j++) {
        cur[j] = Math.min(prev[j] + 1, cur[j - 1] + 1, prev[j - 1] + (a[i - 1] === b[j - 1] ? 0 : 1));
      }
      prev = cur;
    }
    return prev[n];
  }
  function nameSim(a, b) {
    const A = normKey(a), B = normKey(b);
    if (!A || !B) return 0;
    if (A === B) return 1;
    if (A.length > 3 && B.length > 3 && (A.includes(B) || B.includes(A))) return 0.92;
    const ta = new Set(A.split(' ')), tb = new Set(B.split(' '));
    let inter = 0; ta.forEach(w => { if (w.length > 1 && tb.has(w)) inter++; });
    const jac = inter / ((ta.size + tb.size - inter) || 1);
    const maxLen = Math.max(A.length, B.length) || 1;
    const levR = 1 - lev(A, B) / maxLen;
    return Math.max(jac * 0.95, Math.min(levR, 0.95));
  }
  function releaseTitle(filename) {
    const text = String(filename || '').replace(/\.(?:srt|vtt|zip)$/i, '')
      .replace(/[_\.]/g, ' ').trim();
    const markers = /\b(?:19\d{2}|20\d{2}|S\d{1,2}\s*E\d{1,3}|2160p|1080p|720p|480p|WEB[- ]?DL|WEBRip|BluRay|BRRip|BDRip|HDTV|DVDRip|HDTS|HDRip|DVDScr)\b/gi;
    const marker = [...text.matchAll(markers)].find(m => m.index > 0);
    return (marker ? text.slice(0, marker.index) : text)
      .replace(/\s*\([^)]*\)\s*$/, '').replace(/[\s,()\[\]-]+$/, '').trim();
  }
  function releaseFamily(value) {
    const s = String(value || '').toLowerCase();
    if (/blu[- .]?ray|bdrip|brrip/.test(s)) return 'bluray';
    if (/web[- .]?dl|webrip|web[- .]?rip/.test(s)) return 'web';
    if (/hdtv|pdtv/.test(s)) return 'hdtv';
    if (/dvd[- .]?rip|dvdscr/.test(s)) return 'dvd';
    return '';
  }
  function releaseFps(value) {
    const match = String(value || '').match(/(?:^|[^\d])(23[.,]976|24[.,]000|25[.,]000|29[.,]97|30[.,]000|23|24|25|30)\s*(?:fps|hz)(?!\w)/i);
    return match ? Number(match[1].replace(',', '.')) : null;
  }
  function scoreCandidate(c, ctx) {
    const reject = why => ({ total: -100, why });
    const aliases = [...new Set((ctx.aliases || [ctx.name]).map(normKey).filter(Boolean))];
    const title = normKey(c.movieTitle || releaseTitle(c.title));
    const idMatch = !!(ctx.imdbId && c.imdbId && imdbNumber(ctx.imdbId) === imdbNumber(c.imdbId));
    if (ctx.imdbId && c.imdbId && !idMatch) return reject('هوية IMDb مختلفة');
    const exact = aliases.includes(title);
    const sName = Math.max(0, ...aliases.map(name => nameSim(name, title)));
    if (!idMatch && !exact && sName < 0.93) return reject('عنوان فيلم مختلف أو امتداد مضلل');
    const filenameYear = yearFrom(String(c.title).replace(/^\s*(19\d{2}|20\d{2})(?=\s*[. ]*\w)/, ''));
    const year = Number(c.year || filenameYear) || null;
    if (ctx.year && year && Math.abs(ctx.year - year) > 1) return reject('سنة إصدار مختلفة');
    const m = String(c.title).match(/\bS(\d{1,2})\s*[Ee.×x-]\s*(\d{1,3})\b/i);
    const isSeries = !!(ctx.season && ctx.episode);
    const season = c.season == null ? (m ? +m[1] : null) : Number(c.season);
    const episode = c.episode == null ? (m ? +m[2] : null) : Number(c.episode);
    if (isSeries && (season !== ctx.season || episode !== ctx.episode)) return reject('موسم أو حلقة أخرى');
    if (!isSeries && (m || (season && episode))) return reject('ملف حلقة وليس فيلماً');
    let s = exact ? 76 : idMatch ? 68 : 62;
    const evidence = [exact ? 'عنوان مطابق' : 'عنوان قريب'];
    if (idMatch) { s += 36; evidence.push('IMDb مطابق'); }
    if (ctx.year && year) {
      if (ctx.year === year) { s += 19; evidence.push('سنة مطابقة'); }
      else { s -= 16; evidence.push('سنة قريبة'); }
    } else if (ctx.year && !year) s -= 6;
    if (isSeries) { s += 22; evidence.push('الحلقة مطابقة'); }
    const videoRelease = releaseFamily(ctx.releaseName);
    const subRelease = releaseFamily([c.title, ...(c.releases || [])].join(' '));
    if (videoRelease && subRelease) {
      if (videoRelease === subRelease) { s += 14; evidence.push('إصدار مطابق'); }
      else { s -= 18; evidence.push('إصدار مختلف'); }
    }
    const videoFps = releaseFps(ctx.releaseName);
    const subFps = Number(c.fps) || releaseFps(c.title);
    if (videoFps && subFps && Math.abs(videoFps - subFps) / videoFps > 0.012) return reject('معدل إطارات مختلف');
    if (videoFps && subFps) { s += 8; evidence.push('FPS مطابق'); }
    if (c.machine) s -= 5;
    if (c.hi) s -= 3;
    if (c.trusted) s += 3;
    if (c.downloadCount) s += Math.min(4, Math.log10(1 + Number(c.downloadCount)));
    s += ENGINE_TIE[c.engine] || 0;
    return { total: Math.round(s), sName, year, idMatch, evidence };
  }
  function runtimeFit(cues, duration) {
    const last = cues[cues.length - 1];
    if (!last || !Number.isFinite(last.e)) return { ok: false, bonus: 0, reason: 'توقيت ملف غير صالح' };
    if (!duration || duration < 30 * 60) return { ok: true, bonus: 0, gap: null };
    if (cues.length < 20 || last.e < duration * 0.72) {
      return { ok: false, bonus: 0, reason: 'ترجمة قصيرة أو جزئية' };
    }
    const gap = Math.abs(duration - last.e);
    const tolerance = clamp(duration * 0.085, 7 * 60, 13 * 60);
    if (gap > tolerance) return { ok: false, bonus: 0, gap, reason: 'المدة لا تطابق الفيلم' };
    return { ok: true, bonus: Math.round(24 * (1 - gap / tolerance)), gap };
  }
  function subtitleOverlapRatio(left, right) {
    const total = cues => cues.reduce((sum, c) => sum + Math.max(0, c.e - c.s), 0);
    let i = 0, j = 0, overlap = 0;
    while (i < left.length && j < right.length) {
      const a = left[i], b = right[j];
      overlap += Math.max(0, Math.min(a.e, b.e) - Math.max(a.s, b.s));
      if (a.e <= b.e) i++; else j++;
    }
    return Math.min(1, overlap / Math.max(1, Math.min(total(left), total(right))));
  }
  /* ───────────────────── حارس المحركات (تبريد + أخطاء مستقلة) ───────────────────── */
  const Guard = {
    st: {},
    fail(id, err) {
      const code = err && err.status;
      const rate = code === 429 || code === 406;
      const auth = code === 401 || code === 403;
      const prev = this.st[id] || { n: 0, until: 0 };
      prev.n++;
      prev.until = Date.now() + (rate || auth ? 60 : 10) * 60e3;
      this.st[id] = prev;
    },
    cool(id) { const s = this.st[id]; return s && s.until > Date.now(); },
    coolMin(id) { const s = this.st[id]; return s ? Math.ceil((s.until - Date.now()) / 60e3) : 0; },
  };
  /* ───────────────────── المحركات ───────────────────── */
  const Engines = {
    /* 1) OpenSubtitles v1 */
    openSubtitles: {
      id: 'openSubtitles', label: 'OpenSubtitles',
      available() { return !!(S.engineOn.openSubtitles && API.openSubtitles.apiKey); },
      reason() { return !S.engineOn.openSubtitles ? 'موقوف من اللوحة' : !API.openSubtitles.apiKey ? 'لا يوجد مفتاح API' : ''; },
      async jwt() {
        const o = API.openSubtitles;
        if (!o.username || !o.password) return null;
        if (this.session && this.session.expires > Date.now()) return this.session;
        const j = await httpJson({
          url: o.base + '/login',
          method: 'POST',
          headers: { 'Content-Type': 'application/json', 'Api-Key': o.apiKey, 'User-Agent': o.userAgent, Accept: 'application/json' },
          body: JSON.stringify({ username: o.username, password: o.password }),
        });
        if (!j || !j.token) throw new Error('تسجيل دخول OpenSubtitles: تحقق من اسم المستخدم وكلمة المرور');
        const host = j.base_url || 'api.opensubtitles.com';
        if (!/^(?:vip-)?api\.opensubtitles\.com$/.test(host)) throw new Error('مضيف OpenSubtitles غير متوقع');
        this.session = { token: j.token, base: 'https://' + host + '/api/v1', expires: Date.now() + 12 * 3600e3 };
        return this.session;
      },
      async search(ctx, langs) {
        const os = API.openSubtitles;
        const p = new URLSearchParams();
        const series = !!(ctx.season && ctx.episode);
        if (ctx.imdbId) p.set(series ? 'parent_imdb_id' : 'imdb_id', imdbNumber(ctx.imdbId));
        else p.set('query', ctx.name.toLowerCase());
        p.set('languages', langs.slice().sort().join(','));
        p.set('type', series ? 'episode' : 'movie');
        if (ctx.season) p.set('season_number', String(ctx.season));
        if (ctx.episode) p.set('episode_number', String(ctx.episode));
        if (ctx.year && !ctx.season) p.set('year', String(ctx.year));
        const headers = { 'Api-Key': os.apiKey, 'User-Agent': os.userAgent, Accept: 'application/json' };
        let j = await httpJson({ url: os.base + '/subtitles?' + p.toString(), headers });
        if (ctx.imdbId && !(j.data || []).length) {
          p.delete(series ? 'parent_imdb_id' : 'imdb_id');
          p.set('query', ctx.name.toLowerCase());
          j = await httpJson({ url: os.base + '/subtitles?' + p.toString(), headers });
        }
        const out = [];
        for (const item of (j.data || [])) {
          const at = item.attributes || item;
          if (at.foreign_parts_only) continue;
          const lang = ISO3[String(at.language || '').toLowerCase()] || String(at.language || '').toLowerCase();
          if (!langs.includes(lang)) continue;
          const feature = at.feature_details || {};
          for (const file of (at.files || [])) {
            if (!Number.isInteger(Number(file.file_id))) continue;
            out.push({
              engine: this.id, lang,
              title: file.file_name || at.release || feature.title || '',
              movieTitle: feature.parent_title || feature.title || '',
              year: series ? null : (feature.year ? Number(feature.year) : null),
              imdbId: (series ? feature.parent_imdb_id : feature.imdb_id) ?
                'tt' + String(series ? feature.parent_imdb_id : feature.imdb_id) : null,
              season: feature.season_number || null, episode: feature.episode_number || null,
              machine: !!(at.machine_translated || at.ai_translated), hi: !!at.hearing_impaired,
              trusted: !!at.from_trusted, downloadCount: at.download_count || 0,
              releases: at.release ? [at.release] : [], fps: Number(at.fps) || null,
              dl: { id: Number(file.file_id) },
            });
          }
        }
        return out;
      },
      async download(item, ctx) {
        const os = API.openSubtitles;
        const session = await this.jwt();
        const headers = { 'Api-Key': os.apiKey, 'User-Agent': os.userAgent,
          'Content-Type': 'application/json', Accept: 'application/json' };
        if (session) headers.Authorization = 'Bearer ' + session.token;
        const j = await httpJson({
          url: (session ? session.base : os.base) + '/download',
          method: 'POST',
          headers,
          body: JSON.stringify({ file_id: item.dl.id }),
        });
        if (!j.link) throw new Error('لم يُرجع OpenSubtitles رابط تحميل');
        const url = new URL(j.link);
        if (url.protocol !== 'https:' || !/(^|\.)opensubtitles\.(org|com)$/.test(url.hostname))
          throw new Error('OpenSubtitles: رابط تحميل غير موثوق');
        const r = await http({ url: url.href });
        return decodeSubtitleBytes(r.bytes, item.lang, ctx);
      },
    },
    /* 2) SubDL */
    subdl: {
      id: 'subdl', label: 'SubDL',
      available() { return !!(S.engineOn.subdl && API.subdl.apiKey); },
      reason() { return !S.engineOn.subdl ? 'موقوف من اللوحة' : !API.subdl.apiKey ? 'لا يوجد مفتاح API' : ''; },
      async search(ctx, langs) {
        const q = new URLSearchParams();
        q.set('api_key', API.subdl.apiKey);
        q.set(ctx.imdbId ? 'imdb_id' : 'film_name', ctx.imdbId || ctx.name);
        q.set('type', (ctx.season && ctx.episode) ? 'tv' : 'movie');
        q.set('languages', langs.map(l => l.toUpperCase()).join(','));
        if (ctx.season) q.set('season_number', String(ctx.season));
        if (ctx.episode) q.set('episode_number', String(ctx.episode));
        if (ctx.year) q.set('year', String(ctx.year));
        q.set('subs_per_page', '30');
        q.set('unpack', '1');
        q.set('releases', '1');
        const headers = { Accept: 'application/json' };
        let j = await httpJson({ url: API.subdl.base + '/subtitles?' + q.toString(), headers });
        if (ctx.imdbId && j.status !== false && !(j.subtitles || []).length) {
          q.delete('imdb_id'); q.set('film_name', ctx.name);
          j = await httpJson({ url: API.subdl.base + '/subtitles?' + q.toString(), headers });
        }
        if (j.status === false) throw new Error('SubDL: رفض البحث (تحقق من المفتاح أو الحصة)');
        const arr = Array.isArray(j.subtitles) ? j.subtitles : [];
        const film = Array.isArray(j.results) ? j.results[0] : null;
        if (film && ctx.imdbId && film.imdb_id && imdbNumber(film.imdb_id) !== imdbNumber(ctx.imdbId)) return [];
        if (film && ctx.year && film.year && Math.abs(Number(film.year) - ctx.year) > 1) return [];
        if (film && !ctx.imdbId && Math.max(...(ctx.aliases || [ctx.name]).map(a => nameSim(a, film.name || ''))) < 0.93) return [];
        const out = [];
        for (const it of arr) {
          const raw = String(it.language || it.lang || '').toLowerCase();
          const lang = ISO3[raw] || raw;
          const name = it.release_name || it.name || '';
          if (it.full_season && ctx.episode) {
            const files = Array.isArray(it.unpack_files) ? it.unpack_files : [];
            for (const f of files) {
              const fl = ISO3[String(f.language || '').toLowerCase()] || String(f.language || '').toLowerCase();
              if (!langs.includes(fl) || +f.season !== ctx.season || +f.episode !== ctx.episode || !f.url) continue;
              out.push({ engine: this.id, lang: fl,
                title: (f.release_name || name) + ' S' + String(ctx.season).padStart(2, '0') + 'E' + String(ctx.episode).padStart(2, '0'),
                movieTitle: film && film.name, year: film && film.year,
                imdbId: film && film.imdb_id, fps: Number(it.fps) || null,
                releases: Array.isArray(it.releases) ? it.releases : [],
                season: ctx.season, episode: ctx.episode, hi: !!f.hi, dl: { path: f.url },
              });
            }
            continue;
          }
          if (!langs.includes(lang) || !it.url) continue;
          out.push({ engine: this.id, lang, title: name, movieTitle: film && film.name,
            year: film && film.year, season: it.season || null, episode: it.episode || null,
            imdbId: film && film.imdb_id, fps: Number(it.fps) || null,
            releases: Array.isArray(it.releases) ? it.releases : [],
            hi: !!it.hi, dl: { path: it.url } });
        }
        return out;
      },
      async download(item, ctx) {
        const u = new URL(item.dl.path, API.subdl.dlBase);
        if (u.origin !== API.subdl.dlBase || !u.pathname.startsWith('/subtitle/'))
          throw new Error('SubDL: رابط تنزيل غير موثوق');
        const r = await http({ url: u.href, timeout: 30000 });
        return decodeSubtitleBytes(r.bytes, item.lang, ctx);
      },
    },
    /* 3) SubSource */
    subsource: {
      id: 'subsource', label: 'SubSource',
      available() { return !!(S.engineOn.subsource && API.subsource.apiKey); },
      reason() { return !S.engineOn.subsource ? 'موقوف من اللوحة' : !API.subsource.apiKey ? 'لا يوجد مفتاح API' : ''; },
      async movie(ctx) {
        if (!isUsableMovieTitle(ctx && ctx.name)) return null;
        const query = async byId => {
          const q = new URLSearchParams();
          q.set('searchType', byId ? 'imdb' : 'text');
          if (byId && !imdbId(ctx.imdbId)) return null;
          const searchName = cleanTitle(ctx.name).trim();
          if (!byId && !isUsableMovieTitle(searchName)) return null;
          q.set(byId ? 'imdb' : 'q', byId ? ctx.imdbId : searchName.toLowerCase());
          if (!byId && !q.get('q')) return null;
          if (ctx.season) q.set('season', String(ctx.season));
          const j = await httpJson({ url: API.subsource.base + '/movies/search?' + q.toString(),
            headers: { 'X-API-Key': API.subsource.apiKey } });
          const list = Array.isArray(j.data) ? j.data : Array.isArray(j.results) ? j.results : [];
          let best = null, score = -1;
          for (const r of list) {
            const apiId = imdbId(r.imdbId || r.imdb || r.imdb_id);
            if (ctx.imdbId && apiId && imdbNumber(ctx.imdbId) !== imdbNumber(apiId)) continue;
            const year = Number(r.releaseYear || r.year) || null;
            if (ctx.year && year && Math.abs(ctx.year - year) > 1) continue;
            const similarity = Math.max(...(ctx.aliases || [ctx.name]).map(a => nameSim(a, r.title || r.name || '')));
            if (!byId && similarity < 0.93) continue;
            const rank = similarity * 100 + (ctx.year && year === ctx.year ? 20 : 0);
            if (rank > score) { score = rank; best = { ...r, verifiedImdbId: apiId || (byId ? ctx.imdbId : null) }; }
          }
          return best;
        };
        if (ctx.imdbId) {
          const found = await query(true);
          if (found) return found;
        }
        return query(false);
      },
      async listFor(id, lang) {
        const q = new URLSearchParams({ movieId: String(id), language: lang, sort: 'rating', limit: '100' });
        const j = await httpJson({ url: API.subsource.base + '/subtitles?' + q.toString(),
          headers: { 'X-API-Key': API.subsource.apiKey } });
        const arr = j.data || j.subtitles || [];
        return Array.isArray(arr) ? arr : (Array.isArray(arr.subtitles) ? arr.subtitles : []);
      },
      async search(ctx, langs) {
        const movie = await this.movie(ctx);
        const id = movie && (movie.id || movie.movieId);
        if (!id) return [];
        const out = [];
        for (const lang of langs) {
          const list = await this.listFor(id, lang === 'de' ? 'german' : lang === 'ar' ? 'arabic' : lang);
          for (const it of list) {
            const sid = it.subtitleId || it.id;
            if (!sid) continue;
            const release = it.releaseInfo || it.releaseName || it.title ||
              (Array.isArray(it.releaseNames) ? it.releaseNames[0] : '');
            out.push({
              engine: this.id, lang,
              title: release || movie.title, movieTitle: movie.title,
              year: movie.releaseYear || movie.year || null,
              imdbId: movie.verifiedImdbId || null,
              releases: Array.isArray(it.releaseNames) ? it.releaseNames : [],
              fps: Number(it.fps || it.frameRate) || null,
              season: it.seasonNumber || null, episode: it.episodeNumber || null,
              dl: { sid },
            });
          }
        }
        return out;
      },
      async download(item, ctx) {
        const r = await http({
          url: API.subsource.base + '/subtitles/' + encodeURIComponent(item.dl.sid) + '/download',
          headers: { 'X-API-Key': API.subsource.apiKey },
          timeout: 40000,
        });
        const text = decodeSubtitleBytes(r.bytes, item.lang, ctx);
        if (!parseCues(text).length) throw new Error('ملف SubSource فارغ');
        return text;
      },
    },
    /* 4) SubtitleCat: بحث HTML عام؛ لا ندّعي وجود ترجمة إلا بعد تنزيل SRT صالح. */
    subtitleCat: {
      id: 'subtitleCat', label: 'SubtitleCat',
      available() { return !!S.engineOn.subtitleCat; },
      reason() { return S.engineOn.subtitleCat ? '' : 'موقوف من اللوحة'; },
      async search(ctx, langs) {
        const base = API.subtitleCat.base;
        const query = ctx.name + (ctx.year ? ' ' + ctx.year : '') +
          (ctx.season && ctx.episode ? ' S' + String(ctx.season).padStart(2, '0') +
            'E' + String(ctx.episode).padStart(2, '0') : '');
        const r = await http({ url: base + 'index.php?search=' + encodeURIComponent(query), timeout: 18000 });
        const doc = new DOMParser().parseFromString(decodeSubText(r.bytes), 'text/html');
        const rows = Array.from(doc.querySelectorAll('table.sub-table tbody tr')).map(row => {
          const link = row.querySelector('a[href^="subs/"]');
          if (!link) return null;
          const release = link.textContent.trim();
          const from = row.textContent.match(/\(translated from ([^)]+)\)/i);
          const source = from ? from[1].trim().toLowerCase() : '';
          const sourceLang = /german|deutsch/i.test(source) ? 'de' : /arabic/i.test(source) ? 'ar' : '';
          const u = new URL(link.getAttribute('href'), base);
          if (u.origin !== new URL(base).origin || !u.pathname.startsWith('/subs/')) return null;
          const score = scoreCandidate({ engine: this.id, title: release,
            year: yearFrom(release) }, ctx).total;
          return { release, url: u.href, sourceLang, score };
        }).filter(row => row && row.score >= S.minScore);
        rows.sort((a, b) =>
          ((langs.includes(b.sourceLang) ? 16 : 0) + b.score) -
          ((langs.includes(a.sourceLang) ? 16 : 0) + a.score));
        const out = [], seen = new Set();
        for (const row of rows.slice(0, 6)) {
          let detail;
          try { detail = await http({ url: row.url, timeout: 18000 }); }
          catch (e) { log.warn('SubtitleCat: صفحة نتيجة غير متاحة:', e.message); continue; }
          const page = new DOMParser().parseFromString(decodeSubText(detail.bytes), 'text/html');
          for (const a of page.querySelectorAll('div.sub-single a[id^="download_"][href]')) {
            const lang = a.id.replace(/^download_/, '').toLowerCase();
            if (!langs.includes(lang) || seen.has(lang + row.url)) continue;
            const href = a.getAttribute('href');
            if (!href || href === '#') continue;
            const u = new URL(href, base);
            if (u.protocol !== 'https:' || u.origin !== new URL(base).origin) continue;
            seen.add(lang + row.url);
            out.push({ engine: this.id, lang, title: row.release, year: yearFrom(row.release),
              machine: !!row.sourceLang && row.sourceLang !== lang,
              dl: { url: u.href } });
          }
          if (langs.every(lang => out.some(c => c.lang === lang && !c.machine))) break;
        }
        return out;
      },
      async download(item, ctx) {
        const u = new URL(item.dl.url);
        if (u.protocol !== 'https:' || u.origin !== new URL(API.subtitleCat.base).origin)
          throw new Error('SubtitleCat: رابط تنزيل غير موثوق');
        const r = await http({ url: u.href, timeout: 30000 });
        return decodeSubtitleBytes(r.bytes, item.lang, ctx);
      },
    },
    /* 5) Podnapisi: نحتفظ بالشيفرة القديمة للتوافق، لكن نطاقه توقف عن العمل. */
    podnapisi: {
      id: 'podnapisi', label: 'Podnapisi',
      available() { return false; },
      reason() { return 'غير متاح: الموقع متوقف'; },
      async search(ctx, langs) {
        const q = new URLSearchParams();
        q.set('sK', ctx.name);
        q.set('sJ', langs.join(','));
        if (ctx.year) q.set('sY', ctx.year);
        if (ctx.season) q.set('sTS', ctx.season);
        if (ctx.episode) q.set('sTE', ctx.episode);
        q.set('sXML', '1');
        const r = await http({ url: API.podnapisi.searchUrl + '?' + q.toString(), headers: { Accept: 'application/xml, text/xml, */*' } });
        const xml = new DOMParser().parseFromString(decodeSubText(r.bytes), 'text/xml');
        if (xml.querySelector('parsererror')) throw new Error('استجابة Podnapisi غير صالحة');
        const out = [];
        xml.querySelectorAll('result').forEach(node => {
          const g = n => (node.querySelector(n) || {}).textContent || '';
          const pid = g('pid').trim();
          const lang = (g('language') || '').trim().toLowerCase();
          if (!pid || !langs.includes(lang)) return;
          out.push({
            engine: this.id, lang,
            title: (g('release') || ctx.name).trim(),
            year: ctx.year,
            dl: { pid },
          });
        });
        return out;
      },
      async download(item, ctx) {
        const r = await http({
          url: API.podnapisi.dlBase + item.dl.pid + '/download',
          headers: { Referer: 'https://www.podnapisi.net/subtitles/' + item.dl.pid + '/download' },
          timeout: 40000,
        });
        const text = decodeSubtitleBytes(r.bytes, item.lang, ctx);
        if (!parseCues(text).length) throw new Error('ملف Podnapisi فارغ');
        return text;
      },
    },
  };
  /* ───────────────────── الواجهة ───────────────────── */
  let ui = null;
  function buildUI() {
    try {
      if (typeof GM_addStyle === 'function') GM_addStyle(UI_CSS); else injectStyle(UI_CSS);
      const fab = el('button', 'cmsFAB', 'CC');
      fab.title = 'CineSubs — محرك الترجمات التلقائي';
      fab.addEventListener('click', () => ui.toggle(true));
      document.body.appendChild(fab);
      const panel = el('div', 'cmsPanel');
      panel.innerHTML =
        '<div class="cmsHead" id="cmsHead">' +
          '<span class="cmsTitle">CineSubs <small>v' + VERSION + '</small></span>' +
          '<button class="cmsClose" id="cmsClose" title="إغلاق">×</button>' +
        '</div>' +
        '<div class="cmsBody">' +
          '<div class="cmsSec">' +
            '<label class="cmsRow"><span>البحث التلقائي</span><input type="checkbox" id="cmsAuto" ' + (S.auto ? 'checked' : '') + '></label>' +
            '<label class="cmsRow"><span>إظهار الترجمة</span><input type="checkbox" id="cmsShow" ' + (S.show ? 'checked' : '') + '></label>' +
          '</div>' +
          '<div class="cmsSec">' +
            '<div class="cmsH">وضع الترجمة</div>' +
            '<div class="cmsSeg">' +
              '<button type="button" data-mode="both">🇩🇪 ألماني + 🇮🇶 عربي</button>' +
              '<button type="button" data-mode="de">🇩🇪 ألماني فقط</button>' +
            '</div>' +
            '<div class="cmsH">اللغات</div>' +
            '<label class="cmsRow"><span>🇩 الألمانية <em>(رئيسية)</em></span><input type="checkbox" id="cmsDe" ' + (S.langs.de ? 'checked' : '') + '></label>' +
            '<label class="cmsRow"><span>🇮🇶 العربية <em>(ثانوية)</em></span><input type="checkbox" id="cmsAr" ' + (S.langs.ar ? 'checked' : '') + '></label>' +
          '</div>' +
          '<div class="cmsSec">' +
            '<div class="cmsH">الحالة</div>' +
            '<div class="cmsState" id="cmsState"><span class="cmsChip" id="cmsChip">بانتظار المشغل…</span></div>' +
            '<div class="cmsKV"><span>العمل</span><b id="cmsContent">—</b></div>' +
            '<div class="cmsKV"><span>الفيديو</span><b id="cmsVideoDur">—</b></div>' +
            '<div class="cmsKV cmsSub" id="cmsSubDe"><span>ألماني</span><b>—</b></div>' +
            '<div class="cmsKV cmsSub" id="cmsSubAr"><span>عربي</span><b>—</b></div>' +
            '<div class="cmsKV" id="cmsMatchDe"><span>مطابقة ألماني</span><b>—</b></div>' +
            '<div class="cmsKV" id="cmsMatchAr"><span>مطابقة عربي</span><b>—</b></div>' +
            '<div class="cmsKV" id="cmsNativeStatus"><span>مشغل iPhone</span><b>بانتظار الترجمة</b></div>' +
            '<div class="cmsEngines" id="cmsEngines"></div>' +
          '</div>' +
          '<div class="cmsSec">' +
            '<div class="cmsH">الزمان والمزامنة <b id="cmsOffsetVal" class="cmsMono">+0.0s</b></div>' +
            '<div class="cmsBtns">' +
              '<button data-d="-5">−5s</button><button data-d="-1">−1s</button><button data-d="-0.1">−0.1</button>' +
              '<button data-d="0.1">+0.1</button><button data-d="1">+1s</button><button data-d="5">+5s</button>' +
              '<button id="cmsResetOff">تصفير</button>' +
            '</div>' +
          '</div>' +
          '<div class="cmsSec">' +
            '<div class="cmsH">المظهر</div>' +
            '<div class="cmsHint">ألماني ثم عربي، باللون الأبيض. عند اختيار اللغتين لا يُعرض إلا السطران معًا.</div>' +
            '<label class="cmsRow cmsRange"><span>حجم الخط <b id="cmsFontVal">' + S.fontSize + '</b></span><input type="range" id="cmsFont" min="14" max="42" value="' + S.fontSize + '"></label>' +
            '<label class="cmsRow cmsRange"><span>الفاصل بينهما <b id="cmsGapVal">' + S.subtitleGap + 'px</b></span><input type="range" id="cmsGap" min="3" max="14" value="' + S.subtitleGap + '"></label>' +
            '<label class="cmsRow cmsRange"><span>خلفية الشريط العلوي <b id="cmsBgVal">' + Math.round(S.bgOpacity * 100) + '%</b></span><input type="range" id="cmsBg" min="0" max="100" value="' + Math.round(S.bgOpacity * 100) + '"></label>' +
            '<label class="cmsRow cmsRange"><span>خلفية الشريط العربي <b id="cmsArBgVal">' + Math.round(S.arabicBgOpacity * 100) + '%</b></span><input type="range" id="cmsArBg" min="65" max="100" value="' + Math.round(S.arabicBgOpacity * 100) + '"></label>' +
            '<label class="cmsRow"><span>عرض الترجمة فوق الفيديو</span><input type="checkbox" id="cmsOverlay" ' + (S.useOverlay ? 'checked' : '') + '></label>' +
            '<label class="cmsRow"><span>الترجمة داخل مشغل iPhone الأصلي</span><input type="checkbox" id="cmsIOS" ' + (S.nativeFullscreenFallback ? 'checked' : '') + '></label>' +
            '<label class="cmsRow"><span>مسار أصلي على الأجهزة الأخرى أيضاً</span><input type="checkbox" id="cmsNative" ' + (S.keepNativeTracks ? 'checked' : '') + '></label>' +
            '<div class="cmsHint">في ملء شاشة Safari افتح Untertitel واختر DE + AR للغتين أو Deutsch للألمانية فقط. تبقى الترجمتان الأصليتان للعربية والإنجليزية متاحتين.</div>' +
          '</div>' +
          '<div class="cmsSec">' +
            '<div class="cmsH">المحركات <small>(المفاتيح تُضبط داخل كود السكربت)</small></div>' +
            '<label class="cmsRow"><span>OpenSubtitles</span><input type="checkbox" data-eng="openSubtitles" ' + (S.engineOn.openSubtitles ? 'checked' : '') + '></label>' +
            '<label class="cmsRow"><span>SubDL</span><input type="checkbox" data-eng="subdl" ' + (S.engineOn.subdl ? 'checked' : '') + '></label>' +
            '<label class="cmsRow"><span>SubSource</span><input type="checkbox" data-eng="subsource" ' + (S.engineOn.subsource ? 'checked' : '') + '></label>' +
            '<label class="cmsRow"><span>SubtitleCat <em>(بلا مفتاح؛ قد تكون الترجمة آلية)</em></span><input type="checkbox" data-eng="subtitleCat" ' + (S.engineOn.subtitleCat ? 'checked' : '') + '></label>' +
            '<div class="cmsHint">Podnapisi متوقف منذ 2026. يُعرض اسمه للتوضيح فقط ولا تُرسل إليه طلبات.</div>' +
          '</div>' +
          '<div class="cmsSec cmsBtnsWide">' +
            '<button id="cmsRescan" class="cmsPrimary">🔄 إعادة البحث الآن</button>' +
            '<button id="cmsClear">مسح الكاش</button>' +
            '<button id="cmsLogBtn">السجل</button>' +
          '</div>' +
          '<pre class="cmsLog" id="cmsLog" hidden></pre>' +
          '<div class="cmsFoot">Cinemana Studio — v' + VERSION + '</div>' +
        '</div>';
      document.body.appendChild(panel);
      const on = (id, fn) => panel.querySelector(id).addEventListener('change', fn);
      // مزامنة أزرار وضع الترجمة مع حالة اللغات
      const syncMode = () => panel.querySelectorAll('.cmsSeg button[data-mode]').forEach(b =>
        b.classList.toggle('active', (b.dataset.mode === 'both') === !!S.langs.ar));
      on('#cmsAuto', e => { S.auto = e.target.checked; saveSettings(); if (S.auto) Flow.start(true); else Flow.stopIdle('البحث التلقائي متوقف'); });
      on('#cmsShow', e => { S.show = e.target.checked; saveSettings(); applyResults(false); });
      on('#cmsDe', e => { S.langs.de = e.target.checked; saveSettings(); syncMode(); Flow.start(true); });
      on('#cmsAr', e => { S.langs.ar = e.target.checked; saveSettings(); syncMode(); Flow.start(true); });
      // زر وضع الترجمة: ألماني + عربي، أو ألماني فقط
      panel.querySelectorAll('.cmsSeg button[data-mode]').forEach(b =>
        b.addEventListener('click', () => {
          const both = b.dataset.mode === 'both';
          S.langs.de = true;
          S.langs.ar = both;
          saveSettings();
          panel.querySelector('#cmsDe').checked = true;
          panel.querySelector('#cmsAr').checked = both;
          syncMode();
          Flow.start(false);
          applyResults(false);
        }));
      on('#cmsOverlay', e => { S.useOverlay = e.target.checked; saveSettings(); applyResults(false); });
      on('#cmsIOS', e => { S.nativeFullscreenFallback = e.target.checked; saveSettings(); applyResults(false); if (shouldAttachNative()) Flow.start(false); });
      on('#cmsNative', e => { S.keepNativeTracks = e.target.checked; saveSettings(); applyResults(false); if (shouldAttachNative()) Flow.start(false); });
      panel.querySelectorAll('input[data-eng]').forEach(i =>
        i.addEventListener('change', () => {
          S.engineOn[i.dataset.eng] = i.checked; saveSettings();
          const eng = Engines[i.dataset.eng];
          ui.engine(eng.id, eng.available() ? 'جاهز؛ اضغط إعادة البحث' : eng.reason(), eng.available() ? 'ok' : 'warn');
        }));
      panel.querySelector('#cmsFont').addEventListener('input', e => { S.fontSize = +e.target.value; panel.querySelector('#cmsFontVal').textContent = S.fontSize; saveSettings(); paintOverlayStyle(); });
      panel.querySelector('#cmsGap').addEventListener('input', e => { S.subtitleGap = +e.target.value; panel.querySelector('#cmsGapVal').textContent = S.subtitleGap + 'px'; saveSettings(); paintOverlayStyle(); });
      panel.querySelector('#cmsBg').addEventListener('input', e => { S.bgOpacity = e.target.value / 100; panel.querySelector('#cmsBgVal').textContent = e.target.value + '%'; saveSettings(); paintOverlayStyle(); });
      panel.querySelector('#cmsArBg').addEventListener('input', e => { S.arabicBgOpacity = e.target.value / 100; panel.querySelector('#cmsArBgVal').textContent = e.target.value + '%'; saveSettings(); paintOverlayStyle(); });
      panel.querySelectorAll('.cmsBtns button[data-d]').forEach(b =>
        b.addEventListener('click', () => { S.offset = clamp(Math.round((S.offset + parseFloat(b.dataset.d)) * 10) / 10, -30, 30); saveSettings(); ui.setOffset(S.offset); refreshNativeTracks(); }));
      panel.querySelector('#cmsResetOff').addEventListener('click', () => { S.offset = 0; saveSettings(); ui.setOffset(0); refreshNativeTracks(); });
      panel.querySelector('#cmsRescan').addEventListener('click', () => Flow.start(true, true));
      panel.querySelector('#cmsClear').addEventListener('click', () => { Cache.clear(); ui.chip('الكاش مسح — أعد البحث', 'warn'); });
      panel.querySelector('#cmsLogBtn').addEventListener('click', () => { S.showLog = !S.showLog; const l = panel.querySelector('#cmsLog'); l.hidden = !S.showLog; if (S.showLog) l.textContent = log._buf.map(x => new Date(x.t).toLocaleTimeString('ar-EG') + ' ' + x.line).join('\n'); });
      panel.querySelector('#cmsClose').addEventListener('click', () => ui.toggle(false));
      // سحب اللوحة
      const head = panel.querySelector('#cmsHead');
      let drag = null;
      head.addEventListener('mousedown', e => {
        const r = panel.getBoundingClientRect();
        drag = { dx: e.clientX - r.left, dy: e.clientY - r.top };
        e.preventDefault();
      });
      window.addEventListener('mousemove', e => {
        if (!drag) return;
        panel.style.left = clamp(e.clientX - drag.dx, 0, innerWidth - 120) + 'px';
        panel.style.top = clamp(e.clientY - drag.dy, 0, innerHeight - 60) + 'px';
        panel.style.bottom = 'auto';
      });
      window.addEventListener('mouseup', () => { drag = null; S.panelOpen = panel.style.display !== 'none'; saveSettings(); });
      ui = {
        panel, fab,
        toggle(open) { panel.style.display = open ? 'block' : 'none'; S.panelOpen = !!open; saveSettings(); },
        chip(text, cls) { const c = panel.querySelector('#cmsChip'); c.textContent = text; c.className = 'cmsChip ' + (cls || ''); },
        setOffset(v) { panel.querySelector('#cmsOffsetVal').textContent = (v >= 0 ? '+' : '') + v.toFixed(1) + 's'; },
        content(t) { panel.querySelector('#cmsContent').textContent = t; },
        videoDur(t) { panel.querySelector('#cmsVideoDur').textContent = t; },
        sub(lang, text) { const id = lang === 'de' ? '#cmsSubDe' : '#cmsSubAr'; panel.querySelector(id).querySelector('b').textContent = text; },
        match(lang, text, cls) {
          const row = panel.querySelector(lang === 'de' ? '#cmsMatchDe' : '#cmsMatchAr');
          const value = row.querySelector('b');
          value.textContent = text; value.title = text;
          row.className = 'cmsKV ' + (cls || '');
        },
        native(text, cls) {
          const row = panel.querySelector('#cmsNativeStatus');
          row.querySelector('b').textContent = text;
          row.className = 'cmsKV ' + (cls || '');
        },
        engine(id, state, cls) {
          const box = panel.querySelector('#cmsEngines');
          let row = box.querySelector('[data-eng-row="' + id + '"]');
          if (!row) { row = el('div', 'cmsKV', '<span>' + (Engines[id] ? Engines[id].label : id) + '</span><b>—</b>'); row.dataset.engRow = id; box.appendChild(row); }
          row.querySelector('b').textContent = state;
          row.className = 'cmsKV ' + (cls || '');
        },
      };
      for (const id of S.engineOrder) {
        if (!Engines[id].available()) ui.engine(id, Engines[id].reason(), 'warn');
      }
      syncMode();
      if (S.panelOpen) panel.style.display = 'block';
      ui.setOffset(S.offset);
    } catch (e) { log.error('فشل بناء الواجهة:', e.message); }
  }
  function el(tag, cls, html) { const n = document.createElement(tag); if (cls) n.className = cls; if (html != null) n.innerHTML = html; return n; }
  function injectStyle(css) { const s = el('style'); s.textContent = css; (document.head || document.documentElement).appendChild(s); }
  const UI_CSS = `
    #cmsSubOverlay{
      position:fixed;z-index:2147483000;pointer-events:none;display:none;box-sizing:border-box;
      isolation:isolate;direction:ltr;font-family:Arial,Tahoma,'Noto Naskh Arabic',sans-serif;
      --cms-font:22px;--cms-gap:6px;
      --cms-main-bg:rgba(0,0,0,.58);--cms-ar-bg:rgba(0,0,0,.94);
    }
    #cmsSubOverlay .cmsCaptionStage{
      position:absolute;inset-inline:0;bottom:8%;display:flex;flex-direction:column;
      align-items:center;gap:var(--cms-gap);width:100%;
      text-align:center;line-height:1.3;box-sizing:border-box;
    }
    #cmsSubOverlay .cmsCaptionPrimary,#cmsSubOverlay .cmsCaptionSecondary{
      box-sizing:border-box;margin:0;min-width:0;width:max-content;max-width:calc(100% - 18px);
      padding:.1em .55em .15em;border-radius:6px;overflow-wrap:anywhere;
      text-align:center;font-weight:800;letter-spacing:0;line-height:1.34;
      font-size:var(--cms-font);color:#fff !important;
      text-shadow:0 2px 3px #000,0 0 9px rgba(0,0,0,.88);
    }
    #cmsSubOverlay .cmsCaptionPrimary{
      background:var(--cms-main-bg);direction:ltr;unicode-bidi:isolate;
    }
    #cmsSubOverlay .cmsCaptionSecondary{
      background:var(--cms-ar-bg);direction:rtl;unicode-bidi:isolate;
    }
    #cmsSubOverlay .cmsCaptionStage.cmsPairEnter{animation:cmsPairIn .16s ease-out both;}
    @keyframes cmsPairIn{from{opacity:.7;transform:translateY(2px)}to{opacity:1;transform:translateY(0)}}
    @media (prefers-reduced-motion: reduce){#cmsSubOverlay .cmsCaptionStage.cmsPairEnter{animation:none;}}
    .cmsFAB{
      position:fixed;left:16px;bottom:16px;z-index:2147483100;width:46px;height:46px;border-radius:50%;
      background:#df8a1c;color:#1c1610;border:2px solid #1c1610;font-weight:800;font-size:15px;cursor:pointer;
      box-shadow:3px 3px 0 rgba(28,22,16,.85);transition:transform .15s ease;
    }
    .cmsFAB:hover{transform:scale(1.07) rotate(-4deg);}
    .cmsFAB.on{background:#3fae6a;}
    .cmsPanel{
      position:fixed;left:16px;bottom:72px;z-index:2147483100;width:322px;max-height:74vh;display:none;
      background:#17130e;color:#f3ead8;border:2px solid #000;border-radius:14px;overflow:hidden;
      box-shadow:6px 6px 0 rgba(0,0,0,.45);font-family:'Segoe UI',Tahoma,'Noto Naskh Arabic',sans-serif;
      direction:rtl;
    }
    .cmsPanel .cmsHead{display:flex;justify-content:space-between;align-items:center;padding:10px 12px;background:#241d15;cursor:move;user-select:none;border-bottom:1px solid #3a3128;}
    .cmsPanel .cmsTitle{font-weight:700;font-size:14px;}
    .cmsPanel .cmsTitle small{opacity:.55;font-weight:400;}
    .cmsPanel .cmsClose{background:none;border:0;color:#f3ead8;font-size:20px;cursor:pointer;opacity:.7;}
    .cmsPanel .cmsClose:hover{opacity:1;}
    .cmsPanel .cmsBody{padding:10px 12px;overflow-y:auto;max-height:calc(74vh - 44px);}
    .cmsPanel .cmsSec{padding:9px 0;border-bottom:1px dashed #3a3128;}
    .cmsPanel .cmsHint{font-size:11px;color:#d5ae78;line-height:1.5;padding:4px 0;}
    .cmsPanel .cmsSec:last-of-type{border-bottom:0;}
    .cmsPanel .cmsH{font-size:12px;font-weight:700;color:#df8a1c;margin-bottom:7px;display:flex;justify-content:space-between;}
    .cmsPanel .cmsH small{opacity:.5;font-weight:400;}
    .cmsPanel .cmsRow{display:flex;justify-content:space-between;align-items:center;gap:8px;font-size:13px;padding:4px 0;cursor:pointer;}
    .cmsPanel .cmsRow em{opacity:.55;font-style:normal;font-size:11px;}
    .cmsPanel .cmsRow input[type=checkbox]{width:17px;height:17px;accent-color:#df8a1c;cursor:pointer;}
    .cmsPanel .cmsRow input[type=color]{width:38px;height:24px;border:1px solid #3a3128;background:none;cursor:pointer;}
    .cmsPanel .cmsRow input[type=range]{width:150px;accent-color:#df8a1c;}
    .cmsPanel .cmsRange b{font-variant-numeric:tabular-nums;}
    .cmsPanel .cmsChip{display:inline-block;padding:3px 10px;border-radius:20px;font-size:12px;font-weight:700;background:#3a3128;color:#f3ead8;}
    .cmsPanel .cmsChip.ok{background:#1d4030;color:#8fe0b0;}
    .cmsPanel .cmsChip.warn{background:#4a3a17;color:#ffd97a;}
    .cmsPanel .cmsChip.err{background:#4a1f1a;color:#ff9d90;}
    .cmsPanel .cmsChip.busy{background:#26344a;color:#9ec5ff;}
    .cmsPanel .cmsKV{display:flex;justify-content:space-between;gap:8px;font-size:12px;padding:3px 0;}
    .cmsPanel .cmsKV span{opacity:.65;}
    .cmsPanel .cmsKV b{font-weight:600;text-align:left;direction:ltr;max-width:190px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}
    .cmsPanel .cmsKV.ok b{color:#8fe0b0;}
    .cmsPanel .cmsKV.err b{color:#ff9d90;}
    .cmsPanel .cmsKV.warn b{color:#ffd97a;}
    .cmsPanel .cmsBtns{display:flex;flex-wrap:wrap;gap:6px;}
    .cmsPanel .cmsBtnsWide{display:flex;flex-direction:column;gap:7px;}
    .cmsPanel .cmsBtns button,.cmsPanel .cmsBtnsWide button{
      background:#241d15;color:#f3ead8;border:1px solid #3a3128;border-radius:8px;padding:6px 10px;
      font-size:12px;cursor:pointer;font-family:inherit;transition:background .15s;
    }
    .cmsPanel .cmsBtns button:hover,.cmsPanel .cmsBtnsWide button:hover{background:#31281e;}
    .cmsPanel .cmsPrimary{background:#df8a1c !important;color:#17130e !important;font-weight:700;}
    .cmsPanel .cmsSeg{display:grid;grid-template-columns:1fr 1fr;gap:6px;margin:4px 0 10px;}
    .cmsPanel .cmsSeg button{
      background:#241d15;color:#f3ead8;border:1px solid #3a3128;border-radius:8px;padding:8px 6px;
      font-size:12px;cursor:pointer;font-family:inherit;transition:background .15s;
    }
    .cmsPanel .cmsSeg button:hover{background:#31281e;}
    .cmsPanel .cmsSeg button.active{background:#df8a1c;color:#17130e;border-color:#df8a1c;font-weight:700;}
    .cmsPanel .cmsLog{
      background:#0e0b07;border:1px solid #3a3128;border-radius:8px;padding:8px;font-size:10.5px;
      max-height:160px;overflow-y:auto;direction:ltr;text-align:left;white-space:pre-wrap;line-height:1.5;
    }
    .cmsPanel .cmsMono{font-variant-numeric:tabular-nums;color:#8fe0b0;}
    .cmsPanel .cmsFoot{padding:8px 0 2px;font-size:10.5px;opacity:.45;text-align:center;}
    @media(max-width:480px){
      .cmsPanel{left:12px;bottom:calc(66px + env(safe-area-inset-bottom));width:min(360px,calc(100vw - 24px));max-height:min(70dvh,620px);}
      .cmsPanel .cmsBody{max-height:calc(min(70dvh,620px) - 46px);}
      .cmsFAB{bottom:calc(12px + env(safe-area-inset-bottom));left:12px;}
    }
  `;
  /* ───────────────────── طبقة العرض: زوج واحد لا يظهر إلا كاملاً ───────────────────── */
  let overlay = null, stageEl = null, deEl = null, arEl = null;
  let lastPairText = null, lastOverlayWidth = 0;
  function ensureOverlay() {
    if (overlay && overlay.isConnected) return;
    overlay = el('div');
    overlay.id = 'cmsSubOverlay';
    overlay.setAttribute('aria-live', 'off');
    stageEl = el('div', 'cmsCaptionStage');
    deEl = el('div', 'cmsCaptionPrimary');
    arEl = el('div', 'cmsCaptionSecondary');
    deEl.dir = 'ltr'; deEl.lang = 'de';
    arEl.dir = 'rtl'; arEl.lang = 'ar';
    stageEl.appendChild(deEl); stageEl.appendChild(arEl);
    overlay.appendChild(stageEl);
    document.body.appendChild(overlay);
    lastPairText = null; lastOverlayWidth = 0;
    paintOverlayStyle();
  }
  function paintOverlayStyle(width) {
    if (!overlay) return;
    const w = width || lastOverlayWidth || (Flow.video && Flow.video.getBoundingClientRect().width) || 640;
    const font = clamp(Math.round(S.fontSize * clamp(w / 640, .9, 2.8)), 13, 98);
    overlay.style.setProperty('--cms-font', font + 'px');
    overlay.style.setProperty('--cms-gap', S.subtitleGap + 'px');
    overlay.style.setProperty('--cms-main-bg', 'rgba(0,0,0,' + S.bgOpacity + ')');
    overlay.style.setProperty('--cms-ar-bg', 'rgba(0,0,0,' + S.arabicBgOpacity + ')');
  }
  function positionOverlay(v) {
    const r = v.getBoundingClientRect();
    if (r.width * r.height < 900) { overlay.style.display = 'none'; return false; }
    if (Math.abs(r.width - lastOverlayWidth) > 6) {
      lastOverlayWidth = r.width;
      paintOverlayStyle(r.width);
    }
    overlay.style.top = '0px'; overlay.style.left = '0px';
    overlay.style.width = r.width + 'px'; overlay.style.height = r.height + 'px';
    // إعادة تموضع داخل الفيديو نفسه إذا أمكن (تثبيت مع تمرير الصفحة)
    const parent = v.closest('.js-video, .player, [class*="player"], [class*="video"]') || v.parentElement;
    if (parent && getComputedStyle(parent).position === 'relative') {
      overlay.style.position = 'absolute';
      const pr = parent.getBoundingClientRect();
      overlay.style.top = (r.top - pr.top) + 'px';
      overlay.style.left = (r.left - pr.left) + 'px';
      if (overlay.parentElement !== parent) parent.appendChild(overlay);
    } else {
      overlay.style.position = 'fixed';
      overlay.style.top = r.top + 'px';
      overlay.style.left = r.left + 'px';
      if (overlay.parentElement !== document.body) document.body.appendChild(overlay);
    }
    return true;
  }
  /* ───────────────────── مسار أصلي للترجمتين، خاصة ملء شاشة Safari ───────────────────── */
  const nativeTrackVideos = new WeakSet();
  const iosNativeStates = new WeakMap();
  const suppressedTracks = new WeakMap();
  function shouldAttachNative() { return S.keepNativeTracks || (IS_IOS && S.nativeFullscreenFallback); }
  function wantsBoth() { return S.langs.de && S.langs.ar; }
  function pairedNativeCues(deCues, arCues, requireBoth = wantsBoth()) {
    const times = [...new Set([...deCues, ...arCues].flatMap(c => [c.s, c.e]))].sort((a, b) => a - b);
    const deAt = makeSeeker(deCues), arAt = makeSeeker(arCues);
    const merged = [];
    for (let i = 0; i < times.length - 1; i++) {
      const a = times[i], b = times[i + 1];
      if (b <= a) continue;
      const de = deAt((a + b) / 2), ar = arAt((a + b) / 2);
      if (requireBoth ? (!de || !ar) : (!de && !ar)) continue;
      const text = (de ? '\u2066' + de + '\u2069' : '') +
        (de && ar ? '\n' : '') + (ar ? '\u2067' + ar + '\u2069' : '');
      const previous = merged[merged.length - 1];
      if (previous && previous.text === text && previous.e === a) previous.e = b;
      else merged.push({ s: a, e: b, text });
    }
    return merged.map(c => ({ s: Math.max(0, c.s - S.offset), e: c.e - S.offset, text: c.text }))
      .filter(c => c.e > c.s);
  }
  function suppressOriginalTracks(video, active) {
    let saved = suppressedTracks.get(video);
    if (!active) {
      if (saved) {
        for (const [track, mode] of saved) {
          if (track.mode === 'hidden') track.mode = mode;
        }
        saved.clear();
      }
      return;
    }
    if (!saved) { saved = new Map(); suppressedTracks.set(video, saved); }
    for (let i = 0; i < video.textTracks.length; i++) {
      const track = video.textTracks[i];
      if (!isOwnedNativeTrack(track) && track.mode === 'showing') {
        if (!saved.has(track)) saved.set(track, 'showing');
        track.mode = 'hidden';
      }
    }
  }
  function isOwnedNativeTrack(track) {
    const label = String(track && track.label || '');
    return label.startsWith('CineSubs') || label === 'Deutsch' || label === 'DE + AR';
  }
  function setNativeTrackModes(video, nativeFullscreen) {
    if (!video) return;
    const ios = iosNativeStates.get(video);
    if (ios) {
      const hasPair = ios.cueCount > 0;
      const hasGerman = S.langs.de && ios.deCueCount > 0;
      const enabled = S.show && (hasPair || hasGerman) &&
        (!S.useOverlay || nativeFullscreen);
      const showPair = enabled && wantsBoth() && hasPair;
      const showGerman = enabled && !showPair && hasGerman;
      if (showPair || showGerman) suppressOriginalTracks(video, true);
      if (ios.track) ios.track.mode = showPair ? 'showing' : 'hidden';
      if (ios.deTrack) ios.deTrack.mode = showGerman ? 'showing' : 'hidden';
      if (!showPair && !showGerman) suppressOriginalTracks(video, false);
      return;
    }
    const ready = Flow.activeLangs().length > 0 && Flow.activeLangs().every(lang => !!Flow.results[lang]);
    const enabled = S.show && ready && (!S.useOverlay || nativeFullscreen) &&
      (!wantsBoth() || !!Flow.blobUrls.combined);
    const hasBoth = wantsBoth() && Flow.results.de && Flow.results.ar;
    const single = Flow.results.de && S.langs.de ? 'de' : 'ar';
    if (enabled) suppressOriginalTracks(video, true);
    for (let i = 0; i < video.textTracks.length; i++) {
      const track = video.textTracks[i];
      if (!String(track.label || '').startsWith('CineSubs')) continue;
      track.mode = enabled && (hasBoth ? track.label === 'CineSubs DE + AR' :
        !wantsBoth() && track.language === single)
        ? 'showing' : 'hidden';
    }
    if (!enabled) suppressOriginalTracks(video, false);
  }
  function bindNativeTrackEvents(video) {
    if (nativeTrackVideos.has(video)) return;
    nativeTrackVideos.add(video);
    video.addEventListener('webkitbeginfullscreen', () => setNativeTrackModes(video, true));
    video.addEventListener('webkitendfullscreen', () => setNativeTrackModes(video, false));
    video.addEventListener('fullscreenchange', () => setNativeTrackModes(video, !!document.fullscreenElement));
  }
  function attachNativeTracks(video) {
    try {
      if (!video || !shouldAttachNative()) return;
      const deCues = Flow.results.de ? Flow.results.de.cues : [];
      const arCues = Flow.results.ar ? Flow.results.ar.cues : [];
      const combined = pairedNativeCues(deCues, arCues);
      if (IS_IOS) {
        const germanOnly = deCues.map(c => ({ s: Math.max(0, c.s - S.offset),
          e: c.e - S.offset, text: c.text })).filter(c => c.e > c.s);
        const pairCues = deCues.length && arCues.length
          ? pairedNativeCues(deCues, arCues, true) : [];
        let state = iosNativeStates.get(video);
        if (!state && !pairCues.length && !germanOnly.length) {
          suppressOriginalTracks(video, false);
          if (ui) ui.native('بانتظار تحميل الترجمة', 'warn');
          return;
        }
        const Cue = window.VTTCue || window.WebKitVTTCue;
        if (typeof video.addTextTrack !== 'function' || !Cue) {
          if (ui) ui.native('لا يدعم هذا المتصفح مسارات JavaScript', 'warn');
          return;
        }
        if (!state) {
          // Safari iPhone needs JavaScript text tracks; provide the pair and German-only choices.
          state = { track: null, deTrack: null, cueCount: 0, deCueCount: 0 };
          iosNativeStates.set(video, state);
        }
        const fillTrack = (slot, label, lang, cues, keepEmpty = false) => {
          let track = state[slot];
          if (!track && (cues.length || keepEmpty)) track = state[slot] = video.addTextTrack('subtitles', label, lang);
          if (!track) return 0;
          track.mode = 'hidden';
          if (track.cues) for (const cue of Array.from(track.cues)) track.removeCue(cue);
          let count = 0;
          for (const c of cues) {
            try {
              const cue = new Cue(c.s, c.e, c.text);
              cue.align = 'center';
              track.addCue(cue);
              count++;
            } catch (e) { log.warn('تعذر إضافة مقطع إلى مشغل Safari:', e.message); }
          }
          return count;
        };
        state.deCueCount = fillTrack('deTrack', 'Deutsch', 'de', germanOnly);
        state.cueCount = fillTrack('track', 'DE + AR', 'de', pairCues,
          deCues.length > 0 && arCues.length > 0);
        const statusParts = [];
        if (state.cueCount) statusParts.push(state.cueCount + ' مقطع DE + AR');
        if (state.deCueCount) statusParts.push(state.deCueCount + ' مقطع Deutsch');
        if (ui) ui.native(statusParts.length ? 'جاهز: ' + statusParts.join(' + ') :
          'بانتظار تحميل الترجمة', statusParts.length ? 'ok' : 'warn');
        if (!statusParts.length) {
          suppressOriginalTracks(video, false);
          return;
        }
        bindNativeTrackEvents(video);
        setNativeTrackModes(video, !!video.webkitDisplayingFullscreen);
        return;
      }
      video.querySelectorAll('track[data-cms]').forEach(t => t.remove());
      for (const url of Object.values(Flow.blobUrls)) URL.revokeObjectURL(url);
      Flow.blobUrls = {};
      for (const lang of Flow.activeLangs()) {
        const res = Flow.results[lang];
        if (!res) continue;
        const shifted = res.cues.map(c => ({ s: Math.max(0, c.s - S.offset),
          e: c.e - S.offset, text: c.text })).filter(c => c.e > c.s);
        const blob = new Blob(['WEBVTT\n\n' + cuesToVtt(shifted)], { type: 'text/vtt;charset=utf-8' });
        Flow.blobUrls[lang] = URL.createObjectURL(blob);
        const tr = document.createElement('track');
        tr.kind = 'subtitles'; tr.label = 'CineSubs — ' + (LANG_NAME[lang] || lang);
        tr.srclang = lang; tr.src = Flow.blobUrls[lang];
        tr.setAttribute('data-cms', '1');
        video.appendChild(tr);
      }
      if (combined.length && wantsBoth()) {
        const blob = new Blob(['WEBVTT\n\n' + cuesToVtt(combined)], { type: 'text/vtt;charset=utf-8' });
        Flow.blobUrls.combined = URL.createObjectURL(blob);
        const tr = document.createElement('track');
        tr.kind = 'subtitles'; tr.label = 'CineSubs DE + AR';
        tr.srclang = 'de'; tr.src = Flow.blobUrls.combined;
        tr.setAttribute('data-cms', '1');
        video.appendChild(tr);
      }
      bindNativeTrackEvents(video);
      requestAnimationFrame(() => setNativeTrackModes(video, !!video.webkitDisplayingFullscreen));
    } catch (e) {
      log.warn('المسارات الأصلية:', e.message);
      if (IS_IOS && ui) ui.native('تعذر تجهيز مسار Safari', 'err');
    }
  }
  function detachNativeTracks(video) {
    try {
      if (!video) return;
      suppressOriginalTracks(video, false);
      const state = iosNativeStates.get(video);
      if (state) {
        for (const track of [state.track, state.deTrack]) {
          if (!track) continue;
          track.mode = 'hidden';
          if (track.cues) for (const cue of Array.from(track.cues)) track.removeCue(cue);
        }
        state.cueCount = 0;
        state.deCueCount = 0;
      }
      video.querySelectorAll('track[data-cms]').forEach(t => t.remove());
      for (const k in Flow.blobUrls) { if (Flow.blobUrls[k]) URL.revokeObjectURL(Flow.blobUrls[k]); }
      Flow.blobUrls = {};
    } catch (e) { /* تجاهل */ }
  }
  function refreshNativeTracks() {
    const video = Flow.video || findVideo();
    if (video && shouldAttachNative()) attachNativeTracks(video);
  }
  function cuesToVtt(cues) {
    const f = t => {
      const h = Math.floor(t / 3600), m = Math.floor((t % 3600) / 60), s = Math.floor(t % 60), ms = Math.round((t % 1) * 1000);
      return String(h).padStart(2, '0') + ':' + String(m).padStart(2, '0') + ':' +
        String(s).padStart(2, '0') + '.' + String(ms).padStart(3, '0');
    };
    return cues.map((c, i) => (i + 1) + '\n' + f(c.s) + ' --> ' + f(c.e) + '\n' + c.text).join('\n\n');
  }
  /* ───────────────────── التدفق الرئيسي ───────────────────── */
  const Flow = {
    content: null,
    contentKey: null,
    lastKey: null,          // آخر عمل اكتمل بنجاح (إعادة فورية من الكاش)
    video: null,
    results: { de: null, ar: null },
    seekers: { de: null, ar: null },
    badDownloads: new Set(),
    incompatibleKey: null,
    blobUrls: {},
    running: false,
    generation: 0,
    durationValidated: false,
    retryTimer: null,
    missingRetries: 0,
    raf: null,
    activeLangs() {
      const l = [];
      const order = [S.primary, S.secondary].filter((v, i, a) => a.indexOf(v) === i);
      for (const lang of order) if (S.langs[lang]) l.push(lang);
      return l;
    },
    stopIdle(msg) {
      this.running = false; this.generation++;
      clearTimeout(this.retryTimer);
      ui && ui.chip(msg || 'متوقف', 'warn');
    },
    start(auto, force) {
      if (auto && !S.auto && !force) return;
      const langs = this.activeLangs();
      if (!langs.length) { ui.chip('فعّل لغة واحدة على الأقل', 'err'); return; }
      const searchLangs = IS_IOS && shouldAttachNative() ? ['de', 'ar'] : langs;
      const content = detectContent();
      const video = findVideo();
      if (!content) {
        ui.chip('لم يُكتشف اسم العمل من الصفحة بعد', 'warn');
        ui.content('—');
        return;
      }
      ui.content(content.name + (content.year ? ' (' + content.year + ')' : '') +
        (content.imdbId ? ' | ' + content.imdbId : ''));
      if (!video) {
        this.video = null;
        ui.videoDur('غير موجود');
        ui.chip('بانتظار فتح/تشغيل الفيديو قبل البحث', 'warn');
        return;
      }
      const key = keyOf(content);
      if (this.running && this.contentKey === key && !force) return;
      if (this.contentKey !== key) {
        this.resetForNewContent();
        this.missingRetries = 0;
      } else if (force) {
        Cache.clearFor(key);
        this.results = { de: null, ar: null };
        this.seekers = { de: null, ar: null };
        this.badDownloads.clear();
        this.incompatibleKey = null;
        detachNativeTracks(this.video);
      }
      clearTimeout(this.retryTimer);
      this.running = true;
      const token = ++this.generation;
      this.content = content;
      this.contentKey = key;
      this.video = video;
      const dur = videoDuration(video);
      this.durationValidated = !!dur;
      ui.videoDur(dur ? (Math.round(dur / 60) + ' دقيقة — ' + Math.round(dur) + 's') : '—');
      ui.chip('جارٍ البحث: ' + searchLangs.map(l => LANG_NAME[l]).join(' + '), 'busy');
      for (const l of searchLangs) {
        if (this.results[l]) continue;
        const c = Cache.getSub(key, l);
        if (!c) continue;
        const cues = parseCues(c.text);
        if (!cues.length) continue;
        if (content.imdbId && c.imdbId && imdbNumber(content.imdbId) !== imdbNumber(c.imdbId)) continue;
        if (content.year && c.year && Math.abs(Number(c.year) - content.year) > 1) continue;
        if (!runtimeFit(cues, dur).ok) continue;
        this.results[l] = { text: c.text, cues, title: c.title, engine: c.engine,
          machine: !!c.machine, cached: true, score: c.score,
          imdbId: c.imdbId, year: c.year, matchReason: c.matchReason,
          subtitleDuration: c.subtitleDuration };
        this.seekers[l] = makeSeeker(cues);
        ui.match(l, c.matchReason || 'ملف محفوظ واجتاز فحص المدة', 'ok');
        log.info('✓ ' + LANG_NAME[l] + ' من الكاش:', c.title);
      }
      if (searchLangs.every(l => this.results[l])) {
        this.running = false;
        this.lastKey = key;
        this.applyLangs({}, key);
        ui.chip('✓ الترجمة جاهزة (من الكاش)', 'ok');
        ui.fab.classList.add('on');
        return;
      }
      if (Object.keys(this.results).some(l => this.results[l])) this.applyLangs({}, key);
      void (async () => {
        let attempt = 0;
        while (attempt < Math.min(2, S.maxRetries)) {
          if (token !== this.generation || !this.running) return;
          attempt++;
          try {
            await this.runEngines(searchLangs, key, dur, token);
          } catch (e) { log.error('خطأ عام في الدورة:', e.message); }
          if (token !== this.generation || !this.running) return;
          if (this.incompatibleKey === key) break;
          const missing = searchLangs.filter(l => !this.results[l]);
          if (!missing.length) break;
          const canRetry = S.engineOrder.some(id => Engines[id].available() && !Guard.cool(id));
          if (!canRetry || attempt >= Math.min(2, S.maxRetries)) break;
          if (attempt < S.maxRetries) {
            const wait = BACKOFF[Math.min(attempt - 1, BACKOFF.length - 1)];
            ui.chip('إعادة بحث محدودة بعد ' + (wait / 1000) + ' ثوانٍ', 'warn');
            await sleep(wait);
          }
        }
        if (token !== this.generation || !this.running) return;
        this.running = false;
        this.finish(searchLangs, key, dur);
      })().catch(e => { if (token === this.generation) { this.running = false; ui.chip('خطأ بحث: ' + e.message, 'err'); } });
    },
    async runEngines(langs, key, videoDur, token) {
      if (!this.content || !isUsableMovieTitle(this.content.name)) {
        ui.chip('لم يُحدد اسم الفيلم، لا يمكن البحث في مصادر الترجمة', 'err');
        return;
      }
      const order = S.engineOrder.map(id => Engines[id]).filter(e => e && e.available());
      if (!order.length) { ui.chip('لا محرك متاح — فعّل محركاً أو أضف مفتاح API', 'err'); return; }
      const candidates = [];
      for (const eng of order) {
        if (token !== this.generation || langs.every(l => this.results[l])) break;
        if (Guard.cool(eng.id)) { ui.engine(eng.id, 'تبريد (' + Guard.coolMin(eng.id) + ' د)', 'warn'); continue; }
        if (eng.id === 'subtitleCat' && langs.every(lang => this.results[lang] ||
          candidates.some(c => c.lang === lang && scoreCandidate(c, this.content).total >= 118))) {
          ui.engine(eng.id, 'تخطي: نتائج أقوى موثقة', 'ok');
          continue;
        }
        try {
          ui.engine(eng.id, 'يبحث…', 'busy');
          let cands = Cache.getSearch(key, eng.id, langs);
          if (cands === null) {
            cands = await eng.search(this.content, langs);
            if (token !== this.generation) return;
            Cache.setSearch(key, eng.id, cands, langs);
          }
          const plausible = (cands || []).filter(c =>
            langs.includes(c.lang) && scoreCandidate(c, this.content).total >= S.minScore);
          ui.engine(eng.id, (cands || []).length + ' مرشح، ' + plausible.length + ' مطابق أولياً',
            plausible.length ? 'busy' : 'warn');
          if (!cands || !cands.length) { ui.engine(eng.id, 'لا نتائج', 'warn'); continue; }
          candidates.push(...plausible);
        } catch (err) {
          if (token !== this.generation) return;
          Guard.fail(eng.id, err);
          ui.engine(eng.id, '✗ ' + err.message.slice(0, 40), 'err');
          log.error('المحرك', eng.label, err.message);
        }
      }
      if (token !== this.generation) return;
      const seen = new Set();
      const unique = candidates.filter(c => {
        const id = c.engine + '|' + c.lang + '|' + JSON.stringify(c.dl);
        if (seen.has(id)) return false;
        seen.add(id); return true;
      });
      const possible = {
        de: this.results.de ? [this.results.de] : [],
        ar: this.results.ar ? [this.results.ar] : [],
      };
      for (const lang of langs) {
        if (token !== this.generation) return;
        if (this.results[lang]) continue;
        const ranked = unique.filter(c => c.lang === lang)
          .map(c => ({ c, match: scoreCandidate(c, this.content) }))
          .filter(row => row.match.total >= S.minScore)
          .sort((a, b) => b.match.total - a.match.total);
        if (!ranked.length) { ui.match(lang, 'لا توجد نتيجة موثوقة', 'warn'); continue; }
        let attempts = 0;
        const attemptsByEngine = {};
        for (const { c, match } of ranked) {
          if (token !== this.generation || attempts >= 4) break;
          if (Guard.cool(c.engine)) continue;
          if ((attemptsByEngine[c.engine] || 0) >= 2) continue;
          const id = c.engine + '|' + lang + '|' + JSON.stringify(c.dl);
          if (this.badDownloads.has(id)) continue;
          attempts++; attemptsByEngine[c.engine] = (attemptsByEngine[c.engine] || 0) + 1;
          try {
            ui.engine(c.engine, 'يتحقق من: ' + c.title.slice(0, 28), 'busy');
            const text = await Engines[c.engine].download(c, this.content);
            if (token !== this.generation) return;
            const cues = parseCues(text);
            if (cues.length < 10) throw new Error('الملف قصير جداً أو لا يحتوي توقيتات كافية');
            const fit = runtimeFit(cues, videoDur);
            if (!fit.ok) throw new Error(fit.reason);
            const score = match.total + fit.bonus;
            const evidence = [...match.evidence];
            if (fit.gap !== null) evidence.push('فرق مدة ' + Math.round(fit.gap / 60) + ' دقيقة');
            const result = { text, cues, title: c.title, engine: c.engine,
              machine: !!c.machine, score, imdbId: c.imdbId || null,
              year: match.year, subtitleDuration: cues[cues.length - 1].e,
              matchReason: evidence.join('، '), runtimeGap: fit.gap };
            possible[lang].push(result);
            if (!wantsBoth() && match.total >= 118 && fit.gap !== null && fit.gap <= 3 * 60) break;
          } catch (err) {
            if (token !== this.generation) return;
            this.badDownloads.add(id);
            log.warn('استبعاد ترجمة:', c.engine, c.title, err.message);
            if ([401, 403, 406, 429].includes(err.status)) {
              Guard.fail(c.engine, err);
              ui.engine(c.engine, 'توقف مؤقت: ' + err.message.slice(0, 30), 'err');
            }
          }
        }
      }
      let selected = null;
      if (wantsBoth() && possible.de.length && possible.ar.length) {
        let bestPair = null;
        for (const de of possible.de) for (const ar of possible.ar) {
          if (de.imdbId && ar.imdbId && imdbNumber(de.imdbId) !== imdbNumber(ar.imdbId)) continue;
          if (de.year && ar.year && Math.abs(Number(de.year) - Number(ar.year)) > 1) continue;
          const overlap = subtitleOverlapRatio(de.cues, ar.cues);
          if (overlap < 0.12) continue;
          const deRelease = releaseFamily(de.title), arRelease = releaseFamily(ar.title);
          const releaseBonus = deRelease && arRelease ? (deRelease === arRelease ? 9 : -8) : 0;
          const score = de.score + ar.score + Math.round(overlap * 36) + releaseBonus;
          if (!bestPair || score > bestPair.score) bestPair = { de, ar, score, overlap };
        }
        if (bestPair) {
          const note = 'توافق توقيت ' + Math.round(bestPair.overlap * 100) + '%';
          selected = {
            de: { ...bestPair.de, matchReason: bestPair.de.matchReason + '، ' + note },
            ar: { ...bestPair.ar, matchReason: bestPair.ar.matchReason + '، ' + note },
          };
        } else {
          selected = {};
          for (const lang of ['de', 'ar']) {
            const option = possible[lang].sort((a, b) => b.score - a.score)[0];
            if (option) selected[lang] = { ...option,
              matchReason: option.matchReason + '، مسار مستقل؛ لا يتزامن مع اللغة الأخرى' };
            else ui.match(lang, 'لا توجد ترجمة مستقلة صالحة', 'warn');
          }
        }
      }
      if (!selected) {
        selected = {};
        for (const lang of langs) {
          const options = possible[lang].sort((a, b) => b.score - a.score);
          if (options.length) selected[lang] = options[0];
          else ui.match(lang, 'لم يجتز أي ملف فحص الهوية والمدة', 'warn');
        }
      }
      for (const lang of langs) {
        if (token !== this.generation) return;
        const result = selected[lang];
        if (!result) continue;
        this.results[lang] = result;
        this.seekers[lang] = makeSeeker(result.cues);
        ui.match(lang, result.matchReason, 'ok');
        ui.engine(result.engine, '✓ ' + result.title.slice(0, 28), 'ok');
        log.info('اختيرت ' + LANG_NAME[lang] + ':', result.engine, result.title, 'تقييم ' + result.score);
      }
    },
    finish(langs, key, dur) {
      const got = langs.filter(l => this.results[l]);
      const missing = langs.filter(l => !this.results[l]);
      if (got.length) {
        this.lastKey = key;
        for (const l of got) Cache.setSub(key, l, this.results[l]);
        this.applyLangs({}, key);
        const names = got.map(l => LANG_NAME[l]).join(' + ');
        if (missing.length) {
          ui.chip('تم تنزيل ' + names + '؛ ' + missing.map(l => LANG_NAME[l]).join('+') + ' غير متاحة', 'warn');
          this.scheduleMissing(key, missing);
        } else {
          ui.chip('✓ تم تنزيل ' + names, 'ok');
          ui.fab.classList.add('on');
        }
      } else {
        const available = S.engineOrder.filter(id => Engines[id].available());
        ui.chip(this.incompatibleKey === key ? 'ترجمات الفيلم الموجودة لا تتزامن معاً' :
          available.length ? 'لم تُنزّل ترجمة مناسبة؛ راجع حالة المحركات' :
            'لا محرك متاح؛ فعّل SubtitleCat أو أضف مفتاح API', 'err');
        this.scheduleMissing(key, langs);
      }
    },
    scheduleMissing(key, missing) {
      clearTimeout(this.retryTimer);
      if (!S.auto || this.incompatibleKey === key || this.missingRetries >= 2 ||
          !S.engineOrder.some(id => Engines[id].available())) return;
      this.retryTimer = setTimeout(() => {
        if (this.contentKey !== key || !S.auto || !findVideo()) return;
        this.missingRetries++;
        log.info('إعادة بحث محدودة (' + this.missingRetries + '):', missing.join('+'));
        this.start(true);
      }, 15 * 60e3);
    },
    applyLangs(cached, key) {
      // المسارات الأصلية
      const v = findVideo() || this.video;
      if (v) {
        if (shouldAttachNative()) attachNativeTracks(v);
        else detachNativeTracks(v);
      }
      for (const l in this.results) {
        const sub = this.results[l];
        ui.sub(l, sub ? sub.title + ' ← ' + sub.engine + (sub.machine ? ' (آلية)' : '') : '—');
      }
      // إعادة ضبط مؤشرات البحث عند تطبيق جديد
      for (const l of ['de', 'ar']) if (this.results[l]) this.seekers[l] = makeSeeker(this.results[l].cues);
      paintOverlayStyle();
      this.contentKey = key || this.contentKey;
    },
    resetForNewContent() {
      this.generation++;
      this.running = false;
      this.durationValidated = false;
      S.offset = 0; saveSettings();
      if (ui) ui.setOffset(0);
      this.results = { de: null, ar: null };
      this.seekers = { de: null, ar: null };
      this.badDownloads.clear();
      this.incompatibleKey = null;
      clearTimeout(this.retryTimer);
      detachNativeTracks(this.video);
      if (ui) ui.native('بانتظار اللغتين', 'warn');
      this.contentKey = null;
      this.content = null;
      this.video = null;
      if (ui) {
        for (const l of ['de', 'ar']) { ui.sub(l, '—'); ui.match(l, '—'); }
        ui.fab.classList.remove('on');
      }
    },
  };
  /* ───────────────────── حلقة العرض ───────────────────── */
  function hideCaptionPair(video) {
    if (overlay) overlay.style.display = 'none';
    lastPairText = null;
    if (video && !video.webkitDisplayingFullscreen && document.fullscreenElement !== video) {
      let ownTrackShowing = false;
      for (let i = 0; i < video.textTracks.length; i++) {
        const track = video.textTracks[i];
        if (isOwnedNativeTrack(track) && track.mode === 'showing') {
          ownTrackShowing = true;
          break;
        }
      }
      if (!ownTrackShowing) suppressOriginalTracks(video, false);
    }
  }
  function renderLoop() {
    Flow.raf = requestAnimationFrame(renderLoop);
    const v = Flow.video || findVideo();
    if (v) Flow.video = v;
    const v2 = Flow.video;
    if (!v2 || !v2.isConnected) { hideCaptionPair(v2); return; }
    if (!S.show || !S.useOverlay || (!Flow.results.de && !Flow.results.ar)) {
      hideCaptionPair(v2);
      return;
    }
    if (v2.paused || document.hidden || v2.webkitDisplayingFullscreen || document.fullscreenElement === v2) {
      hideCaptionPair(v2);
      return;
    }
    if (wantsBoth() && (!Flow.results.de || !Flow.results.ar)) { hideCaptionPair(v2); return; }
    const t = v2.currentTime + S.offset;
    const de = S.langs.de && Flow.seekers.de ? Flow.seekers.de(t) : null;
    const ar = S.langs.ar && Flow.seekers.ar ? Flow.seekers.ar(t) : null;
    if (wantsBoth() ? (!de || !ar) : (!de && !ar)) { hideCaptionPair(v2); return; }
    ensureOverlay();
    const signature = JSON.stringify([de, ar]);
    if (lastPairText !== signature) {
      lastPairText = signature;
      deEl.textContent = de || '';
      arEl.textContent = ar || '';
      deEl.style.display = de ? '' : 'none';
      arEl.style.display = ar ? '' : 'none';
      stageEl.classList.remove('cmsPairEnter');
      void stageEl.offsetWidth;
      stageEl.classList.add('cmsPairEnter');
    }
    if (!positionOverlay(v2)) { hideCaptionPair(v2); return; }
    overlay.style.display = 'block';
    suppressOriginalTracks(v2, true);
  }
  function applyResults(_silent) {
    paintOverlayStyle();
    const v = Flow.video || findVideo();
    if (v && shouldAttachNative()) attachNativeTracks(v);
    else if (v) detachNativeTracks(v);
  }
  /* ───────────────────── مراقبة الصفحة ───────────────────── */
  let mo = null, pollTimer = null, lastCheckKey = null, domDebounce = null;
  function onDomChanged() {
    try {
      const c = detectContent();
      const key = c ? keyOf(c) : null;
      const v = findVideo() || (Flow.video && Flow.video.isConnected && Flow.video.webkitDisplayingFullscreen
        ? Flow.video : null);
      if (!key && Flow.contentKey) Flow.resetForNewContent();
      if (!v) {
        if (Flow.video) {
          detachNativeTracks(Flow.video);
          if (Flow.contentKey) Flow.stopIdle('المشغل مغلق؛ بانتظار الفيديو');
        }
        Flow.video = null;
        if (c) ui.content(c.raw);
        else ui.content('—');
        ui.videoDur('غير موجود');
        if (!Flow.contentKey) ui.chip('بانتظار فتح/تشغيل الفيديو قبل البحث', 'warn');
        return;
      }
      if (Flow.video !== v) {
        detachNativeTracks(Flow.video);
        Flow.video = v;
        if (key === Flow.contentKey && shouldAttachNative() &&
            (Flow.results.de || Flow.results.ar)) attachNativeTracks(v);
      }
      if (key && key !== Flow.contentKey && S.auto) {
        log.info('اكتُشف مشغل العمل:', c.raw);
        Flow.start(true);
      }
      if (key && key === Flow.contentKey && !Flow.running && !Flow.durationValidated) {
        const duration = videoDuration(v);
        if (duration) {
          Flow.durationValidated = true;
          ui.videoDur(Math.round(duration / 60) + ' دقيقة');
          if (Flow.activeLangs().some(lang => Flow.results[lang] &&
              !runtimeFit(Flow.results[lang].cues, duration).ok)) {
            log.warn('مدة الفيديو ظهرت بعد البحث، إعادة اختيار الترجمات غير المتطابقة');
            Flow.start(true, true);
            return;
          }
        }
      }
      lastCheckKey = key;
    } catch (e) { log.error('onDomChanged:', e.message); }
  }
  function startObservers() {
    try {
      mo = new MutationObserver(() => {
        clearTimeout(domDebounce);
        domDebounce = setTimeout(onDomChanged, 700);
      });
      mo.observe(document.body, { childList: true, subtree: true });
    } catch (e) { log.warn('MutationObserver غير متاح:', e.message); }
    pollTimer = setInterval(() => {
      try {
        if (document.hidden) return;
        onDomChanged();
      } catch (e) { log.error('poll:', e.message); }
    }, 3000);
  }
  /* ───────────────────── الإقلاع ───────────────────── */
  function boot() {
    log.info('CineSubs v' + VERSION + ' جاهز — مصادر بلا مفتاح: SubtitleCat؛ Podnapisi متوقف.');
    buildUI();
    startObservers();
    renderLoop();
    // أوامر قائمة Tampermonkey
    try {
      if (typeof GM_registerMenuCommand === 'function') {
        GM_registerMenuCommand('فتح/إغلاق لوحة CineSubs', () => ui.toggle(S.panelOpen ? false : true));
        GM_registerMenuCommand('تشغيل/إيقاف البحث التلقائي', () => { S.auto = !S.auto; saveSettings(); S.auto ? Flow.start(true) : Flow.stopIdle(); });
        GM_registerMenuCommand('إعادة البحث عن الترجمة', () => Flow.start(true, true));
      }
    } catch (e) { /* تجاهل */ }
    setTimeout(() => { if (S.auto) Flow.start(true); }, 1500);
    window.addEventListener('beforeunload', () => { try { mo && mo.disconnect(); clearInterval(pollTimer); cancelAnimationFrame(Flow.raf); } catch (e) {} });
  }
  if (document.body) boot();
  else document.addEventListener('DOMContentLoaded', boot);
})();