<!doctype html>
<html lang="zh-Hant">
<head>
  <meta charset="utf-8" />
  <title>台灣本島：鄉鎮市區可點選填色（MapChart 風格）v2</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <!-- 移除外部 Leaflet CSS，改為內嵌精簡樣式，避免 dom-to-image 讀取跨網域 CSS 失敗 -->
  <style>
    /* 基礎字體與版面 */
    body { margin:0; font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans TC", sans-serif; }
    .topbar { display:flex; gap:.5rem; align-items:center; flex-wrap:wrap; padding:.6rem .9rem; border-bottom:1px solid #eee; position:sticky; top:0; background:#fff; z-index:1000; }
    .swatch { width:22px; height:22px; border-radius:4px; border:1px solid #ccc; cursor:pointer; display:inline-block; }
    .swatch.active { outline:2px solid #333; outline-offset:2px; }
    #map { height: 78vh; }
    .btn { padding:.45rem .7rem; border:1px solid #ddd; background:#fafafa; cursor:pointer; border-radius:6px; }
    .btn:hover { background:#f0f0f0; }
    .label { color:#444; font-size:.95rem; margin-right:.25rem; }
    .sep { width:1px; height:22px; background:#e7e7e7; margin:0 .5rem; }
    .hint { color:#666; font-size:.85rem; margin-left:auto; }
    .search { padding:.4rem .6rem; border:1px solid #ddd; border-radius:6px; min-width:200px; }

    /* 內嵌精簡版 Leaflet CSS（Canvas 渲染為主） */
    .leaflet-container { position: relative; overflow: hidden; outline: 0; background:#fff; }
    .leaflet-pane, .leaflet-tile, .leaflet-marker-icon, .leaflet-marker-shadow,
    .leaflet-overlay-pane, .leaflet-shadow-pane, .leaflet-marker-pane,
    .leaflet-tooltip-pane, .leaflet-popup-pane, .leaflet-canvas-pane,
    .leaflet-map-pane { position:absolute; left:0; top:0; }
    .leaflet-map-pane { z-index:100; }
    .leaflet-overlay-pane { z-index:400; }
    .leaflet-tooltip-pane { z-index:650; }
    .leaflet-popup-pane { z-index:700; }
    .leaflet-canvas { position:absolute; z-index:200; }
    .leaflet-zoom-animated { will-change: transform; }
    .leaflet-control-container { position:absolute; z-index:800; pointer-events:auto; }
    .leaflet-top, .leaflet-bottom { position:absolute; z-index:1000; pointer-events:none; }
    .leaflet-top { top:0; } .leaflet-bottom { bottom:0; }
    .leaflet-left { left:0; } .leaflet-right { right:0; }
    .leaflet-control { pointer-events:auto; }
    .leaflet-bar { box-shadow:0 1px 5px rgba(0,0,0,.65); border-radius:4px; }
    .leaflet-control-zoom a { background:#fff; border-bottom:1px solid #ccc; width:26px; height:26px; line-height:26px; display:block; text-align:center; text-decoration:none; color:#000; font: 18px/26px Arial,Helvetica,sans-serif; }
    .leaflet-control-zoom a:hover { background:#f4f4f4; }
    .leaflet-control-zoom-in { border-bottom:1px solid #ccc; }
    .leaflet-tooltip { position:absolute; padding:4px 8px; background:#fff; border:1px solid #666; border-radius:2px; color:#111; white-space:nowrap; pointer-events:none; box-shadow:0 1px 3px rgba(0,0,0,.3); font-size:12px; }
  </style>
</head>
<body>
  <div class="topbar">
    <span class="label">顏色：</span>
    <div id="palette"></div>
    <input type="color" id="picker" value="#4575b4" title="自選顏色" />
    <div class="sep"></div>
    <input id="search" class="search" type="search" placeholder="搜尋縣市或鄉鎮市區（例：新北市、板橋區、台北市）" />
    <button class="btn" id="clearBtn">清除填色</button>
    <label class="label"><input type="checkbox" id="exportFull"/> 匯出全台</label>
    <label class="label">解析度
      <select id="exportScale">
        <option value="1">1x</option>
        <option value="2" selected>2x</option>
        <option value="3">3x</option>
        <option value="4">4x</option>
        <option value="6">6x</option>
        <option value="8">8x</option>
      </select>
    </label>
    <label class="label"><input type="checkbox" id="exportHiDPI" checked/> 高解析模式</label>
    <label class="label"><input type="checkbox" id="exportHideControls" checked/> 不含控制</label>
    <button class="btn" id="exportBtn">匯出 PNG</button>
    <span class="hint">提示：左鍵上色 右鍵拖曳 滾輪以游標為錨點縮放 Enter 可觸發「縣市群組填色」。勾選「匯出全台」可輸出完整台灣視圖</span>
  </div>
  <div id="map" role="application" aria-label="台灣本島鄉鎮市區互動地圖"></div>

  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script src="https://unpkg.com/topojson-client@3"></script>
  <script src="https://cdn.jsdelivr.net/npm/dom-to-image-more@3.2.0/dist/dom-to-image-more.min.js"></script>
  <script>
    const ZOOM_MIN = 6, ZOOM_MAX = 12, WHEEL_STEP = 0.25;
    const map = L.map('map', { preferCanvas:true, zoomControl:true, attributionControl:false, minZoom:ZOOM_MIN, maxZoom:ZOOM_MAX, inertia:true, inertiaDeceleration:3500, inertiaMaxSpeed:1500, easeLinearity:0.2, zoomAnimation:true, zoomAnimationThreshold:8, zoomDelta:0.2, zoomSnap:0, scrollWheelZoom:false, maxBoundsViscosity:0.6 }).setView([23.7,121],7);
    map.dragging.disable();

    // 專用條紋 pane（不攔截滑鼠）
    const stripePane = map.createPane('stripePane');
    stripePane.style.zIndex = 650; // overlayPane≈400 < stripePane(650) < popupPane≈700
    stripePane.style.pointerEvents = 'none';

    // 供條紋使用的 SVG renderer（與 Canvas 各自分工）
    const stripeRenderer = L.svg({ pane: 'stripePane' }).addTo(map);

    let currentColor = '#4575b4';
    const defaultStyle = { color:'#666', weight:0.4, fillColor:'#efefef', fillOpacity:0.9 };
    const hoverStyle   = { weight:1.2, color:'#222' };

    const colors = ['#4575b4','#74add1','#abd9e9','#e0f3f8','#fee090','#fdae61','#f46d43','#d73027','#1a9850','#66bd63','#a6d96a','#d9ef8b'];
    const paletteEl = document.getElementById('palette');
    colors.forEach((c,i)=>{ const sw=document.createElement('span'); sw.className='swatch'+(i===0?' active':''); sw.style.background=c; sw.title=c; sw.onclick=()=>{ currentColor=c; document.querySelectorAll('.swatch').forEach(e=>e.classList.remove('active')); sw.classList.add('active'); document.getElementById('picker').value=toHex(c); }; paletteEl.appendChild(sw); });
    document.getElementById('picker').oninput = e => { currentColor = e.target.value; document.querySelectorAll('.swatch').forEach(e => e.classList.remove('active')); };

    const TOWNS_URL='https://cdn.jsdelivr.net/npm/taiwan-atlas/towns-10t.json';
    const EXCLUDE_COUNTIES=new Set(['澎湖縣','金門縣','連江縣']);
    const EXCLUDE_TOWNS=new Set(['綠島鄉','蘭嶼鄉','琉球鄉']);

    let townsLayer;
    const countyIndex = new Map();               // COUNTYNAME -> Layer[]（Canvas）
    const countyFeatures = new Map();            // COUNTYNAME -> Feature[]（源 GeoJSON）

    // clipPath 路徑快取（依縮放層級）
    const clipCache = new Map(); // COUNTYNAME -> { zoom:number, dList:string[] }

    let groupMode = null; // { county, layers, clipId, stripeGroupEl }

    function exitGroupMode(){ if (groupMode) removeStripeDom(); groupMode=null; }

    function unionBoundsOfLayers(layers){
      if (!layers || !layers.length) return null; const valid=[];
      for (const l of layers){ if (l && typeof l.getBounds==='function'){ try{ const b=l.getBounds(); if (!b || (typeof b.isValid==='function' && !b.isValid())) continue; valid.push(b);}catch(_){} } }
      if (!valid.length) return null; let u=L.latLngBounds(valid[0]); for (let i=1;i<valid.length;i++) u.extend(valid[i]); return u;
    }

    const svgns = 'http://www.w3.org/2000/svg';
    function ensureDefs(){ const svg=stripeRenderer._container; let defs=svg.querySelector('defs'); if(!defs){ defs=document.createElementNS(svgns,'defs'); svg.insertBefore(defs, svg.firstChild);} return defs; }
    function removeStripeDom(){ const svg=stripeRenderer._container; if(groupMode?.stripeGroupEl?.parentNode) groupMode.stripeGroupEl.parentNode.removeChild(groupMode.stripeGroupEl); if(groupMode?.clipId){ const cp=svg.querySelector(`#${groupMode.clipId}`); if(cp?.parentNode) cp.parentNode.removeChild(cp);} }
    function buildStripesSVG(countyName, unionBounds, spacingPx=22){
      removeStripeDom();
      const svg = stripeRenderer._container; const root = stripeRenderer._rootGroup || svg; const defs=ensureDefs();
      const clipId = `stripeClip-${Date.now()}-${Math.random().toString(36).slice(2,7)}`;
      const clipPath = document.createElementNS(svgns,'clipPath'); clipPath.setAttribute('id', clipId); defs.appendChild(clipPath);
      const z = map.getZoom(); let cached = clipCache.get(countyName);
      if (!cached || cached.zoom !== z){
        const fc = { type:'FeatureCollection', features: (countyFeatures.get(countyName)||[]) };
        const temp = L.geoJSON(fc, { renderer: stripeRenderer, pane:'stripePane', style:{ opacity:0, fillOpacity:0, stroke:false } }).addTo(map);
        const dList=[]; temp.eachLayer(l=>{ if(l._path){ const d=l._path.getAttribute('d'); if(d) dList.push(d);} }); temp.remove();
        cached = { zoom:z, dList }; clipCache.set(countyName, cached);
      }
      cached.dList.forEach(d=>{ const p=document.createElementNS(svgns,'path'); p.setAttribute('d', d); clipPath.appendChild(p); });
      const g = document.createElementNS(svgns,'g'); g.setAttribute('data-role','county-stripes'); g.setAttribute('clip-path', `url(#${clipId})`);
      const nw=unionBounds.getNorthWest(), se=unionBounds.getSouthEast(); const pNW=map.latLngToLayerPoint(nw), pSE=map.latLngToLayerPoint(se);
      const minX=Math.min(pNW.x,pSE.x), maxX=Math.max(pNW.x,pSE.x), minY=Math.min(pNW.y,pSE.y), maxY=Math.max(pNW.y,pSE.y); const h=maxY-minY;
      for(let x=minX-h; x<=maxX; x+=spacingPx){ const line=document.createElementNS(svgns,'line'); line.setAttribute('x1',x); line.setAttribute('y1',minY); line.setAttribute('x2',x+h); line.setAttribute('y2',maxY); line.setAttribute('stroke','#d00'); line.setAttribute('stroke-width','2'); line.setAttribute('opacity','0.9'); line.setAttribute('vector-effect','non-scaling-stroke'); g.appendChild(line);} root.appendChild(g);
      groupMode.clipId=clipId; groupMode.stripeGroupEl=g;
    }

    function enterGroupMode(countyName){
      exitGroupMode(); const layers=countyIndex.get(countyName)||[]; const union=unionBoundsOfLayers(layers); if(!union) return false;
      groupMode = { county:countyName, layers, clipId:null, stripeGroupEl:null }; buildStripesSVG(countyName, union); layers.forEach(l=>l.bringToFront()); return true;
    }
    const isLayerInCurrentGroup = (lyr)=> !!(groupMode && lyr.feature?.properties?.COUNTYNAME===groupMode.county);

    function safeRedraw(){
      try{ if(townsLayer?.eachLayer) townsLayer.eachLayer(l=> l?.redraw && l.redraw()); }catch(e){ console.warn('逐層重繪失敗', e); }
      finally{ map.invalidateSize({animate:false}); if(groupMode){ const u=unionBoundsOfLayers(groupMode.layers); if(u) buildStripesSVG(groupMode.county, u); } }
    }

    function toggleFill(lyr){
      const prev=lyr.feature?.properties?._fillColor||null;
      if(prev && toHex(prev)===toHex(currentColor)){ lyr.setStyle({ fillColor:defaultStyle.fillColor, fillOpacity:defaultStyle.fillOpacity }); if(lyr.feature?.properties) delete lyr.feature.properties._fillColor; }
      else { lyr.setStyle({ fillColor:currentColor, fillOpacity:0.92 }); if(lyr.feature?.properties) lyr.feature.properties._fillColor=currentColor; }
      try{ lyr.redraw && lyr.redraw(); }catch(_){ }
    }
    function toggleFillGroup(){ if(!groupMode) return; const want=toHex(currentColor); const allSame=groupMode.layers.every(l=> toHex(l.options.fillColor||defaultStyle.fillColor)===want); groupMode.layers.forEach(l=>{ if(allSame){ l.setStyle({ fillColor:defaultStyle.fillColor, fillOpacity:defaultStyle.fillOpacity }); if(l.feature?.properties) delete l.feature.properties._fillColor; } else { l.setStyle({ fillColor:currentColor, fillOpacity:0.92 }); if(l.feature?.properties) l.feature.properties._fillColor=currentColor; } }); }

    // 右鍵拖曳 + 滾輪縮放（以游標為錨點）
    const container = map.getContainer(); let isRightDrag=false, lastPt=null; const debugFlags={ contextmenuPrevented:false };
    container.addEventListener('contextmenu', e=>{ e.preventDefault(); debugFlags.contextmenuPrevented=true; });
    container.addEventListener('pointerdown', e=>{ if(e.button!==2) return; e.preventDefault(); isRightDrag=true; lastPt={x:e.clientX,y:e.clientY}; try{container.setPointerCapture(e.pointerId);}catch(_){} }, {passive:false});
    container.addEventListener('pointermove', e=>{ if(!isRightDrag) return; const dx=e.clientX-lastPt.x, dy=e.clientY-lastPt.y; lastPt={x:e.clientX,y:e.clientY}; map.panBy([-dx,-dy],{animate:false}); if(map.options.maxBounds) map.panInsideBounds(map.options.maxBounds,{animate:false}); }, {passive:false});
    function endRightDrag(e){ if(!isRightDrag) return; isRightDrag=false; try{container.releasePointerCapture(e.pointerId);}catch(_){} }
    container.addEventListener('pointerup', endRightDrag); container.addEventListener('pointercancel', endRightDrag);

    let rafId=0, queuedAnchor=null, queuedZoom=map.getZoom();
    function scheduleWheelZoom(anchor,dz){ queuedAnchor=anchor; queuedZoom=Math.max(ZOOM_MIN,Math.min(ZOOM_MAX,queuedZoom+dz)); if(rafId) return; rafId=requestAnimationFrame(()=>{ const t=queuedZoom,a=queuedAnchor; rafId=0; if(map.setZoomAround) map.setZoomAround(a,t,{animate:true}); else map.setView(a,t,{animate:true}); }); }
    container.addEventListener('wheel', e=>{ e.preventDefault(); scheduleWheelZoom(map.mouseEventToLatLng(e), e.deltaY<0?+WHEEL_STEP:-WHEEL_STEP); }, {passive:false});

    let hoverLayer=null;

    fetch(TOWNS_URL).then(r=>r.json()).then(topo=>{
      const obj=topo.objects.towns||Object.values(topo.objects)[0];
      const gj=topojson.feature(topo,obj);
      gj.features=gj.features.filter(f=> !EXCLUDE_COUNTIES.has(f.properties.COUNTYNAME) && !EXCLUDE_TOWNS.has(f.properties.TOWNNAME));

      townsLayer=L.geoJSON(gj,{
        renderer:L.canvas({padding:0.8}), style:defaultStyle, updateWhenZooming:false,
        onEachFeature:(feature,layer)=>{
          const p=feature.properties; const label=`${p.COUNTYNAME} ${p.TOWNNAME}`;
          if (typeof layer.getBounds==='function'){
            const arr=countyIndex.get(p.COUNTYNAME)||[]; arr.push(layer); countyIndex.set(p.COUNTYNAME,arr);
            const fa=countyFeatures.get(p.COUNTYNAME)||[]; fa.push(feature); countyFeatures.set(p.COUNTYNAME,fa);
          }
          layer.bindTooltip(label,{sticky:true,direction:'top',opacity:.95});
          layer.on({ mouseover:e=>{ hoverLayer=e.target; e.target.setStyle(hoverStyle); }, mouseout:e=>{ if(hoverLayer===e.target) hoverLayer=null; e.target.setStyle({weight:defaultStyle.weight,color:defaultStyle.color}); }, click:e=>{ const lyr=e.target; if(groupMode){ if(isLayerInCurrentGroup(lyr)) toggleFillGroup(); else { exitGroupMode(); toggleFill(lyr);} } else { toggleFill(lyr);} } });
        }
      }).addTo(map);

      const bounds=townsLayer.getBounds(); map.fitBounds(bounds,{padding:[20,20]}); map.setMaxBounds(bounds.pad(0.06));
      map.on('zoomend', ()=>{ clipCache.clear(); safeRedraw(); });
      map.on('resize', ()=>{ clipCache.clear(); safeRedraw(); });
      map.on('click', e=>{ const t=e.originalEvent?.target; if (!t?.closest || !t.closest('.leaflet-interactive')) exitGroupMode(); });

      // 照護：最小化自動檢查，避免重
      setTimeout(()=>{
        console.group('自動檢查 / 條紋快取與 dom-to-image 可用性');
        // 條紋快取
        const anyCounty=[...countyIndex.keys()][0]; const ok=enterGroupMode(anyCounty);
        console.assert(ok && groupMode?.stripeGroupEl, '進入群組模式並建立條紋'); exitGroupMode();
        // dom-to-image smoke test（避免跨網域 CSS 問題）
        const probe=document.createElement('div'); probe.style.cssText='width:10px;height:10px;background:#000'; document.body.appendChild(probe);
        domtoimage.toPng(probe).then(()=>console.log('dom-to-image OK')).catch(e=>console.error('dom-to-image 失敗', e)).finally(()=>probe.remove());
        console.groupEnd();
      },0);
    }).catch(err=>{ alert('載入台灣鄉鎮市區資料失敗，請稍後再試或檢查網路。'); console.error(err); });

    document.getElementById('clearBtn').onclick=()=>{ if(!townsLayer) return; townsLayer.eachLayer(lyr=>{ lyr.setStyle({fillColor:defaultStyle.fillColor, fillOpacity:defaultStyle.fillOpacity}); if (lyr.feature?.properties) delete lyr.feature.properties._fillColor; }); };

    // ===== 匯出（離屏渲染，無外部 CSS 依賴） =====
    function getExportOptions(){ const scale=parseFloat(document.getElementById('exportScale')?.value||'1'); const hideControls=document.getElementById('exportHideControls')?.checked!==false; const hiDPI=document.getElementById('exportHiDPI')?.checked===true; return { scale:isFinite(scale)&&scale>0?scale:1, hideControls, hiDPI }; }

    const waitFrames = (n=2)=> new Promise(r=>{ let i=0; const step=()=> (++i>=n)?r():requestAnimationFrame(step); requestAnimationFrame(step); });

    function snapshotFeatureCollection(){ const fc={ type:'FeatureCollection', features:[] }; if(!townsLayer) return fc; townsLayer.eachLayer(l=>{ if(!l.feature) return; const f=JSON.parse(JSON.stringify(l.feature)); const fill=l.feature?.properties?._fillColor; if(fill) f.properties._fillColor=fill; fc.features.push(f); }); return fc; }

    async function captureOffscreenPNG({ full=false, scale=2 }){
      const W=Math.round(document.getElementById('map').clientWidth*scale);
      const H=Math.round(document.getElementById('map').clientHeight*scale);
      const stage=document.createElement('div'); stage.style.cssText=`position:fixed; left:-10000px; top:-10000px; width:${W}px; height:${H}px; background:white;`; document.body.appendChild(stage);
      const off=L.map(stage,{ preferCanvas:true, zoomControl:false, attributionControl:false, inertia:false });
      const renderer=L.canvas({padding:0.8});
      const fc=snapshotFeatureCollection();
      const offLayer=L.geoJSON(fc,{ renderer, style:(feat)=>({ color:'#666', weight:0.4, fillColor: feat.properties?._fillColor || '#efefef', fillOpacity:0.9 }) }).addTo(off);
      if(full) off.fitBounds(offLayer.getBounds(),{padding:[2,2]}); else off.setView(map.getCenter(), map.getZoom(), {animate:false});
      await waitFrames(2);
      // 需要條紋時建立裁切條紋（僅當前群組）
      if(full && groupMode){ const svgR=L.svg().addTo(off); const svg=svgR._container; let defs=svg.querySelector('defs'); if(!defs){ defs=document.createElementNS(svgns,'defs'); svg.insertBefore(defs, svg.firstChild);} const clipId=`off-clip-${Date.now()}-${Math.random().toString(36).slice(2,7)}`; const cp=document.createElementNS(svgns,'clipPath'); cp.setAttribute('id',clipId); defs.appendChild(cp); const temp=L.geoJSON({ type:'FeatureCollection', features:countyFeatures.get(groupMode.county)||[] },{ renderer:svgR, style:{opacity:0, fillOpacity:0, stroke:false} }).addTo(off); temp.eachLayer(l=>{ if(l._path){ const p=l._path.cloneNode(true); p.removeAttribute('class'); cp.appendChild(p);} }); temp.remove(); const b=offLayer.getBounds(); const pNW=off.latLngToLayerPoint(b.getNorthWest()), pSE=off.latLngToLayerPoint(b.getSouthEast()); const minX=Math.min(pNW.x,pSE.x), maxX=Math.max(pNW.x,pSE.x), minY=Math.min(pNW.y,pSE.y), maxY=Math.max(pNW.y,pSE.y); const h=maxY-minY; const g=document.createElementNS(svgns,'g'); g.setAttribute('clip-path',`url(#${clipId})`); for(let x=minX-h; x<=maxX; x+=22){ const line=document.createElementNS(svgns,'line'); line.setAttribute('x1',x); line.setAttribute('y1',minY); line.setAttribute('x2',x+h); line.setAttribute('y2',maxY); line.setAttribute('stroke','#d00'); line.setAttribute('stroke-width','2'); line.setAttribute('opacity','0.9'); line.setAttribute('vector-effect','non-scaling-stroke'); g.appendChild(line);} svg.appendChild(g); }
      await waitFrames(1);
      const dataUrl=await domtoimage.toPng(stage,{ bgcolor:'white', width:W, height:H, filter:()=>true });
      let finalUrl=dataUrl;
      if(full){ const b=offLayer.getBounds(); const nw=off.latLngToContainerPoint(b.getNorthWest()); const se=off.latLngToContainerPoint(b.getSouthEast()); const left=Math.max(0,Math.floor(Math.min(nw.x,se.x))-2); const top=Math.max(0,Math.floor(Math.min(nw.y,se.y))-2); const right=Math.min(W,Math.ceil(Math.max(nw.x,se.x))+2); const bottom=Math.min(H,Math.ceil(Math.max(nw.y,se.y))+2); const rect={ x:left, y:top, width:Math.max(1,right-left), height:Math.max(1,bottom-top) }; finalU
