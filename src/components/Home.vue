<template>
  <div class="tools-wrapper">
    <date-picker v-model:value="dt" format="YYYY-MM-DD HH" show-time placeholder="Select Time" @ok="onOk" />
    <a-button size="small" type="primary" style="margin: 0 10px;" @click="dt = dt.subtract(1, 'hour');loadPointData()">上一小时</a-button>
    <a-button size="small" type="primary" @click="dt = dt.add(1, 'hour');loadPointData()">下一小时</a-button>
  </div>
  <div class="emission-table-wrapper">
    <table class="emission-table">
      <thead>
        <tr>
          <th>排名</th>
          <th>污染源名称</th>
          <th>贡献值</th>
          <th>贡献占比 (%)</th>
          <th>敏感性分析贡献值</th>
          <th>敏感性分析贡献占比 (%)</th>
          <th>距离 (m)</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(item, idx) in sortedEmissionTableData" :key="item.pollutant_source.id">
          <td>{{ idx + 1 }}</td>
          <td :title="item.pollutant_source.name">{{ item.pollutant_source.name.slice(0, 25) }}</td>
          <td>{{ item.emis_value }}</td>
          <td>{{ item.emis_rate }}</td>
          <td>{{ item.emis_value_s }}</td>
          <td>{{ item.emis_rate_s }}</td>
          <td>{{ item.dist }}</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div id="map"></div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
// 污染源表格数据
const emissionTableData = ref([])
// 排序后的表格数据（按pm10降序）
const sortedEmissionTableData = computed(() => {
  return [...emissionTableData.value].sort((a, b) => b.emis_value - a.emis_value);
});
import axios from 'axios'
import { DatePicker, Button as AButton, message as Message } from 'ant-design-vue';
import dayjs from 'dayjs';

let baseUrl = process.env.NODE_ENV === 'production' ?  "" : "http://127.0.0.1:8000"
console.log(process.env.NODE_ENV, baseUrl);

const dt = ref(dayjs('2025-06-23 06:00:00'))
let map = ref(null)
const choosePoint = ref(null)
const trackerLayer = ref(null)

onMounted(()=>{
  initMap()
})

function onOk(value) {
  loadPointData();
  loadEmissionData();
}

const loadPointData = async () => {
  const point = choosePoint.value
  if (!point) {
    return;
  }
  try {
    const trackerUrl = `${baseUrl}/api/model_wrf_stilt/model_wrf_stilt/get_stilt_data/?receptor_id=${point.id}&time=${dt.value.format('YYYYMMDDHH00')}`
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

let emissionMarkers = [];
function loadEmissionData() {
  const point = choosePoint.value
  if (!point) {
    return;
  }
  const emissionUrl = `${baseUrl}/api/model_wrf_stilt/model_wrf_stilt/get_emission_contribution/?receptor_id=${point.id}&time=${dt.value.format('YYYY-MM-DD HH:00:00')}`
  axios.get(emissionUrl)
    .then(response => {
      // 清除旧的排放源marker
      if (emissionMarkers.length > 0) {
        for (let m of emissionMarkers) {
          map.removeLayer(m);
        }
        emissionMarkers = [];
      }
      const emissionData = response.data;
      if (!Array.isArray(emissionData)) {
        emissionTableData.value = [];
        return;
      }
      emissionTableData.value = emissionData;
      for (let item of emissionData) {
        const src = item.pollutant_source;
        if (!src) continue;
        // 创建带有PM10值的自定义divIcon
        // 先生成内容，计算宽度
        const tempDiv = document.createElement('div');
        tempDiv.className = 'emission-marker-content';
        tempDiv.style.position = 'absolute';
        tempDiv.style.visibility = 'hidden';
        tempDiv.innerHTML = `
          <span class=\"emission-marker-title\">${src.name}</span>
          <span class=\"emission-marker-pm10\">${item.emis_rate}</span>
          <div class=\"emission-marker-arrow\"></div>
        `;
        document.body.appendChild(tempDiv);
        const width = tempDiv.offsetWidth;
        const height = tempDiv.offsetHeight;
        document.body.removeChild(tempDiv);
        // iconAnchor横坐标设为宽度一半，纵坐标设为高度，保证箭头尖端居中对准坐标点
        const divIcon = L.divIcon({
          className: 'emission-marker',
          html: `
            <div class=\"emission-marker-content\">
              <span class=\"emission-marker-title\">${src.name}</span>
              <span class=\"emission-marker-pm10\">${item.emis_rate}</span>
              <div class=\"emission-marker-arrow\"></div>
            </div>
          `,
          iconAnchor: [Math.round(width/2), height],
        });
        const marker = L.marker([src.latitude, src.longitude], {
          icon: divIcon
        });
        marker.addTo(map);
        emissionMarkers.push(marker);
      }
    })
    .catch(error => {
      console.error('Error fetching emission data:', error);
      Message.error(`获取排放数据失败:${point.name}`);
    });
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

  const dataList = await axios.get(`${baseUrl}/api/model_wrf_stilt/receptor/`)
  
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
      loadEmissionData();
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
 /* 污染源表格样式 */
 .emission-table-wrapper {
   position: fixed;
   top: 60px;
   left: 10px;
   z-index: 1100;
   background: rgba(255,255,255,0.98);
   border-radius: 8px;
   box-shadow: 0 2px 8px #0001;
   padding: 10px 16px 10px 10px;
   min-width: 220px;
   max-height: 60vh;
   overflow-y: auto;
 }
 .emission-table {
   width: 100%;
   border-collapse: collapse;
   font-size: 13px;
 }
 .emission-table th, .emission-table td {
   border-bottom: 1px solid #eee;
   padding: 4px 8px;
   text-align: left;
 }
 .emission-table th {
   background: #f7f7f7;
   font-weight: bold;
 }
 .emission-table tr:last-child td {
   border-bottom: none;
 }
/* emission marker 样式 */
.emission-marker-content {
  display: inline-flex;
  align-items: center;
  background: rgba(255,255,255,0.95);
  border: 1px solid #888;
  border-radius: 6px;
  padding: 2px 10px 6px 10px;
  min-width: 80px;
  box-shadow: 0 2px 6px #0002;
  font-size: 13px;
  position: relative;
  white-space: nowrap;
}
.emission-marker-title {
  font-weight: bold;
  margin-right: 8px;
  color: #333;
}
.emission-marker-pm10 {
  color: #d2691e;
}
.emission-marker-arrow {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  bottom: -10px;
  width: 0;
  height: 0;
  border-left: 8px solid transparent;
  border-right: 8px solid transparent;
  border-top: 10px solid #888;
  z-index: 2;
}
.emission-marker-arrow::after {
  content: '';
  position: absolute;
  left: -7px;
  top: -9px;
  width: 0;
  height: 0;
  border-left: 7px solid transparent;
  border-right: 7px solid transparent;
  border-top: 9px solid #fff;
  z-index: 3;
}
</style>
