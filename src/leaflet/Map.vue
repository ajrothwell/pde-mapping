<template>
  <div :class="this.mapContainerClass">
    <!-- the leaflet map -->
    <div class="map"
         ref="map"
         id="map"
    />
      <div>
        <slot />
      </div>
    </div>
  </div>
</template>

<script>
  import {
    Map,
    LatLng,
    LatLngBounds
  } from 'leaflet';
  import bindEvents from './util/bind-events';

  export default {
    props: [
      'center',
      'zoom',
      'attributionPosition',
      'zoomControlPosition',
      'minZoom',
      'maxZoom',
      'drawControl',
    ],
    mounted() {
      // console.log('Map.vue mounted is running, this.$route:', this.$route, 'Object.keys(this.$route.query):', Object.keys(this.$route.query));
      const map = this.$leafletElement = this.createLeafletElement();

      // move zoom control
      map.attributionControl.setPosition(this.$props.attributionPosition);
      map.zoomControl.setPosition(this.$props.zoomControlPosition);

      // put in state
      this.$store.commit('setMap', { map });

      this.setMapView(this.center);

      this.$nextTick(() => {
        map.attributionControl.setPrefix('<a target="_blank" href="//www.phila.gov/it/aboutus/units/Pages/GISServicesGroup.aspx">City of Philadelphia | CityGeo</a>');
      })

      // signal children to mount
      for (let child of this.$children) {
        // REVIEW it seems weird to pass children their own props. trying to
        // remember why this was necessary... binding issue?
        child.parentMounted(this, child.$props);
      }

      // TODO warn if trying to bind an event that doesn't exist
      bindEvents(this, this.$leafletElement, this._events);
      if (this.$config.map.clickToIdentifyFeatures) {
        map.on('click', this.identifyFeatures);
      }

      const editableLayers = this.$store.state.editableLayers;
      if (editableLayers !== null) {
        map.addLayer(editableLayers);
      }

      map.on('draw:drawstart', () => {
        if(this.$store.state.editableLayers !== null){
          this.$store.state.editableLayers.clearLayers();
        }
        this.drawStartChange();
      });
      map.on('draw:drawstop', this.drawStopChange);
      map.on('draw:created', this.drawShapeChange);
      map.on('draw:created', (e) => {editableLayers.addLayer(e.layer);});
    },

    watch: {
      center(nextCenter) {
        // console.log('watch center is firing, nextCenter:', nextCenter);
        if (typeof nextCenter[0] == 'number') {
          this.setMapView(nextCenter);
        }
      },
      zoom(nextZoom) {
        console.log('Map.vue watch zoom is firing')
        if (!nextZoom) return;
        this.$leafletElement.setZoom(nextZoom);
        this.$store.commit('setMapZoom', nextZoom);
      },
      mapBounds(nextBounds) {
        // console.log('watch nextBounds is firing, nextBounds:', nextBounds, 'this.$leafletElement:', this.$leafletElement);
        this.setMapBounds(nextBounds)
      },
      fullScreenMapEnabled() {
        // console.log('Map.vue fullScreenMapEnabled watch is firing');
        this.$leafletElement.invalidateSize();
      },
      webMapDisplayedLayers(nextWebMapDisplayedLayers) {
        let intersectingLayers = [];
        for (let feature of this.intersectingFeatures) {
          intersectingLayers.push(feature.feature.layerName);
        }
        // console.log('map.vue watch nextWebMapDisplayedLayers:', nextWebMapDisplayedLayers, 'intersectingLayers:', intersectingLayers);
        for (let layer of intersectingLayers) {
          if (!nextWebMapDisplayedLayers.includes(layer)) {
            this.$store.commit('setIntersectingFeatures', []);
            return;
          }
        }
      },
    },
    computed: {
      drawStart() {
        return this.$store.state.drawStart;
      },
      drawShape() {
        return this.$store.state.drawShape;
      },
      cyclomediaActive() {
        return this.$store.state.cyclomedia.active;
      },
      mapContainerClass() {
        if (this.cyclomediaActive && this.$config.map.containerClassWCyclo) {
          return this.$config.map.containerClassWCyclo;
        } else if (this.$config.map.containerClass) {
          return this.$config.map.containerClass
        } else {
          return 'map-container'
        }
      },
      fullScreenMapEnabled() {
        return this.$store.state.fullScreenMapEnabled;
      },
      mapBounds() {
        return this.$store.state.map.bounds;
      },
      webMapDisplayedLayers() {
        return this.$store.state.map.webMapDisplayedLayers;
      },
      intersectingFeatures() {
        return this.$store.state.map.intersectingFeatures;
      }
    },
    methods: {
      drawShapeChange(shape) {
        // console.log("drawShapeChange:", shape.layer);
        this.$store.commit('setDrawShape', shape.layer);
        this.$store.commit('setShapeSearchInput', shape.layer._latlngs[0])
      },
      drawStartChange() {
        // console.log("DrawStart is working");
        this.$store.commit('setDrawStart', 'start');
      },
      drawStopChange() {
        // console.log("DrawStart is working");
        this.$store.commit('setDrawStart', null);
      },

      createLeafletElement() {
        const { zoomControlPosition, attributionPosition, ...options } = this.$props;
        return new Map(this.$refs.map, options);
      },
      childDidMount(child) {
        child.addTo(this.$leafletElement);
      },
      setMapView(xy = [], zoom = this.zoom) {
        // console.log('setMapView is running, xy:', xy);
        if (xy.length === 0) return;
        const [ lng, lat ] = xy;
        const latLng = new LatLng(lat, lng);

        // we used "setView" here because when you refreshed the app with an address in the url,
        // "panTo" was getting stepped on by "setZoom" and it was not happening
        this.$nextTick(() => {
          // console.log('Map.vue this.$leafletElement.setView is running, latLng:', latLng);
          this.$leafletElement.setView(latLng, zoom);
        })
        this.$leafletElement.setView(latLng, zoom);
      },
      setMapBounds(bounds) {
        // console.log('setMapBounds is running, bounds:', bounds, bounds.isValid(), 'this.$leafletElement:', this.$leafletElement);
        if (bounds._northEast) {
          // console.log('MAP.VUE SETMAPBOUNDS IS RUNNING:', bounds._northEast.lat, bounds._northEast.lng, bounds._southWest.lat, bounds._southWest.lng);
          // const corner1 = L.latLng(bounds._northEast.lat, bounds._northEast.lng);
          // const corner2 = L.latLng(bounds._southWest.lat, bounds._southWest.lng);
          // const bounds2 = L.latLngBounds(corner2, corner1);
          // console.log('bounds2:', bounds2, bounds2.isValid())
          // this.$leafletElement.fitBounds(bounds);
          const map = this.$leafletElement;
          // console.log('bounds:', bounds, 'this.$leafletElement:', this.$leafletElement, 'map:', map);
          // map.fitBounds(bounds2);
          map.fitBounds([[bounds._northEast.lat, bounds._northEast.lng],[bounds._southWest.lat, bounds._southWest.lng]]);
        }
      },

      // this is used when the click should identify features
      identifyFeatures(e) {
        const map = this.$leafletElement
        const clickBounds = L.latLngBounds(e.latlng, e.latlng);
        // console.log('clickHandler in Map is starting, e:', e, 'clickBounds:', clickBounds);
        // console.log('map._layers', map._layers);
        let intersectingFeatures = [];
        let geometry;
        for (let layer in map._layers) {
          var overlay = map._layers[layer];
          // console.log('layer:', layer, 'overlay:', overlay);
          if (overlay._layers) {
            for (let oLayer in overlay._layers) {
              const feature = overlay._layers[oLayer];
              // console.log('feature:', feature);
              if (feature.feature) {
                geometry = feature.feature.geometry.type;
                // console.log('clickHandler LAYER:', layer, 'FEATURE:', feature, 'GEOMETRY:', geometry);
                let bounds;
                if (geometry === 'Polygon' || geometry === 'MultiPolygon') {
                  // console.log('polygon or multipolygon');
                  if (feature.contains(e.latlng)) {
                    // console.log('about to run checkForDuplicates')
                    this.checkForDuplicates(layer, feature, intersectingFeatures);
                  }
                }
                else if (geometry === 'LineString') {
                  // console.log('Line');
                  bounds = feature.getBounds();
                  if (bounds && clickBounds.intersects(bounds)) {
                    this.checkForDuplicates(layer, feature, intersectingFeatures);
                  }
                } else if (geometry === 'Point') {
                  // console.log('Point');
                  bounds = L.latLngBounds(feature._latlng, feature._latlng);
                  if (bounds && clickBounds.intersects(bounds)) {
                    this.checkForDuplicates(layer, feature, intersectingFeatures);
                  }
                }
              }
            }
          }
        }
        this.$store.commit('setPopupCoords', e.latlng);
        this.$store.commit('setIntersectingFeatures', intersectingFeatures);
      },
      checkForDuplicates(layer, feature, intersectingFeatures) {
        // console.log('checkForDuplicates is running, layer:', layer, 'feature:', feature);
        let ids = []
        for (let i = 0; i < intersectingFeatures.length; i++) {
          ids[i] = layer + '_' + intersectingFeatures[i].feature.id;
        }
        // console.log('layer:', layer, 'feature.feature.id:', feature.feature.id);
        if (!ids.includes(layer + '_' + feature.feature.id)) {
          // console.log('checkForDuplicates going to push to intersectingFeatures:', layer, feature.feature.id);
          intersectingFeatures.push(feature);
        }
      }
    }
  };
</script>

<style>
  .map-container {
    height: 100%;
  }

  .map-container-350 {
    height: 100%;
  }

  .map-container-type2 {
    height: calc(100vh - 109px);
  }

  .map {
    height: 100%;
  }

  @media (max-width: 749px) {
    .map-container {
      height: 250px;
    }

    .map-container-350 {
      height: 350px;
    }
  }

  /* @media screen and (max-width: 40em) { */
  @media screen and (max-width: 750px) {
    .map-container-type2 {
      height: calc(100vh - 141px);
    }
  }
</style>
