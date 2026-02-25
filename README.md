<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>地图标记（含编辑+删除）</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
      font-family: "Microsoft YaHei", sans-serif;
    }
    body {
      width: 100vw;
      height: 100vh;
      overflow: hidden;
    }
    .header {
      height: 60px;
      background: #2578e6;
      color: white;
      display: flex;
      align-items: center;
      padding: 0 20px;
      gap: 15px;
    }
    .header h1 {
      font-size: 20px;
      font-weight: normal;
    }
    #search {
      flex: 1;
      height: 36px;
      border: none;
      border-radius: 4px;
      padding: 0 12px;
      font-size: 14px;
      outline: none;
    }
    #searchBtn {
      padding: 0 15px;
      height: 36px;
      background: #fff;
      color: #2578e6;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    #mapContainer {
      width: 100%;
      height: calc(100% - 60px);
    }

    /* 编辑面板 */
    .marker-panel {
      position: fixed;
      bottom: 30px;
      left: 50%;
      transform: translateX(-50%);
      width: 90%;
      max-width: 400px;
      background: #fff;
      padding: 15px;
      border-radius: 10px;
      box-shadow: 0 0 10px rgba(0,0,0,0.3);
      display: none;
      z-index: 999;
    }
    .marker-panel input,
    .marker-panel textarea {
      width: 100%;
      margin: 6px 0;
      padding: 8px;
      border: 1px solid #ddd;
      border-radius: 4px;
    }
    .marker-panel textarea {
      height: 70px;
      resize: none;
    }
    .img-preview {
      display: flex;
      gap: 6px;
      flex-wrap: wrap;
      margin: 6px 0;
    }
    .img-preview img {
      width: 60px;
      height: 60px;
      object-fit: cover;
      border-radius: 4px;
      border: 1px solid #ddd;
    }
    .btns {
      display: flex;
      gap: 10px;
      margin-top: 10px;
    }
    .btns button {
      flex: 1;
      padding: 8px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    .btn-save {
      background: #2578e6;
      color: #fff;
    }
    .btn-edit {
      background: #ff9500;
      color: #fff;
    }
    .btn-delete {
      background: #ff3b30;
      color: #fff;
    }
    .btn-cancel {
      background: #f1f1f1;
    }
  </style>
</head>
<body>
  <div class="header">
    <h1>地图标记</h1>
    <input type="text" id="search" placeholder="搜索地点">
    <button id="searchBtn">搜索</button>
  </div>
  <div id="mapContainer"></div>

  <!-- 编辑面板 -->
  <div class="marker-panel" id="markerPanel">
    <input type="text" id="markTitle" placeholder="标记名称">
    <textarea id="markRemark" placeholder="写下留言..."></textarea>
    <input type="file" id="markImg" accept="image/*" multiple>
    <div class="img-preview" id="imgPreview"></div>
    <div class="btns">
      <button class="btn-cancel" onclick="closePanel()">取消</button>
      <button class="btn-save" id="saveBtn" onclick="saveMarker()">保存</button>
      <button class="btn-delete" onclick="deleteMarker()" style="display:none;" id="delBtn">删除</button>
    </div>
  </div>

<script src="https://webapi.amap.com/maps?v=2.0&key=fde660c3b7b51b8bd9e96c97a5e535d3"></script>
<script>
  let map;
  let currentLngLat = null;
  let editMarkerId = null;
  let markerList = JSON.parse(localStorage.getItem('markerList')) || [];
  let imgFiles = [];
  let markerInstances = {}; // 保存地图上的标记实例

  // 初始化地图
  initMap();
  function initMap() {
    map = new AMap.Map('mapContainer', { zoom: 12, center: [116.397428, 39.90923] });

    // 定位
    map.plugin('AMap.Geolocation', function () {
      const geo = new AMap.Geolocation({ enableHighAccuracy: true });
      geo.getCurrentPosition();
      AMap.event.addListener(geo, 'complete', (res) => {
        map.setCenter([res.position.lng, res.position.lat]);
      });
    });

    // 点击地图新增标记
    map.on('click', (e) => {
      editMarkerId = null;
      currentLngLat = [e.lnglat.lng, e.lnglat.lat];
      openAddPanel();
    });

    renderAllMarkers();
  }

  // 打开新增面板
  function openAddPanel() {
    document.getElementById('markTitle').value = '';
    document.getElementById('markRemark').value = '';
    document.getElementById('imgPreview').innerHTML = '';
    document.getElementById('delBtn').style.display = 'none';
    document.getElementById('saveBtn').innerText = '保存';
    imgFiles = [];
    document.getElementById('markerPanel').style.display = 'block';
  }

  // 打开编辑面板
  function openEditPanel(marker) {
    editMarkerId = marker.id;
    currentLngLat = marker.lnglat;
    document.getElementById('markTitle').value = marker.title;
    document.getElementById('markRemark').value = marker.remark || '';
    document.getElementById('delBtn').style.display = 'inline-block';
    document.getElementById('saveBtn').innerText = '修改';
    
    const preview = document.getElementById('imgPreview');
    preview.innerHTML = '';
    if (marker.imgs) {
      marker.imgs.forEach(img => {
        const imgEl = document.createElement('img');
        imgEl.src = img;
        preview.appendChild(imgEl);
      });
    }
    imgFiles = [];
    document.getElementById('markerPanel').style.display = 'block';
  }

  function closePanel() {
    document.getElementById('markerPanel').style.display = 'none';
    currentLngLat = null;
    editMarkerId = null;
  }

  // 图片预览
  document.getElementById('markImg').addEventListener('change', (e) => {
    const files = e.target.files;
    if (!files) return;
    imgFiles = Array.from(files);
    const preview = document.getElementById('imgPreview');
    preview.innerHTML = '';
    imgFiles.forEach(file => {
      const url = URL.createObjectURL(file);
      const img = document.createElement('img');
      img.src = url;
      preview.appendChild(img);
    });
  });

  // 保存 / 修改
  function saveMarker() {
    const title = document.getElementById('markTitle').value.trim();
    const remark = document.getElementById('markRemark').value.trim();
    if (!currentLngLat || !title) {
      alert('请填写标记名称');
      return;
    }

    const imgPromises = imgFiles.map(file => {
      return new Promise(resolve => {
        const reader = new FileReader();
        reader.onload = () => resolve(reader.result);
        reader.readAsDataURL(file);
      });
    });

    Promise.all(imgPromises).then(newImgs => {
      if (editMarkerId !== null) {
        // 编辑模式
        const index = markerList.findIndex(m => m.id === editMarkerId);
        if (index > -1) {
          markerList[index].title = title;
          markerList[index].remark = remark;
          if (newImgs.length > 0) {
            markerList[index].imgs = newImgs;
          }
        }
        refreshMapMarkers();
        alert('修改成功！');
      } else {
        // 新增模式
        const marker = {
          id: Date.now(),
          lnglat: currentLngLat,
          title,
          remark,
          imgs: newImgs
        };
        markerList.push(marker);
        addMarkerToMap(marker);
        alert('保存成功！');
      }
      localStorage.setItem('markerList', JSON.stringify(markerList));
      closePanel();
    });
  }

  // 删除标记
  function deleteMarker() {
    if (editMarkerId === null) return;
    if (!confirm('确定要删除这个标记吗？')) return;
    markerList = markerList.filter(m => m.id !== editMarkerId);
    localStorage.setItem('markerList', JSON.stringify(markerList));
    refreshMapMarkers();
    closePanel();
    alert('删除成功！');
  }

  // 添加单个标记到地图
  function addMarkerToMap(marker) {
    const m = new AMap.Marker({
      position: marker.lnglat,
      map: map
    });
    markerInstances[marker.id] = m;

    m.on('click', () => {
      let info = `<h4>${marker.title}</h4><p>${marker.remark || ''}</p>`;
      if (marker.imgs) {
        marker.imgs.forEach(img => {
          info += `<img src="${img}" style="width:100px;margin:5px;">`;
        });
      }
      info += `<br><button onclick="openEditById(${marker.id})">编辑/删除</button>`;
      
      new AMap.InfoWindow({ content: info }).open(map, marker.lnglat);
    });
  }

  // 从弹窗里点编辑
  function openEditById(id) {
    closePanel();
    const marker = markerList.find(m => m.id === id);
    if (marker) openEditPanel(marker);
  }

  // 刷新所有标记
  function refreshMapMarkers() {
    // 清除旧的
    Object.values(markerInstances).forEach(m => m.remove());
    markerInstances = {};
    // 重新渲染
    renderAllMarkers();
  }

  function renderAllMarkers() {
    markerList.forEach(item => addMarkerToMap(item));
  }

  // 搜索
  const searchBtn = document.getElementById('searchBtn');
  const searchInput = document.getElementById('search');
  function doSearch() {
    const kw = searchInput.value.trim();
    if (!kw) return;
    map.plugin('AMap.PlaceSearch', () => {
      const ps = new AMap.PlaceSearch({ city: '全国' });
      ps.search(kw, (sta, res) => {
        if (sta === 'complete' && res.poiList) {
          const p = res.poiList[0];
          map.setCenter([p.location.lng, p.location.lat]);
        }
      });
    });
  }
  searchBtn.onclick = doSearch;
  searchInput.addEventListener('keydown', e => {
    if (e.key === 'Enter') doSearch();
  });
</script>
</body>
</html>
