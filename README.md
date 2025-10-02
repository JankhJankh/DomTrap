# DomTrap
Testing a DOMTrap For Debugging XSS


code:

```
(function installAllDomInterceptors() {
  const installedInterceptors = [];

  function interceptProperty(proto, propName, label) {
    const origDesc = Object.getOwnPropertyDescriptor(proto, propName);
    if (!origDesc || !origDesc.configurable) {
      console.warn(`[interceptor] Cannot override ${label}: property not configurable`);
      return;
    }

    Object.defineProperty(proto, propName, {
      configurable: true,
      enumerable: origDesc.enumerable,
      get: function () {
        return origDesc.get ? origDesc.get.call(this) : undefined;
      },
      set: function (value) {
        try {
          const snippet = typeof value === 'string' ? value.slice(0, 200) : String(value);
          console.log(`[${label}] SET on`, this, 'value (first 200 chars):', snippet);
          debugger;
        } catch (e) {
          console.error(`[${label}] error in interceptor`, e);
        }
        return origDesc.set ? origDesc.set.call(this, value) : undefined;
      }
    });

    installedInterceptors.push(() => Object.defineProperty(proto, propName, origDesc));
  }

  function interceptFunction(obj, fnName, label) {
    const originalFn = obj[fnName];
    if (typeof originalFn !== 'function') {
      console.warn(`[interceptor] ${label} is not a function`);
      return;
    }

    obj[fnName] = function (...args) {
      try {
        const argSnippet = args.map(a =>
          typeof a === 'string' ? a.slice(0, 200) : JSON.stringify(a)
        ).join(', ');
        console.log(`[${label}] CALL on`, this, 'args (first 200 chars each):', argSnippet);
        debugger;
      } catch (e) {
        console.error(`[${label}] error in interceptor`, e);
      }
      return originalFn.apply(this, args);
    };

    installedInterceptors.push(() => { obj[fnName] = originalFn; });
  }

  // Intercept common DOM XSS sinks
  interceptProperty(Element.prototype, 'innerHTML', 'innerHTML');
  interceptProperty(Element.prototype, 'outerHTML', 'outerHTML');

  // insertAdjacentHTML
  interceptFunction(Element.prototype, 'insertAdjacentHTML', 'insertAdjacentHTML');

  // document.write and writeln
  interceptFunction(document, 'write', 'document.write');
  interceptFunction(document, 'writeln', 'document.writeln');

  // Optional proxy for document.innerHTML (nonstandard)
  const origDocDesc = Object.getOwnPropertyDescriptor(Document.prototype, 'innerHTML');
  try {
    Object.defineProperty(Document.prototype, 'innerHTML', {
      configurable: true,
      enumerable: false,
      get: function () {
        return document.documentElement && document.documentElement.innerHTML;
      },
      set: function (value) {
        console.log('[document.innerHTML proxy] SET, forwarding to document.documentElement.innerHTML');
        debugger;
        if (document.documentElement) {
          document.documentElement.innerHTML = value; // triggers intercepted setter
        }
      }
    });
    installedInterceptors.push(() => {
      if (origDocDesc) {
        Object.defineProperty(Document.prototype, 'innerHTML', origDocDesc);
      } else {
        delete Document.prototype.innerHTML;
      }
    });
  } catch (e) {
    console.warn('[interceptor] Could not define Document.prototype.innerHTML proxy:', e);
  }

  // Global cleanup function
  window.__restoreAllDomInterceptors = function () {
    for (const restoreFn of installedInterceptors) {
      try { restoreFn(); } catch (e) { console.warn('Restore failed:', e); }
    }
    try { delete window.__restoreAllDomInterceptors; } catch (_) {}
    console.log('[interceptor] All DOM sinks restored to original behavior.');
  };

  console.log('[interceptor] All DOM write methods hooked. Call window.__restoreAllDomInterceptors() to remove them.');
})();
```


To remove all hooks and restore behavior:
```
window.__restoreAllDomInterceptors?.();
```
