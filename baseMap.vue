<template>
  <div style="width: 100%;height: 100%;">
    <div id="map" ref="_this.map"></div>
    <div id="popup" class="ol-popup" ref="popup">
      <a href="#" id="popup-closer" class="ol-popup-closer" ref="popupCloser"></a>
      <div id="popup-content" ref="popupContent">
      </div>
    </div>
    <div class="mouse-position" ref="mousePosition"></div>
    <div id="mapZoom" class="map-zoom"></div>
    <div id="mapScale" class="map-scale"></div>
    <div id="overviewMap" class="map-overview"></div>
  </div>
</template>

<script>
  import { $on, $off, $once, $emit } from '@/utils/gogocodeTransfer'
  import RootMap from '@/utils/map/rootMap'
  import CommonsMap from '@/utils/map/commons'
  import { Vector as VectorSource } from 'ol/source'
  import {
    Group as LayerGroup,
    Tile as TileLayer,
    Vector as VectorLayer,
    Image as ImageLayer,
    VectorTile as LayerVectorTile,
  } from 'ol/layer'
  import {
    Circle as CircleStyle,
    Fill,
    Stroke,
    Style,
    Text,
    Icon,
  } from 'ol/style'
  import Feature from 'ol/Feature'
  import GeoJSON from 'ol/format/GeoJSON'
  import { getArea, getLength, getDistance } from 'ol/sphere'
  import { LineString, Polygon, Point } from 'ol/geom'
  import { fromLonLat, toLonLat, transform } from 'ol/proj'
  import { Draw, Snap } from "ol/interaction";
  import { unByKey } from "ol/Observable";
  import Overlay from "ol/Overlay";
import * as olProj from 'ol/proj'

  // import * as olProj from 'ol/proj'

  import PorpContent from '@/utils/map/porpContent'
  //import { queryArea } from '../../api/common/system'
  //import { ptglRoad, ptglPt, ptglzc } from '@/api/road/geojson'
  import bus from "@/utils/bus";

  export default {
    name: 'baseMap',
    components: {},
    props: {},
    data() {
      return {
        map: null,
        mapView: null,
        areaArray: [],
        countyArray: [],
        selCity: '',
        selCounty: '',
        draw: null,
        overlay: null,
        measureTooltipElement: null,
        measureTooltip: null,
        // editMeasureImg: require("@/assets/images/deleteMeaureImg.png"),

        // overlay: null,
        // content: null,
      }
    },
    created() { },
    mounted() {
      let _this = this
      let { Center, Zoom } = BASE_CONFIG.MapParams
      let mapOptions = {
        target: 'map',
        mouseDoc: this.$refs.mousePosition,
        zoom: Zoom,
        minZoom: 6,
        center: Center,
        zoomTarget: 'mapZoom',
        scaleTarget: 'mapScale',
        mapOverview: 'overviewMap',
        pop: {
          container: this.$refs.popup,
          closer: this.$refs.popupCloser,
          content: this.$refs.popupContent,
        },
      }
      // this.$nextTick(() => {
      this.mapView = new RootMap(mapOptions)
      this.map = this.mapView.map
      window.rootMap = this.mapView.map
      
      // _this.points = []
      $emit(this, 'setMap', this.map)

      this.map.on('click', function (event) {
        // if( _this.points.length > 1 ) {
        //   console.log(111, JSON.stringify(_this.points))
        //   _this.points = []
        // }else {
        //   _this.points.push(toLonLat(event.coordinate))
        // }

        let feature = _this.map.forEachFeatureAtPixel(
          event.pixel,
          function (feature, layerVetor) {
            // getProperties 是获取feat属性或自定义属性的函数，不是feat对象
            // return feature.getProperties().features
            return feature
          }
        )
       // console.log(feature)
        // if (!feature) {
        // 最好判断下是否有自定义ID
        if (!feature) {
          return
        }
      
        let id = feature.getId();
        if (id) {
          // if (feature[0].getId().indexOf("measure_layer_") > -1) {
          if (id.indexOf("measure_layer_") > -1) {
            //删除绘制方法
            _this.handleDeleteMeasure(id);
          }
          return
        }

        let featurePro = feature.getProperties().features
        let feat = featurePro[0];
       
        let dataType = feat.get('dataType')
        if (featurePro.length > 1) {
          $emit(_this, 'showAssetDialog', featurePro)
          return
        }




        let fId = feat.getId()

        _this.map.showPop(
          PorpContent.porpContent(dataType, feat),
          event.coordinate
          // CommonsMap.getCenterByFeat(feat.getGeometry().getExtent())
        )


        if (dataType === 'ptgl') {
          CommonsMap.removeLayerByName(_this.map, 'ptgl_road_layer')
          let props = feat.getProperties()
          // ptglzc().then((res) => {
          let style = '{"borderNum": 4, "borderColor": "green"}'
          let geojson = {
            type: 'FeatureCollection',
            features: [],
          }


          console.log(event)

          if (geojson['features'].length > 0) {
            let layer = CommonsMap.createVectorLayerByGeoJson(
              'ptgl_road_layer',
              geojson,
              style
            )
            _this.map.addLayer(layer)
          }
          // })
          return
        }
      })
      // this.queryArea()

      this.changeMeasureType();
    },
    methods: {
      showLayerChange(key) {
        this.mapView.switchBaseLayer(key)
      },
      // queryArea() {
      //   queryArea().then((res) => {
      //     this.areaArray = res
      //   })
      // },
      // areaChange(data, type) {
      //   if (type === 'city') {
      //     // 显示下辖区县
      //     this.areaArray.forEach((c) => {
      //       if (c.code === data) {
      //         this.countyArray = c.child
      //         this.selCounty = ''
      //         // 地图定位
      //         CommonsMap.positionByJW(this.map, [c.longitude, c.latitude], 13)
      //         return
      //       }
      //     })
      //   } else if (type === 'county') {
      //     this.countyArray.forEach((c) => {
      //       if (c.code === data) {
      //         // 地图定位
      //         CommonsMap.positionByJW(this.map, [c.longitude, c.latitude], 15)
      //         return
      //       }
      //     })
      //   }
      // },
      // showDetails() {
      //   alert(1)
      //   this.$emit("showDetails", 'showDetails')
      // },
      // showCard() {
      //   alert('showCard', 'showCard')
      // }


      changeMeasureType() {
        let _this = this;
        bus.on("changeMeasureType", function (type) {
          _this.map.removeInteraction(_this.draw);
          // 新增一个图层
          let source = new VectorSource();
          let layerName = "measure_layer_" + type + "_" + new Date().getTime();
          let vector = new VectorLayer({
            name: layerName,
            source: source,
            zIndex: 999,
            style: new Style({
              stroke: new Stroke({
                color: "#ffcc33",
                width: 3
              }),
              image: new CircleStyle({
                radius: 3,
                fill: new Fill({
                  color: "#ffcc33"
                })
              })
            })
          });
          _this.map.addLayer(vector);
          // 创建画操作
          _this.draw = new Draw({
            source: source,
            type: type,
            style: new Style({
              fill: new Fill({
                color: 'rgba(255, 255, 255, 0.2)'
              }),
              stroke: new Stroke({
                color: 'rgba(0, 0, 0, 0.5)',
                lineDash: [10, 10],
                width: 2
              }),
              image: new CircleStyle({
                radius: 5,
                stroke: new Stroke({
                  color: 'rgba(0, 0, 0, 0.7)'
                }),
                fill: new Fill({
                  color: 'rgba(255, 255, 255, 0.2)'
                })
              })
            })
          });
          _this.map.addInteraction(_this.draw);
          _this.createMeasureTooltip();
          let listener;
          // 开始画
          _this.draw.on(
            "drawstart",
            function (evt) {
              let pixel = evt.target.downPx_;
              let coor = _this.map.getCoordinateFromPixel(pixel);
              if (
                coor[0] >= 20037508.342789244 ||
                coor[0] <= -20037508.342789244 ||
                coor[1] >= 20037508.342789244 ||
                coor[1] <= -20037508.342789244
              ) {
                _this.$message.error("请选择正确的位置绘制");
                _this.map.removeInteraction(_this.draw);
                _this.map.removeInteraction(_this.snap);
                return;
              }
              _this.sketch = evt.feature;
              evt.feature.setId(layerName);
              let tooltipCoord = evt.coordinate;
              listener = _this.sketch.getGeometry().on("change", function (evt) {
                let geom = evt.target;
                let output;
                if (geom instanceof Polygon) {
                  output = CommonsMap.formatArea(geom);
                  tooltipCoord = geom.getInteriorPoint().getCoordinates();
                } else if (geom instanceof LineString) {
                  output = CommonsMap.formatLength(geom);
                  tooltipCoord = geom.getLastCoordinate();
                }
                _this.measureTooltipElement.innerHTML = output;
                _this.measureTooltipElement.style.color = "#000";
                _this.measureTooltipElement.style.fontWeight = "bold";
                _this.measureTooltipElement.style.backgroundColor = "rgba(255,255,255,.7)";
                _this.measureTooltip.setPosition(tooltipCoord);
              });
            },
            _this
          );
          // 画结束
          _this.draw.on(
            "drawend",
            function (e) {
              let sketchCoords_ = e.target.sketchCoords_;
              if (type === "Polygon") {
                sketchCoords_ = e.target.sketchCoords_[0];
              }
              let flags = sketchCoords_.map(
                c =>
                  c[0] >= 20037508.342789244 ||
                  c[0] <= -20037508.342789244 ||
                  c[1] >= 20037508.342789244 ||
                  c[1] <= -20037508.342789244
              );
              if (flags.includes(true)) {
                _this.$message.error("请选择正确的位置绘制");
                _this.map.removeInteraction(_this.draw);
                _this.map.removeInteraction(_this.snap);

                if (CommonsMap.getLayerByName(_this.map, layerName)) {
                  _this.map.removeLayer(CommonsMap.getLayerByName(_this.map, layerName));
                }
                _this.measureTooltipElement.innerHTML = null;
                return;
              }
              let geometry = new GeoJSON().writeFeature(e.feature);
              let geoJson = JSON.parse(geometry).geometry;
              let coor;
              if (geoJson.type === "Polygon") {
                coor = geoJson.coordinates[0][geoJson.coordinates[0].length - 1];
              } else {
                coor = geoJson.coordinates[geoJson.coordinates.length - 1];
              }
              _this.handleDrawColse(coor, layerName, type);
              _this.map.removeInteraction(_this.draw);
              _this.measureTooltipElement.className = "tooltip tooltip-static-" + layerName;
              _this.measureTooltip.setOffset([0, -7]);
              _this.sketch = null;
              _this.measureTooltipElement = null;
              _this.createMeasureTooltip();
              unByKey(listener);
            },
            _this
          );
        });
      },

      // 创建新的度量工具提示
      createMeasureTooltip() {
        if (this.measureTooltipElement) {
          this.measureTooltipElement.parentNode.removeChild(
            this.measureTooltipElement
          );
        }
        this.measureTooltipElement = document.createElement("div");
        this.measureTooltipElement.className = "tooltip tooltip-measure";
        this.measureTooltip = new Overlay({
          element: this.measureTooltipElement,
          offset: [0, -15],
          positioning: "bottom-center"
        });
        this.map.addOverlay(this.measureTooltip);
      },



      /**
           * 绘制关闭按钮
           */
      handleDrawColse(coor, layerName, type, flag) {
        console.log(coor, layerName, type, flag);

        if (flag) {
          coor = fromLonLat(coor);
        }
        let anchor = new Feature({
          geometry: new Point(coor)
        });
        anchor.setId(layerName);
        let style;
        if (type === "Point") {
          style = new Style({
            image: new CircleStyle({
              radius: 3,
              fill: new Fill({
                color: "#ffcc33"
              })
            })
          });
        } else {
          style = new Style({
            image: new Icon({
              src: this.editMeasureImg,
              anchor: [0.5, 0.5],
              rotateWithView: true,
              scale: 0.6
            })
          });
        }
        anchor.setStyle(style);
        let source = new VectorSource();
        let vector = new VectorLayer({
          name: layerName + "-close",
          source: source,
          style: style,
          zIndex: 999
        });
        source.addFeature(anchor);
        this.map.addLayer(vector);
      },


      /**
       * 删除测距信息
       */
      handleDeleteMeasure(id) {
        console.log("删除测距", id)
        if (CommonsMap.getLayerByName(this.map, id)) {
          console.log(CommonsMap.getLayerByName(this.map, id));
          this.map.removeLayer(CommonsMap.getLayerByName(this.map, id));
        }
        if (CommonsMap.getLayerByName(this.map, id + "-close")) {
          this.map.removeLayer(CommonsMap.getLayerByName(this.map, id + "-close"));
        }
        let textDom = document.getElementsByClassName(
          "tooltip tooltip-static-" + id
        );
        console.log(textDom, "123");
        if (textDom[0]) {
          textDom[0].remove();
        }
      },

    },
    watch: {},
    emits: ['setMap', 'showAssetDialog'],
  }
</script>

<style></style>

<style lang="scss">
  /* eslint-disable */
  @import '@/styles/map/olMap.scss';
  @import "@/styles/map/commons.scss";

  .map-console {
    position: absolute;
    top: 5px;
    left: 10px;
    width: 300px;
    background-color: rgb(220, 220, 247);
    border-radius: 5px;
    filter: progid:DXImageTransform.Microsoft.Shadow(color=#909090, direction=120, strength=4);
    -moz-box-shadow: 2px 2px 10px #909090;
    -webkit-box-shadow: 2px 2px 10px #909090;
    box-shadow: 2px 2px 10px #909090;
  }

  .map-console2 {
    @extend .map-console;
    top: 100px;
    background-color: rgb(220, 220, 247);
  }
</style>