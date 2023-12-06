<script lang="ts">
import { onMount } from 'svelte';
import * as icons from '../icons';
// @ts-ignore
import RangeSlider from 'svelte-range-slider-pips'
let values = [1960,1990];

// @ts-ignore
declare const L: any;
let map: any;
// let activePopup: any;
let displayedLocations = new Set();

onMount(() => {
    (window as any).adjustPopup = function() {
        map.eachLayer((layer: any) => {
            if (layer.getPopup && layer.isPopupOpen && layer.isPopupOpen()) {
                layer.getPopup()._adjustPan();
                return;
            }
        });
    }
// function mapAction() {
    const baseMaps = {
        '淡色地図': L.tileLayer("https://cyberjapandata.gsi.go.jp/xyz/pale/{z}/{x}/{y}.png", {
        attribution: "<a href='http://maps.gsi.go.jp/development/ichiran.html'>地理院タイル</a>"
        }),
        '標準地図': L.tileLayer("https://cyberjapandata.gsi.go.jp/xyz/std/{z}/{x}/{y}.png", {
        attribution: "<a href='http://maps.gsi.go.jp/development/ichiran.html'>地理院タイル</a>"
        }),
        '色別標高図': L.tileLayer("https://cyberjapandata.gsi.go.jp/xyz/relief/{z}/{x}/{y}.png", {
        attribution: "<a href='http://maps.gsi.go.jp/development/ichiran.html'>地理院タイル</a>"
        }),
        '写真': L.tileLayer("https://cyberjapandata.gsi.go.jp/xyz/seamlessphoto/{z}/{x}/{y}.jpg", {
        attribution: "<a href='http://maps.gsi.go.jp/development/ichiran.html'>地理院タイル</a>"
        }),
        'OpenStreetMap': L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
        attribution: "&copy; <a href='http://osm.org/copyright'>OpenStreetMap</a> contributors"
        })
    };

    let group = L.layerGroup()
    const overlay = {
        "マップピン": group,
    };

    map = L.map("map", L.extend({
        zoomControl:false,
        zoom: 17,
        center: [35.31873, 139.55114],
        layers: baseMaps['写真']
        }, L.Hash.parseHash(location.hash)));
    
    L.control.scale({ maxWidth: 200, position:'bottomright', imperial: false}).addTo(map);
    L.control.zoom({ position: 'bottomleft'}).addTo(map);
    L.control.layers(baseMaps,overlay,{collapsed:false}).addTo(map);
    L.hash(map);

    // ここからSPAQL
    group.addTo(map);
    let lang = "ja";
    let regexp = location.search.match(/^\?(?<lang>[[:alpha:]_]+)$/);
    if (regexp) {
        lang = regexp[1];
    };
    // if (location.search.match(/^\?([[:alpha:]_]+)$/)) lang = RegExp.$1

    // map.on('popupopen', function(e: any) {
    //     activePopup = e.popup;
    // });

    // map.on('popupclose', function() {
    //     activePopup = null;
    // });

    map.on("moveend", async function() {
        let bounds = map.getBounds();
        let sparql =`
        SELECT ?photoObject ?label ?image ?location ?lat ?lon ?creatorUri ?creator ?time ?license ?licenseUri
        WHERE {
                SERVICE <http://virtuoso.orbit.supply:8890/sparql> {
                    ?photoObject
                        jps:spatial ?location .
                    ?location schema:geo [
                        schema:latitude ?lat ;
                        schema:longitude ?lon
                    ] .
                    FILTER(${bounds.getSouth()} <= ?lat && ?lat <= ${bounds.getNorth()} && ${bounds.getWest()} <= ?lon && ?lon <= ${bounds.getEast()})
                } .
                ?photoObject
                    jps:sourceInfo/schema:provider <https://jpsearch.go.jp/entity/chname/鎌倉市図書館近代史資料室> ;
                    rdfs:label ?label ;
                    schema:image ?image ;
                    jps:accessInfo/schema:license ?licenseUri .
                ?licenseUri rdfs:label ?license .
                OPTIONAL { ?photoObject schema:creator ?creatorUriOP . } .
                    BIND(COALESCE(?creatorUriOP, "./") AS ?creatorUri)
                OPTIONAL { ?creatorUri rdfs:label ?creatorOP . } .
                    BIND(COALESCE(?creatorOP, "不明") AS ?creator)
                OPTIONAL { ?photoObject jps:temporal/rdfs:label ?timeOP . } .
                    BIND(COALESCE(?timeOP, "不明") AS ?time)
        } LIMIT 20
        `;

        group.eachLayer((layer: any) => {
            if (!bounds.contains(layer.getLatLng())) {
                    group.removeLayer(layer);
                    displayedLocations.delete(layer.options.myCustomId);
                }
        });

        let url = new URL(`https://jpsearch.go.jp/rdf/sparql?query=${encodeURIComponent(sparql)}`)
        const response = await fetch(url, {
            "headers": {
                "accept": "application/sparql-results+json"
            },
            "method": "GET",
            "mode": "cors"
        })
        const json = await response.json()
        console.log(json);
        json.results.bindings.forEach((obj: any) => {
            // const lonlat = obj.location.value.match(/^Point\((?<lon>.+) (?<lat>.+)\)$/);
            // const lonlat = obj.location.value
            // if (!lonlat) return;
            if (obj.location.value === undefined) return;
            // const lon = parseFloat(lonlat.groups.lon);
            // const lat = parseFloat(lonlat.groups.lat);
            const lon = obj.lon.value
            const lat = obj.lat.value
            const locationId = obj.photoObject.value.match(/\/data\/(.+)$/)[0]
            // let time = "不明"
            // if (obj.time !== undefined) {
            //     time = obj.time.Value
            // }
            
            const qObject = obj.label.value.match(/^Q[0-9]+$/)
            let className = qObject ? 'wikidata q' : 'wikidata'
            let display = `
                <img class="icon" src="${obj.image.value}" alt="${obj.label.value}" />
                `
            let popup = `
                <section class="prosed">
                    <figure><img src="${obj.image.value}" alt="${obj.label.value}" onload="adjustPopup();" /></figure>
                    <div class="card-body">
                        <h2 class="card-title">${obj.label.value}</h2>
                        <div class="overflow-x-auto">
                            <table class="table">
                                <!-- head -->
                                <thead>
                                    <tr><th>Property</th><th>Value</th></tr>
                                </thead>
                                <tbody>
                                    <tr><th>撮影者</th><td><a href="${obj.creatorUri.value}">${obj.creator.value}</a></td></tr>
                                    <tr><th>撮影年</th><td>${obj.time.value}</td></tr>
                                    <tr><th>ライセンス</th><td><a href="${obj.licenseUri.value}">${obj.license.value}</a></td></tr>
                                </tbody>
                            </table>
                        </div>
                        <div class="card-actions justify-end">
                            <a href="${obj.photoObject.value}" class="btn">
                                ${icons.jpsearch}
                                <span>詳細</span>
                            </a>
                            <button class="btn">
                                ${icons.chat_bubble_left_right}
                                <span>コメント</span>
                            </button>
                            <button class="btn">
                                ${icons.pencil_square}
                            </button>
                        </div>
                    </div>
                    
                </section>
                `
            
            if (!displayedLocations.has(locationId)) {
                displayedLocations.add(locationId);
                L.marker([lat, lon], {
                    icon: L.divIcon({
                    html: display,
                    className: className,
                    iconSize: ['auto', 'auto'],
                    iconAnchor: [30,72]
                    }),
                    myCustomId: locationId
                    })
                    .addTo(group)
                    .bindPopup(popup, {
                        className: 'card w-96 bg-base-100 shadow-xl',
                        maxWidth: "auto",
                        minWidth: "auto",
                        autoPan: true
                    })
            }

        });
    }).fire("moveend");
});
</script>
    
<h1>Welcome to SvelteKits</h1>
<p>Visit <a href="https://kit.svelte.dev">kit.svelte.dev</a> to read the documentation</p>
<!-- <div class="wikidata q"></div> -->


<!-- <div id="map" use:mapAction></div> -->
<div id="map"></div>

<!-- <RangeSlider range float pips pipstep={10} first="label" rest="label" min={1900} max={2023} bind:values /> -->
<RangeSlider range float pips pipstep={10} min={1900} max={2023} bind:values />
    
<style>
:root {
  --range-slider:          #d7dada; /* slider main background color */
  --range-handle-inactive: #333; /* inactive handle color */
  --range-handle:          #838de7; /* non-focussed handle color */
  --range-handle-focus:    #4a40d4; /* focussed handle color */
  --range-handle-border:   var(--range-handle); /* handle border color */
  --range-range-inactive:  var(--range-handle-inactive); /* inactive range bar background color */
  --range-range:           var(--range-handle-focus); /* active range background color */
  --range-float-inactive:  var(--range-handle-inactive); /* inactive floating label background color */
  --range-float:           var(--range-handle-focus); /* floating label background color */
  --range-float-text:      white; /* text color on floating label */

  --range-pip:                #d7dada; /*lightslategray; /* color of the base pips */
  --range-pip-text:           var(--range-pip); /* color of the base labels */
  --range-pip-active:         #d7dada; /*darkslategrey; /* active pips (when handle is on a slider-stop) */
  --range-pip-active-text:    #fff; /* var(--range-pip-active); /* active labels (when handle is on a slider-stop) */
  --range-pip-hover:          #333; /*darkslategrey; /* when a slider-stop is hovered */
  --range-pip-hover-text:     var(--range-pip-hover); /* when a slider-stop is hovered */
  --range-pip-in-range:       var(--range-pip-active); /* pips inside the range */
  --range-pip-in-range-text:  #d7dada; /*var(--range-pip-active-text); /* labels inside the range */
}
:global(.range) {
    width: auto;
    overflow: visible;
}
:global(.rangeSlider)  {
    position: absolute !important;
    left: 5rem;
    right: 10rem;
    bottom: 0;
    z-index: 1000;
}
#map {
    position: absolute;
    inset: 0;
}
/* #map div:has(.wikidata) { */
#map :global(.wikidata) { 
    white-space: nowrap;
    font-family: inherit;
    font-size: inherit;
    /* margin-right: 0.2rem; */
    /* margin-bottom: 0.2rem; */
    padding: 6px;
    border: none;
    border-radius: 0.3rem;
    outline: none;

    /* box-shadow: 0 10px 20px rgba(0,0,0,0.19), 0 6px 6px rgba(0,0,0,0.23); */
    box-shadow: 0 14px 28px rgba(0,0,0,0.25), 0 10px 10px rgba(0,0,0,0.22);

    color: #1d1d1d;
    background-color: #efefef;
}
#map :global(.wikidata.q) {
    color: #ff5627;
}
#map :global(.wikidata)::after {
    content: '';
    position: absolute;
    bottom: -22px; /* 吹き出しの下部からしっぽを出す */
    left: 50%; /* 左右中央に配置 */
    margin-left: -12px; /* しっぽの幅の半分だけ左にずらす */
    border-width: 12px;
    border-style: solid;
    border-color: #efefef transparent transparent; /* 吹き出しと同じ背景色 */
}
#map :global(.icon) {
    height: 48px;
    width: 48px;
    object-fit: cover;
    object-position: center;
    vertical-align: middle;
}
#map :global(.leaflet-popup-content-wrapper) {
    padding: 0;
    overflow: hidden;
}
#map :global(.leaflet-popup-content) {
    margin: 0;
}
#map :global(.card) {
    position: absolute;
    display: block;
}
#map :global(.chipdd) {
    width: 380px;
    object-fit: contain;
}
:global(.leaflet-control svg) {
    display: inline;
}
</style>