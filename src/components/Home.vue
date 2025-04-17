<template>
  <div class="tools-wrapper">
    <date-picker v-model:value="dt" format="YYYY-MM-DD HH" show-time placeholder="Select Time" @ok="onOk" />
    <a-button size="small" type="primary" style="margin: 0 10px;" @click="dt = dt.subtract(1, 'hour');loadPointData()">上一小时</a-button>
    <a-button size="small" type="primary" @click="dt = dt.add(1, 'hour');loadPointData()">下一小时</a-button>
  </div>
    
  <div id="map"></div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
import { DatePicker, Button as AButton, message as Message } from 'ant-design-vue';
import dayjs from 'dayjs';

let baseUrl = process.env.NODE_ENV === 'production' ?  "" : "http://127.0.0.1:8000"
console.log(process.env.NODE_ENV, baseUrl);

const dt = ref(dayjs())
let map = ref(null)
const choosePoint = ref(null)
const trackerLayer = ref(null)

onMounted(()=>{
  initMap()
})

function onOk(value) {
  loadPointData();
}

const loadPointData = async () => {
  const point = choosePoint.value
  if (!point) {
    return;
  }
  
  try {
    const trackerUrl = `${baseUrl}/api/model_gfs_stilt/model_gfs_stilt/get_stilt_data/?receptor_id=${point.id}&time=${dt.value.format('YYYYMMDDHH00')}`
    const trackerData = await axios.get(trackerUrl);
    const bounds = trackerData.data.bounds;
    const imgUrl = trackerUrl + '&resp_type=png'
    const pointRes = await axios.get(imgUrl, {
      responseType: 'arraybuffer',
    });
    if (trackerLayer.value) {
      map.removeLayer(trackerLayer.value);
    }
    const imageBlob = new Blob([pointRes.data], { type: 'image/png' });
    // 创建图片URL
    const objectUrl = URL.createObjectURL(imageBlob);
    trackerLayer.value = L.imageOverlay(objectUrl, bounds)
    trackerLayer.value.addTo(map);
  }catch (error) {
    console.error('Error fetching point data:', error);
    Message.error(`获取数据失败:${point.name}`);
  }
  
}

async function initMap() {
  map = L.map('map', {
    center: [36.65748, 117.126399],
    zoom: 9,
    zoomDelta: 0.5,
    zoomSnap: 0.5,
    attributionControl: false,
    zoomControl: false,
    contextmenu: true,
    contextmenuWidth: 140,
  })
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; OpenStreetMap contributors'
  }).addTo(map);
  L.control.scale({ metric: true, imperial: false }).addTo(map);
  L.control.zoom({position: 'bottomright'}).addTo(map);

  const dataList = await axios.get(`${baseUrl}/api/model_gfs_stilt/receptor/`)
  
  // 转换为geojson格式
  let d = dataList.data
  let markers = []
  for (let i of d) {
    var marker = L.marker([i.latitude, i.longitude], {
      title: i.name,
      data: i,
    });
    marker.on('click', (e) => {
      choosePoint.value = e.sourceTarget.options.data
      loadPointData();
    });
    markers.push(marker)
  }
  // 添加图标图层
  var layer = L.layerGroup(markers)
  map.addLayer(layer)
}

</script>
<style>
#map { height: 100vh; width: 100vw; }
.tools-wrapper {
  position: fixed;
  top: 10px;
  left: 10px;
  z-index: 1000;
}
</style>
