<template>
  <div class="fit row">
    <div class="col-3" style="overflow-y: scroll; height: 100%">
      <q-expansion-item
        v-model="showSocialFeatures"
        icon="map"
        label="Social Features"
        caption="Activate the social feature overlays"
      >
        <q-option-group
          v-model="socialFeature"
          :options="socialFeatures"
          color="primary"
          type="radio"
        />

        <Responsive v-if="socialFeature" class="w-full">
          <template #main="{ width }">
            <Chart
              :size="{ width, height: 100 }"
              :data="hist"
              :axis="{
                primary: {
                  type: 'linear',
                  domain: [dataLow, dataHigh],
                  trigger: selectedSquareFeatureValue // NOTE: Workaround for making the VerticalMarker reactive.
                }
              }"
              :margin="{ left: 10, right: 10, top: 10, bottom: 10 }"
            >
              <template #layers>
                <!-- <Grid strokeDasharray="2,2" /> -->
                <Line :dataKeys="['x', 'y']" />
                <!-- TODO: Actually make this correspond to selected square -->
                <!-- NOTE: Seems to be non-reactive, poor library. -->
                <!-- TODO: Cap by datalow/datahigh -->
                <VerticalMarker
                  :value="selectedSquareFeatureValue"
                  label="Square"
                  color="#e76f51"
                  strokeWidth="2"
                />
              </template>
            </Chart>
          </template>
        </Responsive>
      </q-expansion-item>
      <q-expansion-item
        v-model="showComputedFeatures"
        icon="map"
        label="Computed Features"
        caption="Activate the computed feature overlays"
      >
        <q-form @submit="predict" @reset="" class="q-gutter-md q-pa-sm">
          <div class="q-pa-sm q-ma-sm">
            <q-space />
            <div class="text-subtitle1 q-pt-md text-weight-medium"
              >Selection</div
            >

            <div class="bg-blue-grey-1" style="min-height: 64px">
              <q-chip
                removable
                class="text-white"
                :color="
                  buildingIdToVisualize == building ? 'positive' : 'primary'
                "
                @remove="
                  selectedBuildingIds = selectedBuildingIds.filter(
                    v => v != building
                  )
                "
                @click="
                  mapHolder?.map?.setFeatureState(
                    { source: 'buildings', id: building },
                    { highlight: true }
                  );
                  resetHighlight(building);
                  buildingIdToVisualize = building;
                "
                v-for="building in selectedBuildingIds"
                >{{ building }}</q-chip
              >
            </div>
            <q-space />
            <div class="text-subtitle1 q-pt-md text-weight-medium"
              >IDF Visualization</div
            >
            <div class="bg-blue-grey-1">
              <q-img
                :src="`api/idf/?bag_id=${buildingIdToVisualize}&user_id=${user_uuid}`"
                style="margin: auto"
                width="128"
                height="128"
                :ratio="16 / 9"
              >
                <!-- <q-skeleton
                  style="margin: auto"
                  size="256px"
                  square
                  animation="fade"
              /> -->
              </q-img>
            </div>
          </div>

          <q-separator></q-separator>
          <q-select
            clearable
            v-model="retrofit"
            :options="retrofits"
            label="Choose Retrofit Package"
          >
            <template #option="scope">
              <q-item v-bind="scope.itemProps">
                <q-item-section>
                  <q-item-label :title="scope.opt.value">{{
                    scope.opt.label
                  }}</q-item-label>
                  <q-item-label caption>{{
                    JSON.stringify(scope.opt.value)
                      .substring(1)
                      .replace("}", "")
                  }}</q-item-label>
                </q-item-section>
              </q-item>
            </template>
          </q-select>
          <div class="text-subtitle1 q-pt-md text-weight-medium"
            >Heating Retrofits</div
          >
          <q-option-group
            :options="activeHeatingRetrofits"
            type="radio"
            v-model="heatingRetrofit"
            inline
          />

          <div class="text-subtitle1 q-pt-md text-weight-medium"
            >Cooling Retrofits</div
          >
          <q-option-group
            :options="activeCoolingRetrofits"
            type="radio"
            v-model="coolingRetrofit"
            inline
          />
          <!-- <q-radio
            v-for="retro in activeRetrofits.filter(
              r => r.value.type == 'cooling'
            )"
            v-model="retro.value.enabled"
            >{{ retro.label }}</q-radio
          > -->
          <div>
            <q-btn
              label="Predict Retrofit Impact"
              type="submit"
              color="primary"
            />
          </div>
        </q-form>
      </q-expansion-item>
    </div>
    <div class="col-9">
      <mgl-map
        :map-style="style"
        :center="[4.462456, 51.926517]"
        :zoom="zoom"
        @map:zoom="fadeSquare"
      >
        <mgl-custom-control :position="Position.BOTTOM_LEFT">
          <div style="margin: 8px">
            <div class="text-subtitle1 text-weight-bold text-center"
              >Retrofit effect for selected square</div
            >

            <Responsive class="w-full">
              <template #main="{ width }">
                <Chart
                  :key="beforeRetrofitData.heat"
                  :size="{ width: 300, height: 200 }"
                  :data="[beforeRetrofitData, afterRetrofitData]"
                  :margin="{
                    left: 0,
                    top: 20,
                    right: 20,
                    bottom: 0
                  }"
                  :axis="{
                    primary: {
                      type: 'band'
                    },
                    secondary: {
                      domain: [0, 400],
                      type: 'linear',
                      ticks: 8
                    }
                  }"
                >
                  <template #layers>
                    <Grid strokeDasharray="2,2" />
                    <Bar
                      :dataKeys="['name', 'heat']"
                      :barStyle="{ fill: '#ff7a14' }"
                    />
                    <Bar
                      :dataKeys="['name', 'cool']"
                      :barStyle="{ fill: '#0096c7' }"
                    />
                    <Bar
                      :dataKeys="['name', 'elec']"
                      :barStyle="{ fill: '#fffb1d' }"
                    />
                    <Bar
                      :dataKeys="['name', 'gas']"
                      :barStyle="{ fill: '#389b0a' }"
                    />
                    <Bar
                      :dataKeys="['name', 'co2']"
                      :barStyle="{ fill: '#4f4f4e' }"
                    />
                  </template>

                  <template #widgets>
                    <Tooltip
                      color="black"
                      :hide-line="true"
                      :format="'_>10.2f'"
                      :config="{
                        name: { color: 'black', label: '' },
                        heat: { color: '#5b5a5a', label: 'Heating Demand' },
                        cool: { color: '#5b5a5a', label: 'Cooling Demand' },
                        elec: {
                          color: '#5b5a5a',
                          label: 'Electricity Usage (kWh/year)'
                        },
                        gas: { color: '#5b5a5a', label: 'Gas usage (m³/year)' },
                        co2: { color: '#5b5a5a', label: 'CO² Emission' }
                      }"
                    />
                  </template>
                </Chart>
              </template>
            </Responsive>
          </div>
        </mgl-custom-control>

        <mgl-custom-control :position="Position.TOP_LEFT">
          <div style="margin: 8px">
            <div style="width: 150px; height: 48px">{{ socialLabel }}</div>

            <table>
              <tr
                ><td rowspan="8" scope="rowgroup">
                  <div
                    :style="`height:128px;width:16px;background: linear-gradient(0deg,${socialFeature?.lowColor} 0%, ${socialFeature?.highColor} 100%);`"
                  ></div></td
                ><td>{{
                  fillColor?.["fill-color"]?.[5]?.toLocaleString("en-us", {
                    maximumFractionDigits: 2,
                    useGrouping: false
                  })
                }}</td></tr
              >
              <tr><td scope="row">.</td></tr>
              <tr><td scope="row">.</td></tr>
              <tr><td scope="row">.</td></tr>
              <tr
                ><td scope="row">{{
                  fillColor?.["fill-color"]?.[3]?.toLocaleString("en-us", {
                    maximumFractionDigits: 2,
                    useGrouping: false
                  })
                }}</td></tr
              >
            </table>
          </div>
        </mgl-custom-control>

        <mgl-custom-control :position="Position.TOP_RIGHT">
          <button
            class="maplibregl-ctrl-icon"
            :style="{
              color: currentTool === 'general' ? 'deepSkyBlue' : 'grey'
            }"
            @click="currentTool = 'general'"
          >
            <svg>
              <path :d="mdiCursorDefault" fill="currentColor" />
            </svg>
          </button>
        </mgl-custom-control>

        <mgl-custom-control :position="Position.TOP_RIGHT">
          <button
            class="maplibregl-ctrl-icon"
            :style="{
              color: currentTool === 'select' ? 'deepSkyBlue' : 'grey'
            }"
            @click="currentTool = 'select'"
          >
            <svg>
              <path :d="mdiSelect" fill="currentColor" />
            </svg>
          </button>
        </mgl-custom-control>

        <mgl-marker
          :coordinates="[4.125239818838533, 51.9785307340928]"
        ></mgl-marker>
        <mgl-marker
          :coordinates="[4.457889979857661, 51.871067129276618]"
        ></mgl-marker>
        <mgl-popup
          v-if="popupShow"
          :close-button="false"
          :coordinates="popupCoordinates"
          :close-on-click="false"
          :close-on-move="false"
          :text="popupText"
        />
        <mgl-geo-json-source source-id="buildings" :data="buildings">
          <mgl-fill-extrusion-layer
            layer-id="buildings"
            :paint="paintBuildings"
            @mousemove="hoverBuildingEnter"
            @mouseout="hoverBuildingLeave"
            @click="clickBuilding"
          >
          </mgl-fill-extrusion-layer>
        </mgl-geo-json-source>

        <mgl-geo-json-source
          source-id="geojson"
          :generate-id="true"
          :data="rotterdamUrl"
        >
          <mgl-fill-layer
            layer-id="geojson"
            :paint="{ ...defaultPaint, ...fillColor }"
            @mousemove="hoverSquareEnter"
            @mouseout="hoverSquareLeave"
            @click="squareZoom"
          />

          <mgl-line-layer layer-id="select" :paint="paintSelection" />
        </mgl-geo-json-source>

        <mgl-geo-json-source source-id="boundaries" :data="boundaryUrl">
          <mgl-line-layer
            layer-id="boundaries"
            :paint="paintBoundaries"
          ></mgl-line-layer>
        </mgl-geo-json-source>
        <mgl-geo-json-source
          source-id="boundariesLabels"
          :data="boundariesJSON"
        >
          <mgl-symbol-layer
            layer-id="names"
            :layout="{ 'text-field': ['get', 'buurtnaam'] }"
            :paint="{
              'text-color': '#9e0000',
              'text-opacity': 0.75,
              'text-halo-color': 'white',
              'text-halo-width': 1
            }"
          ></mgl-symbol-layer>
        </mgl-geo-json-source>

        <mgl-navigation-control /> </mgl-map
    ></div>
  </div>
</template>

<script setup lang="ts">
import {
  MglMap,
  MglNavigationControl,
  MglCustomControl,
  MglGeoJsonSource,
  MglFillLayer,
  MglLineLayer,
  MglSymbolLayer,
  MglFillExtrusionLayer,
  MglPopup,
  MglMarker,
  Position,
  useMap
} from "@indoorequal/vue-maplibre-gl";

import {
  Responsive,
  Bar,
  Chart,
  Line,
  Marker,
  Grid,
  Tooltip,
  HoverBar
} from "vue3-charts";

import workerUrl from "maplibre-gl/dist/maplibre-gl-worker.mjs?worker&url";
import {
  computed,
  nextTick,
  onMounted,
  ref,
  shallowRef,
  toRaw,
  useTemplateRef,
  watch
} from "vue";
import { Color, useQuasar } from "quasar";
import { center, featureCollection, isNumber, union } from "@turf/turf";
import { loadPyodide, PyodideAPI } from "pyodide";
import { setWorkerUrl } from "maplibre-gl";
import { v4 as uuidv4 } from "uuid";
import * as np from "numpy-ts";

import bdata from "../assets/kralingseveer_buildings.json";
import {
  mdiCursorDefault,
  mdiCursorDefaultClick,
  mdiCursorMove,
  mdiPan,
  mdiSelect
} from "@mdi/js";
import VerticalMarker from "@/components/VerticalMarker.vue";
setWorkerUrl(workerUrl);
// async function main() {

// console.log("Pyodide and GeoPandas are ready!");
// return pyodide;
// }

const rotterdamUrl = "roffa_grid_gwrf_combined_wgs84_fixed.geojson";
const boundaryUrl = "roffa_boundaries_wgs84.geojson";

interface RetrofitResult {
  name: string;
  heat: number;
  cool: number;
  elec: number;
  gas: number;
  co2: number;
}

const beforeRetrofitData = ref<RetrofitResult>({
  name: "Before Retrofit",
  heat: 0,
  cool: 0,
  elec: 0,
  gas: 0,
  co2: 0
});
const afterRetrofitData = ref<RetrofitResult>({
  name: "After Retrofit",
  heat: 0,
  cool: 0,
  elec: 0,
  gas: 0,
  co2: 0
});

const style = "minimal.json";
const zoom = 12;
const mapHolder = useMap();
const buildings = ref({});
const $q = useQuasar();
const pyodideInstance = ref<PyodideAPI>();
const showSocialFeatures = ref<boolean>(false);
const showComputedFeatures = ref<boolean>(false);
const retrofit = ref<RetrofitPackage>();

const activeCoolingRetrofits = ref<ActiveRetrofit[]>([
  {
    label: "None",
    value: { name: "do_nothing", retro_type: "cooling" }
  },
  {
    label: "Air Conditioning",
    value: {
      name: "retrofit_dx_airco",
      retro_type: "cooling",
      params: { dx_cop: 4 }
    }
  }
]);
const activeHeatingRetrofits = ref<ActiveRetrofit[]>([
  {
    label: "None",
    value: { name: "do_nothing", retro_type: "heating" }
  },
  {
    label: "Gas Boiler",
    value: {
      name: "retrofit_gas_boiler",
      retro_type: "heating",
      params: { boiler_eff: 0.95 }
    }
  },
  {
    label: "Heat Pump",
    value: {
      name: "retrofit_heat_pump",
      retro_type: "heating",
      params: { hp_cop: 5 }
    }
  },
  {
    label: "District Heating",
    value: { name: "retrofit_district_heating", retro_type: "heating" }
  }
]);
const coolingRetrofit = ref<ActiveRetrofit>(
  activeCoolingRetrofits.value[0]?.value
);
const heatingRetrofit = ref<ActiveRetrofit>(
  activeHeatingRetrofits.value[0]?.value
);

interface ActiveRetrofit {
  label: string;
  value: {
    name: string;
    retro_type: "heating" | "cooling";
    params?: {
      [key: string]: number;
    };
  };
}

interface RetrofitPackage {
  label: string;
  value: {
    wall: number;
    roof: number;
    floor: number;
    window: number;
    inf_rate?: number;
  };
}
// For each of the retrofit package they choose, they should be able to see the results with
// all heating and cooling system types.
// For heating we have gas boiler
// ((1)gas_boiler, boiler_eff:0.95, (2)heat_pump, hp_cop: 5 and (3) district heating(district_heating)).
// For cooling we have air conditioning (dx_airco, dx_cop:4).
// Then we should also show overheating hours in case that there is no cooling system (no_cooling), and underheating in case there is no heating (no_heating).

const retrofits: RetrofitPackage[] = [
  {
    label: "Shallow",
    value: {
      wall: 3.9,
      roof: 4.2,
      floor: 3.1,
      window: 2
    }
  },
  {
    label: "Deep",
    value: {
      wall: 6.2,
      roof: 8.3,
      floor: 5.7,
      window: 3
    }
  },
  {
    label: "Deep (Limited Infiltration)",
    value: {
      wall: 3.9,
      roof: 4.2,
      floor: 3.1,
      window: 2,
      inf_rate: 0.0004
    }
  }
];

const user_uuid = $q.cookies.useCookie("user", {
  default: () => "",
  expires: "30d"
});

// TODO: Nicest would be to have a tiny tool interface with some predefined tools
//       but I don't think Typescript supports case classes..
const currentTool = ref<"general" | "select">("general");
watch(currentTool, async newTool => {
  console.log("changed!", newTool);
  switch (newTool) {
    case "general":
      mapHolder?.map?.dragPan.enable();
      mapHolder?.map?.getCanvas().style.setProperty("cursor", "");
      break;
    case "select":
      mapHolder?.map?.dragPan.disable();
      mapHolder?.map?.getCanvas().style.setProperty("cursor", "crosshair");
      break;
  }
  console.log(currentTool.value);
});

watch(showSocialFeatures, async newVal => {
  if (!newVal) {
    socialFeature.value = undefined;
  }
});

let hoveredStateId: number | undefined = undefined;
const selectedStateId = ref<number>();
let hoveredBuildingStateId: number | undefined = undefined;
const selectedBuildingIds = ref<number[]>([]);
const buildingIdToVisualize = ref<number>();
const boundariesJSON = ref<object>({ data: {} });
const socialJSON = ref<object>();
const popupCoordinates = ref<[number, number]>([0, 0]);
const popupShow = ref(false);
const popupText = ref("");
let popupTimeout: number = 0;

// watch(buildingIdToVisualize, async newVal => {
//   const res = await fetch(
//     `api/idf/?user_id=${user_uuid.value}?building_id=${newVal}`,
//     {
//       method: "GET"
//     }
//   );
// });

interface SocialFeature {
  label: string;
  value: {
    name: string;
    lowColor: string;
    highColor: string;
  };
}

const socialFeatures: SocialFeature[] = [
  {
    label: "Average Gas Consumption (m³/year)",
    value: {
      name: "avg_gas_consumption_mu",
      lowColor: "#20cf2c",
      highColor: "#6e004f"
    }
  },
  {
    label: "Average Electricity Consumption (kWh/year)",
    value: {
      name: "avg_electricity_consumption_mu",
      lowColor: "#00216e",
      highColor: "#f0e404"
    }
  },
  {
    label: "Rental (%)",
    value: {
      name: "pct_rental_dwellings_mu",
      lowColor: "#111111",
      highColor: "#afafaf"
    }
  },
  {
    label: "Social Rent (% of Rental)",
    value: {
      name: "pct_social_rent_mu",
      lowColor: "#111111",
      highColor: "#afafaf"
    }
  },
  {
    label: "Home Owner (%)",
    value: {
      name: "pct_owner_occupied_dwellings_mu",
      lowColor: "#111111",
      highColor: "#9a2dff"
    }
  },
  {
    label: "Income Band",
    value: {
      name: "income_band",
      lowColor: "#0e2d2d",
      highColor: "#29e2ff"
    }
  },
  {
    label: "Social Benefits",
    value: {
      name: "pct_on_benefits_mu",
      lowColor: "#100e2d",
      highColor: "#ff29b8"
    }
  },
  {
    label: "Dutch Background (%)",
    value: {
      name: "pct_nl_background_mu",
      lowColor: "#e74bd2",
      highColor: "#ffff29"
    }
  },
  {
    label: "Appartment Density (%)",
    value: {
      name: "pct_multi_family_dwellings_mu",
      lowColor: "#3a3a29",
      highColor: "#6f7bff"
    }
  },
  {
    label: "Average Household Size",
    value: {
      name: "avg_household_size_mu",
      lowColor: "#3a3a29",
      highColor: "#ff7c11"
    }
  },
  {
    label: "65+ Percentage (%)",
    value: {
      name: "pct_age_65_plus_mu",
      lowColor: "#212f5c",
      highColor: "#a5a5a5"
    }
  }
] as const;

const socialFeature = ref<SocialFeature["value"]>();

const socialLabel = computed(
  () =>
    socialFeatures.find(s => s.value.name == socialFeature.value?.name)?.label
);

// NOTE: Might not work for categorical data.
const fillColor = computed(() => {
  if (socialFeature.value) {
    return {
      "fill-color": [
        "interpolate",
        ["linear"],
        ["get", socialFeature.value?.name], // Numerical GeoJSON property
        np.percentile(np.array(stats.value), 5),
        socialFeature.value?.lowColor,
        np.percentile(np.array(stats.value), 95),
        socialFeature.value?.highColor
      ]
    };
  } else {
    return {};
  }
});

const selectedSquareFeatureValue = computed(() => {
  if (socialFeature.value && selectedStateId.value) {
    return (
      mapHolder?.map
        ?.querySourceFeatures("geojson")
        .find(f => f.id == selectedStateId.value)?.properties[
        socialFeature.value?.name
      ] ?? undefined
    );
  } else {
    return undefined;
  }
});

const stats = computed(() => {
  if (socialFeature.value) {
    return socialJSON.value.features
      .map(f => f.properties[socialFeature.value?.name])
      .filter(v => v != null && v > 0);
  } else {
    return [];
  }
});

const dataHigh = computed(() => hist.value?.[hist.value?.length - 1]["x"] ?? 0);
const dataLow = computed(() => hist.value?.[0]["x"] ?? 0);

const hist = computed(() => {
  const [counts, binEdges] = np.histogram(np.array(stats.value));
  // np.random.seed(42);
  // const data = np.random.normal(0, 1, [1000]);

  // // Automatic binning (10 bins by default)
  // const [counts, binEdges] = np.histogram(
  //   np.array([1, 2, 3, 4, 5, 2, 3, 4, 1, 3])
  // );
  // console.log("Counts:", counts, "\n");
  // console.log("Bin edges:", binEdges, "\n");
  // console.log("Number of bins:", counts.shape[0], "\n");
  const edges = binEdges.toArray();
  return counts.toArray().map((c, i) => {
    return { x: Math.round(edges[i]), y: c };
  });
});

const defaultPaint = {
  "fill-outline-color": [
    "case",
    ["boolean", ["feature-state", "selected"], false],
    "#ff0000ff",
    "#a09c9c45"
  ],
  "fill-opacity": [
    "case",
    ["boolean", ["feature-state", "hover"], false],
    1,
    0.75
  ],
  "fill-color": "#7f837f94"
};

// let paint = ref({
//   "fill-color": [
//     "interpolate",
//     ["linear"],
//     ["get", "avg_gas_consumption_mu"], // Numerical GeoJSON property
//     0,
//     "#20cf2c",
//     // 500, '#3978eb',
//     2000,
//     "#6e004f"
//   ],
//   "fill-opacity": [
//     "case",
//     ["boolean", ["feature-state", "hover"], false],
//     1,
//     0.9
//   ]
// });

const paintBoundaries = {
  "line-color": "#fc0000",
  "line-layer-opacity": 0.5,
  "line-width": 2
};

const paintSelection = {
  "line-color": "#fc0000",
  "line-opacity": [
    "case",
    ["boolean", ["feature-state", "selected"], false],
    0.5,
    ["boolean", ["feature-state", "hover"], false],
    0.2,
    0
  ],
  "line-width": [
    "case",
    ["boolean", ["feature-state", "selected"], false],
    4,
    2
  ]
};

let paintBuildings = ref({
  // "fill-extrusion-outline-color": "#0000FF",

  "fill-extrusion-color": [
    "case",
    ["boolean", ["feature-state", "highlight"], false],
    "#ff0000",
    ["boolean", ["feature-state", "click"], false],
    "#f8ff2d",
    ["boolean", ["feature-state", "retrofit"], false],
    "#3b77af",
    ["boolean", ["feature-state", "hover"], false],
    "#9c9c9c",
    "#6f6f70"
  ]
});

function message(str: string, color = "purple") {
  $q.notify({
    message: str,
    color: color,
    position: "bottom-right"
  });
}

async function clickBuilding(e) {
  console.log(e);
  message(`Clicked building ${e.features[0].id}`);
  selectBuildings([e.features[0].id]);
  e.preventDefault();
}

// NOTE: May need debouncing for IDF querying. Method call that starts late?
async function selectBuildings(buildingIds: number[]) {
  for (const oldSelectedBuildingId of selectedBuildingIds.value.splice(
    0,
    selectedBuildingIds.value.length
  )) {
    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: oldSelectedBuildingId },
      { click: false }
    );
  }

  for (const newSelectedBuildingId of buildingIds) {
    selectedBuildingIds.value.push(newSelectedBuildingId);

    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: newSelectedBuildingId },
      { click: true }
    );
  }

  buildingIdToVisualize.value = selectedBuildingIds.value[0];
}

function resetHighlight(building: number) {
  setTimeout(() => {
    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: building },
      { highlight: false }
    );
  }, 1000);
}

async function predict() {
  if (selectedBuildingIds.value) {
    message(`Submitting buildings ${selectedBuildingIds.value}`);

    const res2 = await fetch(`api/predict/?user_id=${user_uuid.value}`, {
      method: "POST",
      body: JSON.stringify({
        buildings: selectedBuildingIds.value,
        r_package: retrofit.value?.value ?? {},
        r_active: [coolingRetrofit.value, heatingRetrofit.value]
      }),
      headers: {
        "Content-Type": "application/json"
      }
    });
    const firstBuilding = selectedBuildingIds.value[0];
    const json = JSON.parse(await res2.json());
    for (const buildingId of selectedBuildingIds.value) {
      const prediction = json[buildingId];
      message(
        `Cooling demand: ${prediction["cool"]} | Heating demand: ${prediction["heat"]}`,
        "green"
      );
    }
    await loadBuildings(selectedStateId.value);
  }
}

function hoverBuildingEnter(e) {
  if (e.features.length > 0) {
    if (hoveredBuildingStateId) {
      mapHolder?.map?.setFeatureState(
        { source: "buildings", id: hoveredBuildingStateId },
        { hover: false }
      );
      popupShow.value = false;
      clearTimeout(popupTimeout);
    }
    hoveredBuildingStateId = e.features[0].id;

    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: hoveredBuildingStateId },
      { hover: true }
    );

    popupCoordinates.value = e.lngLat;
    if (e.features[0].properties.has_retrofit) {
      clearTimeout(popupTimeout);
      popupText.value = `Retrofit Details: ${e.features[0].properties.retrofit_details}`;
      popupTimeout = setTimeout(() => {
        if (hoveredBuildingStateId) {
          popupShow.value = true;
        }
      }, 500);
    }
  }
}

function hoverBuildingLeave(e) {
  if (hoveredBuildingStateId) {
    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: hoveredBuildingStateId },
      { hover: false }
    );
  }
  hoveredBuildingStateId = undefined;
  popupShow.value = false;
  clearTimeout(popupTimeout);
}

function hoverSquareEnter(e) {
  if (e.features.length > 0) {
    if (hoveredStateId) {
      mapHolder?.map?.getCanvas().style.setProperty("cursor", "pointer");
      mapHolder?.map?.setFeatureState(
        { source: "geojson", id: hoveredStateId },
        { hover: false }
      );
    }
    if (
      selectedStateId.value === e.features[0].id &&
      mapHolder.map.getZoom() > 17
    )
      return;

    hoveredStateId = e.features[0].id;
    mapHolder?.map?.setFeatureState(
      { source: "geojson", id: hoveredStateId },
      { hover: true }
    );
  }
}

function hoverSquareLeave(e) {
  if (hoveredStateId) {
    mapHolder?.map?.getCanvas().style.setProperty("cursor", "");
    mapHolder?.map?.setFeatureState(
      { source: "geojson", id: hoveredStateId },
      { hover: false }
    );
  }
  hoveredStateId = undefined;
}

// NOTE: This overwrites the hover effect.
//       Maybe having two layers is a solution.
function fadeSquare(e) {
  mapHolder?.map?.setPaintProperty(
    "geojson",
    "fill-layer-opacity",
    (0.9 * Math.pow(zoom, 4)) / Math.pow(mapHolder.map.getZoom(), 4)
  );
}

async function loadBuildings(squareId: number) {
  // NOTE: Maplibre cuts polygons up when zooming in, which is why we have to do this annoying dance of
  //       recombining them based on the clicked id.
  const matchingFragments = mapHolder?.map?.querySourceFeatures("geojson", {
    sourceLayer: "geojson",
    filter: ["==", ["id"], squareId]
  });
  const combined = union(
    featureCollection([...matchingFragments, ...matchingFragments])
  );
  const square = combined.geometry.coordinates[0];

  mapHolder?.map?.flyTo({
    center: center(combined).geometry.coordinates,
    zoom: 18
  });

  // NOTE: This is for when we actually will use a backend.
  const res2 = await fetch(`api/buildings/?user_id=${user_uuid.value}`, {
    method: "POST",
    body: JSON.stringify({ coordinates: square }),
    headers: {
      "Content-Type": "application/json"
    }
  });

  // NOTE: This is for github demo-ing purposes.
  // const str = JSON.stringify({ coordinates: square });
  // const value = await pyodideInstance.value?.runPythonAsync(`
  //   square = json.loads('${str}')
  //   indices = gdf.sindex.query(convex_hull(Polygon(square["coordinates"])), predicate="intersects")
  //   intersected: gpd.GeoDataFrame = gdf.iloc[indices]
  //   intersected.to_json()
  // `);
  // const parsed = JSON.parse(value);

  // console.log(parsed);
  // buildings.value = parsed;
  buildings.value = JSON.parse(await res2.json());

  for (const attr of ["heat", "cool", "elec", "gas", "co2"]) {
    beforeRetrofitData.value[attr] = np.nanmean(
      np.array(
        buildings.value.features.map(feat => feat.properties[`init_${attr}`])
      )
    ) as number;
    afterRetrofitData.value[attr] = np.nanmean(
      np.array(
        buildings.value.features.map(feat => feat.properties[`pred_${attr}`])
      )
    ) as number;
  }

  buildings.value.features.forEach(building => {
    if (building.properties.has_retrofit) {
      mapHolder?.map?.setFeatureState(
        { source: "buildings", id: parseInt(building.id) },
        { retrofit: true }
      );
      console.log("building with retro", building.id);
    }
  });

  console.log(
    "buildings!",
    buildings.value,
    beforeRetrofitData.value,
    afterRetrofitData.value
  );
}

async function squareZoom(e) {
  if (e._defaultPrevented) return;

  if (e.features) {
    if (
      selectedStateId.value === e.features[0].id &&
      mapHolder.map.getZoom() > 17
    )
      return;

    if (selectedStateId.value) {
      mapHolder?.map?.setFeatureState(
        { source: "geojson", id: selectedStateId.value },
        { selected: false }
      );
    }
    selectedStateId.value = e.features[0].id;

    mapHolder?.map?.setFeatureState(
      { source: "geojson", id: selectedStateId.value },
      { selected: true }
    );

    await loadBuildings(selectedStateId.value);
  }
}

const handleMouseDownCapture = e => {
  if (e.button !== 0 || e.shiftKey) return;

  // Define shiftKey: true directly on the native MouseEvent before MapLibre reads it
  Object.defineProperty(e, "shiftKey", {
    get: () => currentTool.value == "select"
  });
};

watch(
  () => mapHolder.isLoaded,
  isLoaded => {
    if (!isLoaded || !mapHolder.map) return;

    const rawMap = toRaw(mapHolder.map);
    rawMap.boxZoom.enable();

    const container = rawMap.getCanvasContainer();
    // Note the `true` third argument (capture phase) so this runs BEFORE MapLibre's listener
    container.addEventListener("mousedown", handleMouseDownCapture, true);
  },
  { immediate: true }
);

onMounted(async () => {
  if (user_uuid.value === "") {
    user_uuid.value = uuidv4();
  }

  const bdata = await (await fetch(boundaryUrl)).json();
  boundariesJSON.value = {
    type: "FeatureCollection",
    features: bdata.features.map(feature => {
      const point = center(feature);
      point.properties = { ...feature.properties };
      return point;
    })
  };

  socialJSON.value = await (await fetch(rotterdamUrl)).json();

  console.log("huh", socialJSON.value);

  // const pyodide = await loadPyodide({
  //   indexURL: "https://cdn.jsdelivr.net/pyodide/v314.0.7/full/"
  // });
  // await pyodide.loadPackage("micropip");
  // const micropip = pyodide.pyimport("micropip");
  // await micropip.install("geopandas");
  // // const res = await fetch("kralingseveer_buildings.json");
  // const data = JSON.stringify(bdata);
  // pyodide.runPython(`
  //   import geopandas as gpd
  //   import json
  //   import io
  //   from shapely import Polygon, convex_hull
  //   gdf = gpd.read_file(io.StringIO('${data}'), fid_as_index=True).set_index("id")
  // `);
  // pyodideInstance.value = pyodide;
  // mapHolder?.map?.boxZoom.disable();
  if (mapHolder?.map) {
    // const handleMouseDown = e => {
    //   if (e.button !== 0 || !mapHolder?.map) return;

    //   // MapLibre's internal boxZoom checks `e.shiftKey` and `e.button === 0`
    //   const fakeShiftEvent = new MouseEvent("mousedown", {
    //     bubbles: true,
    //     cancelable: true,
    //     clientX: e.clientX,
    //     clientY: e.clientY,
    //     button: 0,
    //     buttons: 1,
    //     shiftKey: true
    //   });
    //   e.shiftKey = true;
    //   mapHolder?.map.boxZoom.mousedown(e, e.point);
    // };
    // const container = mapHolder.map.getCanvasContainer();
    // container.addEventListener("mousedown", handleMouseDown);

    //     const fakeShiftEvent = new MouseEvent('mousedown', {
    //   bubbles: true,
    //   cancelable: true,
    //   clientX: e.clientX,
    //   clientY: e.clientY,
    //   button: 0,
    //   buttons: 1,
    //   shiftKey: true,
    // });

    // rawMap.boxZoom._onMouseDown(fakeShiftEvent)

    mapHolder.map.boxZoom._boxZoomEnd = (mapInstance, p0, p1) => {
      const features = mapInstance.queryRenderedFeatures(
        [
          [Math.min(p0.x, p1.x), Math.min(p0.y, p1.y)],
          [Math.max(p0.x, p1.x), Math.max(p0.y, p1.y)]
        ],
        { layers: ["buildings"] }
      );
      const ids: number[] = [
        ...new Set(features.map(feature => feature.id).filter(id => id != null))
      ];

      selectBuildings(ids);
      console.log(ids);
    };
  }
  // mapHolder?.map?.dragPan.disable();
});
</script>

<style lang="scss">
@import "maplibre-gl/dist/maplibre-gl.css";
</style>
