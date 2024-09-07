<template>
    <div>
      <div id="map">
        <button class="btn" @click="resetMap">复位</button>
        <div class="search-container">
          <input type="text" id="locationInput" placeholder="输入具体地点或省市县" v-model="location">
          <button @click="searchLocations">搜索</button>
          <div class="search-results" id="searchResults" v-if="searchResults.length">
            <div v-for="result in searchResults" :key="result.id" @click="goToLocation(result.location)">
              {{ result.name }}
              <div class="location-info">{{ result.pname }} {{ result.cityname }} {{ result.adname }}</div>
            </div>
          </div>
        </div>
        <div class="coordinate-display">
          <div>
            <span>X：</span>
            <input type="text" id="longitude" readonly :value="clickedCoordinate[0]">
          </div>
          <div>
            <span>Y：</span>
            <input type="text" id="latitude" readonly :value="clickedCoordinate[1]">
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script>
  import 'ol/ol.css';
  import { Map, View } from 'ol';
  import TileLayer from 'ol/layer/Tile';
  import XYZ from 'ol/source/XYZ';
  import VectorLayer from 'ol/layer/Vector';
  import VectorSource from 'ol/source/Vector';
  import Feature from 'ol/Feature';
  import Point from 'ol/geom/Point';
  import Overlay from 'ol/Overlay';
  import { ZoomToExtent, ZoomSlider, FullScreen } from 'ol/control';
  
  export default {
    data() {
      return {
        map: null,
        source: null,
        overlay: null,
        location: '',
        searchResults: [],
        clickedCoordinate: [0, 0]
      };
    },
    mounted() {
      this.initializeMap();
    },
    methods: {
      initializeMap() {
        const gaode = new TileLayer({
          title: "高德地图",
          source: new XYZ({
            url: 'http://wprd0{1-4}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&style=7&x={x}&y={y}&z={z}',
            wrapX: false
          })
        });
  
        this.map = new Map({
          target: 'map',
          layers: [gaode],
          view: new View({
            center: [118.55, 36.6],
            zoom: 10,
            projection: 'EPSG:4326'
          })
        });
  
        this.map.addControl(new ZoomToExtent({
          extent: [117.0, 35, 120, 37]
        }));
  
        this.map.addControl(new ZoomSlider());
        this.map.addControl(new FullScreen());
  
        this.source = new VectorSource({});
        this.map.addLayer(new VectorLayer({
          source: this.source
        }));
  
        const container = document.createElement('div');
        container.className = 'ol-popup';
        const content = document.createElement('div');
        container.appendChild(content);
        this.overlay = new Overlay({
          element: container,
          positioning: 'bottom-center',
          stopEvent: false
        });
        this.map.addOverlay(this.overlay);
  
        this.map.on('click', (e) => {
          const point = new Feature({
            geometry: new Point(e.coordinate)
          });
          this.source.clear();
          this.source.addFeature(point);
          this.map.getView().animate({
            center: e.coordinate,
          });
          this.clickedCoordinate = e.coordinate;
        });
      },
  
      resetMap() {
        this.map.getView().setCenter([118.55, 36.6]);
        this.map.getView().setZoom(10);
      },
  
      searchLocations() {
        if (!this.location) {
          alert('请输入具体地点或省市县');
          return;
        }
  
        fetch(`https://restapi.amap.com/v3/place/text?keywords=${encodeURIComponent(this.location)}&citylimit=true&offset=10&key=2ccff1403c08354aeeaa5ea436d3f428`)
          .then(response => response.json())
          .then(data => {
            if (data.status === '1' && data.pois.length > 0) {
              this.searchResults = data.pois;
            } else {
              alert('未找到相关地点，请重新输入');
            }
          })
          .catch(error => {
            console.error('Error:', error);
            alert('请求失败，请重试');
          });
      },
  
      goToLocation(location) {
        if (!location) {
          alert('请选择一个地点');
          return;
        }
  
        const [lng, lat] = location.split(',').map(parseFloat);
        this.map.getView().animate({
          center: [lng, lat],
          zoom: 16
        });
        this.searchResults = [];
      }
    }
  };
  </script>
  
  <style scoped>
  #map {
    width: 100%;
    height: 100vh;
    position: relative;
    z-index: 0; /* 确保地图在全屏模式下不会遮挡其他元素 */
  }
  .btn {
    position: absolute;
    top: 2em;
    right: 4em;
    z-index: 1001; /* 确保按钮在全屏模式下可见 */
  }
  .ol-zoomslider {
    top: 7.5em;
    left: .5em;
    height: 12.5em;
    z-index: 1001; /* 确保缩放滑块在全屏模式下可见 */
  }
  .ol-popup {
    position: absolute;
    background-color: white;
    box-shadow: 0 0.0625em 0.25em rgba(0,0,0,0.2);
    padding: 0.9375em;
    border-radius: 0.625em;
    border: 0.0625em solid #cccccc;
    bottom: 0.75em;
    left: -3.125em;
    min-width: 17.5em;
    z-index: 1001; /* 确保弹出框在全屏模式下可见 */
  }
  .ol-popup:after, .ol-popup:before {
    top: 100%;
    border: solid transparent;
    content: " ";
    height: 0;
    width: 0;
    position: absolute;
    pointer-events: none;
  }
  .ol-popup:after {
    border-top-color: white;
    border-width: 0.625em;
    left: 3em;
    margin-left: -0.625em;
  }
  .ol-popup:before {
    border-top-color: #cccccc;
    border-width: 0.6875em;
    left: 3em;
    margin-left: -0.6875em;
  }
  .search-container {
    position: absolute;
    top: 2em;
    left: 4em;
    z-index: 1001; /* 确保搜索容器在全屏模式下可见 */
    display: flex;
    flex-direction: column;
    align-items: flex-start;
  }
  .search-container input {
    padding: 0.3125em;
    width: 12.5em;
    margin-bottom: 0.3125em;
  }
  .search-container button {
    padding: 0.3125em;
  }
  .search-results {
    position: absolute;
    top: 4em;
    left: 4em;
    z-index: 1001; /* 确保搜索结果在全屏模式下可见 */
    background-color: white;
    border: 1px solid #ccc;
    max-height: 200px;
    overflow-y: auto;
    display: block;
    width: 25em;
  }
  .search-results div {
    padding: 5px;
    cursor: pointer;
  }
  .search-results div:hover {
    background-color: #f0f0f0;
  }
  .search-results .location-info {
    font-size: 0.8em;
    color: #add8e6;
  }
  .coordinate-display {
    position: absolute;
    bottom: 1em;
    left: 1em;
    z-index: 1001; /* 确保坐标显示在全屏模式下可见 */
    display: flex;
    flex-direction: column;
  }
  .coordinate-display div {
    display: flex;
    align-items: center;
  }
  .coordinate-display span {
    margin-right: 0.3125em;
  }
  .coordinate-display input {
    padding: 0.3125em;
    width: 12.5em;
    margin-bottom: 0.3125em;
  }
  @media (max-width: 600px) {
    .search-container input {
      width: 9.375em;
    }
  }
  </style>
  