---
name: vue-component-silently-missing-render-throw
description: "Diagnose a Vue/Nuxt component that is mounted but renders nothing (section vanishes, no console error). Find the render-function throw by walking the live vnode tree and re-invoking the instance's render to get a stack. Works on the cmux browser backend, where addInitScript and console capture are unavailable."
---

# Vue component silently missing: render throw

## Symptom
A sibling in the same parent renders, but the component's DOM slot is just `<!---->`. tab.errors()/tab.console() are empty (cmux does not capture them, and addInitScript is unsupported).

## Procedure
1. Check the parent's childNodes. A `<!---->` where the component should be means the component rendered a comment.
2. Walk the app vnode tree in-page:
```js
const app=document.getElementById('__nuxt').__vue_app__;let inst;
function walk(v){if(!v||inst)return;if(v.component){const c=v.component;if(c.type.__name==='TARGET'){inst=c;return}walk(c.subTree);return}
 if(Array.isArray(v.children))v.children.forEach(walk);if(v.suspense)walk(v.suspense.activeBranch)}
walk(app._container._vnode);
```
3. If `inst.isMounted===true` and `String(inst.subTree.type)` is `Symbol(v-cmt)`, setup succeeded and render threw. Vue's renderComponentRoot caught the error and substituted a comment. Don't chase composable or setup init.
4. Re-run the render to get the stack:
`inst.render.call(inst.proxy,inst.proxy,[],inst.props,inst.setupState,inst.data,inst.ctx)` inside try/catch, then return `e.stack`.
5. Inspect the data feeding the throwing helper through `inst.setupState.<computed>`. Typical cause: the API payload shape is older than the frontend type (backend on a different feature branch, or a stale container).
6. Confirm at the contract level: check the current branch of each repo, and `git grep 'json:"field"' <branch>` in the API repo.

## Judgement
A frontend `?.` guard hides a backend contract gap. Report both and let the user choose between backend fields and graceful frontend degradation. Don't call a null-guard a fix.
