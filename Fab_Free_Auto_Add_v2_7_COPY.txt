// ==UserScript==
// @name         Fab Free Card Auto-Add (v2.7 · Preserve Selected Publisher)
// @namespace    https://www.fab.com/
// @version      2.7.0
// @description  Claim free Fab products from cards. Keep selected publishers and search filters, apply Free only, hide owned, preserve progress.
// @match        https://www.fab.com/*
// @match        https://fab.com/*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(() => {
  'use strict';
  if (window.__fabFreeCardsV27) return;
  window.__fabFreeCardsV27 = true;

  // Uses Fab UI and free results only. No undocumented claim APIs, purchases, or downloads.
  const STATE_KEY = 'fab-quixel-card-adder-v2-state'; // Reuse existing v2 progress.
  const LOCK_KEY = 'fab-quixel-card-adder-v2-lock';
  const TAB_KEY = 'fab-quixel-card-adder-v2-tab';
  const CHECKPOINT_KEY = 'fab-quixel-card-adder-v2-checkpoint';
  const INFLIGHT_KEY = 'fab-quixel-card-adder-v2-inflight';
  const PREFERENCES_KEY = 'fab-free-card-v23-hide-owned-confirmed'; // Preserve previous confirmation key.
  const FILTER_NAV_KEY = 'fab-free-card-v26-filter-nav';
  const FILTER_MIN_SETTLE_MS = 10000; // Wait a full ten seconds AFTER the Free URL is visible.
  const FILTER_MAX_SETTLE_MS = 45000; // Give slow Fab searches enough time to populate.
  const FILTER_URL_TIMEOUT_MS = 30000; // No retry navigation while React/Fab changes the URL.
  const FILTER_GRID_QUIET_MS = 1800; // Require product-grid stability after initial loading.
  const AFTER_PREFS_SETTLE_MS = 1400; // Allow the result grid to update after Save.
  const LOCK_TTL_MS = 30000;
  const instanceId = crypto.randomUUID?.() || `document-${Date.now()}-${Math.random()}`;
  const onSearch = () => /^\/(?:search|category)(?:\/|$)/i.test(location.pathname);
  const pageKey = () => location.pathname + location.search;
  const LISTING_ID = /\/listings\/([0-9a-f]{8}-[0-9a-f-]{27,})/i;
  const FAST_CLICK_PAUSE_MS = 1100;  // Default: ~2-3x faster, but still sequential.
  const NORMAL_CLICK_PAUSE_MS = 2800;
  const VERIFY_TIMEOUT_MS = 12000;
  const SCROLL_PAUSE_MS = 650;
  const VERIFY_POLL_MS = 260;
  const HOVER_PAUSE_MS = 120;
  const LOOP_INTERVAL_MS = 300;
  const MAX_SCROLLS = 1800;
  const MAX_PROCESSED = 8000;
  const FINISH_AFTER_IDLE = 10;
  const tabId = sessionStorage.getItem(TAB_KEY) ||
    (crypto.randomUUID?.() || `${Date.now()}-${Math.random()}`);
  sessionStorage.setItem(TAB_KEY, tabId);

  // Fab sometimes changes the query in stages, or performs a full navigation.
  // A tab-scoped marker survives either behavior. IMPORTANT: wait for URL arrival
  // before starting the full settle timer, and never re-apply during that wait.
  let filterSettling = false;
  let filterArrivalAt = 0;
  let filterDeadline = 0;
  let filterGridSignature = '';
  let filterGridLastChange = 0;
  let filterNav = null;
  let prefsSettleUntil = 0;
  try {
    filterNav = JSON.parse(sessionStorage.getItem(FILTER_NAV_KEY) || 'null');
    if (filterNav && (Date.now() - (filterNav.time || 0) < 90000) &&
        /^\/(?:search|category)(?:\/|$)/i.test(location.pathname)) {
      filterSettling = true;
    } else if (filterNav) {
      sessionStorage.removeItem(FILTER_NAV_KEY);
      filterNav = null;
    }
  } catch { sessionStorage.removeItem(FILTER_NAV_KEY); }


  const pause = ms => new Promise(resolve => setTimeout(resolve, ms));
  const text = el => (el?.innerText || el?.textContent || '').replace(/\s+/g, ' ').trim();
  const isVisible = el => !!(el && el.isConnected && el.getClientRects().length &&
    getComputedStyle(el).visibility !== 'hidden');
  const idOf = url => (String(url || '').match(LISTING_ID) || [])[1]?.toLowerCase() || '';
  const blank = () => ({ run: false, ownerTab: '', target: 5, done: {}, speed: 'fast', attempts: 0,
    clicks: 0, scans: 0, idle: 0, message: 'Open any Fab page and select Test 5 or Run all; Free and hide-owned are configured automatically.' });
  const load = () => { try { return {...blank(), ...JSON.parse(localStorage.getItem(STATE_KEY) || '{}')}; }
    catch { return blank(); } };
  const store = s => { localStorage.setItem(STATE_KEY, JSON.stringify(s)); render(s); };
  const setStatus = message => { const s = load(); if (s.message !== message) { s.message = message; store(s); } };
  const claimDelay = () => load().speed === 'normal' ? NORMAL_CLICK_PAUSE_MS : FAST_CLICK_PAUSE_MS;
  let prefButtonAttempt = 0;
  let prefButtonAttempts = 0;
  let prefChecking = false;
  let prefSaveAttempt = 0;
  let prefSavedPending = false;
  const setPrefDone = () => localStorage.setItem(PREFERENCES_KEY, 'true');
  const prefDone = () => localStorage.getItem(PREFERENCES_KEY) === 'true';
  const visibleNotices = () => [...document.querySelectorAll('[role="status"], [role="alert"], [data-sonner-toast], [data-state="open"]')]
    .filter(isVisible).map(text).join(' | ');
  function throttlingNotice() {
    const notices = visibleNotices();
    return /too many requests|rate limit|try again later|429|slow down|temporarily blocked/i.test(notices) ? notices.slice(0, 180) : '';
  }
  const lock = () => {
    try { return JSON.parse(localStorage.getItem(LOCK_KEY) || '{}'); } catch { return {}; }
  };
  const releaseOwnLock = () => {
    const l = lock();
    if (l.tab === tabId && l.instance === instanceId) localStorage.removeItem(LOCK_KEY);
  };
  // New tabs show the panel but NEVER start claiming by themselves. One runner at a time.
  const ownLock = () => {
    const l = lock();
    if (l.tab && (l.tab !== tabId || (l.instance && l.instance !== instanceId)) &&
        Date.now() - l.time < LOCK_TTL_MS) return false;
    // Avoid writing localStorage every 300 ms; refresh the lease periodically.
    if (l.tab === tabId && l.instance === instanceId && Date.now() - l.time < 8000) return true;
    localStorage.setItem(LOCK_KEY, JSON.stringify({tab: tabId, instance: instanceId, time: Date.now()}));
    return true;
  };
  const isRunner = () => {
    const s = load();
    return s.run && s.ownerTab === tabId && onSearch();
  };
  const stop = message => {
    const s = load();
    // A stale asynchronous operation must not stop a different tab's newer run.
    if (s.run && s.ownerTab && s.ownerTab !== tabId) return;
    s.run = false; s.ownerTab = ''; s.message = message; store(s);
    releaseOwnLock();
  };
  const globalPause = () => {
    const s = load(); s.run = false; s.ownerTab = '';
    s.message = 'Paused from a Fab tab. A claim already sent may still finish.';
    store(s);
    releaseOwnLock();
  };

  function saveCheckpoint(id = '') {
    if (!onSearch()) return;
    try {
      sessionStorage.setItem(CHECKPOINT_KEY, JSON.stringify({
        url: pageKey(), y: Math.round(window.scrollY), id, time: Date.now()
      }));
    } catch (e) { console.warn('[Fab] Could not save scroll checkpoint:', e); }
  }
  const checkpoint = (() => {
    try { return JSON.parse(sessionStorage.getItem(CHECKPOINT_KEY) || '{}'); }
    catch { return {}; }
  })();
  // If Fab itself reloads, remember where the script was before the reload.
  // Scrolling is still needed to repopulate an infinite-scroll grid after a hard reload.
  let recoveryY = (load().run && load().ownerTab === tabId && onSearch() &&
    checkpoint.url === pageKey() && Date.now() - checkpoint.time < 2 * 60 * 60 * 1000)
    ? Math.max(0, Number(checkpoint.y) || 0) : 0;
  let recoveryStalls = 0;

  // If the page reloads halfway through a claim, NEVER blindly click it twice.
  // Mark the uncertain action for review; the user can verify it in My Library.
  try {
    const inflight = JSON.parse(sessionStorage.getItem(INFLIGHT_KEY) || 'null');
    if (inflight?.id && !load().done[inflight.id]) {
      const s = load();
      s.done[inflight.id] = {status:'needs-review', title:inflight.title, time:Date.now()};
      s.message = `Page reloaded during a click on "${inflight.title}". Check My Library before retrying it.`;
      localStorage.setItem(STATE_KEY, JSON.stringify(s));
    }
  } catch (e) { console.warn('[Fab] Could not restore in-flight claim:', e); }
  sessionStorage.removeItem(INFLIGHT_KEY);

  window.addEventListener('pagehide', () => {
    if (isRunner()) saveCheckpoint();
    releaseOwnLock();
  });

  function modalOrChallenge() {
    if (/just a moment|security check|one more step/i.test(document.title)) return 'Epic/Fab security challenge detected';
    if (document.querySelector('iframe[src*="challenge"], #challenge-running, #cf-challenge-running'))
      return 'Security verification detected';
    const dialogs = [...document.querySelectorAll('[role="dialog"], dialog, [aria-modal="true"]')]
      .filter(isVisible);
    for (const el of dialogs) {
      const t = text(el).slice(0, 700);
      if (/sign in|log in|verify|captcha|license|agree|accept|checkout|cart|purchase/i.test(t))
        return 'Fab requires manual sign-in, license acceptance, or confirmation';
    }
    return '';
  }

  // Identify one single listing card, not a surrounding grid with several listings.
  function priceIsFree(card) {
    const t = text(card);
    // Never claim cards carrying a visible nonzero price, even if Free appears in a title.
    if (/(?:from\s*)?(?:US\$|\$|€|£)\s*\d[\d,.]*/i.test(t)) return false;
    return /\b(?:from\s+)?free\b/i.test(t);
  }

  function locateCard(a, id) {
    let p = a, candidate = null;
    for (let depth = 0; depth < 9 && p; depth++, p = p.parentElement) {
      if (p === document.body) break;
      const t = text(p);
      if (t.length > 1200) break;
      const ids = new Set([...p.querySelectorAll('a[href*="/listings/"]')]
        .map(el => idOf(el.href)).filter(Boolean));
      if (ids.size > 1) break;
      if (ids.size === 1 && ids.has(id) &&
          priceIsFree(p)) candidate = p;
    }
    return candidate;
  }

  const cardCache = new WeakMap();
  function cards() {
    const unique = new Map();
    for (const a of document.querySelectorAll('a[href*="/listings/"]')) {
      const id = idOf(a.href);
      if (!id || unique.has(id)) continue;
      let card = cardCache.get(a);
      if (!card || !card.isConnected) {
        card = locateCard(a, id);
        if (card) cardCache.set(a, card);
      }
      if (!card) continue;
      unique.set(id, {id, card, title: text(a).slice(0, 90) || id});
    }
    return [...unique.values()];
  }

  function isOwned(card) {
    return /\b(saved in my library|added to my library|already in my library|view in my library)\b/i.test(text(card));
  }

  function isAddName(s) {
    return /\badd\s+to\s+(?:my\s+)?library\b/i.test(s) || /\bsave\s+to\s+(?:my\s+)?library\b/i.test(s);
  }
  function forbiddenName(s) {
    return /buy|cart|checkout|wish\s?list|download|export|share|report|preview/i.test(s);
  }
  function controlName(el) {
    return [el.getAttribute('aria-label'), el.getAttribute('title'),
      el.getAttribute('data-tooltip'), el.getAttribute('data-testid'),
      el.getAttribute('data-state') === 'open' ? '' : null,
      text(el).slice(0, 90)].filter(Boolean).join(' ').trim();
  }

  // Card icon appears at the top right of the image in the supplied screenshot.
  // Prefer named accessible controls; only use a compact upper-right control as fallback.
  function findAddButton(card) {
    const controls = [...card.querySelectorAll('button, [role="button"]')]
      .filter(el => !el.matches('a, a *') && el.getAttribute('aria-disabled') !== 'true' &&
        !el.disabled && el.isConnected);
    const named = controls.find(el => {
      const name = controlName(el);
      return isAddName(name) && !forbiddenName(name) && !/saved in my library|remove/i.test(name);
    });
    if (named) return {button: named, method: 'labeled icon'};

    const cr = card.getBoundingClientRect();
    const possible = controls.filter(el => {
      const r = el.getBoundingClientRect();
      const name = controlName(el);
      if (forbiddenName(name) || /remove|saved in my library/i.test(name)) return false;
      if (r.width < 12 || r.height < 12 || r.width > 75 || r.height > 75) return false;
      // Icon must sit near the card's upper right (not its bottom buttons).
      if (r.right < cr.right - 75 || r.left > cr.right + 10) return false;
      if (r.top < cr.top - 12 || r.top > cr.top + Math.min(105, cr.height * .45)) return false;
      return !!el.querySelector('svg') || !!el.querySelector('img') || name === '';
    });
    // Never guess if there are multiple different controls in the icon position.
    if (possible.length === 1) return {button: possible[0], method: 'top-right card icon'};
    return null;
  }

  function showMarker(card) {
    card.style.outline = '3px solid #38bdf8';
    card.style.outlineOffset = '-3px';
    setTimeout(() => { if (card.isConnected) { card.style.outline = ''; card.style.outlineOffset = ''; } }, 1800);
  }
  function cardHover(card) {
    card.dispatchEvent(new MouseEvent('mouseover', {bubbles: true, view: window}));
    card.dispatchEvent(new PointerEvent('pointerover', {bubbles: true, pointerType: 'mouse'}));
  }
  const mark = (id, status, title) => {
    const s = load(); s.done[id] = {status, title, time: Date.now()}; store(s);
  };
  const count = (s, status) => Object.values(s.done).filter(x => x.status === status).length;
  // Batch target counts actual new additions, not cards that were already owned.
  const completed = s => count(s, 'added');

  async function claimCard(item) {
    const {id, card, title} = item;
    if (load().done[id]) return;
    card.scrollIntoView({block: 'center', behavior: 'instant'});
    cardHover(card);
    await pause(HOVER_PAUSE_MS);
    if (!isRunner()) return;
    if (isOwned(card)) { mark(id, 'owned', title); return; }
    if (!priceIsFree(card)) return stop('STOP: Card is not clearly free. No action was taken.');
    const match = findAddButton(card);
    if (!match) return stop(`STOP: No unambiguous Add to My Library icon on "${title}". Share a screenshot or element details; product page will NOT open.`);
    const {button, method} = match;
    // Strictly click the button element, never the card or its link.
    if (!button.matches('button, [role="button"]')) return stop('STOP: card icon was not a button');
    showMarker(card);
    setStatus(`Clicking ${method} for "${title}" …`);
    const baseline = document.body.innerText.slice(-1200);
    // Save the pending action first. Do not navigate to a listing or reload the page.
    sessionStorage.setItem(INFLIGHT_KEY, JSON.stringify({id, title, time:Date.now()}));
    try { button.click(); } catch (e) {
      sessionStorage.removeItem(INFLIGHT_KEY);
      return stop(`Click failed: ${e.message}`);
    }
    const s1 = load(); s1.clicks++; store(s1);
    const start = Date.now();
    while (Date.now() - start < VERIFY_TIMEOUT_MS) {
      await pause(VERIFY_POLL_MS);
      if (!isRunner()) return;
      const problem = modalOrChallenge();
      if (problem) return stop(`PAUSED: ${problem}. Complete it manually, then resume.`);
      const throttled = throttlingNotice();
      if (throttled) return stop(`PAUSED: Fab requested slower activity (${throttled}). Wait before resuming; switch to Normal speed.`);
      if (isOwned(card)) { mark(id, 'added', title); sessionStorage.removeItem(INFLIGHT_KEY); saveCheckpoint(id); setStatus(`Added: ${title}`); await pause(claimDelay()); return; }
      // Fab may use a global success toast without immediately updating the card.
      const notices = visibleNotices();
      if (/successfully added|added to (?:my |your )?library|product added to/i.test(notices) &&
          !/successfully added|added to (?:my |your )?library|product added to/i.test(baseline)) {
        mark(id, 'added', title); sessionStorage.removeItem(INFLIGHT_KEY); saveCheckpoint(id); setStatus(`Fab confirmed addition: ${title}`);
        await pause(claimDelay()); return;
      }
    }
    mark(id, 'needs-review', title);
    sessionStorage.removeItem(INFLIGHT_KEY);
    return stop(`PAUSED: clicked "${title}" but Fab did not confirm ownership. Check it manually. This item will be skipped until Reset to prevent duplicate clicks.`);
  }

  const host = document.createElement('div');
  host.style.cssText = 'position:fixed;bottom:18px;right:18px;z-index:2147483647';
  const root = host.attachShadow({mode: 'open'});
  root.innerHTML = `<style>
    .p{width:325px;background:#121923;color:#f1f5f9;padding:13px;border-radius:12px;
       border:1px solid #435064;box-shadow:0 12px 35px #0009;font:12px/1.5 system-ui,Arial}
    h3{font-size:14px;margin:0 0 5px} .status{max-height:90px;overflow:auto;overflow-wrap:anywhere;margin:7px 0;color:#cbd5e1}
    .stats{color:#86efac;margin-bottom:8px} .buttons{display:flex;flex-wrap:wrap;gap:5px}
    button{cursor:pointer;background:#27374e;color:white;border:1px solid #56677f;
      border-radius:6px;font:600 12px system-ui;padding:7px 9px}button:hover{background:#395375}
    .help{font-size:11px;color:#e9c68c;margin-top:9px}
  </style><div class="p">
    <h3>Fab · Free Card Auto-Add v2.7</h3>
    <div class="stats" id="stats">Ready</div><div class="status" id="status">Starting…</div>
    <div class="buttons">
      <button id="test">Test 5</button><button id="all">Run all</button>
      <button id="pause">Pause</button><button id="speed">Speed: Fast</button><button id="prefs">Recheck Library filter</button><button id="inspect">Inspect next</button>
      <button id="reset">Reset progress</button>
    </div>
    <div class="help">Run all preserves your selected publishers and other search filters. Applies Free if needed, waits for results, then hides owned. One runner; no purchases/downloads.</div>
  </div>`;
  document.body.appendChild(host);

  function render(s = load()) {
    const stats = root.getElementById('stats');
    if (!stats) return;
    const runState = !s.run ? 'STOPPED' : !s.ownerTab ? 'PRESS RUN ALL TO RESUME' :
      s.ownerTab === tabId ? 'RUNNING HERE' : 'RUNNING IN ANOTHER FAB TAB';
    stats.textContent = `Added ${count(s, 'added')} | Already saved ${count(s, 'owned')} | ` +
      `To review ${count(s, 'needs-review')} | Scrolls ${s.scans} | Clicks ${s.clicks} | ${runState}`;
    const where = !onSearch() ? 'Run all will open Fab free search from this page. ' : '';
    const background = document.hidden && s.run && s.ownerTab === tabId ? ' (Background tab: Chrome may slow timers)' : '';
    root.getElementById('status').textContent = where + s.message + background;
    root.getElementById('speed').textContent = `Speed: ${s.speed === 'normal' ? 'Normal' : 'Fast'}`;
  }

  // Apply Free only. NEVER remove publisher/seller selections or any other
  // existing URL search filters (keywords, product categories, sort, etc.).
  // On a non-search page there is no search state to inherit; go to /search.
  function targetFreeUrl() {
    const u = onSearch() ? new URL(location.href) : new URL('/search', location.origin);
    u.searchParams.set('is_free', '1');
    u.searchParams.set('ui_filter_price', '1');
    return u;
  }
  function readyFree() {
    const p = new URLSearchParams(location.search);
    // Selected publishers do not affect whether the Free filter is active.
    // ui_filter_price is an optional UI marker during Fab hydration.
    return onSearch() && ['1','true'].includes((p.get('is_free') || '').toLowerCase());
  }


  // The one and only permitted setup navigation: change to the Free results.
  // After that we WAIT, even if Fab temporarily shows an unfiltered route.
  // No reloads, retries or additional redirects while filtering/claiming.
  function waitForFreeResults() {
    if (!filterSettling) return false;
    const now = Date.now();
    // During a client-side URL transition, the old page can remain alive.
    // Do not interpret that moment as a missing Free filter or navigate again.
    if (!readyFree()) {
      const elapsed = now - (filterNav?.time || now);
      if (elapsed < FILTER_URL_TIMEOUT_MS) {
        const left = Math.ceil((FILTER_URL_TIMEOUT_MS - elapsed) / 1000);
        setStatus(`Applying Free… waiting for Fab to finish updating the URL (${left}s max). No retry/reload.`);
        return true;
      }
      stop('PAUSED: Fab did not finish applying the Free URL within 30s. Check the Free chip and press Run all again. No automatic retry was made.');
      return true;
    }
    // Start MINIMUM wait now, AFTER the new Free URL is actually present.
    // This handles SPA transitions and real page navigations consistently.
    if (!filterArrivalAt) {
      filterArrivalAt = now;
      filterDeadline = now + FILTER_MAX_SETTLE_MS;
      filterGridLastChange = now;
      setStatus('Free filter confirmed in URL. Waiting 10 seconds for Fab to load its products…');
      return true;
    }
    const remaining = FILTER_MIN_SETTLE_MS - (now - filterArrivalAt);
    if (remaining > 0) {
      setStatus(`Free filter active. Waiting ${Math.ceil(remaining / 1000)}s for Fab results (no claims yet)…`);
      return true;
    }
    const listingLinks = [...document.querySelectorAll('a[href*="/listings/"]')];
    const mainText = text(document.querySelector('main') || document.body).slice(0, 6000);
    const noResults = /no (?:matching )?(?:products|results)|0 results|nothing found/i.test(mainText);
    if (!listingLinks.length && !noResults) {
      if (now < filterDeadline) {
        setStatus('Free filter active. Fab results are still loading; waiting (maximum 45s)…');
        return true;
      }
      stop('PAUSED: Fab search results did not load after 45s. Check the page, then press Run all.');
      return true;
    }
    // Fab can show the first cards, then replace the grid with more cards.
    // Require 1.8 seconds without changing first/last cards or count.
    const signature = `${listingLinks.length}|${listingLinks[0]?.getAttribute('href') || ''}|${listingLinks.at(-1)?.getAttribute('href') || ''}`;
    if (signature !== filterGridSignature) {
      filterGridSignature = signature;
      filterGridLastChange = now;
    }
    if (!noResults && now - filterGridLastChange < FILTER_GRID_QUIET_MS && now < filterDeadline) {
      setStatus('Free product grid detected. Allowing it to settle before opening preferences…');
      return true;
    }
    filterSettling = false;
    filterNav = null;
    sessionStorage.removeItem(FILTER_NAV_KEY);
    setStatus('Free filter and results ready. Opening User preferences…');
    return false;
  }

  function ensureFree() {
    if (filterSettling) return false; // Critical: do not start another URL change.
    if (readyFree()) return true;
    const u = targetFreeUrl();
    const s = load();
    // One setup navigation per Run all, not two speculative retries while
    // the router is still transitioning. A subsequent run resets this count.
    if ((s.setupNavCount || 0) >= 1) {
      stop('PAUSED: Fab is not showing the Free filter after the first attempt. Check the page and press Run all again.');
      return false;
    }
    s.setupNavCount = 1;
    s.message = 'Applying Free filter without changing your selected publisher or other filters; waiting for Fab navigation.';
    store(s); saveCheckpoint();
    // Arm the gate on THIS document before navigation, and also persist it for
    // any new document created by location.assign().
    filterNav = {url:u.href, time:Date.now()};
    sessionStorage.setItem(FILTER_NAV_KEY, JSON.stringify(filterNav));
    filterSettling = true;
    filterArrivalAt = 0;
    filterGridSignature = '';
    filterGridLastChange = 0;
    if (u.href !== location.href) location.assign(u.href);
    return false;
  }


  // All Content-preferences actions use visible controls in Fab's own interface.
  // Fab has changed its Sort button markup; don't rely on a particular class or tooltip.
  function preferenceDialog() {
    const known = [...document.querySelectorAll('[role="dialog"],dialog,[aria-modal="true"]')]
      .filter(isVisible).find(el => /content preferences/i.test(text(el)));
    if (known) return known;
    const heading = [...document.querySelectorAll('h1,h2,h3,[role="heading"]')]
      .filter(isVisible).find(el => /^content preferences$/i.test(text(el)));
    if (!heading) return null;
    let ancestor = heading.parentElement;
    for (let i = 0; i < 12 && ancestor && ancestor !== document.body; i++, ancestor = ancestor.parentElement) {
      const t = text(ancestor);
      if (/show products in my library/i.test(t) && /mature content/i.test(t) &&
          [...ancestor.querySelectorAll('button,[role="button"]')].some(b => /^save$/i.test(text(b))))
        return ancestor;
    }
    return null;
  }

  function prefRect(el) {
    const r = el.getBoundingClientRect();
    return {rect:r, midY:(r.top + r.bottom)/2};
  }
  function preferenceButton() {
    const clickable = [...document.querySelectorAll('button,[role="button"],a[aria-label],[tabindex="0"]')]
      .filter(el => isVisible(el) && !host.contains(el) &&
        !el.closest('[role="dialog"],dialog,[aria-modal="true"]'));
    // On the user's actual Fab DOM this control is aria-label="User preferences".
    // It is a button bearing Fab's horizontal-sliders icon, not a Sort button.
    const directlyNamed = clickable.filter(el => /^(?:user preferences|content preferences|content settings|filter preferences)$/i.test(
      (el.getAttribute('aria-label') || el.getAttribute('title') || '').trim()));
    if (directlyNamed.length === 1) return directlyNamed[0];
    if (directlyNamed.length > 1) return null; // Never guess between duplicates.

    // Some Fab versions omit the accessible name but expose the slider icon.
    const sliderIconButtons = clickable.filter(el => /sliders[-_ ]?horizontal|icon[-_ ]?sliders/i.test(
      el.querySelector('svg')?.outerHTML || '') &&
      !el.closest('a[href*="/listings/"],[data-testid*="product-card"]'));
    if (sliderIconButtons.length === 1) return sliderIconButtons[0];
    if (sliderIconButtons.length > 1) return null;

    // Other Fab layouts may expose an older Content preferences label.
    const explicitlyLabeled = clickable.filter(el => /user preferences|content preferences|content settings|filter preferences/i.test([
      el.getAttribute('aria-label'),el.getAttribute('title'),el.getAttribute('data-testid'),
      el.getAttribute('data-tooltip'),el.getAttribute('data-tooltip-content')
    ].filter(Boolean).join(' ')));
    if (explicitlyLabeled.length === 1) return explicitlyLabeled[0];

    // Find the Sort by control, even when Fab places the wording inside a nested <span>.
    const sortTextNodes = [...document.querySelectorAll('button,[role="button"],span,div')]
      .filter(el => isVisible(el) && text(el).length < 90 && /^sort\s*by\s*:?/i.test(text(el)));
    const sortParents = new Set();
    for (const el of sortTextNodes) {
      let a = el.closest('button,[role="button"]') || el;
      for (let depth = 0; depth < 3 && a && a !== document.body; depth++, a=a.parentElement) {
        const r = a.getBoundingClientRect();
        if (r.width >= 75 && r.width <= 330 && r.height > 18 && r.height < 75 &&
            r.top < Math.min(innerHeight * 0.5, 310)) sortParents.add(a);
      }
    }
    const sort = [...sortParents].sort((a,b) => b.getBoundingClientRect().right - a.getBoundingClientRect().right)[0];
    const eligible = clickable.filter(el => {
      const r = el.getBoundingClientRect();
      if (r.width < 18 || r.width > 80 || r.height < 18 || r.height > 80) return false;
      if (r.top < 55 || r.top > Math.min(innerHeight * .46, 320)) return false;
      if (!el.querySelector('svg') && !el.querySelector('img')) return false;
      if (el.closest('a[href*="/listings/"]') || el.closest('[data-testid*="product-card"]')) return false;
      const label = [el.getAttribute('aria-label'), el.getAttribute('title'), text(el)].filter(Boolean).join(' ');
      if (/wish|share|bookmark|download|save to library|add to library|search|cart|clear|sort/i.test(label)) return false;
      return true;
    });
    if (sort) {
      const sr = prefRect(sort);
      const adjacent = eligible.map(el => {
        const {rect:r, midY} = prefRect(el);
        return {el, dx:r.left - sr.rect.right, dy:Math.abs(midY - sr.midY)};
      }).filter(x => x.dx >= -9 && x.dx < 125 && x.dy < 30)
        .sort((a,b) => a.dx + a.dy*2 - b.dx - b.dy*2);
      if (adjacent.length === 1 || (adjacent.length > 1 &&
          adjacent[1].dx + adjacent[1].dy*2 > adjacent[0].dx + adjacent[0].dy*2 + 28))
        return adjacent[0].el;
    }

    // Last resort: the sliders button is the icon-only control on the RIGHT of the
    // search toolbar, below the global Fab header. Never guess among multiple icons.
    const topRight = eligible.filter(el => {
      const r = el.getBoundingClientRect();
      return r.left > innerWidth * .78 && r.right < innerWidth + 2;
    }).sort((a,b) => b.getBoundingClientRect().right - a.getBoundingClientRect().right);
    if (topRight.length === 1) return topRight[0];
    if (topRight.length > 1 && topRight[0].getBoundingClientRect().right -
        topRight[1].getBoundingClientRect().right > 75) return topRight[0];
    return null;
  }
  function switchValue(sw) {
    if (sw.matches('input[type="checkbox"]')) return !!sw.checked;
    const a = sw.getAttribute('aria-checked');
    if (a === 'true' || a === 'false') return a === 'true';
    const state = sw.getAttribute('data-state');
    if (state === 'checked' || state === 'unchecked') return state === 'checked';
    return null;
  }
  function librarySwitch(dialog) {
    const switches = [...dialog.querySelectorAll('[role="switch"],input[type="checkbox"],button[aria-checked]')]
      .filter(sw => isVisible(sw) && switchValue(sw) !== null);
    const labels = [...dialog.querySelectorAll('label,span,p,div')]
      .filter(el => isVisible(el) && /^show products in my library$/i.test(text(el)));
    for (const label of labels) {
      for (let scope = label, depth = 0; depth < 6 && scope && scope !== dialog.parentElement;
           scope = scope.parentElement, depth++) {
        const found = switches.filter(sw => scope.contains(sw));
        if (found.length === 1) return found[0];
      }
    }
    // The Fab switch is displayed immediately to the LEFT of its text. This also
    // handles the Radix markup where the switch and label are siblings.
    const options = [];
    for (const label of labels) {
      const lr = label.getBoundingClientRect();
      const ly = (lr.top + lr.bottom)/2;
      for (const sw of switches) {
        const r = sw.getBoundingClientRect();
        const dy = Math.abs((r.top + r.bottom)/2 - ly);
        const dx = lr.left - r.right;
        if (dy < 30 && dx >= -10 && dx < 145) options.push({sw, score:dy*3 + Math.max(0,dx)});
      }
    }
    options.sort((a,b) => a.score - b.score);
    if (options.length === 1 || (options.length > 1 && options[1].score > options[0].score + 20))
      return options[0].sw;
    return null;
  }
  function savePreferences(dialog) {
    const save = [...dialog.querySelectorAll('button,[role="button"]')]
      .find(b => isVisible(b) && /^save$/i.test(text(b)) && !b.disabled);
    if (!save) return false;
    save.click();
    prefSavedPending = true;
    prefSaveAttempt = Date.now();
    return true;
  }
  async function ensureHiddenOwned() {
    if (prefDone()) return true;
    if (prefChecking) return false;
    prefChecking = true;
    try {
      let dialog = preferenceDialog();
      if (prefSavedPending) {
        if (!dialog) {
          prefSavedPending = false;
          setPrefDone();
          prefsSettleUntil = Date.now() + AFTER_PREFS_SETTLE_MS;
          setStatus('Fab saved Content preferences. Waiting briefly for the product grid to update…');
          return true;
        }
        if (Date.now() - prefSaveAttempt < 2600) return false;
        prefSavedPending = false;
        stop('PAUSED: Content preferences did not close after Save. Check Fab for an error, then use Recheck Library filter.');
        return false;
      }
      if (!dialog) {
        // Open automatically, instead of asking the user to click the icon.
        if (Date.now() - prefButtonAttempt < 1800) return false;
        const button = preferenceButton();
        if (!button) {
          setStatus('Could not identify Fab’s Content preferences icon automatically. Click Recheck Library filter or share the button HTML.');
          return false;
        }
        if (prefButtonAttempts >= 4) {
          setStatus('Content preferences did not open after four tries. Please check Fab’s interface; no more automatic clicks.');
          return false;
        }
        prefButtonAttempt = Date.now();
        prefButtonAttempts++;
        button.click();
        await pause(350);
        dialog = preferenceDialog();
        if (!dialog) { setStatus('Opening Content preferences automatically…'); return false; }
      }
      const sw = librarySwitch(dialog);
      if (!sw) {
        setStatus('Content preferences opened, but I could not safely identify the My Library switch. No other switch was changed.');
        return false;
      }
      if (switchValue(sw) === true) {
        sw.click();
        await pause(220);
        if (switchValue(sw) !== false) {
          setStatus('The My Library toggle did not confirm OFF. No further automatic toggle attempts.');
          return false;
        }
      }
      if (!savePreferences(dialog)) {
        setStatus('My Library is OFF, but the Save control was not found. Save manually and click Recheck Library filter.');
        return false;
      }
      setStatus('My Library is OFF. Saving Content preferences…');
      return false;
    } finally { prefChecking = false; }
  }

  function begin(max) {
    const s = load();
    if (s.run && s.ownerTab && s.ownerTab !== tabId)
      return alert('Another Fab tab is already processing. Pause it using any Fab control panel first.');
    if (!ownLock()) return alert('Another Fab tab currently owns the run. Pause it or retry after its lock expires.');
    // A five-item test adds five more items even when earlier tests were completed.
    s.ownerTab = tabId; s.target = max === 5 ? completed(s) + 5 : MAX_PROCESSED;
    s.run = true; s.idle = 0; s.setupNavCount = 0;
    // A fresh Run all can start from an already-filtered search; in that case
    // no navigation is necessary. Discard an old incomplete filter gate.
    if (readyFree()) {
      filterSettling = false; filterNav = null; filterArrivalAt = 0;
      sessionStorage.removeItem(FILTER_NAV_KEY);
    }
    // Verify the preference at the start of each new run, even if it was saved previously.
    localStorage.removeItem(PREFERENCES_KEY); prefButtonAttempts = 0; prefSavedPending = false;
    s.message = 'Preparing: preserving your selected publisher and other search filters, applying Free, hiding owned.';
    store(s);
  }
  root.getElementById('test').onclick = () => begin(5);
  root.getElementById('all').onclick = () => begin(MAX_PROCESSED);
  root.getElementById('pause').onclick = globalPause;
  root.getElementById('prefs').onclick = async () => {
    localStorage.removeItem(PREFERENCES_KEY);
    prefButtonAttempt = 0; prefButtonAttempts = 0; prefSavedPending = false;
    setStatus('Opening Content preferences and checking My Library automatically…');
    if (onSearch()) await ensureHiddenOwned();
  };
  root.getElementById('speed').onclick = () => {
    const s = load(); s.speed = s.speed === 'normal' ? 'fast' : 'normal';
    s.message = `Speed set to ${s.speed}. Changes apply after the current click.`; store(s);
  };
  root.getElementById('reset').onclick = () => {
    if (!confirm('Reset v2 progress? This does not remove anything from Fab My Library.')) return;
    stop('Resetting'); localStorage.removeItem(STATE_KEY); render(blank());
  };
  root.getElementById('inspect').onclick = () => {
    if (!onSearch()) return alert('Inspect next works on Fab search and category results.');
    const s = load(); const next = cards().find(c => !s.done[c.id]);
    if (!next) return setStatus('No unprocessed free cards currently loaded. Scroll farther down.');
    next.card.scrollIntoView({block: 'center'}); cardHover(next.card); showMarker(next.card);
    const info = findAddButton(next.card);
    console.log('[Fab card v2] NEXT CARD', next, 'MATCH', info,
      'BUTTON HTML', info?.button?.outerHTML?.slice(0, 1500));
    setStatus(info ? `Icon found (${info.method}) for ${next.title}. See Console for details.` :
      `No clearly identifiable top-right Add button for ${next.title}. See Console and share a screenshot.`);
  };

  let busy = false;
  let lastPassiveRender = 0;
  async function work() {
    if (busy) return;
    const s = load();
    if (!s.run || s.ownerTab !== tabId) {
      if (Date.now() - lastPassiveRender > 2500) { render(s); lastPassiveRender = Date.now(); }
      return;
    }
    if (!ownLock()) { render({...s, message:'Another tab/document holds the active lock. This tab will not click.'}); return; }
    // Always check the pending Free gate FIRST; do not mistakenly retry the
    // navigation on an intermediate URL before React finishes applying filters.
    if (waitForFreeResults()) return;
    if (!ensureFree()) return;
    if (!(await ensureHiddenOwned())) return;
    if (Date.now() < prefsSettleUntil) return;
    if (completed(s) >= s.target) return stop(`Batch complete: ${completed(s)} new items added. Verify My Library.`);
    const issue = modalOrChallenge();
    if (issue) return stop(`PAUSED: ${issue}; resolve manually.`);
    const throttled = throttlingNotice();
    if (throttled) return stop(`PAUSED: Fab requested slower activity (${throttled}). Wait before resuming at Normal speed.`);
    busy = true;
    try {
      // Recover scroll position after an unexpected site reload. Only the original
      // tab does this. Other Fab tabs remain idle and show the control panel.
      if (recoveryY > 400 && window.scrollY + innerHeight * 0.5 < recoveryY) {
        const oldY = window.scrollY;
        window.scrollBy({top: Math.min(recoveryY - window.scrollY, Math.max(540, innerHeight * .92)), behavior:'instant'});
        await pause(SCROLL_PAUSE_MS);
        if (window.scrollY <= oldY + 4) recoveryStalls++;
        else recoveryStalls = 0;
        if (recoveryStalls >= 12) recoveryY = 0; // Site stopped loading older results.
        const st = load();
        st.message = recoveryY ? `Restoring earlier scroll checkpoint (~${Math.round(recoveryY)}px). No page reload requested.` :
          'Scroll restore finished; continuing through saved product IDs.';
        store(st);
        return;
      }
      recoveryY = 0;
      const available = cards();
      let changed = false;
      for (const item of available) {
        if (load().done[item.id]) continue;
        if (isOwned(item.card)) { mark(item.id, 'owned', item.title); changed = true; }
      }
      const pending = available.find(item => !load().done[item.id]);
      if (pending) {
        await claimCard(pending);
        const latest = load();
        if (isRunner()) { latest.idle = 0; store(latest); }
      } else {
        const st = load();
        if (!isRunner()) return;
        if (st.scans >= MAX_SCROLLS) return stop('Reached scroll safety limit.');
        const before = `${window.scrollY}|${document.documentElement.scrollHeight}|${available.length}`;
        window.scrollBy({top: Math.max(420, innerHeight * .82), behavior:'instant'});
        await pause(SCROLL_PAUSE_MS);
        saveCheckpoint();
        if (!isRunner()) return;
        const loadedCount = cards().length;
        const after = `${window.scrollY}|${document.documentElement.scrollHeight}|${loadedCount}`;
        const bottom = window.scrollY + innerHeight >= document.documentElement.scrollHeight - 130;
        st.scans++;
        st.idle = !changed && bottom && before === after ? st.idle + 1 : 0;
        st.message = `Scanning/scrolling: ${loadedCount} loaded cards, ${Object.keys(st.done).length} checked; idle ${st.idle}/${FINISH_AFTER_IDLE}`;
        store(st);
        if (st.idle >= FINISH_AFTER_IDLE) stop('Reached end of loaded search results. Check My Library and review totals.');
      }
    } catch (e) {
      console.error('[Fab Free Card Auto-Add v2.7]', e);
      stop(`Stopped on error: ${e.message || e}`);
    } finally { busy = false; }
  }
  render();
  // All tabs update their panel from shared storage, including tabs opened later.
  window.addEventListener('storage', e => {
    if ([STATE_KEY, LOCK_KEY].includes(e.key)) render();
  });
  document.addEventListener('visibilitychange', () => render());
  window.addEventListener('pageshow', () => render());
  // No document.hidden/hasFocus gate: keep working in inactive/minimized tabs
  // whenever Chrome schedules the page. Chrome may throttle or discard tabs.
  setInterval(work, LOOP_INTERVAL_MS);
})();