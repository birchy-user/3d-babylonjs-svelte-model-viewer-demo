# Babylon.js piemērs - 3D objektu, modeļu un ainu izveidošana, importēšana un animēšana, izmantojot Babylon.js

## Galvenās atkarības
- NPM (Node Package Manager) versija (pārbaudāma ar `npm -v`) >= 6.14.15
- Node.js (pārbaudāma ar `node -v` terminālī) >= 14.18.0

## Kā palaist?
1. Pēc projekta klonēšanas no GitHub ar `git clone` komandu, jāatver projekta atrašanās vieta (galvenā mape) un jāizpilda komanda `npm install`, lai projekta galvenajā mapē uz lokālās ierīces būtu pieejamas nepieciešamās bibliotēkas un pakotnes
2. Tālāk ir divi varianti, kā var palaist projektu lokāli:
    1. `npm run dev` - tiek palaista izstrādes (_development_) vide, kuru lokāli var atvērt šajā saitē: `http://localhost:5173`
    
    2. `npm run build` - tiek izveidota optimizēta un izlaidei (_production_) gatava minificēta projekta versija ar minimālajiem nepieciešamajiem failiem mapē `dist`. Lai varētu atvērt šīs vides priekšskatu, ir jāizpilda komanda `npm run preview` - tālāk šo vietni lokāli var atvērt šajā saitē: `http://localhost:4173/`

## Piekļuve tiešsaistē
Ja projektu neizdodas palaist uz lokālās ierīces, tad šī projekta jaunāko versiju var apskatīt [šajā saitē](https://robertspelle-3d-babylonjs-svelte-model-viewer-demo.vercel.app/), kas tiek uzturēta ar Vercel pakalpojuma palīdzību


## Palīdzība un jautājumi
Jautājumu gadījumā sazināties ar autoru Robertu Pelli rp18023 (roberts.pelle@gmail.com)

## Projektā iekļauto modeļu izmantošana
Visi ainā izmantotie 3D modeļi atrodas mapē `public/assets/models/`:
* `soccer_ball.glb` ir brīvi pieejams futbola bumbas 3D modelis, kuru var atrast [šajā saitē](https://sketchfab.com/3d-models/soccer-ball-a51de12e975a425184496fbabc728ca3)
* `water_polo_goal_FINAL.mbj` un `water_polo_goal_FINAL.obj` ir autora oriģināli veidots ūdenspolo vārtu modelis OBJ formātā programmatūrā _Blender 4.1._