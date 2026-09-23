<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>AutoTrack</title>
<style>
:root{font-family:system-ui,-apple-system,Segoe UI,Roboto,sans-serif;color:#172033;background:#f3f5f8}
*{box-sizing:border-box}body{margin:0;min-height:100vh}header{background:#1769e0;color:#fff;padding:18px 16px;position:sticky;top:0;z-index:2}header h1{max-width:620px;margin:0 auto;font-size:22px}.wrap{max-width:620px;margin:auto;padding:16px}.card{background:#fff;border-radius:16px;padding:18px;margin-bottom:14px;box-shadow:0 2px 12px #17203312}h2{font-size:18px;margin:0 0 14px}.grid{display:grid;grid-template-columns:1fr 1fr;gap:10px}.metric{background:#eef5ff;border-radius:12px;padding:14px;text-align:center}.metric.cost{background:#edfbf1}.label{font-size:13px;color:#526078}.value{font-size:30px;font-weight:750;margin-top:4px}.unit{font-size:13px;color:#526078}.actions{display:grid;gap:10px;margin-top:16px}button{border:0;border-radius:12px;padding:15px;font-size:17px;font-weight:700;color:#fff;background:#1769e0;cursor:pointer}button:disabled{opacity:.55;cursor:not-allowed}button.stop{background:#d92d3f}.status{display:flex;justify-content:space-between;align-items:center;margin-bottom:14px}.badge{border-radius:99px;padding:6px 10px;font-size:12px;font-weight:700;background:#edf0f4;color:#526078}.badge.on{background:#d9f7e2;color:#16733a}label{display:block;font-size:14px;margin:12px 0 6px}input{width:100%;padding:12px;border:1px solid #ccd3df;border-radius:9px;font-size:16px}.hint,#log{font-size:13px;color:#526078;line-height:1.45}.hint{margin:14px 0 0;text-align:center}.error{color:#b42318}.ok{color:#16733a}.log{background:#f5f7fa;border-radius:10px;padding:10px;min-height:44px;white-space:pre-wrap}.hidden{display:none}.small{font-size:12px;color:#6b7280}
</style>
</head>
<body>
<header><h1>🚗 AutoTrack</h1></header>
<main class="wrap">
<section class="card">
<div class="status"><h2>Viagem atual</h2><span id="status" class="badge">Parado</span></div>
<div class="grid"><div class="metric"><div class="label">Distância</div><div id="distance" class="value">0,00</div><span class="unit">km</span></div><div class="metric cost"><div class="label">Custo estimado</div><div id="cost" class="value">0,00</div><span class="unit">R$</span></div></div>
<div class="actions"><button id="start" type="button">▶ Iniciar viagem</button><button id="stop" type="button" class="stop" disabled>■ Encerrar viagem</button></div>
<p id="message" class="hint">O acesso à localização será solicitado ao iniciar.</p>
</section>
<section class="card"><h2>Configurações do veículo</h2><label for="price">Preço do combustível (R$/L)</label><input id="price" type="number" inputmode="decimal" min="0.01" step="0.01" value="5.80"><label for="consumption">Consumo médio (km/L)</label><input id="consumption" type="number" inputmode="decimal" min="0.1" step="0.1" value="10.0"></section>
<section class="card"><h2>Diagnóstico</h2><div id="log" class="log">Pronto para iniciar.</div><p class="small">Use HTTPS e permita a localização no navegador. Para melhor resultado, mantenha a tela ativa e fique em local aberto.</p></section>
</main>
<script>
'use strict';
(() => {
  const $ = id => document.getElementById(id);
  const start = $('start'), stop = $('stop'), status = $('status'), distance = $('distance'), cost = $('cost'), message = $('message'), log = $('log'), price = $('price'), consumption = $('consumption');
  let watchId = null, last = null, totalKm = 0;
  const R = 6371;
  const fmt = n => Number(n).toLocaleString('pt-BR',{minimumFractionDigits:2,maximumFractionDigits:2});
  function setMessage(text, cls=''){message.textContent=text;message.className='hint '+cls;}
  function setLog(text, cls=''){log.textContent=text;log.className='log '+cls;}
  function update(){distance.textContent=fmt(totalKm);const p=parseFloat(price.value)||0,c=parseFloat(consumption.value)||0;cost.textContent=fmt(c>0?totalKm*p/c:0);}
  function haversine(a,b){const dLat=(b.lat-a.lat)*Math.PI/180,dLon=(b.lon-a.lon)*Math.PI/180;const x=Math.sin(dLat/2)**2+Math.cos(a.lat*Math.PI/180)*Math.cos(b.lat*Math.PI/180)*Math.sin(dLon/2)**2;return R*2*Math.atan2(Math.sqrt(x),Math.sqrt(1-x));}
  function onPosition(pos){const c=pos.coords, current={lat:c.latitude,lon:c.longitude,accuracy:c.accuracy};if(last){const d=haversine(last,current);if(d>=0.005 && d<1){totalKm+=d;update();}}last=current;setLog('GPS ativo\nLat: '+current.lat.toFixed(6)+'\nLon: '+current.lon.toFixed(6)+'\nPrecisão: '+Math.round(current.accuracy)+' m','ok');setMessage('Rastreamento ativo. Distância sendo calculada.','ok');}
  function onError(e){const texts={1:'Permissão de localização negada.',2:'Posição indisponível. Verifique o GPS.',3:'Tempo limite. Tente novamente em local aberto.'};setLog('Erro '+e.code+': '+(texts[e.code]||e.message),'error');setMessage(texts[e.code]||'Erro ao obter localização.','error');stopTracking(false);if(e.code===1) alert('Permissão negada. Abra as permissões do site e permita Localização; depois recarregue a página.');}
  function startTracking(){if(!window.isSecureContext){setMessage('Abra este endereço usando HTTPS.','error');setLog('Contexto inseguro: '+location.href,'error');return;}if(!navigator.geolocation){setMessage('Este navegador não suporta GPS.','error');return;}totalKm=0;last=null;update();status.textContent='Iniciando';status.className='badge on';start.disabled=true;stop.disabled=false;setMessage('Solicitando acesso à localização...');setLog('Chamando watchPosition...');watchId=navigator.geolocation.watchPosition(onPosition,onError,{enableHighAccuracy:true,timeout:20000,maximumAge:3000});}
  function stopTracking(show=true){if(watchId!==null){navigator.geolocation.clearWatch(watchId);watchId=null;}status.textContent='Parado';status.className='badge';start.disabled=false;stop.disabled=true;if(show){setMessage('Viagem encerrada: '+fmt(totalKm)+' km.');setLog('Rastreamento encerrado.');}}
  start.addEventListener('click',startTracking);stop.addEventListener('click',()=>stopTracking(true));price.addEventListener('input',update);consumption.addEventListener('input',update);update();
})();
</script>
</body>
</html>
