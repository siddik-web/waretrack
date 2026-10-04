# Real-time Construction System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make every purchase in warehouse Build mode and the Town planner take real (simulated) time to finish — with a visible rising scaffold — instead of applying instantly, per `docs/superpowers/specs/2026-10-04-construction-system-design.md`.

**Architecture:** A single persisted queue (`save.constr`) holds pending builds. Progress is driven by a new persisted, pause/speed-aware accumulator (`save.simClock`), not the per-site-resettable `sim.sec` the spec originally suggested (see Task 1 note). A shared `tickConstr()` call in the main render loop advances and completes entries; completion re-applies the exact mutation the old instant-build code used to do synchronously.

**Tech Stack:** Vanilla JS, Three.js r128, single `index.html`, no build step, no test framework (see Task 5 for the verification approach this project actually uses).

**Deviation from spec:** the spec says progress should be driven by `sim.sec`. `sim.sec` resets to 0 on every `loadSite()` call (i.e. every site switch), which would corrupt or reverse progress for anything not on the currently-detailed site. This plan introduces `save.simClock` instead: a persisted minute-accumulator that only increments alongside `sim.sec` (same pause/speed/hidden-tab behavior) but is never reset. Same user-facing behavior the spec asked for, correct under site switching and save/reload.

---

### Task 1: Persisted state for the construction queue

**Files:**
- Modify: `index.html:867-869` (`defaults()`)
- Modify: `index.html:878` (`loadSave()`)
- Modify: `index.html` inside `tickBody()`, the `if(dt>0){` block (currently starts at the line containing `sim.sec+=dt;`)

- [ ] **Step 1: Add `constr` and `simClock` to the save defaults**

In `defaults()`, change:

```js
function defaults(){return {v:2,best:{},credits:0,up:{fl:0,spd:0,chg:0},builds:{},xp:0,ach:{},
  stats:{shifts:0,pallets:0,orders:0,rush:0,fixes:0,builds:0,threeStars:0,bestScore:0},
  daily:{last:null,streak:0,bestStreak:0,best:{},cleared:[]},settings:{sound:true,music:true,vol:0.7},tutorial:false,town:{owned:[],b:{},unit7:false,bank:0,names:0},cos:{owned:['fl-classic','van-classic'],fl:'fl-classic',van:'van-classic'}};}
```

to:

```js
function defaults(){return {v:2,best:{},credits:0,up:{fl:0,spd:0,chg:0},builds:{},xp:0,ach:{},
  stats:{shifts:0,pallets:0,orders:0,rush:0,fixes:0,builds:0,threeStars:0,bestScore:0},
  daily:{last:null,streak:0,bestStreak:0,best:{},cleared:[]},settings:{sound:true,music:true,vol:0.7},tutorial:false,town:{owned:[],b:{},unit7:false,bank:0,names:0},cos:{owned:['fl-classic','van-classic'],fl:'fl-classic',van:'van-classic'},constr:[],simClock:0};}
```

- [ ] **Step 2: Merge `constr`/`simClock` in `loadSave()` so old saves upgrade cleanly**

In `loadSave()`, change the line:

```js
  out.best=out.best||{}; out.builds=out.builds||{}; out.ach=out.ach||{}; out.cos=Object.assign(defaults().cos,s.cos||{}); out.town=Object.assign(defaults().town,s.town||{});
```

to:

```js
  out.best=out.best||{}; out.builds=out.builds||{}; out.ach=out.ach||{}; out.cos=Object.assign(defaults().cos,s.cos||{}); out.town=Object.assign(defaults().town,s.town||{});
  out.constr=Array.isArray(s.constr)?s.constr:[]; out.simClock=typeof s.simClock==='number'?s.simClock:0;
```

- [ ] **Step 3: Advance `save.simClock` alongside the shift clock**

Find this block inside `tickBody()`:

```js
  if(dt>0){
    sim.sec+=dt;
    const surge=game.surgeUntil>sim.sec;
```

Change to:

```js
  if(dt>0){
    sim.sec+=dt;
    save.simClock+=dt;
    tickConstr(dt);
    const surge=game.surgeUntil>sim.sec;
```

(`tickConstr` doesn't exist yet — that's Task 2. This will make the syntax check in Step 4 fail with a clear, expected error; that's fine, Task 2 defines it before this is ever actually run.)

- [ ] **Step 4: Verify syntax**

Run:
```bash
cd /e/Development/Games/waretrack && awk '/^<script>$/{f=1;next}/^<\/script>$/{f=0}f' index.html > /tmp/script.js && node --check /tmp/script.js && echo OK
```
Expected: `OK` (referencing an undefined function is not a syntax error, so this passes even before Task 2 defines `tickConstr`).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(construction): add persisted queue and sim-clock state"
```

---

### Task 2: Core construction engine (pure helpers, queue, tick, completion)

**Files:**
- Modify: `index.html` — insert a new block directly after `siteBuilds()` (the line `function siteBuilds(code){return save.builds[code]||(save.builds[code]={bays:0,row:false,l3:[],fc:[],brk:false,decor:[]});}`)
- Modify: `index.html` inside `loadSite()`, at the end of the function (after the closing of `if(game.building) showGhosts();`)

- [ ] **Step 1: Add the construction engine functions**

Insert this whole block immediately after `siteBuilds()`:

```js
function constrDur(cost){return clamp(cost*0.12,5,25);}
function constrProgress(c){return clamp((save.simClock-c.startMin)/c.dur,0,1);}
let constrSeq=0;
function queueConstr(entry){
  const dur=constrDur(entry.cost);
  const c=Object.assign({id:'C'+(++constrSeq)+'-'+Date.now(),startMin:save.simClock,dur},entry);
  save.constr.push(c); writeSave(); return c;
}
const constrScaffolds=new Map();
function removeScaffold(id){
  const sc=constrScaffolds.get(id); if(sc&&sc.riser.parent)sc.riser.parent.remove(sc.riser);
  constrScaffolds.delete(id);
}
function syncWhScaffolds(){
  constrScaffolds.forEach((sc,id)=>{if(sc.kind==='wh')removeScaffold(id);});
  save.constr.filter(c=>c.kind==='wh'&&c.site===site.k).forEach(c=>{
    const size=c.size||[3,3,3];
    const r=box(size[0]*0.55,1,size[2]*0.55,'#f0a623',c.pos[0],0,c.pos[2],world,false);
    r.scale.y=0.02;
    constrScaffolds.set(c.id,{riser:r,full:size[1]||3,kind:'wh'});
  });
}
function completeConstr(c){
  const i=save.constr.indexOf(c); if(i<0)return; save.constr.splice(i,1);
  removeScaffold(c.id);
  if(c.kind==='wh'){
    const b=siteBuilds(SITES.find(s=>s.k===c.site).code), [k,a]=c.key.split(':');
    if(k==='bay')b.bays=(b.bays|0)+1; else if(k==='row')b.row=true; else if(k==='l3')b.l3=(b.l3||[]).concat(+a);
    else if(k==='fc')b.fc=(b.fc||[]).concat(+a); else if(k==='brk')b.brk=true; else if(k==='decor')b.decor=(b.decor||[]).concat(+a);
    save.stats.builds++;
    if(save.stats.builds>=5)unlock('builder');
    if(Object.values(save.builds).reduce((n,x)=>n+((x.decor||[]).length),0)>=3)unlock('decor');
    if(c.site===site.k&&game.phase!=='shift'){
      const vt=view.targetT.clone(), vz=view.zoomT;
      loadSite(game.siteIdx);
      view.targetT.copy(vt); view.target.copy(vt); view.zoomT=vz;
    }
    if(c.site===site.k)popup('Built!',c.pos[0]+OX,c.pos[1]+2,c.pos[2],'gold');
  }else{
    const t=town.tiles.get(c.key);
    if(t){
      Object.assign(t,{type:c.meta.type,face:c.meta.face,name:c.meta.name,color:c.meta.color,wild:false});
      save.town.b[c.key]={type:c.meta.type,face:c.meta.face,name:c.meta.name,color:c.meta.color};
      townChanged([c.key],false);
    }
    popup('Built!',c.pos[0],2,c.pos[2],'gold');
  }
  toast('info','Built: '+c.label,'',ICON.hammer); Snd.play('build');
  writeSave(); renderPanel();
}
function tickConstr(dt){
  if(!save.constr.length)return;
  for(let i=save.constr.length-1;i>=0;i--){
    const c=save.constr[i], p=constrProgress(c), sc=constrScaffolds.get(c.id);
    if(sc&&sc.riser.parent){sc.riser.scale.y=Math.max(0.02,sc.full*p); sc.riser.position.y=sc.riser.scale.y/2;}
    if(p>=1)completeConstr(c);
  }
}
```

**Why `removeScaffold`/`syncWhScaffolds` exist:** `loadSite()` wipes the entire `world` group on every call (including re-entering the same site), so any warehouse-kind scaffold mesh must be recreated after every `loadSite()`, not created once. `syncWhScaffolds()` is the single place that does that — called once wired up in Step 2 below, and again from `buyBuild()` in Task 3 so a freshly-queued build shows immediately without waiting for the next site load.

- [ ] **Step 2: Call `syncWhScaffolds()` at the end of `loadSite()`**

Find the end of `loadSite()` — the line:

```js
  select(null); defaultView(); resize(); dirty=true;
  if(game.building) showGhosts();
}
```

Change to:

```js
  select(null); defaultView(); resize(); dirty=true;
  if(game.building) showGhosts();
  syncWhScaffolds();
}
```

- [ ] **Step 3: Verify syntax**

```bash
cd /e/Development/Games/waretrack && awk '/^<script>$/{f=1;next}/^<\/script>$/{f=0}f' index.html > /tmp/script.js && node --check /tmp/script.js && echo OK
```
Expected: `OK`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(construction): add tick/complete engine and warehouse scaffold sync"
```

---

### Task 3: Warehouse Build mode — queue instead of instant-apply

**Files:**
- Modify: `index.html:2848-2857` (`buildOptions()`)
- Modify: `index.html:2886-2899` (`buyBuild()`)
- Modify: `index.html:3385-3399` (`renderBuildPanel()`)

- [ ] **Step 1: Filter already-queued options out of `buildOptions()`**

Change:

```js
function buildOptions(){
  const out=[];
  if(L.bays<5) out.push({key:'bay',group:'Docks and racks',icon:ICON.dock,label:'New dock bay',desc:'Unload one more truck at a time. Docks re-space evenly.',cost:150,pos:[6.6,1.2,L.bays*2.5],size:[11.8,2.4,3.6]});
  if(L.rows<4) out.push({key:'row',group:'Docks and racks',icon:ICON.rack,label:'Rack row D',desc:'16 more pallet slots. Replaces the bulk floor area.',cost:160,pos:[-12.95,1.55,4],size:[12.2,3.1,1.3]});
  ROWS.forEach((z,r)=>{if(!L.l3.includes(r))out.push({key:'l3:'+r,group:'Docks and racks',icon:ICON.up,label:'Third level · row '+'ABCD'[r],desc:'8 more slots stacked on top of row '+'ABCD'[r]+'.',cost:70,pos:[-12.95,3.9,z],size:[12.2,1.4,1.3]});});
  HOMES.forEach((h,i)=>{if(!L.fc.includes(i))out.push({key:'fc:'+i,group:'Fleet support',icon:ICON.plug,label:'Fast charger · spot '+(i+1),desc:'Forklifts parked here charge twice as fast.',cost:45,pos:[h.x,0.4,h.z],size:[1.25,0.8,1.3]});});
  if(!L.brk) out.push({key:'brk',group:'Fleet support',icon:ICON.coffee,label:'Crew room',desc:'Rested drivers: all forklifts 8% faster.',cost:130,pos:[5,1.4,-17.6],size:[6,2.8,3.2]});
  DECOR.forEach((d,i)=>{if(!L.decor.includes(i))out.push({key:'decor:'+i,group:'Decor',icon:ICON.leaf,label:d.name,desc:'Makes the floor feel like home. Purely for looks.',cost:25,pos:[d.x,0.9,d.z],size:[0.9,1.8,0.9]});});
  return out;
}
```

to:

```js
function buildOptions(){
  const pending=new Set(save.constr.filter(c=>c.kind==='wh'&&c.site===site.k).map(c=>c.key));
  const out=[];
  if(L.bays<5&&!pending.has('bay')) out.push({key:'bay',group:'Docks and racks',icon:ICON.dock,label:'New dock bay',desc:'Unload one more truck at a time. Docks re-space evenly.',cost:150,pos:[6.6,1.2,L.bays*2.5],size:[11.8,2.4,3.6]});
  if(L.rows<4&&!pending.has('row')) out.push({key:'row',group:'Docks and racks',icon:ICON.rack,label:'Rack row D',desc:'16 more pallet slots. Replaces the bulk floor area.',cost:160,pos:[-12.95,1.55,4],size:[12.2,3.1,1.3]});
  ROWS.forEach((z,r)=>{if(!L.l3.includes(r)&&!pending.has('l3:'+r))out.push({key:'l3:'+r,group:'Docks and racks',icon:ICON.up,label:'Third level · row '+'ABCD'[r],desc:'8 more slots stacked on top of row '+'ABCD'[r]+'.',cost:70,pos:[-12.95,3.9,z],size:[12.2,1.4,1.3]});});
  HOMES.forEach((h,i)=>{if(!L.fc.includes(i)&&!pending.has('fc:'+i))out.push({key:'fc:'+i,group:'Fleet support',icon:ICON.plug,label:'Fast charger · spot '+(i+1),desc:'Forklifts parked here charge twice as fast.',cost:45,pos:[h.x,0.4,h.z],size:[1.25,0.8,1.3]});});
  if(!L.brk&&!pending.has('brk')) out.push({key:'brk',group:'Fleet support',icon:ICON.coffee,label:'Crew room',desc:'Rested drivers: all forklifts 8% faster.',cost:130,pos:[5,1.4,-17.6],size:[6,2.8,3.2]});
  DECOR.forEach((d,i)=>{if(!L.decor.includes(i)&&!pending.has('decor:'+i))out.push({key:'decor:'+i,group:'Decor',icon:ICON.leaf,label:d.name,desc:'Makes the floor feel like home. Purely for looks.',cost:25,pos:[d.x,0.9,d.z],size:[0.9,1.8,0.9]});});
  return out;
}
```

- [ ] **Step 2: Make `buyBuild()` queue instead of apply instantly**

Change:

```js
function buyBuild(key){
  const o=buildOptions().find(x=>x.key===key); if(!o)return;
  if(save.credits<o.cost){toast('info','Not enough credits','You need '+(o.cost-save.credits)+' more. Play shifts or the daily challenge to earn them.',ICON.lock);return;}
  save.credits-=o.cost; const b=siteBuilds(site.code); const [k,a]=key.split(':');
  if(k==='bay')b.bays=(b.bays|0)+1; else if(k==='row')b.row=true; else if(k==='l3')b.l3=(b.l3||[]).concat(+a);
  else if(k==='fc')b.fc=(b.fc||[]).concat(+a); else if(k==='brk')b.brk=true; else if(k==='decor')b.decor=(b.decor||[]).concat(+a);
  save.stats.builds++; writeSave();
  Snd.play('build'); toast('info','Built: '+o.label,'−'+o.cost+' credits',ICON.hammer);
  if(save.stats.builds>=5)unlock('builder');
  if(Object.values(save.builds).reduce((n,x)=>n+((x.decor||[]).length),0)>=3)unlock('decor');
  const vt=view.targetT.clone(), vz=view.zoomT;
  buildSel=null; loadSite(game.siteIdx); view.targetT.copy(vt); view.target.copy(vt); view.zoomT=vz; setSpeed(0);
  popup('Built!',o.pos[0]+OX,o.pos[1]+2,o.pos[2],'gold'); renderPanel();
}
```

to:

```js
function buyBuild(key){
  const o=buildOptions().find(x=>x.key===key); if(!o)return;
  if(save.credits<o.cost){toast('info','Not enough credits','You need '+(o.cost-save.credits)+' more. Play shifts or the daily challenge to earn them.',ICON.lock);return;}
  save.credits-=o.cost;
  queueConstr({kind:'wh',site:site.k,key:o.key,pos:o.pos,size:o.size,cost:o.cost,label:o.label});
  syncWhScaffolds();
  Snd.play('build'); toast('info','Building: '+o.label,'Ready in '+Math.ceil(constrDur(o.cost))+' min · −'+o.cost+' credits',ICON.hammer);
  buildSel=null; renderPanel();
}
```

(`showGhosts()` is not called here deliberately — `buildOptions()` now excludes this key, so the next `renderPanel()`/ghost refresh naturally drops its ghost preview. Ghosts are only rebuilt when entering Build mode or clicking a different option, which is consistent with existing behavior elsewhere in the file.)

- [ ] **Step 3: Show queued builds with live progress in the Build panel**

Change:

```js
function renderBuildPanel(){
  const opts=buildOptions(); let h='<div class="build-head"><b>Build at '+esc(site.name)+'</b><span class="pill gold">'+fmtN(save.credits)+' credits</span></div>';
  if(!opts.length) h+='<div class="empty">Everything at this site is built. Try another warehouse from the menu.</div>';
  let g='';
  opts.forEach(o=>{
    if(o.group!==g){g=o.group;h+='<div class="sub"><span>'+g+'</span></div>';}
    const can=save.credits>=o.cost;
    h+='<div class="row click'+(buildSel===o.key?' hl':'')+'" data-bkey="'+o.key+'"><div class="badge tk">'+o.icon+'</div><div class="t"><b>'+esc(o.label)+'</b><small>'+esc(o.desc)+'</small></div><div class="r"><button class="act'+(buildSel===o.key?' solid':'')+'" data-buy-build="'+o.key+'"'+(can?'':' disabled')+'>'+o.cost+' cr</button></div></div>';
  });
  const b=siteBuilds(site.code), owned=[];
  if(b.bays)owned.push(b.bays+' extra dock'+(b.bays>1?'s':'')); if(b.row)owned.push('rack row D'); if((b.l3||[]).length)owned.push(b.l3.length+' third level'+(b.l3.length>1?'s':''));
  if((b.fc||[]).length)owned.push(b.fc.length+' fast charger'+(b.fc.length>1?'s':'')); if(b.brk)owned.push('crew room'); if((b.decor||[]).length)owned.push(b.decor.length+' decor');
  if(owned.length) h+='<div class="sub"><span>Already built here</span></div><div class="empty">'+esc(owned.join(' · '))+'</div>';
  pBody.innerHTML=h;
}
```

to:

```js
function renderBuildPanel(){
  const opts=buildOptions(), queued=save.constr.filter(c=>c.kind==='wh'&&c.site===site.k);
  let h='<div class="build-head"><b>Build at '+esc(site.name)+'</b><span class="pill gold">'+fmtN(save.credits)+' credits</span></div>';
  if(queued.length){
    h+='<div class="sub"><span>Under construction</span></div>';
    queued.forEach(c=>{
      const left=Math.max(0,Math.ceil(c.dur-(save.simClock-c.startMin)));
      h+='<div class="row"><div class="badge tk">'+ICON.hammer+'</div><div class="t"><b>'+esc(c.label)+'</b><small>'+left+' min left</small></div><div class="r"><span class="pill info">'+Math.round(constrProgress(c)*100)+'%</span></div></div>';
    });
  }
  if(!opts.length&&!queued.length) h+='<div class="empty">Everything at this site is built. Try another warehouse from the menu.</div>';
  let g='';
  opts.forEach(o=>{
    if(o.group!==g){g=o.group;h+='<div class="sub"><span>'+g+'</span></div>';}
    const can=save.credits>=o.cost;
    h+='<div class="row click'+(buildSel===o.key?' hl':'')+'" data-bkey="'+o.key+'"><div class="badge tk">'+o.icon+'</div><div class="t"><b>'+esc(o.label)+'</b><small>'+esc(o.desc)+'</small></div><div class="r"><button class="act'+(buildSel===o.key?' solid':'')+'" data-buy-build="'+o.key+'"'+(can?'':' disabled')+'>'+o.cost+' cr</button></div></div>';
  });
  const b=siteBuilds(site.code), owned=[];
  if(b.bays)owned.push(b.bays+' extra dock'+(b.bays>1?'s':'')); if(b.row)owned.push('rack row D'); if((b.l3||[]).length)owned.push(b.l3.length+' third level'+(b.l3.length>1?'s':''));
  if((b.fc||[]).length)owned.push(b.fc.length+' fast charger'+(b.fc.length>1?'s':'')); if(b.brk)owned.push('crew room'); if((b.decor||[]).length)owned.push(b.decor.length+' decor');
  if(owned.length) h+='<div class="sub"><span>Already built here</span></div><div class="empty">'+esc(owned.join(' · '))+'</div>';
  pBody.innerHTML=h;
}
```

- [ ] **Step 4: Verify syntax**

```bash
cd /e/Development/Games/waretrack && awk '/^<script>$/{f=1;next}/^<\/script>$/{f=0}f' index.html > /tmp/script.js && node --check /tmp/script.js && echo OK
```
Expected: `OK`

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(construction): warehouse Build mode queues timed construction"
```

---

### Task 4: Town planner — queue instead of instant-apply

**Files:**
- Modify: `index.html` — the `applyTool()` function (the `fail`/`cost`/`roadCh` block starting with `function applyTool(t){`)
- Modify: `index.html` — `undoTown()`
- Modify: `index.html:1371-1386` (`renderTile()`)

- [ ] **Step 1: Defer the building-tool branch in `applyTool()`**

Change the final `else` branch (the non-`buy`, non-`bulldoze` tool case):

```js
  }else{
    const B=BTYPES[tool];
    if(t.owner!=='player')return fail(t.owner?'This plot is not for sale':'Buy this plot first');
    if(['hq','private','bigdepot'].includes(t.type))return fail('That is part of your warehouse');
    if(t.type!=='grass')return fail('Clear this plot first');
    let face=null; for(const d of [2,0,1,3]) if(isRoadT(nbT(t,d))){face=d;break;}
    if(!['road','trees','park'].includes(tool)&&face==null)return fail('Needs a road on one side');
    cost=B.cost; if(save.credits<cost)return fail('You need '+(cost-save.credits)+' more credits');
    t.type=tool; t.face=face==null?2:face; t.wild=false; roadCh=tool==='road';
    if(tool==='shop'){const p=PSHOPS[T.names%PSHOPS.length];t.name=p[0]+(T.names>=PSHOPS.length?' '+(Math.floor(T.names/PSHOPS.length)+1):'');t.color=p[1];T.names++;}
    T.b[t.key]={type:t.type,face:t.face,name:t.name,color:t.color};
  }
```

to:

```js
  }else{
    const B=BTYPES[tool];
    if(t.owner!=='player')return fail(t.owner?'This plot is not for sale':'Buy this plot first');
    if(['hq','private','bigdepot'].includes(t.type))return fail('That is part of your warehouse');
    if(t.type!=='grass')return fail('Clear this plot first');
    let face=null; for(const d of [2,0,1,3]) if(isRoadT(nbT(t,d))){face=d;break;}
    if(!['road','trees','park'].includes(tool)&&face==null)return fail('Needs a road on one side');
    cost=B.cost; if(save.credits<cost)return fail('You need '+(cost-save.credits)+' more credits');
    t.face=face==null?2:face; t.wild=false; roadCh=tool==='road';
    if(tool==='road'){
      t.type=tool; T.b[t.key]={type:t.type,face:t.face,name:t.name,color:t.color};
    }else{
      let name=t.name,color=t.color;
      if(tool==='shop'){const p=PSHOPS[T.names%PSHOPS.length];name=p[0]+(T.names>=PSHOPS.length?' '+(Math.floor(T.names/PSHOPS.length)+1):'');color=p[1];T.names++;}
      t.type='constr'; t.name=name; t.color=color;
      queueConstr({kind:'town',key:t.key,pos:[tcx(t.i),0,tcz(t.j)],cost,label:name||B.name,meta:{type:tool,face:t.face,name,color}});
    }
  }
```

- [ ] **Step 2: Give construction-site plots a clearer bulldoze message**

Find in `applyTool()`:

```js
  }else if(tool==='bulldoze'){
    if(t.owner!=='player'||!BTYPES[t.type])return fail('Nothing of yours to clear here');
```

Change to:

```js
  }else if(tool==='bulldoze'){
    if(t.type==='constr')return fail('Still under construction — undo instead');
    if(t.owner!=='player'||!BTYPES[t.type])return fail('Nothing of yours to clear here');
```

- [ ] **Step 3: Cancel the matching queue entry on undo**

Change:

```js
function undoTown(){
  const u=townUndo; if(!u){toast('info','Nothing to undo','',ICON.up);return;}
  const t=town.tiles.get(u.key), T=save.town;
  Object.assign(t,u.prev); if(u.owned)T.owned=T.owned.filter(k=>k!==u.key);
  if(BTYPES[t.type]&&t.owner==='player')T.b[t.key]={type:t.type,face:t.face,name:t.name,color:t.color}; else delete T.b[t.key];
  save.credits+=u.cost; writeSave(); townChanged([t.key],u.roadCh||t.type==='road'); townUndo=null;
  toast('info','Undone',u.cost>0?u.cost+' credits back':'',ICON.up); renderPanel();
}
```

to:

```js
function undoTown(){
  const u=townUndo; if(!u){toast('info','Nothing to undo','',ICON.up);return;}
  const t=town.tiles.get(u.key), T=save.town;
  const ci=save.constr.findIndex(c=>c.kind==='town'&&c.key===u.key); if(ci>=0)save.constr.splice(ci,1);
  Object.assign(t,u.prev); if(u.owned)T.owned=T.owned.filter(k=>k!==u.key);
  if(BTYPES[t.type]&&t.owner==='player')T.b[t.key]={type:t.type,face:t.face,name:t.name,color:t.color}; else delete T.b[t.key];
  save.credits+=u.cost; writeSave(); townChanged([t.key],u.roadCh||t.type==='road'); townUndo=null;
  toast('info','Undone',u.cost>0?u.cost+' credits back':'',ICON.up); renderPanel();
}
```

- [ ] **Step 4: Render the construction-site placeholder and its progress riser**

Change:

```js
  else if(t.type==='trees') box(7.6,0.04,7.6,'#d8eedd',0,0.03,0,g,false);
  else if(t.type==='grass') box(7.7,0.03,7.7,'#e0e5f4',0,0.022,0,g,false);
```

to:

```js
  else if(t.type==='trees') box(7.6,0.04,7.6,'#d8eedd',0,0.03,0,g,false);
  else if(t.type==='grass') box(7.7,0.03,7.7,'#e0e5f4',0,0.022,0,g,false);
  else if(t.type==='constr'){
    box(7.4,0.04,7.4,'#e4c98a',0,0.022,0,g,false);
    [[-3.3,-3.3],[3.3,-3.3],[-3.3,3.3],[3.3,3.3]].forEach(([x,z])=>box(0.12,1.1,0.12,'#f0a623',x,0.55,z,g));
    const cc=save.constr.find(x=>x.kind==='town'&&x.key===t.key);
    const r=box(2,1,2,'#f0a623',0,0,0,g,false);
    const p0=cc?constrProgress(cc):0;
    r.scale.y=Math.max(0.02,3*p0); r.position.y=r.scale.y/2;
    if(cc)constrScaffolds.set(cc.id,{riser:r,full:3,kind:'town'});
  }
```

(This is the new tile type `renderTile()` understands. `t.type` is only ever set to `'constr'` by `applyTool()` in Step 1, and only ever read back out by `completeConstr()` in Task 2, which immediately calls `townChanged()` to re-render the tile as its real final type — so `'constr'` is always transient.)

- [ ] **Step 5: Verify syntax**

```bash
cd /e/Development/Games/waretrack && awk '/^<script>$/{f=1;next}/^<\/script>$/{f=0}f' index.html > /tmp/script.js && node --check /tmp/script.js && echo OK
```
Expected: `OK`

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(construction): town planner queues timed construction"
```

---

### Task 5: Self-check and end-to-end browser verification

This project has no test framework (confirmed: no `package.json`, no test runner) and this plan doesn't add one — it follows the project's existing convention of `node --check` for syntax plus manual browser verification (the approach already used for every change made to this file in this session). The one piece of genuinely non-trivial logic — the progress/completion math — gets one small runnable self-check, per the project's own ponytail convention of "non-trivial logic leaves one runnable check behind."

**Files:**
- Modify: `index.html` — insert directly after the `tickConstr()` function from Task 2

- [ ] **Step 1: Add the self-check function**

```js
function constrSelfCheck(){
  const okDur=constrDur(25)===5&&constrDur(1000)===25&&Math.abs(constrDur(100)-12)<1e-9;
  const save0=save.simClock;
  const fakeC={startMin:save.simClock,dur:10};
  const p0=constrProgress(fakeC);
  save.simClock+=10; const p1=constrProgress(fakeC);
  save.simClock+=50; const p2=constrProgress(fakeC);
  save.simClock=save0;
  const okProgress=p0===0&&p1===1&&p2===1;
  console.assert(okDur,'constrDur failed');
  console.assert(okProgress,'progress math failed',{p0,p1,p2});
  return okDur&&okProgress;
}
window.__constrSelfCheck=constrSelfCheck;
```

- [ ] **Step 2: Verify syntax**

```bash
cd /e/Development/Games/waretrack && awk '/^<script>$/{f=1;next}/^<\/script>$/{f=0}f' index.html > /tmp/script.js && node --check /tmp/script.js && echo OK
```
Expected: `OK`

- [ ] **Step 3: Serve the game locally**

```bash
cd /e/Development/Games/waretrack && python3 -m http.server 8765
```
Run this in the background (or a separate terminal) — it needs to stay up for the remaining steps.

- [ ] **Step 4: Run the self-check in a real browser**

Using the `chrome-devtools` MCP tools:
1. `new_page` with `url: "http://localhost:8765/"`
2. `evaluate_script` with `function: "() => window.__constrSelfCheck()"`

Expected: the call returns `true`.

- [ ] **Step 5: Verify a warehouse build queues, shows progress, and completes**

Using `evaluate_script` against the same page (this reaches into the game's module-scope state via the debug hooks already exposed by the self-check step, plus the existing menu flow):
1. Click through the menu to "Free play" at Northgate DC (or call the UI the same way a player would: open Menu → Free play).
2. `evaluate_script`: `"() => { buyBuild('decor:0'); return save.constr.length; }"` — expected: `1` (decor is the cheapest option and finishes fastest, good for a quick manual check).
3. Take a screenshot (`take_screenshot`) and confirm an amber scaffold riser is visible at the decor spot, and the Build panel's "Under construction" row shows a `%` that is `> 0` and `< 100`.
4. `evaluate_script`: `"() => { save.simClock += 10; tickConstr(0); return save.constr.length; }"` — expected: `0` (the entry completed and was removed).
5. Take another screenshot and confirm the decor item now renders normally (no riser) and the Build panel lists it under "Already built here".

- [ ] **Step 6: Verify a town build queues, shows progress, and completes**

1. Open the Town planner, select "House", and tap an owned empty plot (or buy one first if needed).
2. Take a screenshot — confirm the plot shows the fenced construction-site placeholder with a short amber riser, not a house.
3. `evaluate_script`: `"() => { save.simClock += 30; tickConstr(0); return true; }"` (30 sim-minutes comfortably covers a house's ~7-minute `constrDur`).
4. Take a screenshot — confirm the plot now shows the finished house.

- [ ] **Step 7: Verify undo cancels a queued town build**

1. Select "Shop", tap an owned empty plot to queue it.
2. Press `Ctrl+Z` (or trigger `undoTown()` via the UI's undo control).
3. `evaluate_script`: `"() => save.constr.length"` — expected: back to whatever it was before this shop was queued (the entry was removed, not left dangling).
4. Take a screenshot — confirm the plot is back to an empty, for-sale plot.

- [ ] **Step 8: Check for console errors across the whole flow**

`list_console_messages` with `types: ["error","warn"]` — expected: no messages.

- [ ] **Step 9: Stop the local server**

Stop the background `python3 -m http.server` process.

- [ ] **Step 10: Commit**

```bash
git add index.html
git commit -m "feat(construction): add self-check for duration/progress math"
```
