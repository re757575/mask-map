<template>
  <div id="app">
    <div id="mask" :class="!isLoading ? 'hide' : ''"></div>
    <div class="row no-gutters">
      <div class="col-sm-3">
        <div class="toolbox">
          <div class="sticky-top bg-white shadow-sm p-2">
            <div class="form-group d-flex">
              <label for="cityName" class="mr-2 col-form-label text-right"
                >縣市</label
              >
              <div class="flex-fill">
                <select
                  id="cityName"
                  class="form-control"
                  v-model="selectedCity"
                  @change="
                    selectedArea = '';
                    selectedStore = '';
                  "
                >
                  <option value>-- 請選擇 --</option>
                  <option
                    v-for="city in cityList"
                    :value="city.CityName"
                    :key="city.CityName"
                    >{{ city.CityName }}</option
                  >
                </select>
              </div>
            </div>
            <div class="form-group d-flex">
              <label for="area" class="mr-2 col-form-label text-right"
                >地區</label
              >
              <div class="flex-fill">
                <select
                  id="area"
                  class="form-control"
                  v-model="selectedArea"
                  @change="selectedStore = ''"
                >
                  <option value>-- 請選擇 --</option>
                  <option
                    v-for="area in areaList"
                    :value="area.AreaName"
                    :key="area.ZipCode"
                    >{{ area.AreaName }}</option
                  >
                </select>
              </div>
            </div>
            <div class="form-group d-flex">
              <label for="area" class="mr-2 col-form-label text-right"
                >有口罩</label
              >
              <div class="flex-fill">
                <input
                  type="checkbox"
                  class="form-control"
                  v-model="filterHas"
                />
              </div>
            </div>
            <div class="form-group d-flex">
              <label for="area" class="mr-2 col-form-label text-right"
                >搜尋</label
              >
              <div class="flex-fill">
                <input type="text" class="form-control" v-model="keywords" />
              </div>
            </div>
            <p class="mb-0 small text-muted text-right">
              請先選擇區域查看（綠色表示還有口罩）
            </p>
          </div>

          <ul class="list-group">
            <template v-for="pharmacie in pharmaciesFilter">
              <a
                v-bind:class="{
                  highlight:
                    pharmacie.properties.mask_adult ||
                    pharmacie.properties.mask_child,
                  empty:
                    !pharmacie.properties.mask_adult &&
                    !pharmacie.properties.mask_child
                }"
                class="list-group-item text-left"
                :key="pharmacie.properties.id"
                @click="moveTo(pharmacie)"
              >
                <h3>{{ pharmacie.properties.name }}</h3>
                <p class="mb-0">
                  成人口罩：{{ pharmacie.properties.mask_adult }} | 兒童口罩：{{
                    pharmacie.properties.mask_child
                  }}
                </p>
                <p class="mb-0">
                  地址：
                  <a
                    :href="
                      `https://www.google.com.tw/maps/place/${pharmacie.properties.name}/@${pharmacie.geometry.coordinates[1]},${pharmacie.geometry.coordinates[0]}`
                    "
                    target="_blank"
                    title="Google Map"
                    >{{ pharmacie.properties.address }}</a
                  >
                </p>
              </a>
            </template>
          </ul>
        </div>
      </div>

      <div class="col-sm-9">
        <div id="map"></div>
      </div>
    </div>
  </div>
</template>

<script>
// https://quip.com/vdqYAiFHHkaV

// leaflet
import L from "leaflet";
import "leaflet/dist/leaflet.css";
// leaflet.markercluster
import "leaflet.markercluster/dist/MarkerCluster.css";
import "leaflet.markercluster/dist/MarkerCluster.Default.css";
import "leaflet.markercluster";

// 解决使用 markercluster 後,預設 maker 無法顯示
import icon from "leaflet/dist/images/marker-icon.png";
import iconShadow from "leaflet/dist/images/marker-shadow.png";
let DefaultIcon = L.icon({
  iconUrl: icon,
  shadowUrl: iconShadow
});
L.Marker.prototype.options.icon = DefaultIcon;

// 台灣縣市
import cityList from "./assets/tw-city.json";

var greenIcon = new L.Icon({
  iconUrl:
    "https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png",
  shadowUrl:
    "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41]
});
var redIcon = new L.Icon({
  iconUrl:
    "https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-red.png",
  shadowUrl:
    "https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png",
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41]
});

export default {
  name: "App",
  data() {
    return {
      map: null,
      isLoading: false,
      cityList,
      pharmacies: [],
      tmparry: [],
      selectedCity: "",
      selectedArea: "",
      selectedStore: "",
      keywords: "",
      filterHas: true,
      postion: [23.581610299925213, 120.41440780848676]
    };
  },
  computed: {
    areaList: function() {
      const cityFilted = cityList.filter(
        city => city.CityName === this.selectedCity
      );

      if (cityFilted.length) {
        return cityFilted[0].AreaList;
      }
      return [];
    },
    pharmaciesFilter: function() {
      let list = this.pharmacies;

      if (this.keywords) {
        list = list.filter(pharmacie => {
          const { name } = pharmacie.properties;

          return name.indexOf(this.keywords) > -1;
        });
      }

      if (this.filterHas) {
        list = list.filter(pharmacie => {
          const { mask_adult, mask_child } = pharmacie.properties;

          return mask_adult > 0 && mask_child > 0;
        });
      }

      return list;
    }
  },
  watch: {
    selectedCity: function() {
      this.updateData();
    },
    selectedArea: function() {
      this.updateData();
    },
    pharmacies: function(list) {
      // if (!this.selectedArea) return false;

      const markers = L.markerClusterGroup();

      let tmparry = [];
      list.forEach(p => {
        const { geometry, properties } = p;
        const {
          mask_adult,
          mask_child,
          name,
          address,
          phone,
          available,
          updated
        } = properties;
        const { coordinates } = geometry;
        const [lng, lat] = coordinates;

        const icon = mask_adult === 0 ? redIcon : greenIcon;

        const marker = L.marker([lat, lng], { icon, title: name })
          // .addTo(this.map)
          .bindPopup(
            `
            <h3>${name}</h3>
            <p>成人口罩：${mask_adult}</p>
            <p>兒童口罩：${mask_child}</p>
            <p>電話：${phone}</p>
            <p>地址：${address}</p>
            <p>營業：${available}</p>
            <p>更新時間：${updated}</p>
          `
          )
          // .openPopup()
          .on("click", function(e) {
            console.log(e.latlng);
          });

        tmparry.push(marker);

        markers.addLayer(marker);
      });

      this.tmparry = tmparry;

      this.map.addLayer(markers);
    },
    postion: function([lat, lng]) {
      let zoom = 5;
      if (this.selectedStore) {
        this.map.eachLayer(layer => {
          layer.openPopup();
        });
        zoom = 18;
      } else if (this.selectedCity && !this.selectedArea) {
        zoom = 10;
      } else if (this.selectedCity && this.selectedArea) {
        zoom = 12;
      }
      this.map.setView([lat, lng], zoom);
    }
  },
  methods: {
    moveTo(pharmacie) {
      const { geometry, properties } = pharmacie;
      const { coordinates } = geometry;
      const { name } = properties;
      const [lng, lat] = coordinates;

      this.postion = [lat, lng];
      this.selectedStore = name;

      const findMarker = this.tmparry.filter(tt => {
        return tt.options.title == name;
      });

      setTimeout(() => {
        if (findMarker[0]) {
          findMarker[0].openPopup();
        }
      }, 0);
      // console.log(this.map.getCenter());
    },

    updateData(isInit) {
      const url =
        "https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json";

      this.isLoading = true;
      this.$http.get(url).then(response => {
        this.pharmacies = response.data.features.filter(pharmacie => {
          const { county, town } = pharmacie.properties;
          if (this.selectedArea) {
            return county === this.selectedCity && town === this.selectedArea;
          }
          if (this.selectedCity) {
            return county === this.selectedCity;
          }
          return true;
        });

        if (!isInit && this.pharmacies.length) {
          const [lng, lat] = this.pharmacies[0].geometry.coordinates;
          this.postion = [lat, lng];
        }

        this.isLoading = false;
      });
    },
    getRandomLatLng() {
      let bounds = this.map.getBounds(),
        southWest = bounds.getSouthWest(),
        northEast = bounds.getNorthEast(),
        lngSpan = northEast.lng - southWest.lng,
        latSpan = northEast.lat - southWest.lat;

      return L.latLng(
        southWest.lat + latSpan * Math.random(),
        southWest.lng + lngSpan * Math.random()
      );
    }
  },
  created() {},
  mounted() {
    // this.selectedCity = "臺中市";
    // this.selectedArea = "龍井區";

    this.map = L.map("map", {
      center: this.postion,
      zoom: 8
    });

    // window.mymap = this.map;

    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution:
        'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
      maxZoom: 18
    }).addTo(this.map);

    this.updateData(true);
  }
};
</script>

<style lang="scss">
@import "bootstrap/scss/bootstrap";

#mask {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100vh;
  background-color: rgba($color: #000000, $alpha: 0.7);
  z-index: 99999;
  &.hide {
    z-index: 0;
    background-color: transparent;
  }
}

#map {
  height: 100vh;
}
.highlight {
  background: #e9ffe3;
}
.empty {
  background: rgb(238, 108, 147);
}
.toolbox {
  height: 100vh;
  overflow-y: auto;
  a {
    cursor: pointer;
  }
}
</style>
