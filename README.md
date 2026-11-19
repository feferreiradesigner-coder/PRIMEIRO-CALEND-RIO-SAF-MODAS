<!DOCTYPE html>
<html lang="en">
<head><style>
@font-face {
  font-family: "Optimistic";
  font-style: normal;
  font-weight: 400 600;
  font-display: swap;
  src: url("/fonts/OptimisticAI_VF_Optimized.woff2") format("woff2");
}
@font-face {
  font-family: "Optimistic Mono";
  font-style: normal;
  font-weight: 400;
  font-display: swap;
  src: url("/fonts/OptimisticMono_W_TextRegular.woff2") format("woff2");
}
:where(html) {
  font-family: "Optimistic", system-ui, sans-serif;
}
:where(code, pre, kbd, samp) {
  font-family: "Optimistic Mono", ui-monospace, monospace;
}
</style><style data-metaai-safe-area>:root { scroll-padding-top: var(--safe-area-scroll-padding-top, 0px); }
[data-safe-area-inset-root]::before {
  background: inherit;
  content: "";
  height: var(--safe-area-inset-top, 0px);
  inset: 0 0 auto;
  pointer-events: none;
  position: fixed;
  z-index: 2147483647;
}
</style><script data-metaai-safe-area-runtime>// (c) Meta Platforms, Inc. and affiliates. Confidential and proprietary.

(() => {
  const params = new URLSearchParams(window.location.search);
  const rawInset = params.get('top_inset_px');
  const parsedInset = rawInset !== null && /^\d+$/.test(rawInset)
    ? Number(rawInset)
    : 0;
  const topInset = params.get('has_top_banner') === '1'
    ? Math.min(parsedInset, 256)
    : 0;
  document.documentElement.style.setProperty('--safe-area-inset-top', `${topInset}px`);
  document.documentElement.style.setProperty('--safe-area-scroll-padding-top', `${topInset}px`);

  function observeNavigation() {
    const root = document.getElementById('root');
    if (root === null) return;
    const observed = new Set();
    let scheduled = false;

    function syncScrollPadding() {
      const elements = new Set(root.querySelectorAll('[data-safe-area-sticky-top]'));
      for (const element of observed) {
        if (!elements.has(element)) {
          resizeObserver.unobserve(element);
          observed.delete(element);
        }
      }
      for (const element of elements) {
        if (!observed.has(element)) {
          resizeObserver.observe(element);
          observed.add(element);
        }
      }
      let padding = topInset;
      for (const element of elements) {
        const style = window.getComputedStyle(element);
        if (style.position !== 'sticky' && style.position !== 'fixed') continue;
        const top = Number.parseFloat(style.top);
        if (Number.isFinite(top)) {
          padding = Math.max(padding, top + element.getBoundingClientRect().height);
        }
      }
      document.documentElement.style.setProperty(
        '--safe-area-scroll-padding-top', `${Math.ceil(padding)}px`,
      );
    }

    function scheduleUpdate() {
      if (scheduled) return;
      scheduled = true;
      queueMicrotask(() => {
        scheduled = false;
        syncScrollPadding();
      });
    }

    const resizeObserver = new ResizeObserver(scheduleUpdate);
    const mutationObserver = new MutationObserver(scheduleUpdate);
    mutationObserver.observe(root, {
      attributeFilter: ['class', 'data-safe-area-sticky-top', 'hidden', 'style'],
      attributes: true,
      childList: true,
      subtree: true,
    });
    window.addEventListener('resize', scheduleUpdate);
    scheduleUpdate();
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', observeNavigation, {once: true});
  } else {
    observeNavigation();
  }
})();
</script><script data-metaai-records-guard>// (c) Meta Platforms, Inc. and affiliates. Confidential and proprietary.

// When the records API rejects a write because the app is at its record
// limit, this page answers the next writes itself with the same rejection.
// The answers come after a delay, under an exponential backoff, and one probe
// write goes to the network per backoff window. A new record that the API
// accepts, or a delete that removes a record, ends the backoff. The backoff
// also ends when a write comes 60 s or more after the last window, and no probe
// from the last 60 s is in flight. The state is in memory only.
//
// The compiled-HTML contract checks (`stateful_crud_contract.rs` and
// `stateful_hosted_release.rs`) read this file as part of the app. So it must
// not contain a quoted records action, a browser storage API, the JSON request
// header or encoder, a quoted POST method, a collection argument, or a name
// bound to a quoted word. It must also not call a network function with a
// quoted URL.
(() => {
  const nativeFetch = window.fetch;
  if (typeof nativeFetch !== 'function') return;
  const NativeRequest = window.Request;
  const NativeResponse = window.Response;
  const NativeURL = window.URL;
  const NativePromise = window.Promise;
  const parseJson = JSON.parse;
  const reflectApply = Reflect.apply;
  const startTimer = window.setTimeout.bind(window);
  const stopTimer = window.clearTimeout.bind(window);
  const monotonicNow = window.performance.now.bind(window.performance);

  const CAPACITY_STATUS = 409;
  const CAPACITY_BODY = '{"error":"artifact record limit reached"}';
  const FIRST_WINDOW_MS = 1000;
  const MAX_WINDOW_MS = 60000;
  const LOCAL_ANSWER_DELAY_MS = 500;
  const MAX_TRACKED_RECORDS = 2048;
  const actionName = operation => ['records', operation].join('.');
  const CREATE = actionName('create');
  const PUT = actionName('put');
  const GET = actionName('get');
  const LIST = actionName('list');
  const DELETE = actionName('delete');
  const OBSERVED_ACTIONS = [CREATE, PUT, GET, LIST, DELETE];
  const POST_METHOD = ['PO', 'ST'].join('');
  const LOCAL_HEADERS = [
    [['content', 'type'].join('-'), ['application', 'json'].join('/') + '; charset=utf-8'],
    ['cache-control', 'no-store'],
    ['x-metaai-records-guard', 'capacity'],
  ];

  // `epoch` changes when a window opens or closes. A rejected or accepted write
  // that was sent in an earlier epoch does not change the window, so the writes
  // that were in flight when the limit was reached count as one rejection.
  let streak = 0;
  let windowEndsAt = 0;
  let epoch = 0;
  let activeProbe = null;
  const knownIds = new Set();
  const knownKeys = new Map();

  function recordKey(collection, value) {
    return `${collection}\n${value}`;
  }

  function rememberId(collection, id) {
    if (typeof id !== 'string') return;
    const key = recordKey(collection, id);
    knownIds.delete(key);
    knownIds.add(key);
    if (knownIds.size > MAX_TRACKED_RECORDS) knownIds.delete(knownIds.values().next().value);
  }

  // `savedId` is null when the outcome of the write is unknown. Then the key
  // keeps the id that an earlier answer saved.
  function rememberIdempotencyKey(collection, idempotencyKey, savedId) {
    if (typeof idempotencyKey !== 'string') return;
    const key = recordKey(collection, idempotencyKey);
    const id = savedId ?? knownKeys.get(key) ?? null;
    knownKeys.delete(key);
    knownKeys.set(key, id);
    if (knownKeys.size > MAX_TRACKED_RECORDS) knownKeys.delete(knownKeys.keys().next().value);
  }

  function forgetRecord(savedId) {
    knownIds.delete(savedId);
    for (const [idempotencyKey, keySavedId] of knownKeys) {
      if (keySavedId === savedId) knownKeys.delete(idempotencyKey);
    }
  }

  function forgetId(collection, id) {
    if (typeof id === 'string') forgetRecord(recordKey(collection, id));
  }

  function targetsActions(input, init) {
    const isRequest = input instanceof NativeRequest;
    const method = init?.method ?? (isRequest ? input.method : 'GET');
    if (String(method).toUpperCase() !== POST_METHOD) return false;
    const base = window.location.href;
    const url = isRequest ? input.url : String(input);
    return new NativeURL(url, base).href === new NativeURL('actions', base).href;
  }

  // Returns the body text, a promise of it for a Request body, or null when
  // the body cannot be JSON text.
  function requestBody(input, init) {
    const body = init?.body;
    if (body !== undefined && body !== null) return typeof body === 'string' ? body : null;
    return input instanceof NativeRequest ? input.clone().text() : null;
  }

  function requestSignal(input, init) {
    if (init !== undefined && init !== null && init.signal !== undefined) return init.signal;
    return input instanceof NativeRequest ? input.signal : null;
  }

  function parseAction(text) {
    let payload;
    try {
      payload = parseJson(text);
    } catch {
      return null;
    }
    if (payload === null || typeof payload !== 'object' || Array.isArray(payload)) return null;
    const name = payload.action;
    if (!OBSERVED_ACTIONS.includes(name)) return null;
    const args = payload.args ?? {};
    if (typeof args !== 'object' || Array.isArray(args)) return null;
    const collection = args.collection ?? 'items';
    if (typeof collection !== 'string') return null;
    return {name, args, collection};
  }

  function isWrite(action) {
    return action.name === CREATE || action.name === PUT;
  }

  // The records API accepts these writes at the limit when their record
  // exists: a create that replays a known idempotency key, and a put to a known
  // record. A key or id is known when this page saw its record saved, or sent
  // it in a write whose outcome is unknown.
  function mayStillSucceed(action) {
    if (action.name === CREATE) {
      const idempotencyKey = action.args.idempotency_key;
      return typeof idempotencyKey === 'string'
        && knownKeys.has(recordKey(action.collection, idempotencyKey));
    }
    const id = action.args.id;
    return typeof id === 'string' && knownIds.has(recordKey(action.collection, id));
  }

  function rememberUnknownOutcome(action) {
    if (action.name === CREATE) {
      rememberIdempotencyKey(action.collection, action.args.idempotency_key, null);
    } else {
      rememberId(action.collection, action.args.id);
    }
  }

  // The API rejects a known key or id for capacity only when its record does
  // not exist.
  function revokeExemption(action) {
    if (action.name === PUT) {
      forgetId(action.collection, action.args.id);
      return;
    }
    const key = recordKey(action.collection, action.args.idempotency_key);
    const savedId = knownKeys.get(key);
    knownKeys.delete(key);
    if (typeof savedId === 'string') forgetRecord(savedId);
  }

  // A probe that never settles stops holding writes back after the longest
  // window.
  function windowIsOpen(time) {
    if (time < windowEndsAt) return true;
    return activeProbe !== null && time - activeProbe.startedAt < MAX_WINDOW_MS;
  }

  function openWindow(sentEpoch) {
    if (sentEpoch !== epoch) return;
    streak += 1;
    windowEndsAt = monotonicNow() + Math.min(MAX_WINDOW_MS, FIRST_WINDOW_MS * 2 ** (streak - 1));
    epoch += 1;
  }

  function closeWindow() {
    if (streak === 0) return;
    streak = 0;
    windowEndsAt = 0;
    activeProbe = null;
    epoch += 1;
  }

  // The delay keeps an awaited retry loop from running on every microtask.
  function answerLocally(signal) {
    return new NativePromise((resolve, reject) => {
      if (signal?.aborted) {
        reject(signal.reason);
        return;
      }
      let timer = 0;
      const abort = () => {
        stopTimer(timer);
        reject(signal.reason);
      };
      timer = startTimer(() => {
        signal?.removeEventListener('abort', abort);
        resolve(new NativeResponse(CAPACITY_BODY, {status: CAPACITY_STATUS, headers: LOCAL_HEADERS}));
      }, LOCAL_ANSWER_DELAY_MS);
      signal?.addEventListener('abort', abort, {once: true});
    });
  }

  function responseData(text) {
    try {
      const payload = parseJson(text);
      return payload !== null && typeof payload === 'object' ? payload.data : undefined;
    } catch {
      return undefined;
    }
  }

  function track(action, data) {
    const collection = action.collection;
    switch (action.name) {
      case LIST:
        if (Array.isArray(data?.records)) {
          for (const record of data.records) rememberId(collection, record?.id);
        }
        return;
      case DELETE:
        if (data === true) {
          forgetId(collection, action.args.id);
          closeWindow();
        }
        return;
      case CREATE:
        if (typeof data?.id === 'string') {
          rememberIdempotencyKey(collection, action.args.idempotency_key, recordKey(collection, data.id));
        }
        rememberId(collection, data?.id);
        return;
      default:
        rememberId(collection, data?.id);
    }
  }

  async function observe(action, response, sent) {
    try {
      const status = response.status;
      const succeeded = status >= 200 && status < 300;
      // A write that the API accepts only below the limit proves free space.
      if (succeeded && isWrite(action) && !sent.mayStillSucceed && sent.epoch === epoch) {
        closeWindow();
      }
      const capacityCandidate = status === CAPACITY_STATUS && isWrite(action);
      if (!succeeded && !capacityCandidate) return;
      const text = await response.clone().text();
      if (succeeded) {
        track(action, responseData(text));
      } else if (text === CAPACITY_BODY) {
        if (sent.mayStillSucceed) revokeExemption(action);
        openWindow(sent.epoch);
      }
    } finally {
      if (sent.probe !== null && activeProbe === sent.probe) activeProbe = null;
    }
  }

  function handle(action, send, signal) {
    if (action === null) return send();
    const sent = {epoch, mayStillSucceed: false, probe: null};
    if (isWrite(action)) {
      sent.mayStillSucceed = mayStillSucceed(action);
      if (streak > 0 && !sent.mayStillSucceed) {
        const time = monotonicNow();
        if (windowIsOpen(time)) return answerLocally(signal);
        if (time - windowEndsAt < MAX_WINDOW_MS) {
          activeProbe = {startedAt: time};
          sent.probe = activeProbe;
        } else {
          // Another client or the agent can free space where this page cannot
          // see it. So after a quiet period the write goes out as a normal write.
          closeWindow();
          sent.epoch = epoch;
        }
      }
    }
    return send().then(
      response => observe(action, response, sent).then(() => response, () => response),
      error => {
        if (sent.probe !== null && activeProbe === sent.probe) activeProbe = null;
        // A write that fails or is aborted after it is sent can still save its
        // record.
        if (isWrite(action)) rememberUnknownOutcome(action);
        throw error;
      },
    );
  }

  function guardedFetch(input, init) {
    const self = this;
    const args = arguments;
    const send = () => reflectApply(nativeFetch, self, args);
    let body;
    let signal;
    try {
      if (!targetsActions(input, init)) return send();
      body = requestBody(input, init);
      signal = requestSignal(input, init);
    } catch {
      return send();
    }
    if (body === null) return send();
    if (typeof body === 'string') return handle(parseAction(body), send, signal);
    return body.then(text => handle(parseAction(text), send, signal), () => send());
  }

  try {
    window.fetch = guardedFetch;
  } catch {
    // A page that made the network function read-only keeps its own function.
  }
})();
</script><script data-metaai-app-links>(()=>{var W=Object.defineProperty;var o=(t,n)=>W(t,"name",{value:n,configurable:!0});(()=>{const t=window.open,n=window.fetch,r=HTMLAnchorElement.prototype.click,i=Event.prototype.preventDefault,c=Event.prototype.stopPropagation,f=Event.prototype.stopImmediatePropagation,a=Object.getOwnPropertyDescriptor(Event.prototype,"defaultPrevented").get,u=Object.getOwnPropertyDescriptor(Event.prototype,"cancelBubble"),h="/__metaai/blobs/",x=4e3,g=new WeakMap,E=new WeakSet,s=[];let l=null,b=null,m=null,d=null;function p(t){if(typeof t!="string"||t.trim()==="")return null;let e;try{e=new URL(t,document.baseURI)}catch{return null}return(e.protocol==="http:"||e.protocol==="https:")&&e.origin===window.location.origin?e:null}o(p,"sameAppUrl");function k(t){const e=t.trim(),n=e.toLowerCase();return n===""||n==="_self"||n==="_parent"||n==="_top"?!1:![...document.getElementsByName(e)].some(t=>/^(IFRAME|OBJECT|EMBED)$/.test(t.tagName))}o(k,"opensNewContext");function T(t){return/^\/(__metaai|api|assets)\//.test(t.pathname)}o(T,"isFileRoute");function M(t){const e=new URL(window.location.href);return t.hash!==""&&!/^#!?\//.test(t.hash)&&t.pathname===e.pathname&&t.search===e.search}o(M,"samePage");function R(t){return(t.querySelector("img[alt]")?.getAttribute("alt")||t.getAttribute("aria-label")||t.textContent||"").trim().slice(0,200)}o(R,"labelFor");function y(t,e){const n=document.createElement(t);return n.style.cssText=e,n}o(y,"styled");function P(t){const e=Uint8Array.from(atob(t.replace(/-/g,"+").replace(/_/g,"/")),t=>t.charCodeAt(0));return new TextDecoder("utf-8",{fatal:!0}).decode(e)}o(P,"textOf");function S(t){let e={disposition:t.searchParams.get("disposition")};try{const n=t.searchParams.get("token");n&&t.pathname.startsWith(h)&&(e=JSON.parse(P(n.split(".")[0]))||e)}catch{}return e.disposition==="attachment"}o(S,"isAttachment");function D(){try{const t=document.querySelectorAll(":modal");return t[t.length-1]||document.body}catch{return document.body}}o(D,"layerHost");function H(t,e){d===null&&(d=y("div","position:fixed;width:1px;height:1px;margin:-1px;padding:0;border:0;overflow:hidden;clip-path:inset(50%);white-space:nowrap;"),d.setAttribute("role","status"),d.setAttribute("data-metaai-link-status","")),d.parentNode!==e&&e.append(d);const n=d;n.textContent="",window.setTimeout(()=>{n.textContent=t},100)}o(H,"announce");function N(){b?.remove();const t=D();m?.isConnected||(m=document.createElement("style"),m.textContent="[data-metaai-link-note]::backdrop{display:none!important;background:none!important;backdrop-filter:none!important}",(document.head||document.documentElement).append(m));const e="This app cannot open its pages in a new tab.",n=y("div","position:fixed;inset:auto 16px calc(var(--safe-area-inset-bottom, 0px) + 24px);z-index:2147483647;box-sizing:border-box;width:auto;height:auto;max-width:360px;margin:0 auto;padding:12px 16px;border:0;border-radius:10px;background:rgba(17,17,17,0.92);color:#fff;font:400 14px/1.4 system-ui,sans-serif;text-align:center;overflow:visible;pointer-events:none;");n.setAttribute("data-metaai-link-note",""),n.setAttribute("aria-hidden","true"),n.textContent=e;const r=typeof n.showPopover=="function";r&&n.setAttribute("popover","manual"),t.append(n),r&&n.showPopover(),b=n,H(e,t),window.setTimeout(()=>{n.remove(),b===n&&(b=null)},x)}o(N,"showNote");function O(t){for(const e of t)if(!(!(e instanceof Element)||e===document.body||!e.isConnected||typeof e.focus!="function")&&(e.focus({preventScroll:!0}),document.activeElement===e))return}o(O,"focusBack");function w(){if(l===null)return;const{root:t,opener:e,before:n}=l;l=null,t.open&&t.close(),t.remove(),O([e,n,...s.slice().reverse()])}o(w,"closeViewer");function A(t,e,n){const r=document.activeElement;w();const c=typeof HTMLDialogElement=="function"&&typeof HTMLDialogElement.prototype.showModal=="function",f="var(--safe-area-inset-top, 0px)",a=y(c?"dialog":"div",`position:fixed;inset:0;z-index:2147483647;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:16px;box-sizing:border-box;width:100%;height:100%;max-width:none;max-height:none;margin:0;border:0;padding:calc(${f} + 68px) 16px 24px;background:rgba(0,0,0,0.88);color:#fff;overflow:hidden;overscroll-behavior:contain;pointer-events:auto;`);a.setAttribute("data-metaai-file-viewer",""),a.setAttribute("aria-label",e||"File"),c||(a.setAttribute("role","dialog"),a.setAttribute("aria-modal","true"));const u=y("button",`position:absolute;top:calc(${f} + 12px);right:12px;width:44px;height:44px;padding:0;border:0;border-radius:22px;background:rgba(255,255,255,0.18);color:#fff;font:400 28px/44px system-ui,sans-serif;cursor:pointer;pointer-events:auto;`);u.type="button",u.textContent="\xD7",u.setAttribute("aria-label","Close"),u.setAttribute("data-metaai-file-viewer-close","");const h=y("img","display:block;flex:0 1 auto;min-height:0;max-width:100%;max-height:100%;object-fit:contain;border-radius:4px;");h.alt=e,h.addEventListener("error",()=>{const t=y("p","margin:0;font:400 15px/1.4 system-ui,sans-serif;text-align:center;");t.textContent="This file cannot be shown here.",h.replaceWith(t)},{once:!0}),h.src=t,a.append(u,h),a.addEventListener("cancel",t=>{Reflect.apply(i,t,[]),l?.root===a&&w()}),a.addEventListener("close",()=>{l?.root===a&&w()}),l={root:a,opener:n,before:r},document.body.append(a),c&&a.showModal(),u.focus({preventScroll:!0})}o(A,"showViewer");function F(t,e,r){const i=new AbortController;Reflect.apply(n,window,[t.href,{signal:i.signal}]).then(n=>{i.abort(),n.ok?window.location.assign(t.href):A(t.href,e,r)},()=>A(t.href,e,r))}o(F,"download");function C(t,e,n){T(t)?S(t)?F(t,e,n):A(t.href,e,n):M(t)?window.location.hash=t.hash:N()}o(C,"openOutside");function _(n,r){let i=null,c=0,f=n.href;const a=o(e=>{try{const n=p(String(e));if(n===null){i!==null&&l===i&&w(),i=null,c=0,Reflect.apply(t,window,[String(e)]);return}const a=l;C(n,"",r),i=l!==a?l:null,c=i===null?Date.now()+x:0,f=n.href}catch{}},"go"),u={get href(){return f},set href(t){a(t)},assign:a,replace:a,reload(){},toString:o(()=>f,"toString")};return a(n.href),{opener:null,get closed(){return i!==null?l!==i:Date.now()>=c},get location(){return u},set location(t){a(t)},focus(){},blur(){},close(){i!==null&&l===i&&w(),i=null,c=0},postMessage(){},addEventListener(){},removeEventListener(){}}}o(_,"standIn");function B(t){if(l===null)return!1;const e=t.relatedTarget;return t.composedPath().includes(l.root)||e instanceof Node&&l.root.contains(e)}o(B,"insideViewer");function I(t){const e=[...l.root.querySelectorAll("button, a[href]")],n=e.indexOf(document.activeElement);(t.shiftKey?e[n<=0?e.length-1:n-1]:e[n<0||n===e.length-1?0:n+1])?.focus()}o(I,"moveFocus");function U(t){try{if(l!==null&&!l.root.isConnected&&(l=null),l===null){t.type==="focusin"&&t.target instanceof Element&&s[s.length-1]!==t.target&&(s.push(t.target),s.length>5&&s.shift());return}if(t.type==="keydown"&&(t.key==="Escape"||t.key==="Tab")){Reflect.apply(i,t,[]),Reflect.apply(f,t,[]),t.key==="Escape"?w():I(t);return}if(!B(t)||(Reflect.apply(f,t,[]),t.type!=="click"))return;if(t.detail>1){Reflect.apply(i,t,[]);return}const e=t.target;(e===l.root||e instanceof Element&&e.closest("[data-metaai-file-viewer-close]"))&&w()}catch{}}o(U,"shield");function $(t,e){return t instanceof HTMLInputElement?/^(checkbox|radio|file|color)$/.test(t.type)||/^(submit|reset|image)$/.test(t.type)&&t.form!==null:t instanceof HTMLButtonElement?t.type!=="button"&&t.form!==null:t instanceof HTMLLabelElement?t.control!==null&&!t.control.contains(e):t instanceof HTMLMediaElement?t.controls:t instanceof HTMLElement&&t.tagName==="SUMMARY"&&t.parentElement?.tagName==="DETAILS"}o($,"ownClickAction");function j(t){for(const e of t){if(e instanceof Element&&e.matches("a[href], area[href]"))return e;if($(e,t[0]))return null}return null}o(j,"anchorFor");function V(t){const e=g.get(t);if(!(e===void 0||e.held)){g.delete(t);try{if(e.host.removeEventListener(t.type,e.last),Reflect.apply(a,t,[]))return;e.done=!0,Reflect.apply(i,t,[]),C(e.url,e.label,e.opener)}catch{}}}o(V,"decide");function z(t){const e=g.get(t);if(e!==void 0){g.delete(t);try{e.host.removeEventListener(t.type,e.last),e.held&&!e.appCancelled&&C(e.url,e.label,e.opener)}catch{}}}o(z,"settle");function K(t,n){const r=[];for(const i of t)try{const t=i[n];if(typeof t!="function"||E.has(t))continue;const c=o(function(...e){const n=Reflect.apply(t,this,e),r=n===!1?g.get(e[0]):void 0;return r&&(r.appCancelled=!0),n},"wrapper");E.add(c),i[n]=c,r.push(()=>{i[n]===c&&(i[n]=t)})}catch{}return()=>r.forEach(t=>t())}o(K,"watchHandlers");function L(t,n,r,h,x,E){const s={url:n,label:r,opener:h,host:x,held:!1,done:!1,appCancelled:!1,last:null};s.last=e=>{e===t&&V(t)},g.set(t,s),x.addEventListener(t.type,s.last);const l=K(E,`on${t.type}`),b=o(()=>{l(),z(t)},"end");window.setTimeout(b,0);const m=o(()=>{s.held||s.done||g.get(t)!==s||(s.held=!0,Reflect.apply(a,t,[])?s.appCancelled=!0:Reflect.apply(i,t,[]))},"hold"),d=o(t=>s.held||s.done?s.appCancelled:Reflect.apply(a,t,[]),"appView"),p=o((e,n)=>Object.defineProperty(t,e,{configurable:!0,...n}),"define");return p("stopPropagation",{writable:!0,value(){return m(),Reflect.apply(c,this,arguments)}}),p("stopImmediatePropagation",{writable:!0,value(){return m(),Reflect.apply(f,this,arguments)}}),p("preventDefault",{writable:!0,value(){return s.appCancelled=!0,Reflect.apply(i,this,arguments)}}),p("defaultPrevented",{get(){return d(this)}}),p("returnValue",{get(){return!d(this)},set(t){t||(s.appCancelled=!0,Reflect.apply(i,this,[]))}}),u?.get&&u.set&&p("cancelBubble",{get(){return Reflect.apply(u.get,this,[])},set(t){t&&m(),Reflect.apply(u.set,this,[t])}}),b}o(L,"track");function v(t){try{if(!t.cancelable||t.button!==(t.type==="auxclick"?1:0))return;const e=t.composedPath(),n=j(e);if(!n||n.hasAttribute("download")||n.isContentEditable||!t.bubbles&&e[0]!==n)return;const r=p(n.getAttribute("href"));if(r===null)return;(t.type==="auxclick"||t.ctrlKey||t.metaKey||t.shiftKey||k(n.getAttribute("target")||""))&&L(t,r,R(n),n,t.bubbles?window:n,e)}catch{}}o(v,"remember");try{for(const t of["pointerdown","pointerup","mousedown","mouseup","touchstart","touchend","click","auxclick","dblclick","focusin","focusout","keydown","keyup"])window.addEventListener(t,U,!0);window.addEventListener("click",v,!0),window.addEventListener("auxclick",v,!0),window.open=o(function(n,r){try{const t=r==null?"":String(r);if(t.trim()===""||k(t)){const t=p(n==null?"":String(n));if(t!==null)return _(t,document.activeElement)}}catch{}return Reflect.apply(t,this,arguments)},"open"),HTMLAnchorElement.prototype.click=o(function(){try{if(!this.isConnected&&!this.hasAttribute("download")){const t=p(this.getAttribute("href"));if(t!==null&&k(this.getAttribute("target")||"")){const e=new MouseEvent("click",{bubbles:!0,cancelable:!0,composed:!0,view:window}),n=[];for(let t=this;t!==null;t=t.parentNode)n.push(t);const r=L(e,t,R(this),document.activeElement,this.getRootNode(),n);this.dispatchEvent(e),r();return}}}catch{}return Reflect.apply(r,this,arguments)},"click")}catch{}})();})();
</script>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>React Artifact</title>
  <script>(function installDevicePermissionRuntime() {
  if (window.parent === window)
    return;
  let parentOrigin;
  try {
    const embeddingOrigin = window.location.ancestorOrigins?.[0] || document.referrer, { hostname, origin, port, protocol } = new URL(embeddingOrigin), supportedHost = hostname === "meta.ai" || hostname === "ecto1.ai" || [".meta.ai", ".ecto1.ai", ".internalfb.com", ".hatch.club"].some((suffix) => hostname.endsWith(suffix));
    if (hostname === "localhost" && (protocol === "http:" || protocol === "https:") || protocol === "https:" && port === "" && supportedHost)
      parentOrigin = origin;
  } catch {
    return;
  }
  if (!parentOrigin)
    return;
  function request(capability) {
    try {
      const policy = document.permissionsPolicy || document.featurePolicy;
      if (navigator.userActivation?.isActive === !0 && policy?.allowsFeature(capability) === !1)
        window.parent.postMessage({ type: "ecto:artifact-device-permission-request", version: 1, capability }, parentOrigin);
    } catch {}
  }
  function hasAudio(constraints) {
    for (let value = constraints;value != null; value = Object.getPrototypeOf(value)) {
      const descriptor = Object.getOwnPropertyDescriptor(value, "audio");
      if (descriptor)
        return Boolean(descriptor.value);
    }
    return !1;
  }
  function wrap(receiver, name, capability, enabled) {
    try {
      if ((document.permissionsPolicy || document.featurePolicy)?.allowsFeature(capability) !== !1)
        return;
      const native = receiver?.[name];
      if (typeof native !== "function")
        return;
      receiver[name] = function() {
        try {
          if (this === receiver && enabled(arguments))
            request(capability);
        } catch {}
        return Reflect.apply(native, this, arguments);
      };
    } catch {}
  }
  wrap(navigator.mediaDevices, "getUserMedia", "microphone", (args) => hasAudio(args[0]));
  const hasPositionCallback = (args) => typeof args[0] === "function";
  wrap(navigator.geolocation, "getCurrentPosition", "geolocation", hasPositionCallback);
  wrap(navigator.geolocation, "watchPosition", "geolocation", hasPositionCallback);
})();</script>
  <style>@import url('https://fonts.googleapis.com/css2?family=Archivo:wght@400;500;600;700&display=swap');

*, ::before, ::after{
  --tw-border-spacing-x: 0;
  --tw-border-spacing-y: 0;
  --tw-translate-x: 0;
  --tw-translate-y: 0;
  --tw-rotate: 0;
  --tw-skew-x: 0;
  --tw-skew-y: 0;
  --tw-scale-x: 1;
  --tw-scale-y: 1;
  --tw-pan-x:  ;
  --tw-pan-y:  ;
  --tw-pinch-zoom:  ;
  --tw-scroll-snap-strictness: proximity;
  --tw-gradient-from-position:  ;
  --tw-gradient-via-position:  ;
  --tw-gradient-to-position:  ;
  --tw-ordinal:  ;
  --tw-slashed-zero:  ;
  --tw-numeric-figure:  ;
  --tw-numeric-spacing:  ;
  --tw-numeric-fraction:  ;
  --tw-ring-inset:  ;
  --tw-ring-offset-width: 0px;
  --tw-ring-offset-color: #fff;
  --tw-ring-color: rgb(59 130 246 / 0.5);
  --tw-ring-offset-shadow: 0 0 #0000;
  --tw-ring-shadow: 0 0 #0000;
  --tw-shadow: 0 0 #0000;
  --tw-shadow-colored: 0 0 #0000;
  --tw-blur:  ;
  --tw-brightness:  ;
  --tw-contrast:  ;
  --tw-grayscale:  ;
  --tw-hue-rotate:  ;
  --tw-invert:  ;
  --tw-saturate:  ;
  --tw-sepia:  ;
  --tw-drop-shadow:  ;
  --tw-backdrop-blur:  ;
  --tw-backdrop-brightness:  ;
  --tw-backdrop-contrast:  ;
  --tw-backdrop-grayscale:  ;
  --tw-backdrop-hue-rotate:  ;
  --tw-backdrop-invert:  ;
  --tw-backdrop-opacity:  ;
  --tw-backdrop-saturate:  ;
  --tw-backdrop-sepia:  ;
  --tw-contain-size:  ;
  --tw-contain-layout:  ;
  --tw-contain-paint:  ;
  --tw-contain-style:  ;
}

::backdrop{
  --tw-border-spacing-x: 0;
  --tw-border-spacing-y: 0;
  --tw-translate-x: 0;
  --tw-translate-y: 0;
  --tw-rotate: 0;
  --tw-skew-x: 0;
  --tw-skew-y: 0;
  --tw-scale-x: 1;
  --tw-scale-y: 1;
  --tw-pan-x:  ;
  --tw-pan-y:  ;
  --tw-pinch-zoom:  ;
  --tw-scroll-snap-strictness: proximity;
  --tw-gradient-from-position:  ;
  --tw-gradient-via-position:  ;
  --tw-gradient-to-position:  ;
  --tw-ordinal:  ;
  --tw-slashed-zero:  ;
  --tw-numeric-figure:  ;
  --tw-numeric-spacing:  ;
  --tw-numeric-fraction:  ;
  --tw-ring-inset:  ;
  --tw-ring-offset-width: 0px;
  --tw-ring-offset-color: #fff;
  --tw-ring-color: rgb(59 130 246 / 0.5);
  --tw-ring-offset-shadow: 0 0 #0000;
  --tw-ring-shadow: 0 0 #0000;
  --tw-shadow: 0 0 #0000;
  --tw-shadow-colored: 0 0 #0000;
  --tw-blur:  ;
  --tw-brightness:  ;
  --tw-contrast:  ;
  --tw-grayscale:  ;
  --tw-hue-rotate:  ;
  --tw-invert:  ;
  --tw-saturate:  ;
  --tw-sepia:  ;
  --tw-drop-shadow:  ;
  --tw-backdrop-blur:  ;
  --tw-backdrop-brightness:  ;
  --tw-backdrop-contrast:  ;
  --tw-backdrop-grayscale:  ;
  --tw-backdrop-hue-rotate:  ;
  --tw-backdrop-invert:  ;
  --tw-backdrop-opacity:  ;
  --tw-backdrop-saturate:  ;
  --tw-backdrop-sepia:  ;
  --tw-contain-size:  ;
  --tw-contain-layout:  ;
  --tw-contain-paint:  ;
  --tw-contain-style:  ;
}

/*
! tailwindcss v3.4.18 | MIT License | https://tailwindcss.com
*/

/*
1. Prevent padding and border from affecting element width. (https://github.com/mozdevs/cssremedy/issues/4)
2. Allow adding a border to an element by just adding a border-width. (https://github.com/tailwindcss/tailwindcss/pull/116)
*/

*,
::before,
::after {
  box-sizing: border-box;
  /* 1 */
  border-width: 0;
  /* 2 */
  border-style: solid;
  /* 2 */
  border-color: #e5e7eb;
  /* 2 */
}

::before,
::after {
  --tw-content: '';
}

/*
1. Use a consistent sensible line-height in all browsers.
2. Prevent adjustments of font size after orientation changes in iOS.
3. Use a more readable tab size.
4. Use the user's configured `sans` font-family by default.
5. Use the user's configured `sans` font-feature-settings by default.
6. Use the user's configured `sans` font-variation-settings by default.
7. Disable tap highlights on iOS
*/

html,
:host {
  line-height: 1.5;
  /* 1 */
  -webkit-text-size-adjust: 100%;
  /* 2 */
  -moz-tab-size: 4;
  /* 3 */
  -o-tab-size: 4;
     tab-size: 4;
  /* 3 */
  font-family: ui-sans-serif, system-ui, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol", "Noto Color Emoji";
  /* 4 */
  font-feature-settings: normal;
  /* 5 */
  font-variation-settings: normal;
  /* 6 */
  -webkit-tap-highlight-color: transparent;
  /* 7 */
}

/*
1. Remove the margin in all browsers.
2. Inherit line-height from `html` so users can set them as a class directly on the `html` element.
*/

body {
  margin: 0;
  /* 1 */
  line-height: inherit;
  /* 2 */
}

/*
1. Add the correct height in Firefox.
2. Correct the inheritance of border color in Firefox. (https://bugzilla.mozilla.org/show_bug.cgi?id=190655)
3. Ensure horizontal rules are visible by default.
*/

hr {
  height: 0;
  /* 1 */
  color: inherit;
  /* 2 */
  border-top-width: 1px;
  /* 3 */
}

/*
Add the correct text decoration in Chrome, Edge, and Safari.
*/

abbr:where([title]) {
  -webkit-text-decoration: underline dotted;
          text-decoration: underline dotted;
}

/*
Remove the default font size and weight for headings.
*/

h1,
h2,
h3,
h4,
h5,
h6 {
  font-size: inherit;
  font-weight: inherit;
}

/*
Reset links to optimize for opt-in styling instead of opt-out.
*/

a {
  color: inherit;
  text-decoration: inherit;
}

/*
Add the correct font weight in Edge and Safari.
*/

b,
strong {
  font-weight: bolder;
}

/*
1. Use the user's configured `mono` font-family by default.
2. Use the user's configured `mono` font-feature-settings by default.
3. Use the user's configured `mono` font-variation-settings by default.
4. Correct the odd `em` font sizing in all browsers.
*/

code,
kbd,
samp,
pre {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
  /* 1 */
  font-feature-settings: normal;
  /* 2 */
  font-variation-settings: normal;
  /* 3 */
  font-size: 1em;
  /* 4 */
}

/*
Add the correct font size in all browsers.
*/

small {
  font-size: 80%;
}

/*
Prevent `sub` and `sup` elements from affecting the line height in all browsers.
*/

sub,
sup {
  font-size: 75%;
  line-height: 0;
  position: relative;
  vertical-align: baseline;
}

sub {
  bottom: -0.25em;
}

sup {
  top: -0.5em;
}

/*
1. Remove text indentation from table contents in Chrome and Safari. (https://bugs.chromium.org/p/chromium/issues/detail?id=999088, https://bugs.webkit.org/show_bug.cgi?id=201297)
2. Correct table border color inheritance in all Chrome and Safari. (https://bugs.chromium.org/p/chromium/issues/detail?id=935729, https://bugs.webkit.org/show_bug.cgi?id=195016)
3. Remove gaps between table borders by default.
*/

table {
  text-indent: 0;
  /* 1 */
  border-color: inherit;
  /* 2 */
  border-collapse: collapse;
  /* 3 */
}

/*
1. Change the font styles in all browsers.
2. Remove the margin in Firefox and Safari.
3. Remove default padding in all browsers.
*/

button,
input,
optgroup,
select,
textarea {
  font-family: inherit;
  /* 1 */
  font-feature-settings: inherit;
  /* 1 */
  font-variation-settings: inherit;
  /* 1 */
  font-size: 100%;
  /* 1 */
  font-weight: inherit;
  /* 1 */
  line-height: inherit;
  /* 1 */
  letter-spacing: inherit;
  /* 1 */
  color: inherit;
  /* 1 */
  margin: 0;
  /* 2 */
  padding: 0;
  /* 3 */
}

/*
Remove the inheritance of text transform in Edge and Firefox.
*/

button,
select {
  text-transform: none;
}

/*
1. Correct the inability to style clickable types in iOS and Safari.
2. Remove default button styles.
*/

button,
input:where([type='button']),
input:where([type='reset']),
input:where([type='submit']) {
  -webkit-appearance: button;
  /* 1 */
  background-color: transparent;
  /* 2 */
  background-image: none;
  /* 2 */
}

/*
Use the modern Firefox focus style for all focusable elements.
*/

:-moz-focusring {
  outline: auto;
}

/*
Remove the additional `:invalid` styles in Firefox. (https://github.com/mozilla/gecko-dev/blob/2f9eacd9d3d995c937b4251a5557d95d494c9be1/layout/style/res/forms.css#L728-L737)
*/

:-moz-ui-invalid {
  box-shadow: none;
}

/*
Add the correct vertical alignment in Chrome and Firefox.
*/

progress {
  vertical-align: baseline;
}

/*
Correct the cursor style of increment and decrement buttons in Safari.
*/

::-webkit-inner-spin-button,
::-webkit-outer-spin-button {
  height: auto;
}

/*
1. Correct the odd appearance in Chrome and Safari.
2. Correct the outline style in Safari.
*/

[type='search'] {
  -webkit-appearance: textfield;
  /* 1 */
  outline-offset: -2px;
  /* 2 */
}

/*
Remove the inner padding in Chrome and Safari on macOS.
*/

::-webkit-search-decoration {
  -webkit-appearance: none;
}

/*
1. Correct the inability to style clickable types in iOS and Safari.
2. Change font properties to `inherit` in Safari.
*/

::-webkit-file-upload-button {
  -webkit-appearance: button;
  /* 1 */
  font: inherit;
  /* 2 */
}

/*
Add the correct display in Chrome and Safari.
*/

summary {
  display: list-item;
}

/*
Removes the default spacing and border for appropriate elements.
*/

blockquote,
dl,
dd,
h1,
h2,
h3,
h4,
h5,
h6,
hr,
figure,
p,
pre {
  margin: 0;
}

fieldset {
  margin: 0;
  padding: 0;
}

legend {
  padding: 0;
}

ol,
ul,
menu {
  list-style: none;
  margin: 0;
  padding: 0;
}

/*
Reset default styling for dialogs.
*/

dialog {
  padding: 0;
}

/*
Prevent resizing textareas horizontally by default.
*/

textarea {
  resize: vertical;
}

/*
1. Reset the default placeholder opacity in Firefox. (https://github.com/tailwindlabs/tailwindcss/issues/3300)
2. Set the default placeholder color to the user's configured gray 400 color.
*/

input::-moz-placeholder, textarea::-moz-placeholder {
  opacity: 1;
  /* 1 */
  color: #9ca3af;
  /* 2 */
}

input::placeholder,
textarea::placeholder {
  opacity: 1;
  /* 1 */
  color: #9ca3af;
  /* 2 */
}

/*
Set the default cursor for buttons.
*/

button,
[role="button"] {
  cursor: pointer;
}

/*
Make sure disabled buttons don't get the pointer cursor.
*/

:disabled {
  cursor: default;
}

/*
1. Make replaced elements `display: block` by default. (https://github.com/mozdevs/cssremedy/issues/14)
2. Add `vertical-align: middle` to align replaced elements more sensibly by default. (https://github.com/jensimmons/cssremedy/issues/14#issuecomment-634934210)
   This can trigger a poorly considered lint error in some tools but is included by design.
*/

img,
svg,
video,
canvas,
audio,
iframe,
embed,
object {
  display: block;
  /* 1 */
  vertical-align: middle;
  /* 2 */
}

/*
Constrain images and videos to the parent width and preserve their intrinsic aspect ratio. (https://github.com/mozdevs/cssremedy/issues/14)
*/

img,
video {
  max-width: 100%;
  height: auto;
}

/* Make elements with the HTML hidden attribute stay hidden by default */

[hidden]:where(:not([hidden="until-found"])) {
  display: none;
}

.fixed{
  position: fixed;
}

.sticky{
  position: sticky;
}

.inset-0{
  inset: 0px;
}

.z-30{
  z-index: 30;
}

.z-50{
  z-index: 50;
}

.m-1{
  margin: 0.25rem;
}

.m-2{
  margin: 0.5rem;
}

.m-3{
  margin: 0.75rem;
}

.m-4{
  margin: 1rem;
}

.m-5{
  margin: 1.25rem;
}

.m-6{
  margin: 1.5rem;
}

.m-7{
  margin: 1.75rem;
}

.mx-auto{
  margin-left: auto;
  margin-right: auto;
}

.my-2{
  margin-top: 0.5rem;
  margin-bottom: 0.5rem;
}

.mb-1{
  margin-bottom: 0.25rem;
}

.mb-4{
  margin-bottom: 1rem;
}

.mt-0\.5{
  margin-top: 0.125rem;
}

.mt-1{
  margin-top: 0.25rem;
}

.mt-10{
  margin-top: 2.5rem;
}

.mt-2{
  margin-top: 0.5rem;
}

.mt-3{
  margin-top: 0.75rem;
}

.mt-4{
  margin-top: 1rem;
}

.mt-6{
  margin-top: 1.5rem;
}

.block{
  display: block;
}

.inline-block{
  display: inline-block;
}

.flex{
  display: flex;
}

.inline-flex{
  display: inline-flex;
}

.table{
  display: table;
}

.grid{
  display: grid;
}

.aspect-\[9\/16\]{
  aspect-ratio: 9/16;
}

.h-2{
  height: 0.5rem;
}

.h-9{
  height: 2.25rem;
}

.h-full{
  height: 100%;
}

.max-h-\[92vh\]{
  max-height: 92vh;
}

.w-2{
  width: 0.5rem;
}

.w-9{
  width: 2.25rem;
}

.w-full{
  width: 100%;
}

.min-w-0{
  min-width: 0px;
}

.min-w-\[520px\]{
  min-width: 520px;
}

.min-w-\[640px\]{
  min-width: 640px;
}

.max-w-2xl{
  max-width: 42rem;
}

.max-w-3xl{
  max-width: 48rem;
}

.max-w-md{
  max-width: 28rem;
}

.flex-1{
  flex: 1 1 0%;
}

.shrink-0{
  flex-shrink: 0;
}

.rotate-180{
  --tw-rotate: 180deg;
  transform: translate(var(--tw-translate-x), var(--tw-translate-y)) rotate(var(--tw-rotate)) skewX(var(--tw-skew-x)) skewY(var(--tw-skew-y)) scaleX(var(--tw-scale-x)) scaleY(var(--tw-scale-y));
}

.flex-wrap{
  flex-wrap: wrap;
}

.place-items-center{
  place-items: center;
}

.items-start{
  align-items: flex-start;
}

.items-end{
  align-items: flex-end;
}

.items-center{
  align-items: center;
}

.justify-center{
  justify-content: center;
}

.justify-between{
  justify-content: space-between;
}

.gap-1{
  gap: 0.25rem;
}

.gap-2{
  gap: 0.5rem;
}

.gap-3{
  gap: 0.75rem;
}

.space-y-0 > :not([hidden]) ~ :not([hidden]){
  --tw-space-y-reverse: 0;
  margin-top: calc(0px * calc(1 - var(--tw-space-y-reverse)));
  margin-bottom: calc(0px * var(--tw-space-y-reverse));
}

.space-y-2 > :not([hidden]) ~ :not([hidden]){
  --tw-space-y-reverse: 0;
  margin-top: calc(0.5rem * calc(1 - var(--tw-space-y-reverse)));
  margin-bottom: calc(0.5rem * var(--tw-space-y-reverse));
}

.space-y-3 > :not([hidden]) ~ :not([hidden]){
  --tw-space-y-reverse: 0;
  margin-top: calc(0.75rem * calc(1 - var(--tw-space-y-reverse)));
  margin-bottom: calc(0.75rem * var(--tw-space-y-reverse));
}

.space-y-4 > :not([hidden]) ~ :not([hidden]){
  --tw-space-y-reverse: 0;
  margin-top: calc(1rem * calc(1 - var(--tw-space-y-reverse)));
  margin-bottom: calc(1rem * var(--tw-space-y-reverse));
}

.overflow-hidden{
  overflow: hidden;
}

.overflow-x-auto{
  overflow-x: auto;
}

.overflow-y-auto{
  overflow-y: auto;
}

.overflow-x-clip{
  overflow-x: clip;
}

.whitespace-nowrap{
  white-space: nowrap;
}

.rounded-2xl{
  border-radius: 1rem;
}

.rounded-\[24px\]{
  border-radius: 24px;
}

.rounded-full{
  border-radius: 9999px;
}

.border{
  border-width: 1px;
}

.border-b{
  border-bottom-width: 1px;
}

.border-t{
  border-top-width: 1px;
}

.border-\[var\(--line\)\]{
  border-color: var(--line);
}

.border-\[var\(--line-strong\)\]{
  border-color: var(--line-strong);
}

.border-\[var\(--text\)\]{
  border-color: var(--text);
}

.bg-\[\#e8e6e1\]{
  --tw-bg-opacity: 1;
  background-color: rgb(232 230 225 / var(--tw-bg-opacity, 1));
}

.bg-\[\#fff7f0\]{
  --tw-bg-opacity: 1;
  background-color: rgb(255 247 240 / var(--tw-bg-opacity, 1));
}

.bg-\[var\(--accent-soft\)\]{
  background-color: var(--accent-soft);
}

.bg-\[var\(--bg\)\]{
  background-color: var(--bg);
}

.bg-\[var\(--surface\)\]{
  background-color: var(--surface);
}

.bg-\[var\(--text\)\]{
  background-color: var(--text);
}

.bg-black\/50{
  background-color: rgb(0 0 0 / 0.5);
}

.bg-transparent{
  background-color: transparent;
}

.bg-white{
  --tw-bg-opacity: 1;
  background-color: rgb(255 255 255 / var(--tw-bg-opacity, 1));
}

.p-3{
  padding: 0.75rem;
}

.p-4{
  padding: 1rem;
}

.p-5{
  padding: 1.25rem;
}

.px-2\.5{
  padding-left: 0.625rem;
  padding-right: 0.625rem;
}

.px-3{
  padding-left: 0.75rem;
  padding-right: 0.75rem;
}

.px-4{
  padding-left: 1rem;
  padding-right: 1rem;
}

.py-1{
  padding-top: 0.25rem;
  padding-bottom: 0.25rem;
}

.py-1\.5{
  padding-top: 0.375rem;
  padding-bottom: 0.375rem;
}

.py-2{
  padding-top: 0.5rem;
  padding-bottom: 0.5rem;
}

.py-2\.5{
  padding-top: 0.625rem;
  padding-bottom: 0.625rem;
}

.py-3{
  padding-top: 0.75rem;
  padding-bottom: 0.75rem;
}

.py-5{
  padding-top: 1.25rem;
  padding-bottom: 1.25rem;
}

.pb-12{
  padding-bottom: 3rem;
}

.pb-4{
  padding-bottom: 1rem;
}

.pb-5{
  padding-bottom: 1.25rem;
}

.pb-6{
  padding-bottom: 1.5rem;
}

.pt-4{
  padding-top: 1rem;
}

.pt-8{
  padding-top: 2rem;
}

.text-left{
  text-align: left;
}

.text-center{
  text-align: center;
}

.text-\[12px\]{
  font-size: 12px;
}

.text-\[15px\]{
  font-size: 15px;
}

.text-\[16px\]{
  font-size: 16px;
}

.text-sm{
  font-size: 0.875rem;
  line-height: 1.25rem;
}

.text-xs{
  font-size: 0.75rem;
  line-height: 1rem;
}

.font-semibold{
  font-weight: 600;
}

.uppercase{
  text-transform: uppercase;
}

.leading-\[1\.2\]{
  line-height: 1.2;
}

.leading-none{
  line-height: 1;
}

.leading-relaxed{
  line-height: 1.625;
}

.leading-snug{
  line-height: 1.375;
}

.tracking-\[0\.18em\]{
  letter-spacing: 0.18em;
}

.tracking-widest{
  letter-spacing: 0.1em;
}

.text-\[var\(--muted\)\]{
  color: var(--muted);
}

.text-\[var\(--text\)\]{
  color: var(--text);
}

.text-white{
  --tw-text-opacity: 1;
  color: rgb(255 255 255 / var(--tw-text-opacity, 1));
}

.shadow-\[0_8px_30px_rgba\(0\2c 0\2c 0\2c 0\.06\)\]{
  --tw-shadow: 0 8px 30px rgba(0,0,0,0.06);
  --tw-shadow-colored: 0 8px 30px var(--tw-shadow-color);
  box-shadow: var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000), var(--tw-shadow);
}

.outline-none{
  outline: 2px solid transparent;
  outline-offset: 2px;
}

.filter{
  filter: var(--tw-blur) var(--tw-brightness) var(--tw-contrast) var(--tw-grayscale) var(--tw-hue-rotate) var(--tw-invert) var(--tw-saturate) var(--tw-sepia) var(--tw-drop-shadow);
}

.transition{
  transition-property: color, background-color, border-color, text-decoration-color, fill, stroke, opacity, box-shadow, transform, filter, backdrop-filter;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
  transition-duration: 150ms;
}

.transition-all{
  transition-property: all;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
  transition-duration: 150ms;
}

:root{
  --bg:#f6f6f4;
  --surface:#ffffff;
  --surface-2:#efefec;
  --line:#e8e6e1;
  --line-strong:#1c1a17;
  --text:#1c1a17;
  --muted:#6e6a64;
  --soft:#9a958e;
  --accent:#c47a5a;
  --accent-soft:#f7e9e1;
  --terracotta:#c47a5a;
}

*{
  box-sizing:border-box
}

html{
  scroll-behavior:smooth
}

body{
  margin:0;
  background:var(--bg);
  color:var(--text);
  font-family:'Archivo',system-ui,-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;
  -webkit-font-smoothing:antialiased;
  font-size:15px;
  line-height:1.6
}

button{
  font-family:inherit
}

a{
  color:inherit
}

:focus-visible{
  outline:2px solid var(--text);
  outline-offset:3px
}

.check-box{
  width:30px;
  height:30px;
  min-width:30px;
  border:1.5px solid var(--text);
  border-radius:999px;
  display:grid;
  place-items:center;
  background:var(--surface);
  font-size:14px;
  line-height:1;
  cursor:pointer
}

.check-box[aria-pressed="true"]{
  background:var(--text);
  color:#fff
}

.caption-box{
  white-space:pre-wrap;
  overflow-wrap:anywhere
}

.long-text{
  overflow-wrap:anywhere
}

.media-frame{
  background:#111;
  color:#fff;
  min-height:320px;
  border-radius:18px;
  overflow:hidden
}

.media-frame>div{
  width:100%;
  height:100%
}

.card-body{
  display:grid;
  grid-template-rows:0fr;
  transition:grid-template-rows .35s ease
}

.card-body.open{
  grid-template-rows:1fr
}

.card-body>div{
  overflow:hidden
}

.seal-base{
  background:#efefec;
  color:#1c1a17;
  border:1px solid #e2e0db
}

.seal-inova{
  background:#1c1a17;
  color:#fff
}

input{
  font-family:inherit
}

.placeholder\:text-\[var\(--soft\)\]::-moz-placeholder{
  color: var(--soft);
}

.placeholder\:text-\[var\(--soft\)\]::placeholder{
  color: var(--soft);
}

.last\:border-0:last-child{
  border-width: 0px;
}

@media (min-width: 640px){
  .sm\:grid-cols-2{
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .sm\:grid-cols-3{
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }

  .sm\:items-center{
    align-items: center;
  }

  .sm\:p-5{
    padding: 1.25rem;
  }

  .sm\:p-6{
    padding: 1.5rem;
  }

  .sm\:px-5{
    padding-left: 1.25rem;
    padding-right: 1.25rem;
  }

  .sm\:px-6{
    padding-left: 1.5rem;
    padding-right: 1.5rem;
  }
}
</style>
</head>
<body>
  <div id="root"></div>
  
  <script type="module">var Rc=Object.create;var{getPrototypeOf:bc,defineProperty:tn,getOwnPropertyNames:Fc}=Object;var Ic=Object.prototype.hasOwnProperty;var Ue=(e,a,t)=>{t=e!=null?Rc(bc(e)):{};let r=a||!e||!e.__esModule?tn(t,"default",{value:e,enumerable:!0}):t;for(let o of Fc(e))if(!Ic.call(r,o))tn(r,o,{get:()=>e[o],enumerable:!0});return r};var Mr=(e,a)=>()=>(a||e((a={exports:{}}).exports,a),a.exports);var Dc=(e,a)=>{for(var t in a)tn(e,t,{get:a[t],enumerable:!0,configurable:!0,set:(r)=>a[t]=()=>r})};var Ac=(e,a)=>()=>(e&&(a=e(e=0)),a);var Ca=Mr((em)=>{var St=Symbol.for("react.element"),Lc=Symbol.for("react.portal"),Bc=Symbol.for("react.fragment"),Vc=Symbol.for("react.strict_mode"),Qc=Symbol.for("react.profiler"),Uc=Symbol.for("react.provider"),Hc=Symbol.for("react.context"),$c=Symbol.for("react.forward_ref"),Wc=Symbol.for("react.suspense"),Kc=Symbol.for("react.memo"),Yc=Symbol.for("react.lazy"),Oi=Symbol.iterator;function Xc(e){if(e===null||typeof e!=="object")return null;return e=Oi&&e[Oi]||e["@@iterator"],typeof e==="function"?e:null}var Fi={isMounted:function(){return!1},enqueueForceUpdate:function(){},enqueueReplaceState:function(){},enqueueSetState:function(){}},Ii=Object.assign,Di={};function Va(e,a,t){this.props=e,this.context=a,this.refs=Di,this.updater=t||Fi}Va.prototype.isReactComponent={};Va.prototype.setState=function(e,a){if(typeof e!=="object"&&typeof e!=="function"&&e!=null)throw Error("setState(...): takes an object of state variables to update or a function which returns an object of state variables.");this.updater.enqueueSetState(this,e,a,"setState")};Va.prototype.forceUpdate=function(e){this.updater.enqueueForceUpdate(this,e,"forceUpdate")};function Ai(){}Ai.prototype=Va.prototype;function on(e,a,t){this.props=e,this.context=a,this.refs=Di,this.updater=t||Fi}var nn=on.prototype=new Ai;nn.constructor=on;Ii(nn,Va.prototype);nn.isPureReactComponent=!0;var Ri=Array.isArray,Li=Object.prototype.hasOwnProperty,ln={current:null},Bi={key:!0,ref:!0,__self:!0,__source:!0};function Vi(e,a,t){var r,o={},n=null,l=null;if(a!=null)for(r in a.ref!==void 0&&(l=a.ref),a.key!==void 0&&(n=""+a.key),a)Li.call(a,r)&&!Bi.hasOwnProperty(r)&&(o[r]=a[r]);var s=arguments.length-2;if(s===1)o.children=t;else if(1<s){for(var u=Array(s),c=0;c<s;c++)u[c]=arguments[c+2];o.children=u}if(e&&e.defaultProps)for(r in s=e.defaultProps,s)o[r]===void 0&&(o[r]=s[r]);return{$$typeof:St,type:e,key:n,ref:l,props:o,_owner:ln.current}}function Gc(e,a){return{$$typeof:St,type:e.type,key:a,ref:e.ref,props:e.props,_owner:e._owner}}function sn(e){return typeof e==="object"&&e!==null&&e.$$typeof===St}function Zc(e){var a={"=":"=0",":":"=2"};return"$"+e.replace(/[=:]/g,function(t){return a[t]})}var bi=/\/+/g;function rn(e,a){return typeof e==="object"&&e!==null&&e.key!=null?Zc(""+e.key):a.toString(36)}function Pr(e,a,t,r,o){var n=typeof e;if(n==="undefined"||n==="boolean")e=null;var l=!1;if(e===null)l=!0;else switch(n){case"string":case"number":l=!0;break;case"object":switch(e.$$typeof){case St:case Lc:l=!0}}if(l)return l=e,o=o(l),e=r===""?"."+rn(l,0):r,Ri(o)?(t="",e!=null&&(t=e.replace(bi,"$&/")+"/"),Pr(o,a,t,"",function(c){return c})):o!=null&&(sn(o)&&(o=Gc(o,t+(!o.key||l&&l.key===o.key?"":(""+o.key).replace(bi,"$&/")+"/")+e)),a.push(o)),1;if(l=0,r=r===""?".":r+":",Ri(e))for(var s=0;s<e.length;s++){n=e[s];var u=r+rn(n,s);l+=Pr(n,a,t,u,o)}else if(u=Xc(e),typeof u==="function")for(e=u.call(e),s=0;!(n=e.next()).done;)n=n.value,u=r+rn(n,s++),l+=Pr(n,a,t,u,o);else if(n==="object")throw a=String(e),Error("Objects are not valid as a React child (found: "+(a==="[object Object]"?"object with keys {"+Object.keys(e).join(", ")+"}":a)+"). If you meant to render a collection of children, use an array instead.");return l}function zr(e,a,t){if(e==null)return e;var r=[],o=0;return Pr(e,r,"","",function(n){return a.call(t,n,o++)}),r}function Jc(e){if(e._status===-1){var a=e._result;a=a(),a.then(function(t){if(e._status===0||e._status===-1)e._status=1,e._result=t},function(t){if(e._status===0||e._status===-1)e._status=2,e._result=t}),e._status===-1&&(e._status=0,e._result=a)}if(e._status===1)return e._result.default;throw e._result}var ie={current:null},_r={transition:null},jc={ReactCurrentDispatcher:ie,ReactCurrentBatchConfig:_r,ReactCurrentOwner:ln};function Qi(){throw Error("act(...) is not supported in production builds of React.")}em.Children={map:zr,forEach:function(e,a,t){zr(e,function(){a.apply(this,arguments)},t)},count:function(e){var a=0;return zr(e,function(){a++}),a},toArray:function(e){return zr(e,function(a){return a})||[]},only:function(e){if(!sn(e))throw Error("React.Children.only expected to receive a single React element child.");return e}};em.Component=Va;em.Fragment=Bc;em.Profiler=Qc;em.PureComponent=on;em.StrictMode=Vc;em.Suspense=Wc;em.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED=jc;em.act=Qi;em.cloneElement=function(e,a,t){if(e===null||e===void 0)throw Error("React.cloneElement(...): The argument must be a React element, but you passed "+e+".");var r=Ii({},e.props),o=e.key,n=e.ref,l=e._owner;if(a!=null){if(a.ref!==void 0&&(n=a.ref,l=ln.current),a.key!==void 0&&(o=""+a.key),e.type&&e.type.defaultProps)var s=e.type.defaultProps;for(u in a)Li.call(a,u)&&!Bi.hasOwnProperty(u)&&(r[u]=a[u]===void 0&&s!==void 0?s[u]:a[u])}var u=arguments.length-2;if(u===1)r.children=t;else if(1<u){s=Array(u);for(var c=0;c<u;c++)s[c]=arguments[c+2];r.children=s}return{$$typeof:St,type:e.type,key:o,ref:n,props:r,_owner:l}};em.createContext=function(e){return e={$$typeof:Hc,_currentValue:e,_currentValue2:e,_threadCount:0,Provider:null,Consumer:null,_defaultValue:null,_globalName:null},e.Provider={$$typeof:Uc,_context:e},e.Consumer=e};em.createElement=Vi;em.createFactory=function(e){var a=Vi.bind(null,e);return a.type=e,a};em.createRef=function(){return{current:null}};em.forwardRef=function(e){return{$$typeof:$c,render:e}};em.isValidElement=sn;em.lazy=function(e){return{$$typeof:Yc,_payload:{_status:-1,_result:e},_init:Jc}};em.memo=function(e,a){return{$$typeof:Kc,type:e,compare:a===void 0?null:a}};em.startTransition=function(e){var a=_r.transition;_r.transition={};try{e()}finally{_r.transition=a}};em.unstable_act=Qi;em.useCallback=function(e,a){return ie.current.useCallback(e,a)};em.useContext=function(e){return ie.current.useContext(e)};em.useDebugValue=function(){};em.useDeferredValue=function(e){return ie.current.useDeferredValue(e)};em.useEffect=function(e,a){return ie.current.useEffect(e,a)};em.useId=function(){return ie.current.useId()};em.useImperativeHandle=function(e,a,t){return ie.current.useImperativeHandle(e,a,t)};em.useInsertionEffect=function(e,a){return ie.current.useInsertionEffect(e,a)};em.useLayoutEffect=function(e,a){return ie.current.useLayoutEffect(e,a)};em.useMemo=function(e,a){return ie.current.useMemo(e,a)};em.useReducer=function(e,a,t){return ie.current.useReducer(e,a,t)};em.useRef=function(e){return ie.current.useRef(e)};em.useState=function(e){return ie.current.useState(e)};em.useSyncExternalStore=function(e,a,t){return ie.current.useSyncExternalStore(e,a,t)};em.useTransition=function(){return ie.current.useTransition()};em.version="18.3.1"});var Xi=Mr((Am)=>{function dn(e,a){var t=e.length;e.push(a);e:for(;0<t;){var r=t-1>>>1,o=e[r];if(0<Tr(o,a))e[r]=a,e[t]=o,t=r;else break e}}function ze(e){return e.length===0?null:e[0]}function Fr(e){if(e.length===0)return null;var a=e[0],t=e.pop();if(t!==a){e[0]=t;e:for(var r=0,o=e.length,n=o>>>1;r<n;){var l=2*(r+1)-1,s=e[l],u=l+1,c=e[u];if(0>Tr(s,t))u<o&&0>Tr(c,s)?(e[r]=c,e[u]=t,r=u):(e[r]=s,e[l]=t,r=l);else if(u<o&&0>Tr(c,t))e[r]=c,e[u]=t,r=u;else break e}}return a}function Tr(e,a){var t=e.sortIndex-a.sortIndex;return t!==0?t:e.id-a.id}if(typeof performance==="object"&&typeof performance.now==="function")cn=performance,Am.unstable_now=function(){return cn.now()};else Or=Date,mn=Or.now(),Am.unstable_now=function(){return Or.now()-mn};var cn,Or,mn,Ie=[],ta=[],Dm=1,Ee=null,te=3,Ir=!1,qa=!1,kt=!1,Hi=typeof setTimeout==="function"?setTimeout:null,$i=typeof clearTimeout==="function"?clearTimeout:null,Ui=typeof setImmediate<"u"?setImmediate:null;typeof navigator<"u"&&navigator.scheduling!==void 0&&navigator.scheduling.isInputPending!==void 0&&navigator.scheduling.isInputPending.bind(navigator.scheduling);function pn(e){for(var a=ze(ta);a!==null;){if(a.callback===null)Fr(ta);else if(a.startTime<=e)Fr(ta),a.sortIndex=a.expirationTime,dn(Ie,a);else break;a=ze(ta)}}function vn(e){if(kt=!1,pn(e),!qa)if(ze(Ie)!==null)qa=!0,gn(hn);else{var a=ze(ta);a!==null&&yn(vn,a.startTime-e)}}function hn(e,a){qa=!1,kt&&(kt=!1,$i(Nt),Nt=-1),Ir=!0;var t=te;try{pn(a);for(Ee=ze(Ie);Ee!==null&&(!(Ee.expirationTime>a)||e&&!Yi());){var r=Ee.callback;if(typeof r==="function"){Ee.callback=null,te=Ee.priorityLevel;var o=r(Ee.expirationTime<=a);a=Am.unstable_now(),typeof o==="function"?Ee.callback=o:Ee===ze(Ie)&&Fr(Ie),pn(a)}else Fr(Ie);Ee=ze(Ie)}if(Ee!==null)var n=!0;else{var l=ze(ta);l!==null&&yn(vn,l.startTime-a),n=!1}return n}finally{Ee=null,te=t,Ir=!1}}var Dr=!1,Rr=null,Nt=-1,Wi=5,Ki=-1;function Yi(){return Am.unstable_now()-Ki<Wi?!1:!0}function un(){if(Rr!==null){var e=Am.unstable_now();Ki=e;var a=!0;try{a=Rr(!0,e)}finally{a?Et():(Dr=!1,Rr=null)}}else Dr=!1}var Et;if(typeof Ui==="function")Et=function(){Ui(un)};else if(typeof MessageChannel<"u")br=new MessageChannel,fn=br.port2,br.port1.onmessage=un,Et=function(){fn.postMessage(null)};else Et=function(){Hi(un,0)};var br,fn;function gn(e){Rr=e,Dr||(Dr=!0,Et())}function yn(e,a){Nt=Hi(function(){e(Am.unstable_now())},a)}Am.unstable_IdlePriority=5;Am.unstable_ImmediatePriority=1;Am.unstable_LowPriority=4;Am.unstable_NormalPriority=3;Am.unstable_Profiling=null;Am.unstable_UserBlockingPriority=2;Am.unstable_cancelCallback=function(e){e.callback=null};Am.unstable_continueExecution=function(){qa||Ir||(qa=!0,gn(hn))};Am.unstable_forceFrameRate=function(e){0>e||125<e?console.error("forceFrameRate takes a positive int between 0 and 125, forcing frame rates higher than 125 fps is not supported"):Wi=0<e?Math.floor(1000/e):5};Am.unstable_getCurrentPriorityLevel=function(){return te};Am.unstable_getFirstCallbackNode=function(){return ze(Ie)};Am.unstable_next=function(e){switch(te){case 1:case 2:case 3:var a=3;break;default:a=te}var t=te;te=a;try{return e()}finally{te=t}};Am.unstable_pauseExecution=function(){};Am.unstable_requestPaint=function(){};Am.unstable_runWithPriority=function(e,a){switch(e){case 1:case 2:case 3:case 4:case 5:break;default:e=3}var t=te;te=e;try{return a()}finally{te=t}};Am.unstable_scheduleCallback=function(e,a,t){var r=Am.unstable_now();switch(typeof t==="object"&&t!==null?(t=t.delay,t=typeof t==="number"&&0<t?r+t:r):t=r,e){case 1:var o=-1;break;case 2:o=250;break;case 5:o=1073741823;break;case 4:o=1e4;break;default:o=5000}return o=t+o,e={id:Dm++,callback:a,priorityLevel:e,startTime:t,expirationTime:o,sortIndex:-1},t>r?(e.sortIndex=t,dn(ta,e),ze(Ie)===null&&e===ze(ta)&&(kt?($i(Nt),Nt=-1):kt=!0,yn(vn,t-r))):(e.sortIndex=o,dn(Ie,e),qa||Ir||(qa=!0,gn(hn))),e};Am.unstable_shouldYield=Yi;Am.unstable_wrapCallback=function(e){var a=te;return function(){var t=te;te=a;try{return e.apply(this,arguments)}finally{te=t}}}});var Ci={};Dc(Ci,{version:()=>uc,unstable_renderSubtreeIntoContainer:()=>sc,unstable_batchedUpdates:()=>ic,unmountComponentAtNode:()=>lc,render:()=>nc,hydrateRoot:()=>oc,hydrate:()=>rc,flushSync:()=>tc,findDOMNode:()=>ac,createRoot:()=>ec,createPortal:()=>jd,__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED:()=>Jd});function S(e){for(var a="https://reactjs.org/docs/error-decoder.html?invariant="+e,t=1;t<arguments.length;t++)a+="&args[]="+encodeURIComponent(arguments[t]);return"Minified React error #"+e+"; visit "+a+" for the full message or use the non-minified dev environment for full errors and additional helpful warnings."}function La(e,a){ut(e,a),ut(e+"Capture",a)}function ut(e,a){Kt[e]=a;for(e=0;e<a.length;e++)au.add(a[e])}function op(e){if(Bn.call(Zi,e))return!0;if(Bn.call(Gi,e))return!1;if(rp.test(e))return Zi[e]=!0;return Gi[e]=!0,!1}function np(e,a,t,r){if(t!==null&&t.type===0)return!1;switch(typeof a){case"function":case"symbol":return!0;case"boolean":if(r)return!1;if(t!==null)return!t.acceptsBooleans;return e=e.toLowerCase().slice(0,5),e!=="data-"&&e!=="aria-";default:return!1}}function lp(e,a,t,r){if(a===null||typeof a>"u"||np(e,a,t,r))return!0;if(r)return!1;if(t!==null)switch(t.type){case 3:return!a;case 4:return a===!1;case 5:return isNaN(a);case 6:return isNaN(a)||1>a}return!1}function de(e,a,t,r,o,n,l){this.acceptsBooleans=a===2||a===3||a===4,this.attributeName=r,this.attributeNamespace=o,this.mustUseProperty=t,this.propertyName=e,this.type=a,this.sanitizeURL=n,this.removeEmptyString=l}function Fl(e){return e[1].toUpperCase()}function Il(e,a,t,r){var o=ee.hasOwnProperty(a)?ee[a]:null;if(o!==null?o.type!==0:r||!(2<a.length)||a[0]!=="o"&&a[0]!=="O"||a[1]!=="n"&&a[1]!=="N")lp(a,t,o,r)&&(t=null),r||o===null?op(a)&&(t===null?e.removeAttribute(a):e.setAttribute(a,""+t)):o.mustUseProperty?e[o.propertyName]=t===null?o.type===3?!1:"":t:(a=o.attributeName,r=o.attributeNamespace,t===null?e.removeAttribute(a):(o=o.type,t=o===3||o===4&&t===!0?"":""+t,r?e.setAttributeNS(r,a,t):e.setAttribute(a,t)))}function wt(e){if(e===null||typeof e!=="object")return null;return e=Ji&&e[Ji]||e["@@iterator"],typeof e==="function"?e:null}function _t(e){if(Sn===void 0)try{throw Error()}catch(t){var a=t.stack.trim().match(/\n( *(at )?)/);Sn=a&&a[1]||""}return`
`+Sn+e}function kn(e,a){if(!e||En)return"";En=!0;var t=Error.prepareStackTrace;Error.prepareStackTrace=void 0;try{if(a)if(a=function(){throw Error()},Object.defineProperty(a.prototype,"props",{set:function(){throw Error()}}),typeof Reflect==="object"&&Reflect.construct){try{Reflect.construct(a,[])}catch(c){var r=c}Reflect.construct(e,[],a)}else{try{a.call()}catch(c){r=c}e.call(a.prototype)}else{try{throw Error()}catch(c){r=c}e()}}catch(c){if(c&&r&&typeof c.stack==="string"){for(var o=c.stack.split(`
`),n=r.stack.split(`
`),l=o.length-1,s=n.length-1;1<=l&&0<=s&&o[l]!==n[s];)s--;for(;1<=l&&0<=s;l--,s--)if(o[l]!==n[s]){if(l!==1||s!==1)do if(l--,s--,0>s||o[l]!==n[s]){var u=`
`+o[l].replace(" at new "," at ");return e.displayName&&u.includes("<anonymous>")&&(u=u.replace("<anonymous>",e.displayName)),u}while(1<=l&&0<=s);break}}}finally{En=!1,Error.prepareStackTrace=t}return(e=e?e.displayName||e.name:"")?_t(e):""}function ip(e){switch(e.tag){case 5:return _t(e.type);case 16:return _t("Lazy");case 13:return _t("Suspense");case 19:return _t("SuspenseList");case 0:case 2:case 15:return e=kn(e.type,!1),e;case 11:return e=kn(e.type.render,!1),e;case 1:return e=kn(e.type,!0),e;default:return""}}function Hn(e){if(e==null)return null;if(typeof e==="function")return e.displayName||e.name||null;if(typeof e==="string")return e;switch(e){case Wa:return"Fragment";case $a:return"Portal";case Vn:return"Profiler";case Dl:return"StrictMode";case Qn:return"Suspense";case Un:return"SuspenseList"}if(typeof e==="object")switch(e.$$typeof){case ru:return(e.displayName||"Context")+".Consumer";case tu:return(e._context.displayName||"Context")+".Provider";case Al:var a=e.render;return e=e.displayName,e||(e=a.displayName||a.name||"",e=e!==""?"ForwardRef("+e+")":"ForwardRef"),e;case Ll:return a=e.displayName||null,a!==null?a:Hn(e.type)||"Memo";case oa:a=e._payload,e=e._init;try{return Hn(e(a))}catch(t){}}return null}function sp(e){var a=e.type;switch(e.tag){case 24:return"Cache";case 9:return(a.displayName||"Context")+".Consumer";case 10:return(a._context.displayName||"Context")+".Provider";case 18:return"DehydratedFragment";case 11:return e=a.render,e=e.displayName||e.name||"",a.displayName||(e!==""?"ForwardRef("+e+")":"ForwardRef");case 7:return"Fragment";case 5:return a;case 4:return"Portal";case 3:return"Root";case 6:return"Text";case 16:return Hn(a);case 8:return a===Dl?"StrictMode":"Mode";case 22:return"Offscreen";case 12:return"Profiler";case 21:return"Scope";case 13:return"Suspense";case 19:return"SuspenseList";case 25:return"TracingMarker";case 1:case 0:case 17:case 2:case 14:case 15:if(typeof a==="function")return a.displayName||a.name||null;if(typeof a==="string")return a}return null}function ya(e){switch(typeof e){case"boolean":case"number":case"string":case"undefined":return e;case"object":return e;default:return""}}function nu(e){var a=e.type;return(e=e.nodeName)&&e.toLowerCase()==="input"&&(a==="checkbox"||a==="radio")}function up(e){var a=nu(e)?"checked":"value",t=Object.getOwnPropertyDescriptor(e.constructor.prototype,a),r=""+e[a];if(!e.hasOwnProperty(a)&&typeof t<"u"&&typeof t.get==="function"&&typeof t.set==="function"){var{get:o,set:n}=t;return Object.defineProperty(e,a,{configurable:!0,get:function(){return o.call(this)},set:function(l){r=""+l,n.call(this,l)}}),Object.defineProperty(e,a,{enumerable:t.enumerable}),{getValue:function(){return r},setValue:function(l){r=""+l},stopTracking:function(){e._valueTracker=null,delete e[a]}}}}function Lr(e){e._valueTracker||(e._valueTracker=up(e))}function lu(e){if(!e)return!1;var a=e._valueTracker;if(!a)return!0;var t=a.getValue(),r="";return e&&(r=nu(e)?e.checked?"true":"false":e.value),e=r,e!==t?(a.setValue(e),!0):!1}function co(e){if(e=e||(typeof document<"u"?document:void 0),typeof e>"u")return null;try{return e.activeElement||e.body}catch(a){return e.body}}function $n(e,a){var t=a.checked;return U({},a,{defaultChecked:void 0,defaultValue:void 0,value:void 0,checked:t!=null?t:e._wrapperState.initialChecked})}function ji(e,a){var t=a.defaultValue==null?"":a.defaultValue,r=a.checked!=null?a.checked:a.defaultChecked;t=ya(a.value!=null?a.value:t),e._wrapperState={initialChecked:r,initialValue:t,controlled:a.type==="checkbox"||a.type==="radio"?a.checked!=null:a.value!=null}}function iu(e,a){a=a.checked,a!=null&&Il(e,"checked",a,!1)}function Wn(e,a){iu(e,a);var t=ya(a.value),r=a.type;if(t!=null)if(r==="number"){if(t===0&&e.value===""||e.value!=t)e.value=""+t}else e.value!==""+t&&(e.value=""+t);else if(r==="submit"||r==="reset"){e.removeAttribute("value");return}a.hasOwnProperty("value")?Kn(e,a.type,t):a.hasOwnProperty("defaultValue")&&Kn(e,a.type,ya(a.defaultValue)),a.checked==null&&a.defaultChecked!=null&&(e.defaultChecked=!!a.defaultChecked)}function es(e,a,t){if(a.hasOwnProperty("value")||a.hasOwnProperty("defaultValue")){var r=a.type;if(!(r!=="submit"&&r!=="reset"||a.value!==void 0&&a.value!==null))return;a=""+e._wrapperState.initialValue,t||a===e.value||(e.value=a),e.defaultValue=a}t=e.name,t!==""&&(e.name=""),e.defaultChecked=!!e._wrapperState.initialChecked,t!==""&&(e.name=t)}function Kn(e,a,t){if(a!=="number"||co(e.ownerDocument)!==e)t==null?e.defaultValue=""+e._wrapperState.initialValue:e.defaultValue!==""+t&&(e.defaultValue=""+t)}function rt(e,a,t,r){if(e=e.options,a){a={};for(var o=0;o<t.length;o++)a["$"+t[o]]=!0;for(t=0;t<e.length;t++)o=a.hasOwnProperty("$"+e[t].value),e[t].selected!==o&&(e[t].selected=o),o&&r&&(e[t].defaultSelected=!0)}else{t=""+ya(t),a=null;for(o=0;o<e.length;o++){if(e[o].value===t){e[o].selected=!0,r&&(e[o].defaultSelected=!0);return}a!==null||e[o].disabled||(a=e[o])}a!==null&&(a.selected=!0)}}function Yn(e,a){if(a.dangerouslySetInnerHTML!=null)throw Error(S(91));return U({},a,{value:void 0,defaultValue:void 0,children:""+e._wrapperState.initialValue})}function as(e,a){var t=a.value;if(t==null){if(t=a.children,a=a.defaultValue,t!=null){if(a!=null)throw Error(S(92));if(Tt(t)){if(1<t.length)throw Error(S(93));t=t[0]}a=t}a==null&&(a=""),t=a}e._wrapperState={initialValue:ya(t)}}function su(e,a){var t=ya(a.value),r=ya(a.defaultValue);t!=null&&(t=""+t,t!==e.value&&(e.value=t),a.defaultValue==null&&e.defaultValue!==t&&(e.defaultValue=t)),r!=null&&(e.defaultValue=""+r)}function ts(e){var a=e.textContent;a===e._wrapperState.initialValue&&a!==""&&a!==null&&(e.value=a)}function uu(e){switch(e){case"svg":return"http://www.w3.org/2000/svg";case"math":return"http://www.w3.org/1998/Math/MathML";default:return"http://www.w3.org/1999/xhtml"}}function Xn(e,a){return e==null||e==="http://www.w3.org/1999/xhtml"?uu(a):e==="http://www.w3.org/2000/svg"&&a==="foreignObject"?"http://www.w3.org/1999/xhtml":e}function Yt(e,a){if(a){var t=e.firstChild;if(t&&t===e.lastChild&&t.nodeType===3){t.nodeValue=a;return}}e.textContent=a}function cu(e,a,t){return a==null||typeof a==="boolean"||a===""?"":t||typeof a!=="number"||a===0||At.hasOwnProperty(e)&&At[e]?(""+a).trim():a+"px"}function mu(e,a){e=e.style;for(var t in a)if(a.hasOwnProperty(t)){var r=t.indexOf("--")===0,o=cu(t,a[t],r);t==="float"&&(t="cssFloat"),r?e.setProperty(t,o):e[t]=o}}function Gn(e,a){if(a){if(cp[e]&&(a.children!=null||a.dangerouslySetInnerHTML!=null))throw Error(S(137,e));if(a.dangerouslySetInnerHTML!=null){if(a.children!=null)throw Error(S(60));if(typeof a.dangerouslySetInnerHTML!=="object"||!("__html"in a.dangerouslySetInnerHTML))throw Error(S(61))}if(a.style!=null&&typeof a.style!=="object")throw Error(S(62))}}function Zn(e,a){if(e.indexOf("-")===-1)return typeof a.is==="string";switch(e){case"annotation-xml":case"color-profile":case"font-face":case"font-face-src":case"font-face-uri":case"font-face-format":case"font-face-name":case"missing-glyph":return!1;default:return!0}}function Bl(e){return e=e.target||e.srcElement||window,e.correspondingUseElement&&(e=e.correspondingUseElement),e.nodeType===3?e.parentNode:e}function rs(e){if(e=mr(e)){if(typeof jn!=="function")throw Error(S(280));var a=e.stateNode;a&&(a=Lo(a),jn(e.stateNode,e.type,a))}}function pu(e){ot?nt?nt.push(e):nt=[e]:ot=e}function fu(){if(ot){var e=ot,a=nt;if(nt=ot=null,rs(e),a)for(e=0;e<a.length;e++)rs(a[e])}}function vu(e,a){return e(a)}function hu(){}function gu(e,a,t){if(Nn)return e(a,t);Nn=!0;try{return vu(e,a,t)}finally{if(Nn=!1,ot!==null||nt!==null)hu(),fu()}}function Xt(e,a){var t=e.stateNode;if(t===null)return null;var r=Lo(t);if(r===null)return null;t=r[a];e:switch(a){case"onClick":case"onClickCapture":case"onDoubleClick":case"onDoubleClickCapture":case"onMouseDown":case"onMouseDownCapture":case"onMouseMove":case"onMouseMoveCapture":case"onMouseUp":case"onMouseUpCapture":case"onMouseEnter":(r=!r.disabled)||(e=e.type,r=!(e==="button"||e==="input"||e==="select"||e==="textarea")),e=!r;break e;default:e=!1}if(e)return null;if(t&&typeof t!=="function")throw Error(S(231,a,typeof t));return t}function mp(e,a,t,r,o,n,l,s,u){var c=Array.prototype.slice.call(arguments,3);try{a.apply(t,c)}catch(h){this.onError(h)}}function fp(e,a,t,r,o,n,l,s,u){Lt=!1,mo=null,mp.apply(pp,arguments)}function vp(e,a,t,r,o,n,l,s,u){if(fp.apply(this,arguments),Lt){if(Lt){var c=mo;Lt=!1,mo=null}else throw Error(S(198));po||(po=!0,al=c)}}function Ba(e){var a=e,t=e;if(e.alternate)for(;a.return;)a=a.return;else{e=a;do a=e,(a.flags&4098)!==0&&(t=a.return),e=a.return;while(e)}return a.tag===3?t:null}function yu(e){if(e.tag===13){var a=e.memoizedState;if(a===null&&(e=e.alternate,e!==null&&(a=e.memoizedState)),a!==null)return a.dehydrated}return null}function os(e){if(Ba(e)!==e)throw Error(S(188))}function hp(e){var a=e.alternate;if(!a){if(a=Ba(e),a===null)throw Error(S(188));return a!==e?null:e}for(var t=e,r=a;;){var o=t.return;if(o===null)break;var n=o.alternate;if(n===null){if(r=o.return,r!==null){t=r;continue}break}if(o.child===n.child){for(n=o.child;n;){if(n===t)return os(o),e;if(n===r)return os(o),a;n=n.sibling}throw Error(S(188))}if(t.return!==r.return)t=o,r=n;else{for(var l=!1,s=o.child;s;){if(s===t){l=!0,t=o,r=n;break}if(s===r){l=!0,r=o,t=n;break}s=s.sibling}if(!l){for(s=n.child;s;){if(s===t){l=!0,t=n,r=o;break}if(s===r){l=!0,r=n,t=o;break}s=s.sibling}if(!l)throw Error(S(189))}}if(t.alternate!==r)throw Error(S(190))}if(t.tag!==3)throw Error(S(188));return t.stateNode.current===t?e:a}function Su(e){return e=hp(e),e!==null?Eu(e):null}function Eu(e){if(e.tag===5||e.tag===6)return e;for(e=e.child;e!==null;){var a=Eu(e);if(a!==null)return a;e=e.sibling}return null}function kp(e){if(Be&&typeof Be.onCommitFiberRoot==="function")try{Be.onCommitFiberRoot(Fo,e,void 0,(e.current.flags&128)===128)}catch(a){}}function Cp(e){return e>>>=0,e===0?32:31-(Np(e)/wp|0)|0}function Ot(e){switch(e&-e){case 1:return 1;case 2:return 2;case 4:return 4;case 8:return 8;case 16:return 16;case 32:return 32;case 64:case 128:case 256:case 512:case 1024:case 2048:case 4096:case 8192:case 16384:case 32768:case 65536:case 131072:case 262144:case 524288:case 1048576:case 2097152:return e&4194240;case 4194304:case 8388608:case 16777216:case 33554432:case 67108864:return e&130023424;case 134217728:return 134217728;case 268435456:return 268435456;case 536870912:return 536870912;case 1073741824:return 1073741824;default:return e}}function vo(e,a){var t=e.pendingLanes;if(t===0)return 0;var r=0,o=e.suspendedLanes,n=e.pingedLanes,l=t&268435455;if(l!==0){var s=l&~o;s!==0?r=Ot(s):(n&=l,n!==0&&(r=Ot(n)))}else l=t&~o,l!==0?r=Ot(l):n!==0&&(r=Ot(n));if(r===0)return 0;if(a!==0&&a!==r&&(a&o)===0&&(o=r&-r,n=a&-a,o>=n||o===16&&(n&4194240)!==0))return a;if((r&4)!==0&&(r|=t&16),a=e.entangledLanes,a!==0)for(e=e.entanglements,a&=r;0<a;)t=31-Re(a),o=1<<t,r|=e[t],a&=~o;return r}function qp(e,a){switch(e){case 1:case 2:case 4:return a+250;case 8:case 16:case 32:case 64:case 128:case 256:case 512:case 1024:case 2048:case 4096:case 8192:case 16384:case 32768:case 65536:case 131072:case 262144:case 524288:case 1048576:case 2097152:return a+5000;case 4194304:case 8388608:case 16777216:case 33554432:case 67108864:return-1;case 134217728:case 268435456:case 536870912:case 1073741824:return-1;default:return-1}}function xp(e,a){for(var{suspendedLanes:t,pingedLanes:r,expirationTimes:o,pendingLanes:n}=e;0<n;){var l=31-Re(n),s=1<<l,u=o[l];if(u===-1){if((s&t)===0||(s&r)!==0)o[l]=qp(s,a)}else u<=a&&(e.expiredLanes|=s);n&=~s}}function tl(e){return e=e.pendingLanes&-1073741825,e!==0?e:e&1073741824?1073741824:0}function Cu(){var e=Vr;return Vr<<=1,(Vr&4194240)===0&&(Vr=64),e}function wn(e){for(var a=[],t=0;31>t;t++)a.push(e);return a}function dr(e,a,t){e.pendingLanes|=a,a!==536870912&&(e.suspendedLanes=0,e.pingedLanes=0),e=e.eventTimes,a=31-Re(a),e[a]=t}function Mp(e,a){var t=e.pendingLanes&~a;e.pendingLanes=a,e.suspendedLanes=0,e.pingedLanes=0,e.expiredLanes&=a,e.mutableReadLanes&=a,e.entangledLanes&=a,a=e.entanglements;var r=e.eventTimes;for(e=e.expirationTimes;0<t;){var o=31-Re(t),n=1<<o;a[o]=0,r[o]=-1,e[o]=-1,t&=~n}}function Ql(e,a){var t=e.entangledLanes|=a;for(e=e.entanglements;t;){var r=31-Re(t),o=1<<r;o&a|e[r]&a&&(e[r]|=a),t&=~o}}function qu(e){return e&=-e,1<e?4<e?(e&268435455)!==0?16:536870912:4:1}function ls(e,a){switch(e){case"focusin":case"focusout":da=null;break;case"dragenter":case"dragleave":ca=null;break;case"mouseover":case"mouseout":ma=null;break;case"pointerover":case"pointerout":Gt.delete(a.pointerId);break;case"gotpointercapture":case"lostpointercapture":Zt.delete(a.pointerId)}}function Ct(e,a,t,r,o,n){if(e===null||e.nativeEvent!==n)return e={blockedOn:a,domEventName:t,eventSystemFlags:r,nativeEvent:n,targetContainers:[o]},a!==null&&(a=mr(a),a!==null&&Ul(a)),e;return e.eventSystemFlags|=r,a=e.targetContainers,o!==null&&a.indexOf(o)===-1&&a.push(o),e}function Pp(e,a,t,r,o){switch(a){case"focusin":return da=Ct(da,e,a,t,r,o),!0;case"dragenter":return ca=Ct(ca,e,a,t,r,o),!0;case"mouseover":return ma=Ct(ma,e,a,t,r,o),!0;case"pointerover":var n=o.pointerId;return Gt.set(n,Ct(Gt.get(n)||null,e,a,t,r,o)),!0;case"gotpointercapture":return n=o.pointerId,Zt.set(n,Ct(Zt.get(n)||null,e,a,t,r,o)),!0}return!1}function _u(e){var a=Pa(e.target);if(a!==null){var t=Ba(a);if(t!==null){if(a=t.tag,a===13){if(a=yu(t),a!==null){e.blockedOn=a,Pu(e.priority,function(){Mu(t)});return}}else if(a===3&&t.stateNode.current.memoizedState.isDehydrated){e.blockedOn=t.tag===3?t.stateNode.containerInfo:null;return}}}e.blockedOn=null}function jr(e){if(e.blockedOn!==null)return!1;for(var a=e.targetContainers;0<a.length;){var t=ol(e.domEventName,e.eventSystemFlags,a[0],e.nativeEvent);if(t===null){t=e.nativeEvent;var r=new t.constructor(t.type,t);Jn=r,t.target.dispatchEvent(r),Jn=null}else return a=mr(t),a!==null&&Ul(a),e.blockedOn=t,!1;a.shift()}return!0}function is(e,a,t){jr(e)&&t.delete(a)}function _p(){rl=!1,da!==null&&jr(da)&&(da=null),ca!==null&&jr(ca)&&(ca=null),ma!==null&&jr(ma)&&(ma=null),Gt.forEach(is),Zt.forEach(is)}function qt(e,a){e.blockedOn===a&&(e.blockedOn=null,rl||(rl=!0,B.unstable_scheduleCallback(B.unstable_NormalPriority,_p)))}function Jt(e){function a(o){return qt(o,e)}if(0<Ur.length){qt(Ur[0],e);for(var t=1;t<Ur.length;t++){var r=Ur[t];r.blockedOn===e&&(r.blockedOn=null)}}da!==null&&qt(da,e),ca!==null&&qt(ca,e),ma!==null&&qt(ma,e),Gt.forEach(a),Zt.forEach(a);for(t=0;t<la.length;t++)r=la[t],r.blockedOn===e&&(r.blockedOn=null);for(;0<la.length&&(t=la[0],t.blockedOn===null);)_u(t),t.blockedOn===null&&la.shift()}function Tp(e,a,t,r){var o=R,n=lt.transition;lt.transition=null;try{R=1,Hl(e,a,t,r)}finally{R=o,lt.transition=n}}function Op(e,a,t,r){var o=R,n=lt.transition;lt.transition=null;try{R=4,Hl(e,a,t,r)}finally{R=o,lt.transition=n}}function Hl(e,a,t,r){if(ho){var o=ol(e,a,t,r);if(o===null)Pn(e,a,r,go,t),ls(e,r);else if(Pp(o,e,a,t,r))r.stopPropagation();else if(ls(e,r),a&4&&-1<zp.indexOf(e)){for(;o!==null;){var n=mr(o);if(n!==null&&xu(n),n=ol(e,a,t,r),n===null&&Pn(e,a,r,go,t),n===o)break;o=n}o!==null&&r.stopPropagation()}else Pn(e,a,r,null,t)}}function ol(e,a,t,r){if(go=null,e=Bl(r),e=Pa(e),e!==null)if(a=Ba(e),a===null)e=null;else if(t=a.tag,t===13){if(e=yu(a),e!==null)return e;e=null}else if(t===3){if(a.stateNode.current.memoizedState.isDehydrated)return a.tag===3?a.stateNode.containerInfo:null;e=null}else a!==e&&(e=null);return go=e,null}function Tu(e){switch(e){case"cancel":case"click":case"close":case"contextmenu":case"copy":case"cut":case"auxclick":case"dblclick":case"dragend":case"dragstart":case"drop":case"focusin":case"focusout":case"input":case"invalid":case"keydown":case"keypress":case"keyup":case"mousedown":case"mouseup":case"paste":case"pause":case"play":case"pointercancel":case"pointerdown":case"pointerup":case"ratechange":case"reset":case"resize":case"seeked":case"submit":case"touchcancel":case"touchend":case"touchstart":case"volumechange":case"change":case"selectionchange":case"textInput":case"compositionstart":case"compositionend":case"compositionupdate":case"beforeblur":case"afterblur":case"beforeinput":case"blur":case"fullscreenchange":case"focus":case"hashchange":case"popstate":case"select":case"selectstart":return 1;case"drag":case"dragenter":case"dragexit":case"dragleave":case"dragover":case"mousemove":case"mouseout":case"mouseover":case"pointermove":case"pointerout":case"pointerover":case"scroll":case"toggle":case"touchmove":case"wheel":case"mouseenter":case"mouseleave":case"pointerenter":case"pointerleave":return 4;case"message":switch(Sp()){case Vl:return 1;case Nu:return 4;case fo:case Ep:return 16;case wu:return 536870912;default:return 16}default:return 16}}function Ou(){if(eo)return eo;var e,a=$l,t=a.length,r,o="value"in sa?sa.value:sa.textContent,n=o.length;for(e=0;e<t&&a[e]===o[e];e++);var l=t-e;for(r=1;r<=l&&a[t-r]===o[n-r];r++);return eo=o.slice(e,1<r?1-r:void 0)}function ao(e){var a=e.keyCode;return"charCode"in e?(e=e.charCode,e===0&&a===13&&(e=13)):e=a,e===10&&(e=13),32<=e||e===13?e:0}function Hr(){return!0}function ss(){return!1}function Se(e){function a(t,r,o,n,l){this._reactName=t,this._targetInst=o,this.type=r,this.nativeEvent=n,this.target=l,this.currentTarget=null;for(var s in e)e.hasOwnProperty(s)&&(t=e[s],this[s]=t?t(n):n[s]);return this.isDefaultPrevented=(n.defaultPrevented!=null?n.defaultPrevented:n.returnValue===!1)?Hr:ss,this.isPropagationStopped=ss,this}return U(a.prototype,{preventDefault:function(){this.defaultPrevented=!0;var t=this.nativeEvent;t&&(t.preventDefault?t.preventDefault():typeof t.returnValue!=="unknown"&&(t.returnValue=!1),this.isDefaultPrevented=Hr)},stopPropagation:function(){var t=this.nativeEvent;t&&(t.stopPropagation?t.stopPropagation():typeof t.cancelBubble!=="unknown"&&(t.cancelBubble=!0),this.isPropagationStopped=Hr)},persist:function(){},isPersistent:Hr}),a}function $p(e){var a=this.nativeEvent;return a.getModifierState?a.getModifierState(e):(e=Hp[e])?!!a[e]:!1}function Kl(){return $p}function bu(e,a){switch(e){case"keyup":return af.indexOf(a.keyCode)!==-1;case"keydown":return a.keyCode!==229;case"keypress":case"mousedown":case"focusout":return!0;default:return!1}}function Fu(e){return e=e.detail,typeof e==="object"&&"data"in e?e.data:null}function rf(e,a){switch(e){case"compositionend":return Fu(a);case"keypress":if(a.which!==32)return null;return ps=!0,ms;case"textInput":return e=a.data,e===ms&&ps?null:e;default:return null}}function of(e,a){if(Ka)return e==="compositionend"||!Yl&&bu(e,a)?(e=Ou(),eo=$l=sa=null,Ka=!1,e):null;switch(e){case"paste":return null;case"keypress":if(!(a.ctrlKey||a.altKey||a.metaKey)||a.ctrlKey&&a.altKey){if(a.char&&1<a.char.length)return a.char;if(a.which)return String.fromCharCode(a.which)}return null;case"compositionend":return Ru&&a.locale!=="ko"?null:a.data;default:return null}}function fs(e){var a=e&&e.nodeName&&e.nodeName.toLowerCase();return a==="input"?!!nf[e.type]:a==="textarea"?!0:!1}function Iu(e,a,t,r){pu(r),a=yo(a,"onChange"),0<a.length&&(t=new Wl("onChange","change",null,t,r),e.push({event:t,listeners:a}))}function lf(e){Ku(e,0)}function Do(e){var a=Ga(e);if(lu(a))return e}function sf(e,a){if(e==="change")return a}function vs(){Vt&&(Vt.detachEvent("onpropertychange",Au),jt=Vt=null)}function Au(e){if(e.propertyName==="value"&&Do(jt)){var a=[];Iu(a,jt,e,Bl(e)),gu(lf,a)}}function uf(e,a,t){e==="focusin"?(vs(),Vt=a,jt=t,Vt.attachEvent("onpropertychange",Au)):e==="focusout"&&vs()}function df(e){if(e==="selectionchange"||e==="keyup"||e==="keydown")return Do(jt)}function cf(e,a){if(e==="click")return Do(a)}function mf(e,a){if(e==="input"||e==="change")return Do(a)}function pf(e,a){return e===a&&(e!==0||1/e===1/a)||e!==e&&a!==a}function er(e,a){if(Fe(e,a))return!0;if(typeof e!=="object"||e===null||typeof a!=="object"||a===null)return!1;var t=Object.keys(e),r=Object.keys(a);if(t.length!==r.length)return!1;for(r=0;r<t.length;r++){var o=t[r];if(!Bn.call(a,o)||!Fe(e[o],a[o]))return!1}return!0}function hs(e){for(;e&&e.firstChild;)e=e.firstChild;return e}function gs(e,a){var t=hs(e);e=0;for(var r;t;){if(t.nodeType===3){if(r=e+t.textContent.length,e<=a&&r>=a)return{node:t,offset:a-e};e=r}e:{for(;t;){if(t.nextSibling){t=t.nextSibling;break e}t=t.parentNode}t=void 0}t=hs(t)}}function Lu(e,a){return e&&a?e===a?!0:e&&e.nodeType===3?!1:a&&a.nodeType===3?Lu(e,a.parentNode):("contains"in e)?e.contains(a):e.compareDocumentPosition?!!(e.compareDocumentPosition(a)&16):!1:!1}function Bu(){for(var e=window,a=co();a instanceof e.HTMLIFrameElement;){try{var t=typeof a.contentWindow.location.href==="string"}catch(r){t=!1}if(t)e=a.contentWindow;else break;a=co(e.document)}return a}function Xl(e){var a=e&&e.nodeName&&e.nodeName.toLowerCase();return a&&(a==="input"&&(e.type==="text"||e.type==="search"||e.type==="tel"||e.type==="url"||e.type==="password")||a==="textarea"||e.contentEditable==="true")}function ff(e){var a=Bu(),t=e.focusedElem,r=e.selectionRange;if(a!==t&&t&&t.ownerDocument&&Lu(t.ownerDocument.documentElement,t)){if(r!==null&&Xl(t)){if(a=r.start,e=r.end,e===void 0&&(e=a),"selectionStart"in t)t.selectionStart=a,t.selectionEnd=Math.min(e,t.value.length);else if(e=(a=t.ownerDocument||document)&&a.defaultView||window,e.getSelection){e=e.getSelection();var o=t.textContent.length,n=Math.min(r.start,o);r=r.end===void 0?n:Math.min(r.end,o),!e.extend&&n>r&&(o=r,r=n,n=o),o=gs(t,n);var l=gs(t,r);o&&l&&(e.rangeCount!==1||e.anchorNode!==o.node||e.anchorOffset!==o.offset||e.focusNode!==l.node||e.focusOffset!==l.offset)&&(a=a.createRange(),a.setStart(o.node,o.offset),e.removeAllRanges(),n>r?(e.addRange(a),e.extend(l.node,l.offset)):(a.setEnd(l.node,l.offset),e.addRange(a)))}}a=[];for(e=t;e=e.parentNode;)e.nodeType===1&&a.push({element:e,left:e.scrollLeft,top:e.scrollTop});typeof t.focus==="function"&&t.focus();for(t=0;t<a.length;t++)e=a[t],e.element.scrollLeft=e.left,e.element.scrollTop=e.top}}function ys(e,a,t){var r=t.window===t?t.document:t.nodeType===9?t:t.ownerDocument;ll||Ya==null||Ya!==co(r)||(r=Ya,("selectionStart"in r)&&Xl(r)?r={start:r.selectionStart,end:r.selectionEnd}:(r=(r.ownerDocument&&r.ownerDocument.defaultView||window).getSelection(),r={anchorNode:r.anchorNode,anchorOffset:r.anchorOffset,focusNode:r.focusNode,focusOffset:r.focusOffset}),Qt&&er(Qt,r)||(Qt=r,r=yo(nl,"onSelect"),0<r.length&&(a=new Wl("onSelect","select",null,a,t),e.push({event:a,listeners:r}),a.target=Ya)))}function $r(e,a){var t={};return t[e.toLowerCase()]=a.toLowerCase(),t["Webkit"+e]="webkit"+a,t["Moz"+e]="moz"+a,t}function Ao(e){if(Mn[e])return Mn[e];if(!Xa[e])return e;var a=Xa[e],t;for(t in a)if(a.hasOwnProperty(t)&&t in Vu)return Mn[e]=a[t];return e}function Ea(e,a){Wu.set(e,a),La(a,[e])}function Es(e,a,t){var r=e.type||"unknown-event";e.currentTarget=t,vp(r,a,void 0,e),e.currentTarget=null}function Ku(e,a){a=(a&4)!==0;for(var t=0;t<e.length;t++){var r=e[t],o=r.event;r=r.listeners;e:{var n=void 0;if(a)for(var l=r.length-1;0<=l;l--){var s=r[l],u=s.instance,c=s.currentTarget;if(s=s.listener,u!==n&&o.isPropagationStopped())break e;Es(o,s,c),n=u}else for(l=0;l<r.length;l++){if(s=r[l],u=s.instance,c=s.currentTarget,s=s.listener,u!==n&&o.isPropagationStopped())break e;Es(o,s,c),n=u}}}if(po)throw e=al,po=!1,al=null,e}function D(e,a){var t=a[pl];t===void 0&&(t=a[pl]=new Set);var r=e+"__bubble";t.has(r)||(Yu(a,e,2,!1),t.add(r))}function zn(e,a,t){var r=0;a&&(r|=4),Yu(t,e,r,a)}function ar(e){if(!e[Wr]){e[Wr]=!0,au.forEach(function(t){t!=="selectionchange"&&(hf.has(t)||zn(t,!1,e),zn(t,!0,e))});var a=e.nodeType===9?e:e.ownerDocument;a===null||a[Wr]||(a[Wr]=!0,zn("selectionchange",!1,a))}}function Yu(e,a,t,r){switch(Tu(a)){case 1:var o=Tp;break;case 4:o=Op;break;default:o=Hl}t=o.bind(null,a,t,e),o=void 0,!el||a!=="touchstart"&&a!=="touchmove"&&a!=="wheel"||(o=!0),r?o!==void 0?e.addEventListener(a,t,{capture:!0,passive:o}):e.addEventListener(a,t,!0):o!==void 0?e.addEventListener(a,t,{passive:o}):e.addEventListener(a,t,!1)}function Pn(e,a,t,r,o){var n=r;if((a&1)===0&&(a&2)===0&&r!==null)e:for(;;){if(r===null)return;var l=r.tag;if(l===3||l===4){var s=r.stateNode.containerInfo;if(s===o||s.nodeType===8&&s.parentNode===o)break;if(l===4)for(l=r.return;l!==null;){var u=l.tag;if(u===3||u===4){if(u=l.stateNode.containerInfo,u===o||u.nodeType===8&&u.parentNode===o)return}l=l.return}for(;s!==null;){if(l=Pa(s),l===null)return;if(u=l.tag,u===5||u===6){r=n=l;continue e}s=s.parentNode}}r=r.return}gu(function(){var c=n,h=Bl(t),y=[];e:{var g=Wu.get(e);if(g!==void 0){var k=Wl,N=e;switch(e){case"keypress":if(ao(t)===0)break e;case"keydown":case"keyup":k=Kp;break;case"focusin":N="focus",k=xn;break;case"focusout":N="blur",k=xn;break;case"beforeblur":case"afterblur":k=xn;break;case"click":if(t.button===2)break e;case"auxclick":case"dblclick":case"mousedown":case"mousemove":case"mouseup":case"mouseout":case"mouseover":case"contextmenu":k=us;break;case"drag":case"dragend":case"dragenter":case"dragexit":case"dragleave":case"dragover":case"dragstart":case"drop":k=Fp;break;case"touchcancel":case"touchend":case"touchmove":case"touchstart":k=Gp;break;case Qu:case Uu:case Hu:k=Ap;break;case $u:k=Jp;break;case"scroll":k=Rp;break;case"wheel":k=ef;break;case"copy":case"cut":case"paste":k=Bp;break;case"gotpointercapture":case"lostpointercapture":case"pointercancel":case"pointerdown":case"pointermove":case"pointerout":case"pointerover":case"pointerup":k=cs}var C=(a&4)!==0,b=!C&&e==="scroll",m=C?g!==null?g+"Capture":null:g;C=[];for(var d=c,f;d!==null;){f=d;var E=f.stateNode;if(f.tag===5&&E!==null&&(f=E,m!==null&&(E=Xt(d,m),E!=null&&C.push(tr(d,E,f)))),b)break;d=d.return}0<C.length&&(g=new k(g,N,null,t,h),y.push({event:g,listeners:C}))}}if((a&7)===0){e:{if(g=e==="mouseover"||e==="pointerover",k=e==="mouseout"||e==="pointerout",g&&t!==Jn&&(N=t.relatedTarget||t.fromElement)&&(Pa(N)||N[Ge]))break e;if(k||g){if(g=h.window===h?h:(g=h.ownerDocument)?g.defaultView||g.parentWindow:window,k){if(N=t.relatedTarget||t.toElement,k=c,N=N?Pa(N):null,N!==null&&(b=Ba(N),N!==b||N.tag!==5&&N.tag!==6))N=null}else k=null,N=c;if(k!==N){if(C=us,E="onMouseLeave",m="onMouseEnter",d="mouse",e==="pointerout"||e==="pointerover")C=cs,E="onPointerLeave",m="onPointerEnter",d="pointer";if(b=k==null?g:Ga(k),f=N==null?g:Ga(N),g=new C(E,d+"leave",k,t,h),g.target=b,g.relatedTarget=f,E=null,Pa(h)===c&&(C=new C(m,d+"enter",N,t,h),C.target=f,C.relatedTarget=b,E=C),b=E,k&&N)a:{C=k,m=N,d=0;for(f=C;f;f=Ua(f))d++;f=0;for(E=m;E;E=Ua(E))f++;for(;0<d-f;)C=Ua(C),d--;for(;0<f-d;)m=Ua(m),f--;for(;d--;){if(C===m||m!==null&&C===m.alternate)break a;C=Ua(C),m=Ua(m)}C=null}else C=null;k!==null&&ks(y,g,k,C,!1),N!==null&&b!==null&&ks(y,b,N,C,!0)}}}e:{if(g=c?Ga(c):window,k=g.nodeName&&g.nodeName.toLowerCase(),k==="select"||k==="input"&&g.type==="file")var q=sf;else if(fs(g))if(Du)q=mf;else{q=df;var x=uf}else(k=g.nodeName)&&k.toLowerCase()==="input"&&(g.type==="checkbox"||g.type==="radio")&&(q=cf);if(q&&(q=q(e,c))){Iu(y,q,t,h);break e}x&&x(e,g,c),e==="focusout"&&(x=g._wrapperState)&&x.controlled&&g.type==="number"&&Kn(g,"number",g.value)}switch(x=c?Ga(c):window,e){case"focusin":if(fs(x)||x.contentEditable==="true")Ya=x,nl=c,Qt=null;break;case"focusout":Qt=nl=Ya=null;break;case"mousedown":ll=!0;break;case"contextmenu":case"mouseup":case"dragend":ll=!1,ys(y,t,h);break;case"selectionchange":if(vf)break;case"keydown":case"keyup":ys(y,t,h)}var M;if(Yl)e:{switch(e){case"compositionstart":var z="onCompositionStart";break e;case"compositionend":z="onCompositionEnd";break e;case"compositionupdate":z="onCompositionUpdate";break e}z=void 0}else Ka?bu(e,t)&&(z="onCompositionEnd"):e==="keydown"&&t.keyCode===229&&(z="onCompositionStart");if(z&&(Ru&&t.locale!=="ko"&&(Ka||z!=="onCompositionStart"?z==="onCompositionEnd"&&Ka&&(M=Ou()):(sa=h,$l=("value"in sa)?sa.value:sa.textContent,Ka=!0)),x=yo(c,z),0<x.length&&(z=new ds(z,e,null,t,h),y.push({event:z,listeners:x}),M?z.data=M:(M=Fu(t),M!==null&&(z.data=M)))),M=tf?rf(e,t):of(e,t))c=yo(c,"onBeforeInput"),0<c.length&&(h=new ds("onBeforeInput","beforeinput",null,t,h),y.push({event:h,listeners:c}),h.data=M)}Ku(y,a)})}function tr(e,a,t){return{instance:e,listener:a,currentTarget:t}}function yo(e,a){for(var t=a+"Capture",r=[];e!==null;){var o=e,n=o.stateNode;o.tag===5&&n!==null&&(o=n,n=Xt(e,t),n!=null&&r.unshift(tr(e,n,o)),n=Xt(e,a),n!=null&&r.push(tr(e,n,o))),e=e.return}return r}function Ua(e){if(e===null)return null;do e=e.return;while(e&&e.tag!==5);return e?e:null}function ks(e,a,t,r,o){for(var n=a._reactName,l=[];t!==null&&t!==r;){var s=t,u=s.alternate,c=s.stateNode;if(u!==null&&u===r)break;s.tag===5&&c!==null&&(s=c,o?(u=Xt(t,n),u!=null&&l.unshift(tr(t,u,s))):o||(u=Xt(t,n),u!=null&&l.push(tr(t,u,s)))),t=t.return}l.length!==0&&e.push({event:a,listeners:l})}function Ns(e){return(typeof e==="string"?e:""+e).replace(gf,`
`).replace(yf,"")}function Kr(e,a,t){if(a=Ns(a),Ns(e)!==a&&t)throw Error(S(425))}function So(){}function cl(e,a){return e==="textarea"||e==="noscript"||typeof a.children==="string"||typeof a.children==="number"||typeof a.dangerouslySetInnerHTML==="object"&&a.dangerouslySetInnerHTML!==null&&a.dangerouslySetInnerHTML.__html!=null}function kf(e){setTimeout(function(){throw e})}function _n(e,a){var t=a,r=0;do{var o=t.nextSibling;if(e.removeChild(t),o&&o.nodeType===8)if(t=o.data,t==="/$"){if(r===0){e.removeChild(o),Jt(a);return}r--}else t!=="$"&&t!=="$?"&&t!=="$!"||r++;t=o}while(t);Jt(a)}function pa(e){for(;e!=null;e=e.nextSibling){var a=e.nodeType;if(a===1||a===3)break;if(a===8){if(a=e.data,a==="$"||a==="$!"||a==="$?")break;if(a==="/$")return null}}return e}function Cs(e){e=e.previousSibling;for(var a=0;e;){if(e.nodeType===8){var t=e.data;if(t==="$"||t==="$!"||t==="$?"){if(a===0)return e;a--}else t==="/$"&&a++}e=e.previousSibling}return null}function Pa(e){var a=e[Le];if(a)return a;for(var t=e.parentNode;t;){if(a=t[Ge]||t[Le]){if(t=a.alternate,a.child!==null||t!==null&&t.child!==null)for(e=Cs(e);e!==null;){if(t=e[Le])return t;e=Cs(e)}return a}e=t,t=e.parentNode}return null}function mr(e){return e=e[Le]||e[Ge],!e||e.tag!==5&&e.tag!==6&&e.tag!==13&&e.tag!==3?null:e}function Ga(e){if(e.tag===5||e.tag===6)return e.stateNode;throw Error(S(33))}function Lo(e){return e[rr]||null}function ka(e){return{current:e}}function A(e){0>Za||(e.current=fl[Za],fl[Za]=null,Za--)}function F(e,a){Za++,fl[Za]=e.current,e.current=a}function dt(e,a){var t=e.type.contextTypes;if(!t)return Sa;var r=e.stateNode;if(r&&r.__reactInternalMemoizedUnmaskedChildContext===a)return r.__reactInternalMemoizedMaskedChildContext;var o={},n;for(n in t)o[n]=a[n];return r&&(e=e.stateNode,e.__reactInternalMemoizedUnmaskedChildContext=a,e.__reactInternalMemoizedMaskedChildContext=o),o}function fe(e){return e=e.childContextTypes,e!==null&&e!==void 0}function Eo(){A(pe),A(le)}function qs(e,a,t){if(le.current!==Sa)throw Error(S(168));F(le,a),F(pe,t)}function Xu(e,a,t){var r=e.stateNode;if(a=a.childContextTypes,typeof r.getChildContext!=="function")return t;r=r.getChildContext();for(var o in r)if(!(o in a))throw Error(S(108,sp(e)||"Unknown",o));return U({},t,r)}function ko(e){return e=(e=e.stateNode)&&e.__reactInternalMemoizedMergedChildContext||Sa,ba=le.current,F(le,e),F(pe,pe.current),!0}function xs(e,a,t){var r=e.stateNode;if(!r)throw Error(S(169));t?(e=Xu(e,a,ba),r.__reactInternalMemoizedMergedChildContext=e,A(pe),A(le),F(le,e)):A(pe),F(pe,t)}function Gu(e){$e===null?$e=[e]:$e.push(e)}function Cf(e){Bo=!0,Gu(e)}function Na(){if(!Tn&&$e!==null){Tn=!0;var e=0,a=R;try{var t=$e;for(R=1;e<t.length;e++){var r=t[e];do r=r(!0);while(r!==null)}$e=null,Bo=!1}catch(o){throw $e!==null&&($e=$e.slice(e+1)),ku(Vl,Na),o}finally{R=a,Tn=!1}}return null}function Ma(e,a){Ja[ja++]=wo,Ja[ja++]=No,No=e,wo=a}function Zu(e,a,t){ke[Ne++]=We,ke[Ne++]=Ke,ke[Ne++]=Fa,Fa=e;var r=We;e=Ke;var o=32-Re(r)-1;r&=~(1<<o),t+=1;var n=32-Re(a)+o;if(30<n){var l=o-o%5;n=(r&(1<<l)-1).toString(32),r>>=l,o-=l,We=1<<32-Re(a)+o|t<<o|r,Ke=n+e}else We=1<<n|t<<o|r,Ke=e}function Gl(e){e.return!==null&&(Ma(e,1),Zu(e,1,0))}function Zl(e){for(;e===No;)No=Ja[--ja],Ja[ja]=null,wo=Ja[--ja],Ja[ja]=null;for(;e===Fa;)Fa=ke[--Ne],ke[Ne]=null,Ke=ke[--Ne],ke[Ne]=null,We=ke[--Ne],ke[Ne]=null}function Ju(e,a){var t=we(5,null,null,0);t.elementType="DELETED",t.stateNode=a,t.return=e,a=e.deletions,a===null?(e.deletions=[t],e.flags|=16):a.push(t)}function Ms(e,a){switch(e.tag){case 5:var t=e.type;return a=a.nodeType!==1||t.toLowerCase()!==a.nodeName.toLowerCase()?null:a,a!==null?(e.stateNode=a,ye=e,ge=pa(a.firstChild),!0):!1;case 6:return a=e.pendingProps===""||a.nodeType!==3?null:a,a!==null?(e.stateNode=a,ye=e,ge=null,!0):!1;case 13:return a=a.nodeType!==8?null:a,a!==null?(t=Fa!==null?{id:We,overflow:Ke}:null,e.memoizedState={dehydrated:a,treeContext:t,retryLane:1073741824},t=we(18,null,null,0),t.stateNode=a,t.return=e,e.child=t,ye=e,ge=null,!0):!1;default:return!1}}function vl(e){return(e.mode&1)!==0&&(e.flags&128)===0}function hl(e){if(L){var a=ge;if(a){var t=a;if(!Ms(e,a)){if(vl(e))throw Error(S(418));a=pa(t.nextSibling);var r=ye;a&&Ms(e,a)?Ju(r,t):(e.flags=e.flags&-4097|2,L=!1,ye=e)}}else{if(vl(e))throw Error(S(418));e.flags=e.flags&-4097|2,L=!1,ye=e}}}function zs(e){for(e=e.return;e!==null&&e.tag!==5&&e.tag!==3&&e.tag!==13;)e=e.return;ye=e}function Yr(e){if(e!==ye)return!1;if(!L)return zs(e),L=!0,!1;var a;if((a=e.tag!==3)&&!(a=e.tag!==5)&&(a=e.type,a=a!=="head"&&a!=="body"&&!cl(e.type,e.memoizedProps)),a&&(a=ge)){if(vl(e))throw ju(),Error(S(418));for(;a;)Ju(e,a),a=pa(a.nextSibling)}if(zs(e),e.tag===13){if(e=e.memoizedState,e=e!==null?e.dehydrated:null,!e)throw Error(S(317));e:{e=e.nextSibling;for(a=0;e;){if(e.nodeType===8){var t=e.data;if(t==="/$"){if(a===0){ge=pa(e.nextSibling);break e}a--}else t!=="$"&&t!=="$!"&&t!=="$?"||a++}e=e.nextSibling}ge=null}}else ge=ye?pa(e.stateNode.nextSibling):null;return!0}function ju(){for(var e=ge;e;)e=pa(e.nextSibling)}function ct(){ge=ye=null,L=!1}function Jl(e){Oe===null?Oe=[e]:Oe.push(e)}function Mt(e,a,t){if(e=t.ref,e!==null&&typeof e!=="function"&&typeof e!=="object"){if(t._owner){if(t=t._owner,t){if(t.tag!==1)throw Error(S(309));var r=t.stateNode}if(!r)throw Error(S(147,e));var o=r,n=""+e;if(a!==null&&a.ref!==null&&typeof a.ref==="function"&&a.ref._stringRef===n)return a.ref;return a=function(l){var s=o.refs;l===null?delete s[n]:s[n]=l},a._stringRef=n,a}if(typeof e!=="string")throw Error(S(284));if(!t._owner)throw Error(S(290,e))}return e}function Xr(e,a){throw e=Object.prototype.toString.call(a),Error(S(31,e==="[object Object]"?"object with keys {"+Object.keys(a).join(", ")+"}":e))}function Ps(e){var a=e._init;return a(e._payload)}function ed(e){function a(m,d){if(e){var f=m.deletions;f===null?(m.deletions=[d],m.flags|=16):f.push(d)}}function t(m,d){if(!e)return null;for(;d!==null;)a(m,d),d=d.sibling;return null}function r(m,d){for(m=new Map;d!==null;)d.key!==null?m.set(d.key,d):m.set(d.index,d),d=d.sibling;return m}function o(m,d){return m=ga(m,d),m.index=0,m.sibling=null,m}function n(m,d,f){if(m.index=f,!e)return m.flags|=1048576,d;if(f=m.alternate,f!==null)return f=f.index,f<d?(m.flags|=2,d):f;return m.flags|=2,d}function l(m){return e&&m.alternate===null&&(m.flags|=2),m}function s(m,d,f,E){if(d===null||d.tag!==6)return d=An(f,m.mode,E),d.return=m,d;return d=o(d,f),d.return=m,d}function u(m,d,f,E){var q=f.type;if(q===Wa)return h(m,d,f.props.children,E,f.key);if(d!==null&&(d.elementType===q||typeof q==="object"&&q!==null&&q.$$typeof===oa&&Ps(q)===d.type))return E=o(d,f.props),E.ref=Mt(m,d,f),E.return=m,E;return E=uo(f.type,f.key,f.props,null,m.mode,E),E.ref=Mt(m,d,f),E.return=m,E}function c(m,d,f,E){if(d===null||d.tag!==4||d.stateNode.containerInfo!==f.containerInfo||d.stateNode.implementation!==f.implementation)return d=Ln(f,m.mode,E),d.return=m,d;return d=o(d,f.children||[]),d.return=m,d}function h(m,d,f,E,q){if(d===null||d.tag!==7)return d=Ra(f,m.mode,E,q),d.return=m,d;return d=o(d,f),d.return=m,d}function y(m,d,f){if(typeof d==="string"&&d!==""||typeof d==="number")return d=An(""+d,m.mode,f),d.return=m,d;if(typeof d==="object"&&d!==null){switch(d.$$typeof){case Ar:return f=uo(d.type,d.key,d.props,null,m.mode,f),f.ref=Mt(m,null,d),f.return=m,f;case $a:return d=Ln(d,m.mode,f),d.return=m,d;case oa:var E=d._init;return y(m,E(d._payload),f)}if(Tt(d)||wt(d))return d=Ra(d,m.mode,f,null),d.return=m,d;Xr(m,d)}return null}function g(m,d,f,E){var q=d!==null?d.key:null;if(typeof f==="string"&&f!==""||typeof f==="number")return q!==null?null:s(m,d,""+f,E);if(typeof f==="object"&&f!==null){switch(f.$$typeof){case Ar:return f.key===q?u(m,d,f,E):null;case $a:return f.key===q?c(m,d,f,E):null;case oa:return q=f._init,g(m,d,q(f._payload),E)}if(Tt(f)||wt(f))return q!==null?null:h(m,d,f,E,null);Xr(m,f)}return null}function k(m,d,f,E,q){if(typeof E==="string"&&E!==""||typeof E==="number")return m=m.get(f)||null,s(d,m,""+E,q);if(typeof E==="object"&&E!==null){switch(E.$$typeof){case Ar:return m=m.get(E.key===null?f:E.key)||null,u(d,m,E,q);case $a:return m=m.get(E.key===null?f:E.key)||null,c(d,m,E,q);case oa:var x=E._init;return k(m,d,f,x(E._payload),q)}if(Tt(E)||wt(E))return m=m.get(f)||null,h(d,m,E,q,null);Xr(d,E)}return null}function N(m,d,f,E){for(var q=null,x=null,M=d,z=d=0,W=null;M!==null&&z<f.length;z++){M.index>z?(W=M,M=null):W=M.sibling;var _=g(m,M,f[z],E);if(_===null){M===null&&(M=W);break}e&&M&&_.alternate===null&&a(m,M),d=n(_,d,z),x===null?q=_:x.sibling=_,x=_,M=W}if(z===f.length)return t(m,M),L&&Ma(m,z),q;if(M===null){for(;z<f.length;z++)M=y(m,f[z],E),M!==null&&(d=n(M,d,z),x===null?q=M:x.sibling=M,x=M);return L&&Ma(m,z),q}for(M=r(m,M);z<f.length;z++)W=k(M,m,z,f[z],E),W!==null&&(e&&W.alternate!==null&&M.delete(W.key===null?z:W.key),d=n(W,d,z),x===null?q=W:x.sibling=W,x=W);return e&&M.forEach(function(Me){return a(m,Me)}),L&&Ma(m,z),q}function C(m,d,f,E){var q=wt(f);if(typeof q!=="function")throw Error(S(150));if(f=q.call(f),f==null)throw Error(S(151));for(var x=q=null,M=d,z=d=0,W=null,_=f.next();M!==null&&!_.done;z++,_=f.next()){M.index>z?(W=M,M=null):W=M.sibling;var Me=g(m,M,_.value,E);if(Me===null){M===null&&(M=W);break}e&&M&&Me.alternate===null&&a(m,M),d=n(Me,d,z),x===null?q=Me:x.sibling=Me,x=Me,M=W}if(_.done)return t(m,M),L&&Ma(m,z),q;if(M===null){for(;!_.done;z++,_=f.next())_=y(m,_.value,E),_!==null&&(d=n(_,d,z),x===null?q=_:x.sibling=_,x=_);return L&&Ma(m,z),q}for(M=r(m,M);!_.done;z++,_=f.next())_=k(M,m,z,_.value,E),_!==null&&(e&&_.alternate!==null&&M.delete(_.key===null?z:_.key),d=n(_,d,z),x===null?q=_:x.sibling=_,x=_);return e&&M.forEach(function(xr){return a(m,xr)}),L&&Ma(m,z),q}function b(m,d,f,E){if(typeof f==="object"&&f!==null&&f.type===Wa&&f.key===null&&(f=f.props.children),typeof f==="object"&&f!==null){switch(f.$$typeof){case Ar:e:{for(var q=f.key,x=d;x!==null;){if(x.key===q){if(q=f.type,q===Wa){if(x.tag===7){t(m,x.sibling),d=o(x,f.props.children),d.return=m,m=d;break e}}else if(x.elementType===q||typeof q==="object"&&q!==null&&q.$$typeof===oa&&Ps(q)===x.type){t(m,x.sibling),d=o(x,f.props),d.ref=Mt(m,x,f),d.return=m,m=d;break e}t(m,x);break}else a(m,x);x=x.sibling}f.type===Wa?(d=Ra(f.props.children,m.mode,E,f.key),d.return=m,m=d):(E=uo(f.type,f.key,f.props,null,m.mode,E),E.ref=Mt(m,d,f),E.return=m,m=E)}return l(m);case $a:e:{for(x=f.key;d!==null;){if(d.key===x)if(d.tag===4&&d.stateNode.containerInfo===f.containerInfo&&d.stateNode.implementation===f.implementation){t(m,d.sibling),d=o(d,f.children||[]),d.return=m,m=d;break e}else{t(m,d);break}else a(m,d);d=d.sibling}d=Ln(f,m.mode,E),d.return=m,m=d}return l(m);case oa:return x=f._init,b(m,d,x(f._payload),E)}if(Tt(f))return N(m,d,f,E);if(wt(f))return C(m,d,f,E);Xr(m,f)}return typeof f==="string"&&f!==""||typeof f==="number"?(f=""+f,d!==null&&d.tag===6?(t(m,d.sibling),d=o(d,f),d.return=m,m=d):(t(m,d),d=An(f,m.mode,E),d.return=m,m=d),l(m)):t(m,d)}return b}function ei(){jl=et=qo=null}function ai(e){var a=Co.current;A(Co),e._currentValue=a}function gl(e,a,t){for(;e!==null;){var r=e.alternate;if((e.childLanes&a)!==a?(e.childLanes|=a,r!==null&&(r.childLanes|=a)):r!==null&&(r.childLanes&a)!==a&&(r.childLanes|=a),e===t)break;e=e.return}}function it(e,a){qo=e,jl=et=null,e=e.dependencies,e!==null&&e.firstContext!==null&&((e.lanes&a)!==0&&(me=!0),e.firstContext=null)}function qe(e){var a=e._currentValue;if(jl!==e)if(e={context:e,memoizedValue:a,next:null},et===null){if(qo===null)throw Error(S(308));et=e,qo.dependencies={lanes:0,firstContext:e}}else et=et.next=e;return a}function ti(e){_a===null?_a=[e]:_a.push(e)}function td(e,a,t,r){var o=a.interleaved;return o===null?(t.next=t,ti(a)):(t.next=o.next,o.next=t),a.interleaved=t,Ze(e,r)}function Ze(e,a){e.lanes|=a;var t=e.alternate;t!==null&&(t.lanes|=a),t=e;for(e=e.return;e!==null;)e.childLanes|=a,t=e.alternate,t!==null&&(t.childLanes|=a),t=e,e=e.return;return t.tag===3?t.stateNode:null}function ri(e){e.updateQueue={baseState:e.memoizedState,firstBaseUpdate:null,lastBaseUpdate:null,shared:{pending:null,interleaved:null,lanes:0},effects:null}}function rd(e,a){e=e.updateQueue,a.updateQueue===e&&(a.updateQueue={baseState:e.baseState,firstBaseUpdate:e.firstBaseUpdate,lastBaseUpdate:e.lastBaseUpdate,shared:e.shared,effects:e.effects})}function Ye(e,a){return{eventTime:e,lane:a,tag:0,payload:null,callback:null,next:null}}function fa(e,a,t){var r=e.updateQueue;if(r===null)return null;if(r=r.shared,(O&2)!==0){var o=r.pending;return o===null?a.next=a:(a.next=o.next,o.next=a),r.pending=a,Ze(e,t)}return o=r.interleaved,o===null?(a.next=a,ti(r)):(a.next=o.next,o.next=a),r.interleaved=a,Ze(e,t)}function ro(e,a,t){if(a=a.updateQueue,a!==null&&(a=a.shared,(t&4194240)!==0)){var r=a.lanes;r&=e.pendingLanes,t|=r,a.lanes=t,Ql(e,t)}}function _s(e,a){var{updateQueue:t,alternate:r}=e;if(r!==null&&(r=r.updateQueue,t===r)){var o=null,n=null;if(t=t.firstBaseUpdate,t!==null){do{var l={eventTime:t.eventTime,lane:t.lane,tag:t.tag,payload:t.payload,callback:t.callback,next:null};n===null?o=n=l:n=n.next=l,t=t.next}while(t!==null);n===null?o=n=a:n=n.next=a}else o=n=a;t={baseState:r.baseState,firstBaseUpdate:o,lastBaseUpdate:n,shared:r.shared,effects:r.effects},e.updateQueue=t;return}e=t.lastBaseUpdate,e===null?t.firstBaseUpdate=a:e.next=a,t.lastBaseUpdate=a}function xo(e,a,t,r){var o=e.updateQueue;na=!1;var{firstBaseUpdate:n,lastBaseUpdate:l}=o,s=o.shared.pending;if(s!==null){o.shared.pending=null;var u=s,c=u.next;u.next=null,l===null?n=c:l.next=c,l=u;var h=e.alternate;h!==null&&(h=h.updateQueue,s=h.lastBaseUpdate,s!==l&&(s===null?h.firstBaseUpdate=c:s.next=c,h.lastBaseUpdate=u))}if(n!==null){var y=o.baseState;l=0,h=c=u=null,s=n;do{var{lane:g,eventTime:k}=s;if((r&g)===g){h!==null&&(h=h.next={eventTime:k,lane:0,tag:s.tag,payload:s.payload,callback:s.callback,next:null});e:{var N=e,C=s;switch(g=a,k=t,C.tag){case 1:if(N=C.payload,typeof N==="function"){y=N.call(k,y,g);break e}y=N;break e;case 3:N.flags=N.flags&-65537|128;case 0:if(N=C.payload,g=typeof N==="function"?N.call(k,y,g):N,g===null||g===void 0)break e;y=U({},y,g);break e;case 2:na=!0}}s.callback!==null&&s.lane!==0&&(e.flags|=64,g=o.effects,g===null?o.effects=[s]:g.push(s))}else k={eventTime:k,lane:g,tag:s.tag,payload:s.payload,callback:s.callback,next:null},h===null?(c=h=k,u=y):h=h.next=k,l|=g;if(s=s.next,s===null)if(s=o.shared.pending,s===null)break;else g=s,s=g.next,g.next=null,o.lastBaseUpdate=g,o.shared.pending=null}while(1);if(h===null&&(u=y),o.baseState=u,o.firstBaseUpdate=c,o.lastBaseUpdate=h,a=o.shared.interleaved,a!==null){o=a;do l|=o.lane,o=o.next;while(o!==a)}else n===null&&(o.shared.lanes=0);Da|=l,e.lanes=l,e.memoizedState=y}}function Ts(e,a,t){if(e=a.effects,a.effects=null,e!==null)for(a=0;a<e.length;a++){var r=e[a],o=r.callback;if(o!==null){if(r.callback=null,r=t,typeof o!=="function")throw Error(S(191,o));o.call(r)}}}function Ta(e){if(e===pr)throw Error(S(174));return e}function oi(e,a){switch(F(nr,a),F(or,e),F(Ve,pr),e=a.nodeType,e){case 9:case 11:a=(a=a.documentElement)?a.namespaceURI:Xn(null,"");break;default:e=e===8?a.parentNode:a,a=e.namespaceURI||null,e=e.tagName,a=Xn(a,e)}A(Ve),F(Ve,a)}function pt(){A(Ve),A(or),A(nr)}function od(e){Ta(nr.current);var a=Ta(Ve.current),t=Xn(a,e.type);a!==t&&(F(or,e),F(Ve,t))}function ni(e){or.current===e&&(A(Ve),A(or))}function Mo(e){for(var a=e;a!==null;){if(a.tag===13){var t=a.memoizedState;if(t!==null&&(t=t.dehydrated,t===null||t.data==="$?"||t.data==="$!"))return a}else if(a.tag===19&&a.memoizedProps.revealOrder!==void 0){if((a.flags&128)!==0)return a}else if(a.child!==null){a.child.return=a,a=a.child;continue}if(a===e)break;for(;a.sibling===null;){if(a.return===null||a.return===e)return null;a=a.return}a.sibling.return=a.return,a=a.sibling}return null}function li(){for(var e=0;e<On.length;e++)On[e]._workInProgressVersionPrimary=null;On.length=0}function re(){throw Error(S(321))}function ii(e,a){if(a===null)return!1;for(var t=0;t<a.length&&t<e.length;t++)if(!Fe(e[t],a[t]))return!1;return!0}function si(e,a,t,r,o,n){if(Ia=n,Q=a,a.memoizedState=null,a.updateQueue=null,a.lanes=0,oo.current=e===null||e.memoizedState===null?_f:Tf,e=t(r,o),Ut){n=0;do{if(Ut=!1,lr=0,25<=n)throw Error(S(301));n+=1,G=Y=null,a.updateQueue=null,oo.current=Of,e=t(r,o)}while(Ut)}if(oo.current=Po,a=Y!==null&&Y.next!==null,Ia=0,G=Y=Q=null,zo=!1,a)throw Error(S(300));return e}function ui(){var e=lr!==0;return lr=0,e}function Ae(){var e={memoizedState:null,baseState:null,baseQueue:null,queue:null,next:null};return G===null?Q.memoizedState=G=e:G=G.next=e,G}function xe(){if(Y===null){var e=Q.alternate;e=e!==null?e.memoizedState:null}else e=Y.next;var a=G===null?Q.memoizedState:G.next;if(a!==null)G=a,Y=e;else{if(e===null)throw Error(S(310));Y=e,e={memoizedState:Y.memoizedState,baseState:Y.baseState,baseQueue:Y.baseQueue,queue:Y.queue,next:null},G===null?Q.memoizedState=G=e:G=G.next=e}return G}function ir(e,a){return typeof a==="function"?a(e):a}function bn(e){var a=xe(),t=a.queue;if(t===null)throw Error(S(311));t.lastRenderedReducer=e;var r=Y,o=r.baseQueue,n=t.pending;if(n!==null){if(o!==null){var l=o.next;o.next=n.next,n.next=l}r.baseQueue=o=n,t.pending=null}if(o!==null){n=o.next,r=r.baseState;var s=l=null,u=null,c=n;do{var h=c.lane;if((Ia&h)===h)u!==null&&(u=u.next={lane:0,action:c.action,hasEagerState:c.hasEagerState,eagerState:c.eagerState,next:null}),r=c.hasEagerState?c.eagerState:e(r,c.action);else{var y={lane:h,action:c.action,hasEagerState:c.hasEagerState,eagerState:c.eagerState,next:null};u===null?(s=u=y,l=r):u=u.next=y,Q.lanes|=h,Da|=h}c=c.next}while(c!==null&&c!==n);u===null?l=r:u.next=s,Fe(r,a.memoizedState)||(me=!0),a.memoizedState=r,a.baseState=l,a.baseQueue=u,t.lastRenderedState=r}if(e=t.interleaved,e!==null){o=e;do n=o.lane,Q.lanes|=n,Da|=n,o=o.next;while(o!==e)}else o===null&&(t.lanes=0);return[a.memoizedState,t.dispatch]}function Fn(e){var a=xe(),t=a.queue;if(t===null)throw Error(S(311));t.lastRenderedReducer=e;var{dispatch:r,pending:o}=t,n=a.memoizedState;if(o!==null){t.pending=null;var l=o=o.next;do n=e(n,l.action),l=l.next;while(l!==o);Fe(n,a.memoizedState)||(me=!0),a.memoizedState=n,a.baseQueue===null&&(a.baseState=n),t.lastRenderedState=n}return[n,r]}function nd(){}function ld(e,a){var t=Q,r=xe(),o=a(),n=!Fe(r.memoizedState,o);if(n&&(r.memoizedState=o,me=!0),r=r.queue,di(ud.bind(null,t,r,e),[e]),r.getSnapshot!==a||n||G!==null&&G.memoizedState.tag&1){if(t.flags|=2048,sr(9,sd.bind(null,t,r,o,a),void 0,null),Z===null)throw Error(S(349));(Ia&30)!==0||id(t,a,o)}return o}function id(e,a,t){e.flags|=16384,e={getSnapshot:a,value:t},a=Q.updateQueue,a===null?(a={lastEffect:null,stores:null},Q.updateQueue=a,a.stores=[e]):(t=a.stores,t===null?a.stores=[e]:t.push(e))}function sd(e,a,t,r){a.value=t,a.getSnapshot=r,dd(a)&&cd(e)}function ud(e,a,t){return t(function(){dd(a)&&cd(e)})}function dd(e){var a=e.getSnapshot;e=e.value;try{var t=a();return!Fe(e,t)}catch(r){return!0}}function cd(e){var a=Ze(e,1);a!==null&&be(a,e,1,-1)}function Os(e){var a=Ae();return typeof e==="function"&&(e=e()),a.memoizedState=a.baseState=e,e={pending:null,interleaved:null,lanes:0,dispatch:null,lastRenderedReducer:ir,lastRenderedState:e},a.queue=e,e=e.dispatch=Pf.bind(null,Q,e),[a.memoizedState,e]}function sr(e,a,t,r){return e={tag:e,create:a,destroy:t,deps:r,next:null},a=Q.updateQueue,a===null?(a={lastEffect:null,stores:null},Q.updateQueue=a,a.lastEffect=e.next=e):(t=a.lastEffect,t===null?a.lastEffect=e.next=e:(r=t.next,t.next=e,e.next=r,a.lastEffect=e)),e}function md(){return xe().memoizedState}function no(e,a,t,r){var o=Ae();Q.flags|=e,o.memoizedState=sr(1|a,t,void 0,r===void 0?null:r)}function Vo(e,a,t,r){var o=xe();r=r===void 0?null:r;var n=void 0;if(Y!==null){var l=Y.memoizedState;if(n=l.destroy,r!==null&&ii(r,l.deps)){o.memoizedState=sr(a,t,n,r);return}}Q.flags|=e,o.memoizedState=sr(1|a,t,n,r)}function Rs(e,a){return no(8390656,8,e,a)}function di(e,a){return Vo(2048,8,e,a)}function pd(e,a){return Vo(4,2,e,a)}function fd(e,a){return Vo(4,4,e,a)}function vd(e,a){if(typeof a==="function")return e=e(),a(e),function(){a(null)};if(a!==null&&a!==void 0)return e=e(),a.current=e,function(){a.current=null}}function hd(e,a,t){return t=t!==null&&t!==void 0?t.concat([e]):null,Vo(4,4,vd.bind(null,a,e),t)}function ci(){}function gd(e,a){var t=xe();a=a===void 0?null:a;var r=t.memoizedState;if(r!==null&&a!==null&&ii(a,r[1]))return r[0];return t.memoizedState=[e,a],e}function yd(e,a){var t=xe();a=a===void 0?null:a;var r=t.memoizedState;if(r!==null&&a!==null&&ii(a,r[1]))return r[0];return e=e(),t.memoizedState=[e,a],e}function Sd(e,a,t){if((Ia&21)===0)return e.baseState&&(e.baseState=!1,me=!0),e.memoizedState=t;return Fe(t,a)||(t=Cu(),Q.lanes|=t,Da|=t,e.baseState=!0),a}function Mf(e,a){var t=R;R=t!==0&&4>t?t:4,e(!0);var r=Rn.transition;Rn.transition={};try{e(!1),a()}finally{R=t,Rn.transition=r}}function Ed(){return xe().memoizedState}function zf(e,a,t){var r=ha(e);if(t={lane:r,action:t,hasEagerState:!1,eagerState:null,next:null},kd(e))Nd(a,t);else if(t=td(e,a,t,r),t!==null){var o=ue();be(t,e,r,o),wd(t,a,r)}}function Pf(e,a,t){var r=ha(e),o={lane:r,action:t,hasEagerState:!1,eagerState:null,next:null};if(kd(e))Nd(a,o);else{var n=e.alternate;if(e.lanes===0&&(n===null||n.lanes===0)&&(n=a.lastRenderedReducer,n!==null))try{var l=a.lastRenderedState,s=n(l,t);if(o.hasEagerState=!0,o.eagerState=s,Fe(s,l)){var u=a.interleaved;u===null?(o.next=o,ti(a)):(o.next=u.next,u.next=o),a.interleaved=o;return}}catch(c){}finally{}t=td(e,a,o,r),t!==null&&(o=ue(),be(t,e,r,o),wd(t,a,r))}}function kd(e){var a=e.alternate;return e===Q||a!==null&&a===Q}function Nd(e,a){Ut=zo=!0;var t=e.pending;t===null?a.next=a:(a.next=t.next,t.next=a),e.pending=a}function wd(e,a,t){if((t&4194240)!==0){var r=a.lanes;r&=e.pendingLanes,t|=r,a.lanes=t,Ql(e,t)}}function _e(e,a){if(e&&e.defaultProps){a=U({},a),e=e.defaultProps;for(var t in e)a[t]===void 0&&(a[t]=e[t]);return a}return a}function yl(e,a,t,r){a=e.memoizedState,t=t(r,a),t=t===null||t===void 0?a:U({},a,t),e.memoizedState=t,e.lanes===0&&(e.updateQueue.baseState=t)}function bs(e,a,t,r,o,n,l){return e=e.stateNode,typeof e.shouldComponentUpdate==="function"?e.shouldComponentUpdate(r,n,l):a.prototype&&a.prototype.isPureReactComponent?!er(t,r)||!er(o,n):!0}function Cd(e,a,t){var r=!1,o=Sa,n=a.contextType;return typeof n==="object"&&n!==null?n=qe(n):(o=fe(a)?ba:le.current,r=a.contextTypes,n=(r=r!==null&&r!==void 0)?dt(e,o):Sa),a=new a(t,n),e.memoizedState=a.state!==null&&a.state!==void 0?a.state:null,a.updater=Qo,e.stateNode=a,a._reactInternals=e,r&&(e=e.stateNode,e.__reactInternalMemoizedUnmaskedChildContext=o,e.__reactInternalMemoizedMaskedChildContext=n),a}function Fs(e,a,t,r){e=a.state,typeof a.componentWillReceiveProps==="function"&&a.componentWillReceiveProps(t,r),typeof a.UNSAFE_componentWillReceiveProps==="function"&&a.UNSAFE_componentWillReceiveProps(t,r),a.state!==e&&Qo.enqueueReplaceState(a,a.state,null)}function Sl(e,a,t,r){var o=e.stateNode;o.props=t,o.state=e.memoizedState,o.refs={},ri(e);var n=a.contextType;typeof n==="object"&&n!==null?o.context=qe(n):(n=fe(a)?ba:le.current,o.context=dt(e,n)),o.state=e.memoizedState,n=a.getDerivedStateFromProps,typeof n==="function"&&(yl(e,a,n,t),o.state=e.memoizedState),typeof a.getDerivedStateFromProps==="function"||typeof o.getSnapshotBeforeUpdate==="function"||typeof o.UNSAFE_componentWillMount!=="function"&&typeof o.componentWillMount!=="function"||(a=o.state,typeof o.componentWillMount==="function"&&o.componentWillMount(),typeof o.UNSAFE_componentWillMount==="function"&&o.UNSAFE_componentWillMount(),a!==o.state&&Qo.enqueueReplaceState(o,o.state,null),xo(e,t,o,r),o.state=e.memoizedState),typeof o.componentDidMount==="function"&&(e.flags|=4194308)}function ft(e,a){try{var t="",r=a;do t+=ip(r),r=r.return;while(r);var o=t}catch(n){o=`
Error generating stack: `+n.message+`
`+n.stack}return{value:e,source:a,stack:o,digest:null}}function In(e,a,t){return{value:e,source:null,stack:t!=null?t:null,digest:a!=null?a:null}}function El(e,a){try{console.error(a.value)}catch(t){setTimeout(function(){throw t})}}function qd(e,a,t){t=Ye(-1,t),t.tag=3,t.payload={element:null};var r=a.value;return t.callback=function(){To||(To=!0,_l=r),El(e,a)},t}function xd(e,a,t){t=Ye(-1,t),t.tag=3;var r=e.type.getDerivedStateFromError;if(typeof r==="function"){var o=a.value;t.payload=function(){return r(o)},t.callback=function(){El(e,a)}}var n=e.stateNode;return n!==null&&typeof n.componentDidCatch==="function"&&(t.callback=function(){El(e,a),typeof r!=="function"&&(va===null?va=new Set([this]):va.add(this));var l=a.stack;this.componentDidCatch(a.value,{componentStack:l!==null?l:""})}),t}function Is(e,a,t){var r=e.pingCache;if(r===null){r=e.pingCache=new Rf;var o=new Set;r.set(a,o)}else o=r.get(a),o===void 0&&(o=new Set,r.set(a,o));o.has(t)||(o.add(t),e=Kf.bind(null,e,a,t),a.then(e,e))}function Ds(e){do{var a;if(a=e.tag===13)a=e.memoizedState,a=a!==null?a.dehydrated!==null?!0:!1:!0;if(a)return e;e=e.return}while(e!==null);return null}function As(e,a,t,r,o){if((e.mode&1)===0)return e===a?e.flags|=65536:(e.flags|=128,t.flags|=131072,t.flags&=-52805,t.tag===1&&(t.alternate===null?t.tag=17:(a=Ye(-1,1),a.tag=2,fa(t,a,1))),t.lanes|=1),e;return e.flags|=65536,e.lanes=o,e}function se(e,a,t,r){a.child=e===null?ad(a,null,t,r):mt(a,e.child,t,r)}function Ls(e,a,t,r,o){t=t.render;var n=a.ref;if(it(a,o),r=si(e,a,t,r,n,o),t=ui(),e!==null&&!me)return a.updateQueue=e.updateQueue,a.flags&=-2053,e.lanes&=~o,Je(e,a,o);return L&&t&&Gl(a),a.flags|=1,se(e,a,r,o),a.child}function Bs(e,a,t,r,o){if(e===null){var n=t.type;if(typeof n==="function"&&!Si(n)&&n.defaultProps===void 0&&t.compare===null&&t.defaultProps===void 0)return a.tag=15,a.type=n,Md(e,a,n,r,o);return e=uo(t.type,null,r,a,a.mode,o),e.ref=a.ref,e.return=a,a.child=e}if(n=e.child,(e.lanes&o)===0){var l=n.memoizedProps;if(t=t.compare,t=t!==null?t:er,t(l,r)&&e.ref===a.ref)return Je(e,a,o)}return a.flags|=1,e=ga(n,r),e.ref=a.ref,e.return=a,a.child=e}function Md(e,a,t,r,o){if(e!==null){var n=e.memoizedProps;if(er(n,r)&&e.ref===a.ref)if(me=!1,a.pendingProps=r=n,(e.lanes&o)!==0)(e.flags&131072)!==0&&(me=!0);else return a.lanes=e.lanes,Je(e,a,o)}return kl(e,a,t,r,o)}function zd(e,a,t){var r=a.pendingProps,o=r.children,n=e!==null?e.memoizedState:null;if(r.mode==="hidden")if((a.mode&1)===0)a.memoizedState={baseLanes:0,cachePool:null,transitions:null},F(tt,he),he|=t;else{if((t&1073741824)===0)return e=n!==null?n.baseLanes|t:t,a.lanes=a.childLanes=1073741824,a.memoizedState={baseLanes:e,cachePool:null,transitions:null},a.updateQueue=null,F(tt,he),he|=e,null;a.memoizedState={baseLanes:0,cachePool:null,transitions:null},r=n!==null?n.baseLanes:t,F(tt,he),he|=r}else n!==null?(r=n.baseLanes|t,a.memoizedState=null):r=t,F(tt,he),he|=r;return se(e,a,o,t),a.child}function Pd(e,a){var t=a.ref;if(e===null&&t!==null||e!==null&&e.ref!==t)a.flags|=512,a.flags|=2097152}function kl(e,a,t,r,o){var n=fe(t)?ba:le.current;if(n=dt(a,n),it(a,o),t=si(e,a,t,r,n,o),r=ui(),e!==null&&!me)return a.updateQueue=e.updateQueue,a.flags&=-2053,e.lanes&=~o,Je(e,a,o);return L&&r&&Gl(a),a.flags|=1,se(e,a,t,o),a.child}function Vs(e,a,t,r,o){if(fe(t)){var n=!0;ko(a)}else n=!1;if(it(a,o),a.stateNode===null)lo(e,a),Cd(a,t,r),Sl(a,t,r,o),r=!0;else if(e===null){var{stateNode:l,memoizedProps:s}=a;l.props=s;var u=l.context,c=t.contextType;typeof c==="object"&&c!==null?c=qe(c):(c=fe(t)?ba:le.current,c=dt(a,c));var h=t.getDerivedStateFromProps,y=typeof h==="function"||typeof l.getSnapshotBeforeUpdate==="function";y||typeof l.UNSAFE_componentWillReceiveProps!=="function"&&typeof l.componentWillReceiveProps!=="function"||(s!==r||u!==c)&&Fs(a,l,r,c),na=!1;var g=a.memoizedState;l.state=g,xo(a,r,l,o),u=a.memoizedState,s!==r||g!==u||pe.current||na?(typeof h==="function"&&(yl(a,t,h,r),u=a.memoizedState),(s=na||bs(a,t,s,r,g,u,c))?(y||typeof l.UNSAFE_componentWillMount!=="function"&&typeof l.componentWillMount!=="function"||(typeof l.componentWillMount==="function"&&l.componentWillMount(),typeof l.UNSAFE_componentWillMount==="function"&&l.UNSAFE_componentWillMount()),typeof l.componentDidMount==="function"&&(a.flags|=4194308)):(typeof l.componentDidMount==="function"&&(a.flags|=4194308),a.memoizedProps=r,a.memoizedState=u),l.props=r,l.state=u,l.context=c,r=s):(typeof l.componentDidMount==="function"&&(a.flags|=4194308),r=!1)}else{l=a.stateNode,rd(e,a),s=a.memoizedProps,c=a.type===a.elementType?s:_e(a.type,s),l.props=c,y=a.pendingProps,g=l.context,u=t.contextType,typeof u==="object"&&u!==null?u=qe(u):(u=fe(t)?ba:le.current,u=dt(a,u));var k=t.getDerivedStateFromProps;(h=typeof k==="function"||typeof l.getSnapshotBeforeUpdate==="function")||typeof l.UNSAFE_componentWillReceiveProps!=="function"&&typeof l.componentWillReceiveProps!=="function"||(s!==y||g!==u)&&Fs(a,l,r,u),na=!1,g=a.memoizedState,l.state=g,xo(a,r,l,o);var N=a.memoizedState;s!==y||g!==N||pe.current||na?(typeof k==="function"&&(yl(a,t,k,r),N=a.memoizedState),(c=na||bs(a,t,c,r,g,N,u)||!1)?(h||typeof l.UNSAFE_componentWillUpdate!=="function"&&typeof l.componentWillUpdate!=="function"||(typeof l.componentWillUpdate==="function"&&l.componentWillUpdate(r,N,u),typeof l.UNSAFE_componentWillUpdate==="function"&&l.UNSAFE_componentWillUpdate(r,N,u)),typeof l.componentDidUpdate==="function"&&(a.flags|=4),typeof l.getSnapshotBeforeUpdate==="function"&&(a.flags|=1024)):(typeof l.componentDidUpdate!=="function"||s===e.memoizedProps&&g===e.memoizedState||(a.flags|=4),typeof l.getSnapshotBeforeUpdate!=="function"||s===e.memoizedProps&&g===e.memoizedState||(a.flags|=1024),a.memoizedProps=r,a.memoizedState=N),l.props=r,l.state=N,l.context=u,r=c):(typeof l.componentDidUpdate!=="function"||s===e.memoizedProps&&g===e.memoizedState||(a.flags|=4),typeof l.getSnapshotBeforeUpdate!=="function"||s===e.memoizedProps&&g===e.memoizedState||(a.flags|=1024),r=!1)}return Nl(e,a,t,r,n,o)}function Nl(e,a,t,r,o,n){Pd(e,a);var l=(a.flags&128)!==0;if(!r&&!l)return o&&xs(a,t,!1),Je(e,a,n);r=a.stateNode,bf.current=a;var s=l&&typeof t.getDerivedStateFromError!=="function"?null:r.render();return a.flags|=1,e!==null&&l?(a.child=mt(a,e.child,null,n),a.child=mt(a,null,s,n)):se(e,a,s,n),a.memoizedState=r.state,o&&xs(a,t,!0),a.child}function _d(e){var a=e.stateNode;a.pendingContext?qs(e,a.pendingContext,a.pendingContext!==a.context):a.context&&qs(e,a.context,!1),oi(e,a.containerInfo)}function Qs(e,a,t,r,o){return ct(),Jl(o),a.flags|=256,se(e,a,t,r),a.child}function Cl(e){return{baseLanes:e,cachePool:null,transitions:null}}function Td(e,a,t){var r=a.pendingProps,o=V.current,n=!1,l=(a.flags&128)!==0,s;if((s=l)||(s=e!==null&&e.memoizedState===null?!1:(o&2)!==0),s)n=!0,a.flags&=-129;else if(e===null||e.memoizedState!==null)o|=1;if(F(V,o&1),e===null){if(hl(a),e=a.memoizedState,e!==null&&(e=e.dehydrated,e!==null))return(a.mode&1)===0?a.lanes=1:e.data==="$!"?a.lanes=8:a.lanes=1073741824,null;return l=r.children,e=r.fallback,n?(r=a.mode,n=a.child,l={mode:"hidden",children:l},(r&1)===0&&n!==null?(n.childLanes=0,n.pendingProps=l):n=$o(l,r,0,null),e=Ra(e,r,t,null),n.return=a,e.return=a,n.sibling=e,a.child=n,a.child.memoizedState=Cl(t),a.memoizedState=wl,e):mi(a,l)}if(o=e.memoizedState,o!==null&&(s=o.dehydrated,s!==null))return Ff(e,a,l,r,s,o,t);if(n){n=r.fallback,l=a.mode,o=e.child,s=o.sibling;var u={mode:"hidden",children:r.children};return(l&1)===0&&a.child!==o?(r=a.child,r.childLanes=0,r.pendingProps=u,a.deletions=null):(r=ga(o,u),r.subtreeFlags=o.subtreeFlags&14680064),s!==null?n=ga(s,n):(n=Ra(n,l,t,null),n.flags|=2),n.return=a,r.return=a,r.sibling=n,a.child=r,r=n,n=a.child,l=e.child.memoizedState,l=l===null?Cl(t):{baseLanes:l.baseLanes|t,cachePool:null,transitions:l.transitions},n.memoizedState=l,n.childLanes=e.childLanes&~t,a.memoizedState=wl,r}return n=e.child,e=n.sibling,r=ga(n,{mode:"visible",children:r.children}),(a.mode&1)===0&&(r.lanes=t),r.return=a,r.sibling=null,e!==null&&(t=a.deletions,t===null?(a.deletions=[e],a.flags|=16):t.push(e)),a.child=r,a.memoizedState=null,r}function mi(e,a){return a=$o({mode:"visible",children:a},e.mode,0,null),a.return=e,e.child=a}function Gr(e,a,t,r){return r!==null&&Jl(r),mt(a,e.child,null,t),e=mi(a,a.pendingProps.children),e.flags|=2,a.memoizedState=null,e}function Ff(e,a,t,r,o,n,l){if(t){if(a.flags&256)return a.flags&=-257,r=In(Error(S(422))),Gr(e,a,l,r);if(a.memoizedState!==null)return a.child=e.child,a.flags|=128,null;return n=r.fallback,o=a.mode,r=$o({mode:"visible",children:r.children},o,0,null),n=Ra(n,o,l,null),n.flags|=2,r.return=a,n.return=a,r.sibling=n,a.child=r,(a.mode&1)!==0&&mt(a,e.child,null,l),a.child.memoizedState=Cl(l),a.memoizedState=wl,n}if((a.mode&1)===0)return Gr(e,a,l,null);if(o.data==="$!"){if(r=o.nextSibling&&o.nextSibling.dataset,r)var s=r.dgst;return r=s,n=Error(S(419)),r=In(n,r,void 0),Gr(e,a,l,r)}if(s=(l&e.childLanes)!==0,me||s){if(r=Z,r!==null){switch(l&-l){case 4:o=2;break;case 16:o=8;break;case 64:case 128:case 256:case 512:case 1024:case 2048:case 4096:case 8192:case 16384:case 32768:case 65536:case 131072:case 262144:case 524288:case 1048576:case 2097152:case 4194304:case 8388608:case 16777216:case 33554432:case 67108864:o=32;break;case 536870912:o=268435456;break;default:o=0}o=(o&(r.suspendedLanes|l))!==0?0:o,o!==0&&o!==n.retryLane&&(n.retryLane=o,Ze(e,o),be(r,e,o,-1))}return yi(),r=In(Error(S(421))),Gr(e,a,l,r)}if(o.data==="$?")return a.flags|=128,a.child=e.child,a=Yf.bind(null,e),o._reactRetry=a,null;return e=n.treeContext,ge=pa(o.nextSibling),ye=a,L=!0,Oe=null,e!==null&&(ke[Ne++]=We,ke[Ne++]=Ke,ke[Ne++]=Fa,We=e.id,Ke=e.overflow,Fa=a),a=mi(a,r.children),a.flags|=4096,a}function Us(e,a,t){e.lanes|=a;var r=e.alternate;r!==null&&(r.lanes|=a),gl(e.return,a,t)}function Dn(e,a,t,r,o){var n=e.memoizedState;n===null?e.memoizedState={isBackwards:a,rendering:null,renderingStartTime:0,last:r,tail:t,tailMode:o}:(n.isBackwards=a,n.rendering=null,n.renderingStartTime=0,n.last=r,n.tail=t,n.tailMode=o)}function Od(e,a,t){var r=a.pendingProps,o=r.revealOrder,n=r.tail;if(se(e,a,r.children,t),r=V.current,(r&2)!==0)r=r&1|2,a.flags|=128;else{if(e!==null&&(e.flags&128)!==0)e:for(e=a.child;e!==null;){if(e.tag===13)e.memoizedState!==null&&Us(e,t,a);else if(e.tag===19)Us(e,t,a);else if(e.child!==null){e.child.return=e,e=e.child;continue}if(e===a)break e;for(;e.sibling===null;){if(e.return===null||e.return===a)break e;e=e.return}e.sibling.return=e.return,e=e.sibling}r&=1}if(F(V,r),(a.mode&1)===0)a.memoizedState=null;else switch(o){case"forwards":t=a.child;for(o=null;t!==null;)e=t.alternate,e!==null&&Mo(e)===null&&(o=t),t=t.sibling;t=o,t===null?(o=a.child,a.child=null):(o=t.sibling,t.sibling=null),Dn(a,!1,o,t,n);break;case"backwards":t=null,o=a.child;for(a.child=null;o!==null;){if(e=o.alternate,e!==null&&Mo(e)===null){a.child=o;break}e=o.sibling,o.sibling=t,t=o,o=e}Dn(a,!0,t,null,n);break;case"together":Dn(a,!1,null,null,void 0);break;default:a.memoizedState=null}return a.child}function lo(e,a){(a.mode&1)===0&&e!==null&&(e.alternate=null,a.alternate=null,a.flags|=2)}function Je(e,a,t){if(e!==null&&(a.dependencies=e.dependencies),Da|=a.lanes,(t&a.childLanes)===0)return null;if(e!==null&&a.child!==e.child)throw Error(S(153));if(a.child!==null){e=a.child,t=ga(e,e.pendingProps),a.child=t;for(t.return=a;e.sibling!==null;)e=e.sibling,t=t.sibling=ga(e,e.pendingProps),t.return=a;t.sibling=null}return a.child}function If(e,a,t){switch(a.tag){case 3:_d(a),ct();break;case 5:od(a);break;case 1:fe(a.type)&&ko(a);break;case 4:oi(a,a.stateNode.containerInfo);break;case 10:var r=a.type._context,o=a.memoizedProps.value;F(Co,r._currentValue),r._currentValue=o;break;case 13:if(r=a.memoizedState,r!==null){if(r.dehydrated!==null)return F(V,V.current&1),a.flags|=128,null;if((t&a.child.childLanes)!==0)return Td(e,a,t);return F(V,V.current&1),e=Je(e,a,t),e!==null?e.sibling:null}F(V,V.current&1);break;case 19:if(r=(t&a.childLanes)!==0,(e.flags&128)!==0){if(r)return Od(e,a,t);a.flags|=128}if(o=a.memoizedState,o!==null&&(o.rendering=null,o.tail=null,o.lastEffect=null),F(V,V.current),r)break;else return null;case 22:case 23:return a.lanes=0,zd(e,a,t)}return Je(e,a,t)}function zt(e,a){if(!L)switch(e.tailMode){case"hidden":a=e.tail;for(var t=null;a!==null;)a.alternate!==null&&(t=a),a=a.sibling;t===null?e.tail=null:t.sibling=null;break;case"collapsed":t=e.tail;for(var r=null;t!==null;)t.alternate!==null&&(r=t),t=t.sibling;r===null?a||e.tail===null?e.tail=null:e.tail.sibling=null:r.sibling=null}}function oe(e){var a=e.alternate!==null&&e.alternate.child===e.child,t=0,r=0;if(a)for(var o=e.child;o!==null;)t|=o.lanes|o.childLanes,r|=o.subtreeFlags&14680064,r|=o.flags&14680064,o.return=e,o=o.sibling;else for(o=e.child;o!==null;)t|=o.lanes|o.childLanes,r|=o.subtreeFlags,r|=o.flags,o.return=e,o=o.sibling;return e.subtreeFlags|=r,e.childLanes=t,a}function Df(e,a,t){var r=a.pendingProps;switch(Zl(a),a.tag){case 2:case 16:case 15:case 0:case 11:case 7:case 8:case 12:case 9:case 14:return oe(a),null;case 1:return fe(a.type)&&Eo(),oe(a),null;case 3:if(r=a.stateNode,pt(),A(pe),A(le),li(),r.pendingContext&&(r.context=r.pendingContext,r.pendingContext=null),e===null||e.child===null)Yr(a)?a.flags|=4:e===null||e.memoizedState.isDehydrated&&(a.flags&256)===0||(a.flags|=1024,Oe!==null&&(Rl(Oe),Oe=null));return ql(e,a),oe(a),null;case 5:ni(a);var o=Ta(nr.current);if(t=a.type,e!==null&&a.stateNode!=null)bd(e,a,t,r,o),e.ref!==a.ref&&(a.flags|=512,a.flags|=2097152);else{if(!r){if(a.stateNode===null)throw Error(S(166));return oe(a),null}if(e=Ta(Ve.current),Yr(a)){r=a.stateNode,t=a.type;var n=a.memoizedProps;switch(r[Le]=a,r[rr]=n,e=(a.mode&1)!==0,t){case"dialog":D("cancel",r),D("close",r);break;case"iframe":case"object":case"embed":D("load",r);break;case"video":case"audio":for(o=0;o<Dt.length;o++)D(Dt[o],r);break;case"source":D("error",r);break;case"img":case"image":case"link":D("error",r),D("load",r);break;case"details":D("toggle",r);break;case"input":ji(r,n),D("invalid",r);break;case"select":r._wrapperState={wasMultiple:!!n.multiple},D("invalid",r);break;case"textarea":as(r,n),D("invalid",r)}Gn(t,n),o=null;for(var l in n)if(n.hasOwnProperty(l)){var s=n[l];l==="children"?typeof s==="string"?r.textContent!==s&&(n.suppressHydrationWarning!==!0&&Kr(r.textContent,s,e),o=["children",s]):typeof s==="number"&&r.textContent!==""+s&&(n.suppressHydrationWarning!==!0&&Kr(r.textContent,s,e),o=["children",""+s]):Kt.hasOwnProperty(l)&&s!=null&&l==="onScroll"&&D("scroll",r)}switch(t){case"input":Lr(r),es(r,n,!0);break;case"textarea":Lr(r),ts(r);break;case"select":case"option":break;default:typeof n.onClick==="function"&&(r.onclick=So)}r=o,a.updateQueue=r,r!==null&&(a.flags|=4)}else{l=o.nodeType===9?o:o.ownerDocument,e==="http://www.w3.org/1999/xhtml"&&(e=uu(t)),e==="http://www.w3.org/1999/xhtml"?t==="script"?(e=l.createElement("div"),e.innerHTML="<script><\/script>",e=e.removeChild(e.firstChild)):typeof r.is==="string"?e=l.createElement(t,{is:r.is}):(e=l.createElement(t),t==="select"&&(l=e,r.multiple?l.multiple=!0:r.size&&(l.size=r.size))):e=l.createElementNS(e,t),e[Le]=a,e[rr]=r,Rd(e,a,!1,!1),a.stateNode=e;e:{switch(l=Zn(t,r),t){case"dialog":D("cancel",e),D("close",e),o=r;break;case"iframe":case"object":case"embed":D("load",e),o=r;break;case"video":case"audio":for(o=0;o<Dt.length;o++)D(Dt[o],e);o=r;break;case"source":D("error",e),o=r;break;case"img":case"image":case"link":D("error",e),D("load",e),o=r;break;case"details":D("toggle",e),o=r;break;case"input":ji(e,r),o=$n(e,r),D("invalid",e);break;case"option":o=r;break;case"select":e._wrapperState={wasMultiple:!!r.multiple},o=U({},r,{value:void 0}),D("invalid",e);break;case"textarea":as(e,r),o=Yn(e,r),D("invalid",e);break;default:o=r}Gn(t,o),s=o;for(n in s)if(s.hasOwnProperty(n)){var u=s[n];n==="style"?mu(e,u):n==="dangerouslySetInnerHTML"?(u=u?u.__html:void 0,u!=null&&du(e,u)):n==="children"?typeof u==="string"?(t!=="textarea"||u!=="")&&Yt(e,u):typeof u==="number"&&Yt(e,""+u):n!=="suppressContentEditableWarning"&&n!=="suppressHydrationWarning"&&n!=="autoFocus"&&(Kt.hasOwnProperty(n)?u!=null&&n==="onScroll"&&D("scroll",e):u!=null&&Il(e,n,u,l))}switch(t){case"input":Lr(e),es(e,r,!1);break;case"textarea":Lr(e),ts(e);break;case"option":r.value!=null&&e.setAttribute("value",""+ya(r.value));break;case"select":e.multiple=!!r.multiple,n=r.value,n!=null?rt(e,!!r.multiple,n,!1):r.defaultValue!=null&&rt(e,!!r.multiple,r.defaultValue,!0);break;default:typeof o.onClick==="function"&&(e.onclick=So)}switch(t){case"button":case"input":case"select":case"textarea":r=!!r.autoFocus;break e;case"img":r=!0;break e;default:r=!1}}r&&(a.flags|=4)}a.ref!==null&&(a.flags|=512,a.flags|=2097152)}return oe(a),null;case 6:if(e&&a.stateNode!=null)Fd(e,a,e.memoizedProps,r);else{if(typeof r!=="string"&&a.stateNode===null)throw Error(S(166));if(t=Ta(nr.current),Ta(Ve.current),Yr(a)){if(r=a.stateNode,t=a.memoizedProps,r[Le]=a,n=r.nodeValue!==t){if(e=ye,e!==null)switch(e.tag){case 3:Kr(r.nodeValue,t,(e.mode&1)!==0);break;case 5:e.memoizedProps.suppressHydrationWarning!==!0&&Kr(r.nodeValue,t,(e.mode&1)!==0)}}n&&(a.flags|=4)}else r=(t.nodeType===9?t:t.ownerDocument).createTextNode(r),r[Le]=a,a.stateNode=r}return oe(a),null;case 13:if(A(V),r=a.memoizedState,e===null||e.memoizedState!==null&&e.memoizedState.dehydrated!==null){if(L&&ge!==null&&(a.mode&1)!==0&&(a.flags&128)===0)ju(),ct(),a.flags|=98560,n=!1;else if(n=Yr(a),r!==null&&r.dehydrated!==null){if(e===null){if(!n)throw Error(S(318));if(n=a.memoizedState,n=n!==null?n.dehydrated:null,!n)throw Error(S(317));n[Le]=a}else ct(),(a.flags&128)===0&&(a.memoizedState=null),a.flags|=4;oe(a),n=!1}else Oe!==null&&(Rl(Oe),Oe=null),n=!0;if(!n)return a.flags&65536?a:null}if((a.flags&128)!==0)return a.lanes=t,a;return r=r!==null,r!==(e!==null&&e.memoizedState!==null)&&r&&(a.child.flags|=8192,(a.mode&1)!==0&&(e===null||(V.current&1)!==0?X===0&&(X=3):yi())),a.updateQueue!==null&&(a.flags|=4),oe(a),null;case 4:return pt(),ql(e,a),e===null&&ar(a.stateNode.containerInfo),oe(a),null;case 10:return ai(a.type._context),oe(a),null;case 17:return fe(a.type)&&Eo(),oe(a),null;case 19:if(A(V),n=a.memoizedState,n===null)return oe(a),null;if(r=(a.flags&128)!==0,l=n.rendering,l===null)if(r)zt(n,!1);else{if(X!==0||e!==null&&(e.flags&128)!==0)for(e=a.child;e!==null;){if(l=Mo(e),l!==null){a.flags|=128,zt(n,!1),r=l.updateQueue,r!==null&&(a.updateQueue=r,a.flags|=4),a.subtreeFlags=0,r=t;for(t=a.child;t!==null;)n=t,e=r,n.flags&=14680066,l=n.alternate,l===null?(n.childLanes=0,n.lanes=e,n.child=null,n.subtreeFlags=0,n.memoizedProps=null,n.memoizedState=null,n.updateQueue=null,n.dependencies=null,n.stateNode=null):(n.childLanes=l.childLanes,n.lanes=l.lanes,n.child=l.child,n.subtreeFlags=0,n.deletions=null,n.memoizedProps=l.memoizedProps,n.memoizedState=l.memoizedState,n.updateQueue=l.updateQueue,n.type=l.type,e=l.dependencies,n.dependencies=e===null?null:{lanes:e.lanes,firstContext:e.firstContext}),t=t.sibling;return F(V,V.current&1|2),a.child}e=e.sibling}n.tail!==null&&$()>vt&&(a.flags|=128,r=!0,zt(n,!1),a.lanes=4194304)}else{if(!r)if(e=Mo(l),e!==null){if(a.flags|=128,r=!0,t=e.updateQueue,t!==null&&(a.updateQueue=t,a.flags|=4),zt(n,!0),n.tail===null&&n.tailMode==="hidden"&&!l.alternate&&!L)return oe(a),null}else 2*$()-n.renderingStartTime>vt&&t!==1073741824&&(a.flags|=128,r=!0,zt(n,!1),a.lanes=4194304);n.isBackwards?(l.sibling=a.child,a.child=l):(t=n.last,t!==null?t.sibling=l:a.child=l,n.last=l)}if(n.tail!==null)return a=n.tail,n.rendering=a,n.tail=a.sibling,n.renderingStartTime=$(),a.sibling=null,t=V.current,F(V,r?t&1|2:t&1),a;return oe(a),null;case 22:case 23:return gi(),r=a.memoizedState!==null,e!==null&&e.memoizedState!==null!==r&&(a.flags|=8192),r&&(a.mode&1)!==0?(he&1073741824)!==0&&(oe(a),a.subtreeFlags&6&&(a.flags|=8192)):oe(a),null;case 24:return null;case 25:return null}throw Error(S(156,a.tag))}function Af(e,a){switch(Zl(a),a.tag){case 1:return fe(a.type)&&Eo(),e=a.flags,e&65536?(a.flags=e&-65537|128,a):null;case 3:return pt(),A(pe),A(le),li(),e=a.flags,(e&65536)!==0&&(e&128)===0?(a.flags=e&-65537|128,a):null;case 5:return ni(a),null;case 13:if(A(V),e=a.memoizedState,e!==null&&e.dehydrated!==null){if(a.alternate===null)throw Error(S(340));ct()}return e=a.flags,e&65536?(a.flags=e&-65537|128,a):null;case 19:return A(V),null;case 4:return pt(),null;case 10:return ai(a.type._context),null;case 22:case 23:return gi(),null;case 24:return null;default:return null}}function at(e,a){var t=e.ref;if(t!==null)if(typeof t==="function")try{t(null)}catch(r){H(e,a,r)}else t.current=null}function xl(e,a,t){try{t()}catch(r){H(e,a,r)}}function Bf(e,a){if(ul=ho,e=Bu(),Xl(e)){if("selectionStart"in e)var t={start:e.selectionStart,end:e.selectionEnd};else e:{t=(t=e.ownerDocument)&&t.defaultView||window;var r=t.getSelection&&t.getSelection();if(r&&r.rangeCount!==0){t=r.anchorNode;var{anchorOffset:o,focusNode:n}=r;r=r.focusOffset;try{t.nodeType,n.nodeType}catch(E){t=null;break e}var l=0,s=-1,u=-1,c=0,h=0,y=e,g=null;a:for(;;){for(var k;;){if(y!==t||o!==0&&y.nodeType!==3||(s=l+o),y!==n||r!==0&&y.nodeType!==3||(u=l+r),y.nodeType===3&&(l+=y.nodeValue.length),(k=y.firstChild)===null)break;g=y,y=k}for(;;){if(y===e)break a;if(g===t&&++c===o&&(s=l),g===n&&++h===r&&(u=l),(k=y.nextSibling)!==null)break;y=g,g=y.parentNode}y=k}t=s===-1||u===-1?null:{start:s,end:u}}else t=null}t=t||{start:0,end:0}}else t=null;dl={focusedElem:e,selectionRange:t},ho=!1;for(w=a;w!==null;)if(a=w,e=a.child,(a.subtreeFlags&1028)!==0&&e!==null)e.return=a,w=e;else for(;w!==null;){a=w;try{var N=a.alternate;if((a.flags&1024)!==0)switch(a.tag){case 0:case 11:case 15:break;case 1:if(N!==null){var{memoizedProps:C,memoizedState:b}=N,m=a.stateNode,d=m.getSnapshotBeforeUpdate(a.elementType===a.type?C:_e(a.type,C),b);m.__reactInternalSnapshotBeforeUpdate=d}break;case 3:var f=a.stateNode.containerInfo;f.nodeType===1?f.textContent="":f.nodeType===9&&f.documentElement&&f.removeChild(f.documentElement);break;case 5:case 6:case 4:case 17:break;default:throw Error(S(163))}}catch(E){H(a,a.return,E)}if(e=a.sibling,e!==null){e.return=a.return,w=e;break}w=a.return}return N=Hs,Hs=!1,N}function Ht(e,a,t){var r=a.updateQueue;if(r=r!==null?r.lastEffect:null,r!==null){var o=r=r.next;do{if((o.tag&e)===e){var n=o.destroy;o.destroy=void 0,n!==void 0&&xl(a,t,n)}o=o.next}while(o!==r)}}function Uo(e,a){if(a=a.updateQueue,a=a!==null?a.lastEffect:null,a!==null){var t=a=a.next;do{if((t.tag&e)===e){var r=t.create;t.destroy=r()}t=t.next}while(t!==a)}}function Ml(e){var a=e.ref;if(a!==null){var t=e.stateNode;switch(e.tag){case 5:e=t;break;default:e=t}typeof a==="function"?a(e):a.current=e}}function Id(e){var a=e.alternate;a!==null&&(e.alternate=null,Id(a)),e.child=null,e.deletions=null,e.sibling=null,e.tag===5&&(a=e.stateNode,a!==null&&(delete a[Le],delete a[rr],delete a[pl],delete a[Nf],delete a[wf])),e.stateNode=null,e.return=null,e.dependencies=null,e.memoizedProps=null,e.memoizedState=null,e.pendingProps=null,e.stateNode=null,e.updateQueue=null}function Dd(e){return e.tag===5||e.tag===3||e.tag===4}function $s(e){e:for(;;){for(;e.sibling===null;){if(e.return===null||Dd(e.return))return null;e=e.return}e.sibling.return=e.return;for(e=e.sibling;e.tag!==5&&e.tag!==6&&e.tag!==18;){if(e.flags&2)continue e;if(e.child===null||e.tag===4)continue e;else e.child.return=e,e=e.child}if(!(e.flags&2))return e.stateNode}}function zl(e,a,t){var r=e.tag;if(r===5||r===6)e=e.stateNode,a?t.nodeType===8?t.parentNode.insertBefore(e,a):t.insertBefore(e,a):(t.nodeType===8?(a=t.parentNode,a.insertBefore(e,t)):(a=t,a.appendChild(e)),t=t._reactRootContainer,t!==null&&t!==void 0||a.onclick!==null||(a.onclick=So));else if(r!==4&&(e=e.child,e!==null))for(zl(e,a,t),e=e.sibling;e!==null;)zl(e,a,t),e=e.sibling}function Pl(e,a,t){var r=e.tag;if(r===5||r===6)e=e.stateNode,a?t.insertBefore(e,a):t.appendChild(e);else if(r!==4&&(e=e.child,e!==null))for(Pl(e,a,t),e=e.sibling;e!==null;)Pl(e,a,t),e=e.sibling}function ra(e,a,t){for(t=t.child;t!==null;)Ad(e,a,t),t=t.sibling}function Ad(e,a,t){if(Be&&typeof Be.onCommitFiberUnmount==="function")try{Be.onCommitFiberUnmount(Fo,t)}catch(s){}switch(t.tag){case 5:ne||at(t,a);case 6:var r=J,o=Te;J=null,ra(e,a,t),J=r,Te=o,J!==null&&(Te?(e=J,t=t.stateNode,e.nodeType===8?e.parentNode.removeChild(t):e.removeChild(t)):J.removeChild(t.stateNode));break;case 18:J!==null&&(Te?(e=J,t=t.stateNode,e.nodeType===8?_n(e.parentNode,t):e.nodeType===1&&_n(e,t),Jt(e)):_n(J,t.stateNode));break;case 4:r=J,o=Te,J=t.stateNode.containerInfo,Te=!0,ra(e,a,t),J=r,Te=o;break;case 0:case 11:case 14:case 15:if(!ne&&(r=t.updateQueue,r!==null&&(r=r.lastEffect,r!==null))){o=r=r.next;do{var n=o,l=n.destroy;n=n.tag,l!==void 0&&((n&2)!==0?xl(t,a,l):(n&4)!==0&&xl(t,a,l)),o=o.next}while(o!==r)}ra(e,a,t);break;case 1:if(!ne&&(at(t,a),r=t.stateNode,typeof r.componentWillUnmount==="function"))try{r.props=t.memoizedProps,r.state=t.memoizedState,r.componentWillUnmount()}catch(s){H(t,a,s)}ra(e,a,t);break;case 21:ra(e,a,t);break;case 22:t.mode&1?(ne=(r=ne)||t.memoizedState!==null,ra(e,a,t),ne=r):ra(e,a,t);break;default:ra(e,a,t)}}function Ws(e){var a=e.updateQueue;if(a!==null){e.updateQueue=null;var t=e.stateNode;t===null&&(t=e.stateNode=new Lf),a.forEach(function(r){var o=Xf.bind(null,e,r);t.has(r)||(t.add(r),r.then(o,o))})}}function Pe(e,a){var t=a.deletions;if(t!==null)for(var r=0;r<t.length;r++){var o=t[r];try{var n=e,l=a,s=l;e:for(;s!==null;){switch(s.tag){case 5:J=s.stateNode,Te=!1;break e;case 3:J=s.stateNode.containerInfo,Te=!0;break e;case 4:J=s.stateNode.containerInfo,Te=!0;break e}s=s.return}if(J===null)throw Error(S(160));Ad(n,l,o),J=null,Te=!1;var u=o.alternate;u!==null&&(u.return=null),o.return=null}catch(c){H(o,a,c)}}if(a.subtreeFlags&12854)for(a=a.child;a!==null;)Ld(a,e),a=a.sibling}function Ld(e,a){var{alternate:t,flags:r}=e;switch(e.tag){case 0:case 11:case 14:case 15:if(Pe(a,e),De(e),r&4){try{Ht(3,e,e.return),Uo(3,e)}catch(C){H(e,e.return,C)}try{Ht(5,e,e.return)}catch(C){H(e,e.return,C)}}break;case 1:Pe(a,e),De(e),r&512&&t!==null&&at(t,t.return);break;case 5:if(Pe(a,e),De(e),r&512&&t!==null&&at(t,t.return),e.flags&32){var o=e.stateNode;try{Yt(o,"")}catch(C){H(e,e.return,C)}}if(r&4&&(o=e.stateNode,o!=null)){var n=e.memoizedProps,l=t!==null?t.memoizedProps:n,s=e.type,u=e.updateQueue;if(e.updateQueue=null,u!==null)try{s==="input"&&n.type==="radio"&&n.name!=null&&iu(o,n),Zn(s,l);var c=Zn(s,n);for(l=0;l<u.length;l+=2){var h=u[l],y=u[l+1];h==="style"?mu(o,y):h==="dangerouslySetInnerHTML"?du(o,y):h==="children"?Yt(o,y):Il(o,h,y,c)}switch(s){case"input":Wn(o,n);break;case"textarea":su(o,n);break;case"select":var g=o._wrapperState.wasMultiple;o._wrapperState.wasMultiple=!!n.multiple;var k=n.value;k!=null?rt(o,!!n.multiple,k,!1):g!==!!n.multiple&&(n.defaultValue!=null?rt(o,!!n.multiple,n.defaultValue,!0):rt(o,!!n.multiple,n.multiple?[]:"",!1))}o[rr]=n}catch(C){H(e,e.return,C)}}break;case 6:if(Pe(a,e),De(e),r&4){if(e.stateNode===null)throw Error(S(162));o=e.stateNode,n=e.memoizedProps;try{o.nodeValue=n}catch(C){H(e,e.return,C)}}break;case 3:if(Pe(a,e),De(e),r&4&&t!==null&&t.memoizedState.isDehydrated)try{Jt(a.containerInfo)}catch(C){H(e,e.return,C)}break;case 4:Pe(a,e),De(e);break;case 13:Pe(a,e),De(e),o=e.child,o.flags&8192&&(n=o.memoizedState!==null,o.stateNode.isHidden=n,!n||o.alternate!==null&&o.alternate.memoizedState!==null||(vi=$())),r&4&&Ws(e);break;case 22:if(h=t!==null&&t.memoizedState!==null,e.mode&1?(ne=(c=ne)||h,Pe(a,e),ne=c):Pe(a,e),De(e),r&8192){if(c=e.memoizedState!==null,(e.stateNode.isHidden=c)&&!h&&(e.mode&1)!==0)for(w=e,h=e.child;h!==null;){for(y=w=h;w!==null;){switch(g=w,k=g.child,g.tag){case 0:case 11:case 14:case 15:Ht(4,g,g.return);break;case 1:at(g,g.return);var N=g.stateNode;if(typeof N.componentWillUnmount==="function"){r=g,t=g.return;try{a=r,N.props=a.memoizedProps,N.state=a.memoizedState,N.componentWillUnmount()}catch(C){H(r,t,C)}}break;case 5:at(g,g.return);break;case 22:if(g.memoizedState!==null){Ys(y);continue}}k!==null?(k.return=g,w=k):Ys(y)}h=h.sibling}e:for(h=null,y=e;;){if(y.tag===5){if(h===null){h=y;try{o=y.stateNode,c?(n=o.style,typeof n.setProperty==="function"?n.setProperty("display","none","important"):n.display="none"):(s=y.stateNode,u=y.memoizedProps.style,l=u!==void 0&&u!==null&&u.hasOwnProperty("display")?u.display:null,s.style.display=cu("display",l))}catch(C){H(e,e.return,C)}}}else if(y.tag===6){if(h===null)try{y.stateNode.nodeValue=c?"":y.memoizedProps}catch(C){H(e,e.return,C)}}else if((y.tag!==22&&y.tag!==23||y.memoizedState===null||y===e)&&y.child!==null){y.child.return=y,y=y.child;continue}if(y===e)break e;for(;y.sibling===null;){if(y.return===null||y.return===e)break e;h===y&&(h=null),y=y.return}h===y&&(h=null),y.sibling.return=y.return,y=y.sibling}}break;case 19:Pe(a,e),De(e),r&4&&Ws(e);break;case 21:break;default:Pe(a,e),De(e)}}function De(e){var a=e.flags;if(a&2){try{e:{for(var t=e.return;t!==null;){if(Dd(t)){var r=t;break e}t=t.return}throw Error(S(160))}switch(r.tag){case 5:var o=r.stateNode;r.flags&32&&(Yt(o,""),r.flags&=-33);var n=$s(e);Pl(e,n,o);break;case 3:case 4:var l=r.stateNode.containerInfo,s=$s(e);zl(e,s,l);break;default:throw Error(S(161))}}catch(u){H(e,e.return,u)}e.flags&=-3}a&4096&&(e.flags&=-4097)}function Vf(e,a,t){w=e,Bd(e,a,t)}function Bd(e,a,t){for(var r=(e.mode&1)!==0;w!==null;){var o=w,n=o.child;if(o.tag===22&&r){var l=o.memoizedState!==null||Zr;if(!l){var s=o.alternate,u=s!==null&&s.memoizedState!==null||ne;s=Zr;var c=ne;if(Zr=l,(ne=u)&&!c)for(w=o;w!==null;)l=w,u=l.child,l.tag===22&&l.memoizedState!==null?Xs(o):u!==null?(u.return=l,w=u):Xs(o);for(;n!==null;)w=n,Bd(n,a,t),n=n.sibling;w=o,Zr=s,ne=c}Ks(e,a,t)}else(o.subtreeFlags&8772)!==0&&n!==null?(n.return=o,w=n):Ks(e,a,t)}}function Ks(e){for(;w!==null;){var a=w;if((a.flags&8772)!==0){var t=a.alternate;try{if((a.flags&8772)!==0)switch(a.tag){case 0:case 11:case 15:ne||Uo(5,a);break;case 1:var r=a.stateNode;if(a.flags&4&&!ne)if(t===null)r.componentDidMount();else{var o=a.elementType===a.type?t.memoizedProps:_e(a.type,t.memoizedProps);r.componentDidUpdate(o,t.memoizedState,r.__reactInternalSnapshotBeforeUpdate)}var n=a.updateQueue;n!==null&&Ts(a,n,r);break;case 3:var l=a.updateQueue;if(l!==null){if(t=null,a.child!==null)switch(a.child.tag){case 5:t=a.child.stateNode;break;case 1:t=a.child.stateNode}Ts(a,l,t)}break;case 5:var s=a.stateNode;if(t===null&&a.flags&4){t=s;var u=a.memoizedProps;switch(a.type){case"button":case"input":case"select":case"textarea":u.autoFocus&&t.focus();break;case"img":u.src&&(t.src=u.src)}}break;case 6:break;case 4:break;case 12:break;case 13:if(a.memoizedState===null){var c=a.alternate;if(c!==null){var h=c.memoizedState;if(h!==null){var y=h.dehydrated;y!==null&&Jt(y)}}}break;case 19:case 17:case 21:case 22:case 23:case 25:break;default:throw Error(S(163))}ne||a.flags&512&&Ml(a)}catch(g){H(a,a.return,g)}}if(a===e){w=null;break}if(t=a.sibling,t!==null){t.return=a.return,w=t;break}w=a.return}}function Ys(e){for(;w!==null;){var a=w;if(a===e){w=null;break}var t=a.sibling;if(t!==null){t.return=a.return,w=t;break}w=a.return}}function Xs(e){for(;w!==null;){var a=w;try{switch(a.tag){case 0:case 11:case 15:var t=a.return;try{Uo(4,a)}catch(u){H(a,t,u)}break;case 1:var r=a.stateNode;if(typeof r.componentDidMount==="function"){var o=a.return;try{r.componentDidMount()}catch(u){H(a,o,u)}}var n=a.return;try{Ml(a)}catch(u){H(a,n,u)}break;case 5:var l=a.return;try{Ml(a)}catch(u){H(a,l,u)}}}catch(u){H(a,a.return,u)}if(a===e){w=null;break}var s=a.sibling;if(s!==null){s.return=a.return,w=s;break}w=a.return}}function ue(){return(O&6)!==0?$():io!==-1?io:io=$()}function ha(e){if((e.mode&1)===0)return 1;if((O&2)!==0&&j!==0)return j&-j;if(qf.transition!==null)return so===0&&(so=Cu()),so;if(e=R,e!==0)return e;return e=window.event,e=e===void 0?16:Tu(e.type),e}function be(e,a,t,r){if(50<Wt)throw Wt=0,Tl=null,Error(S(185));if(dr(e,t,r),(O&2)===0||e!==Z)e===Z&&((O&2)===0&&(Ho|=t),X===4&&ia(e,j)),ve(e,r),t===1&&O===0&&(a.mode&1)===0&&(vt=$()+500,Bo&&Na())}function ve(e,a){var t=e.callbackNode;xp(e,a);var r=vo(e,e===Z?j:0);if(r===0)t!==null&&ns(t),e.callbackNode=null,e.callbackPriority=0;else if(a=r&-r,e.callbackPriority!==a){if(t!=null&&ns(t),a===1)e.tag===0?Cf(Gs.bind(null,e)):Gu(Gs.bind(null,e)),Ef(function(){(O&6)===0&&Na()}),t=null;else{switch(qu(r)){case 1:t=Vl;break;case 4:t=Nu;break;case 16:t=fo;break;case 536870912:t=wu;break;default:t=fo}t=Yd(t,Vd.bind(null,e))}e.callbackPriority=a,e.callbackNode=t}}function Vd(e,a){if(io=-1,so=0,(O&6)!==0)throw Error(S(327));var t=e.callbackNode;if(st()&&e.callbackNode!==t)return null;var r=vo(e,e===Z?j:0);if(r===0)return null;if((r&30)!==0||(r&e.expiredLanes)!==0||a)a=Ro(e,r);else{a=r;var o=O;O|=2;var n=Ud();if(Z!==e||j!==a)He=null,vt=$()+500,Oa(e,a);do try{$f();break}catch(s){Qd(e,s)}while(1);ei(),_o.current=n,O=o,K!==null?a=0:(Z=null,j=0,a=X)}if(a!==0){if(a===2&&(o=tl(e),o!==0&&(r=o,a=Ol(e,o))),a===1)throw t=ur,Oa(e,0),ia(e,r),ve(e,$()),t;if(a===6)ia(e,r);else{if(o=e.current.alternate,(r&30)===0&&!Uf(o)&&(a=Ro(e,r),a===2&&(n=tl(e),n!==0&&(r=n,a=Ol(e,n))),a===1))throw t=ur,Oa(e,0),ia(e,r),ve(e,$()),t;switch(e.finishedWork=o,e.finishedLanes=r,a){case 0:case 1:throw Error(S(345));case 2:za(e,ce,He);break;case 3:if(ia(e,r),(r&130023424)===r&&(a=vi+500-$(),10<a)){if(vo(e,0)!==0)break;if(o=e.suspendedLanes,(o&r)!==r){ue(),e.pingedLanes|=e.suspendedLanes&o;break}e.timeoutHandle=ml(za.bind(null,e,ce,He),a);break}za(e,ce,He);break;case 4:if(ia(e,r),(r&4194240)===r)break;a=e.eventTimes;for(o=-1;0<r;){var l=31-Re(r);n=1<<l,l=a[l],l>o&&(o=l),r&=~n}if(r=o,r=$()-r,r=(120>r?120:480>r?480:1080>r?1080:1920>r?1920:3000>r?3000:4320>r?4320:1960*Qf(r/1960))-r,10<r){e.timeoutHandle=ml(za.bind(null,e,ce,He),r);break}za(e,ce,He);break;case 5:za(e,ce,He);break;default:throw Error(S(329))}}}return ve(e,$()),e.callbackNode===t?Vd.bind(null,e):null}function Ol(e,a){var t=$t;return e.current.memoizedState.isDehydrated&&(Oa(e,a).flags|=256),e=Ro(e,a),e!==2&&(a=ce,ce=t,a!==null&&Rl(a)),e}function Rl(e){ce===null?ce=e:ce.push.apply(ce,e)}function Uf(e){for(var a=e;;){if(a.flags&16384){var t=a.updateQueue;if(t!==null&&(t=t.stores,t!==null))for(var r=0;r<t.length;r++){var o=t[r],n=o.getSnapshot;o=o.value;try{if(!Fe(n(),o))return!1}catch(l){return!1}}}if(t=a.child,a.subtreeFlags&16384&&t!==null)t.return=a,a=t;else{if(a===e)break;for(;a.sibling===null;){if(a.return===null||a.return===e)return!0;a=a.return}a.sibling.return=a.return,a=a.sibling}}return!0}function ia(e,a){a&=~fi,a&=~Ho,e.suspendedLanes|=a,e.pingedLanes&=~a;for(e=e.expirationTimes;0<a;){var t=31-Re(a),r=1<<t;e[t]=-1,a&=~r}}function Gs(e){if((O&6)!==0)throw Error(S(327));st();var a=vo(e,0);if((a&1)===0)return ve(e,$()),null;var t=Ro(e,a);if(e.tag!==0&&t===2){var r=tl(e);r!==0&&(a=r,t=Ol(e,r))}if(t===1)throw t=ur,Oa(e,0),ia(e,a),ve(e,$()),t;if(t===6)throw Error(S(345));return e.finishedWork=e.current.alternate,e.finishedLanes=a,za(e,ce,He),ve(e,$()),null}function hi(e,a){var t=O;O|=1;try{return e(a)}finally{O=t,O===0&&(vt=$()+500,Bo&&Na())}}function Aa(e){ua!==null&&ua.tag===0&&(O&6)===0&&st();var a=O;O|=1;var t=Ce.transition,r=R;try{if(Ce.transition=null,R=1,e)return e()}finally{R=r,Ce.transition=t,O=a,(O&6)===0&&Na()}}function gi(){he=tt.current,A(tt)}function Oa(e,a){e.finishedWork=null,e.finishedLanes=0;var t=e.timeoutHandle;if(t!==-1&&(e.timeoutHandle=-1,Sf(t)),K!==null)for(t=K.return;t!==null;){var r=t;switch(Zl(r),r.tag){case 1:r=r.type.childContextTypes,r!==null&&r!==void 0&&Eo();break;case 3:pt(),A(pe),A(le),li();break;case 5:ni(r);break;case 4:pt();break;case 13:A(V);break;case 19:A(V);break;case 10:ai(r.type._context);break;case 22:case 23:gi()}t=t.return}if(Z=e,K=e=ga(e.current,null),j=he=a,X=0,ur=null,fi=Ho=Da=0,ce=$t=null,_a!==null){for(a=0;a<_a.length;a++)if(t=_a[a],r=t.interleaved,r!==null){t.interleaved=null;var o=r.next,n=t.pending;if(n!==null){var l=n.next;n.next=o,r.next=l}t.pending=r}_a=null}return e}function Qd(e,a){do{var t=K;try{if(ei(),oo.current=Po,zo){for(var r=Q.memoizedState;r!==null;){var o=r.queue;o!==null&&(o.pending=null),r=r.next}zo=!1}if(Ia=0,G=Y=Q=null,Ut=!1,lr=0,pi.current=null,t===null||t.return===null){X=1,ur=a,K=null;break}e:{var n=e,l=t.return,s=t,u=a;if(a=j,s.flags|=32768,u!==null&&typeof u==="object"&&typeof u.then==="function"){var c=u,h=s,y=h.tag;if((h.mode&1)===0&&(y===0||y===11||y===15)){var g=h.alternate;g?(h.updateQueue=g.updateQueue,h.memoizedState=g.memoizedState,h.lanes=g.lanes):(h.updateQueue=null,h.memoizedState=null)}var k=Ds(l);if(k!==null){k.flags&=-257,As(k,l,s,n,a),k.mode&1&&Is(n,c,a),a=k,u=c;var N=a.updateQueue;if(N===null){var C=new Set;C.add(u),a.updateQueue=C}else N.add(u);break e}else{if((a&1)===0){Is(n,c,a),yi();break e}u=Error(S(426))}}else if(L&&s.mode&1){var b=Ds(l);if(b!==null){(b.flags&65536)===0&&(b.flags|=256),As(b,l,s,n,a),Jl(ft(u,s));break e}}n=u=ft(u,s),X!==4&&(X=2),$t===null?$t=[n]:$t.push(n),n=l;do{switch(n.tag){case 3:n.flags|=65536,a&=-a,n.lanes|=a;var m=qd(n,u,a);_s(n,m);break e;case 1:s=u;var{type:d,stateNode:f}=n;if((n.flags&128)===0&&(typeof d.getDerivedStateFromError==="function"||f!==null&&typeof f.componentDidCatch==="function"&&(va===null||!va.has(f)))){n.flags|=65536,a&=-a,n.lanes|=a;var E=xd(n,s,a);_s(n,E);break e}}n=n.return}while(n!==null)}$d(t)}catch(q){a=q,K===t&&t!==null&&(K=t=t.return);continue}break}while(1)}function Ud(){var e=_o.current;return _o.current=Po,e===null?Po:e}function yi(){if(X===0||X===3||X===2)X=4;Z===null||(Da&268435455)===0&&(Ho&268435455)===0||ia(Z,j)}function Ro(e,a){var t=O;O|=2;var r=Ud();if(Z!==e||j!==a)He=null,Oa(e,a);do try{Hf();break}catch(o){Qd(e,o)}while(1);if(ei(),O=t,_o.current=r,K!==null)throw Error(S(261));return Z=null,j=0,X}function Hf(){for(;K!==null;)Hd(K)}function $f(){for(;K!==null&&!gp();)Hd(K)}function Hd(e){var a=Kd(e.alternate,e,he);e.memoizedProps=e.pendingProps,a===null?$d(e):K=a,pi.current=null}function $d(e){var a=e;do{var t=a.alternate;if(e=a.return,(a.flags&32768)===0){if(t=Df(t,a,he),t!==null){K=t;return}}else{if(t=Af(t,a),t!==null){t.flags&=32767,K=t;return}if(e!==null)e.flags|=32768,e.subtreeFlags=0,e.deletions=null;else{X=6,K=null;return}}if(a=a.sibling,a!==null){K=a;return}K=a=e}while(a!==null);X===0&&(X=5)}function za(e,a,t){var r=R,o=Ce.transition;try{Ce.transition=null,R=1,Wf(e,a,t,r)}finally{Ce.transition=o,R=r}return null}function Wf(e,a,t,r){do st();while(ua!==null);if((O&6)!==0)throw Error(S(327));t=e.finishedWork;var o=e.finishedLanes;if(t===null)return null;if(e.finishedWork=null,e.finishedLanes=0,t===e.current)throw Error(S(177));e.callbackNode=null,e.callbackPriority=0;var n=t.lanes|t.childLanes;if(Mp(e,n),e===Z&&(K=Z=null,j=0),(t.subtreeFlags&2064)===0&&(t.flags&2064)===0||Jr||(Jr=!0,Yd(fo,function(){return st(),null})),n=(t.flags&15990)!==0,(t.subtreeFlags&15990)!==0||n){n=Ce.transition,Ce.transition=null;var l=R;R=1;var s=O;O|=4,pi.current=null,Bf(e,t),Ld(t,e),ff(dl),ho=!!ul,dl=ul=null,e.current=t,Vf(t,e,o),yp(),O=s,R=l,Ce.transition=n}else e.current=t;if(Jr&&(Jr=!1,ua=e,Oo=o),n=e.pendingLanes,n===0&&(va=null),kp(t.stateNode,r),ve(e,$()),a!==null)for(r=e.onRecoverableError,t=0;t<a.length;t++)o=a[t],r(o.value,{componentStack:o.stack,digest:o.digest});if(To)throw To=!1,e=_l,_l=null,e;return(Oo&1)!==0&&e.tag!==0&&st(),n=e.pendingLanes,(n&1)!==0?e===Tl?Wt++:(Wt=0,Tl=e):Wt=0,Na(),null}function st(){if(ua!==null){var e=qu(Oo),a=Ce.transition,t=R;try{if(Ce.transition=null,R=16>e?16:e,ua===null)var r=!1;else{if(e=ua,ua=null,Oo=0,(O&6)!==0)throw Error(S(331));var o=O;O|=4;for(w=e.current;w!==null;){var n=w,l=n.child;if((w.flags&16)!==0){var s=n.deletions;if(s!==null){for(var u=0;u<s.length;u++){var c=s[u];for(w=c;w!==null;){var h=w;switch(h.tag){case 0:case 11:case 15:Ht(8,h,n)}var y=h.child;if(y!==null)y.return=h,w=y;else for(;w!==null;){h=w;var{sibling:g,return:k}=h;if(Id(h),h===c){w=null;break}if(g!==null){g.return=k,w=g;break}w=k}}}var N=n.alternate;if(N!==null){var C=N.child;if(C!==null){N.child=null;do{var b=C.sibling;C.sibling=null,C=b}while(C!==null)}}w=n}}if((n.subtreeFlags&2064)!==0&&l!==null)l.return=n,w=l;else e:for(;w!==null;){if(n=w,(n.flags&2048)!==0)switch(n.tag){case 0:case 11:case 15:Ht(9,n,n.return)}var m=n.sibling;if(m!==null){m.return=n.return,w=m;break e}w=n.return}}var d=e.current;for(w=d;w!==null;){l=w;var f=l.child;if((l.subtreeFlags&2064)!==0&&f!==null)f.return=l,w=f;else e:for(l=d;w!==null;){if(s=w,(s.flags&2048)!==0)try{switch(s.tag){case 0:case 11:case 15:Uo(9,s)}}catch(q){H(s,s.return,q)}if(s===l){w=null;break e}var E=s.sibling;if(E!==null){E.return=s.return,w=E;break e}w=s.return}}if(O=o,Na(),Be&&typeof Be.onPostCommitFiberRoot==="function")try{Be.onPostCommitFiberRoot(Fo,e)}catch(q){}r=!0}return r}finally{R=t,Ce.transition=a}}return!1}function Zs(e,a,t){a=ft(t,a),a=qd(e,a,1),e=fa(e,a,1),a=ue(),e!==null&&(dr(e,1,a),ve(e,a))}function H(e,a,t){if(e.tag===3)Zs(e,e,t);else for(;a!==null;){if(a.tag===3){Zs(a,e,t);break}else if(a.tag===1){var r=a.stateNode;if(typeof a.type.getDerivedStateFromError==="function"||typeof r.componentDidCatch==="function"&&(va===null||!va.has(r))){e=ft(t,e),e=xd(a,e,1),a=fa(a,e,1),e=ue(),a!==null&&(dr(a,1,e),ve(a,e));break}}a=a.return}}function Kf(e,a,t){var r=e.pingCache;r!==null&&r.delete(a),a=ue(),e.pingedLanes|=e.suspendedLanes&t,Z===e&&(j&t)===t&&(X===4||X===3&&(j&130023424)===j&&500>$()-vi?Oa(e,0):fi|=t),ve(e,a)}function Wd(e,a){a===0&&((e.mode&1)===0?a=1:(a=Qr,Qr<<=1,(Qr&130023424)===0&&(Qr=4194304)));var t=ue();e=Ze(e,a),e!==null&&(dr(e,a,t),ve(e,t))}function Yf(e){var a=e.memoizedState,t=0;a!==null&&(t=a.retryLane),Wd(e,t)}function Xf(e,a){var t=0;switch(e.tag){case 13:var{stateNode:r,memoizedState:o}=e;o!==null&&(t=o.retryLane);break;case 19:r=e.stateNode;break;default:throw Error(S(314))}r!==null&&r.delete(a),Wd(e,t)}function Yd(e,a){return ku(e,a)}function Gf(e,a,t,r){this.tag=e,this.key=t,this.sibling=this.child=this.return=this.stateNode=this.type=this.elementType=null,this.index=0,this.ref=null,this.pendingProps=a,this.dependencies=this.memoizedState=this.updateQueue=this.memoizedProps=null,this.mode=r,this.subtreeFlags=this.flags=0,this.deletions=null,this.childLanes=this.lanes=0,this.alternate=null}function we(e,a,t,r){return new Gf(e,a,t,r)}function Si(e){return e=e.prototype,!(!e||!e.isReactComponent)}function Zf(e){if(typeof e==="function")return Si(e)?1:0;if(e!==void 0&&e!==null){if(e=e.$$typeof,e===Al)return 11;if(e===Ll)return 14}return 2}function ga(e,a){var t=e.alternate;return t===null?(t=we(e.tag,a,e.key,e.mode),t.elementType=e.elementType,t.type=e.type,t.stateNode=e.stateNode,t.alternate=e,e.alternate=t):(t.pendingProps=a,t.type=e.type,t.flags=0,t.subtreeFlags=0,t.deletions=null),t.flags=e.flags&14680064,t.childLanes=e.childLanes,t.lanes=e.lanes,t.child=e.child,t.memoizedProps=e.memoizedProps,t.memoizedState=e.memoizedState,t.updateQueue=e.updateQueue,a=e.dependencies,t.dependencies=a===null?null:{lanes:a.lanes,firstContext:a.firstContext},t.sibling=e.sibling,t.index=e.index,t.ref=e.ref,t}function uo(e,a,t,r,o,n){var l=2;if(r=e,typeof e==="function")Si(e)&&(l=1);else if(typeof e==="string")l=5;else e:switch(e){case Wa:return Ra(t.children,o,n,a);case Dl:l=8,o|=8;break;case Vn:return e=we(12,t,a,o|2),e.elementType=Vn,e.lanes=n,e;case Qn:return e=we(13,t,a,o),e.elementType=Qn,e.lanes=n,e;case Un:return e=we(19,t,a,o),e.elementType=Un,e.lanes=n,e;case ou:return $o(t,o,n,a);default:if(typeof e==="object"&&e!==null)switch(e.$$typeof){case tu:l=10;break e;case ru:l=9;break e;case Al:l=11;break e;case Ll:l=14;break e;case oa:l=16,r=null;break e}throw Error(S(130,e==null?e:typeof e,""))}return a=we(l,t,a,o),a.elementType=e,a.type=r,a.lanes=n,a}function Ra(e,a,t,r){return e=we(7,e,r,a),e.lanes=t,e}function $o(e,a,t,r){return e=we(22,e,r,a),e.elementType=ou,e.lanes=t,e.stateNode={isHidden:!1},e}function An(e,a,t){return e=we(6,e,null,a),e.lanes=t,e}function Ln(e,a,t){return a=we(4,e.children!==null?e.children:[],e.key,a),a.lanes=t,a.stateNode={containerInfo:e.containerInfo,pendingChildren:null,implementation:e.implementation},a}function Jf(e,a,t,r,o){this.tag=a,this.containerInfo=e,this.finishedWork=this.pingCache=this.current=this.pendingChildren=null,this.timeoutHandle=-1,this.callbackNode=this.pendingContext=this.context=null,this.callbackPriority=0,this.eventTimes=wn(0),this.expirationTimes=wn(-1),this.entangledLanes=this.finishedLanes=this.mutableReadLanes=this.expiredLanes=this.pingedLanes=this.suspendedLanes=this.pendingLanes=0,this.entanglements=wn(0),this.identifierPrefix=r,this.onRecoverableError=o,this.mutableSourceEagerHydrationData=null}function Ei(e,a,t,r,o,n,l,s,u){return e=new Jf(e,a,t,s,u),a===1?(a=1,n===!0&&(a|=8)):a=0,n=we(3,null,null,a),e.current=n,n.stateNode=e,n.memoizedState={element:r,isDehydrated:t,cache:null,transitions:null,pendingSuspenseBoundaries:null},ri(n),e}function jf(e,a,t){var r=3<arguments.length&&arguments[3]!==void 0?arguments[3]:null;return{$$typeof:$a,key:r==null?null:""+r,children:e,containerInfo:a,implementation:t}}function Xd(e){if(!e)return Sa;e=e._reactInternals;e:{if(Ba(e)!==e||e.tag!==1)throw Error(S(170));var a=e;do{switch(a.tag){case 3:a=a.stateNode.context;break e;case 1:if(fe(a.type)){a=a.stateNode.__reactInternalMemoizedMergedChildContext;break e}}a=a.return}while(a!==null);throw Error(S(171))}if(e.tag===1){var t=e.type;if(fe(t))return Xu(e,t,a)}return a}function Gd(e,a,t,r,o,n,l,s,u){return e=Ei(t,r,!0,e,o,n,l,s,u),e.context=Xd(null),t=e.current,r=ue(),o=ha(t),n=Ye(r,o),n.callback=a!==void 0&&a!==null?a:null,fa(t,n,o),e.current.lanes=o,dr(e,o,r),ve(e,r),e}function Wo(e,a,t,r){var o=a.current,n=ue(),l=ha(o);return t=Xd(t),a.context===null?a.context=t:a.pendingContext=t,a=Ye(n,l),a.payload={element:e},r=r===void 0?null:r,r!==null&&(a.callback=r),e=fa(o,a,l),e!==null&&(be(e,o,l,n),ro(e,o,l)),l}function bo(e){if(e=e.current,!e.child)return null;switch(e.child.tag){case 5:return e.child.stateNode;default:return e.child.stateNode}}function Js(e,a){if(e=e.memoizedState,e!==null&&e.dehydrated!==null){var t=e.retryLane;e.retryLane=t!==0&&t<a?t:a}}function ki(e,a){Js(e,a),(e=e.alternate)&&Js(e,a)}function ev(){return null}function Ni(e){this._internalRoot=e}function Ko(e){this._internalRoot=e}function wi(e){return!(!e||e.nodeType!==1&&e.nodeType!==9&&e.nodeType!==11)}function Yo(e){return!(!e||e.nodeType!==1&&e.nodeType!==9&&e.nodeType!==11&&(e.nodeType!==8||e.nodeValue!==" react-mount-point-unstable "))}function js(){}function av(e,a,t,r,o){if(o){if(typeof r==="function"){var n=r;r=function(){var c=bo(l);n.call(c)}}var l=Gd(a,r,e,0,null,!1,!1,"",js);return e._reactRootContainer=l,e[Ge]=l.current,ar(e.nodeType===8?e.parentNode:e),Aa(),l}for(;o=e.lastChild;)e.removeChild(o);if(typeof r==="function"){var s=r;r=function(){var c=bo(u);s.call(c)}}var u=Ei(e,0,!1,null,null,!1,!1,"",js);return e._reactRootContainer=u,e[Ge]=u.current,ar(e.nodeType===8?e.parentNode:e),Aa(function(){Wo(a,u,t,r)}),u}function Xo(e,a,t,r,o){var n=t._reactRootContainer;if(n){var l=n;if(typeof o==="function"){var s=o;o=function(){var u=bo(l);s.call(u)}}Wo(a,l,e,o)}else l=av(t,a,e,o,r);return bo(l)}var eu,B,au,Kt,Xe,Bn,rp,Gi,Zi,ee,bl,je,Ar,$a,Wa,Dl,Vn,tu,ru,Al,Qn,Un,Ll,oa,ou,Ji,U,Sn,En=!1,Tt,Br,du,At,dp,cp,Jn=null,jn=null,ot=null,nt=null,Nn=!1,el=!1,xa,Lt=!1,mo=null,po=!1,al=null,pp,ku,ns,gp,yp,$,Sp,Vl,Nu,fo,Ep,wu,Fo=null,Be=null,Re,Np,wp,Vr=64,Qr=4194304,R=0,xu,Ul,Mu,zu,Pu,rl=!1,Ur,da=null,ca=null,ma=null,Gt,Zt,la,zp,lt,ho=!0,go=null,sa=null,$l=null,eo=null,ht,Wl,cr,Rp,Cn,qn,xt,Io,us,bp,Fp,Ip,xn,Dp,Ap,Lp,Bp,Vp,ds,Qp,Up,Hp,Wp,Kp,Yp,cs,Xp,Gp,Zp,Jp,jp,ef,af,Yl,Bt=null,tf,Ru,ms,ps=!1,Ka=!1,nf,Vt=null,jt=null,Du=!1,Rt,bt,to,Fe,vf,Ya=null,nl=null,Qt=null,ll=!1,Xa,Mn,Vu,Qu,Uu,Hu,$u,Wu,Ss,It,il,sl,Ft,Dt,hf,Wr,gf,yf,ul=null,dl=null,ml,Sf,ws,Ef,gt,Le,rr,Ge,pl,Nf,wf,fl,Za=-1,Sa,le,pe,ba,$e=null,Bo=!1,Tn=!1,Ja,ja=0,No=null,wo=0,ke,Ne=0,Fa=null,We=1,Ke="",ye=null,ge=null,L=!1,Oe=null,qf,mt,ad,Co,qo=null,et=null,jl=null,_a=null,na=!1,pr,Ve,or,nr,V,On,oo,Rn,Ia=0,Q=null,Y=null,G=null,zo=!1,Ut=!1,lr=0,xf=0,Po,_f,Tf,Of,Qo,Rf,bf,me=!1,wl,Rd,ql,bd,Fd,Zr=!1,ne=!1,Lf,w=null,Hs=!1,J=null,Te=!1,Qf,_o,pi,Ce,O=0,Z=null,K=null,j=0,he=0,tt,X=0,ur=null,Da=0,Ho=0,fi=0,$t=null,ce=null,vi=0,vt=1/0,He=null,To=!1,_l=null,va=null,Jr=!1,ua=null,Oo=0,Wt=0,Tl=null,io=-1,so=0,Kd,Zd,tv,Pt,rv,Ha,Jd,jd=function(e,a){var t=2<arguments.length&&arguments[2]!==void 0?arguments[2]:null;if(!wi(a))throw Error(S(200));return jf(e,a,null,t)},ec=function(e,a){if(!wi(e))throw Error(S(299));var t=!1,r="",o=Zd;return a!==null&&a!==void 0&&(a.unstable_strictMode===!0&&(t=!0),a.identifierPrefix!==void 0&&(r=a.identifierPrefix),a.onRecoverableError!==void 0&&(o=a.onRecoverableError)),a=Ei(e,1,!1,null,null,t,!1,r,o),e[Ge]=a.current,ar(e.nodeType===8?e.parentNode:e),new Ni(a)},ac=function(e){if(e==null)return null;if(e.nodeType===1)return e;var a=e._reactInternals;if(a===void 0){if(typeof e.render==="function")throw Error(S(188));throw e=Object.keys(e).join(","),Error(S(268,e))}return e=Su(a),e=e===null?null:e.stateNode,e},tc=function(e){return Aa(e)},rc=function(e,a,t){if(!Yo(a))throw Error(S(200));return Xo(null,e,a,!0,t)},oc=function(e,a,t){if(!wi(e))throw Error(S(405));var r=t!=null&&t.hydratedSources||null,o=!1,n="",l=Zd;if(t!==null&&t!==void 0&&(t.unstable_strictMode===!0&&(o=!0),t.identifierPrefix!==void 0&&(n=t.identifierPrefix),t.onRecoverableError!==void 0&&(l=t.onRecoverableError)),a=Gd(a,null,e,1,t!=null?t:null,o,!1,n,l),e[Ge]=a.current,ar(e),r)for(e=0;e<r.length;e++)t=r[e],o=t._getVersion,o=o(t._source),a.mutableSourceEagerHydrationData==null?a.mutableSourceEagerHydrationData=[t,o]:a.mutableSourceEagerHydrationData.push(t,o);return new Ko(a)},nc=function(e,a,t){if(!Yo(a))throw Error(S(200));return Xo(null,e,a,!1,t)},lc=function(e){if(!Yo(e))throw Error(S(40));return e._reactRootContainer?(Aa(function(){Xo(null,null,e,!1,function(){e._reactRootContainer=null,e[Ge]=null})}),!0):!1},ic,sc=function(e,a,t,r){if(!Yo(t))throw Error(S(200));if(e==null||e._reactInternals===void 0)throw Error(S(38));return Xo(e,a,t,!1,r)},uc="18.3.1-next-f1338f8080-20240426";var dc=Ac(()=>{eu=Ue(Ca(),1),B=Ue(Xi(),1);au=new Set,Kt={};Xe=!(typeof window>"u"||typeof window.document>"u"||typeof window.document.createElement>"u"),Bn=Object.prototype.hasOwnProperty,rp=/^[:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD][:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD\-.0-9\u00B7\u0300-\u036F\u203F-\u2040]*$/,Gi={},Zi={};ee={};"children dangerouslySetInnerHTML defaultValue defaultChecked innerHTML suppressContentEditableWarning suppressHydrationWarning style".split(" ").forEach(function(e){ee[e]=new de(e,0,!1,e,null,!1,!1)});[["acceptCharset","accept-charset"],["className","class"],["htmlFor","for"],["httpEquiv","http-equiv"]].forEach(function(e){var a=e[0];ee[a]=new de(a,1,!1,e[1],null,!1,!1)});["contentEditable","draggable","spellCheck","value"].forEach(function(e){ee[e]=new de(e,2,!1,e.toLowerCase(),null,!1,!1)});["autoReverse","externalResourcesRequired","focusable","preserveAlpha"].forEach(function(e){ee[e]=new de(e,2,!1,e,null,!1,!1)});"allowFullScreen async autoFocus autoPlay controls default defer disabled disablePictureInPicture disableRemotePlayback formNoValidate hidden loop noModule noValidate open playsInline readOnly required reversed scoped seamless itemScope".split(" ").forEach(function(e){ee[e]=new de(e,3,!1,e.toLowerCase(),null,!1,!1)});["checked","multiple","muted","selected"].forEach(function(e){ee[e]=new de(e,3,!0,e,null,!1,!1)});["capture","download"].forEach(function(e){ee[e]=new de(e,4,!1,e,null,!1,!1)});["cols","rows","size","span"].forEach(function(e){ee[e]=new de(e,6,!1,e,null,!1,!1)});["rowSpan","start"].forEach(function(e){ee[e]=new de(e,5,!1,e.toLowerCase(),null,!1,!1)});bl=/[\-:]([a-z])/g;"accent-height alignment-baseline arabic-form baseline-shift cap-height clip-path clip-rule color-interpolation color-interpolation-filters color-profile color-rendering dominant-baseline enable-background fill-opacity fill-rule flood-color flood-opacity font-family font-size font-size-adjust font-stretch font-style font-variant font-weight glyph-name glyph-orientation-horizontal glyph-orientation-vertical horiz-adv-x horiz-origin-x image-rendering letter-spacing lighting-color marker-end marker-mid marker-start overline-position overline-thickness paint-order panose-1 pointer-events rendering-intent shape-rendering stop-color stop-opacity strikethrough-position strikethrough-thickness stroke-dasharray stroke-dashoffset stroke-linecap stroke-linejoin stroke-miterlimit stroke-opacity stroke-width text-anchor text-decoration text-rendering underline-position underline-thickness unicode-bidi unicode-range units-per-em v-alphabetic v-hanging v-ideographic v-mathematical vector-effect vert-adv-y vert-origin-x vert-origin-y word-spacing writing-mode xmlns:xlink x-height".split(" ").forEach(function(e){var a=e.replace(bl,Fl);ee[a]=new de(a,1,!1,e,null,!1,!1)});"xlink:actuate xlink:arcrole xlink:role xlink:show xlink:title xlink:type".split(" ").forEach(function(e){var a=e.replace(bl,Fl);ee[a]=new de(a,1,!1,e,"http://www.w3.org/1999/xlink",!1,!1)});["xml:base","xml:lang","xml:space"].forEach(function(e){var a=e.replace(bl,Fl);ee[a]=new de(a,1,!1,e,"http://www.w3.org/XML/1998/namespace",!1,!1)});["tabIndex","crossOrigin"].forEach(function(e){ee[e]=new de(e,1,!1,e.toLowerCase(),null,!1,!1)});ee.xlinkHref=new de("xlinkHref",1,!1,"xlink:href","http://www.w3.org/1999/xlink",!0,!1);["src","href","action","formAction"].forEach(function(e){ee[e]=new de(e,1,!1,e.toLowerCase(),null,!0,!0)});je=eu.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,Ar=Symbol.for("react.element"),$a=Symbol.for("react.portal"),Wa=Symbol.for("react.fragment"),Dl=Symbol.for("react.strict_mode"),Vn=Symbol.for("react.profiler"),tu=Symbol.for("react.provider"),ru=Symbol.for("react.context"),Al=Symbol.for("react.forward_ref"),Qn=Symbol.for("react.suspense"),Un=Symbol.for("react.suspense_list"),Ll=Symbol.for("react.memo"),oa=Symbol.for("react.lazy"),ou=Symbol.for("react.offscreen"),Ji=Symbol.iterator;U=Object.assign;Tt=Array.isArray;du=function(e){return typeof MSApp<"u"&&MSApp.execUnsafeLocalFunction?function(a,t,r,o){MSApp.execUnsafeLocalFunction(function(){return e(a,t,r,o)})}:e}(function(e,a){if(e.namespaceURI!=="http://www.w3.org/2000/svg"||"innerHTML"in e)e.innerHTML=a;else{Br=Br||document.createElement("div"),Br.innerHTML="<svg>"+a.valueOf().toString()+"</svg>";for(a=Br.firstChild;e.firstChild;)e.removeChild(e.firstChild);for(;a.firstChild;)e.appendChild(a.firstChild)}});At={animationIterationCount:!0,aspectRatio:!0,borderImageOutset:!0,borderImageSlice:!0,borderImageWidth:!0,boxFlex:!0,boxFlexGroup:!0,boxOrdinalGroup:!0,columnCount:!0,columns:!0,flex:!0,flexGrow:!0,flexPositive:!0,flexShrink:!0,flexNegative:!0,flexOrder:!0,gridArea:!0,gridRow:!0,gridRowEnd:!0,gridRowSpan:!0,gridRowStart:!0,gridColumn:!0,gridColumnEnd:!0,gridColumnSpan:!0,gridColumnStart:!0,fontWeight:!0,lineClamp:!0,lineHeight:!0,opacity:!0,order:!0,orphans:!0,tabSize:!0,widows:!0,zIndex:!0,zoom:!0,fillOpacity:!0,floodOpacity:!0,stopOpacity:!0,strokeDasharray:!0,strokeDashoffset:!0,strokeMiterlimit:!0,strokeOpacity:!0,strokeWidth:!0},dp=["Webkit","ms","Moz","O"];Object.keys(At).forEach(function(e){dp.forEach(function(a){a=a+e.charAt(0).toUpperCase()+e.substring(1),At[a]=At[e]})});cp=U({menuitem:!0},{area:!0,base:!0,br:!0,col:!0,embed:!0,hr:!0,img:!0,input:!0,keygen:!0,link:!0,meta:!0,param:!0,source:!0,track:!0,wbr:!0});if(Xe)try{xa={},Object.defineProperty(xa,"passive",{get:function(){el=!0}}),window.addEventListener("test",xa,xa),window.removeEventListener("test",xa,xa)}catch(e){el=!1}pp={onError:function(e){Lt=!0,mo=e}};ku=B.unstable_scheduleCallback,ns=B.unstable_cancelCallback,gp=B.unstable_shouldYield,yp=B.unstable_requestPaint,$=B.unstable_now,Sp=B.unstable_getCurrentPriorityLevel,Vl=B.unstable_ImmediatePriority,Nu=B.unstable_UserBlockingPriority,fo=B.unstable_NormalPriority,Ep=B.unstable_LowPriority,wu=B.unstable_IdlePriority;Re=Math.clz32?Math.clz32:Cp,Np=Math.log,wp=Math.LN2;Ur=[],Gt=new Map,Zt=new Map,la=[],zp="mousedown mouseup touchcancel touchend touchstart auxclick dblclick pointercancel pointerdown pointerup dragend dragstart drop compositionend compositionstart keydown keypress keyup input textInput copy cut paste click change contextmenu reset submit".split(" ");lt=je.ReactCurrentBatchConfig;ht={eventPhase:0,bubbles:0,cancelable:0,timeStamp:function(e){return e.timeStamp||Date.now()},defaultPrevented:0,isTrusted:0},Wl=Se(ht),cr=U({},ht,{view:0,detail:0}),Rp=Se(cr),Io=U({},cr,{screenX:0,screenY:0,clientX:0,clientY:0,pageX:0,pageY:0,ctrlKey:0,shiftKey:0,altKey:0,metaKey:0,getModifierState:Kl,button:0,buttons:0,relatedTarget:function(e){return e.relatedTarget===void 0?e.fromElement===e.srcElement?e.toElement:e.fromElement:e.relatedTarget},movementX:function(e){if("movementX"in e)return e.movementX;return e!==xt&&(xt&&e.type==="mousemove"?(Cn=e.screenX-xt.screenX,qn=e.screenY-xt.screenY):qn=Cn=0,xt=e),Cn},movementY:function(e){return"movementY"in e?e.movementY:qn}}),us=Se(Io),bp=U({},Io,{dataTransfer:0}),Fp=Se(bp),Ip=U({},cr,{relatedTarget:0}),xn=Se(Ip),Dp=U({},ht,{animationName:0,elapsedTime:0,pseudoElement:0}),Ap=Se(Dp),Lp=U({},ht,{clipboardData:function(e){return"clipboardData"in e?e.clipboardData:window.clipboardData}}),Bp=Se(Lp),Vp=U({},ht,{data:0}),ds=Se(Vp),Qp={Esc:"Escape",Spacebar:" ",Left:"ArrowLeft",Up:"ArrowUp",Right:"ArrowRight",Down:"ArrowDown",Del:"Delete",Win:"OS",Menu:"ContextMenu",Apps:"ContextMenu",Scroll:"ScrollLock",MozPrintableKey:"Unidentified"},Up={8:"Backspace",9:"Tab",12:"Clear",13:"Enter",16:"Shift",17:"Control",18:"Alt",19:"Pause",20:"CapsLock",27:"Escape",32:" ",33:"PageUp",34:"PageDown",35:"End",36:"Home",37:"ArrowLeft",38:"ArrowUp",39:"ArrowRight",40:"ArrowDown",45:"Insert",46:"Delete",112:"F1",113:"F2",114:"F3",115:"F4",116:"F5",117:"F6",118:"F7",119:"F8",120:"F9",121:"F10",122:"F11",123:"F12",144:"NumLock",145:"ScrollLock",224:"Meta"},Hp={Alt:"altKey",Control:"ctrlKey",Meta:"metaKey",Shift:"shiftKey"};Wp=U({},cr,{key:function(e){if(e.key){var a=Qp[e.key]||e.key;if(a!=="Unidentified")return a}return e.type==="keypress"?(e=ao(e),e===13?"Enter":String.fromCharCode(e)):e.type==="keydown"||e.type==="keyup"?Up[e.keyCode]||"Unidentified":""},code:0,location:0,ctrlKey:0,shiftKey:0,altKey:0,metaKey:0,repeat:0,locale:0,getModifierState:Kl,charCode:function(e){return e.type==="keypress"?ao(e):0},keyCode:function(e){return e.type==="keydown"||e.type==="keyup"?e.keyCode:0},which:function(e){return e.type==="keypress"?ao(e):e.type==="keydown"||e.type==="keyup"?e.keyCode:0}}),Kp=Se(Wp),Yp=U({},Io,{pointerId:0,width:0,height:0,pressure:0,tangentialPressure:0,tiltX:0,tiltY:0,twist:0,pointerType:0,isPrimary:0}),cs=Se(Yp),Xp=U({},cr,{touches:0,targetTouches:0,changedTouches:0,altKey:0,metaKey:0,ctrlKey:0,shiftKey:0,getModifierState:Kl}),Gp=Se(Xp),Zp=U({},ht,{propertyName:0,elapsedTime:0,pseudoElement:0}),Jp=Se(Zp),jp=U({},Io,{deltaX:function(e){return"deltaX"in e?e.deltaX:("wheelDeltaX"in e)?-e.wheelDeltaX:0},deltaY:function(e){return"deltaY"in e?e.deltaY:("wheelDeltaY"in e)?-e.wheelDeltaY:("wheelDelta"in e)?-e.wheelDelta:0},deltaZ:0,deltaMode:0}),ef=Se(jp),af=[9,13,27,32],Yl=Xe&&"CompositionEvent"in window;Xe&&"documentMode"in document&&(Bt=document.documentMode);tf=Xe&&"TextEvent"in window&&!Bt,Ru=Xe&&(!Yl||Bt&&8<Bt&&11>=Bt),ms=String.fromCharCode(32);nf={color:!0,date:!0,datetime:!0,"datetime-local":!0,email:!0,month:!0,number:!0,password:!0,range:!0,search:!0,tel:!0,text:!0,time:!0,url:!0,week:!0};if(Xe){if(Xe){if(bt="oninput"in document,!bt)to=document.createElement("div"),to.setAttribute("oninput","return;"),bt=typeof to.oninput==="function";Rt=bt}else Rt=!1;Du=Rt&&(!document.documentMode||9<document.documentMode)}Fe=typeof Object.is==="function"?Object.is:pf;vf=Xe&&"documentMode"in document&&11>=document.documentMode;Xa={animationend:$r("Animation","AnimationEnd"),animationiteration:$r("Animation","AnimationIteration"),animationstart:$r("Animation","AnimationStart"),transitionend:$r("Transition","TransitionEnd")},Mn={},Vu={};Xe&&(Vu=document.createElement("div").style,("AnimationEvent"in window)||(delete Xa.animationend.animation,delete Xa.animationiteration.animation,delete Xa.animationstart.animation),("TransitionEvent"in window)||delete Xa.transitionend.transition);Qu=Ao("animationend"),Uu=Ao("animationiteration"),Hu=Ao("animationstart"),$u=Ao("transitionend"),Wu=new Map,Ss="abort auxClick cancel canPlay canPlayThrough click close contextMenu copy cut drag dragEnd dragEnter dragExit dragLeave dragOver dragStart drop durationChange emptied encrypted ended error gotPointerCapture input invalid keyDown keyPress keyUp load loadedData loadedMetadata loadStart lostPointerCapture mouseDown mouseMove mouseOut mouseOver mouseUp paste pause play playing pointerCancel pointerDown pointerMove pointerOut pointerOver pointerUp progress rateChange reset resize seeked seeking stalled submit suspend timeUpdate touchCancel touchEnd touchStart volumeChange scroll toggle touchMove waiting wheel".split(" ");for(Ft=0;Ft<Ss.length;Ft++)It=Ss[Ft],il=It.toLowerCase(),sl=It[0].toUpperCase()+It.slice(1),Ea(il,"on"+sl);Ea(Qu,"onAnimationEnd");Ea(Uu,"onAnimationIteration");Ea(Hu,"onAnimationStart");Ea("dblclick","onDoubleClick");Ea("focusin","onFocus");Ea("focusout","onBlur");Ea($u,"onTransitionEnd");ut("onMouseEnter",["mouseout","mouseover"]);ut("onMouseLeave",["mouseout","mouseover"]);ut("onPointerEnter",["pointerout","pointerover"]);ut("onPointerLeave",["pointerout","pointerover"]);La("onChange","change click focusin focusout input keydown keyup selectionchange".split(" "));La("onSelect","focusout contextmenu dragend focusin keydown keyup mousedown mouseup selectionchange".split(" "));La("onBeforeInput",["compositionend","keypress","textInput","paste"]);La("onCompositionEnd","compositionend focusout keydown keypress keyup mousedown".split(" "));La("onCompositionStart","compositionstart focusout keydown keypress keyup mousedown".split(" "));La("onCompositionUpdate","compositionupdate focusout keydown keypress keyup mousedown".split(" "));Dt="abort canplay canplaythrough durationchange emptied encrypted ended error loadeddata loadedmetadata loadstart pause play playing progress ratechange resize seeked seeking stalled suspend timeupdate volumechange waiting".split(" "),hf=new Set("cancel close invalid load scroll toggle".split(" ").concat(Dt));Wr="_reactListening"+Math.random().toString(36).slice(2);gf=/\r\n?/g,yf=/\u0000|\uFFFD/g;ml=typeof setTimeout==="function"?setTimeout:void 0,Sf=typeof clearTimeout==="function"?clearTimeout:void 0,ws=typeof Promise==="function"?Promise:void 0,Ef=typeof queueMicrotask==="function"?queueMicrotask:typeof ws<"u"?function(e){return ws.resolve(null).then(e).catch(kf)}:ml;gt=Math.random().toString(36).slice(2),Le="__reactFiber$"+gt,rr="__reactProps$"+gt,Ge="__reactContainer$"+gt,pl="__reactEvents$"+gt,Nf="__reactListeners$"+gt,wf="__reactHandles$"+gt;fl=[];Sa={},le=ka(Sa),pe=ka(!1),ba=Sa;Ja=[],ke=[];qf=je.ReactCurrentBatchConfig;mt=ed(!0),ad=ed(!1),Co=ka(null);pr={},Ve=ka(pr),or=ka(pr),nr=ka(pr);V=ka(0);On=[];oo=je.ReactCurrentDispatcher,Rn=je.ReactCurrentBatchConfig;Po={readContext:qe,useCallback:re,useContext:re,useEffect:re,useImperativeHandle:re,useInsertionEffect:re,useLayoutEffect:re,useMemo:re,useReducer:re,useRef:re,useState:re,useDebugValue:re,useDeferredValue:re,useTransition:re,useMutableSource:re,useSyncExternalStore:re,useId:re,unstable_isNewReconciler:!1},_f={readContext:qe,useCallback:function(e,a){return Ae().memoizedState=[e,a===void 0?null:a],e},useContext:qe,useEffect:Rs,useImperativeHandle:function(e,a,t){return t=t!==null&&t!==void 0?t.concat([e]):null,no(4194308,4,vd.bind(null,a,e),t)},useLayoutEffect:function(e,a){return no(4194308,4,e,a)},useInsertionEffect:function(e,a){return no(4,2,e,a)},useMemo:function(e,a){var t=Ae();return a=a===void 0?null:a,e=e(),t.memoizedState=[e,a],e},useReducer:function(e,a,t){var r=Ae();return a=t!==void 0?t(a):a,r.memoizedState=r.baseState=a,e={pending:null,interleaved:null,lanes:0,dispatch:null,lastRenderedReducer:e,lastRenderedState:a},r.queue=e,e=e.dispatch=zf.bind(null,Q,e),[r.memoizedState,e]},useRef:function(e){var a=Ae();return e={current:e},a.memoizedState=e},useState:Os,useDebugValue:ci,useDeferredValue:function(e){return Ae().memoizedState=e},useTransition:function(){var e=Os(!1),a=e[0];return e=Mf.bind(null,e[1]),Ae().memoizedState=e,[a,e]},useMutableSource:function(){},useSyncExternalStore:function(e,a,t){var r=Q,o=Ae();if(L){if(t===void 0)throw Error(S(407));t=t()}else{if(t=a(),Z===null)throw Error(S(349));(Ia&30)!==0||id(r,a,t)}o.memoizedState=t;var n={value:t,getSnapshot:a};return o.queue=n,Rs(ud.bind(null,r,n,e),[e]),r.flags|=2048,sr(9,sd.bind(null,r,n,t,a),void 0,null),t},useId:function(){var e=Ae(),a=Z.identifierPrefix;if(L){var t=Ke,r=We;t=(r&~(1<<32-Re(r)-1)).toString(32)+t,a=":"+a+"R"+t,t=lr++,0<t&&(a+="H"+t.toString(32)),a+=":"}else t=xf++,a=":"+a+"r"+t.toString(32)+":";return e.memoizedState=a},unstable_isNewReconciler:!1},Tf={readContext:qe,useCallback:gd,useContext:qe,useEffect:di,useImperativeHandle:hd,useInsertionEffect:pd,useLayoutEffect:fd,useMemo:yd,useReducer:bn,useRef:md,useState:function(){return bn(ir)},useDebugValue:ci,useDeferredValue:function(e){var a=xe();return Sd(a,Y.memoizedState,e)},useTransition:function(){var e=bn(ir)[0],a=xe().memoizedState;return[e,a]},useMutableSource:nd,useSyncExternalStore:ld,useId:Ed,unstable_isNewReconciler:!1},Of={readContext:qe,useCallback:gd,useContext:qe,useEffect:di,useImperativeHandle:hd,useInsertionEffect:pd,useLayoutEffect:fd,useMemo:yd,useReducer:Fn,useRef:md,useState:function(){return Fn(ir)},useDebugValue:ci,useDeferredValue:function(e){var a=xe();return Y===null?a.memoizedState=e:Sd(a,Y.memoizedState,e)},useTransition:function(){var e=Fn(ir)[0],a=xe().memoizedState;return[e,a]},useMutableSource:nd,useSyncExternalStore:ld,useId:Ed,unstable_isNewReconciler:!1};Qo={isMounted:function(e){return(e=e._reactInternals)?Ba(e)===e:!1},enqueueSetState:function(e,a,t){e=e._reactInternals;var r=ue(),o=ha(e),n=Ye(r,o);n.payload=a,t!==void 0&&t!==null&&(n.callback=t),a=fa(e,n,o),a!==null&&(be(a,e,o,r),ro(a,e,o))},enqueueReplaceState:function(e,a,t){e=e._reactInternals;var r=ue(),o=ha(e),n=Ye(r,o);n.tag=1,n.payload=a,t!==void 0&&t!==null&&(n.callback=t),a=fa(e,n,o),a!==null&&(be(a,e,o,r),ro(a,e,o))},enqueueForceUpdate:function(e,a){e=e._reactInternals;var t=ue(),r=ha(e),o=Ye(t,r);o.tag=2,a!==void 0&&a!==null&&(o.callback=a),a=fa(e,o,r),a!==null&&(be(a,e,r,t),ro(a,e,r))}};Rf=typeof WeakMap==="function"?WeakMap:Map;bf=je.ReactCurrentOwner;wl={dehydrated:null,treeContext:null,retryLane:0};Rd=function(e,a){for(var t=a.child;t!==null;){if(t.tag===5||t.tag===6)e.appendChild(t.stateNode);else if(t.tag!==4&&t.child!==null){t.child.return=t,t=t.child;continue}if(t===a)break;for(;t.sibling===null;){if(t.return===null||t.return===a)return;t=t.return}t.sibling.return=t.return,t=t.sibling}};ql=function(){};bd=function(e,a,t,r){var o=e.memoizedProps;if(o!==r){e=a.stateNode,Ta(Ve.current);var n=null;switch(t){case"input":o=$n(e,o),r=$n(e,r),n=[];break;case"select":o=U({},o,{value:void 0}),r=U({},r,{value:void 0}),n=[];break;case"textarea":o=Yn(e,o),r=Yn(e,r),n=[];break;default:typeof o.onClick!=="function"&&typeof r.onClick==="function"&&(e.onclick=So)}Gn(t,r);var l;t=null;for(c in o)if(!r.hasOwnProperty(c)&&o.hasOwnProperty(c)&&o[c]!=null)if(c==="style"){var s=o[c];for(l in s)s.hasOwnProperty(l)&&(t||(t={}),t[l]="")}else c!=="dangerouslySetInnerHTML"&&c!=="children"&&c!=="suppressContentEditableWarning"&&c!=="suppressHydrationWarning"&&c!=="autoFocus"&&(Kt.hasOwnProperty(c)?n||(n=[]):(n=n||[]).push(c,null));for(c in r){var u=r[c];if(s=o!=null?o[c]:void 0,r.hasOwnProperty(c)&&u!==s&&(u!=null||s!=null))if(c==="style")if(s){for(l in s)!s.hasOwnProperty(l)||u&&u.hasOwnProperty(l)||(t||(t={}),t[l]="");for(l in u)u.hasOwnProperty(l)&&s[l]!==u[l]&&(t||(t={}),t[l]=u[l])}else t||(n||(n=[]),n.push(c,t)),t=u;else c==="dangerouslySetInnerHTML"?(u=u?u.__html:void 0,s=s?s.__html:void 0,u!=null&&s!==u&&(n=n||[]).push(c,u)):c==="children"?typeof u!=="string"&&typeof u!=="number"||(n=n||[]).push(c,""+u):c!=="suppressContentEditableWarning"&&c!=="suppressHydrationWarning"&&(Kt.hasOwnProperty(c)?(u!=null&&c==="onScroll"&&D("scroll",e),n||s===u||(n=[])):(n=n||[]).push(c,u))}t&&(n=n||[]).push("style",t);var c=n;if(a.updateQueue=c)a.flags|=4}};Fd=function(e,a,t,r){t!==r&&(a.flags|=4)};Lf=typeof WeakSet==="function"?WeakSet:Set;Qf=Math.ceil,_o=je.ReactCurrentDispatcher,pi=je.ReactCurrentOwner,Ce=je.ReactCurrentBatchConfig,tt=ka(0);Kd=function(e,a,t){if(e!==null)if(e.memoizedProps!==a.pendingProps||pe.current)me=!0;else{if((e.lanes&t)===0&&(a.flags&128)===0)return me=!1,If(e,a,t);me=(e.flags&131072)!==0?!0:!1}else me=!1,L&&(a.flags&1048576)!==0&&Zu(a,wo,a.index);switch(a.lanes=0,a.tag){case 2:var r=a.type;lo(e,a),e=a.pendingProps;var o=dt(a,le.current);it(a,t),o=si(null,a,r,e,o,t);var n=ui();return a.flags|=1,typeof o==="object"&&o!==null&&typeof o.render==="function"&&o.$$typeof===void 0?(a.tag=1,a.memoizedState=null,a.updateQueue=null,fe(r)?(n=!0,ko(a)):n=!1,a.memoizedState=o.state!==null&&o.state!==void 0?o.state:null,ri(a),o.updater=Qo,a.stateNode=o,o._reactInternals=a,Sl(a,r,e,t),a=Nl(null,a,r,!0,n,t)):(a.tag=0,L&&n&&Gl(a),se(null,a,o,t),a=a.child),a;case 16:r=a.elementType;e:{switch(lo(e,a),e=a.pendingProps,o=r._init,r=o(r._payload),a.type=r,o=a.tag=Zf(r),e=_e(r,e),o){case 0:a=kl(null,a,r,e,t);break e;case 1:a=Vs(null,a,r,e,t);break e;case 11:a=Ls(null,a,r,e,t);break e;case 14:a=Bs(null,a,r,_e(r.type,e),t);break e}throw Error(S(306,r,""))}return a;case 0:return r=a.type,o=a.pendingProps,o=a.elementType===r?o:_e(r,o),kl(e,a,r,o,t);case 1:return r=a.type,o=a.pendingProps,o=a.elementType===r?o:_e(r,o),Vs(e,a,r,o,t);case 3:e:{if(_d(a),e===null)throw Error(S(387));r=a.pendingProps,n=a.memoizedState,o=n.element,rd(e,a),xo(a,r,null,t);var l=a.memoizedState;if(r=l.element,n.isDehydrated)if(n={element:r,isDehydrated:!1,cache:l.cache,pendingSuspenseBoundaries:l.pendingSuspenseBoundaries,transitions:l.transitions},a.updateQueue.baseState=n,a.memoizedState=n,a.flags&256){o=ft(Error(S(423)),a),a=Qs(e,a,r,t,o);break e}else if(r!==o){o=ft(Error(S(424)),a),a=Qs(e,a,r,t,o);break e}else for(ge=pa(a.stateNode.containerInfo.firstChild),ye=a,L=!0,Oe=null,t=ad(a,null,r,t),a.child=t;t;)t.flags=t.flags&-3|4096,t=t.sibling;else{if(ct(),r===o){a=Je(e,a,t);break e}se(e,a,r,t)}a=a.child}return a;case 5:return od(a),e===null&&hl(a),r=a.type,o=a.pendingProps,n=e!==null?e.memoizedProps:null,l=o.children,cl(r,o)?l=null:n!==null&&cl(r,n)&&(a.flags|=32),Pd(e,a),se(e,a,l,t),a.child;case 6:return e===null&&hl(a),null;case 13:return Td(e,a,t);case 4:return oi(a,a.stateNode.containerInfo),r=a.pendingProps,e===null?a.child=mt(a,null,r,t):se(e,a,r,t),a.child;case 11:return r=a.type,o=a.pendingProps,o=a.elementType===r?o:_e(r,o),Ls(e,a,r,o,t);case 7:return se(e,a,a.pendingProps,t),a.child;case 8:return se(e,a,a.pendingProps.children,t),a.child;case 12:return se(e,a,a.pendingProps.children,t),a.child;case 10:e:{if(r=a.type._context,o=a.pendingProps,n=a.memoizedProps,l=o.value,F(Co,r._currentValue),r._currentValue=l,n!==null)if(Fe(n.value,l)){if(n.children===o.children&&!pe.current){a=Je(e,a,t);break e}}else for(n=a.child,n!==null&&(n.return=a);n!==null;){var s=n.dependencies;if(s!==null){l=n.child;for(var u=s.firstContext;u!==null;){if(u.context===r){if(n.tag===1){u=Ye(-1,t&-t),u.tag=2;var c=n.updateQueue;if(c!==null){c=c.shared;var h=c.pending;h===null?u.next=u:(u.next=h.next,h.next=u),c.pending=u}}n.lanes|=t,u=n.alternate,u!==null&&(u.lanes|=t),gl(n.return,t,a),s.lanes|=t;break}u=u.next}}else if(n.tag===10)l=n.type===a.type?null:n.child;else if(n.tag===18){if(l=n.return,l===null)throw Error(S(341));l.lanes|=t,s=l.alternate,s!==null&&(s.lanes|=t),gl(l,t,a),l=n.sibling}else l=n.child;if(l!==null)l.return=n;else for(l=n;l!==null;){if(l===a){l=null;break}if(n=l.sibling,n!==null){n.return=l.return,l=n;break}l=l.return}n=l}se(e,a,o.children,t),a=a.child}return a;case 9:return o=a.type,r=a.pendingProps.children,it(a,t),o=qe(o),r=r(o),a.flags|=1,se(e,a,r,t),a.child;case 14:return r=a.type,o=_e(r,a.pendingProps),o=_e(r.type,o),Bs(e,a,r,o,t);case 15:return Md(e,a,a.type,a.pendingProps,t);case 17:return r=a.type,o=a.pendingProps,o=a.elementType===r?o:_e(r,o),lo(e,a),a.tag=1,fe(r)?(e=!0,ko(a)):e=!1,it(a,t),Cd(a,r,o),Sl(a,r,o,t),Nl(null,a,r,!0,e,t);case 19:return Od(e,a,t);case 22:return zd(e,a,t)}throw Error(S(156,a.tag))};Zd=typeof reportError==="function"?reportError:function(e){console.error(e)};Ko.prototype.render=Ni.prototype.render=function(e){var a=this._internalRoot;if(a===null)throw Error(S(409));Wo(e,a,null,null)};Ko.prototype.unmount=Ni.prototype.unmount=function(){var e=this._internalRoot;if(e!==null){this._internalRoot=null;var a=e.containerInfo;Aa(function(){Wo(null,e,null,null)}),a[Ge]=null}};Ko.prototype.unstable_scheduleHydration=function(e){if(e){var a=zu();e={blockedOn:null,target:e,priority:a};for(var t=0;t<la.length&&a!==0&&a<la[t].priority;t++);la.splice(t,0,e),t===0&&_u(e)}};xu=function(e){switch(e.tag){case 3:var a=e.stateNode;if(a.current.memoizedState.isDehydrated){var t=Ot(a.pendingLanes);t!==0&&(Ql(a,t|1),ve(a,$()),(O&6)===0&&(vt=$()+500,Na()))}break;case 13:Aa(function(){var r=Ze(e,1);if(r!==null){var o=ue();be(r,e,1,o)}}),ki(e,1)}};Ul=function(e){if(e.tag===13){var a=Ze(e,134217728);if(a!==null){var t=ue();be(a,e,134217728,t)}ki(e,134217728)}};Mu=function(e){if(e.tag===13){var a=ha(e),t=Ze(e,a);if(t!==null){var r=ue();be(t,e,a,r)}ki(e,a)}};zu=function(){return R};Pu=function(e,a){var t=R;try{return R=e,a()}finally{R=t}};jn=function(e,a,t){switch(a){case"input":if(Wn(e,t),a=t.name,t.type==="radio"&&a!=null){for(t=e;t.parentNode;)t=t.parentNode;t=t.querySelectorAll("input[name="+JSON.stringify(""+a)+'][type="radio"]');for(a=0;a<t.length;a++){var r=t[a];if(r!==e&&r.form===e.form){var o=Lo(r);if(!o)throw Error(S(90));lu(r),Wn(r,o)}}}break;case"textarea":su(e,t);break;case"select":a=t.value,a!=null&&rt(e,!!t.multiple,a,!1)}};vu=hi;hu=Aa;tv={usingClientEntryPoint:!1,Events:[mr,Ga,Lo,pu,fu,hi]},Pt={findFiberByHostInstance:Pa,bundleType:0,version:"18.3.1",rendererPackageName:"react-dom"},rv={bundleType:Pt.bundleType,version:Pt.version,rendererPackageName:Pt.rendererPackageName,rendererConfig:Pt.rendererConfig,overrideHookState:null,overrideHookStateDeletePath:null,overrideHookStateRenamePath:null,overrideProps:null,overridePropsDeletePath:null,overridePropsRenamePath:null,setErrorHandler:null,setSuspenseHandler:null,scheduleUpdate:null,currentDispatcherRef:je.ReactCurrentDispatcher,findHostInstanceByFiber:function(e){return e=Su(e),e===null?null:e.stateNode},findFiberByHostInstance:Pt.findFiberByHostInstance||ev,findHostInstancesForRefresh:null,scheduleRefresh:null,scheduleRoot:null,setRefreshHandler:null,getCurrentFiber:null,reconcilerVersion:"18.3.1-next-f1338f8080-20240426"};if(typeof __REACT_DEVTOOLS_GLOBAL_HOOK__<"u"){if(Ha=__REACT_DEVTOOLS_GLOBAL_HOOK__,!Ha.isDisabled&&Ha.supportsFiber)try{Fo=Ha.inject(rv),Be=Ha}catch(e){}}Jd=tv,ic=hi});var pc=Mr((fv,mc)=>{dc();function cc(){if(typeof __REACT_DEVTOOLS_GLOBAL_HOOK__>"u"||typeof __REACT_DEVTOOLS_GLOBAL_HOOK__.checkDCE!=="function")return;try{__REACT_DEVTOOLS_GLOBAL_HOOK__.checkDCE(cc)}catch(e){console.error(e)}}cc(),mc.exports=Ci});var fc=Mr((nv)=>{var fr=Ue(pc(),1);nv.createRoot=fr.createRoot,nv.hydrateRoot=fr.hydrateRoot;var ov});var xc=Ue(Ca(),1),Mc=Ue(fc(),1);var ae=Ue(Ca(),1);var Zo=Ue(Ca(),1);var vc=(e)=>e.replace(/([a-z0-9])([A-Z])/g,"$1-$2").toLowerCase(),Go=(...e)=>e.filter((a,t,r)=>{return Boolean(a)&&a.trim()!==""&&r.indexOf(a)===t}).join(" ").trim();var vr=Ue(Ca(),1);var hc={xmlns:"http://www.w3.org/2000/svg",width:24,height:24,viewBox:"0 0 24 24",fill:"none",stroke:"currentColor",strokeWidth:2,strokeLinecap:"round",strokeLinejoin:"round"};var gc=vr.forwardRef(({color:e="currentColor",size:a=24,strokeWidth:t=2,absoluteStrokeWidth:r,className:o="",children:n,iconNode:l,...s},u)=>{return vr.createElement("svg",{ref:u,...hc,width:a,height:a,stroke:e,strokeWidth:r?Number(t)*24/Number(a):t,className:Go("lucide",o),...s},[...l.map(([c,h])=>vr.createElement(c,h)),...Array.isArray(n)?n:[n]])});var I=(e,a)=>{let t=Zo.forwardRef(({className:r,...o},n)=>Zo.createElement(gc,{ref:n,iconNode:a,className:Go(`lucide-${vc(e)}`,r),...o}));return t.displayName=`${e}`,t};var hr=I("Camera",[["path",{d:"M14.5 4h-5L7 7H4a2 2 0 0 0-2 2v9a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2V9a2 2 0 0 0-2-2h-3l-2.5-3z",key:"1tc9qg"}],["circle",{cx:"12",cy:"13",r:"3",key:"1vg3eu"}]]);var gr=I("Check",[["path",{d:"M20 6 9 17l-5-5",key:"1gmf2c"}]]);var yr=I("ChevronDown",[["path",{d:"m6 9 6 6 6-6",key:"qrunsl"}]]);var Sr=I("Clock",[["circle",{cx:"12",cy:"12",r:"10",key:"1mglay"}],["polyline",{points:"12 6 12 12 16 14",key:"68esgv"}]]);var ea=I("Copy",[["rect",{width:"14",height:"14",x:"8",y:"8",rx:"2",ry:"2",key:"17jyea"}],["path",{d:"M4 16c-1.1 0-2-.9-2-2V4c0-1.1.9-2 2-2h10c1.1 0 2 .9 2 2",key:"zix9uf"}]]);var Er=I("Heart",[["path",{d:"M19 14c1.49-1.46 3-3.21 3-5.5A5.5 5.5 0 0 0 16.5 3c-1.76 0-3 .5-4.5 2-1.5-1.5-2.74-2-4.5-2A5.5 5.5 0 0 0 2 8.5c0 2.3 1.5 4.05 3 5.5l7 7Z",key:"c3ymky"}]]);var kr=I("MessageCircle",[["path",{d:"M7.9 20A9 9 0 1 0 4 16.1L2 22Z",key:"vv11sd"}]]);var Nr=I("Music2",[["circle",{cx:"8",cy:"18",r:"4",key:"1fc0mg"}],["path",{d:"M12 18V2l7 4",key:"g04rme"}]]);var yt=I("Play",[["polygon",{points:"6 3 20 12 6 21 6 3",key:"1oa8hb"}]]);var wr=I("Search",[["circle",{cx:"11",cy:"11",r:"8",key:"4ej97u"}],["path",{d:"m21 21-4.3-4.3",key:"1qie3q"}]]);var wa=I("Sparkles",[["path",{d:"M9.937 15.5A2 2 0 0 0 8.5 14.063l-6.135-1.582a.5.5 0 0 1 0-.962L8.5 9.936A2 2 0 0 0 9.937 8.5l1.582-6.135a.5.5 0 0 1 .963 0L14.063 8.5A2 2 0 0 0 15.5 9.937l6.135 1.581a.5.5 0 0 1 0 .964L15.5 14.063a2 2 0 0 0-1.437 1.437l-1.582 6.135a.5.5 0 0 1-.963 0z",key:"4pj2yx"}],["path",{d:"M20 3v4",key:"1olli1"}],["path",{d:"M22 5h-4",key:"1gvqau"}],["path",{d:"M4 17v2",key:"vumght"}],["path",{d:"M5 18H3",key:"zchphs"}]]);var aa=I("TriangleAlert",[["path",{d:"m21.73 18-8-14a2 2 0 0 0-3.48 0l-8 14A2 2 0 0 0 4 21h16a2 2 0 0 0 1.73-3",key:"wmoenq"}],["path",{d:"M12 9v4",key:"juzpu7"}],["path",{d:"M12 17h.01",key:"p32p05"}]]);var Cr=I("X",[["path",{d:"M18 6 6 18",key:"1bl5f8"}],["path",{d:"m6 6 12 12",key:"d8bk6v"}]]);var yc=Ue(Ca(),1),lv=Symbol.for("react.element"),iv=Symbol.for("react.fragment"),sv=Object.prototype.hasOwnProperty,uv=yc.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED.ReactCurrentOwner,dv={key:!0,ref:!0,__self:!0,__source:!0};function Sc(e,a,t){var r,o={},n=null,l=null;t!==void 0&&(n=""+t),a.key!==void 0&&(n=""+a.key),a.ref!==void 0&&(l=a.ref);for(r in a)sv.call(a,r)&&!dv.hasOwnProperty(r)&&(o[r]=a[r]);if(e&&e.defaultProps)for(r in a=e.defaultProps,a)o[r]===void 0&&(o[r]=a[r]);return{$$typeof:lv,type:e,key:n,ref:l,props:o,_owner:uv.current}}var qr=iv,i=Sc,v=Sc;var qi={1:"POV transformação outra pessoa",2:"Transformação antes depois",3:"POV guarda roupa provou tudo",4:"Provador especial",5:"Antes depois transformação sala",6:"Provador múltiplos looks",7:"Clube Brand bastidor",8:"Lojas Prata combinações",9:"Hering mesmo look corpos diferentes",10:"AMÊ uma peça duas versões"},Ec=[{title:"Slideshow",copy:"Copiar o ritmo de foto atrás de foto, com uma frase curta em cada tela.",why:"Ajuda a mostrar 3 jeitos sem falar muito e prende quem ainda não conhece a SAF."},{title:"Montagem",copy:"Copiar a montagem com cortes curtos, sempre com corpo inteiro vestida.",why:"Ajuda a mostrar mudança de look sem deixar o vídeo longo."},{title:"Lista",copy:"Copiar a lista na tela, com número grande e frase curta em cada item.",why:"Ajuda quem tem pouco tempo a entender rápido e a salvar para depois."},{title:"Transformação",copy:"Copiar o antes simples e o depois vestida, no mesmo lugar e com a mesma luz.",why:"Ajuda a criar desejo sem prometer corpo perfeito."},{title:"Tutorial",copy:"Copiar o passo a passo calmo, com uma ação por tela.",why:"Ajuda a resolver uma dúvida real e a gerar salvamento."},{title:"Arrume se comigo com história",copy:"Copiar o começo com uma cena do dia, depois vestir com calma.",why:"Ajuda quem ainda não conhece a se reconhecer na cena."},{title:"Provador",copy:"Copiar o provador com corpo inteiro, andando e sentada.",why:"Ajuda a mostrar como fica de verdade, não só no cabide."},{title:"Antes e depois",copy:"Copiar o mesmo look em dois momentos, sem julgar o antes.",why:"Ajuda a mostrar caimento e a gerar identificação."},{title:"Isso ou aquilo",copy:"Copiar a escolha entre 1 e 2, com enquete na tela.",why:"Ajuda a gerar comentário fácil e leve."},{title:"Giro do vestido em câmera lenta",copy:"Copiar o giro com calma, com corpo inteiro e luz na frente.",why:"Ajuda a mostrar o tecido em movimento."},{title:"Embalando pedido com som natural",copy:"Copiar as mãos dobrando e embalando, sem pressa.",why:"Ajuda a mostrar cuidado e a gerar confiança."},{title:"Carrossel de 3 looks",copy:"Copiar 3 looks prontos, um por tela, com a mesma base de luz.",why:"Ajuda quem quer look pronto para a semana."}],kc=[{title:"Upside Down, Diana Ross",why:"Para antes e depois. A troca fica leve e fácil de acompanhar."},{title:"Windows in Brazil",why:"Para transformação 0 a 100. O começo é parado e o depois é leve."},{title:"Pump It, Black Eyed Peas",why:"Para mini vlog. O ritmo ajuda a mostrar o dia com leveza."},{title:"All I Want For Christmas Is You, Mariah Carey",why:"Para Natal leve, sem pressa."},{title:"Underneath the Tree, Kelly Clarkson",why:"Para Natal com carinho, sem pressa."},{title:"Luminary, instrumental",why:"Para momento calmo. Ajuda a olhar o detalhe com atenção."},{title:"My Own Universe",why:"Para momento aconchegante. Ajuda o bastidor a ficar calmo."},{title:"Patient Zero, Taylor Swift",why:"Para história mais sentida. Ajuda a falar de tamanho com carinho."},{title:"Nicole Kidman, ADÉLA",why:"Para look com atitude. Ajuda a mostrar escolha sem julgamento."},{title:"DROP IT LIKE ITS HOT x STAYIN ALIVE, ALTÉGO MIX",why:"Para mensagem animada. Usar só se a oferta estiver confirmada."}],Nc=[{title:"Mesmo look em dois corpos",copy:"Copiar o cuidado de mostrar cada mulher sozinha e depois juntas, sem comparar.",why:"Ajuda a falar de comparação com carinho e a gerar identificação."},{title:"Cliente real",copy:"Copiar a foto com autorização, com a peça bem visível e pouco texto.",why:"Ajuda quem quer ver gente real usando, fora da foto de loja."},{title:"Uma peça, 3 jeitos",copy:"Copiar a mesma peça em 3 momentos do dia, com troca de um detalhe só.",why:"Ajuda a mostrar que a roupa rende mais de uma vez."},{title:"Bastidor com mãos",copy:"Copiar a mesa e as mãos, sem precisar aparecer o tempo todo.",why:"Ajuda a mostrar cuidado quando ela está cansada."}],wc=[{id:"f-24nov",day:"Ter 24/11",time:"17h30",format:"Carrossel",seal:"Base",audience:"para quem já conhece e está pensando",minutes:"10 min",piece:"Conjunto Lara",dor:"Ela abre o guarda roupa e sente que não tem nada para o trabalho.",desejo:"Ela quer se olhar e gostar do que vê.",problema:"Ela não sabe como montar um look rápido para o trabalho.",sensivel:"Ela se compara com a amiga e acha que só a amiga fica bem.",story:["Começo. Ela abre o guarda roupa com pressa e nada parece servir para o trabalho.","Meio. Ela vê o Conjunto Lara vestido e entende para onde ele serve.","Prova. Ela anda, senta e mostra o tecido de perto, como fica de verdade.","Convite. Qual cor você usaria no trabalho?"],how:"Vista o conjunto, ande 3 passos, sente, mostre o tecido de perto.",screenText:"Para o trabalho sem complicação",caption:`Ela abre o guarda roupa e nada serve para o trabalho.
Olha o Conjunto Lara vestido, andando e sentada.
Me conta. Qual cor você usaria no trabalho?
Olha com calma. Sem pressa.`,invite:"Qual cor você usaria no trabalho?",look:"Salvamentos e comentários com a cor em 48 horas.",simple:`Se estiver cansada, faça só vestida andando e sentada.
Sem o detalhe de perto.`,media:6},{id:"f-26nov",day:"Qui 26/11",time:"17h30",format:"Carrossel",seal:"Base",audience:"para quem ainda não conhece",minutes:"10 min",piece:"Mesma peça, dois caimentos",dor:"Ela tem medo de não ficar bem na peça.",desejo:"Ela quer ver que o corpo dela também pode.",problema:"Ela não sabe como a peça fica em outro corpo.",sensivel:"Ela se compara o tempo todo e acha que só um corpo fica bem.",story:["Começo. Duas amigas provam a mesma peça e ficam com medo de comparar.","Meio. Cada uma veste a peça no seu corpo, no seu ritmo.","Prova. Elas andam lado a lado e mostram altura e tamanho na tela.","Convite. Me conta sua altura que eu te ajudo no tamanho."],how:"Vista a peça em cada uma, ande 3 passos, filme lado a lado, mostre de perto.",screenText:"Mesma peça, dois corpos",caption:`A mesma peça em dois corpos diferentes.
Olha como veste andando e como o tecido cai.
Me conta sua altura que eu te ajudo no tamanho.
Sem pressa. Você pode olhar com calma.`,invite:"Me conta sua altura.",look:"Comentários com altura e salvamentos em 48 horas.",simple:`Se a convidada não puder, faça sozinha com dois jeitos de usar.
Mantenha a mesma luz.`,media:9},{id:"f-27nov",day:"Sex 27/11",time:"12h30",format:"Foto única",seal:"Base",audience:"para quem já quer comprar",minutes:"10 min",piece:"Black Friday. Peça principal",dor:"Ela quer aproveitar sem pressão.",desejo:"Ela quer comprar com calma e sem culpa.",problema:"Ela não sabe o que entrou na oferta.",sensivel:"Ela sente culpa por gastar e medo de comprar por impulso.",story:["Começo. Ela separou algumas peças para o dia e quer ver com calma.","Meio. Ela vê a peça principal vestida e entende o que entrou.","Prova. Foto limpa, com espaço para respirar e só o que foi confirmado.","Convite. Comenta QUERO se quiser que eu explique sem pressa."],how:"Vista a peça principal, faça uma foto forte de corpo inteiro, com luz boa.",screenText:"Olha com calma. Só o que foi confirmado",caption:`O que entrou na Black Friday. Confirmar desconto, até quando, cupom e frete.
Peça principal vestida, com calma.
Comenta QUERO que eu te explico sem pressa.
Olha com calma.`,invite:"Comenta QUERO.",look:"Comentários QUERO e cliques em 48 horas.",simple:`Se o desconto não estiver confirmado, não postar este Feed.
Postar só nos Stories. Ainda estou separando, te aviso aqui.`,media:4},{id:"f-28nov",day:"Sáb 28/11",time:"11h00",format:"Reel",seal:"Inovação",audience:"para quem ainda não conhece",minutes:"20 min",piece:"POV. Você abre o guarda roupa e nada te anima",dor:"Ela abre o guarda roupa e nada anima.",desejo:"Ela quer se reencontrar na roupa.",problema:"Ela não sabe por onde começar a se arrumar.",sensivel:"Ela sente cansaço e frustração com o próprio guarda roupa.",story:["Começo. Ela abre o guarda roupa e nada anima. É um dia de preguiça.","Meio. Ela pega uma peça e veste sem pensar muito.","Prova. Ela anda, gira e mostra o detalhe de perto.","Convite. Qual você usaria, 1, 2 ou 3?"],how:"Olhe para a arara, pegue a peça, vista, ande, gire, mostre de perto.",screenText:"Você achou a roupa sem pensar",caption:`Você abre o guarda roupa e nada te anima?
Eu peguei essa peça e olha como ficou.
Me conta. Você usaria 1, 2 ou 3?
Salva para lembrar no dia assim também.`,invite:"Qual você usaria, 1, 2 ou 3?",look:"Salvamentos e compartilhamentos em 48 horas.",simple:`Se estiver cansada, faça só vestida andando e girando.
Sem a cena da arara.`,media:3},{id:"f-01dez",day:"Ter 01/12",time:"17h30",format:"Carrossel",seal:"Base",audience:"para quem ainda não conhece",minutes:"10 min",piece:"Saia Milena",dor:"Ela só tem 5 minutos para se arrumar.",desejo:"Ela quer ficar pronta sem stress.",problema:"Ela acha que precisa de roupa nova para sair.",sensivel:"Ela sente falta de tempo e culpa por não se arrumar.",story:["Começo. Ela tem um compromisso e só tem 5 minutos para se arrumar.","Meio. Ela testa a Saia Milena com 3 partes de cima diferentes.","Prova. Ela anda com cada parte e mostra o tecido de perto.","Convite. Qual você usaria primeiro?"],how:"Vista a saia com cada parte de cima, ande 3 passos, mostre o tecido de perto.",screenText:"1 peça, 3 jeitos de sair rápido",caption:`Ela tem 5 minutos para se arrumar. A Saia Milena resolve.
Olha a saia com 3 partes de cima diferentes.
Qual você usaria primeiro?
Compartilha com quem ama peça versátil.`,invite:"Qual você usaria primeiro?",look:"Compartilhamentos e salvamentos em 48 horas.",simple:`Se estiver cansada, faça 2 jeitos em vez de 3.
Mantenha a mesma primeira frase na tela.`,media:8},{id:"f-03dez",day:"Qui 03/12",time:"17h30",format:"Reel",seal:"Base",audience:"para quem já conhece e está pensando",minutes:"12 min",piece:"Uma peça, 3 jeitos",dor:"Ela comprou uma peça e usa só de um jeito.",desejo:"Ela quer usar mais a roupa que já tem.",problema:"Ela não sabe como combinar a peça em 3 ocasiões.",sensivel:"Ela sente que gastou sem precisar e ficou com dinheiro jogado fora.",story:["Começo. Ela comprou uma peça e achou que só dava para usar de um jeito.","Meio. Ela vê a peça no trabalho, no almoço e na saída.","Prova. Ela anda em cada ocasião e mostra o detalhe de perto.","Convite. Manda para a amiga que diz que não tem roupa."],how:"Vista a peça em 3 ocasiões, ande 3 passos em cada uma, mostre o detalhe.",screenText:"1 peça, 3 compromissos",caption:`Ela achou que essa peça só dava para um jeito.
Trabalho, almoço e saída. Uma peça só.
Manda para a amiga que diz que não tem roupa.
Altura e tamanho. Confirmar antes de publicar.`,invite:"Manda para a amiga que diz que não tem roupa.",look:"Compartilhamentos e salvamentos em 48 horas.",simple:`Se estiver cansada, mostre 2 ocasiões em vez de 3.
Mantenha o mesmo lugar.`,media:10},{id:"f-05dez",day:"Sáb 05/12",time:"11h00",format:"Reel",seal:"Inovação",audience:"para quem já conhece e está pensando",minutes:"20 min",piece:"Bastidor. Antes de uma peça chegar até você",dor:"Ela tem medo de comprar online.",desejo:"Ela quer confiar em quem envia.",problema:"Ela não sabe se a loja cuida da peça antes de enviar.",sensivel:"Ela tem medo de ser enganada e de receber algo diferente.",story:["Começo. Ninguém vê a parte antes da peça chegar. É ali que nasce a confiança.","Meio. Ela vê a mesa, a conferência e a dobra.","Prova. Costura de perto, etiqueta e embalagem.","Convite. O que te deixa insegura para comprar online?"],how:"Mostre a mesa, as mãos conferindo, dobre com calma e embale.",screenText:"Antes de uma peça chegar até você",caption:`Antes de uma peça chegar até você, ela passa por aqui.
Eu confiro, dobro e embalo pensando em quem vai vestir.
Me conta. O que te deixa insegura ao comprar online?
Pode falar sem vergonha. Eu te ajudo.`,invite:"O que te deixa insegura ao comprar roupa online?",look:"Comentários com dúvidas e salvamentos em 48 horas.",simple:`Se estiver cansada, filme só mãos conferindo e embalando.
Sem arrumar a mesa inteira.`,media:7},{id:"f-08dez",day:"Ter 08/12",time:"17h30",format:"Carrossel",seal:"Base",audience:"para quem ainda não conhece",minutes:"10 min",piece:"Dica prática. Regata sem sutiã aparecer",dor:"Ela ama regata, mas o sutiã aparece.",desejo:"Ela quer usar sem se preocupar.",problema:"Ela não sabe resolver sem complicar.",sensivel:"Ela sente vergonha do corpo e desiste de usar a peça.",story:["Começo. Ela ama uma regata, mas o sutiã aparece e ela desiste de usar.","Meio. Ela testa 3 jeitos simples de resolver.","Prova. Ela mostra vestida, de perto, com cada jeito.","Convite. Salva para lembrar e conta qual jeito já usou."],how:"Vista a regata com cada jeito, ande 3 passos, mostre de perto.",screenText:"Salva para lembrar",caption:`Ela ama essa regata, mas o sutiã aparece.
Olha 3 jeitos de resolver sem complicar.
Salva para lembrar quando for usar.
Qual jeito você já usou?`,invite:"Salva para lembrar.",look:"Salvamentos e comentários em 48 horas.",simple:`Se estiver cansada, mostre 2 jeitos em vez de 3.
Fale com calma.`,media:4},{id:"f-10dez",day:"Qui 10/12",time:"17h30",format:"Reel",seal:"Base",audience:"para quem já conhece e está pensando",minutes:"12 min",piece:"Mesma base, dois humores",dor:"Ela não sabe se vai mais arrumada ou mais leve.",desejo:"Ela quer ter escolha sem trocar tudo.",problema:"Ela fica indecisa e pensa que precisa trocar o look inteiro.",sensivel:"Ela tem medo de errar e de se sentir deslocada.",story:["Começo. Ela vai sair e não sabe se vai mais arrumada ou mais leve.","Meio. Ela vê a mesma base com duas trocas.","Prova. Ela anda com cada humor e mostra o detalhe de perto.","Convite. Você é mais 1 ou mais 2?"],how:"Vista a mesma base, troque um detalhe, ande, mostre de perto.",screenText:"Mesma base, dois humores",caption:`Mesma base, dois humores.
Só mudando um detalhe, o look já fica outro.
Você é mais 1 ou mais 2? Comenta aqui.
Salva para testar no seu guarda roupa.`,invite:"Qual é mais você, 1 ou 2?",look:"Comentários e salvamentos em 48 horas.",simple:`Se estiver cansada, mostre os dois humores só com corpo inteiro.
Sem o detalhe bem de pertinho.`,media:5},{id:"f-12dez",day:"Sáb 12/12",time:"11h00",format:"Carrossel",seal:"Base",audience:"para quem já quer comprar",minutes:"12 min",piece:"Guia de medidas na prática",dor:"Ela já comprou online e não serviu.",desejo:"Ela quer comprar sem medo.",problema:"Ela não sabe medir o corpo e a peça.",sensivel:"Ela sente vergonha de pedir ajuda com tamanho.",story:["Começo. Ela já comprou roupa online que não serviu e ficou com medo.","Meio. Ela vê a peça vestida, com altura e tamanho.","Prova. Ela vê vestida, sentada, medidas e tecido de perto.","Convite. Salva o guia e comenta LISTA se quiser aviso."],how:"Vista a peça, mostre medidas, altura e tamanho, sente, mostre de perto.",screenText:"Salva esse guia antes de comprar",caption:`Salva esse guia antes de comprar.
Aqui tem como veste, medidas, altura e tamanho.
Se o seu tamanho não estiver aqui, me chama.
Comenta LISTA que eu te aviso primeiro.`,invite:"Salva esse guia.",look:"Salvamentos e comentários LISTA em 48 horas.",simple:`Se estiver cansada, faça 3 telas. Vestida, medidas e como comprar.
Sem pressa.`,media:4},{id:"f-15dez",day:"Ter 15/12",time:"17h30",format:"Carrossel",seal:"Base",audience:"para quem ainda não conhece",minutes:"10 min",piece:"Ideia para fim de ano sem comprar roupa nova",dor:"Ela tem festa e não quer gastar com roupa nova.",desejo:"Ela quer se sentir bonita com o que já tem.",problema:"Ela acha que precisa de roupa nova para a festa.",sensivel:"Ela sente culpa por gastar em dezembro.",story:["Começo. Ela tem festa e não quer gastar com roupa nova.","Meio. Ela monta 2 looks com peças que já mostrou por aqui.","Prova. Ela anda com cada look e mostra o detalhe de perto.","Convite. Qual você usaria na festa?"],how:"Monte 2 looks com peças conhecidas, ande 3 passos, mostre um detalhe.",screenText:"Festa sem roupa nova",caption:`Ela tem festa e não quer gastar com roupa nova.
Olha 2 looks com peças que você já viu por aqui.
Qual você usaria na festa?
Compartilha com quem quer ideia simples.`,invite:"Qual você usaria na festa?",look:"Compartilhamentos e salvamentos em 48 horas.",simple:`Se estiver cansada, mostre 1 look com 2 jeitos de usar.
Mantenha a mesma luz.`,media:8},{id:"f-17dez",day:"Qui 17/12",time:"17h30",format:"Reel",seal:"Inovação",audience:"para quem ainda não conhece",minutes:"25 min",piece:"Mesmo look, dois corpos",dor:"Ela se compara o tempo todo.",desejo:"Ela quer se sentir bonita do jeito dela.",problema:"Ela acha que só um corpo fica bem no look.",sensivel:"Ela se compara com outras mulheres e sente a autoestima baixa.",story:["Começo. Duas amigas provam o mesmo look e ficam com medo de comparar.","Meio. Elas vestem o look e se olham com carinho.","Prova. Elas andam lado a lado, rindo, e mostram o tecido de perto.","Convite. Manda para quem precisa ver isso."],how:"Vista o mesmo look em cada uma, ande lado a lado, mostre de perto.",screenText:"Mesmo look, dois corpos",caption:`Mesmo look, dois corpos.
Olha como fica lindo em cada uma, do jeito de cada uma.
Não tem corpo errado. Tem peça que ajuda você a se sentir você.
Manda para quem precisa ver isso.`,invite:"Manda para quem precisa ver isso.",look:"Comentários e compartilhamentos em 48 horas.",simple:`Se a convidada não puder, faça sozinha com 2 jeitos de usar o mesmo look.
Com calma.`,media:9},{id:"f-19dez",day:"Sáb 19/12",time:"11h00",format:"Foto única",seal:"Base",audience:"para quem já conhece e está pensando",minutes:"10 min",piece:"Comunidade. Foto de cliente com autorização",dor:"Ela quer ver gente real usando.",desejo:"Ela quer se ver naquela cliente.",problema:"Ela não confia só em foto de loja.",sensivel:"Ela sente que não é só ela que tem dúvida com o corpo.",story:["Começo. Uma cliente usou a peça e mandou foto. Ela ficou feliz e com vergonha.","Meio. Ela vê a foto com carinho.","Prova. Foto com autorização, peça bem visível.","Convite. Marca a amiga que ia amar esse look."],how:"Escolha a foto autorizada, com corpo inteiro e peça bem visível.",screenText:"Qual você usaria? 1, 2 ou 3?",caption:`Olha que linda ela com a peça SAF.
Obrigada por deixar a gente mostrar, com tanto carinho.
Qual você usaria? 1, 2 ou 3?
Marca aquela amiga que ia amar esse look.`,invite:"Marca a amiga que usaria a outra.",look:"Marcações e comentários em 48 horas.",simple:`Se não houver foto autorizada, usar foto com a mesma peça e o mesmo texto.
Sem inventar cliente.`,media:2},{id:"f-22dez",day:"Ter 22/12",time:"17h30",format:"Carrossel",seal:"Base",audience:"para quem já conhece e está pensando",minutes:"10 min",piece:"3 looks prontos para a semana",dor:"Ela está sem ideia para a semana.",desejo:"Ela quer ter look pronto.",problema:"Ela abre o celular sem saber o que vestir.",sensivel:"Ela sente cansaço de fim de ano e falta de tempo.",story:["Começo. Ela está sem ideia para a semana e abre o celular sem saber o que vestir.","Meio. Ela vê 3 looks prontos, com peças que já viu.","Prova. Cada look andando, com um detalhe de perto.","Convite. Qual você usaria amanhã?"],how:"Monte 3 looks prontos, ande 3 passos com cada um, mostre um detalhe.",screenText:"3 looks prontos para a semana",caption:`3 looks prontos para a sua semana.
Sem pensar muito. É só olhar, salvar e escolher.
Qual você usaria amanhã?
Compartilha com quem quer a semana mais fácil.`,invite:"Qual você usaria amanhã?",look:"Compartilhamentos e salvamentos em 48 horas.",simple:`Se estiver cansada, faça 2 looks em vez de 3.
Mantenha o mesmo lugar.`,media:7},{id:"f-24dez",day:"Qui 24/12",time:"17h30",format:"Reel",seal:"Base",audience:"para quem ainda não conhece",minutes:"10 min",piece:"Natal leve. Verde e vermelho",dor:"Ela sente pressão de estar arrumada no Natal.",desejo:"Ela quer ficar confortável e bonita.",problema:"Ela não sabe o que vestir para o Natal em casa.",sensivel:"Ela sente pressão de família e medo de ser julgada.",story:["Começo. Ela vai passar o Natal em casa e quer ficar arrumada sem esforço.","Meio. Dizem por aí que quem usa verde no Natal engravida. Isso é só uma história que circula, sem dizer que é verdade.","Prova. Ela anda, senta e mostra o tecido de perto, em verde e em vermelho.","Convite. Você é time verde ou vermelho?"],how:"Vista o look em verde e em vermelho, ande, sente, mostre de perto.",screenText:"Natal em casa, arrumada sem esforço",caption:`Ela vai passar o Natal em casa e quer ficar arrumada sem esforço.
Olha esse look confortável. Sem aperto, sem pressa.
Você é time verde ou vermelho?
Me conta nos comentários.`,invite:"Você é time verde ou vermelho?",look:"Comentários e salvamentos em 48 horas.",simple:`Se estiver cansada, mostre só corpo inteiro vestida, andando e sentada.
Sem trocar de cor.`,media:5},{id:"f-26dez",day:"Sáb 26/12",time:"11h00",format:"Reel",seal:"Inovação",audience:"para quem já conhece e está pensando",minutes:"20 min",piece:"Bastidor leve. Mais um dia organizando as novidades",dor:"Ela quer saber o que vem em janeiro.",desejo:"Ela quer continuar acompanhando.",problema:"Ela não sabe o que esperar das novidades.",sensivel:"Ela tem medo de perder novidade e de ficar de fora.",story:["Começo. Mais um dia organizando as novidades, depois do Natal, com calma.","Meio. Ela vê a arara, a mesa e a separação.","Prova. Mãos separando, embalando e a peça vestida.","Convite. O que você quer ver em janeiro?"],how:"Mostre a mesa, as mãos organizando, vista uma peça, ande 3 passos.",screenText:"Mais um dia organizando as novidades",caption:`Mais um dia organizando as novidades por aqui.
Sem pressa, com calma, pensando em vocês.
Me conta. O que você quer ver em janeiro?
Vamos juntas?`,invite:"O que você quer ver em janeiro?",look:"Comentários e respostas de Stories em 48 horas.",simple:`Filme só a mesa e as mãos, sem precisar aparecer.
Com calma.`,media:7},{id:"f-27dez",day:"Dom 27/12",time:"17h00",format:"Carrossel",seal:"Base",audience:"para quem já conhece e está pensando",minutes:"10 min",piece:"O que vestir na última semana do ano",dor:"Ela quer fechar o ano se sentindo bem.",desejo:"Ela quer fechar com carinho.",problema:"Ela fica indecisa entre 2 opções simples.",sensivel:"Ela sente cansaço de fim de ano e pressão para estar perfeita.",story:["Começo. Ela quer fechar o ano se sentindo bem, sem começar nada com pressa.","Meio. Ela vê 2 opções simples.","Prova. Cada opção andando, com o detalhe de perto.","Convite. Qual você usaria?"],how:"Mostre 2 opções, ande 3 passos com cada uma, mostre um detalhe.",screenText:"Fechar o ano se sentindo bem",caption:`Ela quer fechar o ano se sentindo bem.
Olha 2 opções simples. Sem pressa, sem regra.
Qual você usaria?
Me conta nos comentários.`,invite:"Qual você usaria?",look:"Comentários e salvamentos em 48 horas.",simple:`Se estiver cansada, mostre 1 opção com 2 jeitos de usar.
Sem pressa.`,media:6},{id:"f-30dez",day:"Qua 30/12",time:"17h00",format:"Carrossel",seal:"Base",audience:"para quem ainda não conhece",minutes:"10 min",piece:"O que aprendemos em dezembro",dor:"Ela perdeu conteúdo de dezembro.",desejo:"Ela quer rever o que mais gostou.",problema:"Ela não viu tudo que passou por aqui.",sensivel:"Ela tem medo de perder o que mais ajudou.",story:["Começo. Ela lembra dos looks, do detalhe e do bastidor de dezembro.","Meio. Ela revê 3 looks que mais ajudaram.","Prova. Detalhe de perto e bastidor.","Convite. O que você quer ver primeiro em janeiro?"],how:"Mostre 3 looks que ela já viu, um detalhe de perto, feche com convite.",screenText:"O que aprendemos em dezembro",caption:`O que aprendemos em dezembro?
Que não tem corpo errado. Tem peça que ajuda você a se sentir você.
Obrigada por estar aqui, por perguntar e por vestir com a gente.
O que você quer ver primeiro em janeiro?`,invite:"O que você quer ver primeiro em janeiro?",look:"Comentários e salvamentos em 48 horas.",simple:`Se estiver cansada, faça 3 telas. 2 looks e 1 convite.
Com calma.`,media:4},{id:"f-31dez",day:"Qui 31/12",time:"17h00",format:"Foto única",seal:"Base",audience:"para quem já conhece e está pensando",minutes:"10 min",piece:"Fechamento do ano",dor:"Ela passou um ano cansativo.",desejo:"Ela quer fechar com leveza.",problema:"Ela sente pressão de estar perfeita no réveillon.",sensivel:"Ela se compara com outras mulheres e sente cansaço.",story:["Começo. Chegou o último dia do ano e dá vontade de agradecer com calma.","Meio. Ela vê um look simples.","Prova. Foto com luz boa, peça bem visível.","Convite. Qual peça você amou este ano?"],how:"Escolha um look simples e amado, faça foto de corpo inteiro, com luz boa.",screenText:"Obrigada por estar aqui",caption:`Obrigada por estar aqui.
Por olhar, por perguntar e por vestir com a gente.
Me conta. Qual peça você amou este ano?
Sem oferta, sem pressa. Só obrigada.`,invite:"Comenta uma peça que você amou este ano.",look:"Comentários com carinho e salvamentos em 48 horas.",simple:`Se estiver cansada, poste a foto com uma frase só. Obrigada por estar aqui.
Sem pressa.`,media:2}],Cc=[{id:"s-23nov",date:"23/11",title:"Novidades",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer ver o que chegou e não sabe por onde começar.",desejo:"Ela quer escolher o que quer ver vestida.",problema:"Ela não sabe o que é novo na SAF.",sensivel:"Ela sente medo de perder novidade.",story:["Começo. Olha o que chegou por aqui.","Meio. Ela vê as peças na arara.","Prova. Ela vê uma peça vestida.","Convite. Qual você quer ver com desconto, 1 ou 2?"],how:"Mostre a arara com calma, vista uma peça, ande 3 passos.",screens:["Olha o que chegou","Vestida, para você ver de verdade","Qual você quer ver? 1 ou 2"],phrase:"Olha o que chegou. Me conta qual você quer ver vestida.",invite:"Comenta LISTA que eu te aviso primeiro.",look:"Respostas e enquetes em 48 horas.",simple:`Se estiver cansada, faça 2 stories. Arara e vestida.
Sem pressa.`,media:2},{id:"s-24nov",date:"24/11",title:"Bastidor separando",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver como você separa as peças.",desejo:"Ela quer confiar no cuidado da loja.",problema:"Ela só vê o post pronto e não vê o cuidado.",sensivel:"Ela tem medo de comprar e receber algo sem cuidado.",story:["Começo. Separei com calma para você ver.","Meio. Ela vê a mesa com as peças.","Prova. Mãos dobrando e detalhe de perto.","Convite. Qual você quer ver vestida?"],how:"Mostre a mesa, filme as mãos dobrando, mostre de perto.",screens:["Separei com calma","Olha como eu separo","Olha esse detalhe de pertinho"],phrase:"Separei com calma. Qual você quer ver vestida?",invite:"Me conta qual você quer ver vestida.",look:"Respostas em 48 horas.",simple:`Se estiver cansada, filme só a mesa e as mãos.
Com calma.`,media:7},{id:"s-25nov",date:"25/11",title:"Detalhe de perto",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver o tecido de perto.",desejo:"Ela quer confiar na qualidade.",problema:"Ela fica com dúvida se o tecido é bom.",sensivel:"Ela tem medo de gastar e se arrepender.",story:["Começo. Chega mais perto para ver esse detalhe.","Meio. Ela vê o tecido com a mão passando.","Prova. Costura e botão de perto.","Convite. Qual detalhe você mais olha?"],how:"Mostre o tecido de perto, passe a mão, mostre costura e botão.",screens:["Olha esse tecido de pertinho","Costura que segura o dia todo","E assim fica vestida"],phrase:"Olha esse detalhe de pertinho. Qual detalhe você mais olha?",invite:"Qual detalhe você mais olha antes de comprar?",look:"Respostas em 48 horas.",simple:`Se estiver cansada, faça só tecido de perto e vestida.
Sem pressa.`,media:5},{id:"s-26nov",date:"26/11",title:"Prova de uso",seal:"Base",audience:"para quem já quer comprar",dor:"Ela quer ver como fica de verdade.",desejo:"Ela quer comprar sem medo.",problema:"Ela tem dúvida de tamanho e caimento.",sensivel:"Ela sente vergonha de perguntar sobre tamanho.",story:["Começo. Hoje eu provo para você ver de verdade.","Meio. Ela vê a peça andando.","Prova. Ela vê como fica sentada.","Convite. Me conta sua altura que eu te ajudo."],how:"Vista a peça, ande 3 passos, sente, mostre como fica.",screens:["Provando para você ver","Sentada também fica boa","Me pergunta o que ficou com dúvida"],phrase:"Provei para você ver. Me conta sua altura que eu te ajudo.",invite:"Me conta sua altura que eu te ajudo.",look:"Perguntas sobre tamanho em 48 horas.",simple:`Se estiver cansada, faça só andando e sentada.
Com calma.`,media:6},{id:"s-27nov",date:"27/11",title:"Black Friday",seal:"Base",audience:"para quem já quer comprar",dor:"Ela quer saber a novidade sem pressão.",desejo:"Ela quer comprar com calma.",problema:"Ela fica com dúvida sobre a oferta.",sensivel:"Ela sente culpa por gastar.",story:["Começo. Hoje tem novidade e eu te mostro com calma.","Meio. Ela vê o anúncio só com o que foi confirmado.","Prova. Detalhe da peça principal de perto.","Convite. Comenta QUERO se quiser explicação."],how:"Mostre a peça principal vestida, com calma, só com o que foi confirmado.",screens:["Hoje tem novidade","Só o que foi confirmado","Olha essa peça de pertinho","Ainda dá para olhar com calma"],phrase:"Hoje tem novidade. Olha com calma. Só o que foi confirmado.",invite:"Comenta QUERO que eu te explico sem pressa.",look:"Comentários QUERO em 48 horas.",simple:`Se não houver confirmação, faça só aquecimento. Sem anúncio de oferta.
Sem pressa.`,media:4},{id:"s-28nov",date:"28/11",title:"Bastidor leve",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer um sábado leve.",desejo:"Ela quer ver um look pronto sem pressa.",problema:"Ela espera post longo e fica sem ideia.",sensivel:"Ela sente cansaço de fim de semana.",story:["Começo. Sábado leve por aqui.","Meio. Ela vê a mesa leve.","Prova. Look do dia, corpo inteiro.","Convite. Qual você usaria?"],how:"Mostre a mesa leve, vista o look, ande 3 passos.",screens:["Sábado leve","Look de sábado","Você usaria? 1 ou 2"],phrase:"Sábado leve. Qual você usaria?",invite:"Me conta qual você usaria.",look:"Enquetes em 48 horas.",simple:`Se estiver cansada, faça só o look.
Com calma.`,media:7},{id:"s-29nov",date:"29/11",title:"Pausa leve",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela precisa de pausa.",desejo:"Ela quer olhar com calma o que mais gostou.",problema:"Ela espera conteúdo todo dia e se cansa.",sensivel:"Ela sente pressão de estar sempre presente.",story:["Começo. Domingo de pausa e de olhar com calma.","Meio. Ela revê a semana.","Prova. Resumo curto da semana.","Convite. O que você mais gostou?"],how:"Use uma foto leve, mostre resumo curto, faça pergunta leve.",screens:["Domingo de pausa","O que você mais gostou?","O que você quer ver na próxima semana?"],phrase:"Domingo de pausa. Me conta o que você mais gostou.",invite:"Me conta o que você mais gostou.",look:"Respostas leves em 48 horas.",simple:`Se estiver cansada, faça 1 story só. Pausa.
Sem culpa.`,media:4},{id:"s-30nov",date:"30/11",title:"Novidades",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer começar a semana com novidade.",desejo:"Ela quer escolher o que quer ver primeiro.",problema:"Ela não sabe o que é novo.",sensivel:"Ela tem medo de perder novidade.",story:["Começo. Nova semana, peça nova para você ver.","Meio. Ela vê a peça na arara.","Prova. Peça vestida, corpo inteiro.","Convite. Qual você quer ver primeiro, 1 ou 2?"],how:"Mostre a arara, vista a peça, ande 3 passos.",screens:["Nova semana, peça nova","Vestida, para você ver","Qual você quer ver primeiro? 1 ou 2"],phrase:"Nova semana, peça nova. Qual você quer ver primeiro?",invite:"Comenta LISTA que eu anoto.",look:"Respostas em 48 horas.",simple:`Se estiver cansada, faça arara e vestida.
Sem pressa.`,media:2},{id:"s-01dez",date:"01/12",title:"Bastidor sem aparecer",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver bastidor sem você precisar aparecer.",desejo:"Ela quer ver o cuidado sem pressa.",problema:"Ela pensa que precisa aparecer sempre.",sensivel:"Ela sente cansaço de se expor todo dia.",story:["Começo. Hoje eu te mostro sem pressa.","Meio. Ela vê mesa e mãos.","Prova. Detalhe de perto.","Convite. Qual peça você quer ver vestida?"],how:"Filme mesa e mãos, mostre o detalhe de perto.",screens:["Bastidor sem aparecer","Organizando com calma","Olha esse detalhe"],phrase:"Bastidor sem aparecer. Qual peça você quer ver vestida?",invite:"Qual peça você quer ver vestida?",look:"Respostas em 48 horas.",simple:`Se estiver cansada, filme só as mãos.
Com calma.`,media:7},{id:"s-02dez",date:"02/12",title:"Dica e bastidor",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer dica que facilita o dia.",desejo:"Ela quer se sentir leve com a roupa.",problema:"Ela não sabe usar saia midi com tênis.",sensivel:"Ela tem medo de errar a combinação.",story:["Começo. Hoje tem dica que facilita seu dia.","Meio. Ela vê saia midi com tênis, vestida.","Prova. Ela vê andando, sem complicar.","Convite. Salva para lembrar."],how:"Vista saia midi com tênis, ande 3 passos, mostre como fica leve.",screens:["Saia midi com tênis","Andando, fica assim","Você usaria assim? Sim ou ainda não"],phrase:"Saia midi com tênis. Salva para lembrar.",invite:"Salva para lembrar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, faça só vestida andando.
Sem pressa.`,media:8},{id:"s-03dez",date:"03/12",title:"Apoio ao Feed",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver o que ficou de fora do Feed.",desejo:"Ela quer entender o detalhe que não entrou.",problema:"Ela vê só o post e fica com dúvida.",sensivel:"Ela sente que perdeu algo importante.",story:["Começo. Esse Feed de hoje tem um extra aqui.","Meio. Ela vê um detalhe que não entrou.","Prova. Dúvida comum com resposta curta.","Convite. Olha o Feed com calma."],how:"Mostre o detalhe, responda uma dúvida curta.",screens:["Não entrou no Feed","Dúvida comum, resposta curta","Olha o Feed com calma"],phrase:"Tem extra aqui. Olha o Feed com calma.",invite:"Olha o Feed de hoje com calma.",look:"Cliques no Feed em 48 horas.",simple:`Se estiver cansada, faça só o detalhe.
Com calma.`,media:2},{id:"s-04dez",date:"04/12",title:"Dica prática",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer usar regata sem o sutiã aparecer.",desejo:"Ela quer usar sem se preocupar.",problema:"Ela desiste da regata por causa do sutiã.",sensivel:"Ela sente vergonha do corpo.",story:["Começo. Dica rápida para o seu dia.","Meio. Ela vê o jeito 1.","Prova. Jeito 2 e jeito 3, vestida.","Convite. Salva para lembrar."],how:"Vista a regata com cada jeito, mostre de perto.",screens:["Regata sem sutiã aparecer","Jeito 2, também simples","Salva para lembrar"],phrase:"Regata sem sutiã aparecer. Salva para lembrar.",invite:"Salva para lembrar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, mostre 2 jeitos.
Sem pressa.`,media:4},{id:"s-05dez",date:"05/12",title:"Prévia e comunidade",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver o que vem e gente real.",desejo:"Ela quer se ver na cliente.",problema:"Ela só vê peça nova e não vê gente real.",sensivel:"Ela sente que não é só ela.",story:["Começo. Sexta com prévia do que vem por aí.","Meio. Ela vê a prévia.","Prova. Foto de cliente com autorização.","Convite. O que você quer ver?"],how:"Mostre a prévia, mostre a foto autorizada com carinho.",screens:["Prévia da próxima semana","Ela usou e ficou linda","O que você quer ver no sábado?"],phrase:"Prévia e comunidade. Me conta o que você quer ver.",invite:"Me conta o que você quer ver.",look:"Respostas em 48 horas.",simple:`Se estiver cansada, faça só a prévia.
Com calma.`,media:2},{id:"s-06dez",date:"06/12",title:"Bastidor leve",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer sábado com look pronto.",desejo:"Ela quer escolher um look sem pensar muito.",problema:"Ela fica sem ideia de look.",sensivel:"Ela sente cansaço de fim de semana.",story:["Começo. Sábado com calma e com look pronto.","Meio. Ela vê a arara com looks prontos.","Prova. Um look vestido, andando.","Convite. Qual você usaria amanhã?"],how:"Mostre a arara, vista um look, ande 3 passos.",screens:["Looks prontos de sábado","Pronto para sair","Você usaria? 1 ou 2"],phrase:"Looks prontos de sábado. Qual você usaria amanhã?",invite:"Qual você usaria amanhã?",look:"Enquetes em 48 horas.",simple:`Se estiver cansada, faça só o look.
Com calma.`,media:8},{id:"s-07dez",date:"07/12",title:"Pausa leve",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela precisa respirar.",desejo:"Ela quer responder sem obrigação.",problema:"Ela espera conteúdo todo dia.",sensivel:"Ela sente pressão de estar sempre presente.",story:["Começo. Domingo é dia de respirar.","Meio. Ela vê pausa ou 1 story leve.","Prova. Pergunta leve, sem obrigação.","Convite. Só pausa."],how:"Use 1 story leve ou nenhum, faça pergunta leve.",screens:["Domingo de pausa","Como foi sua semana?"],phrase:"Domingo de pausa. Como foi sua semana?",invite:"Sem convite hoje. Só pausa.",look:"Respostas leves em 48 horas.",simple:`Se estiver cansada, não poste. Pausa.
Sem culpa.`,media:7},{id:"s-08dez",date:"08/12",title:"Novidades",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer novidade boa.",desejo:"Ela quer escolher o que quer ver com mais detalhe.",problema:"Ela não sabe o que chegou.",sensivel:"Ela tem medo de perder novidade.",story:["Começo. Começo de semana com novidade boa.","Meio. Ela vê a peça na arara.","Prova. Peça vestida, corpo inteiro.","Convite. Qual você quer ver com mais detalhe, 1 ou 2?"],how:"Mostre a arara, vista a peça, ande 3 passos.",screens:["Novidade boa","Vestida para você ver","Qual você quer ver com mais detalhe? 1 ou 2"],phrase:"Novidade boa. Qual você quer ver com mais detalhe?",invite:"Comenta LISTA que eu te aviso.",look:"Respostas em 48 horas.",simple:`Se estiver cansada, faça arara e vestida.
Sem pressa.`,media:2},{id:"s-09dez",date:"09/12",title:"Bastidor e dica",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer dica de cinto sem apertar.",desejo:"Ela quer ficar confortável com cinto.",problema:"Ela aperta o cinto e fica desconfortável.",sensivel:"Ela tem medo de marcar o corpo.",story:["Começo. Hoje o bastidor é só mesa e mãos.","Meio. Ela vê a dica de cinto, vestida.","Prova. Como fica sentada com o cinto.","Convite. Salva para lembrar."],how:"Mostre mesa e mãos, vista com cinto, sente, mostre como fica.",screens:["Mesa e mãos","Cinto sem apertar","Sentada, fica assim"],phrase:"Cinto sem apertar. Salva para lembrar.",invite:"Salva para lembrar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, mostre só vestida.
Com calma.`,media:7},{id:"s-10dez",date:"10/12",title:"Apoio ao Feed",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver o que ficou de fora.",desejo:"Ela quer entender o detalhe extra.",problema:"Ela vê só o post e fica com dúvida.",sensivel:"Ela sente que perdeu algo.",story:["Começo. Olha o que ficou de fora do Feed.","Meio. Ela vê o detalhe.","Prova. Dúvida comum com resposta curta.","Convite. Olha o Feed com calma."],how:"Mostre o detalhe, responda uma dúvida curta.",screens:["Ficou de fora do Feed","Dúvida e resposta curta","Olha o Feed com calma"],phrase:"Ficou de fora do Feed. Olha com calma.",invite:"Olha o Feed de hoje com calma.",look:"Cliques no Feed em 48 horas.",simple:`Se estiver cansada, faça só o detalhe.
Com calma.`,media:5},{id:"s-11dez",date:"11/12",title:"Dica prática",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer deixar a blusa por dentro sem marcar.",desejo:"Ela quer se sentir arrumada sem aperto.",problema:"Ela desiste porque a blusa marca.",sensivel:"Ela sente vergonha de marcar o corpo.",story:["Começo. Dica de hoje, simples e rápida.","Meio. Ela vê o jeito 1.","Prova. Jeito 2, vestida, e andando.","Convite. Salva para lembrar."],how:"Vista a blusa por dentro com cada jeito, ande 3 passos.",screens:["Blusa por dentro sem marcar","Jeito 2, também simples","Andando, fica assim"],phrase:"Blusa por dentro sem marcar. Salva para lembrar.",invite:"Salva para lembrar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, mostre 1 jeito.
Com calma.`,media:4},{id:"s-12dez",date:"12/12",title:"Comunidade",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver quem já usou.",desejo:"Ela quer se ver na cliente.",problema:"Ela só vê foto de loja.",sensivel:"Ela sente que não é só ela.",story:["Começo. Sexta com foto de quem já usou.","Meio. Ela vê a foto com autorização.","Prova. Detalhe da peça que ela usou.","Convite. Marca uma amiga."],how:"Mostre a foto autorizada, mostre o detalhe da peça.",screens:["Ela usou e ficou linda","Olha essa peça de pertinho","Marca quem usaria também"],phrase:"Ela usou e ficou linda. Marca quem usaria também.",invite:"Marca a amiga que usaria a outra.",look:"Marcações em 48 horas.",simple:`Se não houver foto autorizada, use foto com a mesma peça.
Sem inventar cliente.`,media:2},{id:"s-13dez",date:"13/12",title:"Bastidor leve",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer sábado leve.",desejo:"Ela quer ver um look leve.",problema:"Ela espera post longo.",sensivel:"Ela sente cansaço de fim de semana.",story:["Começo. Sábado leve, look leve.","Meio. Ela vê o look, corpo inteiro.","Prova. Detalhe de perto.","Convite. Qual você usaria?"],how:"Vista o look, ande 3 passos, mostre o detalhe de perto.",screens:["Sábado leve","Detalhe de pertinho","Você usaria? 1 ou 2"],phrase:"Sábado leve. Qual você usaria?",invite:"Qual você usaria?",look:"Enquetes em 48 horas.",simple:`Se estiver cansada, faça só o look.
Com calma.`,media:6},{id:"s-14dez",date:"14/12",title:"Pausa leve",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer olhar a semana com calma.",desejo:"Ela quer escolher o que mais gostou.",problema:"Ela vê tudo solto e se perde.",sensivel:"Ela sente pressão de acompanhar tudo.",story:["Começo. Domingo para olhar a semana com calma.","Meio. Ela revê os melhores da semana.","Prova. Pergunta para a próxima semana.","Convite. O que você mais gostou?"],how:"Use pausa ou 1 story leve, mostre os melhores da semana.",screens:["Domingo de pausa","O que você mais gostou?","O que você quer ver?"],phrase:"Domingo de pausa. Me conta o que você mais gostou.",invite:"Me conta o que você mais gostou.",look:"Respostas em 48 horas.",simple:`Se estiver cansada, faça 1 story só.
Sem culpa.`,media:4},{id:"s-15dez",date:"15/12",title:"Novidades",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer novidade da semana.",desejo:"Ela quer escolher o que quer ver primeiro.",problema:"Ela não sabe o que chegou.",sensivel:"Ela tem medo de perder novidade.",story:["Começo. Novidade da semana chegou.","Meio. Ela vê a peça na arara.","Prova. Peça vestida, corpo inteiro.","Convite. Qual você quer ver primeiro, 1 ou 2?"],how:"Mostre a arara, vista a peça, ande 3 passos.",screens:["Novidade da semana","Vestida para você ver","Qual você quer ver primeiro? 1 ou 2"],phrase:"Novidade da semana. Qual você quer ver primeiro?",invite:"Comenta LISTA que eu anoto.",look:"Respostas em 48 horas.",simple:`Se estiver cansada, faça arara e vestida.
Sem pressa.`,media:2},{id:"s-16dez",date:"16/12",title:"Bastidor",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer aprender a dobrar para viajar.",desejo:"Ela quer viajar sem amassar a roupa.",problema:"Ela amassa tudo na mala.",sensivel:"Ela sente culpa por não cuidar da roupa.",story:["Começo. Hoje eu mostro como eu separo tudo.","Meio. Ela vê as mãos separando.","Prova. Dica de dobrar para viajar.","Convite. Salva para lembrar."],how:"Mostre as mãos separando, dobre com calma, mostre a peça guardada.",screens:["Separando com calma","Dobrar para viajar sem amassar","Pronta para a mala"],phrase:"Como dobrar para viajar sem amassar. Salva para lembrar.",invite:"Salva para lembrar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, mostre só como dobrar.
Com calma.`,media:7},{id:"s-17dez",date:"17/12",title:"Apoio ao Feed",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver o detalhe do Feed.",desejo:"Ela quer entender como fica andando.",problema:"Ela vê só o post e fica com dúvida.",sensivel:"Ela sente que perdeu o detalhe.",story:["Começo. Tem detalhe do Feed de hoje aqui.","Meio. Ela vê o detalhe que não entrou.","Prova. Como fica andando, com calma.","Convite. Olha o Feed com calma."],how:"Mostre o detalhe, mostre andando com calma.",screens:["Detalhe do Feed de hoje","Andando, fica assim","Olha o Feed com calma"],phrase:"Tem detalhe aqui. Olha o Feed com calma.",invite:"Olha o Feed de hoje com calma.",look:"Cliques no Feed em 48 horas.",simple:`Se estiver cansada, faça só o detalhe.
Com calma.`,media:9},{id:"s-18dez",date:"18/12",title:"Dica prática",seal:"Base",audience:"para quem já quer comprar",dor:"Ela fica entre dois tamanhos.",desejo:"Ela quer escolher sem medo.",problema:"Ela não sabe qual tamanho escolher.",sensivel:"Ela sente vergonha de pedir ajuda com tamanho.",story:["Começo. Dica para você não errar o tamanho.","Meio. Ela vê como escolher entre dois tamanhos.","Prova. As duas medidas lado a lado.","Convite. Me conta sua altura que eu te ajudo."],how:"Mostre as duas medidas lado a lado, fale com calma.",screens:["Entre dois tamanhos","Olha as duas medidas","Me conta sua altura que eu te ajudo"],phrase:"Entre dois tamanhos. Me conta sua altura que eu te ajudo.",invite:"Me conta sua altura que eu te ajudo.",look:"Perguntas sobre tamanho em 48 horas.",simple:`Se estiver cansada, mostre só as duas medidas.
Com calma.`,media:4},{id:"s-19dez",date:"19/12",title:"Comunidade",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver comunidade com carinho.",desejo:"Ela quer se ver na cliente.",problema:"Ela só vê foto de loja.",sensivel:"Ela sente que não é só ela.",story:["Começo. Sexta com comunidade e com carinho.","Meio. Ela vê a foto com autorização.","Prova. Peça vestida, com calma.","Convite. Marca uma amiga."],how:"Mostre a foto autorizada, mostre vestida com calma.",screens:["Ela usou e ficou linda","Vestida, para você ver","Marca quem usaria também"],phrase:"Ela usou e ficou linda. Marca quem usaria também.",invite:"Marca a amiga que usaria também.",look:"Marcações em 48 horas.",simple:`Se não houver foto autorizada, use foto com a mesma peça.
Sem inventar cliente.`,media:2},{id:"s-20dez",date:"20/12",title:"Bastidor leve",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer look de festa sem esforço.",desejo:"Ela quer ficar bonita sem aperto.",problema:"Ela pensa que festa pede aperto.",sensivel:"Ela sente pressão de estar arrumada.",story:["Começo. Sábado com look de festa sem esforço.","Meio. Ela vê o look, corpo inteiro.","Prova. Detalhe de perto.","Convite. Qual você usaria na festa?"],how:"Vista o look, ande 3 passos, mostre o detalhe de perto.",screens:["Festa sem esforço","Detalhe de pertinho","Você usaria? 1 ou 2"],phrase:"Festa sem esforço. Qual você usaria na festa?",invite:"Qual você usaria na festa?",look:"Enquetes em 48 horas.",simple:`Se estiver cansada, faça só o look.
Com calma.`,media:5},{id:"s-21dez",date:"21/12",title:"Novidades leves",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer novidade leve para o Natal.",desejo:"Ela quer escolher se é time verde ou vermelho.",problema:"Ela não sabe o que chegou.",sensivel:"Ela sente pressão de Natal.",story:["Começo. Novidade leve para a semana do Natal.","Meio. Ela vê a peça leve na arara.","Prova. Peça vestida, corpo inteiro.","Convite. Você é time verde ou vermelho?"],how:"Mostre a arara, vista a peça, ande 3 passos.",screens:["Novidade leve","Vestida, para você ver","Você é time verde ou vermelho?"],phrase:"Novidade leve. Você é time verde ou vermelho?",invite:"Você é time verde ou vermelho?",look:"Enquetes em 48 horas.",simple:`Se estiver cansada, faça arara e vestida.
Sem pressa.`,media:4},{id:"s-22dez",date:"22/12",title:"Bastidor",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver bastidor de Natal sem correria.",desejo:"Ela quer ver calma antes do Natal.",problema:"Ela espera correria e se cansa.",sensivel:"Ela sente cansaço de fim de ano.",story:["Começo. Bastidor de Natal, sem correria.","Meio. Ela vê a mesa de Natal.","Prova. Mãos separando e detalhe de perto.","Convite. Qual você quer ver vestida?"],how:"Mostre a mesa, filme as mãos, mostre o detalhe de perto.",screens:["Bastidor de Natal","Separando com calma","Olha esse detalhe"],phrase:"Bastidor de Natal, sem correria. Qual você quer ver vestida?",invite:"Qual peça você quer ver vestida?",look:"Respostas em 48 horas.",simple:`Se estiver cansada, filme só a mesa.
Com calma.`,media:7},{id:"s-23dez",date:"23/12",title:"Dica de look",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer combinar marrom com bege.",desejo:"Ela quer se sentir confortável no Natal em casa.",problema:"Ela tem dúvida de combinar as cores.",sensivel:"Ela tem medo de errar a combinação.",story:["Começo. Dica de look para o Natal em casa.","Meio. Ela vê marrom com bege, vestida.","Prova. Como fica andando e sentada.","Convite. Você é time verde ou vermelho?"],how:"Vista marrom com bege, ande 3 passos, sente, mostre como fica.",screens:["Marrom com bege","Confortável para o Natal em casa","Você é time verde ou vermelho?"],phrase:"Marrom com bege. Confortável para o Natal em casa.",invite:"Você é time verde ou vermelho?",look:"Enquetes em 48 horas.",simple:`Se estiver cansada, faça só vestida.
Com calma.`,media:8},{id:"s-24dez",date:"24/12",title:"História do verde e do vermelho",seal:"Inovação",audience:"para quem ainda não conhece",dor:"Ela quer brincar com verde e vermelho sem pressão.",desejo:"Ela quer escolher a cor que deixa ela mais ela.",problema:"Ela ouve uma história e fica com dúvida.",sensivel:"Ela sente pressão sobre maternidade e sobre o corpo.",story:["Começo. Hoje tem história leve de Natal.","Meio. Dizem por aí que quem usa verde no Natal engravida. Isso é só uma história que circula, sem dizer que é verdade.","Prova. Se você quer, temos vestido verde lindo. Se você não quer, passa de vermelho que também é lindo.","Convite. Responde a enquete. Time verde ou vermelho?"],how:"Fale leve, mostre o verde, mostre o vermelho, faça enquete.",screens:["Dizem por aí","Temos vestido verde lindo","Vermelho também é lindo","Você é time verde ou vermelho?"],phrase:"Dizem por aí. Verde ou vermelho, os dois são lindos.",invite:"Responde a enquete. Time verde ou vermelho?",look:"Enquetes em 48 horas.",simple:`Se estiver cansada, faça 2 stories. Verde e vermelho.
Sem pressa.`,media:5},{id:"s-25dez",date:"25/12",title:"Pausa de Natal",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer Natal com calma.",desejo:"Ela quer passar sem pressão.",problema:"Ela espera conteúdo e se cansa.",sensivel:"Ela sente pressão de estar arrumada no Natal.",story:["Começo. Natal com calma, pode ser de pijama.","Meio. Ela vê pausa ou 1 story leve.","Prova. Look confortável, corpo inteiro, sem pressa.","Convite. Só pausa."],how:"Use pausa ou 1 story leve, mostre look confortável.",screens:["Natal com calma","Pode passar de pijama, tudo bem também"],phrase:"Natal com calma. Pode ser de pijama.",invite:"Sem convite hoje. Só pausa.",look:"Sem medida hoje. Só pausa.",simple:`Se estiver cansada, não poste. Pausa.
Sem culpa.`,media:7},{id:"s-26dez",date:"26/12",title:"Novidades",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer organizar junto depois do Natal.",desejo:"Ela quer ver o que vem em janeiro.",problema:"Ela não sabe o que vem.",sensivel:"Ela tem medo de perder novidade.",story:["Começo. Depois do Natal, a gente organiza juntas.","Meio. Ela vê a arara com novidades.","Prova. Peça vestida, corpo inteiro.","Convite. O que você quer ver em janeiro?"],how:"Mostre a arara, vista a peça, ande 3 passos.",screens:["Depois do Natal","Vestida, para você ver","O que você quer ver em janeiro?"],phrase:"Depois do Natal. O que você quer ver em janeiro?",invite:"Me conta o que você quer ver em janeiro.",look:"Respostas em 48 horas.",simple:`Se estiver cansada, faça arara e vestida.
Sem pressa.`,media:2},{id:"s-27dez",date:"27/12",title:"Bastidor leve",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver o que vem por aí.",desejo:"Ela quer escolher um look.",problema:"Ela não sabe o que vem.",sensivel:"Ela tem medo de perder novidade.",story:["Começo. Sábado com o que vem por aí.","Meio. Ela vê a mesa com peças separadas.","Prova. Um look vestido, andando.","Convite. Qual você usaria?"],how:"Mostre a mesa, vista o look, ande 3 passos.",screens:["Sábado com calma","Olha esse look","Você usaria? 1 ou 2"],phrase:"Sábado com calma. Qual você usaria?",invite:"Qual você usaria?",look:"Enquetes em 48 horas.",simple:`Se estiver cansada, faça só o look.
Com calma.`,media:6},{id:"s-28dez",date:"28/12",title:"Novidades",seal:"Base",audience:"para quem ainda não conhece",dor:"Ela quer novidade boa na última semana.",desejo:"Ela quer escolher o que quer ver primeiro.",problema:"Ela não sabe o que chegou.",sensivel:"Ela tem medo de perder novidade.",story:["Começo. Última semana do ano, novidade boa.","Meio. Ela vê a peça na arara.","Prova. Peça vestida, corpo inteiro.","Convite. Qual você quer ver primeiro, 1 ou 2?"],how:"Mostre a arara, vista a peça, ande 3 passos.",screens:["Última semana do ano","Vestida, para você ver","Qual você quer ver primeiro? 1 ou 2"],phrase:"Última semana do ano. Qual você quer ver primeiro?",invite:"Comenta LISTA que eu anoto.",look:"Respostas em 48 horas.",simple:`Se estiver cansada, faça arara e vestida.
Sem pressa.`,media:2},{id:"s-29dez",date:"29/12",title:"Enquete do ano",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer contar o que mais ajudou.",desejo:"Ela quer rever o que mais gostou.",problema:"Ela vê tudo solto e não lembra.",sensivel:"Ela sente que perdeu algo.",story:["Começo. Me conta o que mais te ajudou.","Meio. Ela vê a enquete.","Prova. Comentários das clientes, com carinho.","Convite. O que você quer ver em janeiro?"],how:"Faça enquete, mostre comentários com carinho.",screens:["Qual conteúdo ajudou mais?","Vocês me ajudaram muito","O que você quer ver em janeiro?"],phrase:"Qual conteúdo ajudou mais? Me conta.",invite:"Me conta o que mais te ajudou.",look:"Enquetes e comentários em 48 horas.",simple:`Se estiver cansada, faça só a enquete.
Com calma.`,media:4},{id:"s-30dez",date:"30/12",title:"Bastidor",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer ver o fechamento do ano.",desejo:"Ela quer fechar com carinho.",problema:"Ela vê tudo solto.",sensivel:"Ela sente cansaço de fim de ano.",story:["Começo. Bastidor do fechamento do ano.","Meio. Ela vê a mesa com as peças do ano.","Prova. Mãos separando e embalando.","Convite. Qual foi sua peça amada do ano?"],how:"Mostre a mesa, filme as mãos, feche com obrigada.",screens:["Fechamento do ano","Separando com calma","Obrigada por estar aqui"],phrase:"Fechamento do ano. Obrigada por estar aqui.",invite:"Me conta sua peça amada do ano.",look:"Respostas em 48 horas.",simple:`Se estiver cansada, filme só a mesa.
Com calma.`,media:7},{id:"s-31dez",date:"31/12",title:"Agradecimento",seal:"Base",audience:"para quem já conhece e está pensando",dor:"Ela quer fechar o ano com carinho.",desejo:"Ela quer fechar com leveza.",problema:"Ela espera oferta e se cansa.",sensivel:"Ela sente cansaço e comparação.",story:["Começo. Obrigada por estar aqui este ano.","Meio. Ela vê look simples, corpo inteiro.","Prova. Agradecimento, com calma.","Convite. Qual peça você amou?"],how:"Mostre look simples, fale com calma, agradeça.",screens:["Obrigada por estar aqui","Vamos juntas em janeiro?","Qual peça você amou este ano?"],phrase:"Obrigada por estar aqui. Vamos juntas em janeiro?",invite:"Comenta uma peça que você amou este ano.",look:"Comentários com carinho em 48 horas.",simple:`Se estiver cansada, poste 1 foto com 1 frase.
Com calma.`,media:2}],qc=[{id:"m-1",title:"1. Como medir o busto",dor:"Ela tem dúvida de onde medir.",desejo:"Ela quer medir sem apertar e anotar com calma.",problema:"Ela mede no lugar errado.",sensivel:"Ela sente vergonha de pedir ajuda com tamanho.",story:["Começo. Vista uma roupa leve.","Meio. Passe a fita na altura do busto.","Prova. Deixe a fita reta nas costas.","Convite. Anote com calma e salve."],how:"Vista roupa leve, passe a fita sem apertar, mostre onde fica a fita.",screen:"Mede aqui, sem apertar",phrase:"Mede aqui, sem apertar. Anota com calma.",invite:"Salva esse destaque antes de comprar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, faça só foto da fita no lugar certo.
Com calma.`,media:5},{id:"m-2",title:"2. Como medir a cintura",dor:"Ela confunde cintura com quadril.",desejo:"Ela quer achar a parte mais fina.",problema:"Ela mede no lugar errado.",sensivel:"Ela sente vergonha de perguntar.",story:["Começo. Ache a parte mais fina, acima do umbigo.","Meio. Passe a fita ali, sem apertar.","Prova. Respire normal e anote.","Convite. Salve para comparar depois."],how:"Mostre a parte mais fina, passe a fita sem apertar, mostre de perto.",screen:"É aqui, não no quadril",phrase:"É aqui, não no quadril. Salva para lembrar.",invite:"Salva esse destaque antes de comprar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, faça só foto da fita no lugar certo.
Com calma.`,media:5},{id:"m-3",title:"3. Como medir o quadril",dor:"Ela tem dúvida de onde é o quadril.",desejo:"Ela quer medir na parte mais larga.",problema:"Ela mede muito em cima ou muito embaixo.",sensivel:"Ela sente vergonha de medir o corpo.",story:["Começo. Ache a parte mais larga do quadril.","Meio. Passe a fita ali, sem apertar.","Prova. Deixe a fita reta e anote.","Convite. Salve para comparar depois."],how:"Mostre a parte mais larga, passe a fita sem apertar, mostre de perto.",screen:"Mede na parte mais larga",phrase:"Mede na parte mais larga. Anota com calma.",invite:"Salva esse destaque antes de comprar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, faça só foto da fita no lugar certo.
Com calma.`,media:5},{id:"m-4",title:"4. Como medir o comprimento",dor:"Ela não sabe se fica curta ou longa.",desejo:"Ela quer saber como fica nela.",problema:"Ela compra sem comparar o comprimento.",sensivel:"Ela tem medo de errar de novo.",story:["Começo. Meça do ombro até onde a peça termina.","Meio. Use a fita reta, sem puxar.","Prova. Anote para comparar com a tabela.","Convite. Salve para lembrar."],how:"Mostre a fita do ombro até o fim, reta, sem puxar.",screen:"Assim você sabe se fica curta ou longa",phrase:"Mede do ombro até o fim. Assim você sabe se fica curta ou longa.",invite:"Salva esse destaque antes de comprar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, faça só foto da fita no comprimento.
Com calma.`,media:5},{id:"m-5",title:"5. Entre dois tamanhos",dor:"Ela fica entre dois tamanhos.",desejo:"Ela quer escolher sem medo.",problema:"Ela escolhe no chute.",sensivel:"Ela sente vergonha de pedir ajuda.",story:["Começo. Meça busto, cintura e quadril.","Meio. Coloque as duas medidas lado a lado.","Prova. Veja onde aperta mais.","Convite. Me conta sua altura e onde aperta mais."],how:"Mostre as duas medidas lado a lado, mostre onde aperta mais.",screen:"Se você está entre dois, me conta sua altura",phrase:"Entre dois tamanhos? Me conta sua altura e onde aperta mais.",invite:"Salva esse destaque antes de comprar.",look:"Perguntas sobre tamanho em 48 horas.",simple:`Se estiver cansada, mostre só as duas medidas lado a lado.
Com calma.`,media:4},{id:"m-6",title:"6. Como fica sentada",dor:"Ela só vê a peça em pé.",desejo:"Ela quer ficar confortável sentada.",problema:"Ela não sabe se aperta sentada.",sensivel:"Ela tem medo de comprar e não usar.",story:["Começo. Vista a peça.","Meio. Sente em uma cadeira.","Prova. Veja se aperta, se sobe ou se marca.","Convite. Salve para lembrar."],how:"Vista a peça, sente, mostre como fica e onde pode apertar.",screen:"Sentada também precisa ficar confortável",phrase:"Sentada também precisa ficar confortável. Salva para lembrar.",invite:"Salva esse destaque antes de comprar.",look:"Salvamentos em 48 horas.",simple:`Se estiver cansada, faça só sentada.
Com calma.`,media:6},{id:"m-7",title:"7. Como pedir ajuda",dor:"Ela tem vergonha de perguntar.",desejo:"Ela quer mandar as medidas e receber ajuda.",problema:"Ela fica com dúvida sozinha.",sensivel:"Ela sente vergonha de pedir ajuda com tamanho.",story:["Começo. Abra a caixinha nos Stories.","Meio. Manda suas medidas.","Prova. Conta sua altura e onde aperta mais.","Convite. Eu te ajudo a escolher, sem pressa."],how:"Mostre a caixinha na tela, fale como fala com uma amiga.",screen:"Me manda suas medidas que eu te ajudo",phrase:"Me manda suas medidas que eu te ajudo a escolher, sem pressa.",invite:"Salva esse destaque antes de comprar.",look:"Perguntas sobre tamanho em 48 horas.",simple:`Se estiver cansada, faça só 1 tela com a frase.
Com calma.`,media:4}];async function xi(e,a){let t=await fetch("actions",{method:"POST",headers:{"content-type":"application/json"},body:JSON.stringify({action:e,args:a})}),r=await t.json().catch(()=>({}));if(!t.ok||r.error)throw Error(r.error||"Não foi possível salvar agora.");return r.data}function Mi(){let[e,a]=ae.useState("inicio"),[t,r]=ae.useState("todos"),[o,n]=ae.useState(""),[l,s]=ae.useState("f-24nov"),[u,c]=ae.useState(new Set),[h,y]=ae.useState(null),[g,k]=ae.useState(""),[N,C]=ae.useState(!1),[b,m]=ae.useState(""),[d,f]=ae.useState(1),[E,q]=ae.useState(!1),x=ae.useRef(null);ae.useEffect(()=>{x.current=h},[h]),ae.useEffect(()=>{let p=!0;return(async()=>{try{let T=((await xi("records.list",{collection:"saf-checklist"}))?.records||[])[0];if(p&&T)y(T.id),c(new Set(T.value?.checked||[]))}catch{if(p)k("Sem conexão com o salvamento. Sua marcação vai ficar só nesta tela por enquanto.")}})(),()=>{p=!1}},[]);async function M(p){let P=new Set(u);if(P.has(p))P.delete(p);else P.add(p);c(P),C(!0),k("");try{let T={checked:[...P]};if(x.current)await xi("records.replace",{collection:"saf-checklist",id:x.current,value:T});else{let Qe=await xi("records.create",{collection:"saf-checklist",value:T,idempotency_key:"saf-checklist-v1"});if(Qe?.id)x.current=Qe.id,y(Qe.id)}k("Salvo.")}catch(T){k(T instanceof Error?T.message:"Não foi possível salvar agora.")}C(!1)}function z(p,P){let T=document.createElement("textarea");T.value=p,T.style.position="fixed",T.style.opacity="0",document.body.appendChild(T),T.select();try{document.execCommand("copy"),m(P),setTimeout(()=>m(""),2000)}catch{m("")}document.body.removeChild(T)}function W(p){f(p),q(!0)}let _=wc.length+Cc.length+qc.length,Me=u.size,xr=Math.round(Me/_*100),zi=o.trim().toLowerCase();function Pi(p){return t==="todos"||t==="base"&&p==="Base"||t==="inovacao"&&p==="Inovação"}function Jo(p){return!zi||p.toLowerCase().includes(zi)}let _i=wc.filter((p)=>Pi(p.seal)&&Jo(`${p.piece} ${p.day} ${p.format}`)),Ti=Cc.filter((p)=>Pi(p.seal)&&Jo(`${p.title} ${p.date}`)),zc=qc.filter((p)=>Jo(p.title));function jo(p){let P=Ec[p.seed%Ec.length],T=kc[p.seed%kc.length],Qe=Nc[p.seed%Nc.length],Oc=qi[p.media];return v("div",{className:"mt-6",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"10. Referências, dentro deste card"}),i("p",{className:"text-sm text-[var(--muted)] mt-1",children:"Use como base para se inspirar. Não copie falas nem legendas. Olhe o jeito de filmar e o ritmo."}),v("div",{className:"grid sm:grid-cols-2 gap-3 mt-3",children:[v("div",{className:"rounded-2xl border border-[var(--line)] bg-[var(--bg)] p-4",children:[v("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)] flex items-center gap-1",children:[i(hr,{size:14})," Referência de formato que performa"]}),i("p",{className:"font-semibold mt-2 long-text",children:P.title}),i("p",{className:"text-sm leading-relaxed mt-2 long-text",children:P.copy}),v("p",{className:"text-sm leading-relaxed mt-2 long-text",children:["Por que ajuda. ",P.why]})]}),v("div",{className:"rounded-2xl border border-[var(--line)] bg-[var(--bg)] p-4",children:[v("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)] flex items-center gap-1",children:[i(yt,{size:14})," Referência de trend"]}),i("p",{className:"font-semibold mt-2 long-text",children:Oc}),i("p",{className:"text-sm leading-relaxed mt-2 long-text",children:"Copiar o jeito de filmar. Começo com cena do dia, depois peça vestida, com calma."}),i("p",{className:"text-sm leading-relaxed mt-2 long-text",children:"Por que ajuda. Mostra um formato que já viajou longe e que dá para adaptar para a SAF."}),v("button",{onClick:()=>W(p.media),className:"mt-3 w-full rounded-full bg-[var(--text)] text-white px-4 py-2.5 text-sm font-semibold flex items-center justify-center gap-2",children:[i(yt,{size:15})," Ver vídeo da referência"]})]}),v("div",{className:"rounded-2xl border border-[var(--line)] bg-[var(--bg)] p-4",children:[v("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)] flex items-center gap-1",children:[i(Nr,{size:14})," Referência de música"]}),i("p",{className:"font-semibold mt-2 long-text",children:T.title}),v("p",{className:"text-sm leading-relaxed mt-2 long-text",children:["Por que combina. ",T.why]}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Buscar na aba Música, depois Tendências, dentro do Instagram, e conferir a setinha de alta."})]}),v("div",{className:"rounded-2xl border border-[var(--line)] bg-[var(--bg)] p-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Referência de história"}),i("p",{className:"font-semibold mt-2 long-text",children:Qe.title}),i("p",{className:"text-sm leading-relaxed mt-2 long-text",children:Qe.copy}),v("p",{className:"text-sm leading-relaxed mt-2 long-text",children:["Por que ajuda. ",Qe.why]})]})]})]})}function en(p){let P=l===p.id,T=u.has(p.id);return v("article",{className:"bg-[var(--surface)] rounded-[24px] shadow-[0_8px_30px_rgba(0,0,0,0.06)] border border-[var(--line)] overflow-hidden",children:[v("div",{className:"flex items-start gap-3 p-4 sm:p-5",children:[i("button",{onClick:()=>M(p.id),"aria-label":T?"Desmarcar como feito":"Marcar como feito",className:"check-box mt-1","aria-pressed":T,children:T?i(gr,{size:15}):null}),i("button",{onClick:()=>s(P?null:p.id),"aria-expanded":P,className:"flex-1 text-left min-w-0",children:p.top}),i("button",{onClick:()=>s(P?null:p.id),"aria-label":P?"Fechar":"Abrir",className:"shrink-0 border border-[var(--line)] w-9 h-9 rounded-full grid place-items-center bg-white",children:i(yr,{size:16,className:`transition ${P?"rotate-180":""}`})})]}),i("div",{className:`card-body ${P?"open":""}`,children:i("div",{children:i("div",{className:"px-4 pb-5 sm:px-5",children:p.children})})})]})}function an(p){return v("div",{className:"border-t border-[var(--line)] pt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"2. Para que serve este post"}),v("ul",{className:"mt-2 space-y-2 leading-relaxed text-[15px]",children:[v("li",{className:"long-text",children:[i("strong",{children:"Dor:"})," ",p.dor]}),v("li",{className:"long-text",children:[i("strong",{children:"Desejo:"})," ",p.desejo]}),v("li",{className:"long-text",children:[i("strong",{children:"Problema:"})," ",p.problema]}),v("li",{className:"long-text",children:[i("strong",{children:"Tópico sensível:"})," ",p.sensivel]})]})]})}function Pc(p,P){return v(en,{id:p.id,top:v(qr,{children:[v("span",{className:"block font-semibold leading-[1.2] long-text",style:{fontSize:"21px"},children:[p.day,". ",p.piece]}),v("span",{className:"mt-2 flex flex-wrap items-center gap-2 text-sm text-[var(--muted)]",children:[v("span",{className:"inline-flex items-center gap-1",children:[i("span",{"aria-hidden":"true",className:"w-2 h-2 rounded-full inline-block",style:{background:"var(--terracotta)"}}),p.time]}),i("span",{children:p.format}),v("span",{className:"inline-flex items-center gap-1",children:[i(Sr,{size:13}),p.minutes]}),i("span",{className:`text-xs font-semibold px-3 py-1 rounded-full ${p.seal==="Base"?"seal-base":"seal-inova"}`,children:p.seal.toUpperCase()}),i("span",{className:"text-xs font-semibold px-3 py-1 rounded-full bg-[var(--accent-soft)]",children:p.audience})]})]}),children:[p.id==="f-27nov"&&v("p",{className:"rounded-2xl border border-[var(--line)] bg-[#fff7f0] p-4 text-sm leading-relaxed mb-4 flex gap-2",children:[i(aa,{size:18,className:"shrink-0 mt-0.5"}),"Antes de publicar, confirmar. Desconto, até quando, cupom e frete. Sem confirmação, sem oferta."]}),i(an,{dor:p.dor,desejo:p.desejo,problema:p.problema,sensivel:p.sensivel}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"3. História em 4 partes"}),i("ol",{className:"mt-2 space-y-2 leading-relaxed text-[15px]",children:p.story.map((T)=>i("li",{className:"long-text",children:T},T))})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"4. Como fazer"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.how})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"5. Texto na tela"}),i("p",{className:"text-[16px] mt-2 long-text",children:p.screenText}),v("button",{onClick:()=>z(p.screenText,`screen-${p.id}`),className:"mt-3 rounded-full border border-[var(--line-strong)] px-4 py-2 text-sm font-semibold flex items-center gap-1",children:[i(ea,{size:14}),b===`screen-${p.id}`?"Copiado!":"Copiar texto da tela"]})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"6. Legenda pronta"}),v("div",{className:"mt-3 rounded-2xl bg-[var(--bg)] border border-[var(--line)] p-4",children:[i("p",{className:"caption-box text-sm leading-relaxed",children:p.caption}),v("button",{onClick:()=>z(p.caption,p.id),className:"mt-3 rounded-full bg-[var(--text)] text-white px-4 py-2.5 text-sm font-semibold flex items-center gap-1",children:[i(ea,{size:14}),b===p.id?"Copiado!":"Copiar legenda"]})]})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"7. Convite final"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.invite})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"8. O que olhar em 48 horas"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.look}),i("p",{className:"text-sm text-[var(--muted)] mt-1",children:"Não julgar por curtida."})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"9. Versão mais simples, se ela estiver cansada"}),i("p",{className:"mt-2 leading-relaxed text-[15px] caption-box",children:p.simple})]}),i("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:i(jo,{seed:P+1,media:p.media})})]},p.id)}function _c(p,P){return v(en,{id:p.id,top:v(qr,{children:[v("span",{className:"block font-semibold leading-[1.2] long-text",style:{fontSize:"21px"},children:[p.date,". ",p.title]}),v("span",{className:"mt-2 flex flex-wrap items-center gap-2 text-sm text-[var(--muted)]",children:[v("span",{className:"inline-flex items-center gap-1",children:[i("span",{"aria-hidden":"true",className:"w-2 h-2 rounded-full inline-block",style:{background:"var(--terracotta)"}}),"Stories do dia"]}),i("span",{className:`text-xs font-semibold px-3 py-1 rounded-full ${p.seal==="Base"?"seal-base":"seal-inova"}`,children:p.seal.toUpperCase()}),i("span",{className:"text-xs font-semibold px-3 py-1 rounded-full bg-[var(--accent-soft)]",children:p.audience})]})]}),children:[i(an,{dor:p.dor,desejo:p.desejo,problema:p.problema,sensivel:p.sensivel}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"3. História em 4 partes"}),i("ol",{className:"mt-2 space-y-2 leading-relaxed text-[15px]",children:p.story.map((T)=>i("li",{className:"long-text",children:T},T))})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"4. Como fazer"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.how})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"5. Texto na tela de cada story"}),i("ol",{className:"mt-2 space-y-2 leading-relaxed text-[15px]",children:p.screens.map((T,Qe)=>v("li",{className:"long-text",children:["Story ",Qe+1,". ",T]},T))})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"6. Frase pronta"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.phrase}),v("button",{onClick:()=>z(p.phrase,p.id),className:"mt-3 rounded-full border border-[var(--line-strong)] px-4 py-2 text-sm font-semibold flex items-center gap-1",children:[i(ea,{size:14}),b===p.id?"Copiado!":"Copiar frase"]})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"7. Convite final"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.invite})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"8. O que olhar em 48 horas"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.look}),i("p",{className:"text-sm text-[var(--muted)] mt-1",children:"Não julgar por curtida."})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"9. Versão mais simples, se ela estiver cansada"}),i("p",{className:"mt-2 leading-relaxed text-[15px] caption-box",children:p.simple})]}),i("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:i(jo,{seed:P+2,media:p.media})})]},p.id)}function Tc(p,P){return v(en,{id:p.id,top:v(qr,{children:[i("span",{className:"block font-semibold leading-[1.2] long-text",style:{fontSize:"21px"},children:p.title}),v("span",{className:"mt-2 flex flex-wrap items-center gap-2 text-sm text-[var(--muted)]",children:[v("span",{className:"inline-flex items-center gap-1",children:[i("span",{"aria-hidden":"true",className:"w-2 h-2 rounded-full inline-block",style:{background:"var(--terracotta)"}}),"Destaque salvo no perfil"]}),i("span",{className:"text-xs font-semibold px-3 py-1 rounded-full seal-base",children:"BASE"})]})]}),children:[i(an,{dor:p.dor,desejo:p.desejo,problema:p.problema,sensivel:p.sensivel}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"3. História em 4 partes"}),i("ol",{className:"mt-2 space-y-2 leading-relaxed text-[15px]",children:p.story.map((T)=>i("li",{className:"long-text",children:T},T))})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"4. Como fazer"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.how})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"5. Texto na tela"}),i("p",{className:"text-[16px] mt-2 long-text",children:p.screen}),v("button",{onClick:()=>z(p.screen,`screen-${p.id}`),className:"mt-3 rounded-full border border-[var(--line-strong)] px-4 py-2 text-sm font-semibold flex items-center gap-1",children:[i(ea,{size:14}),b===`screen-${p.id}`?"Copiado!":"Copiar texto da tela"]})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"6. Frase pronta"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.phrase}),v("button",{onClick:()=>z(p.phrase,p.id),className:"mt-3 rounded-full border border-[var(--line-strong)] px-4 py-2 text-sm font-semibold flex items-center gap-1",children:[i(ea,{size:14}),b===p.id?"Copiado!":"Copiar frase"]})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"7. Convite final"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.invite})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"8. O que olhar em 48 horas"}),i("p",{className:"mt-2 leading-relaxed text-[15px] long-text",children:p.look}),i("p",{className:"text-sm text-[var(--muted)] mt-1",children:"Não julgar por curtida."})]}),v("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:[i("h4",{className:"font-semibold",style:{fontSize:"21px"},children:"9. Versão mais simples, se ela estiver cansada"}),i("p",{className:"mt-2 leading-relaxed text-[15px] caption-box",children:p.simple})]}),i("div",{className:"border-t border-[var(--line)] pt-4 mt-4",children:i(jo,{seed:P+3,media:p.media})})]},p.id)}return v("div",{"data-safe-area-inset-root":!0,style:{boxSizing:"border-box",paddingTop:"var(--safe-area-inset-top)",minHeight:"100vh",background:"var(--bg)",color:"var(--text)"},className:"overflow-x-clip",children:[i("header",{"data-safe-area-sticky-top":!0,style:{top:"var(--safe-area-inset-top)",height:"48px",maxHeight:"48px"},className:"sticky z-30 border-b border-[var(--line)] bg-[var(--bg)]",children:v("div",{className:"mx-auto max-w-3xl px-3 sm:px-6 h-full flex items-center justify-between gap-2",children:[i("p",{className:"font-semibold text-[15px] leading-none whitespace-nowrap",children:"SAF Modas"}),i("nav",{"aria-label":"Seções do portal",className:"flex items-center gap-1",children:[["inicio","Início"],["feed","Feed"],["stories","Stories"],["medidas","Medidas"]].map(([p,P])=>i("button",{onClick:()=>{a(p),r("todos")},"aria-pressed":e===p,className:`rounded-full px-2.5 py-1.5 text-[12px] font-semibold leading-none whitespace-nowrap ${e===p?"bg-[var(--text)] text-white":"border border-[var(--line-strong)] bg-white"}`,children:P},p))})]})}),v("div",{className:"mx-auto max-w-3xl px-4 sm:px-6 pt-4",children:[v("div",{className:"flex justify-between text-xs text-[var(--muted)] mb-1",children:[i("span",{children:"Quantos posts feitos"}),v("span",{children:[Me," de ",_," . ",xr,"%"]})]}),i("div",{className:"h-2 rounded-full bg-[#e8e6e1] overflow-hidden",children:i("div",{className:"h-full rounded-full transition-all",style:{width:`${xr}%`,background:"var(--terracotta)"}})}),i("p",{className:"text-xs text-[var(--muted)] mt-1",children:N?"Salvando…":g}),v("label",{className:"flex items-center gap-2 bg-white border border-[var(--line)] rounded-full px-4 py-2.5 mt-3",children:[i(wr,{size:16,className:"text-[var(--muted)] shrink-0"}),i("input",{value:o,onChange:(p)=>n(p.target.value),placeholder:"Pesquisa por peça. Ex. Saia Milena","aria-label":"Pesquisa por peça",className:"flex-1 min-w-0 outline-none text-sm bg-transparent placeholder:text-[var(--soft)]"})]}),i("div",{className:"flex flex-wrap gap-2 mt-3",children:[["todos","Todos"],["base","Base"],["inovacao","Inovação"]].map(([p,P])=>i("button",{onClick:()=>r(p),"aria-pressed":t===p,className:`rounded-full px-3 py-2 text-xs font-semibold border ${t===p?"bg-[var(--text)] text-white border-[var(--text)]":"bg-white border-[var(--line)]"}`,children:P},p))})]}),v("main",{className:"mx-auto max-w-3xl px-4 sm:px-6 pb-12 text-[15px]",style:{lineHeight:1.6},children:[v("section",{style:{display:e==="inicio"?"block":"none"},"aria-label":"Início",children:[v("div",{className:"pt-8 pb-6",children:[i("p",{className:"text-xs tracking-[0.18em] uppercase text-[var(--muted)]",children:"Tudo que você precisa antes de gravar"}),i("h2",{className:"font-semibold leading-[1.2] mt-3",style:{fontSize:"22px"},children:"Comece por aqui. Sem pressa, vamos juntas."}),i("p",{className:"mt-3 max-w-2xl text-[var(--muted)] leading-relaxed long-text",children:"Este portal é o seu roteiro. Leia esta página, depois abra o Feed ou os Stories. Nada aqui exige que você fale alto, apareça o tempo todo ou entenda palavra difícil de vídeo."}),v("div",{className:"flex flex-wrap gap-2 mt-4",children:[i("button",{onClick:()=>a("feed"),className:"rounded-full bg-[var(--text)] text-white px-4 py-2.5 text-sm font-semibold",children:"Ir para o Feed"}),i("button",{onClick:()=>a("stories"),className:"rounded-full border border-[var(--line-strong)] bg-white px-4 py-2.5 text-sm",children:"Ir para os Stories"}),i("button",{onClick:()=>a("medidas"),className:"rounded-full border border-[var(--line)] bg-white px-4 py-2.5 text-sm",children:"Ir para Medidas"})]}),v("p",{className:"mt-3 text-sm text-[var(--muted)]",children:[i("strong",{className:"text-[var(--text)]",children:"Primeira frase."})," Neste portal, primeira frase sempre quer dizer a primeira frase que aparece escrita na tela."]})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6 mb-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Estratégia revisitada"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"A mulher é o centro. A peça é só o meio."}),i("p",{className:"mt-3 leading-relaxed",children:"A SAF hoje mostra muito a peça no cabide, na superfície e em selfie no espelho. Isso mostra a peça, mas não mostra a mulher vestindo em movimento, não mostra o rosto, não mostra a história. A cliente fica sem saber como fica no corpo dela. Por isso o portal muda o centro: a peça é só o meio, a mulher é o centro."}),i("p",{className:"mt-3 leading-relaxed",children:"Cada post precisa tocar uma dor, acender um desejo, resolver um problema e tocar um tópico sensível com carinho. Sem conteúdo fraco. Sem frase que diminui o corpo. Sem frase que coloca culpa. Sem escassez falsa."})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6 mb-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"As 5 dores"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"O que todo post precisa tocar"}),i("ol",{className:"mt-4 space-y-3 leading-relaxed",children:["1. Abro o guarda roupa e sinto que não tenho nada, mesmo com roupa.","2. Compro online e não serve, e fico com medo de comprar de novo.","3. Vejo uma peça linda e acho que não fica bem no meu corpo.","4. Não tenho tempo para pensar em look, quero algo pronto.","5. Quero me sentir bonita sem parecer que estou tentando demais."].map((p)=>i("li",{className:"long-text",children:p},p))})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6 mb-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Os 4 desejos"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"O que todo post precisa acender"}),i("ol",{className:"mt-4 space-y-3 leading-relaxed",children:["1. Quero me olhar e gostar do que vejo.","2. Quero uma roupa que resolva o dia sem me dar trabalho.","3. Quero me sentir acolhida, sem ser julgada pelo corpo.","4. Quero uma roupa que eu use de verdade, não só uma vez."].map((p)=>i("li",{className:"long-text",children:p},p))})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6 mb-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Tópicos sensíveis"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"A gente toca com carinho, sem atacar ninguém"}),i("p",{className:"mt-3 leading-relaxed",children:"Comparação com outras mulheres, medo de não caber, vergonha de pedir ajuda com tamanho, culpa por gastar, pressão de estar sempre arrumada, cansaço de fim de ano."}),i("p",{className:"mt-3 leading-relaxed",children:"Nunca usar frase que diminui o corpo, nunca usar frase que coloca culpa, nunca usar escassez falsa."})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6 mb-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Regra de intenção"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"7 do jeito que ela já faz, 3 com ideia nova"}),i("p",{className:"mt-3 leading-relaxed",children:"De cada 10 posts, 7 são do jeito que ela já faz, com mais história, e 3 trazem uma ideia nova. Em cada semana, ter pelo menos 2 posts para quem ainda não conhece a SAF, que são os posts mais leves, mais compartilháveis, mais de identificação, como POV, arrume se comigo, antes e depois, uma peça 3 jeitos, isso ou aquilo."}),i("p",{className:"mt-3 leading-relaxed",children:"Ter pelo menos 1 post por semana que resolve uma dúvida real, como tamanho, caimento, como usar, como combinar. Ter pelo menos 1 post por semana que mostra gente real, cliente, bastidor, embalando."}),i("p",{className:"mt-3 leading-relaxed",children:"Para falar de alcance, falar assim: post para quem ainda não conhece a SAF, post para quem já conhece e está pensando, post para quem já quer comprar."})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6 mb-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Como usar o portal"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"4 passos curtos"}),i("ol",{className:"mt-4 space-y-0",children:[["1. Abra o dia de hoje.","No Feed ou nos Stories, ache a data e abra o cartão."],["2. Leia a história.","Cada dia tem uma história em 4 partes. Começo, meio, prova e convite."],["3. Faça com calma.","Siga a linha Como fazer. Comece sempre com corpo inteiro vestida."],["4. Marque como feito.","Toque na bolinha. O portal salva para você."]].map(([p,P])=>v("li",{className:"py-3 border-b border-[var(--line)] last:border-0",children:[i("p",{className:"font-semibold",children:p}),i("p",{className:"text-[var(--muted)] leading-relaxed mt-1 long-text",children:P})]},p))})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6 mb-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Guia de ganchos"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"Primeira frase com no máximo 8 palavras"}),i("p",{className:"mt-3 leading-relaxed",children:"A primeira frase sempre vai escrita na tela. Use uma destas 7 ideias. Escolha a que combina com o dia."}),i("div",{className:"mt-4 space-y-3",children:[["Curiosidade","Ela abre o guarda roupa e nada serve."],["Situação que se reconhece","Você também fica 5 minutos sem saber o que vestir?"],["Transformação","Olha essa peça antes e depois de vestir."],["Pergunta","Qual cor você usaria no trabalho?"],["Chamar um grupo","Para quem trabalha o dia todo em pé."],["Lista com número","1 peça, 3 jeitos de sair rápido."],["POV","POV. Você abre o guarda roupa e nada anima."]].map(([p,P])=>v("div",{className:"rounded-2xl border border-[var(--line)] bg-[var(--bg)] p-4",children:[i("p",{className:"font-semibold",children:p}),i("p",{className:"text-[var(--muted)] leading-relaxed mt-1 long-text",children:P})]},p))}),v("div",{className:"mt-4 rounded-2xl border border-[var(--line)] p-4",children:[i("p",{className:"font-semibold",children:"O que não usar"}),i("p",{className:"text-sm leading-relaxed mt-1",children:"Não usar bronca. Não usar julgamento. Não usar frase que desvaloriza corpo. Falar como fala com uma amiga."})]})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6 mb-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Sem falar"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"Como gravar sem falar"}),i("p",{className:"mt-3 leading-relaxed",children:"Você não precisa falar para gravar. A primeira frase vai escrita na tela. O resto é você vestida, andando, sentando e mostrando de perto."}),i("ul",{className:"mt-4 space-y-3 leading-relaxed",children:["Comece com corpo inteiro vestida, andando 3 passos.","Mostre para onde a peça serve. Trabalho, almoço, saída.","Mostre de perto. Tecido, costura, botão, forro.","Feche com corpo inteiro de novo e uma pergunta só."].map((p)=>v("li",{className:"flex gap-3",children:[i("span",{"aria-hidden":"true",children:"•"}),i("span",{className:"long-text",children:p})]},p))}),i("p",{className:"mt-4 leading-relaxed",children:"Se quiser falar, fale como fala com uma amiga. Frase curta, palavra simples, sem pressa."})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6 mb-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Pouca quantidade"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"Como falar de peças com pouca quantidade sem pressão"}),i("p",{className:"mt-3 leading-relaxed",children:"Nunca invente número de unidades. Use só o número confirmado. Nunca use corre, última chance ou vai esgotar."}),v("div",{className:"mt-4 rounded-2xl border border-[var(--line)] bg-[var(--bg)] p-4",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"5 frases prontas"}),i("ol",{className:"mt-3 space-y-3 leading-relaxed",children:["Chegaram [CONFIRMAR] unidades por tamanho. Vou deixar aqui as numerações que temos hoje.","Escolhi poucas dessa vez porque quis testar esse modelo com vocês antes de trazer mais.","Se o seu tamanho não estiver disponível, me chama que eu te aviso quando chegar de novo.","Comenta LISTA que eu anoto seu nome e te aviso primeiro.","Se essa cor não tiver no seu tamanho, olha essa outra opção que veste parecido."].map((p,P)=>v("li",{className:"flex gap-3",children:[v("span",{className:"font-semibold",children:[P+1,"."]}),i("span",{className:"long-text",children:p})]},p))})]})]}),v("div",{className:"bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5 sm:p-6",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Nossa conversa"}),i("h3",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"Tom de conversa de amiga"}),i("p",{className:"mt-3 leading-relaxed",children:"Acolher, não pressionar. A cliente precisa sentir que pode tirar dúvida sem vergonha. Que ali tem solução para o corpo dela. Que não precisa entender de tendência para se vestir bem."}),i("div",{className:"grid sm:grid-cols-3 gap-3 mt-4",children:["Vamos juntas?","Me conta sua altura que eu te ajudo","Sem pressa, você pode olhar com calma"].map((p)=>i("div",{className:"rounded-2xl border border-[var(--line)] bg-[var(--bg)] p-4",children:i("p",{className:"leading-snug",children:p})},p))})]})]}),v("section",{style:{display:e==="feed"?"block":"none"},"aria-label":"Feed",children:[v("div",{className:"pt-8 pb-4",children:[i("p",{className:"text-xs tracking-[0.18em] uppercase text-[var(--muted)]",children:"Feed. 19 posts. 24/11 a 31/12"}),i("h2",{className:"font-semibold leading-[1.2] mt-2",style:{fontSize:"22px"},children:"Feed, um por vez."}),i("p",{className:"mt-2 max-w-2xl text-[var(--muted)] leading-relaxed",children:"Toque em um dia para abrir. Marque a bolinha quando terminar. Cada card tem tudo dentro. Você consegue gravar sem procurar nada em outro lugar."})]}),v("div",{className:"my-2 bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5",children:[v("p",{className:"font-semibold flex gap-2 items-center",children:[i(wa,{size:16})," Semana Black Friday. 23 a 27/11. Foco. Aquecimento, não promoção com pressa"]}),v("p",{className:"mt-2 leading-relaxed",children:["Ainda não temos todas as informações de Black Friday ",i("strong",{children:"[CONFIRMAR. Quais peças entram, desconto, até quando, cupom, frete]"}),". Por isso esta semana é aquecimento. Mostrar as peças que podem entrar, perguntar o que a cliente quer ver, criar lista de espera. Não anunciar desconto que não foi confirmado. Não usar última chance."]})]}),v("div",{className:"space-y-4 mt-4",children:[_i.map((p,P)=>Pc(p,P)),_i.length===0&&i("p",{className:"bg-white rounded-[24px] border border-[var(--line)] p-5 text-center text-[var(--muted)]",children:"Nenhum card achado para esta pesquisa. Tente outra peça."})]})]}),v("section",{style:{display:e==="stories"?"block":"none"},"aria-label":"Stories",children:[v("div",{className:"pt-8 pb-4",children:[i("p",{className:"text-xs tracking-[0.18em] uppercase text-[var(--muted)]",children:"Stories. Todos os dias. 23/11 a 31/12. 39 dias"}),i("h2",{className:"font-semibold leading-[1.2] mt-2",style:{fontSize:"22px"},children:"Stories leves, sem repetir o Feed."}),i("p",{className:"mt-2 max-w-2xl text-[var(--muted)] leading-relaxed",children:"Toque em um dia para abrir o roteiro. Marque quando terminar. Stories não precisam ser perfeitos. Precisam ser calmos."})]}),v("div",{className:"space-y-4",children:[Ti.map((p,P)=>_c(p,P)),Ti.length===0&&i("p",{className:"bg-white rounded-[24px] border border-[var(--line)] p-5 text-center text-[var(--muted)]",children:"Nenhum card achado para esta pesquisa."})]}),v("div",{className:"mt-4 bg-white rounded-[24px] border border-[var(--line)] p-5 leading-relaxed",children:[v("p",{className:"font-semibold flex gap-2 items-center",children:[i(kr,{size:16})," Ritmo da semana"]}),i("p",{className:"mt-2 text-sm text-[var(--muted)]",children:"Segunda novidades. Terça bastidor sem aparecer. Quarta apoio ao Feed. Quinta bastidor ou dica. Sexta prévia ou comunidade. Sábado bastidor leve. Domingo pausa leve ou 1 story. 27/11 Black Friday em 3 blocos. Dezembro série de Natal."})]})]}),v("section",{style:{display:e==="medidas"?"block":"none"},"aria-label":"Destaque Medidas",children:[v("div",{className:"pt-8 pb-4",children:[i("p",{className:"text-xs tracking-[0.18em] uppercase text-[var(--muted)]",children:"Destaque Medidas. Separado da produção diária"}),i("h2",{className:"font-semibold leading-[1.2] mt-2",style:{fontSize:"22px"},children:"Destaque Medidas."}),i("p",{className:"mt-2 max-w-2xl text-[var(--muted)] leading-relaxed",children:"Este destaque fica salvo no perfil, fora da produção diária. A cliente abre quando tem dúvida de tamanho. Grave com calma, uma vez só, e deixe salvo."}),i("p",{className:"mt-3 font-semibold",children:"Convite do destaque. Salva esse destaque antes de comprar."})]}),i("div",{className:"space-y-4",children:zc.map((p,P)=>Tc(p,P))}),v("div",{className:"mt-6 bg-white rounded-[24px] border border-[var(--line)] shadow-[0_8px_30px_rgba(0,0,0,0.06)] p-5",children:[i("h3",{className:"font-semibold",style:{fontSize:"22px"},children:"Tabela de exemplo"}),i("p",{className:"text-sm text-[var(--muted)] mt-2 leading-relaxed",children:"Preencha depois. Nunca invente números. Deixe [CONFIRMAR] onde faltar dado."}),i("div",{className:"mt-4 border border-[var(--line)] rounded-2xl overflow-x-auto",children:v("table",{className:"w-full min-w-[640px] text-left text-sm",children:[i("thead",{children:v("tr",{className:"border-b border-[var(--line)]",children:[i("th",{className:"p-3 font-semibold",children:"Peça"}),i("th",{className:"p-3 font-semibold",children:"Busto"}),i("th",{className:"p-3 font-semibold",children:"Cintura"}),i("th",{className:"p-3 font-semibold",children:"Quadril"}),i("th",{className:"p-3 font-semibold",children:"Comprimento"}),i("th",{className:"p-3 font-semibold",children:"Tamanho usado"}),i("th",{className:"p-3 font-semibold",children:"Altura da modelo"})]})}),v("tbody",{children:[v("tr",{className:"border-b border-[var(--line)]",children:[i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"})]}),v("tr",{className:"border-b border-[var(--line)]",children:[i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"})]}),v("tr",{children:[i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"}),i("td",{className:"p-3",children:"[CONFIRMAR]"})]})]})]})})]})]}),v("section",{className:"mt-10 bg-white rounded-[24px] border border-[var(--line-strong)] p-5",children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Depois de publicar"}),i("h2",{className:"font-semibold mt-2",style:{fontSize:"22px"},children:"Como saber se funcionou"}),i("div",{className:"mt-4 border border-[var(--line)] rounded-2xl overflow-x-auto",children:v("table",{className:"w-full min-w-[520px] text-left text-sm",children:[i("thead",{children:v("tr",{className:"border-b border-[var(--line)]",children:[i("th",{className:"p-3 font-semibold",children:"Olhar para"}),i("th",{className:"p-3 font-semibold",children:"O que quer dizer"})]})}),v("tbody",{children:[v("tr",{className:"border-b border-[var(--line)]",children:[i("td",{className:"p-3 font-semibold",children:"Salvamentos"}),i("td",{className:"p-3",children:"Ela quer guardar para depois."})]}),v("tr",{className:"border-b border-[var(--line)]",children:[i("td",{className:"p-3 font-semibold",children:"Compartilhamentos"}),i("td",{className:"p-3",children:"Ela quer mandar para uma amiga."})]}),v("tr",{className:"border-b border-[var(--line)]",children:[i("td",{className:"p-3 font-semibold",children:"Comentários com dúvida"}),i("td",{className:"p-3",children:"Ela quer comprar e precisa de ajuda."})]}),v("tr",{children:[i("td",{className:"p-3 font-semibold",children:"Comentários QUERO e cliques"}),i("td",{className:"p-3",children:"Ela quer saber como comprar."})]})]})]})}),i("p",{className:"mt-4 leading-relaxed",children:"Avaliar cada post em 48 horas. Semana no domingo. Não julgar por curtida."}),v("div",{className:"mt-4 rounded-2xl border border-[var(--line-strong)] p-4 flex gap-3 items-start",children:[i(aa,{size:18,className:"shrink-0 mt-0.5"}),v("p",{className:"leading-relaxed",children:[i("strong",{children:"Antes de publicar, validar estoque, tamanhos, preços, peças da Black Friday e caminho de compra."}),i("br",{}),i("span",{className:"text-[var(--muted)] text-sm",children:"Nada aqui substitui a confirmação antes de postar. Se não foi confirmado, não publique como confirmado."})]})]}),v("p",{className:"mt-4 flex gap-2 items-center text-sm",children:[i(Er,{size:16})," Obrigada por estar aqui. Vamos juntas."]})]})]}),i("footer",{className:"border-t border-[var(--line)] py-5 px-4 sm:px-6",children:i("p",{className:"mx-auto max-w-3xl text-xs text-[var(--muted)]",children:"SAF Modas. Portal de conteúdo. Feito para gravar em casa, com calma."})}),i("div",{style:{display:E?"flex":"none"},className:"fixed inset-0 z-50 items-end sm:items-center justify-center bg-black/50 p-3",role:"dialog","aria-modal":"true","aria-label":"Vídeo da referência",children:v("div",{className:"bg-white w-full max-w-md rounded-[24px] p-4 sm:p-5 max-h-[92vh] overflow-y-auto",children:[v("div",{className:"flex items-start justify-between gap-3",children:[v("div",{children:[i("p",{className:"text-xs tracking-widest uppercase text-[var(--muted)]",children:"Referência de direção de câmera"}),i("p",{className:"font-semibold mt-1 long-text",style:{fontSize:"21px"},children:qi[d]}),i("p",{className:"text-sm text-[var(--muted)]",children:"Olhe o jeito de filmar. Não copie falas nem legendas."})]}),i("button",{onClick:()=>q(!1),"aria-label":"Fechar vídeo",className:"w-9 h-9 rounded-full border border-[var(--line)] grid place-items-center shrink-0",children:i(Cr,{size:16})})]}),i("div",{className:"flex flex-wrap gap-2 mt-4",children:[1,2,3,4,5,6,7,8,9,10].map((p)=>i("button",{onClick:()=>f(p),className:`rounded-full px-3 py-2 text-xs font-semibold border ${d===p?"bg-[var(--text)] text-white border-[var(--text)]":"bg-white border-[var(--line)]"}`,children:qi[p]},p))}),v("div",{className:"mt-4",children:[v("div",{style:{display:d===1?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-1","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]}),v("div",{style:{display:d===2?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-2","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]}),v("div",{style:{display:d===3?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-3","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]}),v("div",{style:{display:d===4?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-4","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]}),v("div",{style:{display:d===5?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-5","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]}),v("div",{style:{display:d===6?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-6","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]}),v("div",{style:{display:d===7?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-7","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]}),v("div",{style:{display:d===8?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-8","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]}),v("div",{style:{display:d===9?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-9","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]}),v("div",{style:{display:d===10?"block":"none"},children:[i("div",{className:"media-frame",children:i("div",{className:"block w-full aspect-[9/16]","data-clippy-inline-media-slot":"media-10","aria-label":"Managed media",children:"Loading media"})}),i("p",{className:"text-xs text-[var(--muted)] mt-2",children:"Instagram"})]})]}),i("button",{onClick:()=>q(!1),className:"mt-4 w-full rounded-full bg-[var(--text)] text-white px-4 py-2.5 text-sm font-semibold",children:"Voltar para o card"})]})})]})}Mc.createRoot(document.getElementById("root")).render(i(xc.default.StrictMode,{children:i(Mi,{})}));
</script>
  <script>(function(){function m(a){var h=a.getAttribute("href");if(!h)return;try{var u=new URL(h,document.baseURI);if(((u.protocol==="http:"||u.protocol==="https:")&&u.host!==location.host)||u.protocol==="mailto:"||u.protocol==="tel:"||u.protocol==="sms:"){a.target="_blank";a.rel="noopener noreferrer";}}catch(e){}}function s(){document.querySelectorAll("a[href]").forEach(m);}if(document.readyState!=="loading"){s();}else{document.addEventListener("DOMContentLoaded",s);}document.addEventListener("click",function(e){var a=e.target&&e.target.closest&&e.target.closest("a[href]");if(a){m(a);}},true);})();</script>
  <script type="application/json" data-clippy-inline-media-manifest="1">{"version":1,"slots":[{"slot_id":"media-1","media_kind":"video","media_id":"18107240066463083","data_url":"https://www.instagram.com/reel/Dd2W_a-uO3F/"},{"slot_id":"media-2","media_kind":"video","media_id":"18384949840233334","data_url":"https://www.instagram.com/reel/DfF6JC5qjLU/"},{"slot_id":"media-3","media_kind":"video","media_id":"18407038441084378","data_url":"https://www.instagram.com/reel/DcW9imXvayy/"},{"slot_id":"media-4","media_kind":"video","media_id":"17975813016141000","data_url":"https://www.instagram.com/reel/DeaK1_wgeVi/"},{"slot_id":"media-5","media_kind":"video","media_id":"17887432884382241","data_url":"https://www.instagram.com/reel/DQMVPKwiqeU/"},{"slot_id":"media-6","media_kind":"video","media_id":"17904669252247858","data_url":"https://www.instagram.com/reel/DNv0UXUYitg/"},{"slot_id":"media-7","media_kind":"video","media_id":"18264428785303006","data_url":"https://www.instagram.com/reel/DN5_gGnjldI/"},{"slot_id":"media-8","media_kind":"video","media_id":"18138597844579223","data_url":"https://www.instagram.com/reel/DcJbxR8R9zr/"},{"slot_id":"media-9","media_kind":"video","media_id":"18557835742077608","data_url":"https://www.instagram.com/reel/DcBK2qOKfMp/"},{"slot_id":"media-10","media_kind":"video","media_id":"17946427656336195","data_url":"https://www.instagram.com/reel/DduNX_VSWhg/"}]}</script>
<script data-clippy-inline-media-embed="clippy:inline-media-embed">(function installManagedInlineMediaRuntime(config, environment = {}) {
  const runtimeWindow = environment.window || window, runtimeDocument = environment.document || document, RuntimeMutationObserver = environment.MutationObserver || MutationObserver, runtimeSetTimeout = environment.setTimeout || setTimeout, runtimeClearTimeout = environment.clearTimeout || clearTimeout, slotAttribute = config.slotAttribute, postIdAttribute = config.postIdAttribute, kindAttribute = config.kindAttribute, requestType = config.requestType, responseType = config.responseType, resolverTimeoutMs = config.resolverTimeoutMs || 12000, documentToken = config.requireDocumentToken === !0 ? Array.from(runtimeWindow.crypto.getRandomValues(new Uint8Array(16)), (byte) => byte.toString(16).padStart(2, "0")).join("") : null, slotConfigById = Object.create(null), responseById = Object.create(null), hasOwn = (object, key) => Object.prototype.hasOwnProperty.call(object, key);
  for (const slot of config.slots || [])
    if (slot && slot.slot_id)
      slotConfigById[slot.slot_id] = slot;
  const slotIds = Object.keys(slotConfigById);
  let requestSent = !1, resolverTimeoutId = null;
  function slotElements() {
    return Array.from(runtimeDocument.querySelectorAll(`[${slotAttribute}]`));
  }
  function markError(element, message) {
    if (!element || element.getAttribute("data-clippy-inline-media-embedded") || element.getAttribute("data-clippy-inline-media-error"))
      return;
    const originalLabel = element.textContent && element.textContent.trim();
    if (!element.getAttribute("aria-label") && originalLabel)
      element.setAttribute("aria-label", originalLabel);
    element.textContent = message;
    element.setAttribute("data-clippy-inline-media-error", "1");
  }
  function mediaLabel(element) {
    return element.getAttribute("aria-label") || !element.getAttribute("data-clippy-inline-media-error") && element.textContent || "First-party media";
  }
  function styleWrapper(wrapper, element) {
    wrapper.className = element.className;
    wrapper.style.cssText = element.style.cssText;
    wrapper.setAttribute("data-clippy-inline-media-embedded", "1");
    const label = mediaLabel(element);
    if (label)
      wrapper.setAttribute("aria-label", label);
    if (!wrapper.style.backgroundColor)
      wrapper.style.backgroundColor = "#000";
  }
  function styleMedia(media, element) {
    media.title = mediaLabel(element);
    media.style.width = "100%";
    media.style.height = "100%";
    media.style.border = "0";
    media.style.display = "block";
    media.style.objectFit = "contain";
    media.style.backgroundColor = "#000";
  }
  function render(element, response) {
    if (element.getAttribute("data-clippy-inline-media-embedded"))
      return;
    if (response.error) {
      markError(element, response.error);
      return;
    }
    const slotId = response.slotId, source = response.videoUrl || response.imageUrl;
    if (!source) {
      markError(element, response.blank === !0 ? "" : `Managed first-party media resolver returned no playable media for ${slotId}`);
      return;
    }
    const wrapper = runtimeDocument.createElement("div"), media = runtimeDocument.createElement(response.videoUrl ? "video" : "img");
    styleWrapper(wrapper, element);
    styleMedia(media, element);
    if (media.tagName === "VIDEO") {
      media.src = source;
      media.controls = !0;
      media.playsInline = !0;
      media.preload = "metadata";
      if (response.posterUrl)
        media.poster = response.posterUrl;
    } else {
      media.src = source;
      media.alt = mediaLabel(element);
      media.loading = "lazy";
    }
    wrapper.appendChild(media);
    element.setAttribute("data-clippy-inline-media-embedded", "1");
    element.replaceWith(wrapper);
  }
  function prepareAndRender(element) {
    const slotId = element.getAttribute(slotAttribute), slotConfig = slotId && slotConfigById[slotId];
    if (!slotId || !slotConfig) {
      markError(element, `Managed first-party media slot missing resolved metadata: ${slotId || "unknown"}`);
      return;
    }
    if (!slotConfig.data_url || !slotConfig.media_id || !slotConfig.media_kind) {
      markError(element, `Managed first-party media slot missing resolved metadata: ${slotId}`);
      return;
    }
    element.setAttribute(postIdAttribute, slotConfig.media_id);
    element.setAttribute(kindAttribute, slotConfig.media_kind);
    if (element.tagName === "A")
      element.setAttribute("href", slotConfig.data_url);
    if (hasOwn(responseById, slotId))
      render(element, responseById[slotId]);
  }
  function hydrateMountedSlots() {
    for (const element of slotElements())
      prepareAndRender(element);
  }
  function hydrateAddedSlots(mutations) {
    for (const mutation of mutations)
      for (const node of Array.from(mutation.addedNodes || [])) {
        if (typeof node.getAttribute === "function" && node.getAttribute(slotAttribute))
          prepareAndRender(node);
        if (typeof node.querySelectorAll === "function")
          for (const element of node.querySelectorAll(`[${slotAttribute}]`))
            prepareAndRender(element);
      }
  }
  function cacheResolverErrorForUnresolvedSlots(messageForSlot) {
    for (const slotId of slotIds)
      if (!hasOwn(responseById, slotId)) {
        const error = messageForSlot(slotId);
        responseById[slotId] = {
          slotId,
          error,
          blank: error === ""
        };
      }
    hydrateMountedSlots();
  }
  function clearResolverTimeoutIfComplete() {
    if (resolverTimeoutId !== null && slotIds.every((slotId) => hasOwn(responseById, slotId))) {
      runtimeClearTimeout(resolverTimeoutId);
      resolverTimeoutId = null;
    }
  }
  function requestResolvedMedia() {
    if (requestSent)
      return;
    requestSent = !0;
    const slots = [];
    for (const slotId of slotIds) {
      const slotConfig = slotConfigById[slotId];
      if (!slotConfig.data_url || !slotConfig.media_id || !slotConfig.media_kind) {
        responseById[slotId] = {
          slotId,
          error: `Managed first-party media slot missing resolved metadata: ${slotId}`
        };
        continue;
      }
      slots.push({
        slotId,
        mediaKind: slotConfig.media_kind,
        dataUrl: slotConfig.data_url,
        postId: slotConfig.media_id,
        mediaId: slotConfig.media_id
      });
    }
    if (!slots.length)
      return;
    if (runtimeWindow.parent === runtimeWindow) {
      cacheResolverErrorForUnresolvedSlots(() => "");
      return;
    }
    runtimeWindow.parent.postMessage({
      type: requestType,
      slots,
      ...documentToken === null ? {} : { documentToken }
    }, "*");
    resolverTimeoutId = runtimeSetTimeout(() => {
      resolverTimeoutId = null;
      cacheResolverErrorForUnresolvedSlots((slotId) => `Managed first-party media resolver did not respond for ${slotId}`);
    }, resolverTimeoutMs);
  }
  runtimeWindow.addEventListener("message", (event) => {
    const data = event.data;
    if (event.source !== runtimeWindow.parent || !data || data.type !== responseType || documentToken !== null && data.documentToken !== documentToken || !Array.isArray(data.slots))
      return;
    for (const response of data.slots)
      if (response && response.slotId && slotConfigById[response.slotId])
        responseById[response.slotId] = response;
    clearResolverTimeoutIfComplete();
    hydrateMountedSlots();
  });
  function boot() {
    new RuntimeMutationObserver(hydrateAddedSlots).observe(runtimeDocument.documentElement, { childList: !0, subtree: !0 });
    hydrateMountedSlots();
    requestResolvedMedia();
  }
  if (runtimeDocument.readyState !== "loading")
    boot();
  else
    runtimeDocument.addEventListener("DOMContentLoaded", boot);
})({"slots":[{"slot_id":"media-1","media_kind":"video","media_id":"18107240066463083","data_url":"https://www.instagram.com/reel/Dd2W_a-uO3F/","fallback_required":false},{"slot_id":"media-2","media_kind":"video","media_id":"18384949840233334","data_url":"https://www.instagram.com/reel/DfF6JC5qjLU/","fallback_required":false},{"slot_id":"media-3","media_kind":"video","media_id":"18407038441084378","data_url":"https://www.instagram.com/reel/DcW9imXvayy/","fallback_required":false},{"slot_id":"media-4","media_kind":"video","media_id":"17975813016141000","data_url":"https://www.instagram.com/reel/DeaK1_wgeVi/","fallback_required":false},{"slot_id":"media-5","media_kind":"video","media_id":"17887432884382241","data_url":"https://www.instagram.com/reel/DQMVPKwiqeU/","fallback_required":false},{"slot_id":"media-6","media_kind":"video","media_id":"17904669252247858","data_url":"https://www.instagram.com/reel/DNv0UXUYitg/","fallback_required":false},{"slot_id":"media-7","media_kind":"video","media_id":"18264428785303006","data_url":"https://www.instagram.com/reel/DN5_gGnjldI/","fallback_required":false},{"slot_id":"media-8","media_kind":"video","media_id":"18138597844579223","data_url":"https://www.instagram.com/reel/DcJbxR8R9zr/","fallback_required":false},{"slot_id":"media-9","media_kind":"video","media_id":"18557835742077608","data_url":"https://www.instagram.com/reel/DcBK2qOKfMp/","fallback_required":false},{"slot_id":"media-10","media_kind":"video","media_id":"17946427656336195","data_url":"https://www.instagram.com/reel/DduNX_VSWhg/","fallback_required":false}],"slotAttribute":"data-clippy-inline-media-slot","postIdAttribute":"data-clippy-inline-media-post-id","kindAttribute":"data-clippy-inline-media-kind","requestType":"clippy:inline-media-resolve","responseType":"clippy:inline-media-resolved","resolverTimeoutMs":12000});</script>
<script>(function(){var loc=location.href.replace(/#.*$/,"");var ATTR_NAMES=["data-product-id","data-productid","data-product_id","product-id","productid","product_id","data-source-entity-id","source-entity-id","source_entity_id","data-product","data-metadata","data-meta"];var DATASET_KEYS=["productId","productid","product_id","sourceEntityId","sourceentityid","source_entity_id","product","metadata","meta"];function readProductId(value){if(typeof value!=="string"||value.length===0)return null;if(/^[0-9]{6,}$/.test(value))return value;var match=value.match(/(?:product(?:_|-)?id|source(?:_|-)?entity(?:_|-)?id)["'=:\s]+([0-9]{6,})/i);return match?match[1]:null}function extractProductId(start){for(var node=start;node&&node!==document.body;node=node.parentElement){for(var i=0;i<ATTR_NAMES.length;i++){var attrValue=node.getAttribute&&node.getAttribute(ATTR_NAMES[i]);var attrProductId=readProductId(attrValue);if(attrProductId)return attrProductId}var dataset=node.dataset||null;if(dataset){for(var j=0;j<DATASET_KEYS.length;j++){var dataValue=dataset[DATASET_KEYS[j]];var dataProductId=readProductId(dataValue);if(dataProductId)return dataProductId}}}return null}function isInlineMediaSlotElement(node){return !!(node&&node.getAttribute&&node.getAttribute("data-clippy-inline-media-slot")!==null)}function findInlineMediaSlot(start){for(var node=start;node&&node!==document.body;node=node.parentElement){if(isInlineMediaSlotElement(node))return node}return null}function readInlineMediaUrl(node){if(!node)return null;return node.getAttribute&&((node.getAttribute("data-clippy-inline-media-url")||node.getAttribute("data-url")||node.getAttribute("data_url")))||node.href||null}function stripHash(url){return String(url).replace(/#.*$/,"")}function urlsMatch(a,b){if(!a||!b)return false;try{return stripHash(new URL(a,loc).href)===stripHash(new URL(b,loc).href)}catch(_){return stripHash(a)===stripHash(b)}}function isFirstPartyReelUrl(value){try{var url=new URL(value,loc);if(url.protocol!=="https:")return false;var host=url.hostname.toLowerCase();var supported=host==="instagram.com"||host.endsWith(".instagram.com")||host==="facebook.com"||host.endsWith(".facebook.com");return supported&&/\/reels?\//i.test(url.pathname)}catch(_){return false}}function isInlineMediaUrlClick(node,href){var slot=findInlineMediaSlot(node);if(!slot)return false;var slotUrl=readInlineMediaUrl(slot);if(slotUrl)return urlsMatch(href,slotUrl);return isFirstPartyReelUrl(href)}function findDataHref(start){for(var node=start;node&&node!==document.body;node=node.parentElement){if(node.getAttribute){var href=node.getAttribute("data-href")||node.getAttribute("data-url");if(href)return{href:href,node:node}}}return null}var nativeOpen=window.open;window.open=function(url){if(parent!==window&&typeof url==="string"&&/^https?:\/\//.test(url)){parent.postMessage({type:"ecto:usercontent-link-click",href:url},"*");return null}return nativeOpen?nativeOpen.apply(window,arguments):null};document.addEventListener("click",function(e){var target=e.target instanceof Element?e.target:null;if(!target)return;if(parent===window)return;var a=target.closest?target.closest("a[href]"):null;if(a&&a.href&&/^https?:\/\//.test(a.href)&&a.href.replace(/#.*$/,"")!==loc){if(isInlineMediaUrlClick(a,a.href))return;var productId=extractProductId(target)||extractProductId(a);if(productId){e.preventDefault();parent.postMessage({type:"ecto-artifact-link-click",productId:productId},"*");return}e.preventDefault();parent.postMessage({type:"ecto:usercontent-link-click",href:a.href},"*");return}var dataHref=findDataHref(target);if(dataHref&&/^https?:\/\//.test(dataHref.href)&&dataHref.href.replace(/#.*$/,"")!==loc){if(isInlineMediaUrlClick(dataHref.node,dataHref.href))return;e.preventDefault();parent.postMessage({type:"ecto:usercontent-link-click",href:dataHref.href},"*")}},true)})();</script><script>(function(){var FOCUS_TYPE="ecto:artifact-focus-request";var CLOSE_TYPE="ecto:artifact-close-request";function focusArtifactDocument(){var body=document.body;if(!body)return;try{window.focus();}catch(e){}if(!body.hasAttribute("tabindex"))body.setAttribute("tabindex","-1");try{body.focus({preventScroll:true});}catch(e){try{body.focus();}catch(e2){}}}window.addEventListener("message",function(event){if(event.source!==window.parent)return;var data=event.data;if(!data||typeof data!=="object"||data.type!==FOCUS_TYPE)return;if(document.readyState==="loading"){document.addEventListener("DOMContentLoaded",focusArtifactDocument,{once:true});return;}focusArtifactDocument();});window.addEventListener("keydown",function(event){if(event.key!=="Escape")return;window.setTimeout(function(){if(event.defaultPrevented)return;window.parent.postMessage({type:CLOSE_TYPE},"*");},0);});})();</script></body>
</html>
# PRIMEIRO-CALEND-RIO-SAF-MODAS
